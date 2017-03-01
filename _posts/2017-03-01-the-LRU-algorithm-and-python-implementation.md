---
layout: post
title: "LRU算法及Python实现"
description: "学习LRU算法"
category: "python"
tags: [algorithm, python]
---

面试的过程中被问到"最近最少使用算法"，然后就懵逼了，最后面试官说这就是LRU(Least Recently Used)我才恍然大悟，哦，听说过这个算法，但是..自己不会...。所以这篇文章主要就是以攻破LRU以及Python实现为主。

# 一.概述

这个算法出现在操作系统内存管理上，因为内存是有限且珍贵的，如何更大更高效利用内存呢？就出现了一种**虚拟内存**，就是在内存有限的情况下，扩展部分外存作为虚拟内存。虽然虚拟页式内存管理增大了进程所需的内存空间，但是不可避免的是内存外存的交换，由于外存低速，所消耗的时间也较长。于是人们想有没有一种算法**减少读取外存的次数**。

因为内外存信息替换是以页面为单位，当需要外存的页面是就得把它调到内存中，那么，把哪个页面调出去可以达到调动尽量少的目的？LRU算法就因此而生了，意思就是**使用频繁的xx在后面也可能会使用频繁，反之已经很久没有使用的则很可能以后也不会使用。**

# 二.基本原理

LRU算法其实就是按照**近期最少使用**的条件淘汰数据。既然是近期最少使用，那么肯定有*时间*这个筛选条件，也就是**淘汰截止目前缓存区中最久未被访问过的元素。**

```python
if data in cacheList:
    cacheList.get(data)
else:
    query data
    cacheList.append(data)
```

上面的操作就是不断的像内存中添加数据，久而久之就会内存爆满，如何淘汰旧数据呢，依照上面说的LRU算法，记录下每个元素最后一次被访问的时间戳，并按照此时间戳早晚进行排序，那么，距今最久未用的被淘汰掉，那么可实现一个简单版本就是：

```python
sorted_cache_list = sorted(cacheList, key=lambda i: i.vist_time)
saved_cache_list = sorted_cache_list[n:]
```

图示：

![](media/14883355132675.jpg)


这种操作的时间复杂度O(N), 且非常低效的。

>LRU算法不是绝对公平的，比如某个元素在淘汰前才被访问了一次，但是之前很长一段时间都未被访问，那么按照LRU算法它也会被保存下来，**它无法准确描述“是否常用”这一特性。**

下面是根据上面简单实现原理来实现的一个k-v cache.

LRU常用在缓存置换，实现一个K-V cache则需要下面基本的几个API和因素：

- capacity: cache容量，超出则使用LRU淘汰
- get(key): 获取缓存元素
- set(key, value): 添加缓存
- del(key): 删除缓存元素

一种比较简单的实现方法如下：

```python
In [151]: class LRUCache(object):
     ...:     def __init__(self, capacity):
     ...:         self.capacity = capacity
     ...:         self.tm = 0
     ...:         self.cache = {}    # cache data
     ...:         self.lru = {}      # track the access history with tm
     ...:
     ...:     def get(self, key):
     ...:         if key in self.cache:
     ...:             self.lru[key] = self.tm
     ...:             self.tm += 1   # automatic increment
     ...:             return self.cache[key]
     ...:
     ...:     def set(self, key, value):
     ...:         if len(self.cache) >= self.capacity:
     ...:             old_key = min(self.lru.keys(), key=lambda k: self.lru[k])
     ...:             self.cache.pop(old_key)
     ...:             self.lru.pop(old_key)
     ...:         self.cache[key] = value
     ...:         self.lru[key] = self.tm
     ...:         self.tm += 1
     ...:

In [152]: l = LRUCache(4)

In [153]: l.set('a', 1)

In [154]: l.set('b', 2)

In [155]: l.set('c', 3)

In [156]: l.get('b')
Out[156]: 2

In [157]: l.set('d', 4)

In [158]: l.set('e', 5)

In [159]: l.cache
Out[159]: {'b': 2, 'c': 3, 'd': 4, 'e': 5}

In [160]: l.get('c')
Out[160]: 3

In [161]: l.get('c')
Out[161]: 3

In [162]: l.get('c')
Out[162]: 3

In [163]: l.lru
Out[163]: {'b': 3, 'c': 8, 'd': 4, 'e': 5}

In [164]: l.set('f', 100)

In [165]: l.lru
Out[165]: {'c': 8, 'd': 4, 'e': 5, 'f': 9}

In [166]: l.cache
Out[166]: {'c': 3, 'd': 4, 'e': 5, 'f': 100}
```

