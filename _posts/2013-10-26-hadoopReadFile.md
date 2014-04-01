---
layout: post
title: "HDFS读取文件的重要概念（转）"
category: 
- hadoop
tags: []
---






HDFS一个文件由多个block构成。HDFS在进行block读写的时候是以packet(默认每个packet为64K)为单位进行的。每一个packet由若干个chunk（默认512Byte）组成。Chunk是进行数据校验的基本单位，对每一个chunk生成一个校验和(默认4Byte)并将校验和进行存储。在读取一个block的时候，数据传输的基本单位是packet，每个packet由若干个chunk组成。</br></br>
 
##HDFS客户端读文件示例代码
{% highlight objc %}
FileSystem hdfs = FileSystem.get(new Configuration());
Path path = new Path("/testfile");// reading
FSDataInputStream dis = hdfs.open(path);
byte[] writeBuf = newbyte[1024];
int len = dis.read(writeBuf);
System.out.println(new String(writeBuf, 0, len, "UTF-8"));
dis.close();
hdfs.close();
{% endhighlight %}
##文件的打开
HDFS打开一个文件，需要在客户端调用DistributedFileSystem.open(Path f, int bufferSize)，其实现为：
{% highlight objc %}
public FSDataInputStream open(Path f, int bufferSize) throws IOException {  
    return new DFSClient.DFSDataInputStream(
        dfs.open(getPathName(f), bufferSize, verifyChecksum, statistics));
}
{% endhighlight %}
其中dfs为DistributedFileSystem的成员变量DFSClient，其open函数被调用，其中创建一个DFSInputStream(src, buffersize, verifyChecksum)并返回。</br></br>
DFSClient.DFSDataInputStream实现了HDFS的FSDataInputStream，里面简单包装了DFSInputStream，实际实现是DFSInputStream完成的。</br></br>
在DFSInputStream的构造函数中，openInfo函数被调用，其主要从namenode中得到要打开的文件所对应的blocks的信息，实现如下：
{% highlight objc %}
synchronized void openInfo() throws IOException {
    LocatedBlocks newInfo = callGetBlockLocations(namenode, src, 0, prefetchSize);
    this.locatedBlocks = newInfo;  
    this.currentNode = null;
}
private static LocatedBlocks callGetBlockLocations(ClientProtocol namenode,String src, long start, long length) throws IOException {    
    return namenode.getBlockLocations(src, start, length);
}
{% endhighlight %}
LocatedBlocks主要包含一个链表的List<LocatedBlock> blocks，其中每个LocatedBlock包含如下信息：</br>
### * Block b：此block的信息
###* long offset：此block在文件中的偏移量
###* DatanodeInfo[] locs：此block位于哪些DataNode上
上面namenode.getBlockLocations是一个RPC调用，最终调用NameNode类的getBlockLocations函数。NameNode返回的是根据客户端请求的文件名字，文件偏移量，数据长度，返回文件对应的数据块列表，数据块所在的DataNode节点。</br></br>
 
##文件的顺序读取
 hdfs文件的顺序读取是最经常使用的.文件顺序读取的时候，客户端利用文件打开的时候得到的FSDataInputStream.read(byte[] buffer, int offset, int length)函数进行文件读操作。</br></br>
