---
layout: post
title: "KafkaController之重新被选为leader"
description: ""
category: 
- kafka
tags: []
---

在之前的leader选举过程一文当中说到当在选举过程当中发生异常时，会重新选举leader，这时候会触发LeaderChangeListener的handleDataDeleted方法，如果此时当前broker重新被选为leader，则会回调onControllerResignation：

	//当前broker重新被选举为controller
    def onControllerResignation() {
	    // de-register listeners
	    deregisterReassignedPartitionsListener()
	    deregisterPreferredReplicaElectionListener()
	
	    // shutdown delete topic manager
	    if (deleteTopicManager != null)
	      deleteTopicManager.shutdown()
	
	    // shutdown leader rebalance scheduler
	    if (config.autoLeaderRebalanceEnable)
	      autoRebalanceScheduler.shutdown()
	
	    inLock(controllerContext.controllerLock) {
	      // de-register partition ISR listener for on-going partition reassignment task
	      deregisterReassignedPartitionsIsrChangeListeners()
	      // shutdown partition state machine
	      partitionStateMachine.shutdown()
	      // shutdown replica state machine
	      replicaStateMachine.shutdown()
	      // shutdown controller channel manager
	      if(controllerContext.controllerChannelManager != null) {
	        controllerContext.controllerChannelManager.shutdown()
	        controllerContext.controllerChannelManager = null
	      }
	      // reset controller context
	      controllerContext.epoch=0
	      controllerContext.epochZkVersion=0
	      brokerState.newState(RunningAsBroker)
	
	      info("Broker %d resigned as the controller".format(config.brokerId))
	    }
    }

该方法主要做以下几件事情：

1.清除/admin/reassign_partitions路径的PartitionsReassignedListener。

2.清除/admin/preferred_replica_election路径的PreferredReplicaElectionListener。

3.清除/brokers/topics/topic/partitions/partitionId/state路径的ReassignedPartitionsIsrChangeListener。

这几个listener的作用在上文都有说过。

清除之前leader遗留的状态后会在leader选举时重新调用elect方法：

	def handleDataDeleted(dataPath: String) {
	      inLock(controllerContext.controllerLock) {
	        debug("%s leader change listener fired for path %s to handle data deleted: trying to elect as a leader"
	          .format(brokerId, dataPath))
	        if(amILeader)
	          onResigningAsLeader()//被重新选为leader
	        elect
	      }
	}
elect方法又会回调onControllerFailover方法重新为leader设置一些必要的数据结构及相关路径的listener（具体见上文）。