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
###假设测试
MLlib目前支持皮尔逊的平方（<math xmlns="http://www.w3.org/1998/Math/MathML">
<msup>
  <mi>&#x03C7;<!-- χ --></mi>
  <mn>2</mn>
</msup>
</math>）测试以得到好的和独立的结果。输入数据类型决定了测试是否是好的或独立，好的测试需要输入类型为Vector，而独立的测试需要Matrix作为输入。  
MLlib也支持输入类型RDD[LabeledPoint]以通过平方独立测试来开启特征选择。  
Statistics提供了运行皮尔逊平方测试的方法，下面的例子示范了如何运行和解释假设测试：

	import org.apache.spark.SparkContext
	import org.apache.spark.mllib.linalg._
	import org.apache.spark.mllib.regression.LabeledPoint
	import org.apache.spark.mllib.stat.Statistics._

	val sc: SparkContext = ...

	val vec: Vector = ... // a vector composed of the frequencies of events

	// compute the goodness of fit. If a second vector to test against is not supplied as a parameter, 
	// the test runs against a uniform distribution.  
	val goodnessOfFitTestResult = Statistics.chiSqTest(vec)
	println(goodnessOfFitTestResult) // summary of the test including the p-value, degrees of freedom, 
                                 // test statistic, the method used, and the null hypothesis.

	val mat: Matrix = ... // a contingency matrix

	// conduct Pearson's independence test on the input contingency matrix
	val independenceTestResult = Statistics.chiSqTest(mat) 
	println(independenceTestResult) // summary of the test including the p-value, degrees of freedom...

	val obs: RDD[LabeledPoint] = ... // (feature, label) pairs.

	// The contingency table is constructed from the raw (feature, label) pairs and used to conduct
	// the independence test. Returns an array containing the ChiSquaredTestResult for every feature 
	// against the label.
	val featureTestResults: Array[ChiSqTestResult] = Statistics.chiSqTest(obs)
	var i = 1
	featureTestResults.foreach { result =>
    println(s"Column $i:\n$result")
    i += 1
	} // summary of the test
###产生随机数
生成随机数对随机算法，原型和性能测试是很有用的，MLlib支持使用i.i.d生成随机RDD。  
RandomRDDs提供了生成随机double的RDD或向量RDD的工厂方法：

	import org.apache.spark.SparkContext
	import org.apache.spark.mllib.random.RandomRDDs._

	val sc: SparkContext = ...

	// Generate a random double RDD that contains 1 million i.i.d. values drawn from the
	// standard normal distribution `N(0, 1)`, evenly distributed in 10 partitions.
	val u = normalRDD(sc, 1000000L, 10)
	// Apply a transform to get a random double RDD following `N(1, 4)`.
	val v = u.map(x => 1.0 + 2.0 * x)