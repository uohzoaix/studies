---
layout: post
title: "spark之路第十一课——spark machine learning library"
description: "spark学习系列第十一篇"
category: 
- spark
tags: []
---


MLlib是spark的扩展机器学习库，包含常见的学习算法和工具如分类、回归、聚类、协同过滤、降维等。
###数据类型
MLlib支持在单独机器上存在本地向量和矩阵，同时也支持使用一个或多个RDD进行备份分布式矩阵。本地向量和矩阵是简单的数据模型，下面的一些线性代数操作是[Breeze](http://www.scalanlp.org/)和[jblas](http://mikiobraun.github.io/jblas/)提供的。
####本地向量
本地向量具有整数类型，基于0的索引和双类型值的特性，MLlib支持两种类型的向量：密集和稀疏。密集向量使用一个二维数组来存放向量值，稀疏向量使用两个并行的数组来存放索引和对应的值。例如(1.0,0.0,3.0)可以使用密集格式[1.0,0.0,3.0]和使用稀疏格式(3,[0,2],[1.0,3.0])来表示，稀疏格式中的3表示向量的大小。  
本地向量的基类是Vector，它的两个实现为：DenseVector和SparseVector，建议使用Vectors中的工厂方法来创建本地向量：

	import org.apache.spark.mllib.linalg.{Vector, Vectors}
	// Create a dense vector (1.0, 0.0, 3.0).
	val dv: Vector = Vectors.dense(1.0, 0.0, 3.0)
	// Create a sparse vector (1.0, 0.0, 3.0) by specifying its indices and values 	corresponding to nonzero entries.
	val sv1: Vector = Vectors.sparse(3, Array(0, 2), Array(1.0, 3.0))
	// Create a sparse vector (1.0, 0.0, 3.0) by specifying its nonzero entries.
	val sv2: Vector = Vectors.sparse(3, Seq((0, 1.0), (2, 3.0)))
注意：scala默认会导入scala.collection.immutable.Vector，所以必须显式导入org.apache.spark.mllib.linalg.Vector。
####标记点
标记点是一个本地向量，可以是密集型也可以是稀疏型的，在MLlib中，标记点被用在有监督的学习算法中。对于二分分类来说，标签可以为0或1，对于多分类来说，标签是从0开始递增的：0，1，2，……  
标记点由LabeldPoint类实现：

	import org.apache.spark.mllib.linalg.Vectors
	import org.apache.spark.mllib.regression.LabeledPoint

	// Create a labeled point with a positive label and a dense feature vector.
	val pos = LabeledPoint(1.0, Vectors.dense(1.0, 0.0, 3.0))

	// Create a labeled point with a negative label and a sparse feature vector.
	val neg = LabeledPoint(0.0, Vectors.sparse(3, Array(0, 2), Array(1.0, 3.0)))
#####稀疏数据
在平常的实验当中稀疏的训练数据是很常见的，MLlib支持使用LIBSVM格式读取训练例子，该格式的每行代表一个标记的稀疏特征向量：

	label index1:value1 index2:value2 ...
这种格式的标签是基于1的，在数据加载完成后特征索引会变为基于0的。  
MLUtils.loadLibSVMFile方法会读取训练例子并已LIVSVM格式存储：

	import org.apache.spark.mllib.regression.LabeledPoint
	import org.apache.spark.mllib.util.MLUtils
	import org.apache.spark.rdd.RDD

	val examples: RDD[LabeledPoint] = MLUtils.loadLibSVMFile(sc, "data/mllib/sample_libsvm_data.txt")
