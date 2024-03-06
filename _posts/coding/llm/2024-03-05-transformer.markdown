---
layout: post
title: "【llm】transformer"
subtitle: "transformer架构细节"
date: 2024-03-05 16:15:14
author: "dimi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags: [coding,llm]
mathjax: true   #开启latex公式支持
---

transformer的提出始于google17年的论文 [Attention Is All You Need](https://arxiv.org/abs/1706.03762)。论文如果比较难啃的话也可以参考Umar Jamil的视频 [Attention is all you need (Transformer)](https://youtu.be/bCz4OMemCcA?si=UqbSFU18z_mqx908)。除了三哥口音挺难绷的讲的蛮细节的。

# transform解决的问题

transform 出现以前nlp领域常用的模型是rnn和lstm，有以下几个问题

1. rnn context window太小，无法关联过长的上下文

2. rnn for loop太慢，transform向量化可以一步完成并且可以并行

3. rnn gradient explode/vanish 问题

google 17年直接提出了transfomer，直接掀了rnn的桌子

# transform架构

原始论文中的架构如下，模型为encoder-decoder架构。主要可以划分为4个部分，**embedding、encoder、decoder、softmax output**

![transformer](/img/in-post/llm/transformer.png)
<small class="img-hint">transform模型</small>

以输入为 Your cat is a lovely cat作为输入为例，则输入序列长度为seq=6

## 1. embedding

虽然这一部分叫做embedding，但是实质上是有多步操作将输入seq转化为input matrix

### tokenizer

首先要将输入序列转化为tokens，也就是将文字映射到vocabulary中对应的id，方便处理

### embedding

词嵌入，将word映射到高维向量空间方便模型学习词语间的度量关系，在实际应用中不需要太关注embedding这里的模型，因为一般情况下微调和推理都需要采用和模型训练时相同的embedding模型，直接用huggingface从模型封装的方法即可。

回到例子，将seq tokens做embedding处理，假设embedding向量维度为512（原始论文中就是这个数），则input matrix为6*512 

### positional encoding

![positionalmatrix.png](/img/in-post/llm/positionalmatrix.png)
<small class="img-hint">拿别的图举个栗子</small>

词语在句子中的位置也会携带一些信息，因此需要度量词语之间的“距离“，让模型能够学习到这些信息，选择使用正余弦函数来表示

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

![trigonometricfunc.png](/img/in-post/llm/trigonometricfunc.png)
<small class="img-hint">为什么要用三角函数</small>

结合上面的三个步骤我们成功将输入的seq转化为了input matrix 6*512，**矩阵中的每行代表了一个token**

![embeddinginput](/img/in-post/llm/embeddinginput.jpg)

## 2. encoder

在完成输入seq的处理后，我们进入encoder的结构，在具体介绍encoder结构之前，先来聊一下注意力机制，这个机制是transformer的核心，从论文的命名就可以看出。

### 注意力机制

#### self attention

首先说一下自注意力机制，自注意力机制使模型能够学习到seq中每个单词和其他单词的关系，也就是seq所包含的6个tokens之间的内在联系。因此，叫做**自**注意力。

$$
Attention(Q,K,V)=softmax\Big(\frac{QK^T}{\sqrt{d_k}}\Big)V
$$

以上面的input matrix为例，则$seq=6, d_{model}=d_k=512$(为了简洁表示)，Q，K，V矩阵，即查询矩阵(query matrix)、键矩阵(key matrix)、值矩阵(value matrix)都可以看作为input matrix。

1. $QK^T$矩阵点乘结果可能过大。假设q，k是均值为0方差为1的随机变量，相互独立，则他们的点乘结果$p\cdot q=\sum_{i=1}^{d_k}p_iq_i$，均值为0，方差为$d_k$。

2. 为了防止矩阵点乘结果过大导致softmax的梯度过小，因此对其用$\sqrt{d_k}$进行scale处理。

3. softmax计算后得到6*6的矩阵，并且每行的和应该为1。每个数值代表每个token与其他token的score，比如(1,1)表示your与自身的score，(1,2)表示your与cat的score。正常情况下，矩阵主对角线上的score应该是最高的。

4. 最后乘以矩阵$V$，得到与原始矩阵相同大小的attention矩阵6*512。**矩阵中的每行不仅获取到了每个token的含义（embedding）和位置信息（position encoding），还获得了每个单词与其他单词的关系**，这里可以通过手工推到矩阵乘法看出来。

5. 由于矩阵运算的特性，交换token位置不会改变计算结果。

6. 如果你不想要某些token有交互，可以在使用softmax计算前，将矩阵中对应位置的值设为$-\infty$，这样模型就不会学习到这些关系了，**我们把这个做法称作为mask**，将在decoder中用到。

截止目前，在自注意力的计算中没有引入任何参数。后面计算多头注意力时，会引入参数矩阵，这才是模型要学习的目标。

#### multi-head attention

上面介绍了自注意力，也就是single-head attention，下面介绍一下multi-head attention。

![multiheadatte](/img/in-post/llm/multiheadatte.jpg)
<small class="img-hint">多头注意力机制</small>

$$
MultiHead(Q,K,V) = Concat(head_1...head_h)W^0\\
head_i = Attention(QW_i^Q,KW_i^K,VW_i^V)
$$

Here:

$seq$：seq长度

$d_{model}$: embedding向量的长度，这里是512

$h$: 多头数量，论文中的结构是4

$d_k=d_v$: $d_{model}/h$

1. input matrix (6 * 512)复制4份，按照图示，三份进入多头注意力层，作为$Q,K,V$矩阵，并随机初始化系数矩阵$W^Q,W^K,W^V$ (512 * 512)，得到结果$Q',K',V'$

2. 将三个矩阵$Q',K',V'$按照$d_{model}$的维度拆分为h个小矩阵，每个小矩阵为6 * $d_k$

3. 利用这些小矩阵和上面的attention公式计算$head_{i}$，并重新合并，乘以系数矩阵$W^0$，得到最终结果MultiHead 6 * 512

在这里，没有直接用$Q',K',V'$来计算多头注意力，而是**将其按照$d_{model}$维度拆分为h份分别计算。每份小矩阵都覆盖到了整个seq的6个token，这里是为了能够让模型从不同的角度学习token之间的关系。** 以中文为例，一些词在不同的语意下可以是名词也可以是形容词，llm本质上是一个概率模型，这里是希望模型能够从不同的角度学习这些关系并给出可能性最大的预测。上图中的图例显示了词 making 在不同的'head'中，分别学习到了和其他词的不同关系。

#### 为什么分拆为 query，keys，values?

为啥突然分拆为三个矩阵，并且命名为这三个名称？网上有解释说这个来自于数据库的术语或python的字典结构。

### encoder

有了上面对注意力机制的解释，我们可以详细的看下encoder的结构

## 3. decoder

## 4. softmax output
