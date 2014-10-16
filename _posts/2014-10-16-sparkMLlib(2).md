---
layout: post
title: "spark之路第十二课——spark MLlib之基本统计"
description: "spark学习系列第十二篇"
category: 
- spark
tags: []
---


###基本统计
MLlib提供了通过Statistics的colStats方法来统计RDD[Vector]的列，colStats()方法返回MultivariateStatisticalSummary实例，包含列的最大值，最小值，平均值，方差，非零值，总数等信息：

	import org.apache.spark.mllib.linalg.Vector
	import org.apache.spark.mllib.stat.{MultivariateStatisticalSummary, Statistics}

	val observations: RDD[Vector] = ... // an RDD of Vectors

	// Compute column summary statistics.
	val summary: MultivariateStatisticalSummary = Statistics.colStats(observations)
	println(summary.mean) // a dense vector containing the mean value for each column
	println(summary.variance) // column-wise variance
	println(summary.numNonzeros) // number of nonzeros in each column
###相关性
计算两组数据的相关性在统计操作中是很常见的，MLlib当前支持的相关性方法有皮尔逊和斯皮尔曼两个方法，Statistics提供了计算两个RDD的相关性的方法，当输入是两个RDD[Double]时则输出一个Double数，当输入是一个RDD[Vector]那么会输出相关的矩阵：

	import org.apache.spark.SparkContext
	import org.apache.spark.mllib.linalg._
	import org.apache.spark.mllib.stat.Statistics

	val sc: SparkContext = ...

	val seriesX: RDD[Double] = ... // a series
	val seriesY: RDD[Double] = ... // must have the same number of partitions and cardinality as seriesX

	// compute the correlation using Pearson's method. Enter "spearman" for Spearman's method. If a 
	// method is not specified, Pearson's method will be used by default. 
	val correlation: Double = Statistics.corr(seriesX, seriesY, "pearson")

	val data: RDD[Vector] = ... // note that each Vector is a row and not a column

	// calculate the correlation matrix using Pearson's method. Use "spearman" for Spearman's method.
	// If a method is not specified, Pearson's method will be used by default. 
	val correlMatrix: Matrix = Statistics.corr(data, "pearson")
###多层抽样
不像MLlib中其他的统计方法，多层抽样方法如sampleByKey和sampleByKeyExact可以在RDD的键值对上进行操作，对于多层抽样来说，键可以被当做标签，值可以看做是指定的属性，例如，键可以为男人或女人，或文档id，对应的值可以为这些人年龄的列表或文档中单词列表，sampleByKey方法会随机决定某个观察值是否需要抽样，因此需要提供一个确定的抽样数量，sampleByKeyExact比sampleByKey需要更多明确的资源。  
sampleByKeyExact()方法允许用户抽取确切的<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mo fence="false" stretchy="false">&#x2308;<!-- ⌈ --></mo>
  <msub>
    <mi>f</mi>
    <mi>k</mi>
  </msub>
  <mo>&#x22C5;<!-- ⋅ --></mo>
  <msub>
    <mi>n</mi>
    <mi>k</mi>
  </msub>
  <mo fence="false" stretchy="false">&#x2309;<!-- ⌉ --></mo>
  <mspace width="thinmathspace" />
  <mi mathvariant="normal">&#x2200;<!-- ∀ --></mi>
  <mi>k</mi>
  <mo>&#x2208;<!-- ∈ --></mo>
  <mi>K</mi>
</math>
数据，其中<math xmlns="http://www.w3.org/1998/Math/MathML">
<msub>
  <mi>f</mi>
  <mi>k</mi>
</msub>
</math>是键k的期望银子，<math xmlns="http://www.w3.org/1998/Math/MathML">
<msub>
  <mi>n</mi>
  <mi>k</mi>
</msub>
</math>是键值对中键为k的数量，<math xmlns="http://www.w3.org/1998/Math/MathML">
<mi>K</mi>
</math>是键的集合。没有替换的抽样需要一个额外的RDD来确保抽样数量，而带有替换的抽样需要两个额外的RDD：

	import org.apache.spark.SparkContext
	import org.apache.spark.SparkContext._
	import org.apache.spark.rdd.PairRDDFunctions

	val sc: SparkContext = ...

	val data = ... // an RDD[(K, V)] of any key value pairs
	val fractions: Map[K, Double] = ... // specify the exact fraction desired from each key

	// Get an exact sample from each stratum
	val approxSample = data.sampleByKey(withReplacement = false, fractions)
	val exactSample = data.sampleByKeyExact(withReplacement = false, fractions)
