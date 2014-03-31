---
layout: post
title: "solr整合spatial查询"
category: 
- geohash
- solr
- spatial
tags: []
---








Solr的空间（即经纬度）搜索可以有多种实现方式，其字段类型可以是solr自带的 point（solr.PoinType，表示坐标）,location（solr.LatLonType，表示经纬 度），location_rpt(solr.SpatialRecursivePrefixTreeFieldType，可以由多边形组成的一个地方)。 这里主要讲location_rpt（其他两个比较简单），location_rpt中的属性distErrPct表示无坐标形状的精度，默认值为 0.025（需介于0到0.5），属性maxDistErr表示在geohash算法中均分的最终误差（均分20次后得到的值就为该值），units表示 用什么来度量距离，默认为degrees，即经纬度的度数，一度代表111.2千米（注意：目前solr中只支持degrees，如果不设置该值或者设置 成其他值会报Must specify units=”degrees” on field types with class xxx）。另外可以自定义字段类型，比如几何图形的，bbox形式的，点向量形式，四叉树形式的。自定义字段类型时需要继承 AbstractSpatialFieldType，并且重载init和newSpatialStrategy方 法，newSpatialStrategy方法返回的是继承于SpatialStrategy的类，主要用于对地理位置信息进行索引。在init方法中会 调用super.init(schema,args)方法即AbstractSpatialFieldType.init(IndexSchema schema,Map<String,String> args)方法，在该方法中会调用SpatialContextFactory.makeSpatialContext方 法，SpatialContextFactory是spatial4j项目里的类，在该方法中主要是获取SpatialContext，在 spatial4j中有两种SpatialContext一种是原生的SpatialContext，另一种是继承了SpatialContext的 JtsSpatialContext（需要jts包的支持，这种类型可以处理几何图形的位置信息），该方法首先会去获取 SpatialContextFactory，有两种方法：一种是获取配置在fieldType中的spatialContextFactory属性的 值，一种是获取系统变量SpatialContextFactory的值（可以通过 System.setProperty(“SpatialContextFactory”,xxx.class.getName())设值），这两种方法 各有优缺点，第一种方法可以对定义多种不同的处理factory，但是比较繁琐，每一个fieldType都要定义这个属性，第二种方法简单但是只能定义 一个factory，所以如果你的应用中只需要一个factory则采用第二种方式否则采用第一种方式。在spatial4j提供了两种factory即 SpatialContextFactory和JtsSpatialContextFactory。获取到factory后就可以通过它的 newSpatialContext方法获取到真正的SpatialContext了。</br></br>
这时初始化工作已经完 成，接下来就是开始做索引了，由于要将具体的经纬度值解析为某个具体的图形这样solr就可以通过spatial来搜索某个经纬度对应的其他的经纬度值 了，所以做索引的过程重要的是将经纬度解析为图形的过程，这个过程可以通过spatial4j的ShapeReadWriter(简单的坐标)或 JtsShapeReadWriter（复杂的几何图形）的readShape方法实现。
索引做完之后，就可以查询了，比如：
{% highlight objc %}
fq={!geofilt pt=45.15,-93.85 sfield=geohash d=5}就表示搜索离经纬度45.15,-93.85的距离为5千米的那些地点，这里是如何计算距离的呢，由于是2D平面，那么可以使用欧几里得定理来计算距离。
fq=geohash:[45,-94 TO 46,-93] 表示搜索45,-94到46,-93之间的一些地点。
fq=geohash:"Intersects(-74.093 41.042 -69.347 44.558)" 表示获取-74.093 41.042代表的平面与-69.347 44.558代表的平面相交的那些地点。这让我想起了很多电影里的场景。
fq=geohash:"IsWithin"(POLYGON((-10 30, -40 40, -10 -20, 40 20, 0 0, -10 30))) distErrPct=0" 表示搜索这些经纬度组成的多边形之内的那些地点。
{% endhighlight %}
在 搜索的结果列表中可以对列表进行按距离排序，只要在搜索时指定score=distance（不指定这个话那么查询结果的score都为1.0则不存在排 序之说了）和sort=score asc(或desc)即可。如&fl=*,score&sort=score asc&q= {!geofilt score=distance sfield=geohash pt=54.729696,-98.525391 d=10}  注 意：这种查询不能放在fq进行查询，否则没有效果。</br></br>
好了，solr和spatial的整合基本上已经完成，接下来主要工作是研究spatial4j是如何将经纬度解析为图形的。