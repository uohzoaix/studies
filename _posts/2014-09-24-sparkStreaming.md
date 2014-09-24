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
```scala
import org.apache.spark.\_
```
