---
layout: post
title: "KafkaController之leader选举成功"
description: ""
category: 
- kafka
tags: []
---


上文说到在kafka集群中如何选择leader，在选完leader后就会调用onControllerFailOver方法。该方法会处理以下几个事情：

1.读取zk中上个controller写入的epoch和version

2.将第一步读取出来的epoch加1并写入zk中，以便让其他broker知晓已经有broker被选为controller了（通过比较zk中的epoch与自身的epoch）

3.向/admin/reassign_partitions注册PartitionsReassignedListener，该listener主要是监测/admin/reassign_partitions数据改变的事件，该路径的数据是由ReassignPartitionsCommand命令行写入，写入数据的具体格式类似为：{version:1,partitions:{topic:xxx,partition:1,replicas:[1,2]}}。当写入数据后，该listener就会触发handleDataChange方法：

	def handleDataChange(dataPath: String, data: Object) {
	    debug("Partitions reassigned listener fired for path %s. Record partitions to be reassigned %s"
	      .format(dataPath, data))
	    val partitionsReassignmentData = ZkUtils.parsePartitionReassignmentData(data.toString)
	    //在所要分配的partition中去除正在被分配的那些partition
	    val partitionsToBeReassigned = inLock(controllerContext.controllerLock) {
	      partitionsReassignmentData.filterNot(p => controllerContext.partitionsBeingReassigned.contains(p._1))
	    }
	    partitionsToBeReassigned.foreach { partitionToBeReassigned =>
	      inLock(controllerContext.controllerLock) {
	        //判断该topic是否是待删除的topic
	        if(controller.deleteTopicManager.isTopicQueuedUpForDeletion(partitionToBeReassigned._1.topic)) {
	          error("Skipping reassignment of partition %s for topic %s since it is currently being deleted"
	            .format(partitionToBeReassigned._1, partitionToBeReassigned._1.topic))
	          controller.removePartitionFromReassignedPartitions(partitionToBeReassigned._1)
	        } else {
	          val context = new ReassignedPartitionsContext(partitionToBeReassigned._2)
	          controller.initiateReassignReplicasForTopicPartition(partitionToBeReassigned._1, context)
	        }
	      }
	    }
    }
removePartitionFromReassignedPartitions方法是将待删除的partition从/admin/reassign_partitions路径删除并将最新的待分配的partition数据更新到/admin/reassign_partitions中：

	def removePartitionFromReassignedPartitions(topicAndPartition: TopicAndPartition) {
	    if(controllerContext.partitionsBeingReassigned.get(topicAndPartition).isDefined) {
	      // stop watching the ISR changes for this partition
	      zkClient.unsubscribeDataChanges(ZkUtils.getTopicPartitionLeaderAndIsrPath(topicAndPartition.topic, topicAndPartition.partition),
	        controllerContext.partitionsBeingReassigned(topicAndPartition).isrChangeListener)
	    }
	    // read the current list of reassigned partitions from zookeeper
	    ///admin/reassign_partitions的数据是正在分配replica的那些partition，当分配完后相应的partition会被删除
	    val partitionsBeingReassigned = ZkUtils.getPartitionsBeingReassigned(zkClient)
	    // remove this partition from that list
	    //更新正在分配的那些partition
	    val updatedPartitionsBeingReassigned = partitionsBeingReassigned - topicAndPartition
	    // write the new list to zookeeper
	    //如果没有正在分配的partition了，那么直接删除该path，否则修改该path的数据
	    ZkUtils.updatePartitionReassignmentData(zkClient, updatedPartitionsBeingReassigned.mapValues(_.newReplicas))
	    // update the cache. NO-OP if the partition's reassignment was never started
	    controllerContext.partitionsBeingReassigned.remove(topicAndPartition)
    }
