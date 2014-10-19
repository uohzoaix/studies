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