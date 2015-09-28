---
layout: post
title: "Kafka集群消费数据"
description: ""
category: 
- kafka
tags: []
---


上上篇文章说到真正存储数据供客户端消费的类是KafkaStream这个类，该类的定义为：

	class KafkaStream[K,V](private val queue: BlockingQueue[FetchedDataChunk],
                        consumerTimeoutMs: Int,
                        private val keyDecoder: Decoder[K],
                        private val valueDecoder: Decoder[V],
                        val clientId: String)
    extends Iterable[MessageAndMetadata[K,V]] with java.lang.Iterable[MessageAndMetadata[K,V]]
可以看到它本身就是一个iterator，其中的queue是用来存储从broker fetch到的数据的。该类的iterator()方法返回的是ConsumerIterator实例：

	private val iter: ConsumerIterator[K,V] = new ConsumerIterator[K,V](queue, consumerTimeoutMs, keyDecoder, valueDecoder, clientId)
该遍历器会一直阻塞直至queue中有数据可以读取，当读取到shutdownCommand则停止读取数据，客户端消费数据代码流程：while(hasNext())——>next()，最终将逻辑转变为：while(IteratorTemplate.hasNext())——>IteratorTemplate.maybeComputeNext()——>ConsumerIterator.makeNext()——>ConsumerIterator.next()——>IteratorTemplate.next()。主要的makeNext()方法如下：

	protected def makeNext(): MessageAndMetadata[K, V] = {
	    var currentDataChunk: FetchedDataChunk = null
	    // if we don't have an iterator, get one
	    var localCurrent = current.get()
	    if(localCurrent == null || !localCurrent.hasNext) {
	      if (consumerTimeoutMs < 0)
	        //发出抓取数据请求
	        currentDataChunk = channel.take
	      else {
	        currentDataChunk = channel.poll(consumerTimeoutMs, TimeUnit.MILLISECONDS)
	        if (currentDataChunk == null) {
	          // reset state to make the iterator re-iterable
	          //没有数据则终止遍历器的状态为初始状态以便遍历器可以一直阻塞
	          resetState()
	          throw new ConsumerTimeoutException
	        }
	      }
	      //数据为shutdownCommand则退出遍历
	      if(currentDataChunk eq ZookeeperConsumerConnector.shutdownCommand) {
	        //只有在发出shutdown命令时才认为queue没有消息
	        debug("Received the shutdown command")
	        return allDone
	      } else {
	        currentTopicInfo = currentDataChunk.topicInfo
	        //当前抓取的offset
	        val cdcFetchOffset = currentDataChunk.fetchOffset
	        //上次抓取的offset
	        val ctiConsumeOffset = currentTopicInfo.getConsumeOffset
	        //offset以fetch的为主
	        if (ctiConsumeOffset < cdcFetchOffset) {
	          error("consumed offset: %d doesn't match fetch offset: %d for %s;\n Consumer may lose data"
	            .format(ctiConsumeOffset, cdcFetchOffset, currentTopicInfo))
	          currentTopicInfo.resetConsumeOffset(cdcFetchOffset)
	        }
	        localCurrent = currentDataChunk.messages.iterator
	
	        current.set(localCurrent)
	      }
	      // if we just updated the current chunk and it is empty that means the fetch size is too small!
	      if(currentDataChunk.messages.validBytes == 0)
	        throw new MessageSizeTooLargeException("Found a message larger than the maximum fetch size of this consumer on topic " +
	                                               "%s partition %d at fetch offset %d. Increase the fetch size, or decrease the maximum message size the broker will allow."
	                                               .format(currentDataChunk.topicInfo.topic, currentDataChunk.topicInfo.partitionId, currentDataChunk.fetchOffset))
	    }
	    var item = localCurrent.next()
	    // reject the messages that have already been consumed
	    while (item.offset < currentTopicInfo.getConsumeOffset && localCurrent.hasNext) {
	      item = localCurrent.next()
	    }
	    consumedOffset = item.nextOffset
	
	    item.message.ensureValid() // validate checksum of message to ensure it is valid
	
	    new MessageAndMetadata(currentTopicInfo.topic, currentTopicInfo.partitionId, item.message, item.offset, keyDecoder, valueDecoder)
    }
其中该遍历器有以下四种状态：

	class State
	object DONE extends State
	object READY extends State
	object NOT_READY extends State
	object FAILED extends State
初始状态为NOT_READY，正常消费过程中状态为READY，退出遍历为DONE，消费过程出错状态为FAILED。在正常消费过程中状态是在NOT_READY和READY之间来回切换的：

	def hasNext(): Boolean = {
	    if(state == FAILED)
	      throw new IllegalStateException("Iterator is in failed state")
	    state match {
	        //当读取到shutdownCommand时，DONE被设置为true
	      case DONE => false
	        //不管有没有消息都是ready状态，保证了shallow iterator
	      case READY => true
	      case _ => maybeComputeNext()
	    }
    }
    
    def next(): T = {
	    if(!hasNext())
	      throw new NoSuchElementException()
	    state = NOT_READY
	    if(nextItem == null)
	      throw new IllegalStateException("Expected item but none found.")
	    nextItem
    }
    
    def maybeComputeNext(): Boolean = {
	    state = FAILED
	    nextItem = makeNext()
	    if(state == DONE) {
	      false
	    } else {
	      state = READY
	      true
	    }
    }
即当调用hasNext()方法时，初始状态为NOT_READY，通过case _ => maybeComputeNext()将状态设为READY，最后调用next()方法又将状态设为NOT_READY，这样保证了遍历器可以一直遍历下去。

最后，留一个疑问，上面说到的queue，是谁在什么时候会把数据放到这个queue里呢？