initiateReassignReplicasForTopicPartition方法判断该partition是否需要重新分配replica，仅仅在所要分配的新的replica与之前向该partition分配的replica不一致并且新的replica都是有效的情况下才会为该partition重新分配replica：

	def onPartitionReassignment(topicAndPartition: TopicAndPartition, reassignedPartitionContext: ReassignedPartitionsContext) {
	    val reassignedReplicas = reassignedPartitionContext.newReplicas
	    //判断所需分配的replica是否都在之前分配的isr中
	    areReplicasInIsr(topicAndPartition.topic, topicAndPartition.partition, reassignedReplicas) match {
	      case false =>
	        //需要新分配的replicas有些不在以前的isr中，这时候需要修改相应的数据
	        info("New replicas %s for partition %s being ".format(reassignedReplicas.mkString(","), topicAndPartition) +
	          "reassigned not yet caught up with the leader")
	        //所需要分配的新的replica
	        val newReplicasNotInOldReplicaList = reassignedReplicas.toSet -- controllerContext.partitionReplicaAssignment(topicAndPartition).toSet
	        //所有replica
	        val newAndOldReplicas = (reassignedPartitionContext.newReplicas ++ controllerContext.partitionReplicaAssignment(topicAndPartition)).toSet
	        //1. Update AR in ZK with OAR + RAR.
	        updateAssignedReplicasForPartition(topicAndPartition, newAndOldReplicas.toSeq)
	        //2. Send LeaderAndIsr request to every replica in OAR + RAR (with AR as OAR + RAR).
	        //向所有replica发送修改leaderAndIsr请求
	        updateLeaderEpochAndSendRequest(topicAndPartition, controllerContext.partitionReplicaAssignment(topicAndPartition),
	          newAndOldReplicas.toSeq)
	        //3. replicas in RAR - OAR -> NewReplica
	        //将需要添加的新的replica状态设置为NewReplica
	        startNewReplicasForReassignedPartition(topicAndPartition, reassignedPartitionContext, newReplicasNotInOldReplicaList)
	        info("Waiting for new replicas %s for partition %s being ".format(reassignedReplicas.mkString(","), topicAndPartition) +
	          "reassigned to catch up with the leader")
	      case true =>
	        //所要分配的replica都在之前分配的isr中
	        //4. Wait until all replicas in RAR are in sync with the leader.
	        //不需要再进行分配的那些replica
	        val oldReplicas = controllerContext.partitionReplicaAssignment(topicAndPartition).toSet -- reassignedReplicas.toSet
	        //5. replicas in RAR -> OnlineReplica
	        //将需要分配的replica的状态置为Online
	        reassignedReplicas.foreach { replica =>
	          replicaStateMachine.handleStateChanges(Set(new PartitionAndReplica(topicAndPartition.topic, topicAndPartition.partition,
	            replica)), OnlineReplica)
	        }
	        //6. Set AR to RAR in memory.
	        //7. Send LeaderAndIsr request with a potential new leader (if current leader not in RAR) and
	        //   a new AR (using RAR) and same isr to every broker in RAR
	        //把最新需要分配的replica放入缓存并且向所有replica发出修改leaderAndIsr的消息
	        //当所需分配的replica不包含当前该partition的leader则需要在要重新分配的replicas中重新选取leader
	        moveReassignedPartitionLeaderIfRequired(topicAndPartition, reassignedPartitionContext)
	        //8. replicas in OAR - RAR -> Offline (force those replicas out of isr)
	        //9. replicas in OAR - RAR -> NonExistentReplica (force those replicas to be deleted)
	        //将需要删除的replica的状态设置为NonExistentReplica
	        stopOldReplicasOfReassignedPartition(topicAndPartition, reassignedPartitionContext, oldReplicas)
	        //10. Update AR in ZK with RAR.
	        //更新zk中的数据
	        updateAssignedReplicasForPartition(topicAndPartition, reassignedReplicas)
	        //11. Update the /admin/reassign_partitions path in ZK to remove this partition.
	        //分配完成后更新/admin/reassign_partitions的数据，如果/admin/reassign_partitions中没有需要分配的partition则删除该路径
	        removePartitionFromReassignedPartitions(topicAndPartition)
	        info("Removed partition %s from the list of reassigned partitions in zookeeper".format(topicAndPartition))
	        controllerContext.partitionsBeingReassigned.remove(topicAndPartition)
	        //12. After electing leader, the replicas and isr information changes, so resend the update metadata request to every broker
	        //向所有broker发送UpdateMetadataRequest
	        sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq, Set(topicAndPartition))
	        // signal delete topic thread if reassignment for some partitions belonging to topics being deleted just completed
	        deleteTopicManager.resumeDeletionForTopics(Set(topicAndPartition.topic))
	    }
    }
