title: python 3.5 面向对象高级编程
date: 2016/8/10 10:07:12  
categories: python
tags:
	- 读书笔记

---

当时学的是2.x版本，有点过时了，现在看了[廖雪峰的博客](http://www.liaoxuefeng.com/)的3.5版本的语法介绍，做一些摘录。

<!--more-->


### 使用__slots__
我们可以给实例绑定任何属性和方法。
创建实例：
```python
class Student(object):
    pass
```

给实例绑定方法：
```python
>>> def set_age(self, age): # 定义一个函数作为实例方法
...     self.age = age
...
>>> from types import MethodType
>>> s.set_age = MethodType(set_age, s) # 给实例绑定一个方法
>>> s.set_age(25) # 调用实例方法
>>> s.age # 测试结果
25
```
**注意：**
1. 这里使用MethodType方法给实例绑定方法，之后调用的时候就不用设置self了。如果使用s.set_age = set_age的方式绑定，那么调用时要自己传入self变量。 
2. MethodType()这个方法不要用在给类绑定属性上。

给类绑定方法:
```python
>>> def set_score(self, score):
...     self.score = score
...
>>> Student.set_score = set_score
```
**注意：**像这样给类绑定方法后，实例变量不用自己传入self了。

如果我们想要限制实例的属性。比如，只允许对Student实例添加name和age属性。我们可以使用__slots__来限制属性。
```python
class Student(object):
    __slots__ = ('name', 'age') # 用tuple定义允许绑定的属性名称
```
使用__slots__要注意，__slots__定义的属性仅对当前类实例起作用，对继承的子类是不起作用的,即**__slots__属性并不会被继承**

slots的本质不是限制实例添加属性，而是优化性能。slots绑定的实例属性不保存在dict中，所以在有大量实例存在的情况下能减少hash table的内存开销。不能给实例增加,不能给实例动态添加属性只是__slots__的副作用。

### 使用@property
类的属性都是暴露出来的，写起来方便，但是没办法检查参数。如果使用get，set方法又显得麻烦。可以使用装饰器(decorator)中的@property装饰器。
```python
class Student(object):

    @property
    def score(self):
        return self._score

    @score.setter
    def score(self, value):
        if not isinstance(value, int):
            raise ValueError('score must be an integer!')
        if value < 0 or value > 100:
            raise ValueError('score must between 0 ~ 100!')
        self._score = value
```
注意：
1. 把一个getter方法变成属性，只需要加上@property就可以了，此时，@property本身又创建了另一个装饰器@score.setter，负责把一个setter方法变成属性赋值.
2. 只定义getter方法，不定义setter方法就是一个只读属性。

### 多重继承
python允许多重继承：
```python
class Dog(Mammal, Runnable):
    pass
```
**如果继承的类有同名方法，按照继承的顺序执行，即先执行Mammal里的，没有再执行Runnable里的。**

在设计类的继承关系时，通常，主线都是单一继承下来的，例如，Ostrich继承自Bird。但是，如果需要“混入”额外的功能，通过多重继承就可以实现，比如，让Ostrich除了继承自Bird外，再同时继承Runnable。这种设计通常称之为MixIn。
```python
class Dog(Mammal, RunnableMixIn, CarnivorousMixIn):
    pass
```

感觉MixIn就是个约定啊，并没有太多实质效果啊=。=

### 定制类
形如__xxx__的变量或者函数名就要注意，这些在Python中是有特殊用途的。

#### __str__
怎么才能打印得好看呢？只需要定义好__str__()方法，返回一个好看的字符串就可以了：
```python
>>> class Student(object):
...     def __init__(self, name):
...         self.name = name
...     def __str__(self):
...         return 'Student object (name: %s)' % self.name
...
>>> print(Student('Michael'))
Student object (name: Michael)
```

#### __iter__
如果一个类想被用于**for...in**循环，类似list或tuple那样，就必须实现一个**__iter__()**方法，该方法返回一个迭代对象，然后，Python的for循环就会不断调用该迭代对象的**__next__()**方法拿到循环的下一个值，直到遇到**StopIteration**错误时退出循环。
```python
class Fib(object):
    def __init__(self):
        self.a, self.b = 0, 1 # 初始化两个计数器a，b

    def __iter__(self):
        return self # 实例本身就是迭代对象，故返回自己

    def __next__(self):
        self.a, self.b = self.b, self.a + self.b # 计算下一个值
        if self.a > 100000: # 退出循环的条件
            raise StopIteration();
        return self.a # 返回下一个值

>>> for n in Fib():
...     print(n)
```

#### __getitem__
要表现得像list那样按照下标取出元素，需要实现__getitem__()方法。__getitem__()传入的参数可能是一个int，也可能是一个切片对象slice:
```python
class Fib(object):
    def __getitem__(self, n):
        if isinstance(n, int): # n是索引
            a, b = 1, 1
            for x in range(n):
                a, b = b, a + b
            return a
        if isinstance(n, slice): # n是切片
            start = n.start
            stop = n.stop
            if start is None:
                start = 0
            a, b = 1, 1
            L = []
            for x in range(stop):
                if x >= start:
                    L.append(a)
                a, b = b, a + b
            return L
```
与之对应的是__setitem__()方法，把对象视作list或dict来对集合赋值。最后，还有一个__delitem__()方法，用于删除某个元素。

#### __getattr__
正常情况下，当我们调用类的方法或属性时，如果不存在，就会报错。要避免这个错误，除了可以加上一个score属性外，Python还有另一个机制，那就是写一个__getattr__()方法，动态返回一个属性。
```python
class Student(object):

    def __init__(self):
        self.name = 'Michael'

    def __getattr__(self, attr):
        if attr=='score':
            return 99
```

当调用**不存在的属性**时，比如score，Python解释器会试图调用__getattr__(self, 'score')来尝试获得属性，这样，我们就有机会返回score的值.

#### __call__
任何类，只需要定义一个__call__()方法，就可以直接对实例进行调用。
```python
class Student(object):
    def __init__(self, name):
        self.name = name

    def __call__(self):
        print('My name is %s.' % self.name)

>>> s = Student('Michael')
>>> s() # self参数不要传入
My name is Michael.
```

### 枚举类
Python提供了Enum类来实现这个功能
```python
from enum import Enum
Month = Enum('Month', ('Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'))

for name, member in Month.__members__.items():
    print(name, '=>', member, ',', member.value)
```

value属性则是自动赋给成员的int常量，默认从1开始计数。
如果需要更精确地控制枚举类型，可以从Enum派生出自定义类：

```python
from enum import Enum, unique
class Weekday(Enum):
    Sun = 0 # Sun的value被设定为0
    Mon = 1
    Tue = 2
    Wed = 3
    Thu = 4
    Fri = 5
    Sat = 6
```

### 元类
没看

最后再次感谢廖老师的辛勤劳动。