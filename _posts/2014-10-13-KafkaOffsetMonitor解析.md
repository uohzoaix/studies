---
layout: post
title: "KafkaOffsetMonitor分析"
description: "kafka监控"
category: 
- kafka
tags: []
---


KafkaOffsetMonitor（下文简称KOM）本质上就是对ZkUtils中的：  

	val ConsumersPath = "/consumers"
	val BrokerIdsPath = "/brokers/ids"
	val BrokerTopicsPath = "/brokers/topics"
	val TopicConfigPath = "/config/topics"
	val TopicConfigChangesPath = "/config/changes"
	val ControllerPath = "/controller"
	val ControllerEpochPath = "/controller_epoch"
	val ReassignPartitionsPath = "/admin/reassign_partitions"
	val DeleteTopicsPath = "/admin/delete_topics"
	val PreferredReplicaLeaderElectionPath = "/admin/preferred_replica_election"
这些属性进行读取操作。  
KOM是使用angular.js来实现类似MVC功能的，首先需要定义app.js文件，在KOM中的app.js文件为：

	var app = angular.module('offsetapp',["offsetapp.controllers", "offsetapp.directives",  "ngRoute"],function($routeProvider) {$routeProvider
	.when("/", {templateUrl: "views/grouplist.html",controller: "GroupListCtrl"})
	.when("/group/:group", {templateUrl: "views/group.html",controller: "GroupCtrl"})
	.when("/group/:group/:topic", {templateUrl: "views/topic.html",controller:"TopicCtrl"})
	.when("/clusterviz", {templateUrl: "views/cluster-viz.html",controller:"ClusterVizCtrl"})
	.when("/activetopicsviz", {templateUrl: "views/activetopics-viz.html",controller: "ActiveTopicsVizCtrl"})
	.when("/topics", {templateUrl: "views/topiclist.html",controller:"TopicListCtrl"})
	.when("/topicdetail/:group", {templateUrl: "views/topic-detail.html",controller: "TopicDetailCtrl"});
	});
	angular.module("offsetapp.services", ["ngResource"]).factory("offsetinfo",["$resource", "$http", function($resource, $http) {
	function groupPartitions(cb) {
		return function(data) {
			var groups = _(data.offsets).groupBy(function(p) {
				var t = p.timestamp;
				if(!t) t = 0;
				return p.group+p.topic+t.toString();
			});
			groups = groups.values().map(function(partitions) {
				return {
					group: partitions[0].group,
					topic: partitions[0].topic,
					partitions: partitions,
					logSize: _(partitions).pluck("logSize").reduce(function(sum, num) {
						return sum + num;
					}),
					offset: _(partitions).pluck("offset").reduce(function(sum, num) {
						return sum + num;
					}),
					timestamp: partitions[0].timestamp
				};
			}).value();
			data.offsets = groups;
			cb(data);
		};
	}
	return {
			getGroup: function(group, cb) {
				return $resource("./group/:group").get({group:group}, groupPartitions(cb));
			},
			topicDetail: function(group, cb) {
            	return $resource("./topicdetails/:group").get({group:group}, groupPartitions(cb));
            },
            loadClusterViz: function(group, cb) {
                cb(loadViz("#dataviz-container", "/clusterlist"))
            },
            loadTopicConsumerViz: function(group, cb) {
                cb(loadViz("#dataviz-container", "/activetopics"))
            },
			listGroup: function() {return $http.get("./group");},
			listTopics: function() {return $http.get("./topiclist");},
			getTopic: function(group, topic, cb) {
				return $resource("./group/:group/:topic").get({group:group, topic: topic}, groupPartitions(cb));
			}
		};
	}
	]);