4.向/admin/preferred_replica_election注册PreferredReplicaElectionListener，/admin/preferred_replica_election路径的数据是由PreferredReplicaLeaderElectionCommand命令行写入，该命令行的作用是为partition重新选举leader replica的，写入的数据格式为：{partitions:[{topic:foo,partition:1},{topic:foobar,partition:2}]}，当/admin/preferred_replica_election路径的数据发生改变时，就会触发PreferredReplicaElectionListener的handleDataChange方法：

	def handleDataChange(dataPath: String, data: Object) {
	    debug("Preferred replica election listener fired for path %s. Record partitions to undergo preferred replica election %s"
	            .format(dataPath, data.toString))
	    inLock(controllerContext.controllerLock) {
	      val partitionsForPreferredReplicaElection = PreferredReplicaLeaderElectionCommand.parsePreferredReplicaElectionData(data.toString)
	      ////正在选举leader replica的那些partition
	      if(controllerContext.partitionsUndergoingPreferredReplicaElection.size > 0)
	        info("These partitions are already undergoing preferred replica election: %s"
	          .format(controllerContext.partitionsUndergoingPreferredReplicaElection.mkString(",")))
	      //去除那些正在选举replica的partition
	      val partitions = partitionsForPreferredReplicaElection -- controllerContext.partitionsUndergoingPreferredReplicaElection
	      //去除那些需要被删除的topic对应的partition
	      val partitionsForTopicsToBeDeleted = partitions.filter(p => controller.deleteTopicManager.isTopicQueuedUpForDeletion(p.topic))
	      if(partitionsForTopicsToBeDeleted.size > 0) {
	        error("Skipping preferred replica election for partitions %s since the respective topics are being deleted"
	          .format(partitionsForTopicsToBeDeleted))
	      }
	      controller.onPreferredReplicaElection(partitions -- partitionsForTopicsToBeDeleted)
	    }
    }
onPreferredReplicaElection方法的定义为：

	def onPreferredReplicaElection(partitions: Set[TopicAndPartition], isTriggeredByAutoRebalance: Boolean = false) {
	    info("Starting preferred replica leader election for partitions %s".format(partitions.mkString(",")))
	    try {
	      //修改正在选举leader replica的数据
	      controllerContext.partitionsUndergoingPreferredReplicaElection ++= partitions
	      //将这些topic设置为延迟删除
	      deleteTopicManager.markTopicIneligibleForDeletion(partitions.map(_.topic))
	      //将这些partition的状态置为Online并为这些partition分别选举一个replica作为leader，选举的方法是直接取分配的所有replica的第一个作为leader
	      partitionStateMachine.handleStateChanges(partitions, OnlinePartition, preferredReplicaPartitionLeaderSelector)
	    } catch {
	      case e: Throwable => error("Error completing preferred replica leader election for partitions %s".format(partitions.mkString(",")), e)
	    } finally {
	      removePartitionsFromPreferredReplicaElection(partitions, isTriggeredByAutoRebalance)
	      deleteTopicManager.resumeDeletionForTopics(partitions.map(_.topic))
	    }
    }
逻辑并不难，只是为指定的partition选举一个leader replica出来，选举的方法是直接取replicas的第一个作为leader。

