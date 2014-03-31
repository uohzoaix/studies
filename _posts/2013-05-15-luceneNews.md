---
layout: post
title: "lucene新特性"
category: 
- lucene
tags: []
---





刚用maven构建solr spatial项目时，始终run不起来，而且报的错是IllegalArgumentException：unknown parameters:{enablePositionIncrements=true}，很明显它的意思是说在某个类初始化的时候传递的参数 enablePositionIncrements不合法，不合法无非就是类型不正确或是该类根本不需要该参数（即多余的参数），可是之前做的项目用的都 是enablePositionIncrements=true啊，怎么这次就不行了呢？唯一有区别的是这次是使用maven构建的，maven构建的好 处是自己不需要去下载一大堆的jar包，只需要一条命令就可以让maven到它的仓库去下载到你本地。是不是maven下载的jar版本和我们以前用的版 本不一致，查看pom.xml文件果然配置的版本号为5.0-SNAPSHOT,前几天刚把solr升级为4.3。找到问题根源了，将maven下载下来 的lucene的jar包打开一看，版本号为lucene-core-5.0-20130514.183811-239.jar，我去刚好是昨天更新 的，lucene社区太活跃了吧（一般solr和lucene都是同时更新的而且版本号也是一样的）。说到活跃咱来看看它们到底有多活跃吧，记得年初我把 solr从3.6升级到4.0，没过几天，4.1出来了，没升级，过几天4.2又出来了，不升级，上个礼拜4.3又出来了，前天升级为4.3，没想到昨天 5.0的快照版又出来了，没用上几天新版本就出来了,这让我们如何是好，当然这只是快照版不适合生产环境，所以没必要在意5.0，不过从4.3一下子到 5.0说明solr和lucene有一些革命性的改变，所以期待5.0的stable版问世。回到问题上，用jad查看源码果然5.0的 StopFileterFactory类不再需要enablePositionIncrements参数，我在网上搜lucene5.0的文档和 wiki，很可惜暂时还没有相关的解释。好吧，既然不要这个参数那我把该参数删掉重新run成功。