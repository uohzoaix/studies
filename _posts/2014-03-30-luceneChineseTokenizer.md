---
layout: post
title: "lucene之ChineseTokenizer解析"
description: "lucene自带的中文分词ChineseTokenizer代码分析"
category: 
- lucne
- 分词
tags: []
---



三个最主要的方法：
{% highlight objc %}
private final void push(char c) {</br>
    //length==0表明当前分词操作只读取了一个字符，start指示该字符在input中的位置，由于分词时offset++了，这时需要-1才是正确的start值</br>
    7. if (length == 0) start = offset-1;            // start of token</br>
    //length指示当前已分词的长度</br>
    buffer[length++] = Character.toLowerCase(c);  // buffer it</br>
}
{% endhighlight %}

{% highlight objc %}
private final boolean flush() {</br>
    if (length>0) {</br>
        //System.out.println(new String(buffer, 0, length));</br>
        //该词的内容</br>
        termAtt.setTermBuffer(buffer, 0, length);</br>
        //该词的位置信息：起始和结束</br>
        offsetAtt.setOffset(correctOffset(start), correctOffset(start+length));</br>
            return true;</br>
    }</br>
    else</br>
        return false;</br></br>
}
{% endhighlight %}

public boolean incrementToken() throws IOException {</br>
        clearAttributes();</br>
        //length代表当前已分词的长度，每次进入该方法都是一次新的分词，所以初始化为0</br>
        length = 0;</br>
        //每次分词从哪个位置开始，该值在每次调用该方法共享其值</br>
        start = offset;</br>
        while (true) {</br>
            final char c;</br>
            //表明已读取一个字符</br>
            offset++;</br>
            //第一次bufferIndex和dataLen都为0，将整个input读取到buffer中，bufferIndex指示buffer当前读取的位置</br>
          1.  if (bufferIndex >= dataLen) {</br>
                dataLen = input.read(ioBuffer);</br>
                bufferIndex = 0;</br>
            }</br>
            //dataLen==-1表明input已经读完即分词到句子末尾</br>
          2.  if (dataLen == -1) {</br>
              offset--;</br>
              return flush();</br>
            } else</br>
          3.      c = ioBuffer[bufferIndex++];</br>
            switch(Character.getType(c)) {</br>
            //除中文字符外的字符</br>
            case Character.DECIMAL_DIGIT_NUMBER:</br>
            case Character.LOWERCASE_LETTER:</br>
            case Character.UPPERCASE_LETTER:</br>
           4.     push(c);//将当前字符放进buffer中</br>
                //如果当前已分词长度到达了一个词的最大长度，则将当前这个词返回</br>
                if (length == MAX_WORD_LEN) return flush();</br>
                break;//否则继续while循环</br>
            case Character.OTHER_LETTER:</br>
                //当前字符是中文</br>
                if (length>0) {//如果之前已有其他字符</br>
                    //这里是最难以理解的：因为lucene自带的中文分词会将中文和其他字符分开处理，所以在遇到中文字符时之前已经遍历过其他字符则必须将之前的字符返回作为一个单独的词对待，当前这个中文字符必须等待下次进入该方法时处理，所以必须将bufferIndex和offset递减，否则该中文字符会被丢失</br>
           5.      bufferIndex--;</br>
                    offset--;</br>
                    return flush();</br>
                }</br>
                //该中文字符是该词的第一个字符，也立即返回（lucene自带的中文分词会将中文按字分开）</br>
          6.    push(c);</br>
                return flush();</br>
            default:</br>
                if (length>0) return flush();</br>
                break;</br>
            }</br>
        }</br>
} </br></br>
例子1：str=中华人民共和国</br></br>
首先offset++变为1，进入1，这时dataLen=7，接着到3，读取到“中”，bufferIndex=1，接着进入6，进入push方法，此时length=0，start=offset-1=0，length++变为1，接着进入flush方法termAtt=buffer[0-1]=“中”，offsetAtt=[0-1]。第二次offset++变为2，bufferIndex=1<dataLen=7,进入3，读取到“国”，bufferIndex=2，接着流程如上。直到bufferIndex=dataLen再次进入1，由于input已经在第一次全部读取出来，此时dataLen=-1，直接返回。</br></br>
例子2：str=abc中华人民共和国</br></br>
首先offset++变为1，进入1，这时dataLen=10，接着到3，读取到“a”，bufferIndex=1，接着进入4，进入push方法， 此时length=0，start=offset-1=0，length++变为1，继续以上操作直到读取到“中”，此时bufferIndex=4，offset=4，start=0，length=3（由于此时并没有执行push方法），这时候bufferIndex和offset递减为3并且将abc返回出去，接下来流程如例子1.
