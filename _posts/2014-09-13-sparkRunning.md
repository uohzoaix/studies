---
layout: post
title: "spark之路第二天——spark启动方式"
description: "spark学习系列第二篇"
category: 
- spark
tags: []
---


###1.本地运行
上篇文章的运行方式就是在本地的运行方式，并且是单线程的，如果你想多线程运行spark，则使用如下方式（以4个线程运行）：
{% highlight objc %}
MASTER=local[4] ./spark-shell
{% endhighlight %}
###2.standalone mode
首先需要关闭hadoop，到spark的sbin目录下运行：
{% highlight objc %}
./start-master.sh
{% endhighlight %}
以上命令会在终端输出日志文件路径，tail该日志文件可看到类似spark://HOST:PORT的URL，如下：
{% highlight objc %}
14/09/13 22:21:54 INFO Master: Starting Spark master at spark://uohzoaix.local:7077
14/09/13 22:21:55 INFO MasterWebUI: Started MasterWebUI at http://localhost:8080
14/09/13 22:21:55 INFO Master: I have been elected leader! New state: ALIVE
{% endhighlight %}
表示spark的master节点已启动成功，在地址栏输入http://localhost:8080可看到spark的一些信息。日志中的spark://uohzoaix.local:7077是用来连接workers或者作为SparkContext的master参数进行传递。</br></br>
上面已经启动了master节点，这时需要启动workers节点并连接到master节点，到bin目录下运行：
{% highlight objc %}
./spark-class org.apache.spark.deploy.worker.Worker spark://IP:PORT
{% endhighlight %}
其中spark://IP:PORT就是上面日志中出现的那个URL。当出现如下信息表示启动成功：
{% highlight objc %}
14/09/13 22:31:18 INFO Worker: Starting Spark worker localhost:51259 with 4 cores, 7.0 GB RAM
14/09/13 22:31:18 INFO Worker: Spark home: /Users/uohzoaix/Downloads/spark-1.0.2
14/09/13 22:31:19 INFO WorkerWebUI: Started WorkerWebUI at http://localhost:8081
14/09/13 22:31:19 INFO Worker: Connecting to master spark://uohzoaix.local:7077...
14/09/13 22:31:19 INFO Worker: Successfully registered with master spark://uohzoaix.local:7077
{% endhighlight %}
此时刷新localhost:8080则可发现有worker已经注册到master上了。也可以通过http://localhost:8081来浏览worker的一些信息。其中7.0GB RAM是由于spark会预留1GB的内存给操作系统。</br></br>
这个时候就可以启动spark-shell了：
{% highlight objc %}
MASTER=spark://IP:PORT ./spark-shell
{% endhighlight %}
再次刷新localhost:8080可再running applications栏里出现spark shell的信息。</br></br>
此时运行如下代码：
{% highlight objc %}
val bcv = sc.broadcast(Array(1,2,3))
bcv.value
{% endhighlight %}
终端输出res0: Array[Int] = Array(1, 2, 3)。</br>
若要关闭spark集群执行：./sbin/stop-master.sh即可。
###3.脚本方式
在conf/spark-env.sh文件中添加：
{% highlight objc %}
JAVA_HOME=/usr/local/lib/jdk1.7.0_45
{% endhighlight %}
接着启动hadoop集群，通过localhost:8080可以看到集群的启动状况。
###4.YARN和Mesos
待续……</br></br>
由于目前spark多是已standalone方式运行，接下来会详细讲解该方式。</br></br>
spark第二课结束。