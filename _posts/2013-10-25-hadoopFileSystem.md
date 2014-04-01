---
layout: post
title: "hadoop FileSystem分析"
category: 
- hadoop
tags: []
---





首先来看看FileSystem这个类，在hadoop运行时hadoop需要知道你到底使用的是哪种文件系统，至此hadoop支持多种文件系统，有：LocalFileSystem，DistributedFileSystem，S3FileSystem，HftpFileSystem，InMemroryFileSystem（这几种均可通过配置core-site.xml文件来设定特定的格式），在hadoop运行开始，我们需要提供运行的输入和输出目录，而hadoop并不知道这些文件是存在哪里，到底是在本地还是在其他节点或是其他机架上，而我们往往需要配置core-site.xml文件的fs.default.name=hdfs://xx:xx（该值默认为file:///即本地模式），在作业运行时hadoop会读取该配置项并且将值构造为一个URI并取出其shema，此处为hdfs，获取到具体的schema后即可通过读取core-site.xml中的fs.${schema}.impl获取具体的文件系统格式了，源码如下：

锁释放的时候线程有可能会创建了文件系统，所以在获得锁之后需要FileSystem oldfs = map.get(key);检查是否已经实例化过了文件系统。</br>
在每次作业开始获取文件系统之前hadoop需要将之前作业实例化出来的各种文件系统全部close掉也释放资源，代码如下：
{% highlight objc %}
private static class ClientFinalizer extends Thread {
    	public synchronized void run() {
      		try {
        			FileSystem.closeAll();
      		} catch (IOException e) {
        			LOG.info("FileSystem.closeAll() threw an exception:\n" + e);
     		}
   	}
}
private static final ClientFinalizer clientFinalizer = new ClientFinalizer();
{% endhighlight %}
这个ClientFinalizer类在实例化FileSystem的时候即会初始化并启动closeAll的线程，该线程的主要工作是讲cache里的各个文件系统删除并close：
{% highlight objc %}
synchronized void closeAll() throws IOException {
      	List<IOException> exceptions = new ArrayList<IOException>();
      	for(; !map.isEmpty(); ) {
        		Map.Entry<Key, FileSystem> e = map.entrySet().iterator().next();
        		final Key key = e.getKey();
        		final FileSystem fs = e.getValue();
        		//remove from cache
        		remove(key, fs);
        		if (fs != null) {
          			try {
            			fs.close();
         			}
          			catch(IOException ioe) {
            			exceptions.add(ioe);
          			}
       		}
      	}
      	if (!exceptions.isEmpty()) {
        		throw MultipleIOException.createIOException(exceptions);
      	}
}
{% endhighlight %}
该类的其他方法均为一些文件意义上的操作，如创建文件，删除文件，修改文件名，过滤文件，获取文件状态信息，移动文件等，这些操作均可通过hadoop的文件命令实现。