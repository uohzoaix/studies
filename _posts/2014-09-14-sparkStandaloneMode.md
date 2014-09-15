---
layout: post
title: "spark之路第三课——spark启动方式之standalone"
description: "spark学习系列第三篇"
category: 
- spark
tags: []
---


上篇文章讲了spark的各种运行方式，由于standalone mode是目前最常见的方式，所以这篇文章主要讲一下这种方式。</br></br>
###1.启动参数
在上篇文章中启动master和workers节点时没有附加任何的参数，但spark提供了如下的一些参数用于启动时的一些环境设置：</br>
*-i IP,--ip IP——>要监听的IP地址或者DNS机器名*</br>
*-p PORT, --port PORT——>要监听的端口（默认：master 7077，workers 随机）*</br>
*--webui-port PORT——>web UI的端口（默认：master 8080，workers 8081）*</br>
*-c CORES, --cores CORES——>允许spark应用使用的机器CPU数量（默认：所有，该值只对workers有作用）*</br>
*-m MEM, --memory MEM——>允许spark应用使用的内存（默认：总内存-1G，该值也只对workers有作用）*</br>
*-d DIR, --work-dir DIR——>分配空间和日志输出的目录（默认：SPARK_HOME/work该值也只对workers有作用）*</br>
###2.集群启动脚本
在以standalone方式启动spark集群时，需要在spark安装目录下创建conf/slaves文件，该文件的内容是你需要启动的那些workers的hostname，每行一个，并且master机器必须能通过ssh连接到所有列出的workers机器。启动和停止集群的相关命令如下：</br>
*sbin/start-master.sh——>在当前及其上启动一个master实例*</br>
*sbin/start-slaves.sh——>在conf/slaves文件列出的所有workers所在机器上启动一个slave实例*</br>
*sbin/start-all.sh——>相当于执行了上述两个命令*</br>
*sbin/stop-master.sh——>停止master实例*</br>
*sbin/stop-slaves.sh——>停止所有的workers实例*</br>
*sbin/stop-all.sh——>相当于执行了上述两个命令*</br>
除此之外，也可通过修改conf/spark-env.sh文件配置整个集群的环境变量，修改完该文件后需要将该文件拷贝到所有的workers机器上才能生效。</br>
下面是一些常见的环境变量：</br>
*SPARK_MASTER_IP——>为master节点指定一个IP地址*</br>
*SPARK_MASTER_PORT——>为master节点指定一个port（默认：7077）*</br>
*SPARK_MASTER_WEBUI_PORT——>master web UI指定的端口（默认：8080）*</br>
*SPARK_MASTER_OPTS——>在master节点上的一些属性，形式如：-Dx=y，具体见下。*</br>
*SPARK_LOCAL_DIRS——>分配空间使用的目录，包括map的输出文件和RDD存放的目录，应该放在本地硬盘中一边快速的取存。也可以使用逗号分隔指定多个目录。*</br>
*SPARK_WORKER_CORES——>指定每个worker节点使用的CPU数量*</br>
*SPARK_WORKER_MEMORY——>指定每个worker节点使用的内存*</br>
*SPARK_WORKER_PORT——>若指定了该值，则在启动worker节点时会使用该端口号（默认：随机）*</br>
*SPARK_WORKER_WEBUI_PORT——>worker web UI端口（默认：8081）*</br>
*SPARK_WORKER_INSTANCES——>指定每台机器上的worker实例数量（默认：1），如果有很多机器或者想要启动多个worker进程可以加大该值，一旦大于1，则需要设置SPARK_WORKER_CORES来指定worker节点使用的CPU数量，否则每个worker会尝试使用所有的CPU。*</br>
*SPARK_WORKER_DIR——>应用运行的目录，包括日志输出（默认：SPARK_HOME/work）*</br>
*SPARK_WORKER_OPTS——>同SPARK_MASTER_OPTS*</br>
*SPARK_DAEMON_MEMORY——>为master和workers分配的后台运行内存（默认：512M）*</br>
*SPARK_DAEMON_JAVA_OPTS——>JVM的参数配置，形式如：-Dx=y*</br>
*SPARK_PUBLIC_DNS——>公共DNS（默认：空）*</br>
注意：启动脚本暂时不能在windows机器上运行，需要手动启动master和workers节点。</br>
下面是SPARK_MASTER_OPTS支持的一些系统属性：</br>
*spark.deploy.retainedApplications——>在web UI上能显示的已完成的应用的最大值（默认：200），那些较旧的应用程序会从web UI上删除。*</br>
*spark.deploy.retainedDrivers——>在web UI上能显示的已完成的drivers的最大值（默认：200），那些较旧的drivers会从web UI上删除。*</br>
*spark.deploy.spreadOut——>配置独立集群是否需要将任务分散到多个节点上运行（true）或整合这些任务到尽可能少的节点上（false）。当为true时有益于数据在HDFS上存储，当为false时适合计算量较大的任务。（默认：true）。该值会在后续文章中详细介绍。*</br>
*spark.deploy.defaultCores——>在没有设置spark.cores.max的情况下可通过该值设定分配给任务的CPU数，如果没有设置该值，任务会默认使用全部的CPU。将该值设小对于分享式集群来说可以防止某个用户使用整个集群导致其他用户得不到集群的使用权。（默认：负无穷大）*</br>
*spark.worker.timeout——>master在该时间内没有收到worker的心跳会认为该worker已经挂掉。（默认：60s）*</br>
下面是SPARK_WORKER_OPTS支持的一些系统属性：
*spark.worker.cleanup.enabled——>定时清理worker和任务目录，该属性只在standalone模式下有作用，如果设置为true那么不管任务是否在运行都会进行清理。（默认：false）*
*spark.worker.cleanup.interval——>worker清理旧任务目录的间隔（默认：1800s）*
*spark.worker.cleanup.appDataTtl——>数据在worker上保留的时间，取决于当前机器的可用硬盘空间。由于任务的jars和logs都会被下载到worker机器上，所以特别是当频繁运行任务时硬盘可用空间会急剧下降（默认：3600\*24\*7）*
###3.提交任务到集群
将spark://IP:PORT参数传递给SparkContext的构造方法中即可。
###4.资源调度
目前standalone模式只提供了FIFO调度方式。为了允许作业的并发执行，可以控制每一个作业可获得资源的最大值，以免个别任务占用过多的资源而使其他任务无法运行，可以设置spark.cores.max属性设置spark能够使用的内核数量。如：
{% highlight objc %}
val conf = new SparkConf()
             .setMaster(...)
             .setAppName(...)
             .set("spark.cores.max", "10")
