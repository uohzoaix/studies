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
public UpdateRequestProcessor createProcessor(SolrQueryRequest req,SolrQueryResponse rsp)
  {

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

        } else if (!(factory instanceof LogUpdateProcessorFactory)) {    // TODO: use a marker interface for this?

          // skip anything that is not the log factory

          continue;

        }

      }//getInstance方法的第三个参数指的是该处理器的next即下一个处理器，这样就形成了一条链了
      processor = factory.getInstance(req, rsp, last);

      last = processor == null ? last : processor;

    }//这里返回最后一个处理器即LogUpdateProcessor，它的next是DistributedUpdateProcessor，DistributedUpdateProcessor的next是RunUpdateProcessor
    return last;
  }
{% endhighlight %}

