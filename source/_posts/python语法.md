title: python 3.5基础语法
date: 2016/8/7 10:07:12  
categories: python
tags:
	- 读书笔记

---

python还是大一时学的第一门课编程课，学的时候由于没有编程基础，最主要的是完全没有好好学，很煎熬。现在真是悔不当初。
当时学的是2.x版本，有点过时了，现在看了[廖雪峰的博客](http://www.liaoxuefeng.com/)的3.5版本的语法介绍，做一些摘录。

<!--more-->


## Python基础

Python采用缩进方式,写出来的代码就像下面的样子：
```python
# print absolute value of an integer:
a = 100
if a >= 0:
    print(a)
else:
    print(-a)
```
以#开头的语句是注释
其他每一行都是一个语句，当语句以冒号:结尾时，缩进的语句视为代码块.

### 数据类型和变量
#### 字符串
字符串是以单引号'或双引号"括起来的任意文本.如果字符串内部既包含'又包含",可以用转义字符\来标识。
要计算str包含多少个字符，可以用len()函数。
%运算符就是用来格式化字符串，如：
```python
>>> 'Hello, %s' % 'world'
'Hello, world'
>>> 'Hi, %s, you have $%d.' % ('Michael', 1000000)
'Hi, Michael, you have $1000000.'
```
有些时候，字符串里面的%是一个普通字符怎么办？这个时候就需要转义，用%%来表示一个%.

#### 布尔值
一个布尔值只有True、False两种值
布尔值可以用and、or和not运算。

#### 空值
空值是Python里一个特殊的值，用None表示。

#### 变量
等号=是赋值语句，可以把任意数据类型赋值给变量，同一个变量可以反复赋值，而且可以是不同类型的变量

### 数组和元组
#### list
list是一种**可变的**有序的集合，用len()函数可以获得list元素的个数
```python
classmates = ['Michael', 'Bob', 'Tracy']
```
要把某个元素替换成别的元素，可以直接赋值给对应的索引位置
list里面的元素的数据类型也可以不同
列表可以看成一个多维数组 s[2][1]拿到元素。
如果一个list中一个元素也没有，就是一个空的list，它的长度为0

#### tuple
另一种有序列表叫元组，但是tuple一旦初始化就不能修改
```python
classmates = ('Michael', 'Bob', 'Tracy')
```
因为tuple不可变，所以代码更安全。如果可能，能用tuple代替list就尽量用tuple。
当你定义一个tuple时，在定义的时候，tuple的元素就必须被确定下来.

如果要定义一个空的tuple，可以写成(),但是，要定义一个只有1个元素的tuple,必须写成如下
```python
t = (1,)
```
这是因为括号()既可以表示tuple，又可以表示数学公式中的小括号，这就产生了歧义

### 条件判断
if-else语句实现：
```python
age = 20
if age >= 18:
    print('your age is', age)
    print('adult')
else:
    print('your age is', age)
    print('teenager')
```
注意不要少写了冒号:
完全可以用elif做更细致的判断,elif是else if的缩写

### 循环
#### for...in
依次把list或tuple中的每个元素迭代出来
```python
names = ['Michael', 'Bob', 'Tracy']
for name in names:
    print(name)
```
Python提供一个range()函数，可以生成一个整数序列，再通过list()函数可以转换为list.在for...in中可以简写成range()
```python
sum = 0
for x in range(101):
    sum = sum + x
print(sum)
```

#### while
```python
sum = 0
n = 99
while n > 0:
    sum = sum + n
    n = n - 2
print(sum)
```

### 容器
#### dict
```python
d = {'Michael': 95, 'Bob': 75, 'Tracy': 85}
```
**要保证hash的正确性，作为key的对象就不能变。在Python中，字符串、整数等都是不可变的，因此，可以放心地作为key。而list是可变的，就不能作为key**

#### set
set和dict类似，**也是一组key的集合**，但不存储value。由于key不能重复，所以，在set中，没有重复的key。
要创建一个set，需要**提供一个list作为输入集合**：
```python
>>> s = set([1, 2, 3])
>>> s
{1, 2, 3}
```
重复元素在set中自动被过滤
```python
>>> s = set([1, 1, 2, 2, 3, 3])
>>> s
{1, 2, 3}
```

## 函数
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
1. 必选参数在前，默认参数在后，否则Python的解释器会报错。
2. 默认参数必须指向不变对象
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

## 高级特性
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

## 函数式编程
函数式编程的一个特点就是，允许把函数本身作为参数传入另一个函数，还允许返回一个函数！
Python对函数式编程提供部分支持。由于Python允许使用变量，因此，Python不是纯函数式编程语言。

### 高阶函数
#### 变量可以指向函数
把函数本身赋值给变量,即：变量可以指向函数。
```python
>>> f = abs
>>> f(-10)
10
```
说明变量f现在已经指向了abs函数本身。直接调用abs()函数和调用变量f()完全相同。

#### 传入函数
那么一个函数就可以接收另一个函数作为参数，这种函数就称之为高阶函数
```python
def add(x, y, f):
    return f(x) + f(y)
```

#### map
map()函数接收两个参数，一个是函数，一个是Iterable，map将传入的函数依次作用到序列的每个元素，并把结果作为新的Iterator返回。
```python
>>> def f(x):
...     return x * x
...
>>> r = map(f, [1, 2, 3, 4, 5, 6, 7, 8, 9])
>>> list(r)
[1, 4, 9, 16, 25, 36, 49, 64, 81]
```

#### reduce
reduce把一个函数作用在一个序列[x1, x2, x3, ...]上，这个函数**必须接收两个参数**，reduce把结果继续和序列的下一个元素做累积计算，其效果就是：
```python
reduce(f, [x1, x2, x3, x4]) = f(f(f(x1, x2), x3), x4)
```

#### filter
filter()也接收一个函数和一个序列.filter()把传入的函数依次作用于每个元素，然后根据返回值是True还是False决定保留还是丢弃该元素。
```python
def is_odd(n):
    return n % 2 == 1
list(filter(is_odd, [1, 2, 4, 5, 6, 9, 10, 15]))
# 结果: [1, 5, 9, 15]
```

#### sorted
使用sorted()函数就可以对list进行排序。
sorted()函数也是一个高阶函数，它还可以接收一个key函数来实现自定义的排序：
```python
>>> sorted([36, 5, -12, 9, -21], key=abs)
[5, 9, -12, -21, 36]
```
**key指定的函数将作用于list的每一个元素上，并根据key函数返回的结果进行排序。对比原始的list和经过key=abs处理过的list：**
```python
list = [36, 5, -12, 9, -21]
keys = [36, 5,  12, 9,  21]
```

### 返回函数
高阶函数除了接收函数作为参数外，还能将函数作为结果返回。好处是，不需要立即执行，在想要调用的时候执行。
```python
def count():
    fs = []
    for i in range(1, 4):
        def f():
             return i*i
        fs.append(f)
    return fs

f1, f2, f3 = count()

>>> f1()
9
>>> f2()
9
>>> f3()
9
```
本例中，执行count()返回了一个以f函数作为元素的数组，分别赋给f1，f2，f3。这里面执行三个函数的结果都是9，因为外层i在循环的时候并没有执行i*i，当循环完后，i为3，由于闭包性，i=3被保存在栈中，直到函数执行。
因此，返回闭包时牢记的一点就是：返回函数不要引用任何循环变量，或者后续会发生变化的变量。

### 匿名函数
```python
lambda x: x * x
=>
def f(x):
    return x * x
```

关键字lambda表示匿名函数，冒号前面的x表示函数参数。
匿名函数有个限制，就是**只能有一个表达式**，不用写return，返回值就是该表达式的结果。
匿名函数也是一个函数对象，也可以把匿名函数赋值给一个变量，再利用变量来调用该函数

### 装饰器
















最后再次感谢廖老师的辛勤劳动。






