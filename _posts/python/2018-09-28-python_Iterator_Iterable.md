---
layout: post
title: "Python Iterator TypeError: object of type 'listiterator' has no len()"
keywords: ["python"]
description: "python"
category: "python"
tags: ["python","python"]
---

在计算的len的时候碰到

```
TypeError: object of type 'listiterator' has no len()
```

说明这个对象是个iterator，是没有len的，翻了下google

普遍的做法是使用len(list(iterator))

```
>>> l1 = [1, 2, 3]
>>> it = iter(l1)
>>> it
<listiterator object at 0x1013089d0>
>>> type(it)
<type 'listiterator'>
>>> l2 = list(it)
>>> l2
[1, 2, 3]
>>> list(it)
[]
>>> l2
[1, 2, 3]
>>> l1
[1, 2, 3]
>>> 
>>> l1 = [1, 2, 3]
>>> it = iter(l1)
>>> len(list(it))
3
>>> list(it)
[]
```

### 注意

这个terator：list(it)是不能再用的，也就是重复遍历的话只有第一次有数据的

```
>>> l1 = [1, 2, 3]
>>> it = iter(l1)
>>> for i in it:
...     print i
... 
1
2
3
>>> for i in it:
...     print i
... 
>>> for i in l1:
...     print i
... 
1
2
3
>>> for i in l1:
...     print i
... 
1
2
3
>>>
```