FSDataInputStream会调用其封装的DFSInputStream的read(byte[] buffer, int offset, int length)函数，实现如下：
{% highlight objc %}
public synchronized int read(byte buf[], int off, int len) throws IOException {    
    ...    
    if (pos < getFileLength()) {    
        int retries = 2;    
        while (retries > 0) {      
            try {        
                if (pos > blockEnd) {//首次pos=0,blockEnd=-1,必定调用方法blockSeekTo,初始化blockEnd,以后是读完了当前块，需要读下一个块，才会调用blockSeekTo
                    currentNode = blockSeekTo(pos);//根据pos选择块和数据节点，选择算法是遍历块所在的所有数据节点，选择第一个非死亡节点
                }
                int realLen = Math.min(len, (int) (blockEnd - pos + 1));
                int result = readBuffer(buf, off, realLen);
                if (result >= 0) {
                    pos += result;        
                } else {          
                    throw new IOException("Unexpected EOS from the reader");        
                }        
                ...        
                return result;      
            } catch (ChecksumException ce) {        
                throw ce;                
            } catch (IOException e) {      
                ...        
                if (currentNode != null) { 
                    addToDeadNodes(currentNode); 
                }//遇到无法读的DataNode,添加到死亡节点        
                if (--retries == 0) {//尝试读三次都失败，就抛出异常         
                    throw e;        
                }      
            }    
        }    
    }    
    return -1;
}
{% endhighlight %}
 blockSeekTo函数会更新blockEnd,并创建对应的BlockReader,这里的BlockReader的初始化和上面的fetchBlockByteRange差不多，如果客户端和块所属的DataNode是同个节点，则初始化一个通过本地读取的BlockReader，否则创建一个通过Socket连接DataNode的BlockReader。</br></br>
BlockReader的创建也是通过BlockReader.newBlockReader创建的，具体分析请看后面。
readBuffer方法比较简单，直接调用BlockReader的read方法直接读取数据。
BlockReader的read方法就根据请求的块起始偏移量，长度，通过socket连接DataNode，获取块内容，BlockReader的read方法不会做缓存优化。</br></br>
 
##文件的随机读取
对于MapReduce，在提交作业时，已经确定了每个map和reduce要读取的文件，文件的偏移量，读取的长度，所以MapReduce使用的大部分是文件的随机读取。</br></br>
文件随机读取的时候，客户端利用文件打开的时候得到的FSDataInputStream.read(long position, byte[] buffer, int offset, int length)函数进行文件读操作。</br></br>
FSDataInputStream会调用其封装的DFSInputStream的read(long position, byte[] buffer, int offset, int length)函数，实现如下：
{% highlight objc %}
public int read(long position, byte[] buffer, int offset, int length)throws IOException {
    long filelen = getFileLength(); 
    int realLen = length;  
    if ((position + length) > filelen) {    
        realLen = (int)(filelen - position);  
    }  
    //首先得到包含从offset到offset + length内容的block列表  
    //比如对于64M一个block的文件系统来说，欲读取从100M开始，长度为128M的数据，则block列表包括第2，3，4块block  
    List<LocatedBlock> blockRange = getBlockRange(position, realLen);  
    int remaining = realLen;  
    //对每一个block，从中读取内容  
    //对于上面的例子，对于第2块block，读取从36M开始，读取长度28M，对于第3块，读取整一块64M，对于第4块，读取从0开始，长度为36M，共128M数据  
    for (LocatedBlock blk : blockRange) {    
        long targetStart = position - blk.getStartOffset();    
        long bytesToRead = Math.min(remaining, blk.getBlockSize() - targetStart);    fetchBlockByteRange(blk, targetStart, targetStart + bytesToRead - 1, buffer, offset);    
        remaining -= bytesToRead;    
        position += bytesToRead;    
        offset += bytesToRead;  
    }  
    ...
    return realLen;
}
{% endhighlight %}
getBlockRange方法根据文件的偏移量和长度，获取对应的数据块信息。主要是根据NameNode类的getBlockLocations方法实现，并做了缓存和二分查找等优化。</br></br>
 fetchBlockByteRange方法真正从数据块读取内容，实现如下：
 {% highlight objc %}
