---
layout: post
title: "spark之路第十一课——spark MLlib之数据类型"
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
###本地矩阵
本地矩阵拥有数字类型的行和列索引以及double类型的值，MLlib支持密集矩阵，其entry值是存储在一个double的数组中的，如下矩阵：
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
<mrow>
  <mo>(</mo>
  <mtable rowspacing="4pt" columnspacing="1em">
    <mtr>
      <mtd>
        <mn>1.0</mn>
      </mtd>
      <mtd>
        <mn>2.0</mn>
      </mtd>
    </mtr>
    <mtr>
      <mtd>
        <mn>3.0</mn>
      </mtd>
      <mtd>
        <mn>4.0</mn>
      </mtd>
    </mtr>
    <mtr>
      <mtd>
        <mn>5.0</mn>
      </mtd>
      <mtd>
        <mn>6.0</mn>
      </mtd>
    </mtr>
  </mtable>
  <mo>)</mo>
</mrow>
</math>
在一维数组中的存储方式为[1.0, 3.0, 5.0, 2.0, 4.0, 6.0]。  
本地矩阵的基类为Matrix，DenseMatrix是其的一个实现，推荐使用Matrices的工厂方法来创建本地矩阵：

	import org.apache.spark.mllib.linalg.{Matrix, Matrices}

	// Create a dense matrix ((1.0, 2.0), (3.0, 4.0), (5.0, 6.0))
	val dm: Matrix = Matrices.dense(3, 2, Array(1.0, 3.0, 5.0, 2.0, 4.0, 6.0))
###分散矩阵
分散矩阵拥有long类型的行和列索引以及double类型的值，分散存储在多个RDD中，选择正确的格式来存储大的分散矩阵是很重要的，将分散矩阵转换为不同格式需要全局shuffle，这个代价是高的。到目前为止有三种分担矩阵类型。
####RowMatrix
RowMatrix是面向行的分散矩阵，它使没有行索引的，它的每行都代表一个本地向量，所以列的数量必须在整数范围中。可以使用RDD[Vector]实例来创建RowMatrix，这样就可以计算列的统计信息了：

	import org.apache.spark.mllib.linalg.Vector
	import org.apache.spark.mllib.linalg.distributed.RowMatrix

	val rows: RDD[Vector] = ... // an RDD of local vectors
	// Create a RowMatrix from an RDD[Vector].
	val mat: RowMatrix = new RowMatrix(rows)

	// Get its size.
	val m = mat.numRows()
	val n = mat.numCols()
####IndexedRowMatrix
IndexedRowMatrix与RowMatrix是很相似的，但拥有行索引。它的每行使用long类型的索引和一个本地向量来表示。可以使用RDD[IndexedRow]来创建IndexedRowMatrix，IndexedRow是(Long,Vector)的封装，也可以通过将行索引去除而由IndexedRowMatrix变为RowMatrix：

	import org.apache.spark.mllib.linalg.distributed.{IndexedRow, IndexedRowMatrix, RowMatrix}

	val rows: RDD[IndexedRow] = ... // an RDD of indexed rows
	// Create an IndexedRowMatrix from an RDD[IndexedRow].
	val mat: IndexedRowMatrix = new IndexedRowMatrix(rows)

	// Get its size.
	val m = mat.numRows()
	val n = mat.numCols()

	// Drop its row indices.
	val rowMat: RowMatrix = mat.toRowMatrix()
####CoordinateMatrix
CoordinateMatrix是由有行和列索引以及值的多个entry组成，每个entry格式为(i: Long, j: Long, value: Double)，i是行索引，j是列索引，value就是entry值。CoordinateMatrix可以通过RDD[MatrixEntry]实例创建，MatrixEntry是(Long,Long,Double)的封装，它可以通过调用toIndexRowMatrix转换为拥有稀疏行的IndexedRowMatrix：

	import org.apache.spark.mllib.linalg.distributed.{CoordinateMatrix, MatrixEntry}

	val entries: RDD[MatrixEntry] = ... // an RDD of matrix entries
	// Create a CoordinateMatrix from an RDD[MatrixEntry].
	val mat: CoordinateMatrix = new CoordinateMatrix(entries)

	// Get its size.
	val m = mat.numRows()
	val n = mat.numCols()

	// Convert it to an IndexRowMatrix whose rows are sparse vectors.
	val indexedRowMatrix = mat.toIndexedRowMatrix()