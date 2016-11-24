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