5.为/brokers/topics注册TopicChangeListener和DeleteTopicsListener，当topic数量发生改变时，就会触发TopicChangeListener的handleChildChange方法：

	def handleChildChange(parentPath : String, children : java.util.List[String]) {
      inLock(controllerContext.controllerLock) {
        if (hasStarted.get) {
          try {
            //当前在zk中最新的topic数据
            val currentChildren = {
              import JavaConversions._
              debug("Topic change listener fired for path %s with children %s".format(parentPath, children.mkString(",")))
              (children: Buffer[String]).toSet
            }
            //新增的topic
            val newTopics = currentChildren -- controllerContext.allTopics
            //被删除的topic
            val deletedTopics = controllerContext.allTopics -- currentChildren
            controllerContext.allTopics = currentChildren

            //获取新增的topic对应的replica分布情况
            val addedPartitionReplicaAssignment = ZkUtils.getReplicaAssignmentForTopics(zkClient, newTopics.toSeq)
            //更新partitionReplicaAssignment数据
            controllerContext.partitionReplicaAssignment = controllerContext.partitionReplicaAssignment.filter(p =>
              !deletedTopics.contains(p._1.topic))
            controllerContext.partitionReplicaAssignment.++=(addedPartitionReplicaAssignment)
            info("New topics: [%s], deleted topics: [%s], new partition replica assignment [%s]".format(newTopics,
              deletedTopics, addedPartitionReplicaAssignment))
            if(newTopics.size > 0)
              //为新增的topic注册partitionChangeListener，并将这些topic对应的partition的状态设置为OnlinePartition
              controller.onNewTopicCreation(newTopics, addedPartitionReplicaAssignment.keySet.toSet)
          } catch {
            case e: Throwable => error("Error while handling new topic", e )
          }
        }
      }
    }
当删除一个topic时（通过命令行：TopicCommand完成），比如删除xxx这个topic，这时就会在zk中创建/admin/delete_topics/xxx的路径，创建完成之后就会触发DeleteTopicsListener的handleChildChange方法：

	def handleChildChange(parentPath : String, children : java.util.List[String]) {
      inLock(controllerContext.controllerLock) {
        //待删除的topic
        var topicsToBeDeleted = {
          import JavaConversions._
          (children: Buffer[String]).toSet
        }
        debug("Delete topics listener fired for topics %s to be deleted".format(topicsToBeDeleted.mkString(",")))
        val nonExistentTopics = topicsToBeDeleted.filter(t => !controllerContext.allTopics.contains(t))
        if(nonExistentTopics.size > 0) {
          warn("Ignoring request to delete non-existing topics " + nonExistentTopics.mkString(","))
          //不存在的topic直接删除路径
          nonExistentTopics.foreach(topic => ZkUtils.deletePathRecursive(zkClient, ZkUtils.getDeleteTopicPath(topic)))
        }
        topicsToBeDeleted --= nonExistentTopics
        if(topicsToBeDeleted.size > 0) {
          info("Starting topic deletion for topics " + topicsToBeDeleted.mkString(","))
          // mark topic ineligible for deletion if other state changes are in progress
          topicsToBeDeleted.foreach { topic =>
            //该topic正在选举leader replica
            val preferredReplicaElectionInProgress =
              controllerContext.partitionsUndergoingPreferredReplicaElection.map(_.topic).contains(topic)
            //该topic正在分配partition
            val partitionReassignmentInProgress =
              controllerContext.partitionsBeingReassigned.keySet.map(_.topic).contains(topic)
            if(preferredReplicaElectionInProgress || partitionReassignmentInProgress)
              //延迟删除该topic
              controller.deleteTopicManager.markTopicIneligibleForDeletion(Set(topic))
          }
          // add topic to deletion list
          //加入删除队列，唤醒TopicDeletionThread
          controller.deleteTopicManager.enqueueTopicsForDeletion(topicsToBeDeleted)
        }
      }
    }

DeleteTopicsThread线程主要做三件事：

（1）.向所有的broker发送UpdateMetadata请求，以使broker不再接受待删除的topic的请求

（2）.设置topic的replica的状态为OffLine，这时会发送StopReplicaRequest到相应的replica并向leader replica发送LeaderAndIsrRequest，如果leader replica也被设置为OffLine，那么leader会被设置为-1

