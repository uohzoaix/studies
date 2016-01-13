---
layout: post
title: "Kafka集群之如何确定leader"
description: ""
category: 
- kafka
tags: []
---


在kafka启动后，会初始化一个非常关键的类：KfkaController，该类在kafka运行过程当中会做很多的事情，包括控制partition和replica的状态，选举kafka leader，autoblance等，选举kafka的leader是通过：

	private val controllerElector = new ZookeeperLeaderElector(controllerContext, ZkUtils.ControllerPath, onControllerFailover,
    onControllerResignation, config.brokerId)
完成的。

启动后会调用ZookeeperLeaderElector的elect方法将当前的broker作为leader写入zk的/controller节点中并对该节点注册LeaderChangeListener，当该节点的数据发生变化时说明有其他的broker已经被选举为leader了，这时候更新当前broker的内存数据即可，如果选举时发生异常则会删除该节点触发重新选举机制。

也就是说broker的leader选举并不想zk leader选举那么复杂，简单来说就是哪台broker先启动它就会成为leader，这时候如果其他的broker启动完成后会读取/controller节点的数据更新其各自的内存数据。

接着在向kafka生产数据时，都是通过DefaultEventHandler类来进行一系列的操作：

1.挑选指定topic中replica的leader不为空的partition作为存放数据的partition。

2.向partition的leader broker发送数据。

3.向某个partition未发送成功的数据会重试发送，重试一定次数后抛出异常。

在partition接收到生产的数据时：

	def appendMessagesToLeader(messages: ByteBufferMessageSet, requiredAcks: Int = 0) = {
    inReadLock(leaderIsrUpdateLock) {
      //只有在本地replica是leader的时候才会写日志
      val leaderReplicaOpt = leaderReplicaIfLocal()
      leaderReplicaOpt match {
        case Some(leaderReplica) =>
          val log = leaderReplica.log.get
          val minIsr = log.config.minInSyncReplicas
          val inSyncSize = inSyncReplicas.size

          // Avoid writing to leader if there are not enough insync replicas to make it safe
          if (inSyncSize < minIsr && requiredAcks == -1) {
            throw new NotEnoughReplicasException("Number of insync replicas for partition [%s,%d] is [%d], below required minimum [%d]"
              .format(topic, partitionId, inSyncSize, minIsr))
          }

          val info = log.append(messages, assignOffsets = true)
          // probably unblock some follower fetch requests since log end offset has been updated
          replicaManager.tryCompleteDelayedFetch(new TopicPartitionOperationKey(this.topic, this.partitionId))
          // we may need to increment high watermark since ISR could be down to 1
          maybeIncrementLeaderHW(leaderReplica)
          info
        case None =>
          throw new NotLeaderForPartitionException("Leader not local for partition [%s,%d] on broker %d"
            .format(topic, partitionId, localBrokerId))
      }
    }
    }
代码中首先会判断是否将数据发送到了leader中，如果不是，则不做写数据操作。在正常情况下接收到数据的broker就是leader，因为DefaultEventHandler已经指定了向leader发送数据：

	def leaderReplicaIfLocal(): Option[Replica] = {
    leaderReplicaIdOpt match {
      case Some(leaderReplicaId) =>
        //如果leaderReplica就是该partition所在的broker，就返回该replica否则返回None
        if (leaderReplicaId == localBrokerId)
          getReplica(localBrokerId)
        else
          None
      case None => None
    }
    }

kafka还是有很多的细节需要深入的去弄懂，下篇文章会讲一下数据如何在replica之间进行备份。