title: python 3.5 函数
date: 2016/8/7 10:07:12  
categories: python
tags:
	- 读书笔记

---

当时学的是2.x版本，有点过时了，现在看了[廖雪峰的博客](http://www.liaoxuefeng.com/)的3.5版本的语法介绍，做一些摘录。

<!--more-->



### 调用函数
#### 数据类型转换
Python内置的常用函数还包括数据类型转换函数，比如int()函数可以把其他数据类型转换为整数

#### 函数名
函数名其实就是指向一个函数对象的引用，完全可以把函数名赋给一个变量，相当于给这个函数起了一个“别名”
```python
>>> a = abs # 变量a指向abs函数
>>> a(-1) # 所以也可以通过a调用abs函数
1
```

### 定义函数
定义一个函数要使用def语句，依次写出函数名、括号、括号中的参数和冒号:，然后，在缩进块中编写函数体，函数的返回值用return语句返回。
```python
def my_abs(x):
    if x >= 0:
        return x
    else:
        return -x
```
可以使用from abstest import my_abs来导入my_abs()函数，abstest是my_abs所在的文件名.

#### 空函数
用pass语句定义一个什么事都不做的空函数，不用pass会报错。
```python
def nop():
    pass
# 或者    
if age >= 18:
    pass
```

#### 返回多个值
```python
import math
def move(x, y, step, angle=0):
    nx = x + step * math.cos(angle)
    ny = y - step * math.sin(angle)
    return nx, ny
```

得到返回值
```python
>>> x, y = move(100, 100, 60, math.pi / 6)
>>> print(x, y)
151.96152422706632 70.0
>>> r = move(100, 100, 60, math.pi / 6)
>>> print(r)
(151.96152422706632, 70.0)
```
其实Python函数返回的仍然是单一值.返回值是一个tuple！但是，在语法上，返回一个tuple可以省略括号，而多个变量可以同时接收一个tuple，按位置赋给对应的值，所以，Python的函数返回多值其实就是返回一个tuple，但写起来更方便。

### 函数的参数
#### 默认参数
python不支持重载，可以使用默认参数的方式替代。
```python
def power(x, n=2):
    s = 1
    while n > 0:
        n = n - 1
        s = s * x
    return s
```
第二个参数n的默认值设定为2.这样，当我们调用power(5)时，相当于调用power(5, 2).

但是需要注意：
1. 必选参数在前，默认参数在后，否则Python的解释器会报错。**如果不按顺序提供参数时，需要写成 参数名=xx 的形式。**
2. 默认参数必须指向不变对象。
例如：
```python
def add_end(L=[]):
    L.append('END')
    return L
    
>>> add_end()
['END']
>>> add_end()
['END', 'END']
>>> add_end()
['END', 'END', 'END']
```
当L缺省后，L指向一个数组对象的地址。每次append后，那个地址的数组元素发生改变。

如果要默认是list，可以这么写：
```python
def add_end(L=None):
    if L is None:
        L = []
    L.append('END')
    return L
```

#### 可变参数
通常可以通过list或tuple实现传入不确定数量的参数。python支持可变参数，写法如下:
```python
def calc(*numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum

>>> calc(1, 2, 3)
14
>>> calc(1, 3, 5, 7)
84
```
**可变参数允许你传入0个或任意个参数，这些可变参数在函数调用时自动组装为一个tuple。因此无法再内部函数改变传入的list**
**不是可变参数的话，参数需要一一对应**

如果已经有一个list或者tuple，要调用一个可变参数可以写成这样：
```python
>>> nums = [1, 2, 3]
>>> print(nums)
[1, 2, 3]
>>> print(*nums)
1, 2, 3
# 调用
>>> calc(*nums)
14
```
**nums表示一个数组或者元组
\*nums表示取出nums这个list里的所有元素，代表多个参数**

#### 关键字参数
关键字参数允许你传入0个或任意个含参数名的参数，这些关键字参数在函数内部自动组装为一个dict。如：
```python
# 定义函数
def person(name, age, **kw):
    print('name:', name, 'age:', age, 'other:', kw)
# 调用
>>> person('Adam', 45, gender='M', job='Engineer')
name: Adam age: 45 other: {'gender': 'M', 'job': 'Engineer'}
# 另一种简化写法
>>> extra = {'city': 'Beijing', 'job': 'Engineer'}
>>> person('Jack', 24, **extra)
name: Jack age: 24 other: {'city': 'Beijing', 'job': 'Engineer'}
```
**kw表示一个dict
\*\*kw表示取出kw中的所有键值对元素，代表多个参数**

**另外，和上面的list传入的是tuple一样，kw获得的dict是外部传入的一份拷贝，在函数内部对kw的修改不会影响到外部dict。**

#### 命名关键字参数
不知道这么脑残的语法有什么意义。



最后再次感谢廖老师的辛勤劳动。

