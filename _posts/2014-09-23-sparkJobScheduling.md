---
layout: post
title: "spark之路第五六——任务调度"
description: "spark学习系列第六篇"
category: 
- spark
tags: []
---


spark中有多种资源调度方式。
###1.应用间调度
运行中的集群里，每个spark应用都拥有独立的executor JVM用来运行任务和存储数据。如果多个用户需要共享这个集群，那么集群管理者会使用不同的方式来管理资源分配情况。</br>
最简单的一种方式是将资源进行静态分区，通过这种方式，每个应用会得到它能使用的最大的资源量并且在它的整个运行过程中一直都会被持有。spark的standalone和YARN，粗粒度mesos模式都是使用的这种方式。具体的资源分配方式见下：</br>
<li>standalone模式：在默认情况下，提交到standalone模式的应用会以一种FIFO的顺序被一次执行，并且每个应用都会尝试去使用所有可用的节点。可以通过设置spark.cores.max或spark.deploy.defaultCores属性来设置每个应用能使用的节点数。除此之外设置spark.executor.memory可以设置每个应用的内存使用量
<li>mesos：为了使用静态分区，可以设置spark.mesos.coarse=true，也可以像standalone模式时设置spark.cores.max和spark.executor.memory
<li>YARN：YARN模式下的--num-executor选项可以设置将会有多少个executors分配给集群，并且--executor-memory和--executor-cores设置每个executor使用的资源</br>
mesos模式还有另外一种方式能够动态分配CPU，使用这种方式spark的应用也可以拥有固定并且独立的内存分配，但是当其他应用程序也有任务运行在这台机器上时，那么这种方式就很有用了。但是这种方式会增加应用的等待时间，因为当当前节点已经有任务在运行时，那么某个应用程序如果想增加cores那么就会等待较长的一段时间。可以使用mesos://URL（不需要设置spark.mesos.coarse=true）就可以使用这种方式。</br>
需要注意的是这两种方式都不能共享应用程序之间的内存。
###2.应用程序内调度
在一个指定的应用程序（一个SparkContext实例）里面，多个从不同线程提交的job可以并行运行。本节的job指的是某个spark的action（如：save，collect）和任何计算这些action的任务。spark的调度器是线程安全，所以它可以同时响应多个请求。</br>
默认情况spark的调度器是按FIFO顺序执行job的，每个job会被分成多个步骤（如：map，reduce），第一个job会优先获得资源，接着第二个job，以此类推运行。如果队列中的处于head的job不需要使用整个集群的资源，后面的job会立即得到运行，当队列中处于head的job很大，那么后续的job就会明显的被延迟执行。</br>
在spark0.8以后，spark提供了公平调度方式。通过使用公平调度方式，spark使用循环的方式为job指派任务，所以每个job会得到相近的资源。这意味着当一个长时间的job正在运行时，短时间的job也能立即得到资源被执行。</br>
通过以下代码就可以使用公平调度器：
{% highlight objc %}
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.set("spark.scheduler.mode", "FAIR")
val sc = new SparkContext(conf)
{% endhighlight %}
####公平调度器池
公平调度器同时也支持将job归并入池中，这时可以对每个池进行不同的设置。在这种情况下就可以对那些重要的job所在的池设置高优先级使其能得到优先执行。</br>
在没有任何的干预下，最新提交的job将会放入默认池中，但SparkContext通过设置spark.scheduler.pool属性后，那么在这个线程中提交的job（比如RDD.save,count,collect等）都会被放入该属性设置的池中：
{% highlight objc %}
// Assuming sc is your SparkContext variable
sc.setLocalProperty("spark.scheduler.pool", "pool1")
{% endhighlight %}
当需要清楚当前线程的池则简单将值设置为null即可。
####池的默认行为
默认情况下，每个池得到的资源都是一样的（默认池中的job也会得到一样的资源），除默认池外其他池中的job还是按FIFO形式执行。
####池的相关属性
<li>schedulingMode：可以设置为FIFO和FAIR，指定池中job的资源分配方式
<li>weight：设置池得到资源的权重值。默认每个池的权重值为1，当将某个池的权重设置为2，那么这个池分配的资源是其他池的两倍。
<li>minShare：除了weight属性外，每个池还可以指定分配的最小资源量。公平调度器会首先为每个池分配该值的资源，然后会通过设置的weight属性为池重新分配分配。该值默认为0</br>
这些属性是通过一个xml文件进行设置，如conf/fairscheduler.xml.template文件，然后以如下方式将文件赋给SparkConf：
{% highlight objc %}
conf.set("spark.scheduler.allocation.file", "/path/to/file")
{% endhighlight %}
xml文件内容如下：
{% highlight objc %}
<?xml version="1.0"?>
<allocations>
  <pool name="production">
    <schedulingMode>FAIR</schedulingMode>
    <weight>1</weight>
    <minShare>2</minShare>
  </pool>
  <pool name="test">
    <schedulingMode>FIFO</schedulingMode>
    <weight>2</weight>
    <minShare>3</minShare>
  </pool>
</allocations>
{% endhighlight %}
注意：在xml文件中未配置的pool将会使用spark的默认调度方式和属性（schedulerMode=FIFO，weight=1，minShare=0）