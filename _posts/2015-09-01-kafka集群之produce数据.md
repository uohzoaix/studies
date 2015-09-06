---
layout: post
title: "Kafka集群生产数据"
description: ""
category: 
- kafka
tags: []
---

kafka生产数据主要通过Producer类完成，在生产数据时有两种方式可以使用：sync及async，顾名思义，sync方式是一旦有数据产生就马上进行处理，async是将产生的数据放入一个队列中，等待相关线程（ProducerSendThread）去批量处理数据。这两种方式最终都会由DefaultEventHandler的handle方法处理产生的数据。

在async方式中，将产生的数据放入queue时有三种不同的放入方式：

1）当queue.enqueue.timeout.ms=0，则立即放入queue中并返回true，若queue已满，则立即返回false

2）当queue.enqueue.timeout.ms<0，则立即放入queue，若queue已满，则一直等待queue释放空间

3）当queue.enqueue.timeout.ms>0，则立即放入queue中并返回true，若queue已满，则等待queue.enqueue.timeout.ms指定的时间以让queue释放空间，若时间到queue还是没有足够空间，则立即返回false

ProducerSendThread线程会不断地从queue中取出数据直到取出的数据为shutdownCommand，在取出数据后，如果取出的数据总量达到设置的batch.num.messages属性的值或者取出的数据为null则立即处理取出的所有数据。

最后真正处理数据由DefaultEventHandler.handle()方法进行：

	def handle(events: Seq[KeyedMessage[K,V]]) {
		//序列化数据，分别由keyEncoder和valueEncoder对key和value进行序列化
	    val serializedData = serialize(events)
	    serializedData.foreach {
	      keyed =>
	        val dataSize = keyed.message.payloadSize
	        producerTopicStats.getProducerTopicStats(keyed.topic).byteRate.mark(dataSize)
	        producerTopicStats.getProducerAllTopicsStats.byteRate.mark(dataSize)
	    }
	    var outstandingProduceRequests = serializedData
	    var remainingRetries = config.messageSendMaxRetries + 1
	    val correlationIdStart = correlationId.get()
	    debug("Handling %d events".format(events.size))
	    //重试次数达到一定次数退出不再发送
	    while (remainingRetries > 0 && outstandingProduceRequests.size > 0) {
	      topicMetadataToRefresh ++= outstandingProduceRequests.map(_.topic)
	      //超过刷新间隔
	      if (topicMetadataRefreshInterval >= 0 &&
	          SystemTime.milliseconds - lastTopicMetadataRefreshTime > topicMetadataRefreshInterval) {
	        //更新topic信息
	        CoreUtils.swallowError(brokerPartitionInfo.updateInfo(topicMetadataToRefresh.toSet, correlationId.getAndIncrement))
	        //sendPartitionPerTopicCache存储的是上次将数据发送到了哪个partition，存储起来是为了在一段时间内能够一直发送到该partition，不会频繁的去取模决定发送到哪个partition，一段时间后clear掉该数据是为了能够使数据分布均匀
	        sendPartitionPerTopicCache.clear()
	        topicMetadataToRefresh.clear
	        lastTopicMetadataRefreshTime = SystemTime.milliseconds
	      }
	      outstandingProduceRequests = dispatchSerializedData(outstandingProduceRequests)
	      //发送之后如果还有数据未发送则sendPartitionPerTopicCache.clear()，确保这次发送的partition和上次的partition不一样，达到均衡分布的目的
	      if (outstandingProduceRequests.size > 0) {
	        //重试发送
	        info("Back off for %d ms before retrying send. Remaining retries = %d".format(config.retryBackoffMs, remainingRetries-1))
	        // back off and update the topic metadata cache before attempting another send operation
	        Thread.sleep(config.retryBackoffMs)
	        // get topics of the outstanding produce requests and refresh metadata for those
	        CoreUtils.swallowError(brokerPartitionInfo.updateInfo(outstandingProduceRequests.map(_.topic).toSet, correlationId.getAndIncrement))
	        sendPartitionPerTopicCache.clear()
	        remainingRetries -= 1
	        producerStats.resendRate.mark()
	      }
	    }
	    if(outstandingProduceRequests.size > 0) {
	      producerStats.failedSendRate.mark()
	      val correlationIdEnd = correlationId.get()
	      error("Failed to send requests for topics %s with correlation ids in [%d,%d]"
	        .format(outstandingProduceRequests.map(_.topic).toSet.mkString(","),
	        correlationIdStart, correlationIdEnd-1))
	      throw new FailedToSendMessageException("Failed to send messages after " + config.messageSendMaxRetries + " tries.", null)
	    }
    }
