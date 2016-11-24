---
layout: post
title: "Python import hook"
category: "python"
tags: [python]
---

之前写了个私有包叫ituandui，通过pip安装到了本地，然后我试图更改源码，写个单元测试，如下组织结构

```bash
[ituandui] tree 
.
├── __init__.py
├── sstr.py
├── test
│   ├── __init__.py
│   └── test_sstr.py
└── valid.py
```

在test_sstr.py中写单元测试时，`import ituandui` 其实时导入pip安装过的那个包，我想怎么才能改变这种机制呢？我想有如下办法：

1. pip remove 本地的ituandui包。
2. pip install 操作应该在python虚拟环境下运行，保持开发环境的独立性。
3. 更改包名。
4. 改成相对导入，`from ..sstr import *`, 然后 `python -m ituandui.test.test_sstr` 
5. import hook机制更改导入机制。

前4个都比较简单，我想可能是我之前的方式不对，**我现在深刻体会到了：开发项目时永远都要创建虚拟环境，避免污染。**

切入主题就是因为不了解Python import钩子，借此机会学习一下。

# 一. 作用域和命名空间

## 1.1 概念

命名空间和作用域很容易搞混，命名空间具体的概念如下：

>**命名空间表示标识符（identifier）的可见范围, 一个标识符可在多个命名空间中定义，它在不同命名空间中的含义是互不相干的**

这样，不同命名空间就避免了冲突。看上面的定义，就会发现与作用域很像啊，那么命名空间与作用域的区别是啥呢？

>**在编程语言中，命名空间是对作用域的一种特殊的抽象。**命名空间包含了处于该作用域内的标识符，且本身也用一个标识符来表示，这样便将一系列在逻辑上相关的标识符用一个标识符组织了起来。

可以认为：**命名空间也是一种作用域**

**在C++和Python等中，命名空间本身的标识符也属于一个外层的命名空间，也就是说命名空间可以嵌套，构成一个命名空间树，树根则是无名的全局命名空间。**


## 1.2 Python命名空间

**Python命名空间就像一个字典，k是变量名，v是变量值。**

