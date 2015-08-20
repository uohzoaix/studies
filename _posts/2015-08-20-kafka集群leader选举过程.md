---
layout: post
title: "kafka集群leader（controller）的选举过程"
description: ""
category: 
- kafka
tags: []
---


kafka集群启动时，KafkaServer会启动多个KafkaController（每个broker一个）：

    //在kafka集群启动时每个broker都会拥有一个kafkaController实例，但这时还未选出leader
    def startup() = {
	    inLock(controllerContext.controllerLock) {
	      info("Controller starting up");
	      registerSessionExpirationListener()
	      isRunning = true
	      controllerElector.startup
	      info("Controller startup complete")
	    }
    }

在调用controllerElector.startup后集群就开始通过zookeeper选举leader了：

	def startup {
	    inLock(controllerContext.controllerLock) {
	      //electionPath的data发生变化就会通知leaderChangeListener
	      controllerContext.zkClient.subscribeDataChanges(electionPath, leaderChangeListener)
	      elect
	    }
    }

LeaderChangeListener的定义为：

	class LeaderChangeListener extends IZkDataListener with Logging {
	    /**
	     * Called when the leader information stored in zookeeper has changed. Record the new leader in memory
	     * @throws Exception On any error.
	     */
	    @throws(classOf[Exception])
	    def handleDataChange(dataPath: String, data: Object) {
	      inLock(controllerContext.controllerLock) {
	        leaderId = KafkaController.parseControllerId(data.toString)
	        info("New leader is %d".format(leaderId))
	      }
	    }
	
	    /**
	     * Called when the leader information stored in zookeeper has been delete. Try to elect as the leader
	     * @throws Exception
	     *             On any error.
	     */
	    //在选举时发生错误时会删除path，这时会触发listener的该方法
	    @throws(classOf[Exception])
	    def handleDataDeleted(dataPath: String) {
	      inLock(controllerContext.controllerLock) {
	        debug("%s leader change listener fired for path %s to handle data deleted: trying to elect as a leader"
	          .format(brokerId, dataPath))
	        if(amILeader)
	          onResigningAsLeader()//被重新选为leader
	        elect
	      }
	    }
    }

在主要的elect方法中：

	def elect: Boolean = {
	    val timestamp = SystemTime.milliseconds.toString
	    val electString = Json.encode(Map("version" -> 1, "brokerid" -> brokerId, "timestamp" -> timestamp))
	   
	    leaderId = getControllerID 
	    /* 
	     * We can get here during the initial startup and the handleDeleted ZK callback. Because of the potential race condition, 
	     * it's possible that the controller has already been elected when we get here. This check will prevent the following 
	     * createEphemeralPath method from getting into an infinite loop if this broker is already the controller.
	     */
	    if(leaderId != -1) {
	       debug("Broker %d has been elected as leader, so stopping the election process.".format(leaderId))
	       return amILeader
	    }
	
	    try {
	      createEphemeralPathExpectConflictHandleZKBug(controllerContext.zkClient, electionPath, electString, brokerId,
	        (controllerString : String, leaderId : Any) => KafkaController.parseControllerId(controllerString) == leaderId.asInstanceOf[Int],
	        controllerContext.zkSessionTimeout)
	      info(brokerId + " successfully elected as leader")
	      leaderId = brokerId
	      onBecomingLeader()
	    } catch {
	      case e: ZkNodeExistsException =>
	        // If someone else has written the path, then
	        leaderId = getControllerID 
	
	        if (leaderId != -1)
	          debug("Broker %d was elected as leader instead of broker %d".format(leaderId, brokerId))
	        else
	          warn("A leader has been elected but just resigned, this will result in another round of election")
	
	      case e2: Throwable =>
	        error("Error while electing or becoming leader on broker %d".format(brokerId), e2)
	        resign()
	    }
	    amILeader
    }

broker只是将自身的数据发送给zookeeper的/controller的path中，这个时候就会触发上面的LeaderChangeListener的handleDataChange方法，该方法就会直接将该broker作为leader，这个时候其他的broker同样会调用elect方法，在发送数据到zookeeper之前会读取/controller的数据，如果发现有broker抢先成为leader了，则直接返回：

	leaderId = getControllerID 
    /* 
     * We can get here during the initial startup and the handleDeleted ZK callback. Because of the potential race condition, 
     * it's possible that the controller has already been elected when we get here. This check will prevent the following 
     * createEphemeralPath method from getting into an infinite loop if this broker is already the controller.
     */
    if(leaderId != -1) {
       debug("Broker %d has been elected as leader, so stopping the election process.".format(leaderId))
       return amILeader
    }

如果发现没有其他broker发送数据到zookeeper，那么将自身数据发送过去并成为leader，当在发送数据到zookeeper过程中出现Throwable异常时，会调用resign()方法：
	def resign() = {
    	leaderId = -1
    	deletePath(controllerContext.zkClient, electionPath)
	}
这时就会触发LeaderChangeListener的handleDataDeleted方法，该方法就会重新去选举leader。

从上面看出kafka集群的controller选举并没有很大的特点，只是将先来者看成为leader。发送到zookeeper只是为了让其他的broker知晓leader已经被选举出来了，你可以不用再去参加选举了。