下面是controller.js文件：
 
	angular.module('offsetapp.controllers',["offsetapp.services"])
	.controller("GroupCtrl", ["$scope", "$interval", "$routeParams", "offsetinfo", function($scope, $interval, $routeParams, offsetinfo) {
		offsetinfo.getGroup($routeParams.group, function(d) {
			$scope.info = d;
			$scope.loading=false;
		});
		$scope.loading=true;
		$scope.group = $routeParams.group;
	}])
	.controller("GroupListCtrl", ["$scope", "offsetinfo", function($scope, offsetinfo) {
		$scope.loading = true;
		offsetinfo.listGroup().success(function(d) {
			$scope.loading=false;
			$scope.groups = d;
		});
	}])
    .controller("TopicListCtrl", ["$scope", "offsetinfo", function($scope, offsetinfo) {
        $scope.loading = true;
        offsetinfo.listTopics().success(function(d) {
        	$scope.loading=false;
           	$scope.topics = d;
        });
    }])
	.controller("TopicDetailCtrl", ["$scope", "$interval", "$routeParams", "offsetinfo", function($scope, $interval, $routeParams, offsetinfo) {
    	offsetinfo.topicDetail($routeParams.group, function(d) {
    		$scope.info = d;
    		$scope.loading=false;
    	});
    	$scope.loading=true;
		$scope.group = $routeParams.group;
    }])
    .controller("ClusterVizCtrl", ["$scope", "$interval", "$routeParams", "offsetinfo", function($scope, $interval, $routeParams, offsetinfo) {
    	$scope.loading = true;
        offsetinfo.loadClusterViz($routeParams.group, function(d) {
       	});
    }])
    .controller("ActiveTopicsVizCtrl", ["$scope", "$interval", "$routeParams", "offsetinfo", function($scope, $interval, $routeParams, offsetinfo) {
        $scope.loading = true;
        offsetinfo.loadTopicConsumerViz($routeParams.group, function(d) {
        });
    }])
	.controller("TopicCtrl", ["$scope", "$routeParams", "offsetinfo", function($scope, $routeParams, offsetinfo) {
		$scope.group = $routeParams.group;
		$scope.topic = $routeParams.topic;
		$scope.data = [];
		offsetinfo.getTopic($routeParams.group, $routeParams.topic, function(d) {
			$scope.data = d.offsets;
		});
	}]);
KOM中主要的一些流程都会在这两个文件中体现，关于画图的如果有时间的话会讲讲。
首先来看index.html文件中的
 
	<li><a href="#">Consumer Groups</a></li>
	<li><a href="/#/topics">Topic List</a></li>
	<li class="dropdown">
    <a  href="javascript:void(0)"  class="dropdown-toggle" data-toggle="dropdown">Visualizations <b class="caret"></b></a>
    <ul class="dropdown-menu">
    <li><a href="/#/activetopicsviz">Active Topic Consumers</a></li>
    <li><a href="/#/clusterviz">Cluster Overview</a></li>
    </ul>
	</li>
 
代码块。
<li>其中"#"表示访问项目根目录：对比app.js文件的
 
	.when("/", {templateUrl: "views/grouplist.html",controller: "GroupListCtrl"})
 
表示当访问项目根目录时使用的模板文件是grouplist.html，使用的controller是GroupListCtrl，继续看controller.js中的GroupListCtrl定义：
 
	.controller("GroupListCtrl", ["$scope", "offsetinfo", function($scope, offsetinfo) {
	$scope.loading = true;
	offsetinfo.listGroup().success(function(d) {
		$scope.loading=false;
		$scope.groups = d;
	});
	}])
 
会调用offsetinfo.listGroup()方法，再到app.js文件中查看listGroup方法定义：
 
	listGroup: function() {return $http.get("./group");}
 
这个时候会使用http模块映射到group这个path上，到这里就要看scala的代码了，进到OffsetGetterWeb.scala中，该类继承了UnfilteredWebApp类，在UnfilteredWebApp中定义了启动方法，可知KOM是使用jetty作为web容器的。回到OffsetGetterWeb的setup方法，该方法中首先会启动一个定时器，该定时器会定时将每个group的每个topic的相关信息（如offset，logSize，createTime，broker等）写到内置的db中，同时也会清除那些过期的数据。继续看group这个path的定义：
 
	case GET(Path(Seg("group" :: Nil))) =>
    	JsonContent ~> ResponseString(write(getGroups(args)))
 
调用getGroups方法首先会初始化zkClient和使用zkClient构造OffsetGetter类，接着调用OffsetGetter的getGroups方法：
 
	def getGroups: Seq[String] = {
    	try {
        	ZkUtils.getChildren(zkClient, ZkUtils.ConsumersPath)
   		} catch {
        	case NonFatal(t) =>
            	error(s"could not get groups because of ${t.getMessage}", t)
            	Seq()
    	}
	}
 
