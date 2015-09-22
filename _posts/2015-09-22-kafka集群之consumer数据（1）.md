---
layout: post
title: "Kafka集群消费数据前准备工作"
description: ""
category: 
- kafka
tags: []
---


kafka在消费数据时会通过Consumer对象创建ConsumerConnector，创建好ConsumerConnector之后会通过它创建KafkaStream，KafkaStream可以理解为一个数据流，数据的产出就是通过它完成，具体如何消费数据下篇文章再详细解析。这篇文章主要讲一下ConsumerConnector。

ConsumerConnector的一个子类为ZookeeperConsumerConnector，主要用来维护消费者与zookeeper之间的交互信息：

（1）每个消费者在一个消费者组中有唯一的id，生成方式为：

	val consumerIdString = {
	    var consumerUuid : String = null
	    config.consumerId match {
	      case Some(consumerId) // for testing only
	      => consumerUuid = consumerId
	      case None // generate unique consumerId automatically
	      => val uuid = UUID.randomUUID()
	      consumerUuid = "%s-%d-%s".format(
	        InetAddress.getLocalHost.getHostName, System.currentTimeMillis,
	        uuid.getMostSignificantBits().toHexString.substring(0,8))
	    }
	    config.groupId + "_" + consumerUuid
    }
这个id在zookeeper中存放的路径为/consumers/[group_id]/ids/[consumer_id] -> topic1,...topicN，每个消费者会将它的id注册为临时znode并且将它所消费的topic设置为znode的值。

（2）在消费时，每个topic的partition只能被一个消费者组中的唯一的一个消费者消费（即一个partition可以被多个消费者消费，但这多个消费者必须在不同的消费者组中），partition对应消费者的关系为：/consumers/[group_id]/owner/[topic]/[broker_id-partition_id] --> consumer_node_id。

（3）上面提到，一个partition可以被不同的消费者组中的不同消费者消费，所以不同的消费者组必须维护他们各自对该partition消费的最新的offset，这个offset和partition的关系为：/consumers/[group_id]/offsets/[topic]/[broker_id-partition_id] --> offset_counter_value。

接下来该ConsumerConnector会进行连接zk，创建抓取数据线程等操作：

	//连接zk
    connectZk()
    //初始化ConsumerFetcherManager，它是管理数据抓取线程的
    createFetcher()
    //管理与offset相关操作的连接
    ensureOffsetManagerConnected()

    //定时提交offset
    if (config.autoCommitEnable) {
	    scheduler.startup
	    info("starting auto committer every " + config.autoCommitIntervalMs + " ms")
	    scheduler.schedule("kafka-consumer-autocommit",
	                       autoCommit,
	                       delay = config.autoCommitIntervalMs,
	                       period = config.autoCommitIntervalMs,
	                       unit = TimeUnit.MILLISECONDS)
    }
抓取数据的线程是通过ConsumerFetcherManager类来管理的，createFetcher()方法并不立即创建数据抓取线程，而是在createMessageStreams方法中初始化该线程，该方法留到后面再讲。

