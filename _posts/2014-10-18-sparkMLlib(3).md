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
许多标准的机器学习方法可以表达为凸的优化问题，例如，找到凸函数<math xmlns="http://www.w3.org/1998/Math/MathML">
<mi>f</mi>
</math>的极小值取决于变向量<math xmlns="http://www.w3.org/1998/Math/MathML">
<mrow class="MJX-TeXAtom-ORD">
  <mi mathvariant="bold">w</mi>
</mrow>
</math>，该变向量包含<math xmlns="http://www.w3.org/1998/Math/MathML">
<mi>d</mi>
</math>个entry，可以将该问题转换为优化问题<math xmlns="http://www.w3.org/1998/Math/MathML">
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
</math>，其中目标函数为如下形式：
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
其中向量<math xmlns="http://www.w3.org/1998/Math/MathML">
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
</math>是相应的标签。