也就是说getGroups就是读取zookeeper中的/consumers目录的数据，读取完成之后通过$scope.groups = d;代码将结果赋给$scope.groups，这样grouplist.html中就可以通过遍历groups来得到每个group了：
 
	<li ng-repeat="g in groups" class="list-group-item"><a href="./#/group/{{g}}">{{g}}</a></li>
 
得到所有的groups之后，通过./#/group/{{g}}链接可以访问每个group的具体信息。
<li>./#/group/{{g}}链接在app.js中的
 
	.when("/group/:group", {templateUrl: "views/group.html",controller: "GroupCtrl"})
 
代码块中定义了，:group会作为参数传递给GroupCtrl控制器，返回的结果会在group.html中显示。GroupCtrl的定义为：
 
	.controller("GroupCtrl", ["$scope", "$interval", "$routeParams", "offsetinfo",function($scope, $interval,$routeParams, offsetinfo){
	offsetinfo.getGroup($routeParams.group, function(d) {
		$scope.info = d;$scope.loading=false; 
	});
	$scope.loading=true; 
	$scope.group = $routeParams.group; 
	}])
 
该控制器会调用offsetinfo.getGroup方法，app.js中对getGroup的定义为：
 
	getGroup: function(group, cb) {
		return $resource("./group/:group").get({group:group}, groupPartitions(cb));
	}
 
进入OffsetGetterWeb中查看对./group/:group这个path的定义：
 
	case GET(Path(Seg("group" :: group :: Nil))) =>
    	val info = getInfo(group, args)
    	JsonContent ~> ResponseString(write(info)) ~> Ok
 
首先会调用getInfo方法：
 
	def getInfo(group: String, args: OWArgs): KafkaInfo = withOG(args) {
    	_.getInfo(group)
	}
 
接着调用OffsetGetter的getInfo方法：
 
	def getInfo(group: String, topics: Seq[String] = Seq()): KafkaInfo = {
    	val off = offsetInfo(group, topics)
    	val brok = brokerInfo()
    	KafkaInfo(
        	brokers = brok.toSeq,
        	offsets = off
    	)
	}
 
最终的返回结果会通过$scope.info = d;将结果赋给score的info字段，该字段里包括brokers和offsets两个信息，在group.html中就可以获取对应的字段信息进行显示了：
 
	<tr ng-repeat="b in info.brokers">
	<td>{{b.id}}</td>
	<td>{{b.host}}</td>
	<td>{{b.port}}</td>
	</tr>
	<tbody ng-repeat="c in info.offsets">
	<tr class="topic-row">
		<td><a href="#/group/{{c.group}}/{{c.topic}}">{{c.topic}}</a></td>
		<td></td>
		<td>{{c.offset}}</td>
		<td>{{c.logSize}}</td>
		<td>{{c.logSize - c.offset}}</td>
		<td></td>
		<td></td>
		<td></td>
	</tr>
	<tr class="partition-row" ng-repeat="p in c.partitions">
		<td></td>
		<td>{{p.partition}}</td>
		<td>{{p.offset}}</td>
		<td>{{p.logSize}}</td>
		<td>{{p.logSize - p.offset}}</td>
		<td>{{p.owner}}</td>
		<td><moment timestamp="{{p.creation}}"></moment></td>
		<td><moment timestamp="{{p.modified}}"></moment></td>
	</tr>
	</tbody>
 