（3）.设置topic的replica的状态为ReplicaDeletionStarted，这时会向broker发送StopReplicaRequest，进而删除replica的所有临时数据

主要代码如下：

	private def startReplicaDeletion(replicasForTopicsToBeDeleted: Set[PartitionAndReplica]) {
	    replicasForTopicsToBeDeleted.groupBy(_.topic).foreach { case(topic, replicas) =>
	      //该topic的有效replica
	      var aliveReplicasForTopic = controllerContext.allLiveReplicas().filter(p => p.topic.equals(topic))
	      //无效的replica
	      val deadReplicasForTopic = replicasForTopicsToBeDeleted -- aliveReplicasForTopic
	      //已删除的replica
	      val successfullyDeletedReplicas = controller.replicaStateMachine.replicasInState(topic, ReplicaDeletionSuccessful)
	      //待删除的replica
	      val replicasForDeletionRetry = aliveReplicasForTopic -- successfullyDeletedReplicas
	      // move dead replicas directly to failed state
	      replicaStateMachine.handleStateChanges(deadReplicasForTopic, ReplicaDeletionIneligible)
	      // send stop replica to all followers that are not in the OfflineReplica state so they stop sending fetch requests to the leader
	      //将待删除的replica的状态设置为Offline
	      replicaStateMachine.handleStateChanges(replicasForDeletionRetry, OfflineReplica)
	      debug("Deletion started for replicas %s".format(replicasForDeletionRetry.mkString(",")))
	      controller.replicaStateMachine.handleStateChanges(replicasForDeletionRetry, ReplicaDeletionStarted,
	        new Callbacks.CallbackBuilder().stopReplicaCallback(deleteTopicStopReplicaCallback).build)
	      if(deadReplicasForTopic.size > 0) {
	        debug("Dead Replicas (%s) found for topic %s".format(deadReplicasForTopic.mkString(","), topic))
	        markTopicIneligibleForDeletion(Set(topic))
	      }
	    }
    }
6.为/brokers/ids注册BrokerChangeListener，/brokers/ids路径的数据由KafkaHealthcheck类写入，该类的startup方法会在KafkaServer中被调用，将当前的brokerid写入该路径中。当有broker增减时即/brokers/ids路径会有子路径发生变化就会触发该listener的handleChildChange方法：

	def handleChildChange(parentPath : String, currentBrokerList : java.util.List[String]) {
      info("Broker change listener fired for path %s with children %s".format(parentPath, currentBrokerList.mkString(",")))
      inLock(controllerContext.controllerLock) {
        if (hasStarted.get) {
          ControllerStats.leaderElectionTimer.time {
            try {
              val curBrokerIds = currentBrokerList.map(_.toInt).toSet
              //新增的broker
              val newBrokerIds = curBrokerIds -- controllerContext.liveOrShuttingDownBrokerIds
              val newBrokerInfo = newBrokerIds.map(ZkUtils.getBrokerInfo(zkClient, _))
              val newBrokers = newBrokerInfo.filter(_.isDefined).map(_.get)
              //要删除的broker
              val deadBrokerIds = controllerContext.liveOrShuttingDownBrokerIds -- curBrokerIds
              //更新内存中broker数据
              controllerContext.liveBrokers = curBrokerIds.map(ZkUtils.getBrokerInfo(zkClient, _)).filter(_.isDefined).map(_.get)
              info("Newly added brokers: %s, deleted brokers: %s, all live brokers: %s"
                .format(newBrokerIds.mkString(","), deadBrokerIds.mkString(","), controllerContext.liveBrokerIds.mkString(",")))
              //创建新增broker的channel,用于发送和接收数据
              newBrokers.foreach(controllerContext.controllerChannelManager.addBroker(_))
              //删除待删broker对应的channel
              deadBrokerIds.foreach(controllerContext.controllerChannelManager.removeBroker(_))
              if(newBrokerIds.size > 0)
                controller.onBrokerStartup(newBrokerIds.toSeq)
              if(deadBrokerIds.size > 0)
                controller.onBrokerFailure(deadBrokerIds.toSeq)
            } catch {
              case e: Throwable => error("Error while handling broker changes", e)
            }
          }
        }
      }
    }