private void fetchBlockByteRange(LocatedBlock block, long start,long end, byte[] buf, int offset) throws IOException {
    Socket dn = null;  
    int numAttempts = block.getLocations().length;  
    //此while循环为读取失败后的重试次数  
    while (dn == null && numAttempts-- > 0 ) {    
        //选择一个DataNode来读取数据    
        DNAddrPair retval = chooseDataNode(block);    
        DatanodeInfo chosenNode =retval.info;    
        InetSocketAddress targetAddr = retval.addr;    
        BlockReader reader = null;    
        int len = (int) (end - start + 1);    
        try {      
            if (shouldTryShortCircuitRead(targetAddr)) {        
                //如果要读取的块所属的DataNode与客户端是同一个节点，直接通过本地磁盘访问，减少网络流量        
                reader = getLocalBlockReader(conf, src, block.getBlock(),accessToken, chosenNode, DFSClient.this.socketTimeout, start);      
            } else {        
                //创建Socket连接到DataNode        
                dn = socketFactory.createSocket();        
                dn.connect(targetAddr, socketTimeout);        
                dn.setSoTimeout(socketTimeout);        
                //利用建立的Socket链接，生成一个reader负责从DataNode读取数据        reader = BlockReader.newBlockReader(dn, src, block.getBlock().getBlockId(), accessToken,block.getBlock().getGenerationStamp(), start, len, buffersize, verifyChecksum, clientName);      
            }              
            //读取数据      
            int nread = reader.readAll(buf, offset, len);      
            return;    
        } finally {      
            IOUtils.closeStream(reader);      
            IOUtils.closeSocket(dn);      
            dn = null;   
        }    
        //如果读取失败，则将此DataNode标记为失败节点    
        addToDeadNodes(chosenNode);  
    }
}
{% endhighlight %}
读取块内容，会尝试该数据块所在的所有DataNode，如果失败，就把对应的DataNode加入到失败节点，下次选择节点就会忽略失败节点(只在独立的客户端缓存失败节点，不上报到namenode)。</br></br>
BlockReader的创建也是通过BlockReader.newBlockReader创建的，具体分析请看后面。
最后，通过BlockReader的readAll方法读取块的完整内容。</br></br>
 
##dfsclient和datanode的通信协议
###dfsclient的连接
dfsclient首次连接datanode时，通信协议实现主要是BlockReader.newBlockReader方法的实现，如下：
{% highlight objc %}
public static BlockReader newBlockReader( Socket sock, String file,long blockId,long genStamp,long startOffset, long len,int bufferSize, boolean verifyChecksum,String clientName) throws IOException {  
    //使用Socket建立写入流，向DataNode发送读指令  
    DataOutputStream out = new DataOutputStream(new BufferedOutputStream(NetUtils.getOutputStream(sock,HdfsConstants.WRITE_TIMEOUT)));  
    out.writeShort( DataTransferProtocol.DATA_TRANSFER_VERSION );  
    out.write( DataTransferProtocol.OP_READ_BLOCK );  
    out.writeLong( blockId );  out.writeLong( genStamp );  
    out.writeLong( startOffset );  
    out.writeLong( len );  
    Text.writeString(out, clientName);  
    out.flush();  
    //使用Socket建立读入流，用于从DataNode读取数据  
    DataInputStream in = new DataInputStream(new BufferedInputStream(NetUtils.getInputStream(sock),bufferSize));  
    short status = in.readShort();
    //块读取的状态标记，一般是成功  
    DataChecksum checksum = DataChecksum.newDataChecksum( in );  
    long firstChunkOffset = in.readLong();  
    //生成一个reader，主要包含读入流，用于读取数据  
    return new BlockReader( file, blockId, in, checksum, verifyChecksum,startOffset, firstChunkOffset, sock );
}
{% endhighlight %}
这里的startOffset是相对于块的起始偏移量，len是要读取的长度。
DataChecksum.newDataChecksum(in),会从DataNode获取该块的checksum加密方式，加密长度。
BlockReader的readAll函数就是用上面生成的DataInputStream读取数据。
 下面是是读数据块时，客户端发送的信息：
 </br>
version&nbsp;&nbsp;operator&nbsp;&nbsp;blockid&nbsp;&nbsp;generationStamp&nbsp;&nbsp;startOffset&nbsp;&nbsp;length&nbsp;&nbsp;clientName&nbsp;&nbsp;accessToken
####operator:byte Client所需要的操作，读取一个block、写入一个block等等
####version:short Client所需要的数据与Datanode所提供数据的版本是否一致
####blockId:long 所要读取block的blockId
####generationStamp:long 所需要读取block的generationStamp
####startOffset:long 读取block的的起始位置
####length:long 读取block的长度
####clientName:String Client的名字
####accessToken:Token Client提供的验证信息，用户名密码等

