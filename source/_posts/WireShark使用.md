title: WireShark 简单使用
date: 2017/9/19 10:07:12  
categories: 计算机
tags:
	- 学习笔记
---

WireShark 用的不是很多，但是每次用的时候总是忘记怎么用。所以决定详细记录一下，看了 《WireShark数据包分析实战》

<!--more-->

##玩转捕获数据包

### 分析数据包

#### 查找数据包

使用 Ctrl-F 打开查找面板。注意，这是查找，不是筛选。所以该显示多少条还是显示多少条，只是可以快速跳转了。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/WireShark_2.png?raw=true)

提供了四种搜索选项，这里列举三个。还有一个关于显示控制器的，后面会详细说到：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/WireShark_1.png?raw=true)

#### 标记数据包

使用 Ctrl-M 可以标记某个数据包。它就会以黑底白字的方式显示出来。

### 设定时间显示格式和相对参考

这两个在主菜单里就有，时间显示格式在视图菜单中，相对时间参考在编辑菜单中。

### 使用过滤器

WireShark 提供了两种过滤器：

- 捕获过滤器：只有那些满足给定的包含/排除表达式的数据包会被捕获
- 显示过滤器：将捕获的过滤器按照表达式展示或者隐藏数据包

#### 捕获过滤器





