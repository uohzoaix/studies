---
layout: post
title: "storm集群配置"
description: "实时流计算框架storm单机配置"
category: 
- storm
tags: []
---


关于storm框架的介绍就不说了，网上可参考。该文主要是集群配置的一些步骤及会出现的相关问题解说。</br></br>
###1.安装zookeeper
很简单，直接将zookeeper下载下来即可，并将其下的bin目录配置到PATH里。使用./zkServer.sh start即可启动，./zkCli.sh是客户端的工具，如果两个命令不报错就说明zk配置成功，使用./zkServer.sh stop可停止zk。
###2.安装zeroMQ
wget http://download.zeromq.org/zeromq-2.1.7.tar.gz</br>
tar -xvf zeromq-2.1.7.tar.gz</br>
cd zeromq-2.1.7</br>
./configure</br>
make && make install</br>
在macosx系统中使用：brew install --universal zeromq安装即可。
###3.安装jzmq
git clone https://github.com/nathanmarz/jzmq.git</br>
cd jzmq</br>
./autogen.sh</br>
./configure</br>
make</br>
make install</br>
在这一步当中可能会遇到如下几个错误：</br>
（1）make[1]: *** 没有规则可以创建“org/zeromq/ZMQ.class”需要的目标“classdist_noinst.stamp”。修正方法，创建classdist_noinst.stamp文件：</br>
touch src/classdist_noinst.stamp</br>
（2）错误：无法访问 org.zeromq.ZMQ   |   make[1]: *** 没有规则可以创建“all”需要的目标“org/zeromq/ZMQ$Context.class”。修正方法，进入src目录，手动编译相关java代码：</br>
javac  ./src/org/zeromq/*.java</br>
（3）jvm找不到jni_md.h文件。修正方法：</br>
在系统中查找该文件并将该文件拷贝到相关的include目录。</br>
###4.安装storm
下载storm，解压，将bin目录配置到PATH中。修改conf/storm.yaml文件：
{% highlight objc %}
storm.zookeeper.servers: 
       - 127.0.0.1 
storm.zookeeper.port: 2181 
nimbus.host: "127.0.0.1" 
storm.local.dir: "/home/tp/storm-0.8.2/tmp/storm" 
supervisor.slots.ports: 
        - 6700 
        - 6701 
        - 6702 
        - 6703
{% endhighlight %}
注意：该文件的每一项的开始处需要加空格，冒号后也必须加空格。</br>
配置文件说明：</br>
storm.local.dir表示storm需要用到的本地目录。nimbus.host表示那一台机器是master机器，即nimbus。storm.zookeeper.servers表示哪几台机器是zookeeper服务器。storm.zookeeper.port表示zookeeper的端口号，这里一定要与zookeeper配置的端口号一致，否则会出现通信错误，切记切记。当然你也可以配superevisor.slot.port，supervisor.slots.ports表示supervisor节点的槽数，就是最多能跑几个worker进程（每个sprout或bolt默认只启动一个worker，但是可以通过conf修改成多个）。</br></br>
安装工作就到此结束。下面是一个例子：</br>
将网上下载下来的storm-starter项目中的src/jvm下的代码导入到eclipse某个新建的项目中，添加storm目录下lib目录中的所有jar包，还需要commons-collections和storm-xxx.jar包，最后将multilang下的resource目录放到项目根目录，待没有报错后将该eclipse项目打成jar包如stormstarter.jar。接着启动zookeeper，然后storm nimbus&启动主节点，storm supervisor&启动从节点，storm ui&启动ui相关的服务，最后将刚生成的jar包发布到storm中：storm jar stormstarter.jar storm.starter.WordCountTopology stormstarter，stormstarter是该任务的名字，可以随便写。如果没有报错输入http://127.0.0.1:8080可看到该任务的运行情况。如果想停止某个任务则运行：storm kill stormstarter