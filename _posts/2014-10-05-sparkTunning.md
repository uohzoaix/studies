---
layout: post
title: "spark之路第九课——spark性能调优"
description: "spark学习系列第九篇"
category: 
- spark
tags: []
---

由于spark的很多计算都是基于内存的，所以集群中的任何资源都会成为spark程序的瓶颈：CPU，网络带宽或内存。如果内存足够装下数据的话，那么网络带宽会成为最主要的瓶颈，但是很多时候还是需要做一些性能调优的工作，如使用序列化格式进行存储RDD数据。这篇文章主要讲两点：数据序列化（对网络性能是至关重要的并且能够减少内存使用）和内存优化。
###数据序列化
序列化在分布式应用中扮演着很重要的角色，如果将对象序列化成字节或从字节反序列化成对象这个过程很慢的话，那么会大大降低计算的速度。所以，数据格式是优化的首要任务。spark提供了两种序列化方式：
<li>java序列化：spark默认使用java的ObjectOutputStream来序列化对象，该方式虽然可扩展但速度比较低下
<li>kryo序列化：kryo方式速度比java提供的方式会更快，但不支持所有的Serializable类并且需要将需要使用的类注册到程序当中  
可以通过SparkConf来进行序列化方式的变更：conf.set("spark.serializer","org.apache.spark.serializer.KryoSerializer")，该选项会作用于worker节点之间数据的shuffle过程和将RDD数据序列化到磁盘过程。由于需要自定义注册，所以kryo方式不是默认的序列化方式，在网络集中的应用中推荐使用kryo方式。  
使用kryo注册自定义类时，需要创建一个继承与org.apache.spark.serializer.KryoRegistrator类的公共类并且设置spark.kryo.registrator属性值为该公共类：
{% highlight objc %}
import com.esotericsoftware.kryo.Kryo
import org.apache.spark.serializer.KryoRegistrator
class MyRegistrator extends KryoRegistrator {
  override def registerClasses(kryo: Kryo) {
    kryo.register(classOf[MyClass1])
    kryo.register(classOf[MyClass2])
  }
}
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
conf.set("spark.kryo.registrator", "mypackage.MyRegistrator")
val sc = new SparkContext(conf)
{% endhighlight %}
如果需要序列化的对象较大时，则需要将spark.kryoserializer.buffer.mb的值设大，默认为2。  
如果没有将自定义类注册到kryo中，那么kryo也会工作，但会将类名存储起来，将会是一个很大的浪费。
###内存优化
内存优化中有三个建议：对象使用的内存量，访问这些对象的开销，垃圾回收器的开支。  
默认java对象的访问速度是很快的，但这些对象通常会花费比字段中的原生数据大2-5被的空间，主要原因如下：
<li>每个不同的java对象有一个对象头，它大概有16字节，其中内容包括指向类的指针信息。对于只有一个int类型的字段的对象来说，在内存中会花费比字段数据更大的空间
<li>java的string大约需要花费40个字节来存储原生的数据（包括char数组和它的长度），并且使用两个字节来存储每个字符，因为string内部使用的编码是utf-8的，因此一个10字符的string会消耗60字节的内存
<li>常见的集合类如HashMap和LinkedList，它们中的每一个entry都含有一个包装类（如Map.Entry），这种对象不仅含有对象头，还有指向下个对象的指针
<li>原生类型的集合里存储的都是原生类型的装箱对象（如Integer）  
这节主要内容是如何决定对象的内存使用-改变数据结构或以序列化格式存储数据。
####计算内存消耗
计算数据集将消耗的内存量的最好方式是创建一个RDD并将其放入cache中，接下来观察SparkContext的日志，这些日志会显示每个分片消耗了多少内存，这样就可以知道整个数据集消耗了多少内存了。日志信息如下：
{% highlight objc %}
INFO BlockManagerMasterActor: Added rdd_0_1 in memory on mbk.local:50311 (size: 717.5 KB, free: 332.3 MB)
{% endhighlight %}
上述信息意味着RDD 0的分片1消耗了717.5kb的内存。
####数据结构调优
减少内存消耗的首要任务是避免java对象中添加基于指针的数据结构和封装对象，有如下几种方式避免：  
1.使用对象数组和原生类型代替标准的java或scala集合类，fastutil类库为原生类型提供了和java标准库类似的集合类。  
2.尽量避免带有很多小对象和指针的嵌套结构。  
3.使用数值型ID或者枚举类来代替string作为主键。  
4.如果机器内存少于32GB，添加-XX:+UseCompressedOops的JVM选项使指针由8字节改为4字节，可以将这些JVM选项配置在spark-env.sh。
####序列化RDD存储
当对象仍然很大，那么可以讲对象序列化后进行存储以减少内存使用，序列化后spark会将每个RDD分片作为一个很大的字节数组进行存储，这种方式唯一的缺点是访问这些序列化数据会变得很慢，因为必须反序列化每个对象。
####垃圾回收器调优
当java需要腾出空间给新的对象时，需要跟踪所有的java对象来找出没用的对象，由于垃圾回收器的代价是与java对象的数量成比例的，所以使用较少对象的数据结构（如使用ints数组代替LinkedList）会减少相应的代价。一个更好的方法是讲对象序列化，这时每个RDD分片就只有一个对象了（字节数组）。  
#####衡量GC影响
GC调优的第一步是收集GC发生的频率和每次GC消耗的时间等统计信息，这个工作可以通过添加-verbose:gc -XX:+PrintGCDetails -XX:PrintGCTimeStamps属性到SPARK_JAVA_OPTS环境变量中。这样当spark任务运行时，worker节点的日志就会显示GC每次发生的信息。
#####缓存大小调优
GC当中一个重要的配置参数是缓存RDD需要多少内存，默认，spark使用配置的spark.executor.memory值的60%的内存来缓存RDD，意味着40%的内存可用于任务执行时对象的创建。  
当发现任务执行速度减慢并且发现GC频繁工作或者内存溢出，需要降低该百分比以减少内存消耗，比如需要降低为50%，则使用conf.set("spark.storage.memoryFraction","0.5")。结合序列化缓存，使用更小的缓存大小对减轻GC问题是很有作用的。
#####高级GC调优
为了更多的GC调优，需要对JVM中的内存管理有更多的了解：
<li>java堆空间是被分成年轻代和老年代两个区域，年轻代存放生命周期较短的对象而老年代存放生命周期长的对象
<li>年轻代又被分成Eden，Survivor1，Survivor2三个区域
<li>GC处理过程的简单描述：当Eden满了，一个较小的GC会在Eden区运行并且Eden中活的对象和Survivor1中的对象会被拷到Survivor2中，如果一个对象很老或者Survivor2满了，那么该对象会被移动到老年代中，最后当老年代即将满了，一个full GC会被调用  
spark中GC调优的目标是保证只有生命周期很长的RDD被存储在老年代并且新生代能有更多的空间来存储生命周期短的对象。这会避免full GC收集任务执行过程中产生的临时对象。下面是一些有用的步骤：
<li>检查是否有过多的垃圾回收器，如果一个full GC在任务完成之前被调用多次，那么意味着没有足够的内存给正在执行的任务
<li>在打印的GC统计信息中，如果老年代将要满了，这时候就要减少用于缓存的内存了，可通过spark.storage.memoryFraction属性进行设置，缓存更少的对象好于让任务执行速度的减慢。
<li>如果有过多的小GC，那么分配多一点的内存给Eden会有帮助。可以设置Eden的大小为每个任务需要的内存量大小。如果Eden大小为E，那么可以使用-Xmn=4/3\*E设置年轻代的大小。
<li>如果某个任务时从HDFS中读取数据，那么任务使用的内存可以通过从HDFS读取的数据块的大小进行估略。一般为压缩的块大小为块大小的2-3倍，所以如果希望有3或4个任务在工作空间中并且HDFS的块大小为64M，那么Eden大小可以设置为4\*3\*64M。
<li>当每次改变了设置观察GC发生的频率和时间
###另外需要考虑的事情
####并行度
如果每个操作的并行度并不高的话那么集群将得不到合理的利用，spark自动会基于每个文件的大小设置map任务的数量对于像groupByKey和reduceByKey等reduce操作，spark使用父RDD最大的分片数进行设置。可以通过设置spark.default.parallelism来改变默认的并行度，通常情况，为每个CPU设置2-3个任务即可。
####reduce任务的内存使用
有时可能会遇到OutOfMemoryError不是因为内存不够，而是因为某个任务的工作集（比如groupByKey中的某个reduce任务）较大。spark的shuffle操作（sortByKey，groupByKey，reduceByKey，join等）会在每个任务中建立一个hash表来处理grouping。最简单的方式是增加并行度来使每个任务的输入集更小。
####广播大变量
使用SparkContext支持的广播函数可以很大程度的减少序列化任务的大小和部署任务的开销，如果任务使用任何大对象，那么可以将这些对象转换为广播变量，spark会打印master机器的序列化大小，所以可以观察哪个任务较大，通常大于20KB的任务是值得优化的。