该方法中的onBrokerStartup和onBrokerFailure方法比较重要，这两个方法确保partition和replica在集群中动态发生变化。

onBrokerStartup方法主要做下面几件事情：

（1）向新增的broker发送UpdateMetadata请求，该请求会修改broker的一些内存数据，比如partition及replica的分配情况。

（2）将新增broker对应的replica状态置为Online

（3）将之前该broker被置为New或Offline的partition的状态重新置为Online

（4）将新增broker中正在分配partition的topic重新分配partition（onPartitionReassignment）。

（5）删除新增broker中需要删除的topic。

onBrokerFailure方法主要做一下几件事情：

（1）将partition的leader在将要去除的broker中的那些partition的状态置为Offline。

（2）将那些在需要去除的broker中但不需要删除的topic对应的broker的状态置为Offline。

（3）删除那些需要删除的topic。

7.初始化ControllerContext，包括当前可用broker，所有topic，partition及其replica分配情况，partition的leaderIsr信息等数据：

	private def initializeControllerContext() {
	    // update controller cache with delete topic information
	    controllerContext.liveBrokers = ZkUtils.getAllBrokersInCluster(zkClient).toSet
	    controllerContext.allTopics = ZkUtils.getAllTopics(zkClient).toSet
	    controllerContext.partitionReplicaAssignment = ZkUtils.getReplicaAssignmentForTopics(zkClient, controllerContext.allTopics.toSeq)
	    controllerContext.partitionLeadershipInfo = new mutable.HashMap[TopicAndPartition, LeaderIsrAndControllerEpoch]
	    controllerContext.shuttingDownBrokerIds = mutable.Set.empty[Int]
	    // update the leader and isr cache for all existing partitions from Zookeeper
	    //将zk中的leaderIsr信息放入partitionLeadershipInfo中
	    updateLeaderAndIsrCache()
	    // start the channel manager
	    startChannelManager()
	    //更新partitionsUndergoingPreferredReplicaElection的数据，保留可以选举leader的partition
	    initializePreferredReplicaElection()
	    //更新partitionsBeingReassigned的数据，保留可以分配replica的partition
	    initializePartitionReassignment()
	    //初始化TopicDeletionManager
	    initializeTopicDeletion()
	    info("Currently active brokers in the cluster: %s".format(controllerContext.liveBrokerIds))
	    info("Currently shutting brokers in the cluster: %s".format(controllerContext.shuttingDownBrokerIds))
	    info("Current list of topics in the cluster: %s".format(controllerContext.allTopics))
    }
8.为所有topic（路径为/brokers/topics/xxx）注册AddPartitionsListener，当某个topic的数据发生变化时就会触发handleDataChange方法：

	def handleDataChange(dataPath : String, data: Object) {
      inLock(controllerContext.controllerLock) {
        try {
          info("Add Partition triggered " + data.toString + " for path " + dataPath)
          val partitionReplicaAssignment = ZkUtils.getReplicaAssignmentForTopics(zkClient, List(topic))
          //新增的partition
          val partitionsToBeAdded = partitionReplicaAssignment.filter(p =>
            !controllerContext.partitionReplicaAssignment.contains(p._1))
          if(controller.deleteTopicManager.isTopicQueuedUpForDeletion(topic))
            error("Skipping adding partitions %s for topic %s since it is currently being deleted"
                  .format(partitionsToBeAdded.map(_._1.partition).mkString(","), topic))
          else {
            if (partitionsToBeAdded.size > 0) {
              info("New partitions to be added %s".format(partitionsToBeAdded))
              //将新增partition的状态置为Online
              controller.onNewPartitionCreation(partitionsToBeAdded.keySet.toSet)
            }
          }
        } catch {
          case e: Throwable => error("Error while handling add partitions for data path " + dataPath, e )
        }
      }
    }
