---
layout: post
title: "spark之路第十三课——spark MLlib之分类和回归"
description: "spark学习系列第十三篇"
category: 
- spark
tags: []
---

MLlib支持多种方法用来处理二分分类，多类分类以及回归分析，下表列出了问题及对应的处理方法：
<table>
<thead>
<tr class="header">
<th align="left">问题类型</th>
<th align="left">支持的方法</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">二分分类</td>
<td align="left">现行SVM，逻辑回归，决策树，贝叶斯</td>
</tr>
<tr class="even">
<td align="left">多类分类</td>
<td align="left">决策树，贝叶斯</td>
</tr>
<tr class="odd">
<td align="left">回归</td>
<td align="left">线性最小二乘法，套索，岭回归</td>
</tr>
</tbody>
</table>
下面是对这些方法更详细的描述：
###线性方法
####数学表达式
许多标准的机器学习方法可以表达为凸的优化问题，例如，找到凸函数
<math xmlns="http://www.w3.org/1998/Math/MathML">
<mi>f</mi>
</math>
的极小值取决于变向量
<math xmlns="http://www.w3.org/1998/Math/MathML">
<mrow class="MJX-TeXAtom-ORD">
  <mi mathvariant="bold">w</mi>
</mrow>
</math>
，该变向量包含
<math xmlns="http://www.w3.org/1998/Math/MathML">
<mi>d</mi>
</math>
个entry，可以将该问题转换为优化问题
<math xmlns="http://www.w3.org/1998/Math/MathML">
  <munder>
    <mo form="prefix" movablelimits="true">min</mo>
    <mrow class="MJX-TeXAtom-ORD">
      <mrow class="MJX-TeXAtom-ORD">
        <mi mathvariant="bold">w</mi>
      </mrow>
      <mo>&#x2208;<!-- ∈ --></mo>
      <msup>
        <mrow class="MJX-TeXAtom-ORD">
          <mi mathvariant="double-struck">R</mi>
        </mrow>
        <mi>d</mi>
      </msup>
    </mrow>
  </munder>
  <mspace width="thickmathspace" />
  <mi>f</mi>
  <mo stretchy="false">(</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <mo stretchy="false">)</mo>
</math>
，其中目标函数为如下形式：
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
<mtable>
  <mlabeledtr>
    <mtd id="mjx-eqn-eqregPrimal">
      <mtext>(1)</mtext>
    </mtd>
    <mtd>
      <mi>f</mi>
      <mo stretchy="false">(</mo>
      <mrow class="MJX-TeXAtom-ORD">
        <mi mathvariant="bold">w</mi>
      </mrow>
      <mo stretchy="false">)</mo>
      <mo>:=</mo>
      <mi>&#x03BB;<!-- λ --></mi>
      <mspace width="thinmathspace" />
      <mi>R</mi>
      <mo stretchy="false">(</mo>
      <mrow class="MJX-TeXAtom-ORD">
        <mi mathvariant="bold">w</mi>
      </mrow>
      <mo stretchy="false">)</mo>
      <mo>+</mo>
      <mfrac>
        <mn>1</mn>
        <mi>n</mi>
      </mfrac>
      <munderover>
        <mo>&#x2211;<!-- ∑ --></mo>
        <mrow class="MJX-TeXAtom-ORD">
          <mi>i</mi>
          <mo>=</mo>
          <mn>1</mn>
        </mrow>
        <mi>n</mi>
      </munderover>
      <mi>L</mi>
      <mo stretchy="false">(</mo>
      <mrow class="MJX-TeXAtom-ORD">
        <mi mathvariant="bold">w</mi>
      </mrow>
      <mo>;</mo>
      <msub>
        <mrow class="MJX-TeXAtom-ORD">
          <mi mathvariant="bold">x</mi>
        </mrow>
        <mi>i</mi>
      </msub>
      <mo>,</mo>
      <msub>
        <mi>y</mi>
        <mi>i</mi>
      </msub>
      <mo stretchy="false">)</mo>
      <mtext>&#xA0;</mtext>
      <mo>.</mo>
    </mtd>
  </mlabeledtr>