val sc = new SparkContext(conf)
{% endhighlight %}
另外你也可以设置spark.deploy.defaultCores来设置任务默认能够使用的内核数量。如：
{% highlight objc %}
export SPARK_MASTER_OPTS="-Dspark.deploy.defaultCores=<value>"
{% endhighlight %}
###5.高可用性
standalone模式会将失败的worker节点的任务全部移到另外一个worker上，这样的决策是由master机器来决定的，所以这会带来单点失败问题：一旦master崩溃，新的任务就不会被创建了。下面是解决该问题的两种方案：
####1.zookeeper管理master
使用zookeeper来选举leader和存储相应的状态，这时可以启动多个master连接到同一个zookeeper，其中的一个会被选举为leader其他的会作为备用。当leader挂掉，另外一个master会被选举为leader并恢复挂掉的master的状态，这样就可以继续为worker服务了。这整个操作可能会消耗1至2分钟的时间，当然这段时间只会对新任务有影响，其他的正在运行的任务是不会受到影响的。</br>
为了使用这种方式，在spark-env.sh文件中需要设置以下系统属性：
*spark.deploy.recoveryMode——>如果设置为ZOOKEEPER则表示使用该种方式（默认：NONE）*
*spark.deploy.zookeeper.url——>zookeeper集群地址，如192.168.1.100:2181,192.168.1.101:2181*
*spark.deploy.zookeeper.dir——>zookeeper存储状态的目录（默认：/spark）*</br>
注意：当zookeeper不能选举其中一个master为leader，这时整个集群会发生问题，因为所有的master都会被任务是leader导致master互不相干的工作。</br>
当zookeeper配置正确后，这时就可以启动这些master（可能在不同的机器上，只需要保证它们连在同一个zookeeper集群上即可），当需要部署新任务或添加worker到spark集群中时，需要将leader的IP地址传递给SparkContext，你也可以将所有的master机器的IP地址传递进去，如spark://host1:port1,host2:port2。</br>
当某个leader挂掉后，新的leader会连接所有之前注册过的任务或worker来通知它们leader已经改变，所以你可以在任务时候添加新的master，你唯一需要担心的是新的任务或worker是否能找到新添加进来的leader。