真正发送数据的方法是dispatchSerializedData：

	private def dispatchSerializedData(messages: Seq[KeyedMessage[K,Message]]): Seq[KeyedMessage[K, Message]] = {
	    val partitionedDataOpt = partitionAndCollate(messages)
	    partitionedDataOpt match {
	      case Some(partitionedData) =>
	        val failedProduceRequests = new ArrayBuffer[KeyedMessage[K, Message]]
	        for ((brokerid, messagesPerBrokerMap) <- partitionedData) {
	          if (logger.isTraceEnabled) {
	            messagesPerBrokerMap.foreach(partitionAndEvent =>
	              trace("Handling event for Topic: %s, Broker: %d, Partitions: %s".format(partitionAndEvent._1, brokerid, partitionAndEvent._2)))
	          }
	          val messageSetPerBrokerOpt = groupMessagesToSet(messagesPerBrokerMap)
	          messageSetPerBrokerOpt match {
	            case Some(messageSetPerBroker) =>
	              //发送到broker对应的partition中
	              val failedTopicPartitions = send(brokerid, messageSetPerBroker)
	              failedTopicPartitions.foreach(topicPartition => {
	                messagesPerBrokerMap.get(topicPartition) match {
	                    //将发送有误的topicPartition对应的数据重新放入failedProduceRequests
	                  case Some(data) => failedProduceRequests.appendAll(data)
	                  case None => // nothing
	                }
	              })
	            case None => // failed to group messages
	              messagesPerBrokerMap.values.foreach(m => failedProduceRequests.appendAll(m))
	          }
	        }
	        failedProduceRequests
	      case None => // failed to collate messages
	        messages
	    }
    }
partitionAndCollate方法是将数据决定放入topic的哪个partition中：

	//将数据放入topic的指定partition对应的arrayBuffer中
    def partitionAndCollate(messages: Seq[KeyedMessage[K,Message]]): Option[Map[Int, collection.mutable.Map[TopicAndPartition, Seq[KeyedMessage[K,Message]]]]] = {
	    val ret = new HashMap[Int, collection.mutable.Map[TopicAndPartition, Seq[KeyedMessage[K,Message]]]]
	    try {
	      for (message <- messages) {
	        //获取topic的所有partition
	        val topicPartitionsList = getPartitionListForTopic(message)
	        //决定将数据放在哪个partition，如果sendPartitionPerTopicCache有相应的数据（表示未到刷新时间）则直接返回相应的partition，否则通过取模算出并将结果放入sendPartitionPerTopicCache
	        val partitionIndex = getPartition(message.topic, message.partitionKey, topicPartitionsList)
	        val brokerPartition = topicPartitionsList(partitionIndex)
	
	        // postpone the failure until the send operation, so that requests for other brokers are handled correctly
	        val leaderBrokerId = brokerPartition.leaderBrokerIdOpt.getOrElse(-1)
	
	        var dataPerBroker: HashMap[TopicAndPartition, Seq[KeyedMessage[K,Message]]] = null
	        ret.get(leaderBrokerId) match {
	          case Some(element) =>
	            dataPerBroker = element.asInstanceOf[HashMap[TopicAndPartition, Seq[KeyedMessage[K,Message]]]]
	          case None =>
	            dataPerBroker = new HashMap[TopicAndPartition, Seq[KeyedMessage[K,Message]]]
	            ret.put(leaderBrokerId, dataPerBroker)
	        }
	
	        //将数据放入leader的指定partition中
	        val topicAndPartition = TopicAndPartition(message.topic, brokerPartition.partitionId)
	        var dataPerTopicPartition: ArrayBuffer[KeyedMessage[K,Message]] = null
	        dataPerBroker.get(topicAndPartition) match {
	          case Some(element) =>
	            dataPerTopicPartition = element.asInstanceOf[ArrayBuffer[KeyedMessage[K,Message]]]
	          case None =>
	            dataPerTopicPartition = new ArrayBuffer[KeyedMessage[K,Message]]
	            dataPerBroker.put(topicAndPartition, dataPerTopicPartition)
	        }
	        dataPerTopicPartition.append(message)
	      }
	      Some(ret)
	    }catch {    // Swallow recoverable exceptions and return None so that they can be retried.
	      case ute: UnknownTopicOrPartitionException => warn("Failed to collate messages by topic,partition due to: " + ute.getMessage); None
	      case lnae: LeaderNotAvailableException => warn("Failed to collate messages by topic,partition due to: " + lnae.getMessage); None
	      case oe: Throwable => error("Failed to collate messages by topic, partition due to: " + oe.getMessage); None
	    }
    }
