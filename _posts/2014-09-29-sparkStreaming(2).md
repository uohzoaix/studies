---
layout: post
title: "spark之路第八课——streaming(2)"
description: "spark学习系列第八篇"
category: 
- spark
tags: []
---


接上篇：  
###缓存/持久化
和RDD一样，DStream中的数据也允许被持久化到内存中，使用persist()方法就会自动将DStream中的RDD持久化到内存。这在数据需要被多次使用的时候是非常有用的，一些基于窗口的操作如reduceByWindow和reduceByKeyAndWindow和一些基于状态的操作如updateStateByKey，这些操作默认情况下是自动会被持久化的，不需要手动调用persist()方法。  
从kafka，flume，sockets等数据源接收到的数据默认的持久化级别是将数据进行拷贝放置在两个节点中，这样就可以达到容错的目的了。
###检查点
有状态的操作都是在多批数据上进行操作的，例如那些基于窗口的操作和updateStateByKey。由于有状态的操作会依赖之前的数据，所以它们会持续的累积元数据，为了清理这些元数据，streaming支持定时checkpoint并将中间数据保存到HDFS中。但是checkpoint会增加保存到HDFS的代价，这会使进程间通信的时间更长。因此checkpoint的间隔的设置是很重要的，当设置为1秒，那么为每块数据进行checkpoint显然会减少吞吐量，相反的，当间隔时间较长时，那么每个任务的数据量就会急剧增加。一般情况下，应该将checkpoint的间隔时间设置为sliding间隔时间的5-10倍。  
为了启用checkpoint，必须提供HDFS路径用于保存RDD数据：
{% highlight objc %}
ssc.checkpoint(hdfsPath) // assuming ssc is the StreamingContext or JavaStreamingContext
{% endhighlight %}
checkpoint的间隔时间设置如下：
{% highlight objc %}
dstream.checkpoint(checkpointInterval)
{% endhighlight %}
###部署应用程序
spark streaming应用的部署和其他的spark应用是类似的，当使用kafka，flume等数据源时需要将这些额外的包及其依赖包都添加进去。  
如果一个正在运行的streaming应用需要升级（即添加新的代码），有两种可能的机制：
<li>当新的应用已经启动并且准备开始工作，那么旧的应用可以关闭掉，因为数据源支持将同样的数据发送到两个不同的目的地
<li>如果旧的应用程序使用stop()方法关闭时，那么会确保收到的数据在彻底关闭之前也能被正确的处理，处理完成之后新的应用就可以启动了。这样的功能只有具有源端缓冲功能的数据源（如kafka，flume）才具备，这种缓冲功能会将没有被接收或处理的数据放到缓冲区，待新的应用启动后，缓冲区里的数据就能被消费了
###监控应用程序
当StreamingContext被使用时，spark的webUI会显示出Streaming标签，这个标签中会显示正在运行的接收者的统计结果（如接收到的记录，接收错误等信息）和数据块处理信息（如处理时间，队列延迟时间等），这些信息可以用来监控streaming应用的运行进度。  
下面两个信息是非常重要的-处理时间和调度延迟。处理时间是处理每个数据块的用时，调度延迟是当前数据块等待被处理的时间。如果数据块处理时间始终都大于数据块等待被处理的时间，那么可以说明这个系统不能及时的处理数据块，在这种情况下可以适当减少数据块的处理时间。  
spark streaming程序的运行进度也可以通过SteamingListener接口进行监控，它可以得到数据接受者的状态和处理时间，但是该接口是正在开发的接口，需要谨慎使用。
###性能优化
如果想要spark streaming获得更好的性能那么需要一些优化手段，这节主要是讲如何通过参数和配置选项来提升应用程序的性能，也就是：  
1.减少每批数据的处理时间以提升集群资源的使用率  
2.合理设置每批数据的大小以便能被尽快的处理
####减少数据的处理时间
在spark中有一些选项是可以使数据的处理时间减少的：
#####数据接收并行度