ensureOffsetManagerConnected()方法主要是创建与管理offset的broker之间的socket连接，主要通过channelToOffsetManager方法完成创建：

	//该方法其实就是为了建立与group对应的partition所在的broker之间的连接
    def channelToOffsetManager(group: String, zkClient: ZkClient, socketTimeoutMs: Int = 3000, retryBackOffMs: Int = 1000) = {
     //与任意一台broker之间创建BlockingChannel
     var queryChannel = channelToAnyBroker(zkClient)

     var offsetManagerChannelOpt: Option[BlockingChannel] = None

     while (!offsetManagerChannelOpt.isDefined) {

       var coordinatorOpt: Option[BrokerEndPoint] = None

       while (!coordinatorOpt.isDefined) {
         try {
           if (!queryChannel.isConnected)
             queryChannel = channelToAnyBroker(zkClient)
           debug("Querying %s:%d to locate offset manager for %s.".format(queryChannel.host, queryChannel.port, group))
           //向broker集群发送获取指定group对应的partition的消费者信息
           queryChannel.send(ConsumerMetadataRequest(group))
           val response = queryChannel.receive()
           val consumerMetadataResponse =  ConsumerMetadataResponse.readFrom(response.buffer)
           debug("Consumer metadata response: " + consumerMetadataResponse.toString)
           if (consumerMetadataResponse.errorCode == ErrorMapping.NoError)
             coordinatorOpt = consumerMetadataResponse.coordinatorOpt
           else {
             debug("Query to %s:%d to locate offset manager for %s failed - will retry in %d milliseconds."
                  .format(queryChannel.host, queryChannel.port, group, retryBackOffMs))
             Thread.sleep(retryBackOffMs)
           }
         }
         catch {
           case ioe: IOException =>
             info("Failed to fetch consumer metadata from %s:%d.".format(queryChannel.host, queryChannel.port))
             queryChannel.disconnect()
         }
       }

       val coordinator = coordinatorOpt.get
       //partition的leader对应的broker与连接的broker一样则将之前创建的BlockingChannel直接作为offsetManagerChannel
       if (coordinator.host == queryChannel.host && coordinator.port == queryChannel.port) {
         offsetManagerChannelOpt = Some(queryChannel)
       } else {
         //否则重新创建与broker之间的连接
         val connectString = "%s:%d".format(coordinator.host, coordinator.port)
         var offsetManagerChannel: BlockingChannel = null
         try {
           debug("Connecting to offset manager %s.".format(connectString))
           offsetManagerChannel = new BlockingChannel(coordinator.host, coordinator.port,
                                                      BlockingChannel.UseDefaultBufferSize,
                                                      BlockingChannel.UseDefaultBufferSize,
                                                      socketTimeoutMs)
           offsetManagerChannel.connect()
           offsetManagerChannelOpt = Some(offsetManagerChannel)
           //关闭之前创建的与任意一台broker之间的连接
           queryChannel.disconnect()
         }
         catch {
           case ioe: IOException => // offsets manager may have moved
             info("Error while connecting to %s.".format(connectString))
             if (offsetManagerChannel != null) offsetManagerChannel.disconnect()
             Thread.sleep(retryBackOffMs)
             offsetManagerChannelOpt = None // just in case someone decides to change shutdownChannel to not swallow exceptions
         }
       }
     }

     offsetManagerChannelOpt.get
    }
定时提交offset逻辑很简单，如果存储介质是zookeeper则直接写到zookeeper中，如果存储介质是kafka则通过上面创建的BlockingChannel写到文件中。需要注意的是dual.commit.enabled这个配置选项，默认情况下如果offsets.storage设置为kafka则改选项为true，说明在写offset的时候既会写到kafka也会写到zookeeper，读offset时也是读取zookeeper和kafka中最大的那个。该选项是为了避免在从基于zookeeper存取offset迁移到基于kafka存取时产生的offset错误。如果在之后的情况中不会存在迁移的情况，那么该选项可以设置为false。

至此，ZookeeperConsumerConnector的初始化工作就完成了，接下来就通过该connector的createMessageStreams方法创建KafkaStream：

	def createMessageStreams[K,V](topicCountMap: Map[String,Int], keyDecoder: Decoder[K], valueDecoder: Decoder[V])
      : Map[String, List[KafkaStream[K,V]]] = {
	    if (messageStreamCreated.getAndSet(true))
	      throw new MessageStreamsExistException(this.getClass.getSimpleName +
	                                   " can create message streams at most once",null)
	    consume(topicCountMap, keyDecoder, valueDecoder)
    }