在发送到指定的broker时需要将发送有错误的数据重新获取到以重新发送：

	//返回发送有错误的topicPartition
    private def send(brokerId: Int, messagesPerTopic: collection.mutable.Map[TopicAndPartition, ByteBufferMessageSet]) = {
	    if(brokerId < 0) {
	      warn("Failed to send data since partitions %s don't have a leader".format(messagesPerTopic.map(_._1).mkString(",")))
	      messagesPerTopic.keys.toSeq
	    } else if(messagesPerTopic.size > 0) {
	      val currentCorrelationId = correlationId.getAndIncrement
	      val producerRequest = new ProducerRequest(currentCorrelationId, config.clientId, config.requestRequiredAcks,
	        config.requestTimeoutMs, messagesPerTopic)
	      var failedTopicPartitions = Seq.empty[TopicAndPartition]
	      try {
	        //获取broker对应的SyncProducer（在启动broker时创建）
	        val syncProducer = producerPool.getProducer(brokerId)
	        debug("Producer sending messages with correlation id %d for topics %s to broker %d on %s:%d"
	          .format(currentCorrelationId, messagesPerTopic.keySet.mkString(","), brokerId, syncProducer.config.host, syncProducer.config.port))
	        val response = syncProducer.send(producerRequest)
	        debug("Producer sent messages with correlation id %d for topics %s to broker %d on %s:%d"
	          .format(currentCorrelationId, messagesPerTopic.keySet.mkString(","), brokerId, syncProducer.config.host, syncProducer.config.port))
	        if(response != null) {
	          //有数据未发送过去
	          if (response.status.size != producerRequest.data.size)
	            throw new KafkaException("Incomplete response (%s) for producer request (%s)".format(response, producerRequest))
	          if (logger.isTraceEnabled) {
	            val successfullySentData = response.status.filter(_._2.error == ErrorMapping.NoError)
	            successfullySentData.foreach(m => messagesPerTopic(m._1).foreach(message =>
	              trace("Successfully sent message: %s".format(if(message.message.isNull) null else message.message.toString()))))
	          }
	          //发送有错误或者处理有误的数据
	          val failedPartitionsAndStatus = response.status.filter(_._2.error != ErrorMapping.NoError).toSeq
	          failedTopicPartitions = failedPartitionsAndStatus.map(partitionStatus => partitionStatus._1)
	          if(failedTopicPartitions.size > 0) {
	            val errorString = failedPartitionsAndStatus
	              .sortWith((p1, p2) => p1._1.topic.compareTo(p2._1.topic) < 0 ||
	                                    (p1._1.topic.compareTo(p2._1.topic) == 0 && p1._1.partition < p2._1.partition))
	              .map{
	                case(topicAndPartition, status) =>
	                  topicAndPartition.toString + ": " + ErrorMapping.exceptionFor(status.error).getClass.getName
	              }.mkString(",")
	            warn("Produce request with correlation id %d failed due to %s".format(currentCorrelationId, errorString))
	          }
	          failedTopicPartitions
	        } else {
	          Seq.empty[TopicAndPartition]
	        }
	      } catch {
	        case t: Throwable =>
	          warn("Failed to send producer request with correlation id %d to broker %d with data for partitions %s"
	            .format(currentCorrelationId, brokerId, messagesPerTopic.map(_._1).mkString(",")), t)
	          messagesPerTopic.keys.toSeq
	      }
	    } else {
	      List.empty
	    }
    }
至此，数据的生产已经完成，但该过程并不涉及写磁盘操作，只是将批量的请求发送到socket（BlockingChannel）中，真正的持久化操作会在网络层接收到请求后再由具体的类来完成（详细过程会在讲KafkaApi的时候说到）。
