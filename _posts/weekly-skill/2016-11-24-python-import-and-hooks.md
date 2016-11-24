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

# 一. 作用域和命名空间（Scope and Namespace）

## 1.1 概念

命名空间和作用域很容易搞混，命名空间具体的概念如下：

>**命名空间表示标识符（identifier）的可见范围, 一个标识符可在多个命名空间中定义，它在不同命名空间中的含义是互不相干的**

这样，不同命名空间就避免了冲突。看上面的定义，就会发现与作用域很像啊，那么命名空间与作用域的区别是啥呢？

>**在编程语言中，命名空间是对作用域的一种特殊的抽象。**命名空间包含了处于该作用域内的标识符，且本身也用一个标识符来表示，这样便将一系列在逻辑上相关的标识符用一个标识符组织了起来。

可以认为：**命名空间也是一种作用域**

**在C++和Python等中，命名空间本身的标识符也属于一个外层的命名空间，也就是说命名空间可以嵌套，构成一个命名空间树，树根则是无名的全局命名空间。**





