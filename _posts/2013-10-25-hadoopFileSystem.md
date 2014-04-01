---
layout: post
title: "hadoop FileSystem分析"
category: 
- hadoop
tags: []
---





首先来看看FileSystem这个类，在hadoop运行时hadoop需要知道你到底使用的是哪种文件系统，至此hadoop支持多种文件系统，有：LocalFileSystem，DistributedFileSystem，S3FileSystem，HftpFileSystem，InMemroryFileSystem（这几种均可通过配置core-site.xml文件来设定特定的格式），在hadoop运行开始，我们需要提供运行的输入和输出目录，而hadoop并不知道这些文件是存在哪里，到底是在本地还是在其他节点或是其他机架上，而我们往往需要配置core-site.xml文件的fs.default.name=hdfs://xx:xx（该值默认为file:///即本地模式），在作业运行时hadoop会读取该配置项并且将值构造为一个URI并取出其shema，此处为hdfs，获取到具体的schema后即可通过读取core-site.xml中的fs.${schema}.impl获取具体的文件系统格式了，源码如下：
{% highlight objc %}
/** Returns the configured filesystem implementation.*/
public static FileSystem get(Configuration conf) throws IOException {
	return get(getDefaultUri(conf), conf);
}
/** Get the default filesystem URI from a configuration.
   * @param conf the configuration to access
   * @return the uri of the default filesystem
*/
public static URI getDefaultUri(Configuration conf) {
	return URI.create(fixName(conf.get(FS_DEFAULT_NAME_KEY, "file:///")));
}
/** Returns the FileSystem for this URI's scheme and authority.  The scheme
   * of the URI determines a configuration property name,
   * <tt>fs.<i>scheme</i>.class</tt> whose value names the FileSystem class.
   * The entire URI is passed to the FileSystem instance's initialize method.
*/
public static FileSystem get(URI uri, Configuration conf) throws IOException {
	String scheme = uri.getScheme();
	String authority = uri.getAuthority();
	if (scheme == null && authority == null) {     // use default FS
		return get(conf);
	}
	if (scheme != null && authority == null) {     // no authority
		URI defaultUri = getDefaultUri(conf);
		// if scheme matches default
		if (scheme.equals(defaultUri.getScheme()) && defaultUri.getAuthority() != null) {  // & default has authority
			return get(defaultUri, conf);              // return default
		}
	}
	String disableCacheName = String.format("fs.%s.impl.disable.cache", scheme);
    	if (conf.getBoolean(disableCacheName, false)) {
      		return createFileSystem(uri, conf);
    	}
    	return CACHE.get(uri, conf);
}

private static FileSystem createFileSystem(URI uri, Configuration conf) throws IOException {
    	Class<> clazz = conf.getClass("fs." + uri.getScheme() + ".impl", null);
    	LOG.debug("Creating filesystem for " + uri);
    	if (clazz == null) {
      		throw new IOException("No FileSystem for scheme: " + uri.getScheme());
      	}
    	FileSystem fs = (FileSystem)ReflectionUtils.newInstance(clazz, conf);
    	fs.initialize(uri, conf);
    	return fs;
}
{% endhighlight %}
上面的源码清楚的表示了如何实例化文件系统，其中fs.${schema}.impl.disable.cache表示是否缓存该文件系统如果不缓存每次新作业运行都会调用createFileSystem方法类实例化文件系统，如果缓存则可从cache中取，该值默认为true。我们来看看Cache的get方法：
{% highlight objc %}
FileSystem get(URI uri, Configuration conf) throws IOException{
      	Key key = new Key(uri, conf);
      	FileSystem fs = null;
      	synchronized (this) {
        		fs = map.get(key);
      	}
      	if (fs != null) {
        		return fs;
      	}
      	fs = createFileSystem(uri, conf);
      	synchronized (this) {  // refetch the lock again
        		FileSystem oldfs = map.get(key);
        		if (oldfs != null) { // a file system is created while lock is releasing
          			fs.close(); // close the new file system
          			return oldfs;  // return the old file system
        		}
        		// now insert the new file system into the map
        		if (map.isEmpty() && !clientFinalizer.isAlive()) {
          			Runtime.getRuntime().addShutdownHook(clientFinalizer);
        		}
        		fs.key = key;
        		map.put(key, fs);
        		return fs;
      	}
}
{% endhighlight %}
代码中使用同步快来获取文件系统，如果缓存中存在则直接返回缓存中的，如果不存在则重新去获取锁防止多线程影响去创建文件系统，可是在当
{% highlight objc %}
synchronized (this) {
        fs = map.get(key);
      }
{% endhighlight %}
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