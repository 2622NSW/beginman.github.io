---
layout: post
title: "wks-02:Vim配置文件语法学习"
category: "一周新技术"
tags: vim
---

上一周新技术[wks-01:Vagrant学习](http://beginman.cn/2016/11/13/vagrant/) 对Vagrant有所了解，这周来学习vim配置文件语法。

这篇博客的目标：

1. 知道Vim的基本术语，如"buffer"、"window"、 "normal mode"、"insert mode"、"text mode"。
2. 搞懂vim配置文件
3. 搞懂其语法
4. 写自己的一套简单的配置

[这是我的vim配置文件](https://gist.github.com/BeginMan/baac945fc21df885e19e5cb16ad99c45), 效果如下：

![](http://beginman.qiniudn.com/2016-11-21-14797259160690.jpg)

当然这都是从别人的博客里学来的，虽然一直用vim但是都是小打小闹，这一次希望有所突破，那就从配置文件入手吧。


# 一.Vimrc文件

`~/.vimrc`，每次启动vim时，vim都会自动执行其代码。在VIM中执行`:echo $MYVIMRC`可查看路径。

# 二.打印命令

`echo`和`echom`命令，在vim中可执行`:help command`来查看命令的帮助手册。

如果要打印信息，输入`:echo "字符串"`,或`:echom '字符串'`, 字符串单双引号都行，就在屏幕底部打印出来。它们的用法一样，差别就是：

>`:echo`命令 会打印输出，但是一旦你的脚本运行完毕，那些输出信息就会消失。使用`:echom`打印的信息 会保存下来，你可以执行`:messages`命令再次查看那些信息。

如果在vimrc文件中添加一行代码，则每次打开vim就会打印你所编写的内容，如下：

```bash
# vim ~/.vimrc
echo ">^.^<, 加油Beginman!"

# 打开 vim
$ vim
$ >^.^<, 加油Beginman!
```

注释：`"`

# 三.设置选项

主要有两种选项：

1. 布尔选项（值为"on"或"off"）
2. 键值选项

执行方式：`:set 选项名`

比如，显示和隐藏行号

```bash
:set number     # 显示
:set nonumber   # 隐藏
```

**VIM中有很多`no`前缀表示反义。**

number是一个布尔选项：可以off、可以on

所有的布尔选项都是这种配置方法。

- `:set <name>`打开选项
- `:set no<name>`关闭选项。
- `:set <name>!` 来回切换
- `:set <name>?` 查看选项当前值
- `:set <name>=<value>` 键值选项, 有些选项并不只有off或on两种状态，它们需要一个值, 比如`:set numberwidth=10` 改变行号的列宽。

同理，这些`<name>` 都是命令，可`:help <name>`来查看帮助手册。

# 四.基本映射

这个必须要掌握，很牛逼。

先尝试一个小例子，在normal模式下， `:map - x`, 然后将光标置于文本某处，键入`-`, 可发现vim删除了当前光标下的字符，就如同你按下`x`一样。再搞一个`:map - dd` 则删除一行。

**映射的格式**：

    映射命令  按键组合 命令组合
    maptype  key     action

**maptype表示映射的类型**，分为两大类，带nore的和不带nore的, 具体类型通过`:help map-overview`可以查到。如：`nmap c ^i#<Esc>j`, 意思就是映射normal模式下的按键c为：`^i#<Esc>j`，就是在该行开头加上`#`号，然后下移一行。

常用的映射类型如下：

- map: 在所有模式下可用的映射
- vmap：在visual和select模式下可用的映射
- nmap：在normal模式下可用的映射
- imap：在insert模式下可用的映射
- omap：用于motion的一部分的映射。比如vw就是visual模式下选中一个词，可以用omap定义类似于w这样的动作操作符。
- cmap：用于在命令行下（输入:或/之类后）可用的映射

**key表示映射的键**: 各种特殊符号具体的表示方式见`:help key-notation`

**action就是映射出来的动作**。可以是一串字符串，或者调用一个函数，还可以是调用一个vim命令

比如在编辑模式下，将ESC键映射为jk: `inoremap jk <ESC>`

对**特殊字符**的处理，使用`<keyname>`告诉Vim一个特殊的按键。

```bash
:map <space> viw
```

你也可以映射修饰键入Ctrl和Alt。执行：

```bash
:map <c-d> dd
```

Ctrl+d将执行dd命令。

## 4.1 模式映射

## 4.2 精确映射

## 4.3 Leaders

## 4.4 在Vimrc中处理映射

## 4.5 Abbreviations

## 4.6 更多的Mappings

## 4.7 锻炼你的手指


# 参考

- [《笨方法学Vimscript》](http://learnvimscriptthehardway.onefloweroneworld.com/)
- [vim 配置文件语法](http://blog.csdn.net/trochiluses/article/details/21776365)

（补坑中...）



