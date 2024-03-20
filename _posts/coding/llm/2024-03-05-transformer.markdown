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

transformer的提出始于google17年的论文 [Attention Is All You Need](https://arxiv.org/abs/1706.03762)。论文如果比较难啃的话也可以参考Umar Jamil的视频 [Attention is all you need (Transformer)](https://youtu.be/bCz4OMemCcA?si=UqbSFU18z_mqx908)。除了三哥口音挺难绷的讲的蛮细节的。还可以参考这篇[文章](https://zhuanlan.zhihu.com/p/338817680)

# transformer解决的问题

transformer 出现以前nlp领域常用的模型是rnn和lstm，有以下几个问题

1. rnn context window太小，无法关联过长的上下文
2. rnn for loop太慢，transform向量化可以一步完成并且可以并行
3. rnn gradient explode/vanish 问题

google 17年直接提出了transfomer，直接掀了rnn的桌子

# 一些机制

首先介绍下模型中用到的一些思想

## 1. 注意力机制

### self attention

首先说一下自注意力机制，下面以输入序列为6个单词为例，则input matrix为6*512，其中512为embedding向量维度。自注意力机制使模型能够学习到seq中每个单词和其他单词的关系，也就是seq所包含的6个tokens之间的内在联系，因此，叫做**自注意力**，计算公式如下：
$$
Attention(Q,K,V)=softmax\Big(\frac{QK^T}{\sqrt{d_k}}\Big)V
$$
其中$seq=6, d_{model}=d_k=512$(为了简洁表示令两个相等)，$Q，K，V$矩阵，即查询矩阵(query matrix)、键矩阵(key matrix)、值矩阵(value matrix)在这里都可以看作为input matrix。

1. $QK^T$矩阵点乘结果可能过大：假设$q，k$是均值为0方差为1的随机变量，相互独立，则他们的点乘结果$p\cdot q=\sum_{i=1}^{d_k}p_iq_i$，均值为0，方差为$d_k$。为了防止矩阵点乘结果中某个值过大导致softmax的梯度过小，因此对其用$\sqrt{d_k}$进行scale处理。
2. softmax计算后得到6*6的矩阵，并且每行的和应该为1。每个数值代表每个token与其他token的score，比如(1,1)表示第一个token与自身的score，(1,2)表示第一个token和第二个token的score。正常情况下，矩阵主对角线上的score应该是最高的。
3. 最后乘以矩阵V，得到与原始矩阵相同大小的attention矩阵6*512。**矩阵中的每行不仅获取到了每个token的含义（embedding）和位置信息（position encoding），还获得了每个单词与其他单词的关系**，这里可以通过手工推导矩阵乘法看出来。
4. 由于矩阵运算的特性，交换token位置不会改变计算结果。
5. 如果你不想要某些token有交互，可以在使用softmax计算前，将矩阵中对应位置的值设为$-\infty$，这样softmax结果为0，模型就不会学习到这些关系了，**我们把这个做法称作为mask**，将在decoder中用到。

截止目前，在自注意力的计算中没有引入任何参数。后面计算多头注意力时，会引入参数矩阵，这才是模型要学习的目标。

### multi-head attention

上面介绍了自注意力，也就是single-head attention，下面介绍一下multi-head attention，即多头注意力。

![multiheadatte](/img/in-post/llm/multiheadatte.jpg)
<small class="img-hint">多头注意力</small>

$$
MultiHead(Q,K,V) = Concat(head_1...head_h)W^0\\
head_i = Attention(QW_i^Q,KW_i^K,VW_i^V)
$$

Here:  
$seq$：seq长度  
$d_{model}$: embedding向量的长度，这里是512  
$h$: 多头数量，论文中的结构是8  
$d_k=d_v$: $d_{model}/h$ = 64

1. input matrix (6 * 512)复制为两份份，按照图示，一份直接进入残差连接层用于与最后注意力计算结果相加，另一份进入多头注意力层，复制后作为$Q,K,V$矩阵，并随机初始化系数矩阵$W^Q,W^K,W^V$ (512 * 512)，得到结果$Q',K',V'$
2. 将三个矩阵$Q',K',V'$按照$d_{model}$的维度拆分为h个小矩阵，每个小矩阵为$6 * d_k$
3. 利用这些小矩阵和上面的attention公式计算$head_{i}$，并重新合并，乘以系数矩阵$W^0$，得到最终结果MultiHead 6 * 512

在这里，没有直接用Q',K',V'来计算多头注意力，而是**将其按照$d_{model}$维度拆分为h份分别计算。每份小矩阵都覆盖到了整个seq的6个token，这里是为了能够让模型从不同的角度学习token之间的关系。** 以中文为例，一些词在不同的语意下可以是名词也可以是形容词，llm本质上是一个概率模型，从**不同的角度学习**这些关系并给出可能性最大的预测。上图中的图例显示了词 making 在不同的'head'中，分别学习到了和其他词的不同关系。

### 为什么分拆为 query，keys，values?

为啥突然分拆为三个矩阵，并且命名为这三个名称？论文中有解释说这个来自于数据库的术语或python的字典结构。根据query，通过key去查询value值，从而学习到seq中每个词与其他词的关系。

## 2. residule block 残差模块

transformer的结构中多次出现add&norm的结构，这里add是指残差连接层，而norm是指layer norm。 
add层为**residule block**，数据在这里进行**residule connection（残差连接）**，那么为什么要引入这一层？

在进行**深层**网络学习的过程中，有两个避不开的问题：

- 梯度消失/爆炸
- 网络退化

![shallow&deepNet](/img/in-post/llm/shallow&deepNet.png)
<small class="img-hint">训练集上浅层网络的表现依然优于深层,显然此时与过拟合无关</small>

一方面，当网络足够深时，很容易出现梯度消失的问题，这个问题可以通过Normalization等方式解决，使得模型最终能够收敛。  
另一方面，因为神经网络帮我们避免了繁重的特征工程过程，借助神经网络中的非线形操作，可以帮助我们更好地拟合模型的特征。为了增加模型的表达能力，一种直觉的想法是，**增加网络的深度，一来使得网络的每一层都尽量学到不同的模式，二来更好地利用网络的非线性拟合能力。** 因此，理想中的深网络，其表现不应该差于浅网络。但是实时上是**深层网络的表现不一定比浅层网络表现更好**。这被称为**退化问题（degradation problem）**，原因是随着网络越来越深，训练变得原来越难，网络的优化变得越来越难。理论上，越深的网络，效果应该更好；但实际上，由于训练难度，过深的网络会产生退化问题，效果反而不如相对较浅的网络。

2015年的[ResNet](https://arxiv.org/pdf/1512.03385.pdf)提出了残差连接的结构，这个用于解决深层网络训练问题的模型最早被用于图像任务处理上，现在已经成为一种普适性的深度学习方法。主要是为了解决**梯度消失**和**权重矩阵的退化**两个问题。  

![residuleblock](/img/in-post/llm/residuleblock.png)
<small class="img-hint">残差网络</small>

$X$表示输入，$F(X)$表示两层函数的输出，强行将一个输入添加到函数的输出的时候，虽然我们仍然可以用$G(X)$来描述输入输出的关系，但是这个$G(X)$却可以明确的拆分为$F(X)$和X的线性叠加。  
[论文](https://arxiv.org/pdf/1701.09175.pdf)认为神经网络的退化才是难以训练深层网络根本原因所在，而不是梯度消散。虽然梯度范数大，但是如果网络的可用自由度对这些范数的贡献非常不均衡，也就是每个层中只有少量的隐藏单元对不同的输入改变它们的激活值，而大部分隐藏单元对不同的输入都是相同的反应，此时整个权重矩阵的秩不高。并且随着网络层数的增加，连乘后使得整个秩变的更低。  
这也是我们常说的网络退化问题，虽然是一个很高维的矩阵，但是大部分维度却没有信息，表达能力没有看起来那么强大。

## 3. layer norm

transformer结构中的norm为batch norm（注意与batch norm的差别）。通过对层的激活值的归一化，可以加速模型的训练过程，使其更快的收敛，具体细节可参考这篇论文[Layer Normalization](https://arxiv.org/abs/1607.06450)。这里我们简单的来了解一下layer normalization（LN）和batch normalization（BN）：

- batchNorm是在batch上进行处理，对每个batch的数据进行归一化，对小batchsize效果不好；即对同一个batch下不同样本的同一个特征做归一化
- layerNorm在通道方向上做归一化，主要对RNN作用明显；即对同一个样本的不同特征做归一化（针对所有样本）

在BN和LN都能使用的场景中，BN的效果一般优于LN，原因是基于不同数据，同一特征得到的归一化特征更不容易损失信息。  
LN对所有的特征进行缩放，这显得很没道理。比如我们计算出【身高、体重、年龄】三个特征的均值方差并对其进行缩放，事实上会因为特征的量纲不同而产生很大的影响。但是BN则没有这个影响，因为BN是对一列进行缩放，一列的量纲单位都是相同的。  
那么我们为什么还要使用LN呢？因为NLP领域中，LN更为合适。如果我们将一批文本组成一个batch，那么BN的操作方向是，对每句话的第一个词进行操作。但语言文本的复杂性是很高的，任何一个词都有可能放在初始位置，且词序可能并不影响我们对句子的理解。而BN是针对每个位置进行缩放，这不符合NLP的规律，而LN则是针对一句话进行缩放的，且LN一般用在第三维度，如[batchsize, seq_len, dims]中的dims，一般为**词向量**的维度，这一维度**各个特征的量纲应该相同**。因此也不会遇到上面因为特征的量纲不同而导致的缩放问题。

回到transformer结构中，Add & Norm层中的计算公式如下
$$
LayerNorm(X+MultiHeadAttention(X)) \\
LayerNorm(X+FeedForward(X))
$$

# transformer架构

完成了主要机制的介绍后，下面具体看下transformer的细节。原始论文中的架构如下，模型为encoder-decoder架构。主要可以划分为4个部分，**embedding、encoder、decoder、softmax output**

![transformer](/img/in-post/llm/transformer.png)
<small class="img-hint">transform模型</small>

以输入为 Your cat is a lovely cat作为输入为例，则输入序列长度为seq=6

## 一. embedding

虽然这一部分叫做embedding，但是实质上是有多步操作将输入seq转化为input matrix

### 1. tokenizer

首先要将输入序列转化为tokens，也就是将文字映射到vocabulary中对应的id，方便处理；一般和embedding放在一起处理

### 2. embedding

词嵌入，将word映射到高维向量空间方便模型学习词语间的度量关系，在实际应用中不需要太关注embedding这里的模型，因为一般情况下微调和推理都需要采用和模型训练时相同的embedding模型，直接用huggingface从模型封装的方法即可。  
回到例子，将seq tokens做embedding处理，假设embedding向量维度为512（原始论文中就是这个数），则input matrix为6*512 

### 3. positional encoding

![positionalmatrix.png](/img/in-post/llm/positionalmatrix.png)
<small class="img-hint">拿别的图举个栗子</small>

词语在句子中的位置也会携带一些信息，因此需要度量词语之间的“距离“，让模型能够学习到这些信息，选择使用正余弦函数来表示

$$
P(k, 2i)=\sin\Big(\frac{k}{n^{2i/d}}\Big)\\
P(k, 2i+1)=\cos\Big(\frac{k}{n^{2i/d}}\Big)
$$

Here:  
$L$：seq长度  
$k$：token在seq中的位置, $0 \leq k < L$  
$d$：embedding 空间维度  
$P(k, j)$：位置函数，将input seq中位置k的映射到positional matrix中的$(k,j)$  
$n$：用户自定义的标量，在原始论文中被设为10000  
$i$：$0 \leq i < d/2$

通过以上公式，得到positional matrix，与input matrix相加，得到最终的input matrix

![trigonometricfunc.png](/img/in-post/llm/trigonometricfunc.png)
<small class="img-hint">为什么要用三角函数</small>

结合上面的三个步骤我们成功将输入的seq转化为了input matrix 6*512，**矩阵中的每行代表了一个token**

![embeddinginput](/img/in-post/llm/embeddinginput.jpg)

## 二. encoder

在完成输入seq的处理后，我们进入encoder的结构，在encoder中，可以看到是由Multi-Head Attention, Add & Norm, Feed Forward, Add & Norm组成的。上面已经了解了Multi-Head Attention、Add & Norm的计算过程，现在了解一下 Feed Forward 部分。  
Feed Forward 层比较简单，是一个两层的全连接层，第一层的激活函数为 Relu，第二层不使用激活函数，对应的公式如下：

$$
FFN(x) = max(0, xW_1 + b_1 )W_2 + b_2
$$

X是输入，Feed Forward 最终得到的输出矩阵的维度与X一致。

通过上面描述的 Multi-Head Attention, Feed Forward, Add & Norm 就可以构造出一个 Encoder block，Encoder block 接收输入矩阵$X_{(n \times d)}$，并输出一个矩阵$O_{(n \times d)}$。通过多个 Encoder block 叠加就可以组成 Encoder。  
第一个 Encoder block 的输入为句子单词的表示向量矩阵，后续 Encoder block 的输入是前一个 Encoder block 的输出，最后一个 Encoder block 输出的矩阵就是编码信息矩阵 C，这一矩阵后续会用到 Decoder 中作为输入。

## 三. decoder

decoder架构整体与encoder类似，但是存在一些区别：

- 包含两个 Multi-Head Attention 层
- 第一个 Multi-Head Attention 层采用了 Masked 操作
- 第二个 Multi-Head Attention 层的K, V矩阵使用 Encoder 输出的编码信息矩阵C进行计算，而Q矩阵使用上一个 Decoder block 的输出计算

### 1. Masked Multi-Head Attention

Masked指的是屏蔽了矩阵主对角线以上元素的操作，我们的模型是生成模型，即需要根据第 i 个单词，才可以生成第 i+1 个单词。通过 Masked 操作可以防止第 i 个单词知道 i+1 个单词之后的信息。  
类似上面的attention计算，不同的是在计算得到$QK^T$矩阵之后，softmax处理之前，需要做mask处理，屏蔽主对角线上侧的信息

![mask.png](/img/in-post/llm/mask.png)
<small class="img-hint">mask操作</small>

后面的计算与多头注意力计算一致

### 2. 第二个 Multi-Head Attention

第二个 Multi-Head Attention 变化不大，主要的区别在于其中 Self-Attention 的 $K, V$矩阵不是使用 上一个 Decoder block 的输出计算的，而是使用 Encoder 输出的编码信息矩阵 $C$ 计算的。根据 Encoder 的输出 $C$ 得到 $K, V$，根据上一个 Decoder block 的输出 $Z$ 得到 $Q$ ，如果是第一个 Decoder block 则使用输入矩阵 $X$ （这里是被mask处理过的矩阵）进行计算，后续的计算方法与之前描述的一致。

## 四. softmax output

模型的最后一部分是利用Softmax预测下一个单词，在之前的网络层我们可以得到一个最终的输出 Z，因为 Mask 的存在，使得单词 0 的输出 $Z_0$ 只包含单词 0 的信息，Softmax 根据输出矩阵的每一行预测下一个单词。

