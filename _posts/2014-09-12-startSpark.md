---
layout: post
title: "spark之路第一天"
description: "spark学习系列第一篇"
category: 
- spark
tags: []
---


###1.启动hadoop
到hadoop安装目录的bin下运行如下语句：
{% highlight objc %}
./start-all
{% endhighlight %}
###2.上传文件
同样在bin目录运行，input为hadoop文件系统中的目录，可通过hadoop fs的相关命令查看已经存在的文件：
{% highlight objc %}
./hadoop fs -put README.md input
{% endhighlight %}
###3.启动spark
启动spark的方式有很多，下面是其中一种简单方式，关于其他的一些方式以后会详细介绍。
在spark安装目录的bin目录下，运行：
{% highlight objc %}
MASTER=local ./spark-shell
{% endhighlight %}
看到如下输出即表示运行成功：
{% highlight objc %}
14/09/12 20:05:52 INFO Executor: Using REPL class URI: http://192.168.1.101:55267
14/09/12 20:05:52 INFO SparkILoop: Created spark context..
Spark context available as sc.
{% endhighlight %}
###4.读取hadoop文件
{% highlight objc %}
val textFile=sc.textFile("input/README.txt")
#统计文件行数
textFile.count()
#按空格分隔统计各个词的数量。flatMap,map,reduceByKey均为scala语法
val count=textFile.flatMap(line => line.split(" ")).map(word => (word,1)).reduceByKey(_+_)
#输出统计结果
count.collect()
{% endhighlight %}
sc是SparkContext类，它是spark的入口。关于其更加详细介绍待续。</br></br>
spark第一课结束。