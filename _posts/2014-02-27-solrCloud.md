---
layout: post
title: "solrcloud代码分析"
category: 
- solrCloud
tags: []
---






前几天把公司的solr配置成了solrcloud，好处就不说了，可是发现主节点和从节点的时间字段不一样，其他字段的值都是相同的，一开始觉得很诡异，网上搜索相关的问题也没有找到答案，只能看看源码了。由于solrcloud是通过zookeeper来进行管理的，不是跑在eclipse上的，所以想要调试基本上是不可能的，我只能通过猜测来定位了。</br></br>
在发起索引建立的请求后首先会通过filter然后来到DataImportHandler的hadleRequestBody(req,rsp)方法，主要代码如下：
{% highlight objc %}
if (DataImporter.SHOW_CONF_CMD.equals(command)) {
    //显示solr的配置文件信息没啥说的
    String dataConfigFile = params.get("config");
    String dataConfig = params.get("dataConfig");
    if(dataConfigFile != null) {
        dataConfig = SolrWriter.getResourceAsString(req.getCore().getResourceLoader().openResource(dataConfigFile));
    }
    if(dataConfig==null)  {
        rsp.add("status", DataImporter.MSG.NO_CONFIG_FOUND);
    } else {
        // Modify incoming request params to add wt=raw
        ModifiableSolrParams rawParams = new ModifiableSolrParams(req.getParams());
        rawParams.set(CommonParams.WT, "raw");
        req.setParams(rawParams);
        ContentStreamBase content = new ContentStreamBase.StringStream(dataConfig);
        rsp.add(RawResponseWriter.CONTENT, content);
    }
    return;
}
rsp.add("initArgs", initArgs);
String message = "";
if (command != null) {
    rsp.add("command", command);
}
// If importer is still null
if (importer == null) {
    rsp.add("status", DataImporter.MSG.NO_INIT);
    return;
}
if (command != null && DataImporter.ABORT_CMD.equals(command)) {
    //终止操作命令也没啥说的
    importer.runCmd(requestParams, null);
} else if (importer.isBusy()) {
    message = DataImporter.MSG.CMD_RUNNING;
} else if (command != null) {
    if (DataImporter.FULL_IMPORT_CMD.equals(command) || DataImporter.DELTA_IMPORT_CMD.equals(command) || IMPORT_CMD.equals(command)) {
        //full-import或delta-import或import命令，主要是在这里
        importer.maybeReloadConfiguration(requestParams, defaultParams);
        //获取索引更新处理链，在构造SolrCore这个类的时候会进行初始化，solr默认提供LogUpdateProcessorFactory，DistributedUpdateProcessorFactory，RunUpdateProcessorFactory三个处理器，可通过配置solrconfig进行修改
        UpdateRequestProcessorChain processorChain = req.getCore().getUpdateProcessingChain(params.get(UpdateParams.UPDATE_CHAIN));
        //通过上面的处理器工厂创建处理器，分析见下面
        UpdateRequestProcessor processor = processorChain.createProcessor(req, rsp);
        SolrResourceLoader loader = req.getCore().getResourceLoader();
        SolrWriter sw = getSolrWriter(processor, loader, requestParams, req);
        if (requestParams.isDebug()) {
            if (debugEnabled) {
                // Synchronous request for the debug mode
                importer.runCmd(requestParams, sw);
                rsp.add("mode", "debug");
                rsp.add("documents", requestParams.getDebugInfo().debugDocuments);
                if (requestParams.getDebugInfo().debugVerboseOutput != null) {
                    rsp.add("verbose-output", requestParams.getDebugInfo().debugVerboseOutput);
                }
            } else {
                message = DataImporter.MSG.DEBUG_NOT_ENABLED;
            }
        } else {
            // Asynchronous request for normal mode
            if(requestParams.getContentStream() == null && !requestParams.isSyncMode()){
                importer.runAsync(requestParams, sw);
            } else {
                importer.runCmd(requestParams, sw);
            }
        }
    } else if (DataImporter.RELOAD_CONF_CMD.equals(command)) {
        if(importer.maybeReloadConfiguration(requestParams, defaultParams)) {
            message = DataImporter.MSG.CONFIG_RELOADED;
        } else {
            message = DataImporter.MSG.CONFIG_NOT_RELOADED;
        }
    }
}
{% endhighlight %}
处理器工厂创建处理器方法：
{% highlight objc %}
public UpdateRequestProcessor createProcessor(SolrQueryRequest req,SolrQueryResponse rsp){
 	UpdateRequestProcessor processor = null;
  	UpdateRequestProcessor last = null;
  	final String distribPhase = req.getParams().get(DistributingUpdateProcessorFactory.DISTRIB_UPDATE_PARAM);
  	final boolean skipToDistrib = distribPhase != null;
  	boolean afterDistrib = true;  // we iterate backwards, so true to start
  	for (int i = chain.length-1; i>=0; i--) {
    		//通过从后向前的方式进行创建，即最开始创建RunUpdateProcessor，接着DistributedUpdateProcessor，最后才是LogUpdateProcessor
    		UpdateRequestProcessorFactory factory = chain[i];
    		if (skipToDistrib) {
        			if (afterDistrib) {
          				if (factory instanceof DistributingUpdateProcessorFactory) {
					afterDistrib = false;
          				}
			} else if (!(factory instanceof LogUpdateProcessorFactory)) {
				// TODO: use a marker interface for this?
				// skip anything that is not the log factory
          				continue;
          			}
		}
		//getInstance方法的第三个参数指的是该处理器的next即下一个处理器，这样就形成了一条链了
		processor = factory.getInstance(req, rsp, last);
      		last = processor == null ? last : processor;
    	}
	//这里返回最后一个处理器即LogUpdateProcessor，它的next是DistributedUpdateProcessor，DistributedUpdateProcessor的next是RunUpdateProcessor
	return last;
}
{% endhighlight %}
获取SolrWriter进行索引创建，processor就是上面的处理器，索引创建主要是靠upload方法完成：
{% highlight objc %}
private SolrWriter getSolrWriter(final UpdateRequestProcessor processor,final SolrResourceLoader loader, final RequestInfo requestParams, SolrQueryRequest req) {
	return new SolrWriter(processor, req) {
		@Override
		public boolean upload(SolrInputDocument document) {
			try {
				return super.upload(document);
			} catch (RuntimeException e) {
				LOG.error( "Exception while adding: " + document, e);
				return false;
			}
		}
	};
}
{% endhighlight %}
接下来看runAsync和runCmd两个方法，runAsync从字面看就是以异步方式进行，无非就是重新开一个线程执行runCmd方法，进到runCmd方法，主要代码如下：
{% highlight objc %}
if (FULL_IMPORT_CMD.equals(command) || IMPORT_CMD.equals(command)) {
	doFullImport(sw, reqParams);
} else if (command.equals(DELTA_IMPORT_CMD)) {
	doDeltaImport(sw, reqParams);
}
{% endhighlight %}
接着进入doFullImport方法:
{% highlight objc %}
//根据db-data-config.xml文件创建dataimport.properties文件，该文件记录了上次做索引的时间可用于delta-import
DIHProperties dihPropWriter = createPropertyWriter();
//将索引开始时间写入dataimport.properties文件
setIndexStartTime(dihPropWriter.getCurrentTimestamp());
//创建DocBuilder用于获取数据并构造成Document，如何获取数据下面会讲到（通过DocBuilder的EntityProcessorWrapper currentEntityProcessorWrapper属性进行操作）
docBuilder = new DocBuilder(this, writer, dihPropWriter, requestParams);
checkWritablePersistFile(writer, dihPropWriter);
docBuilder.execute();
{% endhighlight %}
进入DocBuilder.execute():
{% highlight objc %}
public void execute() {
 	List<EntityProcessorWrapper> epwList = null;
	try {
		//下面就是一些状态信息的设置没啥说的
		dataImporter.store(DataImporter.STATUS_MSGS, statusMessages);
		config = dataImporter.getConfig();
		final AtomicLong startTime = new AtomicLong(System.currentTimeMillis());
		statusMessages.put(TIME_ELAPSED, new Object() {
			@Override
			public String toString() {
				return getTimeElapsedSince(startTime.get());
			}
		});
		statusMessages.put(DataImporter.MSG.TOTAL_QUERIES_EXECUTED,importStatistics.queryCount);
		statusMessages.put(DataImporter.MSG.TOTAL_ROWS_EXECUTED,importStatistics.rowsCount);
		statusMessages.put(DataImporter.MSG.TOTAL_DOC_PROCESSED,importStatistics.docCount);
		statusMessages.put(DataImporter.MSG.TOTAL_DOCS_SKIPPED,importStatistics.skipDocCount);
		//获取正在做索引的entity
		List<String> entities = reqParams.getEntitiesToRun();
		// Trigger onImportStart
		if (config.getOnImportStart() != null) {
			invokeEventListener(config.getOnImportStart());
		}
		AtomicBoolean fullCleanDone = new AtomicBoolean(false);
		//we must not do a delete of *:* multiple times if there are multiple root entities to be run
		Map<String,Object> lastIndexTimeProps = new HashMap<String,Object>();
		lastIndexTimeProps.put(LAST_INDEX_KEY, dataImporter.getIndexStartTime());
		epwList = new ArrayList<EntityProcessorWrapper>(config.getEntities().size());
		for (Entity e : config.getEntities()) {
			//通过entity获取entity处理器，默认是SqlEntityProcessor
			epwList.add(getEntityProcessorWrapper(e));
		}
		for (EntityProcessorWrapper epw : epwList) {
			if (entities != null && !entities.contains(epw.getEntity().getName()))
				continue;
			lastIndexTimeProps.put(epw.getEntity().getName() + "." + LAST_INDEX_KEY, propWriter.getCurrentTimestamp());
			//设置currentEntityProcessorWrapper以便操作数据库
			currentEntityProcessorWrapper = epw;
			//读取db-data-config.xml文件获取删除索引的配置
			String delQuery = epw.getEntity().getAllAttributes().get("preImportDeleteQuery");
			if (dataImporter.getStatus() == DataImporter.Status.RUNNING_DELTA_DUMP) {
				//执行删除索引操作
				cleanByQuery(delQuery, fullCleanDone);
				//执行增量索引
				doDelta();
				delQuery = epw.getEntity().getAllAttributes().get("postImportDeleteQuery");
				if (delQuery != null) {
					fullCleanDone.set(false);
					cleanByQuery(delQuery, fullCleanDone);
				}
			} else {
				cleanByQuery(delQuery, fullCleanDone);
				//执行全量索引操作
				doFullDump();
				delQuery = epw.getEntity().getAllAttributes().get("postImportDeleteQuery");
				if (delQuery != null) {
					fullCleanDone.set(false);
					cleanByQuery(delQuery, fullCleanDone);
				}
			}
			statusMessages.remove(DataImporter.MSG.TOTAL_DOC_PROCESSED);
		}
		if (stop.get()) {
			// Dont commit if aborted using command=abort
			statusMessages.put("Aborted", new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.ROOT).format(new Date()));
			rollback();
		} else {
			// Do not commit unnecessarily if this is a delta-import and no documents were created or deleted
			if (!reqParams.isClean()) {
				if (importStatistics.docCount.get() > 0 || importStatistics.deletedDocCount.get() > 0) {
					finish(lastIndexTimeProps);
				}
			} else {
				// Finished operation normally, commit now
				finish(lastIndexTimeProps);
			}
			if (config.getOnImportEnd() != null) {
				invokeEventListener(config.getOnImportEnd());
			}
		}
		statusMessages.remove(TIME_ELAPSED);
		statusMessages.put(DataImporter.MSG.TOTAL_DOC_PROCESSED, ""+ importStatistics.docCount.get());
		if(importStatistics.failedDocCount.get() > 0)
			statusMessages.put(DataImporter.MSG.TOTAL_FAILED_DOCS, ""+ importStatistics.failedDocCount.get());
		statusMessages.put("Time taken", getTimeElapsedSince(startTime.get()));
		LOG.info("Time taken = " + getTimeElapsedSince(startTime.get()));
	} catch(Exception e){
		throw new RuntimeException(e);
	} finally{
		if (writer != null) {
			writer.close();
		}
      		if (epwList != null) {
        			closeEntityProcessorWrappers(epwList);
      		}
      		if(reqParams.isDebug()) {
        			reqParams.getDebugInfo().debugVerboseOutput = getDebugLogger().output;
      		}
    	}
}
{% endhighlight %}
进入cleanByQuery方法：
{% highlight objc %}
private void cleanByQuery(String delQuery, AtomicBoolean completeCleanDone) {
	delQuery = getVariableResolver().replaceTokens(delQuery);
	if (reqParams.isClean()) {//如果需要执行clean操作（可选）
		if (delQuery == null && !completeCleanDone.get()) {
			//db-data-config没有配置索引删除则删除所有索引
			writer.doDeleteAll();
			completeCleanDone.set(true);
		} else if (delQuery != null) {
			writer.deleteByQuery(delQuery);
		}
	}
}
{% endhighlight %}

