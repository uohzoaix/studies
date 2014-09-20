---
layout: post
title: "spark之路第五课——配置spark"
description: "spark学习系列第五篇"
category: 
- spark
tags: []
---



spark提供了三种方式进行相关的一些属性配置：
###1.Spark properties
这种方式就是在程序中进行属性的设置，将属性传递给SparkConf类即可：
{% highlight objc %}
val conf = new SparkConf()
             .setMaster("local")
             .setAppName("CountingSheep")
             .set("spark.executor.memory", "1g")
val sc = new SparkContext(conf)
{% endhighlight %}
从代码中可以看出它既支持常见属性（masterURL,appName）的设置还支持key-value形式。
####动态加载
很多时候你不愿意以这种硬编码的方式来设置属性，你可以通过无参的SparkConf的构造方法来构造SparkConf类，在运行时使用如下方式启动：
{% highlight objc %}
./bin/spark-submit --name "My app" --master local[4] --conf spark.shuffle.spill=false 
  --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" myApp.jar
{% endhighlight %}
上述命令也会读取conf/spark-defaults.conf文件加载属性。
####spark支持的属性
可以通过http://<driver>:4040页面上的Environment标签查看所有已经正确设置的属性。</br>
<table>
<thead>
<tr class="header">
<th align="left">属性名</th>
<th align="left">默认值</th>
<th align="left">注释</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">spark.app.name</td>
<td align="left">(none)</td>
<td align="left">任务名称</td>
</tr>
<tr class="even">
<td align="left">spark.master</td>
<td align="left">(none)</td>
<td align="left">集群地址</td>
</tr>
<tr class="odd">
<td align="left">spark.executor.memory</td>
<td align="left">512m</td>
<td align="left">每个执行器进程所用内存大小</td>
</tr>
<tr class="even">
<td align="left">spark.serializer</td>
<td align="left">org.apache.spark.serializer.JavaSerializer</td>
<td align="left">对象序列化所用的类，默认的JavaSerializer性能太差，推荐使用org.apache.spark.serializer.KryoSerializer，你也可以通过集成org.apache.spark.Serializer来实现自己的序列化器</td>
</tr>
<tr class="odd">
<td align="left">spark.kryo.registrator</td>
<td align="left">(none)</td>
<td align="left">当使用了KryoSerializer，可以设置该值为KryoRegistrator将自定义类注册到Kryo</td>
</tr>
<tr class="even">
<td align="left">spark.local.dir</td>
<td align="left">/tmp</td>
<td align="left">输出文件和RDD存储的目录，可以逗号分隔指定多个目录</td>
</tr>
<tr class="odd">
<td align="left">spark.logConf</td>
<td align="left">false</td>
<td align="left">指定日志级别为INFO</td>
</tr>
<tr class="even">
<td align="left">spark.executor.extraJavaOptions</td>
<td align="left">(none)</td>
<td align="left">JVM选项，不能以这种方式设置spark属性和使用内存大小</td>
</tr>
<tr class="odd">
<td align="left">spark.files.userClassPathFirst</td>
<td align="left">false</td>
<td align="left">是否使用户添加的jar包优先于spark自身的jar包</td>
</tr>
</tbody>
</table>
#####shuffle过程使用的属性
属性名|默认值|注释
:---------------|:---------------|:---------------
spark.shuffle.consolidateFiles|false|如果设置为true，那么shuffle过程产生的中间文件会被整合到一起，这会提高reduce任务的效率。当使用ext4或xfs文件系统时建议设置为true。但如果文件系统是ext3形式的，该选项会恶化机器性能，特别是CPU核数大于8时
spark.shuffle.spill|true|设置为true则会限制在reduce阶段的内存使用量，超出部分会写到硬盘中，超出的阀值通过spark.shuffle.memoryFraction指定
spark.shuffle.spill.compress|true|是否压缩shuffle期间溢出的数据。通过spark.io.compression.codec设置压缩方式
spark.shuffle.memoryFraction|0.2|如果spark.shuffle.spill设置为true，那么shuffle期间内存使用最大为总内存*该值，超出部分会写到硬盘，如果经常会溢出，则可适当增大该值。
spark.shuffle.compress|true|是否压缩输出文件
spark.shuffle.file.buffer.kb|32|每次shuffle过程驻留在内存的buffer大小（单位：字节），在shuffle中间数据的产生过程中可减少硬盘的IO操作
spark.reducer.maxMbInFlight|48|设置reduce任务能同时从map任务的输出文件中取多大的数据（单位：M）。在内存较少的情况下需要降低该值
spark.shuffle.manager|HASH|指定如何shuffle数据，默认为HASH，从1.1后新增一种基于排序的方式（SORT），可以更有效的使用内存
spark.shuffle.sort.bypassMergeThreshold|200|在基于排序的方式时，在没有map端的聚合操作或者reduce分区小于该值时应该避免合并排序后数据
#####spark UI相关参数
属性名|默认值|注释
:---------------|:---------------|:---------------
spark.ui.port|4040|任务控制台使用端口
spark.ui.retainedStages|1000|在垃圾回收器收集之前spark UI能保留的最大stage数量
spark.ui.killEnabled|true|允许通过web ui界面停止stage和jobs
spark.eventLog.enabled|false|是否记录spark事件的日志
spark.eventLog.compress|false|是否压缩事件产生的日志
spark.eventLog.dir|file://tmp/spark-events|spark事件产生日志的目录，在这个目录里，每个任务会创建一个子目录存放各个任务的日志文件
#####压缩序列化相关参数
属性名|默认值|注释
:---------------|:---------------|:---------------
spark.broadcast.compress|true|是否压缩需要广播的数据
spark.rdd.compress|false|RDD数据在序列化之后是否进一步进行压缩后再存储到内存或磁盘上
spark.io.compression.codec|snappy|RDD数据或shuffle输出数据使用的压缩算法，有lz4，lzf和snappy三种方式
spark.io.compression.snappy.block.size|32768|在snappy压缩时指定的块大小（字节），降低该值也会降低shuffle过程使用的内存
spark.io.compression.lz4.block.size|32768|和上述类似，只不过只在压缩方式为lz4时有效
spark.closure.serializer|org.apache.spark.serializer.JavaSerializer|序列化类
spark.serializer.objectStreamReset|100|当序列化方式使用JavaSerializer时，序列化器会缓存对象以免写入冗余的数据，但这会使垃圾回收器停止对这些对象进行垃圾收集。所以当使用reset序列化器后就会使垃圾回收器重新收集那些旧对象。该值设置为-1则表示禁止周期性的reset，默认情况下每100个对象就会被reset一次序列化器
spark.kryo.referenceTracking|true|当使用kryo序列化器时，是否跟踪对同一个对象的引用情况，这对对象引用有循环引用或同一对象有多个副本的情况是很有用的。否则可以设置为false以提高性能
spark.kryo.registrationRequired|false|是否需要使用kryo来注册对象。当为true时，如果序列化一个未使用kryo注册的对象则会抛出异常，当为false，kryo会将未注册的类的名字一起写到序列化对象中，所以这会带来性能开支，所以在用户还没有从注册队列中删除相应的类时应该设置为true
spark.kryoserializer.buffer.mb|0.064|kryo的序列化缓冲区的初始值。每个worker的每个core都会有一个缓冲区
spark.kryoserializer.buffer.max.md|64|kryo序列化缓冲区允许的最大值（单位：M），这个值必须大于任何你需要序列化的对象。当遇到buffer limit exceeded异常时可以适当增大该值