---
layout: post
title: "backtrader keynote"
subtitle: "backtrader的一些使用记录"
date: 2023-08-14 20:21:17
author: "dimi"
header-img: "img/bg-material.jpg"
catalog: true
tags: [coding,quant,backtrader]
---

## self.p 
self.p 指的是 strategy 或 indicator 中的 params dict。

The `params` dictionary is a dictionary-like object that contains the parameters that were passed to the strategy or indicator when it was initialized. These parameters can be used to customize the behavior of the strategy or indicator.

```python
class MyStrategy(bt.Strategy):
    params = (
        ('my_param', 10),
        ('my_other_param', True),
    )

    def __init__(self):
        print(self.p.my_param)
        print(self.p.my_other_param)
```

In this example, `self.p.my_param` would evaluate to `10`, and `self.p.my_other_param` would evaluate to `True`.

You can also modify the values of the `params` dictionary at runtime, which can be useful for implementing dynamic behavior in your strategy or indicator. For example:

```python
class MyStrategy(bt.Strategy):
    params = (
        ('my_param', 10),
        ('my_other_param', True),
    )

    def __init__(self):
        self.p.my_param = 20
        print(self.p.my_param)
```