{% highlight objc %}
public void doDeleteAll() {
	try {
		DeleteUpdateCommand deleteCommand = new DeleteUpdateCommand(req);
		deleteCommand.query = "*:*";
		//通过处理器进行删除操作，处理器不陌生吧，上面说过的
		processor.processDelete(deleteCommand);
	 catch (IOException e) {
		throw new DataImportHandlerException(DataImportHandlerException.SEVERE,"Exception in full dump while deleting all documents.", e);
	}
}
{% endhighlight %}
好了删除的先说到这里，先说增加索引吧，因为问题是出在增加索引的时候，看完代码会发现增加和删除的基本上一样的流程。</br></br>

进入doFullDump()方法：
{% highlight objc %}
private void doFullDump() {
	addStatusMessage("Full Dump Started");   
	buildDocument(getVariableResolver(), null, null, currentEntityProcessorWrapper, true, null);
}
{% endhighlight %}

{% highlight objc %}
private void buildDocument(VariableResolver vr, DocWrapper doc,Map<String, Object> pk, EntityProcessorWrapper epw, boolean isRoot,ContextImpl parentCtx, List<EntityProcessorWrapper> entitiesToDestroy) {
	ContextImpl ctx = new ContextImpl(epw, vr, null,pk == null ? Context.FULL_DUMP : Context.DELTA_DUMP,session, parentCtx, this);
	epw.init(ctx);
	if (!epw.isInitalized()) {
		entitiesToDestroy.add(epw);
		epw.setInitalized(true);
	}
	if (reqParams.getStart() > 0) {
		getDebugLogger().log(DIHLogLevels.DISABLE_LOGGING, null, null);
	}
	if (verboseDebug) {
		getDebugLogger().log(DIHLogLevels.START_ENTITY, epw.getEntity().getName(), null);
	}
	int seenDocCount = 0;
	try {
		while (true) {
			if (stop.get())
				return;
			if(importStatistics.docCount.get() > (reqParams.getStart() + reqParams.getRows())) break;
			try {
				seenDocCount++;
				if (seenDocCount > reqParams.getStart()) {
					getDebugLogger().log(DIHLogLevels.ENABLE_LOGGING, null, null);
				}
				if (verboseDebug && epw.getEntity().isDocRoot()) {
					getDebugLogger().log(DIHLogLevels.START_DOC, epw.getEntity().getName(), null);
				}
				if (doc == null && epw.getEntity().isDocRoot()) {
					doc = new DocWrapper();
					ctx.setDoc(doc);
					Entity e = epw.getEntity();
					while (e.getParentEntity() != null) {
              					addFields(e.getParentEntity(), doc, (Map<String, Object>) vr.resolve(e.getParentEntity().getName()), vr);
              					e = e.getParentEntity();
            				}
          				}
				//这里获取从数据库读取出来的每一样数据，epw就是之前说过的EntityProcessorWrapper
          				Map<String, Object> arow = epw.nextRow();
          				if (arow == null) {
            				break;
          				}
          				// Support for start parameter in debug mode
          				if (epw.getEntity().isDocRoot()) {
            				if (seenDocCount <= reqParams.getStart())
             					continue;
            				if (seenDocCount > reqParams.getStart() + reqParams.getRows()) {
              					LOG.info("Indexing stopped at docCount = " + importStatistics.docCount);
              					break;
            				}
          				}
          				if (verboseDebug) {
            				getDebugLogger().log(DIHLogLevels.ENTITY_OUT, epw.getEntity().getName(), arow);
          				}
          				importStatistics.rowsCount.incrementAndGet();
          				if (doc != null) {
					//特殊处理也没啥特殊的，不讲
            				handleSpecialCommands(arow, doc);
					//将数据库数据组装成EntityFiled并构造成Document，不讲，很简单。
            				addFields(epw.getEntity(), doc, arow, vr);
          				}
          				if (epw.getEntity().getChildren() != null) {
            				vr.addNamespace(epw.getEntity().getName(), arow);
            				for (EntityProcessorWrapper child : epw.getChildren()) {
						buildDocument(vr, doc,child.getEntity().isDocRoot() ? pk : null, child, false, ctx, entitiesToDestroy);
            				}
            				vr.removeNamespace(epw.getEntity().getName());
          				}
          				if (epw.getEntity().isDocRoot()) {
            				if (stop.get())
              					return;
            				if (!doc.isEmpty()) {
						//好了，重点来了，看到upload方法是不是有点似曾相似，是的，上面说过的。构造好Document需要由SolrWriter进行写到硬盘中
              					boolean result = writer.upload(doc);
              					if(reqParams.isDebug()) {
                						reqParams.getDebugInfo().debugDocuments.add(doc);
              					}
              					doc = null;
              					if (result){
                						importStatistics.docCount.incrementAndGet();
              					} else {
                						importStatistics.failedDocCount.incrementAndGet();
              					}
            				}
          				}
        			} catch (DataImportHandlerException e) {
          				if (verboseDebug) {
            				getDebugLogger().log(DIHLogLevels.ENTITY_EXCEPTION, epw.getEntity().getName(), e);
          				}
          				if(e.getErrCode() == DataImportHandlerException.SKIP_ROW){
            				continue;
          				}
          				if (isRoot) {
            				if (e.getErrCode() == DataImportHandlerException.SKIP) {
              					importStatistics.skipDocCount.getAndIncrement();
              					doc = null;
            				} else {
              					SolrException.log(LOG, "Exception while processing: "+ epw.getEntity().getName() + " document : " + doc, e);
            				}
            				if (e.getErrCode() == DataImportHandlerException.SEVERE)
              					throw e;
          				} else
            				throw e;
        			} catch (Throwable t) {
          				if (verboseDebug) {
            				getDebugLogger().log(DIHLogLevels.ENTITY_EXCEPTION, epw.getEntity().getName(), t);
          				}
          				throw new DataImportHandlerException(DataImportHandlerException.SEVERE, t);
        			} finally {
          				if (verboseDebug) {
            				getDebugLogger().log(DIHLogLevels.ROW_END, epw.getEntity().getName(), null);
            				if (epw.getEntity().isDocRoot())
              					getDebugLogger().log(DIHLogLevels.END_DOC, null, null);
          				}
        			}
      		}
    	} finally {
      		if (verboseDebug) {
        			getDebugLogger().log(DIHLogLevels.END_ENTITY, null, null);
      		}
    	}
}
{% endhighlight %}



进入nextRow()方法：
{% highlight objc %}
public Map<String, Object> nextRow() {
    	if (rowcache != null) {
      		return getFromRowCache();
   	}
    	while (true) {
      		Map<String, Object> arow = null;
      		try {
			//通过entity处理代理类即上面说过的SqlEntityProcessor来获取具体的数据，内部通过JdbcDataSource进行连接数据源进行获取，集体代码就不讲了
        			arow = delegate.nextRow();
      		} catch (Exception e) {
        			if(ABORT.equals(onError)){
          				wrapAndThrow(SEVERE, e);
        			} else {
          				//SKIP is not really possible. If this calls the nextRow() again the Entityprocessor would be in an inconisttent state           
          				SolrException.log(log, "Exception in entity : "+ entityName, e);
          				return null;
        			}
      		}
      		if (arow == null) {
        			return null;
      		} else {
			//获取到一行数据以后对改行数据进行转换处理，transformer可通过自己扩展及配置，主要是对数据库取出的数据进行二次转换并重新设置，这里下面还会继续讲到。
        			arow = applyTransformer(arow);
        			if (arow != null) {
          				delegate.postTransform(arow);
          				return arow;
        			}
      		}
    	}
}
{% endhighlight %}


进入SolrWriter的upload()方法：
{% highlight objc %}
public boolean upload(SolrInputDocument d) {
    	try {
      		AddUpdateCommand command = new AddUpdateCommand(req);
      		command.solrDoc = d;
      		command.commitWithin = commitWithin;
		//再次邂逅processor
      		processor.processAdd(command);
    	} catch (Exception e) {
      		log.warn("Error creating document : " + d, e);
      		return false;
    	}
    	return true;
}
{% endhighlight %}
进入LogUpdateProcessor.processAdd()方法：
{% highlight objc %}
public void processAdd(AddUpdateCommand cmd) throws IOException {
    	if (logDebug) { log.debug("PRE_UPDATE " + cmd.toString() + " " + req); }
    	// call delegate first so we can log things like the version that get set later
	//由LogUpdateProcessor的next(还记得是什么吗)即DistributedUpdateProcessor来处理。 
	if (next != null) next.processAdd(cmd);
    	// Add a list of added id's to the response
    	if (adds == null) {
   		adds = new ArrayList<String>();
		toLog.add("add",adds);
    	}
    	if (adds.size() < maxNumToLog) {
 		long version = cmd.getVersion();
      		String msg = cmd.getPrintableId();
		if (version != 0) msg = msg + " (" + version + ')';
      		adds.add(msg);
    	}
    	numAdds++;
}
{% endhighlight %}
下面进入重点的重点DistributedUpdateProcessor.processAdd()方法，看名字就知道是为分布式而生的，当然你的solr如果不是分布式的也会进入到这里的，具体判断见代码分析：
{% highlight objc %}
public void processAdd(AddUpdateCommand cmd) throws IOException {
    	updateCommand = cmd;
	//是否是zookeeper管理的集群，如果没配置zookeeper，sold也会通过update.distrib参数来检查当前索引操作是在leader节点还是在slave节点
    	if (zkEnabled) {
      		zkCheck();
		//配置了solrcloud的就会在这里获取到所有的节点，setupRequest代码有点复杂，主要是通过zkController来获取集群的配置信息来
      		nodes = setupRequest(cmd.getHashableId(), cmd.getSolrInputDocument());
    	} else {
      		isLeader = getNonZkLeaderAssumption(req);
    	}
    	boolean dropCmd = false;
    	if (!forwardToLeader) {//不是由子节点向主节点请求
		//versionAdd()方法也比较复杂，主要是根据VersionBucket来修改当前Document的_version_值
      		dropCmd = versionAdd(cmd);
    	}
    	if (dropCmd) {
      		// TODO: do we need to add anything to the response?
      		return;
    	}
    	ModifiableSolrParams params = null;
    	if (nodes != null) {
      		params = new ModifiableSolrParams(filterParams(req.getParams()));
      		params.set(DISTRIB_UPDATE_PARAM,(isLeader ?DistribPhase.FROMLEADER.toString() :
                  	DistribPhase.TOLEADER.toString()));
      		if (isLeader) {
        			params.set("distrib.from", ZkCoreNodeProps.getCoreUrl(
            		zkController.getBaseUrl(), req.getCore().getName()));
      		}
      		params.set("distrib.from", ZkCoreNodeProps.getCoreUrl(zkController.getBaseUrl(), req.getCore().getName()));
		//这里进行分布式添加索引
      		cmdDistrib.distribAdd(cmd, nodes, params);
    	}
    	// TODO: what to do when no idField?
    	if (returnVersions && rsp != null && idField != null) {
      		if (addsResponse == null) {
        			addsResponse = new NamedList<String>();
        			rsp.add("adds",addsResponse);
      		}
      		if (scratch == null) scratch = new CharsRef();
		idField.getType().indexedToReadable(cmd.getIndexedId(), scratch);
		addsResponse.add(scratch.toString(), cmd.getVersion());
    	}
    	// TODO: keep track of errors?  needs to be done at a higher level though since
    	// an id may fail before it gets to this processor.
    	// Given that, it may also make sense to move the version reporting out of this
    	// processor too.
}
{% endhighlight %}
进入getHashableId()方法，这个方法很重要，判断将当前索引放到哪个节点就是通过获取这个索引的uniqueKey的值来进行hash计算定位放到哪个shard，如果通过该值没有获取到某个具体的shard则solrcloud会默认选择该Document所在的那个shard：
{% highlight objc %}
public String getHashableId() {
    	String id = null;
    	IndexSchema schema = req.getSchema();
    	SchemaField sf = schema.getUniqueKeyField();
    	if (sf != null) {
      		if (solrDoc != null) {
        			SolrInputField field = solrDoc.getField(sf.getName());
        			int count = field == null ? 0 : field.getValueCount();
        			if (count == 0) {
          				if (overwrite) {
            				throw new SolrException(SolrException.ErrorCode.BAD_REQUEST,"Document is missing mandatory uniqueKey field: "+ sf.getName());
          				}
        			} else if (count > 1) {
          				throw new SolrException(SolrException.ErrorCode.BAD_REQUEST,"Document contains multiple values for uniqueKey field: " + field);
        			} else {
          				return field.getFirstValue().toString();
        			}
      		}
    	}
    	return id;
}
{% endhighlight %}
SetupRequest方法中有 Slice slice = coll.getRouter().getTargetSlice(id, doc, req.getParams(), coll);这行代码：通过上面获取到的id来定位目标分片。进入getTargetSlice()方法：
{% highlight objc %}
public Slice getTargetSlice(String id, SolrInputDocument sdoc, SolrParams params, DocCollection collection) {
    	if (id == null) id = getId(sdoc, params);
    	int hash = sliceHash(id, sdoc, params);
    	return hashToSlice(hash, collection);
}

protected int sliceHash(String id, SolrInputDocument sdoc, SolrParams params) {
    	return Hash.murmurhash3_x86_32(id, 0, id.length(), 0);
}

protected Slice hashToSlice(int hash, DocCollection collection) {
	//collection.getSlices()获取该collection的所有分片也就是所有节点
    	for (Slice slice : collection.getSlices()) {
		//计算该节点的索引存储范围，范围计算见下面
      		Range range = slice.getRange();
      		if (range != null && range.includes(hash)) return slice;
    	}
    	throw new SolrException(SolrException.ErrorCode.BAD_REQUEST, "No slice servicing hash code " + Integer.toHexString(hash) + " in " + collection);
}
{% endhighlight %}


在完成solrcloud集群配置完成之后创建几点是通过Overseer的createCollection方法进行的：
{% highlight objc %}
private ClusterState createCollection(ClusterState state, String collectionName, int numShards) {
        	log.info("Create collection {} with numShards {}", collectionName, numShards);
        	DocRouter router = DocRouter.DEFAULT;
	//这里是主要的代码，为每个节点分配接收索引的范围
        	List<DocRouter.Range> ranges = router.partitionRange(numShards, router.fullRange());
        	Map<String, DocCollection> newCollections = new LinkedHashMap<String,DocCollection>();
        	Map<String, Slice> newSlices = new LinkedHashMap<String,Slice>();
        	newCollections.putAll(state.getCollectionStates());
        	for (int i = 0; i < numShards; i++) {
          		final String sliceName = "shard" + (i+1);
          		Map<String,Object> sliceProps = new LinkedHashMap<String,Object>(1);
          		sliceProps.put(Slice.RANGE, ranges.get(i));
          		newSlices.put(sliceName, new Slice(sliceName, null, sliceProps));
        	}
        	// TODO: fill in with collection properties read from the /collections/collectionName node
        	Map<String,Object> collectionProps = defaultCollectionProps();
        	DocCollection newCollection = new DocCollection(collectionName, newSlices, collectionProps, router);
        	newCollections.put(collectionName, newCollection);
        	ClusterState newClusterState = new ClusterState(state.getLiveNodes(), newCollections);
        	return newClusterState;
}
{% endhighlight %}
partitionRange()方法，partitions代表节点数目，算法很简单，不细讲：
{% highlight objc %}
public List<Range> partitionRange(int partitions, Range range) {
    	int min = range.min;//Integer.MIN_VALUE
    	int max = range.max;//Integer.MAX_VALUE
    	assert max >= min;
    	if (partitions == 0) return Collections.EMPTY_LIST;
    	long rangeSize = (long)max - (long)min;
    	long rangeStep = Math.max(1, rangeSize / partitions);
    	List<Range> ranges = new ArrayList<Range>(partitions);
    	long start = min;
    	long end = start;
    	// keep track of the idealized target to avoid accumulating rounding errors
    	long targetStart = min;
    	long targetEnd = targetStart;
    	// Round to avoid splitting hash domains across ranges if such rounding is not significant.
    	// With default bits==16, one would need to create more than 4000 shards before this
    	// becomes false by default.
    	boolean round = rangeStep >= (1<<bits)*16;
    	while (end < max) {
      		targetEnd = targetStart + rangeStep;
      		end = targetEnd;
      		if (round && ((end & mask2) != mask2)) {
        			// round up or down?
        			int increment = 1 << bits;  // 0x00010000
        			long roundDown = (end | mask2) - increment ;
        			long roundUp = (end | mask2) + increment;
        			if (end - roundDown < roundUp - end && roundDown > start) {
          				end = roundDown;
        			} else {
          				end = roundUp;
        			}
      		}
      		// make last range always end exactly on MAX_VALUE
      		if (ranges.size() == partitions - 1) {
        			end = max;
      		}
      		ranges.add(new Range((int)start, (int)end));
      		start = end + 1L;
      		targetStart = targetEnd + 1L;
    	}
        	return ranges;
}
{% endhighlight %}


由cmdDistrib.distribAdd()方法最终会到sumit()（注意每个节点都会调用）方法：
{% highlight objc %}
public void submit(final Request sreq) {
	//获取该节点的url（在创建该节点就会确定的）以便发起http请求
    	final String url = sreq.node.getUrl();
    	Callable<Request> task = new Callable<Request>() {
	      	@Override
	      	public Request call() throws Exception {
	        		Request clonedRequest = null;
	        		try {
	          			clonedRequest = new Request();
	          			clonedRequest.node = sreq.node;
	          			clonedRequest.ureq = sreq.ureq;
	          			clonedRequest.retries = sreq.retries;
	          			String fullUrl;
	          			if (!url.startsWith("http://") && !url.startsWith("https://")) {
	            			fullUrl = "http://" + url;
	          			} else {
	            			fullUrl = url;
	          			}
	          			HttpSolrServer server = new HttpSolrServer(fullUrl,updateShardHandler.getHttpClient());
	          			if (Thread.currentThread().isInterrupted()) {
	            			clonedRequest.rspCode = 503;
	            			clonedRequest.exception = new SolrException(ErrorCode.SERVICE_UNAVAILABLE, "Shutting down.");
	            			return clonedRequest;
	          			}
	          			clonedRequest.ursp = server.request(clonedRequest.ureq);
	          			// currently no way to get the request body.
	        		} catch (Exception e) {
	          			clonedRequest.exception = e;
	          			if (e instanceof SolrException) {
	            			clonedRequest.rspCode = ((SolrException) e).code();
	          			} else {
	            			clonedRequest.rspCode = -1;
	          			}
	        		} finally {
	          			semaphore.release();
	        		}
	        		return clonedRequest;
      		}
	};
    	try {
      		semaphore.acquire();
    	} catch (InterruptedException e) {
      		Thread.currentThread().interrupt();
      		throw new SolrException(ErrorCode.SERVICE_UNAVAILABLE, "Update thread interrupted", e);
    	}
    	try {
      		pending.add(completionService.submit(task));
    	} catch (RejectedExecutionException e) {
      		semaphore.release();
      		throw new SolrException(ErrorCode.SERVICE_UNAVAILABLE, "Shutting down", e);
    	}
}
{% endhighlight %}


server.request()方法中重要的代码Collection<ContentStream> streams = requestWriter.getContentStreams(request);</br></br>

进入UpdateRequestExt的getContentStreams()方法，将Document构造成一个xml文件并转换成字符流作为参数进行http请求，收到请求后会以xml的方式添加索引（这个过程就很简单了，会调用lucene的IndexWriter）：
{% highlight objc %}
public Collection<ContentStream> getContentStreams() throws IOException {
    	return ClientUtils.toContentStreams(getXML(), ClientUtils.TEXT_XML);
}
{% endhighlight %}
{% highlight objc %}
public String getXML() throws IOException {
    	StringWriter writer = new StringWriter();
    	writeXML(writer);
    	writer.flush();
    	String xml = writer.toString();
    	return (xml.length() > 0) ? xml : null;
}
{% endhighlight %}
{% highlight objc %}
public void writeXML(Writer writer) throws IOException {
    	List<List<SolrDoc>> getDocLists = getDocLists(documents);
    	for (List<SolrDoc> docs : getDocLists) {
      		if ((docs != null && docs.size() > 0)) {
        			SolrDoc firstDoc = docs.get(0);
        			int commitWithin = firstDoc.commitWithin != -1 ? firstDoc.commitWithin : this.commitWithin;
        			boolean overwrite = firstDoc.overwrite;
        			if (commitWithin > -1 || overwrite != true) {
          				writer.write("<add commitWithin=\"" + commitWithin + "\" " + "overwrite=\"" + overwrite + "\">");
        			} else {
          				writer.write("<add>");
        			}
        			if (documents != null) {
          				for (SolrDoc doc : documents) {
            				if (doc != null) {
              					ClientUtils.writeXML(doc.document, writer);
            				}
          				}
        			}
        			writer.write("</add>");
      		}
    	}
    	// Add the delete commands
    	boolean deleteI = deleteById != null && deleteById.size() > 0;
    	boolean deleteQ = deleteQuery != null && deleteQuery.size() > 0;
    	if (deleteI || deleteQ) {
      		writer.append("<delete>");
      		if (deleteI) {
        			for (Map.Entry<String,Long> entry : deleteById.entrySet()) {
          				writer.append("<id");
          				Long version = entry.getValue();
          				if (version != null) {
            				writer.append(" version=\"" + version + "\"");
          				}
          				writer.append(">");
          				XML.escapeCharData(entry.getKey(), writer);
          				writer.append("</id>");
        			}
      		}
      		if (deleteQ) {
        			for (String q : deleteQuery) {
          				writer.append("<query>");
          				XML.escapeCharData(q, writer);
          				writer.append("</query>");
        			}
      		}
      		writer.append("</delete>");
    	}
}
{% endhighlight %}


进入ClientUtil.writeXML()方法：
{% highlight objc %}
public static void writeXML( SolrInputDocument doc, Writer writer ) throws IOException{
    	writer.write("<doc boost=\""+doc.getDocumentBoost()+"\">");
    	for( SolrInputField field : doc ) {
      		float boost = field.getBoost();
      		String name = field.getName();
      		for( Object v : field ) {
        			String update = null;
        			if (v instanceof Map) {
          				// currently only supports a single value
          				for (Entry<Object,Object> entry : ((Map<Object,Object>)v).entrySet()) {
            				update = entry.getKey().toString();
            				v = entry.getValue();
            				if (v instanceof Collection) {
              					Collection values = (Collection) v;
              					for (Object value : values) {
                						writeVal(writer, boost, name, value, update);
                						boost = 1.0f;
              					}
            				} else  {
              					writeVal(writer, boost, name, v, update);
              					boost = 1.0f;
            				}
          				}
        			} else  {
          				writeVal(writer, boost, name, v, update);
          				// only write the boost for the first multi-valued field
          				// otherwise, the used boost is the product of all the boost values
          				boost = 1.0f;
        			}
      		}
    	}
    	writer.write("</doc>");
}
{% endhighlight %}
继续跟进writeVal():
{% highlight objc %}
private static void writeVal(Writer writer, float boost, String name, Object v, String update) throws IOException {
    	if (v instanceof Date) {
      		v = DateUtil.getThreadLocalDateFormat().format( (Date)v );
    	} else if (v instanceof byte[]) {
      		byte[] bytes = (byte[]) v;
      		v = Base64.byteArrayToBase64(bytes, 0, bytes.length);
    	} else if (v instanceof ByteBuffer) {
      		ByteBuffer bytes = (ByteBuffer) v;
      		v = Base64.byteArrayToBase64(bytes.array(), bytes.position(),bytes.limit() - bytes.position());
    	}
    	if (update == null) {
      		if( boost != 1.0f ) {
        			XML.writeXML(writer, "field", v.toString(), "name", name, "boost", boost);
      		} else if (v != null) {
        			XML.writeXML(writer, "field", v.toString(), "name", name );
      		}
    	} else {
      		if( boost != 1.0f ) {
        			XML.writeXML(writer, "field", v.toString(), "name", name, "boost", boost, "update", update);
      		} else {
        			if (v == null)  {
          				XML.writeXML(writer, "field", null, "name", name, "update", update, "null", true);
        			} else  {
          				XML.writeXML(writer, "field", v.toString(), "name", name, "update", update);
        			}
      		}
    	}
}
{% endhighlight %}
看到这里就豁然开朗了，第一行代码的if(v instance Date)，由于数据库的时间字段是Timestamp extends Date类型的，所以这里又将字符串时间值进行了转换，所以该索引的时间字段值就和主节点的值不一样了。</br></br>
解决方法可以为entity加上自定义的transformer主动将时间字段值toString()，在上面的nextRow()操作就会调用这个transformer，到了writeVal中v就不再是Date类型了，还有一种方法就是修改sql语句convert(varchar(19),120,updateTime) as updateTime，当然推荐后者了，从简单和做索引的速度来看就知道了。