###DataNode对dfsclient的响应
DataNode负责与客户端代码的通信协议交互的逻辑，主要是DataXceiver的readBlock方法实现的：
{% highlight objc %}
private void readBlock(DataInputStream in) throws IOException {  
    //读取指令  
    long blockId = in.readLong(); 
    Block block = new Block( blockId, 0 , in.readLong());  
    long startOffset = in.readLong();  
    long length = in.readLong();  
    String clientName = Text.readString(in);  
    //创建一个写入流，用于向客户端写数据  
    OutputStream baseStream = NetUtils.getOutputStream(s,datanode.socketWriteTimeout);  
    DataOutputStream out = new DataOutputStream(new BufferedOutputStream(baseStream, SMALL_BUFFER_SIZE));  
    //生成BlockSender用于读取本地的block的数据，并发送给客户端  
    //BlockSender有一个成员变量InputStream blockIn用于读取本地block的数据  BlockSender blockSender = new BlockSender(block, startOffset, length,true, true, false, datanode, clientTraceFmt);  
    out.writeShort(DataTransferProtocol.OP_STATUS_SUCCESS); 
    // 发送操作成功的状态  
    //向客户端写入数据  
    long read = blockSender.sendBlock(out, baseStream, null);  
    ……  
    } finally {    
        IOUtils.closeStream(out);    
        IOUtils.closeStream(blockSender);  
}
{% endhighlight %}
DataXceiver的sendBlock用于发送数据，数据发送包括应答头和后续的数据包。应答头如下（包含DataXceiver中发送的成功标识。DataXceiver的sendBlock的实现如下：
{% highlight objc %}
long sendBlock(DataOutputStream out, OutputStream baseStream,                  BlockTransferThrottler throttler) throws IOException {    
    ...    
    try {      
        try {        
            checksum.writeHeader(out);//写入checksum的加密类型和加密长度        
            if ( chunkOffsetOK ) {          
                out.writeLong( offset );       
            }        
            out.flush();      
        } catch (IOException e) { 
            //socket error        
            throw ioeToSocketException(e);      
        }            
        ...      
        ByteBuffer pktBuf = ByteBuffer.allocate(pktSize);      
        while (endOffset > offset) {//循环写入数据包        
            long len = sendChunks(pktBuf, maxChunksPerPacket,streamForSendChunks); 
            offset += len;       
            totalRead += len + ((len + bytesPerChecksum - 1)/bytesPerChecksum*checksumSize); 
            seqno++;      
        }      
        try {        
            out.writeInt(0); //标记结束            
            out.flush();      
        } catch (IOException e) { //socket error        
            throw ioeToSocketException(e);      
        }    
    }    
    ...    
    return totalRead;
}
{% endhighlight %}
DataXceiver的sendChunks尽可能在一个packet发送多个chunk，chunk的个数由maxChunks和剩余的块内容决定，实现如下:
{% highlight objc %}
//默认是crc校验，bytesPerChecksum默认是512，checksumSize默认是4，表示数据块每512个字节，做一次checksum校验，checksum的结果是4个字节
private int sendChunks(ByteBuffer pkt, int maxChunks, OutputStream out) throws IOException {    
    int len = Math.min((int) (endOffset - offset),bytesPerChecksum * maxChunks);//len是要发送的数据长度    
    if (len == 0) {      
        return 0;    
    }    
    int numChunks = (len + bytesPerChecksum - 1)/bytesPerChecksum;//这次要发送的chunk数量    
    int packetLen = len + numChunks*checksumSize + 4;//packetLen是整个包的长度，包括包头，校验码，数据    
    pkt.clear();        
    // write packet header    
    pkt.putInt(packetLen);//整个packet的长度    
    pkt.putLong(offset);//块的偏移量    
    pkt.putLong(seqno);//序列号    
    pkt.put((byte)((offset + len >= endOffset) ? 1 : 0));//是否最后一个packet    
    pkt.putInt(len);//发送的数据长度        
    int checksumOff = pkt.position();    
    int checksumLen = numChunks * checksumSize;    
    byte[] buf = pkt.array();        
    if (checksumSize > 0 && checksumIn != null) {      
        try {        
            checksumIn.readFully(buf, checksumOff, checksumLen);//填充chucksum的内容
        } catch (IOException e) {       
        ...      
        }    
    }        
    int dataOff = checksumOff + checksumLen;    
    if (blockInPosition < 0) {      
        IOUtils.readFully(blockIn, buf, dataOff, len);//填充块数据的内容      
        if (verifyChecksum) {//默认是false,不验证        
            //校验处理      
        }    
    }        
    try {      
        //通过socket发送数据到客户端          
    } catch (IOException e) {      
        throw ioeToSocketException(e);    
    }    
    ...    
    return len;
}
{% endhighlight %}
数据组织成数据包来发送，数据包结构如下：
packetLen&nbsp;&nbsp;offset&nbsp;&nbsp;sequenceNum&nbsp;&nbsp;isLastPacket&nbsp;&nbsp;startOffset&nbsp;&nbsp;dataLen&nbsp;&nbsp;checksum&nbsp;&nbsp;data
####packetLen:int packet的长度，包括数据、数据的校验等等
####offset:long packet在block中的偏移量
####sequenceNum:long 该packet在这次block读取时的序号
####isLastPacket:byte packet是否是最后一个
####dataLen:int 该packet所包含block数据的长度，纯数据不包括校验和其他
####checksum:该packet每一个chunk的校验和，有多少个chunk就有多少个校验和
####data:该packet所包含的block数据
数据传输结束的标志，是一个packetLen长度为0的包。客户端可以返回一个两字节的应答OP_STATUS_CHECKSUM_OK(5)。
###dfsclient读取块内容
 hdfs文件的随机和顺序分析逻辑，都分析到BlockReader的readAll方法和read方法，这两个方法完成对数据块的内容读取。而readAll方法最后也是调用read方法，所以这里重点分析BlockReader的read方法，实现如下:
 {% highlight objc %}
