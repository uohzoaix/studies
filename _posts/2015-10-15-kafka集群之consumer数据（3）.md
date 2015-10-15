---
layout: post
title: "Kafka集群消费数据续"
description: ""
category: 
- kafka
tags: []
---


上篇文章留了一个疑问：数据由谁在什么时候放入queue中。这里来解答一下。

之前说到在ZookeeperConsumerConnector类中初始化consumer时会初始化存放数据的queue，我们再来看下初始化之后的操作：

	private def addPartitionTopicInfo(currentTopicRegistry: Pool[String, Pool[Int, PartitionTopicInfo]],
                                      partition: Int, topic: String,
                                      offset: Long, consumerThreadId: ConsumerThreadId) {
	      val partTopicInfoMap = currentTopicRegistry.getAndMaybePut(topic)
	
	      //queue用来存放数据
	      val queue = topicThreadIdAndQueues.get((topic, consumerThreadId))
	      val consumedOffset = new AtomicLong(offset)
	      val fetchedOffset = new AtomicLong(offset)
	      val partTopicInfo = new PartitionTopicInfo(topic,
	                                                 partition,
	                                                 queue,
	                                                 consumedOffset,
	                                                 fetchedOffset,
	                                                 new AtomicInteger(config.fetchMessageMaxBytes),
	                                                 config.clientId)
	      partTopicInfoMap.put(partition, partTopicInfo)
	      debug(partTopicInfo + " selected new offset " + offset)
	      checkpointedZkOffsets.put(TopicAndPartition(topic, partition), offset)
	    }
    }
其中val queue = topicThreadIdAndQueues.get((topic, consumerThreadId))就是初始化的queue，可以看出某个topic的一个消费者只对应一个queue，随后会实例化PartitionTopicInfo，即指定的queue只用来存放topic的某个partition接收到的数据。

回到之前说到的为partition创建fetcher数据线程：

	override def createFetcherThread(fetcherId: Int, sourceBroker: BrokerEndPoint): AbstractFetcherThread = {
    new ConsumerFetcherThread(
      "ConsumerFetcherThread-%s-%d-%d".format(consumerIdString, fetcherId, sourceBroker.id),
      config, sourceBroker, partitionMap, this)
    }
ConsumerFetcherThread类继承了AbstractFetcherThread类，AbstractFetcherThread的doWork()方法：

	override def doWork() {

	    inLock(partitionMapLock) {
	      partitionMap.foreach {
	        case((topicAndPartition, partitionFetchState)) =>
	          if(partitionFetchState.isActive)
	            fetchRequestBuilder.addFetch(topicAndPartition.topic, topicAndPartition.partition,
	              partitionFetchState.offset, fetchSize)
	      }
	    }
	
	    val fetchRequest = fetchRequestBuilder.build()
	
	    if (!fetchRequest.requestInfo.isEmpty)
	      //处理请求
	      processFetchRequest(fetchRequest)
	    else {
	      trace("There are no active partitions. Back off for %d ms before sending a fetch request".format(fetchBackOffMs))
	      partitionMapCond.await(fetchBackOffMs, TimeUnit.MILLISECONDS)
	    }
    }
在processFetchRequest方法中首先通过SimpleConsumer.fetch()方法向topic请求数据，接着会将请求到的数据通过ConsumerFetcherThread.processPartitionData方法将具体数据放入上面说到的queue中：

	def processPartitionData(topicAndPartition: TopicAndPartition, fetchOffset: Long, partitionData: FetchResponsePartitionData) {
    val pti = partitionMap(topicAndPartition)
    if (pti.getFetchOffset != fetchOffset)
      throw new RuntimeException("Offset doesn't match for partition [%s,%d] pti offset: %d fetch offset: %d"
                                .format(topicAndPartition.topic, topicAndPartition.partition, pti.getFetchOffset, fetchOffset))
    pti.enqueue(partitionData.messages.asInstanceOf[ByteBufferMessageSet])
    }
pti.enqueue方法的定义：

	/**
     * Enqueue a message set for processing.
     */
    def enqueue(messages: ByteBufferMessageSet) {
	    val size = messages.validBytes
	    if(size > 0) {
	      val next = messages.shallowIterator.toSeq.last.nextOffset
	      trace("Updating fetch offset = " + fetchedOffset.get + " to " + next)
	      //这里的chunkQueue就是在实例化PartitionTopicInfo时传入的queue，可以查看ZookeeperConsumerConnector.addPartitionTopicInfo方法
	      chunkQueue.put(new FetchedDataChunk(messages, this, fetchedOffset.get))
	      fetchedOffset.set(next)
	      debug("updated fetch offset of (%s) to %d".format(this, next))
	      consumerTopicStats.getConsumerTopicStats(topic).byteRate.mark(size)
	      consumerTopicStats.getConsumerAllTopicStats().byteRate.mark(size)
	    } else if(messages.sizeInBytes > 0) {
	      chunkQueue.put(new FetchedDataChunk(messages, this, fetchedOffset.get))
	    }
    }
