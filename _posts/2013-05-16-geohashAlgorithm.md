---
layout: post
title: "geohash和查找附近"
category: 
- geohash
- solr
- spatial
tags: []
---







对于空间经纬度查询的时候，我们需要将经纬度转换为字符串，其中的好处是便于存储和查询比较。而转换的算法中用的较多的 是geohash算法（http://en.wikipedia.org/wiki/Geohash），看完该wiki对geohash就已经有了大致的 了解了，原理其实很简单，具体实现网上也一大堆，各种语言的都有。有了代表经纬度的字符串之后，我们就可以根据该字符串来搜索附近的一些地方了，比如现在 由经纬度转化为字符串为bcdefghj(存储在数据库的geohash字段)，这时候搜索这个经纬度附近的地点可以使用sql:where geohash like ‘bcdefg%’，查询出附近的地点之后再使用geohash算法对这些字符串进行decode即可获取真实的经纬度地址了，然后再将这些经纬度地址渲 染到地图上用户就可以看到了。</br></br>
问题：</br>
1.上面的like ‘bcdefg%’是如何确定的呢？这里有个经验总结（取于网上）：前6为相同的就说明查找附近1千米的地点，当然相同位数越多地址就越近。</br></br>
2. 如果某个用户的地址频繁变化那么查询对数据库的压力是很大的，做成索引也不可避免该问题，比如某用户需要查找他附近1千米的其他用户，但是该用户的地址是 经常变的，所以使用数据库存储和查询毕然会给数据库带来巨大压力，这时候就可以使用缓存（并不真正放入数据库或索引中）来实现用户信息的存储，缓存的格式 为substring(geohash,0,6)àuserID,即在缓存中substring(geohash,0,6)这个键对应的是一个 userID的list，所以当用户地址改变了我只要获取他的geohash前六位就可以根据该值从缓存中取出一些userIDs，取出的userIDs 就是该用户附近1千米的其他用户。基本的流程为：每当用户的位置改变时，先从缓存中的substring(geohash,0,6)键对应的list删除 该用户id，然后获取该用户最新的位置并获取geohash值的前六位并将该值作为键放入缓存中，其对应的值就是该用户的userID，这样我就能收集到 geohash值前六位对应的一些userID，并且是实时更新的。</br></br>
但是该用什么缓存产品呢？市面上比较成熟的 有memcache和redis，据我了解memcache过于笨重，redis比较轻量级使用也很简单，其中的命令有很多可以很好地支持各种需求。所以 这里使用redis的java客户端jedis来实现上述功能。若以后数据量忒大的话可以考虑做成redis集群来减少单个redis服务器的压力。</br></br>
如若需要对附近的一些地点离我的距离进行排序的话，则可以使用距离公式来算出具体的距离值，公式有使用余弦定理及弧度计算的和google使用的：
{% highlight objc %}
余弦定理及弧度计算：distance = acos(cos(latitude1)*cos(latitude2)*cos(longitude1- longitude2)+sin(latitude1)*sin(latitude2))*radius;
Google 使用的：distance=2*asin(sqrt(pow(sin((latitude1- latitude2)/2),2)+cos(latitude1)*cos(latitude2)*pow(sin((langitude1-langitude2)/2),2)))*radius; （其中latitude代表纬度，longitude代表经度，radius是地球的半径，acos是反余弦,asin是反正弦）
{% endhighlight %}
一般会使用google的那个，会更精确。
{% highlight objc %}
知识（网上的）：在纬度相等的情况下：
经度每隔0.00001度，距离相差约1米；
每隔0.0001度，距离相差约10米；
每隔0.001度，距离相差约100米；
每隔0.01度，距离相差约1000米；
每隔0.1度，距离相差约10000米。
在经度相等的情况下：
纬度每隔0.00001度，距离相差约1.1米；
每隔0.0001度，距离相差约11米；
每隔0.001度，距离相差约111米；
每隔0.01度，距离相差约1113米；
每隔0.1度，距离相差约11132米。
{% endhighlight %}