public synchronized int read(byte[] buf, int off, int len) throws IOException {
    //第一次read, 忽略前面的额外数据      
    if (lastChunkLen < 0 && startOffset > firstChunkOffset && len > 0) {        
        int toSkip = (int)(startOffset - firstChunkOffset);        
        if ( skipBuf == null ) {          
            skipBuf = newbyte[bytesPerChecksum];        
        }        
        if ( super.read(skipBuf, 0, toSkip) != toSkip ) {
            //忽略          // should never happen          
            throw new IOException("Could not skip required number of bytes");        
        }      
    }            
    boolean eosBefore = gotEOS;      
    int nRead = super.read(buf, off, len);      
    // if gotEOS was set in the previous read and checksum is enabled :      
    if (dnSock != null && gotEOS && !eosBefore && nRead >= 0&& needChecksum()) {
        //checksum is verified and there are no errors.        
        checksumOk(dnSock);      
    }      
    return nRead;
}
{% endhighlight %}
super.read即是FSInputChecker的read方法，实现如下：
{% highlight objc %}
public synchronized int read(byte[] b, int off, int len) throws IOException {   
    //参数检查    
    int n = 0;    
    for (;;) {      
        int nread = read1(b, off + n, len - n);      
        if (nread <= 0)         
            return (n == 0) ? nread : n;      
        n += nread;      
        if (n >= len)        
            return n;    
    }
}
//read1的len被忽略，只返回一个chunk的数据长度(最后一个chunk可能不足一个完整chunk的长度)
private int read1(byte b[], int off, int len)  throws IOException {    
    int avail = count-pos;    
    if( avail <= 0 ) {      
        if(len>=buf.length) {        
            //直接读取一个数据chunk到用户buffer，避免多余一次复制
            //很巧妙，buf初始化的大小是chunk的大小，默认是512，这里的代码会在块的剩余内容大于一个chunk的大小时调用
            int nread = readChecksumChunk(b, off, len);        
            return nread;      
        } else {        
            //读取一个数据chunk到本地buffer,也是调用readChecksumChunk方法
            //很巧妙，buf初始化大小是chunk的大小，默认是512，这里的代码会在块的剩余内容不足一个chunk的大小时进入调用
            fill();        
            if( count <= 0 ) {          
                return -1;        
            } else {          
                avail = count;        
            }      
        }    
    }        
    //从本地buffer拷贝数据到用户buffer，避免最后一个chunk导致数组越界    
    int cnt = (avail < len) ? avail : len;    
    System.arraycopy(buf, pos, b, off, cnt);    
    pos += cnt;    
    return cnt;    
}
{% endhighlight %}
FSInputChecker的readChecksumChunk会读取一个数据块的chunk，并做校验，实现如下: 
{% highlight objc %}
//只返回一个chunk的数据长度(默认512，最后一个chunk可能不足一个完整chunk的长度)private int readChecksumChunk(byte b[], int off, int len)  throws IOException {    
    // invalidate buffer    
    count = pos = 0;              
    int read = 0;    
    boolean retry = true;    
    int retriesLeft = numOfRetries; 
    //本案例中，numOfRetries是1，也就是说不会多次尝试    
    do {      
        retriesLeft--;      
        try {        
            read = readChunk(chunkPos, b, off, len, checksum);        
            if( read > 0 ) {          
                if( needChecksum() ) {
                    //这里会做checksum校验            
                    sum.update(b, off, read);            
                    verifySum(chunkPos);          
                }          
                chunkPos += read;        
            }         
            retry = false;      
        } catch (ChecksumException ce) {          
            ...          
            if (retriesLeft == 0) {
                //本案例中，numOfRetries是1，也就是说不会多次尝试,失败了，直接抛出异常
                throw ce;          
            }
            //如果读取的chunk校验失败，以当前的chunkpos为起始偏移量，尝试新的副本          if (seekToNewSource(chunkPos)) {            
                seek(chunkPos);          
            } else {            
                //找不到新的副本,抛出异常            
                throw ce;          
            }        
        }    
    } while (retry);    
    return read;
}
{% endhighlight %}
readChunk方法由BlockReader实现，分析如下:
{% highlight objc %}
//只返回一个chunk的数据长度(默认512，最后一个chunk可能不足一个完整chunk的长度)protected synchronized int readChunk(long pos, byte[] buf, int offset,int len, byte[] checksumBuf) throws IOException {      
    //读取一个 DATA_CHUNK.      
    long chunkOffset = lastChunkOffset;      
    if ( lastChunkLen > 0 ) {        
        chunkOffset += lastChunkLen;      
    }            
    //如果先前的packet已经读取完毕，就读下一个packet。      
    if (dataLeft <= 0) {        
        //读包的头部        
        int packetLen = in.readInt();        
        long offsetInBlock = in.readLong();        
        long seqno = in.readLong();        
        boolean lastPacketInBlock = in.readBoolean();        
        int dataLen = in.readInt();        //校验长度                
        lastSeqNo = seqno;        
        isLastPacket = lastPacketInBlock;        
        dataLeft = dataLen;        
        adjustChecksumBytes(dataLen);        
        if (dataLen > 0) {          
            IOUtils.readFully(in, checksumBytes.array(), 0,checksumBytes.limit());
            //读取当前包的所有数据块内容对应的checksum,后面的流程会讲checksum和读取的chunk内容做校验        
        }      
    }      
    int chunkLen = Math.min(dataLeft, bytesPerChecksum); 
    //确定此次读取的chunk长度，正常情况下是一个bytesPerChecksum(512字节)，当文件最后不足一个bytesPerChecksum,读取剩余的内容。            
    if ( chunkLen > 0 ) {        
        IOUtils.readFully(in, buf, offset, chunkLen);//读取一个数据块的chunk        checksumBytes.get(checksumBuf, 0, checksumSize);      
    }            
    dataLeft -= chunkLen;      
    lastChunkOffset = chunkOffset;      
    lastChunkLen = chunkLen;      
    ...      
    if ( chunkLen == 0 ) {        
        return -1;      
    }      
    return chunkLen;
}
{% endhighlight %}
总结
 本文前面概要介绍了dfsclient读取文件的示例代码，顺序读取文件和随机读取文件的概要流程，最后还基于dfsclient和datanode读取块的过程，做了一个详细的分析。