9.启动checkAndTriggerPartitionRebalance线程，该线程主要任务是检查当前集群中topic的leader replica与分配的replicas的第一个不同的topic数量是否占到总topic的数量的指定比例，如果占到了就为这些topic重新选取leader replica：

	private def checkAndTriggerPartitionRebalance(): Unit = {
	    if (isActive()) {
	      trace("checking need to trigger partition rebalance")
	      // get all the active brokers
	      var preferredReplicasForTopicsByBrokers: Map[Int, Map[TopicAndPartition, Seq[Int]]] = null
	      inLock(controllerContext.controllerLock) {
	        preferredReplicasForTopicsByBrokers =
	          //topic-0:[1,2] topic-1:[1,2,3] topic-2:[2,3]
	          //1:{topic-0:[1,2],topic-1:[1,2,3]} 2:{topic-2:[2,3]}
	          //按分配的replica的第一个元素进行groupBy
	          controllerContext.partitionReplicaAssignment.filterNot(p => deleteTopicManager.isTopicQueuedUpForDeletion(p._1.topic)).groupBy {
	            case(topicAndPartition, assignedReplicas) => assignedReplicas.head
	          }
	      }
	      debug("preferred replicas by broker " + preferredReplicasForTopicsByBrokers)
	      // for each broker, check if a preferred replica election needs to be triggered
	      preferredReplicasForTopicsByBrokers.foreach {
	        //这里的leaderBroker指的是assignedReplicas.head，即第一个replica作为leader
	        case(leaderBroker, topicAndPartitionsForBroker) => {
	          var imbalanceRatio: Double = 0
	          var topicsNotInPreferredReplica: Map[TopicAndPartition, Seq[Int]] = null
	          inLock(controllerContext.controllerLock) {
	            //获取topic当前leader与分配的replica第一个元素不同的topic
	            topicsNotInPreferredReplica =
	              topicAndPartitionsForBroker.filter {
	                case(topicPartition, replicas) => {
	                  controllerContext.partitionLeadershipInfo.contains(topicPartition) &&
	                  controllerContext.partitionLeadershipInfo(topicPartition).leaderAndIsr.leader != leaderBroker
	                }
	              }
	            debug("topics not in preferred replica " + topicsNotInPreferredReplica)
	            //broker下所有的topic数量
	            val totalTopicPartitionsForBroker = topicAndPartitionsForBroker.size
	            val totalTopicPartitionsNotLedByBroker = topicsNotInPreferredReplica.size
	            imbalanceRatio = totalTopicPartitionsNotLedByBroker.toDouble / totalTopicPartitionsForBroker
	            trace("leader imbalance ratio for broker %d is %f".format(leaderBroker, imbalanceRatio))
	          }
	          // check ratio and if greater than desired ratio, trigger a rebalance for the topic partitions
	          // that need to be on this broker
	          if (imbalanceRatio > (config.leaderImbalancePerBrokerPercentage.toDouble / 100)) {
	            topicsNotInPreferredReplica.foreach {
	              case(topicPartition, replicas) => {
	                inLock(controllerContext.controllerLock) {
	                  // do this check only if the broker is live and there are no partitions being reassigned currently
	                  // and preferred replica election is not in progress
	                  if (controllerContext.liveBrokerIds.contains(leaderBroker) &&
	                      controllerContext.partitionsBeingReassigned.size == 0 &&
	                      controllerContext.partitionsUndergoingPreferredReplicaElection.size == 0 &&
	                      !deleteTopicManager.isTopicQueuedUpForDeletion(topicPartition.topic) &&
	                      controllerContext.allTopics.contains(topicPartition.topic)) {
	                    //重新选择leader
	                    onPreferredReplicaElection(Set(topicPartition), true)
	                  }
	                }
	              }
	            }
	          }
	        }
	      }
	    }
    }
至此，broker被选举为controller（leader）之后的操作就全部完成了。下文会讲broker在某些情况下重新被选举为leader之后的一些操作。