我于今年1月份写过一篇博客总结了[**python 导入机制**](http://beginman.cn/2016/01/27/python-import-mechanism), 洋洋洒洒的一大篇，讽刺的是，过了快一年了，又开始研究Python的导入机制了，**学的东西没有印在脑子里，没有自己深刻的认识，无异于浪费时间精力重复而已。**

### 1.2.1 命名空间分类

**有三类命名空间：**

1. **内置**命名空间，built-in names（包括内置函数，内置常量，内置类型）
2. **全局**命名空间，如一个模块的global names（这个模块定义的函数，类，变量）
3. **局部**命名空间，如一个函数的所有local names；类对象的所有属性（数据成员，成员函数）等

其层次结构如下：

![](http://beginman.qiniudn.com/2016-11-24-14799711459759.jpg)

![](http://beginman.qiniudn.com/2016-11-24-14799684567225.jpg)

### 1.2.2 命名空间生命周期

1. 内置命名空间, Python解释器启动的时候被创建，解释器退出的时候才被删除
2. 全局命名空间, module被import的时候创建，在解释器退出的时候退出
3. 局部命名空间, 每次被调用的时候创建，返回或抛出异常时被删除

### 1.2.3 命名空间查找顺序

![](http://beginman.qiniudn.com/2016-11-24-14799737305176.jpg)

### 1.2.4 命名空间的访问

如下：

- `locals() `访问局部命名空间
- `globals()`访问全局命名空间

测试代码如下：

```python
In [2]: name = 'beginman'

In [3]: globals()
Out[3]:
{
 ......
 '__builtin__': <module '__builtin__' (built-in)>,
 '__builtins__': <module '__builtin__' (built-in)>,
 '__doc__': 'Automatically created module for IPython interactive environment',
 '__name__': '__main__',
 'name': 'beginman'
}

In [9]: def ts(t):
   ...:     """doc..."""
   ...:     a = 100
   ...:     func = lambda m: m * 100
   ...:     print locals()
   ...:
   ...:

In [10]: ts(1)
{'a': 100, 't': 1, 'func': <function <lambda> at 0x10e38fed8>}
```

从上可知，**内置命名也同样被包含在一个模块中，它被称作 `__builtin__`**

## 1.3 作用域

写了大多命名空间的东西了，也相当于变相地写作用域。刚才`globals()`和`locals()`函数告一段落，这里来看看`global`和`nonlocal`**语句**, 注意，可没有`locals`语句啊。

global用来声明当前模块的全局命名空间的变量，不管定义与否都会添加到全局空间中。

如下测试:

```python

def test():
    a = 1

    def get_local():
        a = 10

    def do_global():
        global a
        a = 100

    def change_global():
        # globals()可变, 但locals()只读
        globals()['a'] = 0

    def _local():
        del globals()['a']

    get_local()
    print a     # local a

    do_global()     # 设置 a 为全局变量, 且值为100
    print a         # 按照命名空间访问顺序,先从局部开始, 所有是1

    change_global() # 同上
    print a

test()
print a   

# out:
1
1
1
0
```

如果在test()执行体加上`_local()`执行函数，那么将会删除全局变量a， 此时在最后的`print a` 就会`NameError`。

`nonlocal`在Py3才有, 它在函数或其他作用域中使用外层(非全局)变量, 从声明处从里到外的namespace去搜寻这个变量，直到模块的全局域（不包括全局域），找到了则引用这个命名空间的这个名字和对象，若作赋值操作，则直接改变外层域中的这个名字的绑定。nonlocal语句声明的变量不会在当前scope的namespace字典中加入一个key-value对，如果在外层域中没有找到，则如下报错。

```python
>>>SyntaxError: no binding for nonlocal 'spam' found
```

如下测试：

```python
def test():
    a = 1

    def get_local():
        a = 10

    def do_global():
        global a
        a = 100
        
    def do_nonlocal():
        nonlocal a
        a = 1000

    get_local()
    print(a)     # local a

    do_nonlocal()
    print(a)        # 1000

    do_global()     # 设置 a 为全局变量, 且值为100
    print(a)         # 1000


test()
print(a)            # 100
# out:
1
1000
1000
100
```

# 二. 包、模块(package and module)

关于这块, 我之前总结的[**python 导入机制**](http://beginman.cn/2016/01/27/python-import-mechanism/#python) 又得搬出来了，就不BB了。

就是说说**模块对象**这个玩意.

## 2.1 模块对象

import 时，代码只加载和编译一次，后续的import语句将模块名称绑定到前一次导入所创建的模块对象上。保存在`sys.modules`中，以K-V形式，K是模块名，V是模块对象，既当前所加载的模块。

在sys源码中，modules定义如下：

```python
modules = {} # real value of type <type 'dict'> skipped
```

下面来个小程序实验下，比如test目录如下：

```bash
.
├── __init__.py
├── a.py
├── b.py
```

`__init__.py` 代码如下：

```python
import sys
from a import *
from b import *

for k, v in sys.modules.items():
    print k, v
```

输入如下：

```python
a <module 'a' from '/xx/test/a.pyc'>
b <module 'a' from '/xx/test/b.pyc'>

posix <module 'posix' (built-in)>
site <module 'site' from '/usr/xx/python/lib/site.pyc'>
...
```

可发现：

1. module对象
2. 有内置的module对象
3. 文件都是pyc, 编译过的字节码

## 2.2 import 和 from 的区别

还是上面的代码，不过`__init__.py`改成这样：

```python
from a import *
from b import *

print globals()
```

使用了`from .. import ..`的形式，来看下全局命名空间里的玩意儿。输出如下：

```python
{'test_b': <function test_b at 0x107800b90>, 'test_a': <function test_a at 0x107800b18>, '__builtins__': <module '__builtin__' (built-in)>, '__file__': 'xxx.py}
```

在改下`__init__.py`, 这次使用`import`方式:

```python
import a
import b

print globals()
```

输出如下：

```python
{{'a': <module 'a' from '/xx/test/a.pyc'>, 'b': <module 'b' from '/xx/test/b.pyc'>}
```

看，两个命名空间形式完全不一样。原因如下：

1. `from` 语句用于将模块具体定义加载到当前命名空间中，不会创建一个名称来引用模块命名空间，而是将模块定义的对象放在了当前的命名空间。使用 from module import xx，实际是从另一个模块(module)中将指定的函数和属性等导入到自己的名字空间，这样就可以直接访问它们却不需要引用它们所来源的模块。
2. 使用 `import module`，模块自身被导入，但是它保持着自已的名字空间，需要使用模块名来访问它的函数或属性：`module.xx`

## 2.3 关于import, from 变量赋值的问题

如2.2所说，现在思考一个问题，既然from将定义的变量放在当前命名空间了，那么如果更改了该变量，是否所有的引用都会更改呢？ 还是上面的例子，进行试验：

```python
# a.py
m = 10
def test_a():
    print m
    
# b.py
n = 10

def test_b():
    print n
    
# __init__.py
from a import m, test_a     # from module import  xxx
import b                    # import module

print m
test_a()

# 重新赋值
m = 100
print m
test_a()

print '---------\n'

print b.n
b.test_b()

b.n = 1000
print b.n
b.test_b()
```

先猜想一下输出吧, ... 5s后.

打印如下：

```bash
10
10
100
10

10
10
1000
1000
```

这里可发现from 导入的变量重新赋值后没有任何变化，而import形式的却发生变化了。

这样要说下Python的赋值操作了。

>Python变量的赋值不是一种存储操作，也就是说，上例中对m的赋值不会将新值存储在m中并覆盖旧值，而是将创建包含值为100的新对象，并用名称m来引用它，此时，m不再绑定到导入模块中的值（既a.py下的m的值）,而是绑定到其他对象上。

**赋值操作只是把名字和对象做一个绑定，也就是绑定或重绑定的作用（bind or rebind）**

那么b.n = 1000为什么生效了呢？因为module是个对象，那么但凡是对象，都有句点操作`.`, b.n是可变的，所以`b.n=1000`就是将值为1000的对象绑定到b.n上，还是原先的对象。那么test_b函数`print n` 这里的n就是b模块对象的n变量了，既b.n。

![](http://beginman.qiniudn.com/2016-11-24-14799804812034.jpg)




