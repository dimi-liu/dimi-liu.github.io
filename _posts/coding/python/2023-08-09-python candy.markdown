---
layout: post
title: "python candy"
subtitle: "some tricks here"
date: 2023-08-09 11:21:17
author: "dimi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags: [coding,python]
---

在这里记录一些python的tricky方法

## built-in
#### getattr
python build-in 方法，用来返回obj的name的方法

```python
getattr(object, name[, default])

```
- `object`: The object whose attribute you want to retrieve.
- `name`: A string that specifies the name of the attribute you want to retrieve.
- `default` (optional): The value to return if the attribute does not exist. If `default` is not specified and the attribute does not exist, a `AttributeError` will be raised.

```python
class MyClass:
    def __init__(self):
        self.x = 1
        self.y = 2

obj = MyClass()

# Get the value of the 'x' attribute
x = getattr(obj, 'x')
print(x)  # Output: 1

# Get the value of the 'y' attribute
y = getattr(obj, 'y')
print(y)  # Output: 2

# Try to get the value of a non-existent attribute
z = getattr(obj, 'z', 'default')
print(z)  # Output: 'default'

import numpy
getattr(numpy, 'abs')(-1) # Output: '1'
```

#### with方法
1. 紧跟with后面的语句被求值后，返回对象的 __enter__() 方法被调用，这个方法的返回值将被赋值给as后面的变量
2. 当with后面的代码块全部被执行完之后，将调用前面返回对象的 __exit__() 方法

eg. 用来屏蔽方法内print
```python
class hiddenPrints:
    def __enter__(self):
        self._original_stdout = sys.stdout
        sys.stdout = open(os.devnull, 'w')
    def __exit__(self, exc_type, exc_val, exc_tb):
        sys.stdout.close()
        sys.stdout = self._original_stdout

# 调用时
with hiddenPrints():
    pass
```

## numpy相关
#### np.ma 掩码数组
```python
x = np.array([1, 2, 3, -1, 5])
mx = np.ma.masked_array(x, mask=[0, 0, 0, 1, 0])
mx.tolist() # [1, 2, 3, None, 5]
mx1 = np.ma.masked_array(x, mask>1)
mx1.tolist() # [1, None, None, -1, None]
```

## pandas相关
#### pd.read_sql/pd.read_sql_query/pd.read_sql_table
pd.read_sql + sqlalchemy 执行sql报错  
ObjectNotExecutableError: Not an executable object: ''  
sqlalchemy 2.0 有改动，不能直接传入字符串，需要用 sqlalchemy.text() 方法，详见[stackOverflow原贴](https://stackoverflow.com/questions/75284194/pandasql-sqldf-objectnotexecutableerror-not-an-executable-object-select)
```python
pd.read_sql(sql=text("sql"), con=conn)
```



