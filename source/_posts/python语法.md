title: python 3.5 基础语法
date: 2016/8/7 10:07:12  
categories: python
tags:
	- 读书笔记

---

python还是大一时学的第一门课编程课，学的时候由于没有编程基础，最主要的是完全没有好好学，很煎熬。现在真是悔不当初。
当时学的是2.x版本，有点过时了，现在看了[廖雪峰的博客](http://www.liaoxuefeng.com/)的3.5版本的语法介绍，做一些摘录。

<!--more-->
  

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

















最后再次感谢廖老师的辛勤劳动。