这样就实现了数据从无到有的过程。

到这里基本上差不多已经清楚consumer消费数据是怎么来的了，不过还有两个问题：
1.在AbstractFetcherThread类的doWord()方法那些fetch请求是怎么来的？

2.通过SimpleConsumer如何获取数据？

对于第一个问题，在创建fetcher线程时会默认将当前消费的offset放入partitionMap(这个就是在doWork方法里存放请求的map)：

	fetcherThreadMap(brokerAndFetcherId).addPartitions(partitionAndOffsets.map { case (topicAndPartition, brokerAndInitOffset) =>
          topicAndPartition -> brokerAndInitOffset.initOffset
        })
    //具体的存放请求方法
    def addPartitions(partitionAndOffsets: Map[TopicAndPartition, Long]) {
    partitionMapLock.lockInterruptibly()
	    try {
	      for ((topicAndPartition, offset) <- partitionAndOffsets) {
	        // If the partitionMap already has the topic/partition, then do not update the map with the old offset
	        if (!partitionMap.contains(topicAndPartition))
	          partitionMap.put(
	            topicAndPartition,
	            if (PartitionTopicInfo.isOffsetInvalid(offset)) new PartitionFetchState(handleOffsetOutOfRange(topicAndPartition))
	            else new PartitionFetchState(offset)
	          )}
	      partitionMapCond.signalAll()
	    } finally {
	      partitionMapLock.unlock()
	    }
    }
第二个问题先看下SimpleConsumer的sendRequest方法：

	private def sendRequest(request: RequestOrResponse): Receive = {
	    lock synchronized {
	      var response: Receive = null
	      try {
	        //获取与broker之间的连接
	        getOrMakeConnection()
	        //向broker发送请求
	        blockingChannel.send(request)
	        //阻塞获取broker返回的消息
	        response = blockingChannel.receive()
	      } catch {
	        case e : Throwable =>
	          info("Reconnect due to socket error: %s".format(e.toString))
	          // retry once
	          try {
	            reconnect()
	            blockingChannel.send(request)
	            response = blockingChannel.receive()
	          } catch {
	            case e: Throwable =>
	              disconnect()
	              throw e
	          }
	      }
	      response
	    }
将请求发送到broker后最后会由KafkaApis来处理相应的请求：

	def handleFetchRequest(request: RequestChannel.Request) {
	    val fetchRequest = request.requestObj.asInstanceOf[FetchRequest]
	
	    // the callback for sending a fetch response
	    def sendResponseCallback(responsePartitionData: Map[TopicAndPartition, FetchResponsePartitionData]) {
	      responsePartitionData.foreach { case (topicAndPartition, data) =>
	        // we only print warnings for known errors here; if it is unknown, it will cause
	        // an error message in the replica manager already and hence can be ignored here
	        if (data.error != ErrorMapping.NoError && data.error != ErrorMapping.UnknownCode) {
	          debug("Fetch request with correlation id %d from client %s on partition %s failed due to %s"
	            .format(fetchRequest.correlationId, fetchRequest.clientId,
	            topicAndPartition, ErrorMapping.exceptionNameFor(data.error)))
	        }
	
	        // record the bytes out metrics only when the response is being sent
	        BrokerTopicStats.getBrokerTopicStats(topicAndPartition.topic).bytesOutRate.mark(data.messages.sizeInBytes)
	        BrokerTopicStats.getBrokerAllTopicsStats().bytesOutRate.mark(data.messages.sizeInBytes)
	      }
	
	      val response = FetchResponse(fetchRequest.correlationId, responsePartitionData)
	      requestChannel.sendResponse(new RequestChannel.Response(request, new FetchResponseSend(response)))
	    }
	
	    // call the replica manager to fetch messages from the local replica
	    replicaManager.fetchMessages(
	      fetchRequest.maxWait.toLong,
	      fetchRequest.replicaId,
	      fetchRequest.minBytes,
	      fetchRequest.requestInfo,
	      sendResponseCallback)
    }
对于KafkaApis如何获取相应的请求就不再说了，在之前的讲解SocketServer那篇文章就说过了。

总结：kafka的设计真的很完美，通过一个总控控制各个不同的请求，看它的代码会很开心。

全文完:)