下面是一些中间过程：
 
	private def offsetInfo(group: String, topics: Seq[String] = Seq()): Seq[OffsetInfo] = {
    	val topicList = if (topics.isEmpty) {
        	try {
				//读取"/consumers/group/offsets"路径数据
        		ZkUtils.getChildren(zkClient, s"${ZkUtils.ConsumersPath}/$group/offsets").toSeq
      		} catch {
        		case _: ZkNoNodeException => Seq()
      		}
    	} else {
      		topics
    	}
		//将topicList进行遍历并调用processTopic(group, _)，第二个参数为某个具体的topic
    	topicList.sorted.flatMap(processTopic(group, _))
	}
	private def processTopic(group: String, topic: String): Seq[OffsetInfo] = {
		//利用getPartitionsForTopics方法来获取具体的topic信息
    	val pidMap = ZkUtils.getPartitionsForTopics(zkClient, Seq(topic))
    	for {
      		//获取topic的所有partition
      		partitions <- pidMap.get(topic).toSeq
	  		//获取topic的partition id
      		pid <- partitions.sorted
      		info <- processPartition(group, topic, pid)
    	} yield info
	}
	private def processPartition(group: String, topic: String, pid: Int): 		Option[OffsetInfo] = {
    		try { 
      			//读取/consumers/group/offsets/topic/pid目录获取topic的offset和zk统计信息
      			val (offset, stat: Stat) = ZkUtils.readData(zkClient, s"${ZkUtils.ConsumersPath}/$group/offsets/$topic/$pid")
      			//读取/consumers/group/owners/topic/pid获取topic的owner
      			val (owner, _) = ZkUtils.readDataMaybeNull(zkClient, s"${ZkUtils.ConsumersPath}/$group/owners/$topic/$pid")
      			//获取topic分片的leader并以leader的id构造SimpleConsumer
	  			//getLeaderForPartition方法首先会读取/brokers/topics/topic/partitions/pid/state信息
      			ZkUtils.getLeaderForPartition(zkClient, topic, pid) match {
        			case Some(bid) =>
          				val consumerOpt = consumerMap.getOrElseUpdate(bid, getConsumer(bid))
          				consumerOpt map {
            				consumer =>
			  					//以topic和partition id构造TopicAndPartition
              					val topicAndPartition = TopicAndPartition(topic, pid)
              					val request = OffsetRequest(immutable.Map(topicAndPartition -> PartitionOffsetRequestInfo(OffsetRequest.LatestTime, 1)))
			  					//获取相应topic的partition的log大小
              					val logSize = consumer.getOffsetsBefore(request).partitionErrorAndOffsets(topicAndPartition).offsets.head
              					//返回OffsetInfo信息
              					OffsetInfo(group = group,
                					topic = topic,
                					partition = pid,
                					offset = offset.toLong,
                					logSize = logSize,
                					owner = owner,
                					creation = Time.fromMilliseconds(stat.getCtime),
                					modified = Time.fromMilliseconds(stat.getMtime))
          					}
        			case None =>
          				error("No broker for partition %s - %s".format(topic, pid))
          				None
      			}
    		} catch {
      			case NonFatal(t) =>
        		error(s"Could not parse partition info. group: [$group] topic: [$topic]", t)
        		None
    		}
	}
	private def getConsumer(bid: Int): Option[SimpleConsumer] = {
    	try {
	  		//读取/brokers/ids/bid的broker信息来构造SimpleConsumer
      		ZkUtils.readDataMaybeNull(zkClient, ZkUtils.BrokerIdsPath + "/" + bid) match {
      			case (Some(brokerInfoString), _) =>
      				Json.parseFull(brokerInfoString) match {
            			case Some(m) =>
              				val brokerInfo = m.asInstanceOf[Map[String, Any]]
              				val host = brokerInfo.get("host").get.asInstanceOf[String]
              				val port = brokerInfo.get("port").get.asInstanceOf[Int]
              				Some(new SimpleConsumer(host, port, 10000, 100000, "ConsumerOffsetChecker"))
            			case None =>
              				throw new BrokerNotAvailableException("Broker id %d does not exist".format(bid))
          			}
        		case (None, _) =>
          			throw new BrokerNotAvailableException("Broker id %d does not exist".format(bid))
      		}
    	} catch {
      		case t: Throwable =>
        		error("Could not parse broker info", t)
        		None
    	}
	}
 
<li>group.html文件中的#/group/{{c.group}}/{{c.topic}}链接是获取某个topic的具体信息。app.js对该链接的定义为：
 
	.when("/group/:group/:topic", {templateUrl: "views/topic.html",controller: "TopicCtrl"})
 
controller中对TopicCtrl的定义为：
 
	.controller("TopicCtrl", ["$scope", "$routeParams", "offsetinfo",
	function($scope, $routeParams, offsetinfo) {
		$scope.group = $routeParams.group;
		$scope.topic = $routeParams.topic;
		$scope.data = [];
		offsetinfo.getTopic($routeParams.group, $routeParams.topic, function(d) {
			$scope.data = d.offsets;
		});
	}
	])
 
