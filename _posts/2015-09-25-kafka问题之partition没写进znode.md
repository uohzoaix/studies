---
layout: post
title: "Kafka问题探索并解决"
description: ""
category: 
- kafka
tags: []
---


好了，这几天忙于ctr预估的事情，心在终于有空写写前几天遇到的kafka问题了。

我们的kafka监控是用的开源的KafkaOffsetMonitor工具，这个工具之前文章也解析过源码，很简单的一个工具，可是前几天在监控界面上突然只能看到某一个topic的一个partition在消费，另一个partition并没有在消费，看着lag数字不断的往上涨，很是紧张（这些数字都是涉及到钱的），马上查看了统计程序日志，并没有报错，好好的在处理那个正在消费的partition的数据。又切换到kafka的日志里，发现有一个partition的offset确实没有在增加（只有增加才表示真正消费了数据）。又切换到zookeeper的znode上，查看了该group的消费者情况：get /consumers/group/ids，发现numChildren并不等于topic的数量，接着又查看该group下的该topic的partition的消费者情况：get /consumers/group/owners/topic/0及get /consumers/group/owners/topic/1，发现两者的消费者并不相同，上篇文章说到在kafka消费数据的时候会将topic的消费者注册到zookeeper中，并且某个group的某个topic只能被一个consumer消费，如果某个topic有多个消费者在消费，那么说明有其他的group也在消费该topic，找到问题了，删除另外一个consumer即可：delete /consumers/group/ids/consumeridString，这个时候重启消费程序正常了。

--------------我只是一段不想写太多--------------

但在监控界面上那个topic的消费信息不能显示了，没有显示任何一个partition，查看统计程序日志，愉快的消费着，查看partition的offset：get /consumers/group/offsets/topic/partitionId，两个partition的offset都愉快的在涨。好了，查看之前分析KafkaOffsetMonitor源码的文章，界面上显示的信息是读取/brokers/topics/topic/partitions的信息，查看znode发现该znode的numChildren为0，显然是不对的，正常的应该是有一个名为partitions的child path的，存储的是topic的各个partition的信息，如：get /brokers/topics/topic/partitions/partitionId/state的value：{"controller_epoch":4,"leader":4,"version":1,"leader_epoch":1,"isr":[1,2,3,4]}。回到kafka的源码，这个znode会在AddPartitionsListener这个listener中处理partition的增删，查看kafka的controller.log文件，报错了，好开心：

	java.util.NoSuchElementException: key not found: [topic,1]
        at scala.collection.MapLike$class.default(MapLike.scala:228)
        at scala.collection.AbstractMap.default(Map.scala:58)
        at scala.collection.mutable.HashMap.apply(HashMap.scala:64)
        at kafka.controller.ControllerContext$$anonfun$replicasForPartition$1.apply(KafkaController.scala:109)
        at kafka.controller.ControllerContext$$anonfun$replicasForPartition$1.apply(KafkaController.scala:108)
        at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:244)
        at scala.collection.TraversableLike$$anonfun$map$1.apply(TraversableLike.scala:244)
        at scala.collection.immutable.Set$Set1.foreach(Set.scala:74)
        at scala.collection.TraversableLike$class.map(TraversableLike.scala:244)
        at scala.collection.AbstractSet.scala$collection$SetLike$$super$map(Set.scala:47)
        at scala.collection.SetLike$class.map(SetLike.scala:93)
        at scala.collection.AbstractSet.map(Set.scala:47)
        at kafka.controller.ControllerContext.replicasForPartition(KafkaController.scala:108)
        at kafka.controller.KafkaController.onNewPartitionCreation(KafkaController.scala:472)
        at kafka.controller.PartitionStateMachine$AddPartitionsListener$$anonfun$handleDataChange$1.apply$mcV$sp(PartitionStateMachine.scala:503)
        at kafka.controller.PartitionStateMachine$AddPartitionsListener$$anonfun$handleDataChange$1.apply(PartitionStateMachine.scala:492)
        at kafka.controller.PartitionStateMachine$AddPartitionsListener$$anonfun$handleDataChange$1.apply(PartitionStateMachine.scala:492)
        at kafka.utils.Utils$.inLock(Utils.scala:538)
        at kafka.controller.PartitionStateMachine$AddPartitionsListener.handleDataChange(PartitionStateMachine.scala:491)
        at org.I0Itec.zkclient.ZkClient$6.run(ZkClient.java:547)
        at org.I0Itec.zkclient.ZkEventThread.run(ZkEventThread.java:71)
回到源码，报错的是下面这行：

	val replicas = partitionReplicaAssignment(p)
就是说partitionReplicaAssignment中没有相应的key，再读源码TopicChangeListener中会对partitionReplicaAssignment进行处理，在增删topic的时候即改变/brokers/topics数据时会触发该listener，这时我重新创建（先删除再添加）这个topic后还是报错，再分析报错信息，发现重建topic的时候并不是按照先处理TopicChangeListener再处理AddPartitionsListener的顺序执行，而是会反过来执行，这样就会出现在执行AddPartitionsListener的时候找不到相应的数据了，写入zookeeper前就发生错误了。

--------------我只是一段不想写太多--------------

最后没办法我只能手动创建partitions这个path，最后的路径为/brokers/topics/topic/partitions/partitionId/state，该路径的内容为{"controller_epoch":4,"leader":4,"version":1,"leader_epoch":1,"isr":[1,2,3,4]}。这样在监控页面上就看到了partition的消费情况了。

我们使用的kafka版本是0.8.1.1并不是最新版，升级以后再来继续写。