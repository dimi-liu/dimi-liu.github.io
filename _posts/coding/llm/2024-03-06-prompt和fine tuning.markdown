---
layout: post
title: "【llm】prompt和fine tuning"
subtitle: "模型的提升和微调"
date: 2024-03-05 16:15:14
author: "dimi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags: [coding,llm]
mathjax: true   #开启latex公式支持
---

本篇介绍下llm的prompt和fine tuning

# 一、prompt

使用各种网页端ui时，经常会有以下几个参数供用户调节，

- Max New Tokens
- Top k & Top p
- Temperature

这些参数会影响到在模型推理过程中的输出

## 1. Max New Tokens

有的ui上也叫做max_length，顾名思义，定义了推断输出的最长tokens。如果生成的token达到了参数值，会被强制stop

## 2. 模型输出模式 greedy vs. random sampling

llm模型实质是个概率模型，因此最终softmax层输出的是概率分布。在不同的输出模式下，模型可以选择不同的策略来决定token值。

![multiheadatte](/img/in-post/llm/prompt&finetuning/outputStrategy.jpg)
<small class="img-hint">inference输出模式</small>

- greedy：greedy模式下，模型会直接选择概率最大的单词/token输出
- random sampling：基于输出的概率分布随机取样输出下一个token，如上图，输出'cake'的概率是20%，但是实际输出的是bababa

## 3. Top k & Top p

基于上面random sampling的输出模式，引入top k和top p两个参数。

![multiheadatte](/img/in-post/llm/prompt&finetuning/topk.jpg)
<small class="img-hint">top k输出</small>

top k：从结果中截取top k个结果，并重新根据权重计算概率分布；如上图，top k = 3后，结果只会在前三个单词中产生。

![multiheadatte](/img/in-post/llm/prompt&finetuning/topp.jpg)
<small class="img-hint">top p输出</small>

top p：计算概率分布cdf，只会从cdf小于top p的单词中选择结果，舍弃掉小概率出现的token。如上图，top p = 0.3后，结果只会在前两个单词中产生。

## 4. Temperature

