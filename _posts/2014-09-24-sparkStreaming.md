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
