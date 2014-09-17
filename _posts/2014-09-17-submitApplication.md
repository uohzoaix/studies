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
