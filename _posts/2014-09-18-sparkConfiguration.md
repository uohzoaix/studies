---
layout: post
title: "spark之路第四课——提交spark任务"
description: "spark学习系列第四篇"
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
属性名|默认值|注释
:---------------|:---------------|:---------------
spark.app.name|(none)|任务名称
spark.master|(none)|集群地址
spark.executor.memory|512m|每个执行器进程所用内存大小
spark.serializer|org.apache.spark.serializer.JavaSerializer|对象序列化所用的类，默认的JavaSerializer性能太差，推荐使用org.apache.spark.serializer.KryoSerializer，你也可以通过集成org.apache.spark.Serializer来实现自己的序列化器
spark.kryo.registrator|(none)|当使用了KryoSerializer，可以设置该值为KryoRegistrator将自定义类注册到Kryo
