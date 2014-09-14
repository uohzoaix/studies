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
**-i IP,--ip IP——>要监听的IP地址或者DNS机器名</br>
-p PORT, --port PORT——>要监听的端口（默认：master 7077，workers 随机）</br>
--webui-port PORT——>web UI的端口（默认：master 8080，workers 8081）</br>
-c CORES, --cores CORES——>允许spark应用使用的机器CPU数量（默认：所有，该值只对workers有作用）</br>
-m MEM, --memory MEM——>允许spark应用使用的内存（默认：总内存-1G，该值也只对workers有作用）</br>
-d DIR, --work-dir DIR——>分配空间和日志输出的目录（默认：SPARK_HOME/work该值也只对workers有作用）</br>
###2.集群启动脚本
在以standalone方式启动spark集群时，需要在spark安装目录下创建conf/slaves文件，该文件的内容是你需要启动的那些workers的hostname，每行一个，并且master机器必须能通过ssh连接到所有列出的workers机器。启动和停止集群的相关命令如下：</br>
sbin/start-master.sh——>在当前及其上启动一个master实例</br>
sbin/start-slaves.sh——>在conf/slaves文件列出的所有workers所在机器上启动一个slave实例</br>
sbin/start-all.sh——>相当于执行了上述两个命令</br>
sbin/stop-master.sh——>停止master实例</br>
sbin/stop-slaves.sh——>停止所有的workers实例</br>
sbin/stop-all.sh——>相当于执行了上述两个命令</br>
除此之外，也可通过修改conf/spark-env.sh文件配置整个集群的环境变量，修改完该文件后需要将该文件拷贝到所有的workers机器上才能生效。</br>
下面是一些常见的环境变量：</br>
SPARK_MASTER_IP——>为master节点指定一个IP地址</br>
SPARK_MASTER_PORT——>为master节点指定一个port（默认：7077）</br>
SPARK_MASTER_WEBUI_PORT——>master web UI指定的端口（默认：8080）</br>
SPARK_MASTER_OPTS——>在master节点上的一些属性，形式如：-Dx=y，具体见下。</br>
SPARK_LOCAL_DIRS——>分配空间使用的目录，包括map的输出文件和RDD存放的目录，应该放在本地硬盘中一边快速的取存。也可以使用逗号分隔指定多个目录。</br>
SPARK_WORKER_CORES——>指定每个worker节点使用的CPU数量</br>
SPARK_WORKER_MEMORY——>指定每个worker节点使用的内存</br>
SPARK_WORKER_PORT——>若指定了该值，则在启动worker节点时会使用该端口号（默认：随机）</br>
SPARK_WORKER_WEBUI_PORT——>worker web UI端口（默认：8081）</br>
SPARK_WORKER_INSTANCES——>指定每台机器上的worker实例数量（默认：1），如果有很多机器或者想要启动多个worker进程可以加大该值，一旦大于1，则需要设置SPARK_WORKER_CORES来指定worker节点使用的CPU数量，否则每个worker会尝试使用所有的CPU。</br>
SPARK_WORKER_DIR——>应用运行的目录，包括日志输出（默认：SPARK_HOME/work）</br>
SPARK_WORKER_OPTS——>同SPARK_MASTER_OPTS</br>
SPARK_DAEMON_MEMORY——>为master和workers分配的后台运行内存（默认：512M）</br>
SPARK_DAEMON_JAVA_OPTS——>JVM的参数配置，形式如：-Dx=y</br>
SPARK_PUBLIC_DNS——>公共DNS（默认：空）</br>
注意：启动脚本暂时不能在windows机器上运行，需要手动启动master和workers节点。</br>
下面是SPARK_MASTER_OPTS支持的一些系统属性：</br>
spark.deploy.retainedApplications——>在web UI上能显示的已完成的应用的最大值（默认：200），那些较旧的应用程序会从web UI上删除。</br>
spark.deploy.retainedDrivers——>在web UI上能显示的已完成的drivers的最大值（默认：200），那些较旧的drivers会从web UI上删除。</br>
spark.deploy.spreadOut——>配置独立集群是否需要将任务分散到多个节点上运行（true）或整合这些任务到尽可能少的节点上（false）。当为true时有益于数据在HDFS上存储，当为false时适合计算量较大的任务。（默认：true）。该值会在后续文章中详细介绍。</br>
spark.deploy.defaultCores——>在没有设置spark.cores.max的情况下可通过该值设定分配给任务的CPU数，如果没有设置该值，任务会默认使用全部的CPU。将该值设小对于分享式集群来说可以防止某个用户使用整个集群导致其他用户得不到集群的使用权。（默认：负无穷大）</br>
spark.worker.timeout——>master在该时间内没有收到worker的心跳会认为该worker已经挂掉。（默认：60s）</br>
