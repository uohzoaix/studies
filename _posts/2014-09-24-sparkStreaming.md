---
layout: post
title: "spark之路第七课——streaming"
description: "spark学习系列第七篇"
category: 
- spark
tags: []
---

spark streaming是spark核心api的一部分，它具有可扩展，高吞吐，容错等特点，所以在处理流失数据时是非常有用的。它可以从kafka，flume，twitter，zeromq，kinesis或者tcp中读取数据经过一系列的高阶函数如map，reduce，join和window处理，最后将处理完成的数据放入文件系统，数据库或者Dashboard。</br>
spark streaming中的DStream表示一个连续的数据流，DStream可以通过kafka等数据源创建而来或者通过在其他的DStream上使用高阶操作而来，其实DStream也就是一系列的RDD。</br>
###例子
下面的例子是计算从tcp接收到的数据中的单词个数。</br>
首先需要导入streaming需要的类，StreamingContext是使用streaming的入口，下面创建了一个2个线程和每隔1秒进行批处理的StreamingContext类：
{% highlight objc %}
import org.apache.spark.\_
import org.apache.spark.streaming.\_
import org.apache.spark.streaming.StreamingContext.\_
// Create a local StreamingContext with two working thread and batch interval of 1 second
val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))
{% endhighlight %}
接下来既可以通过StreamingContext来创建DStream（也就是监听tcp进行数据接收）：
{% highlight objc %}
// Create a DStream that will connect to hostname:port, like localhost:9999
val lines = ssc.socketTextStream("localhost", 9999)
{% endhighlight %}
接下来将接收到的每行数据按空格分隔：
{% highlight objc %}
// Split each line into words
val words = lines.flatMap(\_.split(" "))
{% endhighlight %}
上面使用flatMap创建了一个新的DStream，接下来就可以统计每个单词的个数了：
{% highlight objc %}
import org.apache.spark.streaming.StreamingContext.\_
// Count each word in each batch
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(\_ + \_)
// Print the first ten elements of each RDD generated in this DStream to the console
wordCounts.print()
{% endhighlight %}
上面代码首先将每个单词变成（word,1）形式，然后通过reduceByKey将word相等的pair的个数相加。最后打印结果（这个结果也只是每秒（通过最开始程序指定）接收到的数据的统计结果）。</br>
由于spark是懒加载的，所以上面的程序并不会真正进行计算，待执行到下面代码后程序才会从头开始处理程序：
{% highlight objc %}
ssc.start()             // Start the computation
ssc.awaitTermination()  // Wait for the computation to terminate
{% endhighlight %}
注意：  
<li>当程序调用了streamingContext.start()方法之后，新的计算逻辑就不能添加进来
<li>当程序调用了streamingContext.stop()方法后，那么这个程序里面的计算逻辑就不能被重新使用
<li>在同一时间内一个JVM里只会存在一个StreamingContext
<li>因为初始化StreamingContext时也会初始化一个SparkContext，而stop()方法在停止streamingContext同时也会停止SparkContext，如果只想停止StreamingContext，则可以将stop()方法的stopSparkContext参数设为false
<li>SparkContex可以被用来创建多个StreamingContext，只要其他的StreamingContext都停止了
###DStream
DStream在spark中表示一系列的数据流，其中的内容就是一个个的RDD数据集，在DStream上的操作会被转化为操作其内部的RDD。
####Input DStream
input DStream是从各种数据源接收到的流，每个input DStream都有一个receiver对象用来从数据源接收数据并且将数据存放在内存中。所以每个input DStream只会接收一个单独的数据流，但可以创建多个input DStream以便能并行的接收到多个数据流。  
接收器是作为一个长时间运行的任务运行在spark的worker或executor中，因此一个接收器会占用一个cpu，所以spark streaming应用需要分配足够的cpu来处理接收到的数据。  
注意：
<li>当分配给spark streaming应用的cpu少于或等于input DSteam数或接收器数，那么该应用就会将所有cpu用来接收数据，这样接收到的数据就不能被处理
<li>当在本地运行并且master URL设置为local，那么只有一个cpu用来运行任务，这是很不高效的做法，因为这个cpu会被数据接收器占用，这时就没有cpu用来处理数据了
