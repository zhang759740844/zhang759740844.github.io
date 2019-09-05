title: python 3.5 高级特性
date: 2016/8/8 10:07:12  
categories: python
tags:
	- 学习笔记

---

当时学的是2.x版本，有点过时了，现在看了[廖雪峰的博客](http://www.liaoxuefeng.com/)的3.5版本的语法介绍，做一些摘录。

<!--more-->




### 切片
切片（Slice）操作符用来简化经常取指定索引范围的操作
```python
>>> L = ['Michael', 'Sarah', 'Tracy', 'Bob', 'Jack']
>>> L[0:3]
['Michael', 'Sarah', 'Tracy']
```
从索引0开始取，直到索引3为止，但不包括索引3,正好是3个元素.

各种用法示例：
```python
# Python支持L[-1]取倒数第一个元素,同样支持倒数切片
>>> L[-2:]
['Bob', 'Jack']
>>> L[-2:-1]
['Bob']

# 前10个数，每两个取一个：
>>> L[:10:2]
[0, 2, 4, 6, 8]

# 所有数，每5个取一个：
>>> L[::5]
[0, 5, 10, 15, 20, 25, 30, 35, 40, 45, 50, 55, 60, 65, 70, 75, 80, 85, 90, 95]

# 什么都不写，只写[:]就可以原样复制一个list
>>> L[:]
[0, 1, 2, 3, ..., 99]

# tuple也是一种list，唯一区别是tuple不可变。因此，tuple也可以用切片操作，只是操作的结果仍是tuple
>>> (0, 1, 2, 3, 4, 5)[:3]
(0, 1, 2)

#字符串'xxx'也可以看成是一种list，每个元素就是一个字符。因此，字符串也可以用切片操作，只是操作结果仍是字符串：
>>> 'ABCDEFG'[:3]
'ABC'
>>> 'ABCDEFG'[::2]
'ACEG'
```

### 迭代
我们可以通过for循环来遍历这个list或tuple，这种遍历我们称为迭代（Iteration）。只要是可迭代对象，无论有无下标，都可以迭代。

判断一个对象是否是可迭代对象：
```python
from collections import Iterable
isinstance('abc', Iterable)
```

### 列表生成式
感觉没啥用。

### 生成器
列表元素可以按照某种算法在不断循环的过程中推算出后续元素，不必创建完整的list，节省大量空间。这种一边循环一边计算的机制，叫做generator。
例如实现斐波那契函数：
```python
def fib(max):
    n, a, b = 0, 0, 1
    while n < max:
        yield b
        a, b = b, a + b
        n = n + 1
    return 'done'
    
for n in fib(6):
	print(n)
```
如果一个函数定义中包含yield关键字，那么这个函数就不再是一个普通函数，而是一个generator.
函数是顺序执行，遇到return语句或者最后一行函数语句就返回。而变成generator的函数，在每次调用next()的时候执行，遇到yield语句返回，再次执行时从上次返回的yield语句处继续执行。

其实就相当于在yield处有个断点，可以获得当时的yield处的值，供for循环内使用。




最后再次感谢廖老师的辛勤劳动。