调用的getTopic方法在OffsetGetterWeb中的定义为：
 
	case GET(Path(Seg("group" :: group :: topic :: Nil))) =>
    	val offsets = args.db.offsetHistory(group, topic)
    	JsonContent ~> ResponseString(write(offsets)) ~> Ok
 
这里看到只会获取内置的db的信息，OffsetDB中offsetHistory方法定义为，查出来结果就是上面getInfo方法的结果：
 
	def offsetHistory(group: String, topic: String): OffsetHistory = 		database.withSession {
    		implicit s =>
	  			//在OFFSETS表中查找相应group和topic的信息
      			val o = offsets
        			.where(off => off.group === group && off.topic === topic)
        			.sortBy(_.timestamp)
        			.map(_.forHistory)
        			.list()
      			OffsetHistory(group, topic, o)
	}
	val offsets = TableQuery[Offset]
	//Offset类定义了OFFSETS表的一些字段和查询语句信息
	class Offset(tag: Tag) extends Table[DbOffsetInfo](tag, "OFFSETS") {
    	def id = column[Int]("id", O.PrimaryKey, O.AutoInc)
    	val group = column[String]("group")
    	val topic = column[String]("topic")
    	val partition = column[Int]("partition")
    	val offset = column[Long]("offset")
    	val logSize = column[Long]("log_size")
    	val owner = column[Option[String]]("owner")
    	val timestamp = column[Long]("timestamp")
    	val creation = column[Time]("creation")
    	val modified = column[Time]("modified")
    	def * = (id.?, group, topic, partition, offset, logSize, owner, timestamp, creation, modified).shaped <>(DbOffsetInfo.parse, DbOffsetInfo.unparse)
    	def forHistory = (timestamp, partition, owner, offset, logSize) <>(OffsetPoints.tupled, OffsetPoints.unapply)
    	def idx = index("idx_search", (group, topic))
    	def tidx = index("idx_time", (timestamp))
    	def uidx = index("idx_unique", (group, topic, partition, timestamp), unique = true)
	}
 
db会在OffsetGetterWeb中创建：lazy val db = new OffsetDB(dbName)
<li>回到index.html文件中，/#/topics链接在app.js中的定义为：
 
	.when("/topics", {templateUrl: "views/topiclist.html",controller: "TopicListCtrl"})
 
TopicListCtrl的定义为：
 
	.controller("TopicListCtrl", ["$scope", "offsetinfo",
    function($scope, offsetinfo) {
        $scope.loading = true;
        offsetinfo.listTopics().success(function(d) {
            $scope.loading=false;
            $scope.topics = d;
        });
    }
	])
 
该controller会调用offsetinfo.listTopics()方法，app.js中listTopics方法的定义为：
listTopics: function() {return $http.get("./topiclist");}，回到OffsetGetterWeb中，topiclist的定义为：
 
	case GET(Path(Seg("topiclist" :: Nil))) =>
    	JsonContent ~> ResponseString(write(getTopics(args)))
		def getTopics(args: OWArgs) = withOG(args) {
    		_.getTopics
  		}
 
 
	def getTopics: Seq[String] = {
    	try {
	 		//读取/brokers/topics路径的数据，返回所有的topic，并且以字母的大小顺序排序
      		ZkUtils.getChildren(zkClient, ZkUtils.BrokerTopicsPath).sortWith(_ < _)
    	} catch {
      		case NonFatal(t) =>
        		error(s"could not get topics because of ${t.getMessage}", t)
        		Seq()
    	}
	}
 
topiclist.html文件中对所得到的所有topic进行遍历：
 
	<li ng-repeat="g in topics" class="list-group-item"><a href="./#/topicdetail/{{g}}">{{g}}</a></li>
 
<li>/#/topicdetail/{{g}}链接在app.js中的定义为：
 
	.when("/topicdetail/:group", {templateUrl: "views/topic-detail.html",controller: "TopicDetailCtrl"})
 
TopicDetailCtrl的定义为：
 
	.controller("TopicDetailCtrl", ["$scope", "$interval", "$routeParams", "offsetinfo", function($scope, $interval, $routeParams, offsetinfo) {
    offsetinfo.topicDetail($routeParams.group, function(d) {
    	$scope.info = d;
    	$scope.loading=false;
    });
    $scope.loading=true;
	$scope.group = $routeParams.group;
	}])
 
