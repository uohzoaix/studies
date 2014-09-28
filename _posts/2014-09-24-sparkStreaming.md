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
####基本数据源
上面例子使用了tcp作为数据源，另外还有很多其他的基本数据源：
<li>文件系统：为了从文件系统（hdfs等）读取数据，可以按下面方式创建DStream：
{% highlight objc %}
streamingContext.fileStream\[keyClass, valueClass, inputFormatClass\](dataDirectory)
{% endhighlight %}
spark streaming会从dataDirectory文件夹读取所有文件（除该文件夹中子文件夹里的文件外），需要注意这些文件必须有相同的数据格式；这些文件必须以原子移动或重命名的方式在这个文件夹内创建；一旦这些文件被创建，那么它就不能被改变了，所以以后在这些文件里添加新的内容时这些新的内容将不会被读取。
<li>基于actor的数据流：这种方式可以通过streamingContext.actorStream(actorProps, actor-name)来创建DStream
<li>RDD队列：为了能够测试spark streaming程序，可以使用streamingContext.queueStream(queueOfRDDs)方式创建DStream
####高级数据源
高级数据源包括之前提到的kafka，flume，twitter等，比如要使用twitter的数据流创建DStream，需要做以下事情：  
1.将spark-streaming-twitter_2.10添加到sbt或maven中  
2.导入TwitterUtils类并使用TwitterUtils.createStream(sparkStreamingContext)方法来创建DStream  
3.生成jar包部署改程序
####DStream转换
DStream支持的常见转换方式有以下几种：
<table>
<thead>
<tr class="header">
<th align="left">转换方法</th>
<th align="left">意义</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">map(func)</td>
<td align="left">将源DStream中的每一个元素经过func函数的处理并返回一个新的DStream</td>
</tr>
<tr class="even">
<td align="left">flatMap(func)</td>
<td align="left">类似map，但每个元素会返回0或多个元素</td>
</tr>
<tr class="odd">
<td align="left">filter(func)</td>
<td align="left">返回func函数返回true的元素组成的DStream</td>
</tr>
<tr class="even">
<td align="left">repartition(numPartitions)</td>
<td align="left">将DStream重新进行分区</td>
</tr>
<tr class="odd">
<td align="left">union(otherStream)</td>
<td align="left">返回包含源DStream和otherDStream的新DStream</td>
</tr>
<tr class="even">
<td align="left">count()</td>
<td align="left">返回源DStream中每个RDD数据中的元素个数</td>
</tr>
<tr class="odd">
<td align="left">reduce(func)</td>
<td align="left">使用func函数（该函数接受2个参数并返回一个结果）计算源DStream中的每个RDD数据的元素</td>
</tr>
<tr class="even">
<td align="left">countByValue()</td>
<td align="left">返回DStream中每个元素的出现次数</td>
</tr>
<tr class="odd">
<td align="left">reduceByKey(func,[numTasks])</td>
<td align="left">DStream格式为(K,V)格式，通过func函数的处理返回(K,V)格式数据。在本地模式numTasks的默认值为2，在集群模式下默认值为spark.default.parallelism配置的值</td>
</tr>
<tr class="even">
<td align="left">join(otherStream,[numTasks])</td>
<td align="left">由(K,V)和(K,W)返回(K,(V,W))</td>
</tr>
<tr class="odd">
<td align="left">cogroup(otherStream,[numTasks])</td>
<td align="left">由(K,V)和(K,W)返回(K,Seq[V],Seq[W])</td>
</tr>
<tr class="even">
<td align="left">transform(func)</td>
<td align="left">DStream中的每个RDD通过func函数的处理转变为新的RDD</td>
</tr>
<tr class="odd">
<td align="left">updateStateByKey(func)</td>
<td align="left">改变DStream中key的状态</td>
</tr>
</tbody>
</table>
下面详细讲下最后两个装换方法：  
#####UpdateStateByKey
这个方法允许获取数据新的状态，需要以下两个步骤：  
1.首先需要定义数据的初始状态：状态可以为多种数据类型  
2.定义状态改变函数：该函数利用数据之前的状态和stream中新的值返回新的状态  
下面的例子中状态改变函数将计数作为状态，其中pairs DStream是（word,1）格式：
{% highlight objc %}
def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
    val newCount = ...  // add the new values with the previous running count to get the new count
    Some(newCount)
}
val runningCounts = pairs.updateStateByKey[Int](updateFunction _)
{% endhighlight %}
上面的例子中每个word都会传递到状态改变函数中处理，传递的newValues就是一系列的1，所以上述例子也可以用来计算每个word的次数。
#####转换操作
转换操作允许DStream执行RDD-RDD函数，转换操作可以用于任何RDD上。例如当你想实时将输入数据与之前计算好的数据进行合并执行filter操作，可以：
{% highlight objc %}
val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...) // RDD containing spam information
val cleanedDStream = wordCounts.transform(rdd => {
  rdd.join(spamInfoRDD).filter(...) // join data stream with spam information to do data cleaning
  ...
})
{% endhighlight %}
#####窗口操作
spark streaming也提供了窗口计算，它允许在滑动窗口上的数据进行转换操作。下图介绍了滑动窗口：
![Alt text](https://spark.apache.org/docs/latest/img/streaming-dstream-window.png)
上图显示，每个时刻窗口会在源DStream上滑动，在这个窗口里的RDD会被合并和进行相应的操作生成窗口DStream。在上图的情形下，相关操作会在最后3个时间单元的数据上进行，并且滑动2个时间单元，所以窗口操作需要两个参数：
<li>窗口长度-窗口的持续时间（上图中的3）
<li>滑动间隔时间-窗口间隔多久进行一次操作（上图中的2）
这两个参数必须是源DStream中数据单元（上图为1）的倍数（上图为1）。  
下面是窗口操作的一个例子，假设需要在最后的30秒每10秒进行一次wordCount，代码如下（30秒作为窗口持续时间参数，10秒作为窗口操作间隔时间）：
{% highlight objc %}
// Reduce last 30 seconds of data, every 10 seconds
val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))
{% endhighlight %}
例子中用到了reduceByKeyAndWindow操作，其他的一些常见窗口操作如下：
<table>
<thead>
<tr class="header">
<th align="left">转换方法</th>
<th align="left">意义</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">window(windowLength,slideInterval)</td>
<td align="left">返回进过窗口批处理的DStream</td>
</tr>
<tr class="even">
<td align="left">countByWindow(windowLength,slideInterval)</td>
<td align="left">计算流数据中每个滑动窗口中的元素数量</td>
</tr>
<tr class="odd">
<td align="left">reduceByWindow(func,windowLength,slideInterval)</td>
<td align="left">使用func在每个窗口中计算元素的总数</td>
</tr>
<tr class="even">
<td align="left">reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks])</td>
<td align="left">每个滑动窗口中计算(K,V)中每个key的总数，返回格式为(K,V)</td>
</tr>
<tr class="odd">
<td align="left">reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval, [numTasks])</td>
<td align="left">上述方法的一个高效实现，基于上个窗口的数据进行计算</td>
</tr>
<tr class="even">
<td align="left">countByValueAndWindow(windowLength, slideInterval, [numTasks])</td>
<td align="left">计算(K,V)返回(K,Long)</td>
</tr>
</tbody>
</table>
#####DStream输出操作
输出操作允许DStream的数据放入外置系统如数据库或文件系统，由于输出操作需要将最终数据放入外置系统中，所以执行输出操作的时候会开始真正执行之前的转换操作。下表是一些输出操作：
<table>
<thead>
<tr class="header">
<th align="left">输出操作</th>
<th align="left">意义</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">print()</td>
<td align="left">输出DStream中每批数据的前十个元素，常用于调试</td>
</tr>
<tr class="even">
<td align="left">saveAsObjectFiles(prefix,[suffix])</td>
<td align="left">将DStream中的内容保存为序列化的SequenceFile，每批数据产生的文件名为：prefix-TIME_IN_MS[.suffix]</td>
</tr>
<tr class="odd">
<td align="left">saveAsHadoopFiles(prefix,[suffix])</td>
<td align="left">将DStream中的内容保存为hadoop的text file，每批数据产生的文件名为：prefix-TIME_IN_MS[.suffix]</td>
</tr>
<tr class="even">
<td align="left">foreachRDD(func)</td>
<td align="left">最常见的输出操作，流中的RDD都会通过func函数处理，这个func的作用是将RDD中的数据放入外置系统如文件、数据库</td>
</tr>
</tbody>
</table>
#####foreachRDD的设计模式
dstream.foreachRDD是非常有用的操作，但使用它需要避免如下的几个错误：
<li>将数据写入外置系统中需要创建一个连接对象（如：tcp连接）并使用这个连接将数据发送到远程系统。为达到这个目的，开发人员可能会尝试在spark的driver中创建这个连接，并在spark的worker中使用这个连接来保存RDD中的数据，代码如下：
{% highlight objc %}
dstream.foreachRDD(rdd => {
      val connection = createNewConnection()  // executed at the driver
      rdd.foreach(record => {
          connection.send(record) // executed at the worker
      })
  })
{% endhighlight %}
这种方式是不正确的，因为这需要将这个连接序列化以便将它从driver传送到worker中，这样的连接对象是不能在机器之间进行传送的。上面会发生序列化错误（连接对象不能被序列化）和初始化错误（连接对象需要在worker端进行初始化）等错误。正确的做法是在worker端进行创建连接对象。  
<li>但这又会带来另外一个问题-每个数据记录都会创建一个新的连接，如：
{% highlight objc %}
dstream.foreachRDD(rdd => {
      rdd.foreach(record => {
          val connection = createNewConnection()
          connection.send(record)
          connection.close()
      })
  })
{% endhighlight %}
创建连接对象是需要时间和资源的，所以这种方式会明显的增加不必要的资源消耗和降低机器的吞吐能力，一个更好的解决方案是使用rdd.foreachPartition-创建一个单独的连接对象并且通过这个连接发送一个RDD中所有的数据：
{% highlight objc %}
dstream.foreachRDD(rdd => {
      rdd.foreachPartition(partitionOfRecords => {
          val connection = createNewConnection()
          partitionOfRecords.foreach(record => connection.send(record))
          connection.close()
      })
  })
{% endhighlight %}
但这种方法在多个RDD分区中还是会创建多个连接。
<li>最后，下面方法是使用静态的连接池来对连接进行重用：
{% highlight objc %}
dstream.foreachRDD(rdd => {
      rdd.foreachPartition(partitionOfRecords => {
          // ConnectionPool is a static, lazily initialized pool of connections
          val connection = ConnectionPool.getConnection()
          partitionOfRecords.foreach(record => connection.send(record))
          ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
      })
  })
{% endhighlight %}
注意：池中的连接应该是根据需要记性懒式创建并且在一段时间之内不再使用需要将其释放，这种方式是最高效的。
######其它需要注意的
<li>DStream是懒式执行的，所以DStream中必须要有RDD的action操作，这样接收到的数据才会被执行，否则接收到的数据会被丢弃
<li默认，输出操作是一个时刻执行一次操作的，并且它们的执行顺序是按应用程序中定义它们的顺序的