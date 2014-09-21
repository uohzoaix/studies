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
#####执行时相关属性
属性名|默认值|注释
:---------------|:---------------|:---------------
spark.default.parallelism|<li>local mode:本地机器的CPU数量<li>mesos file grained mode:8<li>其他模式：所有执行器节点的cpu数量之和与2的最大值|当没有显式设置该值表示系统使用集群中运行shuffle操作（如groupByKey，reduceByKey）的默认的任务数
spark.broadcast.factory|org.apache.spark.broadcast.TorrentBroadcastFactory|广播时使用的实现类
spark.broadcast.blockSize|4096|TorrentBroadcastFactory的块大小。该值过大会降低广播时的并行度（速度变慢），过小的话BlockManager的性能不能发挥到最佳
spark.files.overwrite|false|通过SparkContext.addFile()添加的文件是否可以覆盖之前已经存在并且内容不匹配的文件
spark.files.fetchTimeout|false|获取由driver通过SparkContext.addFile()添加的文件时是否启用通信超时
spark.storage.memoryFraction|0.6|java heap用于spark内存缓存的比例，该值不应该大于jvm中老生代对象的大小。当你自己设置了老生代的大小时可以适当加大该值
spark.storage.unrollFraction|0.2|spark.storage.memoryFraction中用于展开块的内存比例，当没有足够内存来展开新的块的时候会通过丢弃已经存在的旧的块来腾出空间
spark.tachyonStore.baseDir|System.getProperty("java.io.tmpdir")|Tachyon文件系统存放RDD的目录。tachyon文件系统的URL通过spark.tachyonStore.url进行设置。可通过逗号分隔设置多个目录
spark.storage.memoryMapThreshold|8192|以字节为单位的快大小，用于磁盘读取一个块大小时进行内存映射。这可以防止spark在内存映射时使用很小的块，一般情况下，对块进行内存映射的开销接近或低于操作系统的页大小
spark.tachyonStore.url|tachyon://localhost:19998|tachyon文件系统的url
spark.cleaner.ttl|(infinite)|spark记录任何元数据（stages生成、task生成等）的持续时间。定期清理可以确保将超期的元数据删除，这在运行长时间任务时是非常有用的，如运行7*24的spark streaming任务。RDD持久化在内存中的超期数据也会被清理
spark.hadoop.validateOutputSpecs|true|当为true时，在使用saveAsHadoopFile或者其他变体时会验证数据输出的合理性（如检查输出目录是否还存在）。
spark.executor.heartbeatInterval|10000|每个executor向driver发送心跳的间隔时间（毫秒）。
#####网络相关属性
属性名|默认值|注释
:---------------|:---------------|:---------------
spark.driver.host|(本地主机名)|driver监听的IP或主机名，用于与执行器和standalone模式的master节点进行通信
spark.driver.port|(随机)|driver监听的端口号
spark.fileserver.port|(随机)|driver的HTTP文件服务器监听的端口
spark.broadcast.port|(随机)|driver的广播服务器监听的端口，该参数对于torrent广播模式是没有作用的
spark.replClassServer.port|(随机)|driver的HTTP类服务器监听的端口，只用于spark shell
spark.blockManager.port|(随机)|所有块管理者监听的端口
spark.executor.port|(随机)|executor监听的端口，用于与driver进行通信
spark.port.maxRetries|16|绑定到某个端口的最大重试次数
spark.akka.frameSieze|10|以MB为单位的driver和executor之间通信信息的大小，该值越大，driver可以接受更大的计算结果（如在一个很大的数据集上使用collect()方法）
spark.akka.threads|4|用于通信的actor线程数，当在很大的集群中driver拥有更多的CPU内核数的driver可以适当增加该属性的值
spark.akka.timeout|100|spark节点之间通信的超时时间（秒）
spark.akka.heartbeat.pauses|600|下面三个参数通常一起使用。如果启用错误探测器，有助于对恶意的executor的定位，而对于由于GC暂停或网络滞后引起的情况下，不需要开启错误探测器，另外错误探测器的开启会导致由于心跳信息的频繁交换引起网络泛滥。设大该值可以禁用akka内置的错误探测器，表示akka可接受的心跳停止时间（秒）
spark.akka.failure-detector.threshold|300.0|设大该值可以禁用akka内置的错误探测器，对应akka的akka.remote.transport-failure-detector.threshold
spark.akka.heartbeat.interval|1000|设大该值可以禁用akka内置的错误探测器，该值越大会减少网络负载，越小就会向akka的错误探测器发送信息
#####调度相关属性
属性名|默认值|注释
:---------------|:---------------|:---------------
spark.task.cpus|1|为每个人物分配的cpu数
spark.task.maxFailures|4|每个单独任务允许的失败次数，必须设置为大于1，重试次数=该值-1
spark.scheduler.mode|FIFO|提交到SparkContext的任务的调度模式。设置为FAIR表示使用公平的方式进行调度而不是以队列的方式
spark.cores.max|(未设置)|当应用程序运行在standalone集群活着粗粒度共享模式mesos集群时，应用程序向集群请求的最大cpu内核总数（不是指每台机器，而是整个集群）。如果不设置，对于standalone集群将使用spark.deploy.defaultCores的值，而mesos将使用集群中可用的内核
spark.mesos.coarse|false|如果设置为true，在mesos集群上就会使用粗粒度共享模式，这种模式使得spark获得一个长时间运行的mesos任务而不是一个spark任务对应一个mesos任务。这对短查询会带来更低的等待时间，但资源会在整个spark任务的执行期间内被占用
spark.speculation|false|当设置为true时，将会推断任务的执行情况，当一个或多个任务在stage里执行较慢时，这些任务会被重新发布
spark.speculation.interval|100|spark推断任务执行情况的间隔时间（毫秒）
spark.speculation.quantile|0.75|推断启动前，stage必须要完成总task的百分比
spark.speculation.multiplier|1.5|比已完成task的运行速度中位数慢多少倍才启用推断
spark.locality.wait|3000|启动一个本地数据任务的等待时间，当等待时间超过该值时，就会启动下一个本地优先级别的任务。该设置同样可以应用到各优先级别的本地性之间（本地进程，本地节点，本地机架，任意节点），当然可以通过spark.locality.wait.node等参数设置不同优先级别的本地性
spark.locality.wait.process|spark.locality.wait|本地进程的本地等待时间，它会影响尝试访问缓存数据的任务
spark.locality.wait.node|spark.locality.wait|本地节点的本地等待时间，当设置为0，就会忽略本地节点并立即在本地机架上寻找
spark.locality.wait.rack|spark.locality.wait|本地机架的本地等待时间
spark.scheduler.revive.interval|1000|复活重新获取资源的任务的最长时间间隔（毫秒）
spark.scheduler.minRegisteredResourcesRatio|0|在调度开始之前已注册的资源需要达到的最小比例，如果不设置该属性的话，那么调度开始之前需要等待的最大时间由spark.scheduler.maxRegisteredResourcesWaitingTime设置
spark.scheduler.maxRegisteredResourcesWaitingTime|30000|调度开始之前需要等待的最大时间(毫秒)
spark.localExecution.enabled|false|在spark调用first()或take()等任务时是否将任务发送给集群，当设置为true时会使这些任务执行速度加快，但是可能需要将整个分区的数据装载到driver