</mtable>
</math>
其中向量
<math xmlns="http://www.w3.org/1998/Math/MathML">
  <msub>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">x</mi>
    </mrow>
    <mi>i</mi>
  </msub>
  <mo>&#x2208;<!-- ∈ --></mo>
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="double-struck">R</mi>
    </mrow>
    <mi>d</mi>
  </msup>
</math>是训练数据集，<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mn>1</mn>
  <mo>&#x2264;<!-- ≤ --></mo>
  <mi>i</mi>
  <mo>&#x2264;<!-- ≤ --></mo>
  <mi>n</mi>
</math>并且<math xmlns="http://www.w3.org/1998/Math/MathML">
  <msub>
    <mi>y</mi>
    <mi>i</mi>
  </msub>
  <mo>&#x2208;<!-- ∈ --></mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="double-struck">R</mi>
  </mrow>
</math>
是相应的需要预测的标签。当<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi>L</mi>
  <mo stretchy="false">(</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <mo>;</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo>,</mo>
  <mi>y</mi>
  <mo stretchy="false">)</mo>
</math>可以表示为<math xmlns="http://www.w3.org/1998/Math/MathML">
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mi>x</mi>
</math>和<math xmlns="http://www.w3.org/1998/Math/MathML">
<mi>y</mi>
</math>时可以调用linear方法，许多的MLlib的分类和回归算法都属于数学表达式这个种类。  
方法<math xmlns="http://www.w3.org/1998/Math/MathML">
<mi>f</mi>
</math>含有两个部分：正则控制模型的复杂度和使用损失测量训练数据上模型的错误。损失函数<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi>L</mi>
  <mo stretchy="false">(</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <mo>;</mo>
  <mo>.</mo>
  <mo stretchy="false">)</mo>
</math>在<math xmlns="http://www.w3.org/1998/Math/MathML">
<mrow class="MJX-TeXAtom-ORD">
  <mi mathvariant="bold">w</mi>
</mrow>
</math>中是典型的凸函数，正则参数<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi>&#x03BB;<!-- λ --></mi>
  <mo>&#x2265;<!-- ≥ --></mo>
  <mn>0</mn>
</math>定义了最小化损失（如训练错误）和最小化模型复杂度（如避免过度）之间的取舍。
####损失函数
下表总结了损失函数和MLlib支持的方法的变化率：
<table>
<thead>
<tr class="header">
<th align="left">名称</th>
<th align="left">损失函数<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi>L</mi>
  <mo stretchy="false">(</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <mo>;</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo>,</mo>
  <mi>y</mi>
  <mo stretchy="false">)</mo>
</math></th>
<th align="left">变化率</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">合页损失</td>
<td align="left"><math xmlns="http://www.w3.org/1998/Math/MathML">
  <mo form="prefix" movablelimits="true">max</mo>
  <mo fence="false" stretchy="false">{</mo>
  <mn>0</mn>
  <mo>,</mo>
  <mn>1</mn>
  <mo>&#x2212;<!-- − --></mo>
  <mi>y</mi>
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo fence="false" stretchy="false">}</mo>
  <mo>,</mo>
  <mspace width="1em" />
  <mi>y</mi>
  <mo>&#x2208;<!-- ∈ --></mo>
  <mo fence="false" stretchy="false">{</mo>
  <mo>&#x2212;<!-- − --></mo>
  <mn>1</mn>
  <mo>,</mo>
  <mo>+</mo>
  <mn>1</mn>
  <mo fence="false" stretchy="false">}</mo>