会调用offsetinfo.topicDetail方法，topicDetail方法在app.js中的定义为：
 
	topicDetail: function(group, cb) {
    	return $resource("./topicdetails/:group").get({group:group}, groupPartitions(cb));
	}
 
topicdetails/:group这个path在OffsetGetterWeb中的定义为：
 
	case GET(Path(Seg("topicdetails" :: group :: Nil))) =>
        JsonContent ~> ResponseString(write(getTopicDetail(group, args)))
 
scala代码：
 
	def getTopicDetail(topic: String, args: OWArgs) = withOG(args) {
    	_.getTopicDetail(topic)
	}	
	/**
   	* returns details for a given topic such as the active consumers pulling off of it
  	 * @param topic
   	* @return
   	*/
	def getTopicDetail(topic: String): TopicDetails = {
    	val topicMap = getActiveTopicMap
    	if (topicMap.contains(topic)) {
      		TopicDetails(topicMap(topic).map(consumer => {
        		ConsumerDetail(consumer.toString)
      		}).toSeq)
    	} else {
      		TopicDetails(Seq(ConsumerDetail("Unable to find Active Consumers")))
    	}
	}
	/**
   	* returns a map of active topics-> list of consumers from zookeeper, ones that have IDS attached to them
   	*
   	* @return
   	*/
	def getActiveTopicMap: Map[String, Seq[String]] = {
    try {
	  	//获取所有的consumers，结果为group集合
      	ZkUtils.getChildren(zkClient, ZkUtils.ConsumersPath).flatMap {
        	group =>
          	try {
				//获取指定group的所有topic
            	ZkUtils.getConsumersPerTopic(zkClient, group).keySet.map {
              		key =>
                		key -> group
            	}
          	} catch {
            	case NonFatal(t) =>
              	error(s"could not get consumers for group $group", t)
              	Seq()
          	}
      	}.groupBy(_._1).mapValues {
        	_.unzip._2
      	}
    } catch {
      	case NonFatal(t) =>
        	error(s"could not get topic maps because of ${t.getMessage}", t)
        	Map()
    }
	}
 
返回的结果会在topic-detail.html中显示。
<li>index.html文件中的/#/activetopicsviz链接在app.js中的定义为：
 
	.when("/activetopicsviz", {templateUrl: "views/activetopics-viz.html",controller: "ActiveTopicsVizCtrl"})
 
ActiveTopicsVizCtrl的定义为：
 
	.controller("ActiveTopicsVizCtrl", ["$scope", "$interval", "$routeParams", "offsetinfo",function($scope, $interval, $routeParams, offsetinfo) {
    $scope.loading = true;
    offsetinfo.loadTopicConsumerViz($routeParams.group, function(d) {
    });
	}])
                        
会调用offsetinfo.loadTopicConsumerViz方法，loadTopicConsumerViz方法在app.js中的定义为：
 
	loadTopicConsumerViz: function(group, cb) {
    	cb(loadViz("#dataviz-container", "/activetopics"))
	}
 
/activetopics这个path在OffsetGetterWeb中的定义为：
 
	case GET(Path(Seg("activetopics" :: Nil))) =>
    	JsonContent ~> ResponseString(write(getActiveTopics(args)))
         
getActiveTopics方法上面已经讲过，loadViz方法是js的画图工具方法，暂时不讲。
<li>/#/clusterviz链接在app.js中的定义为：
 
	.when("/clusterviz", {templateUrl: "views/cluster-viz.html",controller: "ClusterVizCtrl"})
 
ClusterVizCtrl的定义为：
 
	.controller("ClusterVizCtrl", ["$scope", "$interval", "$routeParams", "offsetinfo",function($scope, $interval, $routeParams, offsetinfo) {
    $scope.loading = true;
    offsetinfo.loadClusterViz($routeParams.group, function(d) {
    });
	}])
 	  
offsetinfo.loadClusterViz方法在app.js中的定义为：
 
	loadClusterViz: function(group, cb) {
    	cb(loadViz("#dataviz-container", "/clusterlist"))
	}
 
