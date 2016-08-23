title: python 3.5 模块
date: 2016/8/8 10:07:12  
categories: python
tags:
	- 读书笔记

---

当时学的是2.x版本，有点过时了，现在看了[廖雪峰的博客](http://www.liaoxuefeng.com/)的3.5版本的语法介绍，做一些摘录。

<!--more-->

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
#### 不带参数
在代码运行期间动态增加功能的方式，称之为“装饰器”（Decorator）。
```python
def log(func):
    def wrapper(*args, **kw):
    	# 函数对象有一个__name__属性，可以拿到函数的名字
        print('call %s():' % func.__name__)
        return func(*args, **kw)
    return wrapper
@log
def now():
    print('2015-3-25')
    
>>> now()
call now():
2015-3-25
```

此处把**@log**放到**now()**定义处，相当于执行了
**now = log(now)**: **now() => wrapper()**
将原方法作为参数传入。类似于装饰者模式，只不过由于python的动态性，不需要调用新定义的方法，只要调用原方法就可以动态解析。

#### 带参数
如果log带参数
```python
def log(text):
    def decorator(func):
        def wrapper(*args, **kw):
            print('%s %s():' % (text, func.__name__))
            return func(*args, **kw)
        return wrapper
    return decorator

@log('execute')
def now():
    print('2015-3-25')
```

首先执行log('execute')，返回的是decorator函数，再调用返回的函数，参数是now函数，返回值最终是wrapper函数。即**now()=>wrapper()**

#### 带来的问题
上面的过程解析已经说明，最后now()的调用，都转化成了wrapper()的调用。那么，在调用**now.__name__**时，结果就会使wrapper，而不是now。
因此，需要将**wrapper.__name__ = func.__name__**。可以使用python内置的方法**functools.wraps**

```
import functools
def log(func):
    @functools.wraps(func)
    def wrapper(*args, **kw):
        print('call %s():' % func.__name__)
        return func(*args, **kw)
    return wrapper
# 或者
import functools
def log(text):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kw):
            print('%s %s():' % (text, func.__name__))
            return func(*args, **kw)
        return wrapper
    return decorator  
```

### 偏函数
使用functools.partial可以创建一个新的函数，这个新函数可以固定住原函数的部分参数，从而在调用时更简单

假设要转换大量的二进制字符串，每次都传入int(x, base=2)非常麻烦，于是，我们想到，可以定义一个int2()的函数，默认把base=2传进去：
```python
def int2(x, base=2):
    return int(x, base)
```

我们可以使用**functools.partial**创建一个偏函数，不需要自己定义int2
```python
>>> import functools
>>> int2 = functools.partial(int, base=2)
>>> int2('1000000')
64
>>> int2('1010101')
85
```
简单总结functools.partial的作用就是，把一个函数的某些参数给固定住（也就是设置默认值），返回一个新的函数，调用这个新函数会更简单

创建偏函数时，实际上可以接收函数对象、\*args和\*\*kw这3个参数.
```python
max2 = functools.partial(max, 10)
```
实际上会把10作为\*args的一部分**自动加到左边**，也就是：
```python
max2(5, 6, 7)
相当于：
args = (10, 5, 6, 7)
max(*args)
```

最后再次感谢廖老师的辛勤劳动。