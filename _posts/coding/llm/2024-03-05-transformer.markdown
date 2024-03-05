---
layout: post
title: "transformer"
subtitle: "llm"
date: 2024-03-05 16:15:14
author: "dimi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags: [coding,llm]
---

### transformer

#### transform解决的问题

1. rnn context window太小，无法关联过长的上下文

2. rnn for loop太慢，transform向量化可以一步完成并且可以并行

3. rnn gradient explode/vanish 问题

#### transform架构

原始论文中的架构如下，模型为encoder-decoder架构。

主要可以划分为4个部分，**embedding、encoder、decoder、softmax output**

****![transformer.png](./transformer.png) 

##### 1. embedding

###### tokenizer

将文字映射到vocabulary中对应的id，方便处理；词嵌入，将word映射到高维向量空间方便模型学习度量，很古早的方法了，推理阶段直接用huggingface从模型封装的方法就行

###### embedding

经过处理后，将输入的sequence映射为matrix，假设embedding向量维度为512，输入sequence 'Your cat is a lovely cat' 为6个词，则input matrix为6*512

###### positional encoding

用于获取input sequence在全文中的位置信息，为了保证scale的值不过于离谱，使用正余弦函数来表示

![positionalmatrix.png](./positionalmatrix.png)

$$
P(k, 2i)=\sin\Big(\frac{k}{n^{2i/d}}\Big)\\
P(k, 2i+1)=\cos\Big(\frac{k}{n^{2i/d}}\Big)
$$

Here:

$L$：seq长度

$k$: token在seq中的位置, $0 \leq k < L$

$d$: embedding 空间维度

$P(k, j)$: 位置函数，将input seq中位置k的映射到positional matrix中的$(k,j)$

$n$: 用户自定义的标量，在原始论文中被设为10000

$i$:  $0 \leq i < d/2$

通过以上公式，得到positional matrix，与input matrix相加，得到最终的input matrix

##### 2. encoder

###### self attention

$$
Attention(Q,K,V)=softmax\Big(\frac{QK^T}{\sqrt{d_k}}\Big)V
$$

self-attention：允许模型获取输入序列中词语之间的关系，所以叫做自注意力

以上面的input matrix 6*512为例，则$seq=6, d_model=d_k=512$，Q，K，V分别为查询矩阵(query matrix)、键矩阵(key matrix)、值矩阵(value matrix)

###### multi-head attention

$$
\begin{align}
MultiHead(Q,K,V) &= Concat(Head_1...Head_h)W^0\\
head_i &= Attention(QW_i^Q,KW_i^K,VW_i^V)
\end{align}
$$

##### 3. decoder

##### 4. softmax output