上面的例子，存在瓶颈的地方是`min`操作，会处理整个cache，时间复杂度O(n), 如果这个dict cache有序的化我们就不用`min`操作了，直接根据时间戳已经排好序了，超出阈值直接删除cache旧数据。这里需要用到`collections`模块的`OrderedDict`类了。

**在Python中 dict由于Hash特性，是无序的，但是collections模块的OrderedDict则是有序的字典对象。**

那么上面的例子可以通过OrderDict改写。改写规则如下：

- 对于get操作，先pop然后再插入，既把它摘出来放在表头上
- 对于set操作，如果缓存在列表里，把它摘出来放在表头上，否则直接放在表头

```python
import collections

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = collections.OrderedDict()

    def get(self, key):
        try:
            value = self.cache.pop(key)
            self.cache[key] = value
            return value
        except KeyError:
            return -1

    def set(self, key, value):
        try:
            self.cache.pop(key)
        except KeyError:
            if len(self.cache) >= self.capacity:
                self.cache.popitem(last=False)
        self.cache[key] = value
```

这个算法比上面一个高效多了，主要利用OrderedDict的有序性，对于旧元素的删除，则通过popitem方法，该方法源码如下：

```python
class OrderedDict(dict):
    # .....
    
    def popitem(self, last=True):
        '''od.popitem() -> (k, v), return and remove a (key, value) pair.
        Pairs are returned in LIFO order if last is true or FIFO order if false.

        '''
        if not self:
            raise KeyError('dictionary is empty')
        key = next(reversed(self) if last else iter(self))
        value = self.pop(key)
        return key, value
```

`popitem`默认采用LIFO(后进先出)原则，如果把last参数改成false则变成FIFO(先进先出)。

**注意，OrderedDict的Key会按照插入的顺序排列，不是Key本身排序：**

```python
In [194]: o['e'] = 1

In [195]: o['a'] = 2

In [196]: o['c'] = 3

In [197]: o.keys()   # 按照插入key的顺序返回
Out[197]: ['e', 'a', 'c']
```
如下实现FIFO dict, 超出容量则删除最早插入的key:

```python
from collections import OrderedDict

class FIFOdict(OrderedDict):
     ...:     def __init__(self, capacity):
     ...:         super(FIFOdict, self).__init__()
     ...:         self.capacity = capacity
     ...:
     ...:     def __setitem__(self, k, v):
     ...:         if len(self) >= self.capacity:
     ...:             last = self.popitem(last=False)
     ...:             print "remove last: ", last
     ...:
     ...:         if k in self:
     ...:             del self[key]
     ...:         OrderedDict.__setitem__(self, k, v)

In [230]: f = FIFOdict(3)

In [231]: f['d'] = 1

In [232]: f['a'] = 2

In [233]: f['e'] = 3

In [234]: f['b'] = 4
remove last:  ('d', 1)

In [235]: f['a'] = 6
remove last:  ('a', 2)

In [236]: f
Out[236]: FIFOdict([('e', 3), ('b', 4), ('a', 6)])
```

如果使用循环链表则，元素被命中时，移到链表头。执行淘汰时，从链表尾开始淘汰。

# 三.应用

常应用于缓存中的数据淘汰。在Py 3.2版本，[functools模板添加了`lru_cache`装饰器](https://docs.python.org/3/library/functools.html#functools.lru_cache)。

```python
>>> from functools import lru_cache
>>> @lru_cache(maxsize=None)
... def fib(n):
...     if n < 2:
...             return n
...     return fib(n-1) + fib(n-2)
...
>>> [fib(n) for n in range(16)]
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610]
>>> fib.cache_info()
CacheInfo(hits=28, misses=16, maxsize=None, currsize=16)

>>> fib.cache_clear()
>>> fib.cache_info()
CacheInfo(hits=0, misses=0, maxsize=None, currsize=0)
```

还有一个 [PyLRU库 A least recently used (LRU) cache for Python.](https://github.com/jlhutch/pylru)也挺不错的。


# 参考

- [剖析LRU算法及LinkedHashMap源码实现机制](http://www.yufengof.com/2015/11/18/lru-and-linkedhashmap-source-code/)
- [LRU CACHE IN PYTHON](https://www.kunxi.org/blog/2014/05/lru-cache-in-python/)