该方法的topicCountMap参数是topic对应的consumer数量，其中consume方法的定义如下：

	//该方法并不真正的读取数据，只是初始化存放数据的queue，真正消费数据的是对该queue进行shallow iterator（no stop）
    //在kafka的运行过程中，会有线程将数据放入partition对应的queue中
    def consume[K, V](topicCountMap: scala.collection.Map[String,Int], keyDecoder: Decoder[K], valueDecoder: Decoder[V])
      : Map[String,List[KafkaStream[K,V]]] = {
	    debug("entering consume ")
	    if (topicCountMap == null)
	      throw new RuntimeException("topicCountMap is null")
	
	    //创建制定数量的consumerid
	    val topicCount = TopicCount.constructTopicCount(consumerIdString, topicCountMap)
	
	    val topicThreadIds = topicCount.getConsumerThreadIdsPerTopic
	
	    // make a list of (queue,stream) pairs, one pair for each threadId
	    val queuesAndStreams = topicThreadIds.values.map(threadIdSet =>
	      threadIdSet.map(_ => {
	        val queue =  new LinkedBlockingQueue[FetchedDataChunk](config.queuedMaxMessages)
	        val stream = new KafkaStream[K,V](
	          queue, config.consumerTimeoutMs, keyDecoder, valueDecoder, config.clientId)
	        (queue, stream)
	      })
	    ).flatten.toList
	
	    val dirs = new ZKGroupDirs(config.groupId)
	    //将消费者信息写到zookeeper
	    registerConsumerInZK(dirs, consumerIdString, topicCount)
	    reinitializeConsumer(topicCount, queuesAndStreams)
	
	    //返回KafkaStream
	    loadBalancerListener.kafkaMessageAndMetadataStreams.asInstanceOf[Map[String, List[KafkaStream[K,V]]]]
    }
在该方法中核心代码是reinitializeConsumer(topicCount, queuesAndStreams)，在reinitializeConsumer方法中主要做以下几件事：

（1）将topic对应的消费者线程id及对应的LinkedBlockingQueue放入topicThreadIdAndQueues中，LinkedBlockingQueue是真正存放数据的queue，下面会对该queue进行详细的讲解。

（2）注册sessionExpirationListener，在session失效重新创建session时调用：

	def handleNewSession() {
      /**
       *  When we get a SessionExpired event, we lost all ephemeral nodes and zkclient has reestablished a
       *  connection for us. We need to release the ownership of the current consumer and re-register this
       *  consumer in the consumer registry and trigger a rebalance.
       */
      info("ZK expired; release old broker parition ownership; re-register consumer " + consumerIdString)
      //有新session创建，需要将原来的topic注册信息清除掉
      loadBalancerListener.resetState()
      //重新注册consumer到zk
      registerConsumerInZK(dirs, consumerIdString, topicCount)
      // explicitly trigger load balancing for this consumer
      //进行reblance
      loadBalancerListener.syncedRebalance()
      // There is no need to resubscribe to child and state changes.
      // The child change watchers will be set inside rebalance when we read the children list.
    }
（2）向/consumers/group/ids注册loadBalancerListener，当该path的子path发生变化时（即consumer增删）会调用handleChildChange，该方法会触发syncedRebalance方法：

	def syncedRebalance() {
      rebalanceLock synchronized {
        rebalanceTimer.time {
          for (i <- 0 until config.rebalanceMaxRetries) {
            if(isShuttingDown.get())  {
              return
            }
            info("begin rebalancing consumer " + consumerIdString + " try #" + i)
            var done = false
            var cluster: Cluster = null
            try {
              cluster = getCluster(zkClient)
              done = rebalance(cluster)
            } catch {
              case e: Throwable =>
                /** occasionally, we may hit a ZK exception because the ZK state is changing while we are iterating.
                  * For example, a ZK node can disappear between the time we get all children and the time we try to get
                  * the value of a child. Just let this go since another rebalance will be triggered.
                  **/
                info("exception during rebalance ", e)
            }
            info("end rebalancing consumer " + consumerIdString + " try #" + i)
            if (done) {
              return
            } else {
              /* Here the cache is at a risk of being stale. To take future rebalancing decisions correctly, we should
               * clear the cache */
              info("Rebalancing attempt failed. Clearing the cache before the next rebalancing operation is triggered")
            }
            //reblance出现问题需要暂时将读取消息的线程关闭以免重试出现数据重复的现象
            // stop all fetchers and clear all the queues to avoid data duplication
            closeFetchersForQueues(cluster, kafkaMessageAndMetadataStreams, topicThreadIdAndQueues.map(q => q._2))
            Thread.sleep(config.rebalanceBackoffMs)
          }
        }
      }

      throw new ConsumerRebalanceFailedException(consumerIdString + " can't rebalance after " + config.rebalanceMaxRetries +" retries")
    }
