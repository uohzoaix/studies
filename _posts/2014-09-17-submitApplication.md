---
layout: post
title: "spark之路第四课——提交spark任务"
description: "spark学习系列第四篇"
category: 
- spark
tags: []
---


spark是使用spark-submit这个命令来提交任务的。用法如下：</br>
{% highlight objc %}
./bin/spark-submit 
  --class <main-class>
  --master <master-url> 
  --deploy-mode <deploy-mode> 
  --conf <key>=<value> 
  ...  other options
  <application-jar> 
  [application-arguments]
{% endhighlight %}
各个参数解释如下：</br>
*--class:一个spark任务的入口方法，一般指main方法。如：org.apache.spark.examples.SparkPi)*</br>
*-master:集群的master URL。如spark://23.195.26.187:7077*</br>
*--deploy-mode:部署方式，有cluster何client两种方式，默认为client*</br>
*--conf:额外的属性*</br>
*application-jar:指定的jar目录，路径必须在整个集群当中可见*</br>
*application-argument:main方法的参数*</br>
###两种部署方式
#####client
一种常见的提交方式是从一个网关机器提交到worker机器上，这个时候应该选择client模式，因为这时driver是直接通过spark-submit进程部署的，任何的输入和输出都会在终端上输出。因此这种方式在需要REPL(如：spark shell)的时候是很适合的。</br>
#####cluster
当应用是通过远程进行提交的，应该选择cluster模式，因为这种模式会减少drivers和executors之间的带宽负载。但是目前standalone，mesos或python应用都不支持该模式。</br>
下面是一些spark-submit的例子：</br>
{% highlight objc %}
\# Run application locally on 8 cores
./bin/spark-submit 
  --class org.apache.spark.examples.SparkPi 
  --master local[8] 
  /path/to/examples.jar 
  100

\# Run on a Spark standalone cluster
./bin/spark-submit 
  --class org.apache.spark.examples.SparkPi 
  --master spark://207.184.161.138:7077 
  --executor-memory 20G 
  --total-executor-cores 100 
  /path/to/examples.jar 
  1000

\# Run on a YARN cluster
export HADOOP_CONF_DIR=XXX
./bin/spark-submit 
  --class org.apache.spark.examples.SparkPi 
  --master yarn-cluster   # can also be yarn-client for client mode
  --executor-memory 20G 
  --num-executors 50 
  /path/to/examples.jar 
  1000

\# Run a Python application on a cluster
./bin/spark-submit 
  --master spark://207.184.161.138:7077 
  examples/src/main/python/pi.py 
  1000
{% endhighlight %}
###master urls
传递给spark-submit中的master url参数可以是如下形式的一种：</br>
*local:本地运行spark且只有一个worker线程*</br>
*local[K]:本地运行spark且有K个worker线程。k一般设置为机器的CPU数量*</br>
*local[\*]:本地运行spark并以尽可能多的worker线程运行*</br>
*spark://HOST:PORT:连接指定的standalone集群*</br>
*mesos://HOST:PORT:连接到指定的mesos集群，如果mesos集群使用zookeeper管理，则为mesos://zk://....*</br>
*yarn-client:以client方式连接到yarn集群*</br>
*yarn-cluster:以cluster模式连接到yarn集群*</br>
###从配置文件读取参数
除了在命令上指定参数外，spark-submit还会读取指定的配置文件中的一些属性，默认会读取conf/spark-defaults.conf。另外一种方式是直接在程序中里的SparkConf类中指定，这三种方式的优先级为SparkConf>spark-submit参数设置>配置文件。</br>
如果你想知道某个参数到底来自哪种方式，可以将--verbose选项传递给spark-submit。</br>
###依赖管理
spark-submit命令可以通过--jars来添加应用程序用到的其他的jar包，这些jar包会被发送到集群中。这些jars可以使用下面几种方式进行添加：</br>
*file:——>指定http文件服务器的绝对路径，每个executor会从文件服务器上进行下载*</br>
*hdfs:,http:,https:,ftp:——>从指定的路径下载*</br>
*local:——>表示直接从当前的worker节点下载本地文件，所以这种方式不存在IO*</br>
注意：由于每个executor节点的SparkContext都会将jar包和文件拷贝到工作目录，所以机器的空闲空间会逐渐减少，YARN集群模式会自动清理这些jar包和文件，但standalone模式需要配置spark.worker.cleanup.appDataTtl属性以使用自动清理功能。