</math></td>
<td align="left"><math xmlns="http://www.w3.org/1998/Math/MathML">
<mrow>
  <mo>{</mo>
  <mtable columnalign="left left" rowspacing=".2em" columnspacing="1em">
    <mtr>
      <mtd>
        <mo>&#x2212;<!-- − --></mo>
        <mi>y</mi>
        <mo>&#x22C5;<!-- ⋅ --></mo>
        <mrow class="MJX-TeXAtom-ORD">
          <mi mathvariant="bold">x</mi>
        </mrow>
      </mtd>
      <mtd>
        <mtext>if&#xA0;</mtext>
        <mrow class="MJX-TeXAtom-ORD">
          <mi>y</mi>
          <msup>
            <mrow class="MJX-TeXAtom-ORD">
              <mi mathvariant="bold">w</mi>
            </mrow>
            <mi>T</mi>
          </msup>
          <mrow class="MJX-TeXAtom-ORD">
            <mi mathvariant="bold">x</mi>
          </mrow>
          <mo>&lt;</mo>
          <mn>1</mn>
        </mrow>
        <mo>,</mo>
      </mtd>
    </mtr>
    <mtr>
      <mtd>
        <mn>0</mn>
      </mtd>
      <mtd>
        <mtext>otherwise</mtext>
        <mo>.</mo>
      </mtd>
    </mtr>
  </mtable>
</mrow>
</math></td>
</tr>
<tr class="even">
<td align="left">逻辑损失</td>
<td align="left"><math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi>log</mi>
  <mo>&#x2061;<!-- ⁡ --></mo>
  <mo stretchy="false">(</mo>
  <mn>1</mn>
  <mo>+</mo>
  <mi>exp</mi>
  <mo>&#x2061;<!-- ⁡ --></mo>
  <mo stretchy="false">(</mo>
  <mo>&#x2212;<!-- − --></mo>
  <mi>y</mi>
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo stretchy="false">)</mo>
  <mo stretchy="false">)</mo>
  <mo>,</mo>
  <mspace width="1em" />
  <mi>y</mi>
  <mo>&#x2208;<!-- ∈ --></mo>
  <mo fence="false" stretchy="false">{</mo>
  <mo>&#x2212;<!-- − --></mo>
  <mn>1</mn>
  <mo>,</mo>
  <mo>+</mo>
  <mn>1</mn>
  <mo fence="false" stretchy="false">}</mo>
</math></td>
<td align="left"><math xmlns="http://www.w3.org/1998/Math/MathML">
  <mo>&#x2212;<!-- − --></mo>
  <mi>y</mi>
  <mrow>
    <mo>(</mo>
    <mn>1</mn>
    <mo>&#x2212;<!-- − --></mo>
    <mfrac>
      <mn>1</mn>
      <mrow>
        <mn>1</mn>
        <mo>+</mo>
        <mi>exp</mi>
        <mo>&#x2061;<!-- ⁡ --></mo>
        <mo stretchy="false">(</mo>
        <mo>&#x2212;<!-- − --></mo>
        <mi>y</mi>
        <msup>
          <mrow class="MJX-TeXAtom-ORD">
            <mi mathvariant="bold">w</mi>
          </mrow>
          <mi>T</mi>
        </msup>
        <mrow class="MJX-TeXAtom-ORD">
          <mi mathvariant="bold">x</mi>
        </mrow>
        <mo stretchy="false">)</mo>
      </mrow>
    </mfrac>
    <mo>)</mo>
  </mrow>
  <mo>&#x22C5;<!-- ⋅ --></mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
</math></td>
</tr>
<tr class="odd">
<td align="left">平方损失</td>
<td align="left"><math xmlns="http://www.w3.org/1998/Math/MathML">
  <mfrac>
    <mn>1</mn>
    <mn>2</mn>
  </mfrac>
  <mo stretchy="false">(</mo>
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo>&#x2212;<!-- − --></mo>
  <mi>y</mi>
  <msup>
    <mo stretchy="false">)</mo>
    <mn>2</mn>
  </msup>
  <mo>,</mo>
  <mspace width="1em" />
  <mi>y</mi>
  <mo>&#x2208;<!-- ∈ --></mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="double-struck">R</mi>
  </mrow>
</math></td>
<td align="left"><math xmlns="http://www.w3.org/1998/Math/MathML">
  <mo stretchy="false">(</mo>
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo>&#x2212;<!-- − --></mo>
  <mi>y</mi>
  <mo stretchy="false">)</mo>
  <mo>&#x22C5;<!-- ⋅ --></mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
</math></td>
</tr>
</tbody>
</table>
####正则化
正则化的目的是为了使用简单的模型和避免过度设计，目前MLlib支持下列正则化方式：
<table>
<thead>
<tr class="header">
<th align="left">名称</th>
<th align="left">正则化方法<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi>R</mi>
  <mo stretchy="false">(</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <mo stretchy="false">)</mo>
</math></th>
<th align="left">变化率</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">0（未正则化）</td>
<td align="left">0</td>
<td align="left"><math xmlns="http://www.w3.org/1998/Math/MathML">
<mrow class="MJX-TeXAtom-ORD">
  <mn mathvariant="bold">0</mn>
</mrow>
</math></td>
</tr>
<tr class="even">
<td align="left">L2</td>
<td align="left"><math xmlns="http://www.w3.org/1998/Math/MathML">
  <mfrac>
    <mn>1</mn>
    <mn>2</mn>
  </mfrac>
  <mo>&#x2225;<!-- ∥ --></mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <msubsup>
    <mo fence="false" stretchy="false">&#x2225;<!-- ∥ --></mo>
    <mn>2</mn>
    <mn>2</mn>
  </msubsup>
</math></td>
<td align="left"><math xmlns="http://www.w3.org/1998/Math/MathML">
<mrow class="MJX-TeXAtom-ORD">
  <mi mathvariant="bold">w</mi>
</mrow>
</math></td>
</tr>
<tr class="odd">
<td align="left">L1</td>
<td align="left"><math xmlns="http://www.w3.org/1998/Math/MathML">
  <mo fence="false" stretchy="false">&#x2225;<!-- ∥ --></mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <msub>
    <mo fence="false" stretchy="false">&#x2225;<!-- ∥ --></mo>
    <mn>1</mn>
  </msub>
</math></td>
<td align="left"><math xmlns="http://www.w3.org/1998/Math/MathML">
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="normal">s</mi>
    <mi mathvariant="normal">i</mi>
    <mi mathvariant="normal">g</mi>
    <mi mathvariant="normal">n</mi>
  </mrow>
  <mo stretchy="false">(</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <mo stretchy="false">)</mo>
</math></td>
</tr>
</tbody>
</table>
表中<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="normal">s</mi>
    <mi mathvariant="normal">i</mi>
    <mi mathvariant="normal">g</mi>
    <mi mathvariant="normal">n</mi>
  </mrow>
  <mo stretchy="false">(</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <mo stretchy="false">)</mo>
</math>是由<math xmlns="http://www.w3.org/1998/Math/MathML">
<mrow class="MJX-TeXAtom-ORD">
  <mi mathvariant="bold">w</mi>
</mrow>
</math>中所有entry的符号（<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mo>&#x00B1;<!-- ± --></mo>
  <mn>1</mn>
</math>）组成。
###二分分类
二分分类旨在将item分成两个种类：积极和消极。MLlib支持两个线性方法：线性支持向量机（SVM）和逻辑回归，这两种方法都支持L1和L2正则化变体，在MLlib中训练数据集表示为LabeledPoint的一个RDD，在本文的数学表达式中，训练标签<math xmlns="http://www.w3.org/1998/Math/MathML">
<mi>y</mi>
</math>表示为<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mo>+</mo>
  <mn>1</mn>
</math>（正）和<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mo>-</mo>
  <mn>1</mn>
</math>（负），然而在MLlib中使用<math xmlns="http://www.w3.org/1998/Math/MathML">
<mn>0</mn>
</math>来表示负的。
####线性支持向量机
线性SVM是大规模分类任务的标准方法，它是由损失函数组成的线性方法：
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>L</mi>
  <mo stretchy="false">(</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <mo>;</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo>,</mo>
  <mi>y</mi>
  <mo stretchy="false">)</mo>
  <mo>:=</mo>
  <mo form="prefix" movablelimits="true">max</mo>
  <mo fence="false" stretchy="false">{</mo>
  <mn>0</mn>
  <mo>,</mo>
  <mn>1</mn>
  <mo>&#x2212;<!-- − --></mo>
  <mi>y</mi>
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo fence="false" stretchy="false">}</mo>
  <mo>.</mo>
</math>
默认，线性SVM使用L2正则化来进行训练，同时也只是L1正则化，线性SVM算法输出一个SVM模型，给定表示为<math xmlns="http://www.w3.org/1998/Math/MathML">
<mrow class="MJX-TeXAtom-ORD">
  <mi mathvariant="bold">x</mi>
</mrow>
</math>的新数据点，那么模型可通过<math xmlns="http://www.w3.org/1998/Math/MathML">
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
</math>进行预测，默认如果<math xmlns="http://www.w3.org/1998/Math/MathML">
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo>&#x2265;<!-- ≥ --></mo>
  <mn>0</mn>
</math>那么输出的是正的，否则是负的。
####逻辑回归
逻辑回归在预测二分答复中应用广泛，表达式为：
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>L</mi>
  <mo stretchy="false">(</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <mo>;</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo>,</mo>
  <mi>y</mi>
  <mo stretchy="false">)</mo>
  <mo>:=</mo>
  <mi>log</mi>
  <mo>&#x2061;<!-- ⁡ --></mo>
  <mo stretchy="false">(</mo>
  <mn>1</mn>
  <mo>+</mo>
  <mi>exp</mi>
  <mo>&#x2061;<!-- ⁡ --></mo>
  <mo stretchy="false">(</mo>
  <mo>&#x2212;<!-- − --></mo>
  <mi>y</mi>
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo stretchy="false">)</mo>
  <mo stretchy="false">)</mo>
  <mo>.</mo>
</math>
逻辑回归算法输出一个逻辑回归模型，对于给定的<math xmlns="http://www.w3.org/1998/Math/MathML">
<mrow class="MJX-TeXAtom-ORD">
  <mi mathvariant="bold">x</mi>
</mrow>
</math>数据点，模型可通过应用逻辑函数
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="normal">f</mi>
  </mrow>
  <mo stretchy="false">(</mo>
  <mi>z</mi>
  <mo stretchy="false">)</mo>
  <mo>=</mo>
  <mfrac>
    <mn>1</mn>
    <mrow>
      <mn>1</mn>
      <mo>+</mo>
      <msup>
        <mi>e</mi>
        <mrow class="MJX-TeXAtom-ORD">
          <mo>&#x2212;<!-- − --></mo>
          <mi>z</mi>
        </mrow>
      </msup>
    </mrow>
  </mfrac>
</math>
进行预测，其中<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi>z</mi>
  <mo>=</mo>
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
</math>。默认如果<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="normal">f</mi>
  </mrow>
  <mo stretchy="false">(</mo>
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mi>x</mi>
  <mo stretchy="false">)</mo>
  <mo>&gt;</mo>
  <mn>0.5</mn>
</math>则输出正的，否则输出负的。
####评估指标
MLlib支持常见的二分分类评估指标，包括精确，召回，F值，ROC，精密召回曲线和AUC，AUC在比较多个模型的性能是很常用的，精确/召回/F值可以帮助决定在预测中适当的下限值。  
下面的例子说明了如何加载数据集，在数据集上执行训练算法和使用结果模型进行预测：

	import org.apache.spark.SparkContext
	import org.apache.spark.mllib.classification.SVMWithSGD
	import org.apache.spark.mllib.evaluation.BinaryClassificationMetrics
	import org.apache.spark.mllib.regression.LabeledPoint
	import org.apache.spark.mllib.linalg.Vectors
	import org.apache.spark.mllib.util.MLUtils

	// Load training data in LIBSVM format.
	val data = MLUtils.loadLibSVMFile(sc, "data/mllib/sample_libsvm_data.txt")

	// Split data into training (60%) and test (40%).
	val splits = data.randomSplit(Array(0.6, 0.4), seed = 11L)
	val training = splits(0).cache()
	val test = splits(1)

	// Run training algorithm to build the model
	val numIterations = 100
	val model = SVMWithSGD.train(training, numIterations)

	// Clear the default threshold.
	model.clearThreshold()

	// Compute raw scores on the test set. 
	val scoreAndLabels = test.map { point =>
  		val score = model.predict(point.features)
  		(score, point.label)
	}

	// Get evaluation metrics.
	val metrics = new BinaryClassificationMetrics(scoreAndLabels)
	val auROC = metrics.areaUnderROC()

	println("Area under ROC = " + auROC)
SVMWithSGD.train()方法默认会使用正则参数处理L2正则化，如果需要配置该算法，可以创建新对象并调用set方法来自定义SVMWithSGD，所有的其他MLlib算法也支持这种方式来进行自定义。例如下面的代码产生一个L1正则化变体，它使用了正则参数1.0，并运行200次训练算法：

	import org.apache.spark.mllib.optimization.L1Updater

	val svmAlg = new SVMWithSGD()
	svmAlg.optimizer.
  		setNumIterations(200).
  		setRegParam(0.1).
  		setUpdater(new L1Updater)
	val modelL1 = svmAlg.run(training)
LogisticRegressionWithSGD的用法与SVMWithSGD类似。
###线性最小二乘法，套索，岭回归
线性最小二乘法在处理回归问题是最常见的算法，表达式为：
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>L</mi>
  <mo stretchy="false">(</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">w</mi>
  </mrow>
  <mo>;</mo>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo>,</mo>
  <mi>y</mi>
  <mo stretchy="false">)</mo>
  <mo>:=</mo>
  <mfrac>
    <mn>1</mn>
    <mn>2</mn>
  </mfrac>
  <mo stretchy="false">(</mo>
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <mrow class="MJX-TeXAtom-ORD">
    <mi mathvariant="bold">x</mi>
  </mrow>
  <mo>&#x2212;<!-- − --></mo>
  <mi>y</mi>
  <msup>
    <mo stretchy="false">)</mo>
    <mn>2</mn>
  </msup>
  <mo>.</mo>
</math>
很多相关的回归方法衍生于不同类型的正则化：普通最小二乘法或线性最小二乘法没有使用正则化；岭回归使用L2正则化；套索使用L1正则化。对于所有的这些模型来说，平均损失或训练错误：<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mfrac>
    <mn>1</mn>
    <mi>n</mi>
  </mfrac>
  <munderover>
    <mo>&#x2211;<!-- ∑ --></mo>
    <mrow class="MJX-TeXAtom-ORD">
      <mi>i</mi>
      <mo>=</mo>
      <mn>1</mn>
    </mrow>
    <mi>n</mi>
  </munderover>
  <mo stretchy="false">(</mo>
  <msup>
    <mrow class="MJX-TeXAtom-ORD">
      <mi mathvariant="bold">w</mi>
    </mrow>
    <mi>T</mi>
  </msup>
  <msub>
    <mi>x</mi>
    <mi>i</mi>
  </msub>
  <mo>&#x2212;<!-- − --></mo>
  <msub>
    <mi>y</mi>
    <mi>i</mi>
  </msub>
  <msup>
    <mo stretchy="false">)</mo>
    <mn>2</mn>
  </msup>
</math>被称为均方差。  
下面的例子解释了如果加载训练数据，并将数据解析为LabeledPoint的RDD。使用LinearRegressionWithSGD创建一个简单的线性模型来预测标签值，最后会计算均方差：

	import org.apache.spark.mllib.regression.LinearRegressionWithSGD
	import org.apache.spark.mllib.regression.LabeledPoint
	import org.apache.spark.mllib.linalg.Vectors

	// Load and parse the data
	val data = sc.textFile("data/mllib/ridge-data/lpsa.data")
	val parsedData = data.map { line =>
  		val parts = line.split(',')
  		LabeledPoint(parts(0).toDouble, Vectors.dense(parts(1).split(' 		').map(_.toDouble)))
	}

	// Building the model
	val numIterations = 100
	val model = LinearRegressionWithSGD.train(parsedData, numIterations)

	// Evaluate model on training examples and compute training error
	val valuesAndPreds = parsedData.map { point =>
  		val prediction = model.predict(point.features)
  		(point.label, prediction)
	}
	val MSE = valuesAndPreds.map{case(v, p) => math.pow((v - p), 2)}.mean()
	println("training Mean Squared Error = " + MSE)
RidgeRegressionWithSGD和LassoWithSGD的用法与LinearRegressionWithSGD类似。
###流式线性回归
当数据是以流的方式进来时，那么选择合适的回归模型是很有用的，MLlib目前支持使用普通最小二乘法实现流式回归。  
下面的例子解释了如果加载训练数据和从两个不同的输入流测试数据，将流解析为标签点，对第一个流使用线性回归模型，并预测第二个流。  
首先导入必要的类来解析输入数据并创建模型：

	import org.apache.spark.mllib.linalg.Vectors
	import org.apache.spark.mllib.regression.LabeledPoint
	import org.apache.spark.mllib.regression.StreamingLinearRegressionWithSGD
然后创建输入流来训练和测试数据，在这个例子中使用了标签点来训练和测试流，但是在实际中可能更倾向于使用未标记的向量来测试数据：

	val trainingData = ssc.textFileStream('/training/data/dir').map(LabeledPoint.parse)
	val testData = ssc.textFileStream('/testing/data/dir').map(LabeledPoint.parse)
接着将模型的权重初始化为0：

	val numFeatures = 3
	val model = new StreamingLinearRegressionWithSGD()
    	.setInitialWeights(Vectors.zeros(numFeatures))
最后将流进行注册来训练和测试并启动任务：

	model.trainOn(trainingData)
	model.predictOnValues(testData.map(lp => (lp.label, lp.features))).print()

	ssc.start()
	ssc.awaitTermination()
现在就可以看到文本文件保存到训练和测试目录中了，文件的每行形式为(y,[x1,x2,x3])，其中y表示标签，x1,x2,x3表示特征。如果有新的文件放入/training/data/dir中模型就会自动更新，如果有新的文件放入/testing/data/dir中就会看到预测结果，如果测试目录中的数据越多，那么预测就会更准确。
###决策树
决策树算法在机器学习中用到的地方很多，因为它容易学习并易于处理类别特征。MLlib支持使用连续和类别特征来处理二分和多类分类和回归，实现的方式是使用行来对数据进行分区，这样就允许多个实例来对数据进行分布式训练。
####基本算法
决策树是一种对特征空间进行二分分区的贪婪算法，该树会对每个最底的（叶子）分区预测相同的标签，每个分区会被从一系列可能的split中贪婪的选择最适合的split，这样可以使树节点的信息能够增长到最大。换句话说，在树节点选择出来的split会从集合<math xmlns="http://www.w3.org/1998/Math/MathML">
  <munder>
    <mi>argmax</mi>
    <mi>s</mi>
  </munder>
  <mi>I</mi>
  <mi>G</mi>
  <mo stretchy="false">(</mo>
  <mi>D</mi>
  <mo>,</mo>
  <mi>s</mi>
  <mo stretchy="false">)</mo>
</math>中选择出来，其中<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mi>I</mi>
  <mi>G</mi>
  <mo stretchy="false">(</mo>
  <mi>D</mi>
  <mo>,</mo>
  <mi>s</mi>
  <mo stretchy="false">)</mo>
</math>是信息增长当split<math xmlns="http://www.w3.org/1998/Math/MathML">
<mi>s</mi>
</math>被应用到数据集<math xmlns="http://www.w3.org/1998/Math/MathML">
<mi>D</mi>
</math>中。
####节点杂质和信息收益