/clusterlist这个path在OffsetGetterWeb中的定义为：
 
	case GET(Path(Seg("clusterlist" :: Nil))) =>
    	JsonContent ~> ResponseString(write(getClusterViz(args)))
	def getClusterViz(args: OWArgs) = withOG(args) {
    	_.getClusterViz
	}
	def getClusterViz: Node = {
	//获取集群中所有的broker
	//通过读取/brokers/ids路径数据获取所有的broker id，然后遍历这些id读取/brokers/ids/id路径获取具体的broker的元数据并返回
    val clusterNodes = ZkUtils.getAllBrokersInCluster(zkClient).map((broker) => {
      	Node(broker.getConnectionString(), Seq())
    })
    Node("KafkaCluster", clusterNodes)
	}
 
源码中最核心的部分基本上就这些了，下面的代码是java写的：
 
	ZkClient zkClient = new ZkClient("xxx.xxx.xxx.xxx:2182", 30000, 30000, 	ZKStringSerializer.getInstance());
	logger.info("kafka groupinfo----:" + ZkUtils.getChildren(zkClient, 	ZkUtils.ConsumersPath()));
	List<String> groups = JavaConversions.seqAsJavaList(ZkUtils.getChildren(zkClient, ZkUtils.ConsumersPath()));
	for (String group : groups) {
		Map<String, scala.collection.immutable.List<String>> topicConsumers = JavaConversions.asJavaMap(ZkUtils.getConsumersPerTopic(zkClient, group));
		for (Map.Entry<String, scala.collection.immutable.List<String>> entry : topicConsumers.entrySet()) {
			logger.info("topic name:" + entry.getKey());
			JavaConversions.asJavaList(entry.getValue());
		}
	}
	logger.info("kafka offsetinfo----:" + ZkUtils.getChildren(zkClient, ZkUtils.ConsumersPath() + "/" + Server.NS + "rtb/offsets"));
	List<String> topics = JavaConversions.seqAsJavaList(ZkUtils.getChildren(zkClient, ZkUtils.ConsumersPath() + "/" + Server.NS + "rtb/offsets"));
	for (String topic : topics) {
		logger.info("查询topic info：" + topic);
		List<String> tl = new ArrayList<String>();
		tl.add(topic);
		Map<String, Seq<Object>> topicInfo = JavaConversions.asJavaMap(ZkUtils.getPartitionsForTopics(zkClient, JavaConversions.asScalaBuffer(tl).toSeq()));
		List<Object> partitions = JavaConversions.asJavaList(topicInfo.get(topic).toSeq());
		logger.info(topic + "的partitions:" + partitions);
		for (Object obj : partitions) {
			Tuple2<String, Stat> data = ZkUtils.readData(zkClient, ZkUtils.ConsumersPath() + "/" + Server.NS + "rtb/offsets/" + topic + "/" + Integer.valueOf(obj.toString()));
			Tuple2<Option<String>, Stat> owners = ZkUtils.readDataMaybeNull(zkClient, ZkUtils.ConsumersPath() + "/" + Server.NS + "rtb/owners/" + topic + "/" + Integer.valueOf(obj.toString()));
			Option<Object> objs = ZkUtils.getLeaderForPartition(zkClient, topic, Integer.valueOf(obj.toString()));
			logger.info(topic + "的offset:" + data._1 + ",stat:" + data._2.toString() + ",owner:" + owners._1 + ",bid:" + objs.get());
			Tuple2<Option<String>, Stat> stats = ZkUtils.readDataMaybeNull(zkClient, ZkUtils.BrokerIdsPath() + "/" + objs.get());
			logger.info(stats.toString());
		}
	}
	static class ZKStringSerializer implements ZkSerializer {		static final ZKStringSerializer instance = new ZKStringSerializer();		private ZKStringSerializer() {		}		public byte[] serialize(Object data) throws ZkMarshallingError {			if (data == null) {				throw new NullPointerException();			}			return data.toString().getBytes();		}		public Object deserialize(byte[] bytes) throws ZkMarshallingError {			try {				return bytes == null ? null : new String(bytes, "UTF-8");			} catch (UnsupportedEncodingException e) {				e.printStackTrace();			}			return null;		}		public static ZKStringSerializer getInstance() {			return instance;		}	}
 