其中reblance方法主要做下面几件事情：

<1>关闭数据抓取线程，获取之前为topic设置的存放数据的queue并清空该queue

<2>为各个partition重新分配threadid
<3>获取partition最新的offset并重新初始化新的PartitionTopicInfo(topic,partition,queue,consumedOffset,fetchedOffset,new AtomicInteger(config.fetchMessageMaxBytes),config.clientId)，其中queue就是上面说的存放数据的那个queue，consumedOffset和fetchedOffset都为partition最新的offset。
<4>重新将partition对应的新的consumer信息写入zookeeper
<5>重新创建partition的fetcher线程

全部代码如下：

	private def rebalance(cluster: Cluster): Boolean = {
      val myTopicThreadIdsMap = TopicCount.constructTopicCount(
        group, consumerIdString, zkClient, config.excludeInternalTopics).getConsumerThreadIdsPerTopic
      //获取全部的broker进行reblance
      val brokers = getAllBrokersInCluster(zkClient)
      if (brokers.size == 0) {
        // This can happen in a rare case when there are no brokers available in the cluster when the consumer is started.
        // We log an warning and register for child changes on brokers/id so that rebalance can be triggered when the brokers
        // are up.
        warn("no brokers found when trying to rebalance.")
        //brokers/ids发生变化就会触发loadBalancerListener的handleChildChange方法
        zkClient.subscribeChildChanges(ZkUtils.BrokerIdsPath, loadBalancerListener)
        true
      }
      else {
        /**
         * fetchers must be stopped to avoid data duplication, since if the current
         * rebalancing attempt fails, the partitions that are released could be owned by another consumer.
         * But if we don't stop the fetchers first, this consumer would continue returning data for released
         * partitions in parallel. So, not stopping the fetchers leads to duplicate data.
         */
        //获取指定topic的消息队列，并清除该队列，以免数据冗余
        closeFetchers(cluster, kafkaMessageAndMetadataStreams, myTopicThreadIdsMap)
        if (consumerRebalanceListener != null) {
          info("Invoking rebalance listener before relasing partition ownerships.")
          consumerRebalanceListener.beforeReleasingPartitions(
            if (topicRegistry.size == 0)
              new java.util.HashMap[String, java.util.Set[java.lang.Integer]]
            else
              mapAsJavaMap(topicRegistry.map(topics =>
                topics._1 -> topics._2.keys
              ).toMap).asInstanceOf[java.util.Map[String, java.util.Set[java.lang.Integer]]]
          )
        }
        releasePartitionOwnership(topicRegistry)
        val assignmentContext = new AssignmentContext(group, consumerIdString, config.excludeInternalTopics, zkClient)
        //为各个partition重新分配threadid
        val globalPartitionAssignment = partitionAssignor.assign(assignmentContext)
        //当前consumerid持有的topicpartition——>threadids
        val partitionAssignment = globalPartitionAssignment.get(assignmentContext.consumerId)
        val currentTopicRegistry = new Pool[String, Pool[Int, PartitionTopicInfo]](
          valueFactory = Some((topic: String) => new Pool[Int, PartitionTopicInfo]))

        // fetch current offsets for all topic-partitions
        val topicPartitions = partitionAssignment.keySet.toSeq

        //获取各个partition的offset
        val offsetFetchResponseOpt = fetchOffsets(topicPartitions)

        if (isShuttingDown.get || !offsetFetchResponseOpt.isDefined)
          false
        else {
          val offsetFetchResponse = offsetFetchResponseOpt.get
          topicPartitions.foreach(topicAndPartition => {
            val (topic, partition) = topicAndPartition.asTuple
            val offset = offsetFetchResponse.requestInfo(topicAndPartition).offset
            val threadId = partitionAssignment(topicAndPartition)
            //将topic及其partition信息存入topicRegistry，并设置存放数据的queue（即存放消费partition数据的queue）
            addPartitionTopicInfo(currentTopicRegistry, partition, topic, offset, threadId)
          })

          /**
           * move the partition ownership here, since that can be used to indicate a truly successful rebalancing attempt
           * A rebalancing attempt is completed successfully only after the fetchers have been started correctly
           */
          //重新将partition对应的新的consumer信息写入zookeeper
          if(reflectPartitionOwnershipDecision(partitionAssignment)) {
            allTopicsOwnedPartitionsCount = partitionAssignment.size

            partitionAssignment.view.groupBy { case(topicPartition, consumerThreadId) => topicPartition.topic }
                                      .foreach { case (topic, partitionThreadPairs) =>
              newGauge("OwnedPartitionsCount",
                new Gauge[Int] {
                  def value() = partitionThreadPairs.size
                },
                ownedPartitionsCountMetricTags(topic))
            }

            topicRegistry = currentTopicRegistry
            // Invoke beforeStartingFetchers callback if the consumerRebalanceListener is set.
            if (consumerRebalanceListener != null) {
              info("Invoking rebalance listener before starting fetchers.")

              // Partition assignor returns the global partition assignment organized as a map of [TopicPartition, ThreadId]
              // per consumer, and we need to re-organize it to a map of [Partition, ThreadId] per topic before passing
              // to the rebalance callback.
              val partitionAssginmentGroupByTopic = globalPartitionAssignment.values.flatten.groupBy[String] {
                case (topicPartition, _) => topicPartition.topic
              }
              val partitionAssigmentMapForCallback = partitionAssginmentGroupByTopic.map({
                case (topic, partitionOwnerShips) =>
                  val partitionOwnershipForTopicScalaMap = partitionOwnerShips.map({
                    case (topicAndPartition, consumerThreadId) =>
                      topicAndPartition.partition -> consumerThreadId
                  })
                  topic -> mapAsJavaMap(collection.mutable.Map(partitionOwnershipForTopicScalaMap.toSeq:_*))
                    .asInstanceOf[java.util.Map[java.lang.Integer, ConsumerThreadId]]
              })
              consumerRebalanceListener.beforeStartingFetchers(
                consumerIdString,
                mapAsJavaMap(collection.mutable.Map(partitionAssigmentMapForCallback.toSeq:_*))
              )
            }
            updateFetcher(cluster)
            true
          } else {
            false
          }
        }
      }
    }

（3）向/brokers/topics/topic注册topicPartitionChangeListener，在topic数据发生变化时调用：

	def handleDataChange(dataPath : String, data: Object) {
      try {
        info("Topic info for path " + dataPath + " changed to " + data.toString + ", triggering rebalance")
        // queue up the rebalance event
        //触发loadBlance线程进行reblance
        loadBalancerListener.rebalanceEventTriggered()
        // There is no need to re-subscribe the watcher since it will be automatically
        // re-registered upon firing of this event by zkClient
      } catch {
        case e: Throwable => error("Error while handling topic partition change for data path " + dataPath, e )
      }
    }
（4）显式调用loadBalancerListener.syncedRebalance()，即调用上面的reblance方法进行consumer的初始化工作

消费数据的准备工作就是这些，一句话概括就是：为指定topic的各个partition创建consumer线程。下文会具体讲数据是如何消费的。
