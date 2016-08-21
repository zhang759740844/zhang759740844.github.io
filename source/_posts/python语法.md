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

## 模块
Python又引入了按目录来组织模块的方法，称为包（Package）。引入了包以后，只要顶层的包名不与别人冲突，那所有模块都不会与别人冲突。
请注意，每一个包目录下面都会有一个__init\_\_.py的文件，这个文件是必须存在的，否则，Python就把这个目录当成普通目录，而不是一个包。__init\_\_.py可以是空文件，也可以有Python代码，因为__init__.py本身就是一个模块

### 使用模块
```python
import sys
def test():
	pass
if __name__=='__main__':
	test()
```
导入sys模块后,就有了变量sys指向该模块，利用sys这个变量，就可以访问sys模块的所有功能。
sys模块有一个argv变量，用list存储了**命令行的所有参数**。argv至少有一个元素，因为第一个参数永远是该.py文件的名称。
在命令行运行该模块文件时，Python解释器把一个特殊变量__name__置为__main__，而如果在其他地方导入该模块时，if判断将失败，因此，这种if测试可以让一个模块通过命令行运行时执行一些额外的代码，最常见的就是运行测试。

作用域：
- 正常的函数和变量名是公开的（public），可以被直接引用
- 类似__xxx__这样的变量是特殊变量，可以被直接引用，但是有特殊用途。
- 类似_xxx和__xxx这样的函数或变量就是非公开的（private），不应该被直接引用。

### 安装第三方模块
使用pip3 install XXX 安装第三方库
当我们试图加载一个模块时，Python会在指定的路径下搜索对应的.py文件，如果找不到，就会报错。
搜索路径存放在sys模块的path变量中
```python
import sys
print(sys.path)
```
当要添加自己的搜索目录时可以
1. 直接修改sys.path:**sys.path.append('/Users/xxx/xxx')**
2. 设置环境变量PYTHONPATH

## 面向对象编程
### 类和实例
定义类是通过class关键字，后面紧接着是类名，紧接着是(object)，表示该类是从哪个类继承下来的。
```python
class Student(object)
	pass
```

定义好类，就可以创建出实例了。
```python
bart = Student()
```

可以**自由地给一个实例变量绑定属性**，比如，给实例bart绑定一个name属性：
```python
bart.name = 'Zachary'
```
和静态语言不同，Python允许对实例变量绑定任何数据，也就是说，对于两个实例变量，虽然它们都是同一个类的不同实例，但拥有的变量名称都可能不同。

创建实例的时候可以使用特殊的**__init__**方法初始化：
```python
class Student(object):
    def __init__(self, name, score):
        self.name = name
        self.score = score
```
注意：
1. 注意到__init__方法的第一个参数永远是self，表示创建的实例本身，因此，在__init__方法内部，就可以把各种属性绑定到self，因为self就指向创建的实例本身。
2. 有了__init__方法，在创建实例的时候，就不能传入空的参数了，必须传入与__init__方法匹配的参数，但self不需要传，Python解释器自己会把实例变量传进去。

另外：
1. 和普通的函数相比，在类中定义的函数只有一点不同，就是第一个参数永远是实例变量self，并且，调用时，不用传递该参数
2. 如果是类的方法，不需要传入self，使用类名.方法名调用。和其他语言一样。

### 访问限制
如果要让内部属性不被外部访问，可以把属性的名称前加上两个下划线__，就变成了一个私有变量（private），只有内部可以访问，外部不能访问。

oc中使用.h和.m定义属性的方式区分共有私有。
Java中通过public，private的方式区分共有私有。
python通过__的方式区分共有私有。

以_开头的变量表示：可以访问，但是最好视为私有变量。

### 继承和多态
继承和多态和其他语言没什么不同，就不重复了。

对于静态语言（例如Java）来说，如果需要传入Animal类型，则传入的对象必须是Animal类型或者它的子类，否则，将无法调用run()方法。
对于Python这样的动态语言来说，则不一定需要传入Animal类型。我们只需要保证传入的对象有一个run()方法就可以了

### 获取对象信息
#### type
使用type()函数,判断对象类型
```python
>>> import types
>>> def fn():
...     pass
...
>>> type(fn)==types.FunctionType
True
>>> type(abs)==types.BuiltinFunctionType
True
>>> type(lambda x: x)==types.LambdaType
True
>>> type((x for x in range(10)))==types.GeneratorType
True
```

#### isinstance()
使用isinstance()函数,判断class的继承关系
判断基本类型：
```python
>>> isinstance('a', str)
True
>>> isinstance(123, int)
True
>>> isinstance(b'a', bytes)
True
```
判断一个变量是否是某些类型中的一种，比如下面的代码就可以判断是否是list或者tuple：
```python
>>> isinstance([1, 2, 3], (list, tuple))
True
>>> isinstance((1, 2, 3), (list, tuple))
True
```

#### dir()
如果要获得一个对象的**所有属性和方法**，可以使用dir()函数，它返回一个包含字符串的list.

仅仅把属性和方法列出来是不够的，配合**getattr()**、**setattr()**以及**hasattr()**，我们可以直接**操作一个对象的状态**：
```python
>>> class MyObject(object):
...     def __init__(self):
...         self.x = 9
...     def power(self):
...         return self.x * self.x
...
>>> obj = MyObject()
```
紧接着，可以测试该对象的属性：
```python
>>> hasattr(obj, 'x') # 有属性'x'吗？
True
>>> obj.x
9
>>> hasattr(obj, 'y') # 有属性'y'吗？
False
>>> setattr(obj, 'y', 19) # 设置一个属性'y'
>>> hasattr(obj, 'y') # 有属性'y'吗？
True
>>> getattr(obj, 'y') # 获取属性'y'
19
>>> obj.y # 获取属性'y'
19
```
如果试图获取不存在的属性，会抛出AttributeError的错误
可以传入一个default参数，如果属性不存在，就返回默认值：
```python
>>> getattr(obj, 'z', 404) # 获取属性'z'，如果不存在，返回默认值404
404
```

也可以获得对象的方法：
```python
>>> hasattr(obj, 'power') # 有属性'power'吗？
True
>>> getattr(obj, 'power') # 获取属性'power'
<bound method MyObject.power of <__main__.MyObject object at 0x10077a6a0>>
>>> fn = getattr(obj, 'power') # 获取属性'power'并赋值到变量fn
>>> fn # fn指向obj.power
<bound method MyObject.power of <__main__.MyObject object at 0x10077a6a0>>
>>> fn() # 调用fn()与调用obj.power()是一样的
81
```

**感觉上，用set，get，has方法和直接设置没什么太大区别。**

### 实例属性和类属性
python中并没有static修饰符，在一个class中定义的属性，实例和类都可以访问：
```python
>>> class Student(object):
...     name = 'Student'
...
>>> s = Student() # 创建实例s
>>> print(s.name) # 打印name属性，因为实例并没有name属性，所以会继续查找class的name属性
Student
>>> print(Student.name) # 打印类的name属性
Student
>>> s.name = 'Michael' # 给实例绑定name属性
>>> print(s.name) # 由于实例属性优先级比类属性高，因此，它会屏蔽掉类的name属性
Michael
>>> print(Student.name) # 但是类属性并未消失，用Student.name仍然可以访问
Student
>>> del s.name # 如果删除实例的name属性
>>> print(s.name) # 再次调用s.name，由于实例的name属性没有找到，类的name属性就显示出来了
Student
```
在编写程序的时候，千万不要把实例属性和类属性使用相同的名字，因为相同名称的实例属性将屏蔽掉类属性，但是当你删除实例属性后，再使用相同的名称，访问到的将是类属性。

## 面向对象高级编程
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

## IO编程
### 文件读写
#### 读文件
使用Python内置的open()函数，传入文件名和标示符
```python
>>> f = open('/Users/test.txt', 'r')
```
标示符'r'表示读.

如果文件打开成功，接下来，调用read()方法可以一次读取文件的全部内容，Python把内容读到内存，用一个str对象表示：
```python
>>> f.read()
'Hello, world!'
```

最后一步是调用close()方法关闭文件。文件使用完毕后必须关闭，因为文件对象会占用操作系统的资源，并且操作系统同一时间能打开的文件数量也是有限的
```python
>>> f.close()
```

由于文件读写时都有可能产生IOError，一旦出错，后面的f.close()就不会调用。所以，为了保证无论是否出错都能正确地关闭文件，我们可以使用try ... finally来实现：
```python
try:
    f = open('/path/to/file', 'r')
    print(f.read())
finally:
    if f:
        f.close()
```

但是每次都这么写实在太繁琐，所以，Python引入了with语句来自动帮我们调用close()方法：
```python
with open('/path/to/file', 'r') as f:
    print(f.read())
```
这和前面的try ... finally是一样的，但是代码更佳简洁，并且不必调用f.close()方法。

调用read()会一次性读取文件的全部内容，如果文件有10G，内存就爆了，所以，要保险起见，可以反复调用read(size)方法，每次最多读取size个字节的内容。另外，调用readline()可以每次读取一行内容，调用readlines()一次读取所有内容并按行返回list。因此，要根据需要决定怎么调用。

如果文件很小，read()一次性读取最方便；如果不能确定文件大小，反复调用read(size)比较保险；如果是配置文件，调用readlines()最方便：
```python
for line in f.readlines():
    print(line.strip()) # 把末尾的'\n'删掉
```

#### 二进制文件
前面讲的默认都是读取文本文件，并且是UTF-8编码的文本文件。要读取二进制文件，比如图片、视频等等，用'rb'模式打开文件即可：
```python
>>> f = open('/Users/michael/test.jpg', 'rb')
>>> f.read()
b'\xff\xd8\xff\xe1\x00\x18Exif\x00\x00...' # 十六进制表示的字节
```

#### 文件读写

阅读: 100273
读写文件是最常见的IO操作。Python内置了读写文件的函数，用法和C是兼容的。

读写文件前，我们先必须了解一下，在磁盘上读写文件的功能都是由操作系统提供的，现代操作系统不允许普通的程序直接操作磁盘，所以，读写文件就是请求操作系统打开一个文件对象（通常称为文件描述符），然后，通过操作系统提供的接口从这个文件对象中读取数据（读文件），或者把数据写入这个文件对象（写文件）。

读文件

要以读文件的模式打开一个文件对象，使用Python内置的open()函数，传入文件名和标示符：

>>> f = open('/Users/michael/test.txt', 'r')
标示符'r'表示读，这样，我们就成功地打开了一个文件。

如果文件不存在，open()函数就会抛出一个IOError的错误，并且给出错误码和详细的信息告诉你文件不存在：

>>> f=open('/Users/michael/notfound.txt', 'r')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
FileNotFoundError: [Errno 2] No such file or directory: '/Users/michael/notfound.txt'
如果文件打开成功，接下来，调用read()方法可以一次读取文件的全部内容，Python把内容读到内存，用一个str对象表示：

>>> f.read()
'Hello, world!'
最后一步是调用close()方法关闭文件。文件使用完毕后必须关闭，因为文件对象会占用操作系统的资源，并且操作系统同一时间能打开的文件数量也是有限的：

>>> f.close()
由于文件读写时都有可能产生IOError，一旦出错，后面的f.close()就不会调用。所以，为了保证无论是否出错都能正确地关闭文件，我们可以使用try ... finally来实现：

try:
    f = open('/path/to/file', 'r')
    print(f.read())
finally:
    if f:
        f.close()
但是每次都这么写实在太繁琐，所以，Python引入了with语句来自动帮我们调用close()方法：

with open('/path/to/file', 'r') as f:
    print(f.read())
这和前面的try ... finally是一样的，但是代码更佳简洁，并且不必调用f.close()方法。

调用read()会一次性读取文件的全部内容，如果文件有10G，内存就爆了，所以，要保险起见，可以反复调用read(size)方法，每次最多读取size个字节的内容。另外，调用readline()可以每次读取一行内容，调用readlines()一次读取所有内容并按行返回list。因此，要根据需要决定怎么调用。

如果文件很小，read()一次性读取最方便；如果不能确定文件大小，反复调用read(size)比较保险；如果是配置文件，调用readlines()最方便：

for line in f.readlines():
    print(line.strip()) # 把末尾的'\n'删掉
file-like Object

像open()函数返回的这种有个read()方法的对象，在Python中统称为file-like Object。除了file外，还可以是内存的字节流，网络流，自定义流等等。file-like Object不要求从特定类继承，只要写个read()方法就行。

StringIO就是在内存中创建的file-like Object，常用作临时缓冲。

#### 二进制文件

前面讲的默认都是读取文本文件，并且是UTF-8编码的文本文件。要读取二进制文件，比如图片、视频等等，用'rb'模式打开文件即可：

>>> f = open('/Users/michael/test.jpg', 'rb')
>>> f.read()
b'\xff\xd8\xff\xe1\x00\x18Exif\x00\x00...' # 十六进制表示的字节
字符编码

#### 字符编码
要读取非UTF-8编码的文本文件，需要给open()函数传入encoding参数，例如，读取GBK编码的文件：
```python
>>> f = open('/Users/michael/gbk.txt', 'r', encoding='gbk')
>>> f.read()
'测试'
```

遇到有些编码不规范的文件，你可能会遇到UnicodeDecodeError，因为在文本文件中可能夹杂了一些非法编码的字符。遇到这种情况，open()函数还接收一个errors参数，表示如果遇到编码错误后如何处理。最简单的方式是直接忽略：
```python
>>> f = open('/Users/michael/gbk.txt', 'r', encoding='gbk', errors='ignore')
```

#### 写文件
写文件和读文件是一样的，唯一区别是调用open()函数时，传入标识符'w'或者'wb'表示写文本文件或写二进制文件：
```python
>>> f = open('/Users/michael/test.txt', 'w')
>>> f.write('Hello, world!')
>>> f.close()
```

你可以反复调用write()来写入文件，但是务必要调用f.close()来关闭文件。当我们写文件时，操作系统往往不会立刻把数据写入磁盘，而是放到内存缓存起来，空闲的时候再慢慢写入。只有调用close()方法时，操作系统才保证把没有写入的数据全部写入磁盘。忘记调用close()的后果是数据可能只写了一部分到磁盘，剩下的丢失了。所以，还是用with语句来得保险：
```python
with open('/Users/michael/test.txt', 'w') as f:
    f.write('Hello, world!')
```

### StringIO和BytesIO
读取写内容到内存中，不写了。用到再看。

### 操作文件和目录
```python
>>> import os
# 查看当前目录的绝对路径:
>>> os.path.abspath('.')
'/Users/michael'
# 在某个目录下创建一个新目录，首先把新目录的完整路径表示出来:
>>> os.path.join('/Users/michael', 'testdir')
'/Users/michael/testdir'
# 然后创建一个目录:
>>> os.mkdir('/Users/michael/testdir')
# 删掉一个目录:
>>> os.rmdir('/Users/michael/testdir')
```
把两个路径合成一个时，不要直接拼字符串，而要通过os.path.join()函数，这样可以正确处理不同操作系统的路径分隔符。
同样的道理，要拆分路径时，也不要直接去拆字符串，而要通过os.path.split()函数，这样可以把一个路径拆分为两部分，后一部分总是最后级别的目录或文件名：
```python
>>> os.path.split('/Users/michael/testdir/file.txt')
('/Users/michael/testdir', 'file.txt')
```

文件操作
```python
# 对文件重命名:
>>> os.rename('test.txt', 'test.py')
# 删掉文件:
>>> os.remove('test.py')
```

### 序列化
我们把变量从内存中变成可存储或传输的过程称之为序列化，在Python中叫pickling。
序列化之后，就可以把序列化后的内容写入磁盘，或者通过网络传输到别的机器上。
反过来，把变量内容从序列化的对象重新读到内存里称之为反序列化，即unpickling。

Python提供了pickle模块来实现序列化。
```python
>>> import pickle
>>> d = dict(name='Bob', age=20, score=88)
>>> f = open('dump.txt', 'wb')
>>> pickle.dump(d, f)
>>> f.close()
```
pickle.dumps()方法把任意对象序列化成一个bytes，然后，就可以把这个bytes写入文件。或者用另一个方法pickle.dump()直接把对象序列化后写入一个file-like Object.
当我们要把对象从磁盘读到内存时，可以先把内容读到一个bytes，然后用pickle.loads()方法反序列化出对象，也可以直接用pickle.load()方法从一个file-like Object中直接反序列化出对象:
```python
>>> f = open('dump.txt', 'rb')
>>> d = pickle.load(f)
>>> f.close()
>>> d
{'age': 20, 'score': 88, 'name': 'Bob'}
```

### JSON
Python内置的json模块提供了非常完善的Python对象到JSON格式的转换。
```python
>>> import json
>>> d = dict(name='Bob', age=20, score=88)
>>> json.dumps(d)
'{"age": 20, "score": 88, "name": "Bob"}'
```
dumps()方法返回一个str，内容就是标准的JSON。类似的，dump()方法可以直接把JSON写入一个file-like Object。

要把JSON反序列化为Python对象，用loads()或者对应的load()方法，前者把JSON的字符串反序列化，后者从file-like Object中读取字符串并反序列化：
```python
>>> json_str = '{"age": 20, "score": 88, "name": "Bob"}'
>>> json.loads(json_str)
{'age': 20, 'score': 88, 'name': 'Bob'}
```

## 进程和线程
### 多进程
#### multiprocessing
multiprocessing模块就是跨平台版本的多进程模块。
multiprocessing模块提供了一个Process类来代表一个进程对象
```python
from multiprocessing import Process
import os

# 子进程要执行的代码
def run_proc(name):
    print('Run child process %s (%s)...' % (name, os.getpid()))

if __name__=='__main__':
    print('Parent process %s.' % os.getpid())
    p = Process(target=run_proc, args=('test',))
    print('Child process will start.')
    p.start()
    p.join()
    print('Child process end.')
```
创建子进程时，只需要传入一个执行函数和函数的参数，创建一个Process实例，用start()方法启动，这样创建进程比fork()还要简单。
join()方法可以等待子进程结束后再继续往下运行，通常用于进程间的同步。

#### pool
如果要启动大量的子进程，可以用进程池的方式批量创建子进程：
```python
from multiprocessing import Pool
import os, time, random

def long_time_task(name):
    print('Run task %s (%s)...' % (name, os.getpid()))
    start = time.time()
    time.sleep(random.random() * 3)
    end = time.time()
    print('Task %s runs %0.2f seconds.' % (name, (end - start)))

if __name__=='__main__':
    print('Parent process %s.' % os.getpid())
    p = Pool(4)
    for i in range(5):
        p.apply_async(long_time_task, args=(i,))
    print('Waiting for all subprocesses done...')
    p.close()
    p.join()
    print('All subprocesses done.')
```
结果如下：
```python
Parent process 669.
Waiting for all subprocesses done...
Run task 0 (671)...
Run task 1 (672)...
Run task 2 (673)...
Run task 3 (674)...
Task 2 runs 0.14 seconds.
Run task 4 (673)...
Task 1 runs 0.27 seconds.
Task 3 runs 0.86 seconds.
Task 0 runs 1.41 seconds.
Task 4 runs 1.91 seconds.
All subprocesses done.
```

对Pool对象调用join()方法会等待所有子进程执行完毕，调用join()之前必须先调用close()，调用close()之后就不能继续添加新的Process了。
注意输出的结果，task 0，1，2，3是立刻执行的，而task 4要等待前面某个task完成后才执行，这是因为Pool的默认大小在我的电脑上是4，因此，最多同时执行4个进程。这是Pool有意设计的限制，并不是操作系统的限制。如果改成：`p = ool(5)`就可以同时跑5个进程。

### 子进程&进程间通信
到时候再看

## 多线程
启动一个线程就是把一个函数传入并创建Thread实例，然后调用start()开始执行：
```python
import time, threading

# 新线程执行的代码:
def loop():
    print('thread %s is running...' % threading.current_thread().name)
    n = 0
    while n < 5:
        n = n + 1
        print('thread %s >>> %s' % (threading.current_thread().name, n))
        time.sleep(1)
    print('thread %s ended.' % threading.current_thread().name)

print('thread %s is running...' % threading.current_thread().name)
t = threading.Thread(target=loop, name='LoopThread')
t.start()
t.join()
print('thread %s ended.' % threading.current_thread().name)
```
执行结果如下:

```python
thread MainThread is running...
thread LoopThread is running...
thread LoopThread >>> 1
thread LoopThread >>> 2
thread LoopThread >>> 3
thread LoopThread >>> 4
thread LoopThread >>> 5
thread LoopThread ended.
thread MainThread ended.
```
任何进程默认就会启动一个线程，我们把该线程称为主线程，主线程又可以启动新的线程，Python的threading模块有个current_thread()函数，它永远返回当前线程的实例。主线程实例的名字叫MainThread，子线程的名字在创建时指定，我们用LoopThread命名子线程。名字仅仅在打印时用来显示，完全没有其他意义，如果不起名字Python就自动给线程命名为Thread-1，Thread-2

### Lock
多线程和多进程最大的不同在于，多进程中，同一个变量，各自有一份拷贝存在于每个进程中，互不影响，而多线程中，所有变量都由所有线程共享，所以，任何一个变量都可以被任何一个线程修改，因此，线程之间共享数据最大的危险在于多个线程同时改一个变量，把内容给改乱了。
```python
balance = 0
lock = threading.Lock()

def run_thread(n):
    for i in range(100000):
        # 先要获取锁:
        lock.acquire()
        try:
            # 放心地改吧:
            change_it(n)
        finally:
            # 改完了一定要释放锁:
            lock.release()
```
创建一个锁就是通过threading.Lock()来实现.当多个线程同时执行lock.acquire()时，只有一个线程能成功地获取锁，然后继续执行代码，其他线程就继续等待直到获得锁为止。获得锁的线程用完后一定要释放锁，否则那些苦苦等待锁的线程将永远等待下去，成为死线程。所以我们用try...finally来确保锁一定会被释放。


### ThreadLocal
一个ThreadLocal变量虽然是全局变量，但每个线程都只能读写自己线程的独立副本，互不干扰。ThreadLocal解决了参数在一个线程中各个函数之间互相传递的问题。
```python
import threading

# 创建全局ThreadLocal对象:
local_school = threading.local()

def process_student():
    # 获取当前线程关联的student:
    std = local_school.student
    print('Hello, %s (in %s)' % (std, threading.current_thread().name))

def process_thread(name):
    # 绑定ThreadLocal的student:
    local_school.student = name
    process_student()

t1 = threading.Thread(target= process_thread, args=('Alice',), name='Thread-A')
t2 = threading.Thread(target= process_thread, args=('Bob',), name='Thread-B')
t1.start()
t2.start()
t1.join()
t2.join()
```
执行结果：

```python
Hello, Alice (in Thread-A)
Hello, Bob (in Thread-B)
```

全局变量local_school就是一个ThreadLocal对象，每个Thread对它都可以读写student属性，但互不影响。你可以把local_school看成全局变量，但每个属性如local_school.student都是线程的局部变量，可以任意读写而互不干扰，也不用管理锁的问题，ThreadLocal内部会处理。

可以理解为全局变量local_school是一个dict，不但可以用local_school.student，还可以绑定其他变量，如local_school.teacher等等。

ThreadLocal最常用的地方就是为每个线程绑定一个数据库连接，HTTP请求，用户身份信息等，这样一个线程的所有调用到的处理函数都可以非常方便地访问这些资源。

### 分布式进程
用到再看

## 正则表达式
### 基本用法
- \d可以匹配一个数字
- \w可以匹配一个字母或数字
- .可以匹配任意字符
- *表示任意个字符（包括0个）
- 用+表示至少一个字符
- 用?表示0个或1个字符
- 用{n}表示n个字符
- 用{n,m}表示n-m个字符
- \s可以匹配一个空格（也包括Tab等空白符）
- 要做更精确地匹配，可以用[]表示范围	
 + [0-9a-zA-Z\_]可以匹配一个数字、字母或者下划线
- [0-9a-zA-Z\_]可以匹配一个数字、字母或者下划线
 + (P|p)ython可以匹配'Python'或者'python'
- ^表示行的开头
- $表示行的结束
- ^py$就变成了整行匹配,就只能匹配'py'

### re模块
Python提供re模块，包含所有正则表达式的功能。
Python的字符串本身也用\转义，因此我们强烈建议使用Python的r前缀，就不用考虑转义的问题了：
```python
s = r'ABC\-001' # Python的字符串
# 对应的正则表达式字符串不变：
# 'ABC\-001'
```

match()方法判断是否匹配，如果匹配成功，返回一个Match对象，否则返回None。常见的判断方法就是：
```python
test = '用户输入的字符串'
if re.match(r'正则表达式', test):
    print('ok')
else:
    print('failed')
```

### 切分字符串
```python
>>> re.split(r'[\s\,\;]+', 'a,b;; c  d')
['a', 'b', 'c', 'd']
```

除了简单地判断是否匹配之外，正则表达式还有提取子串的强大功能。用()表示的就是要提取的分组（Group）。比如：**^(\d{3})-(\d{3,8})$**分别定义了两个组，可以直接从匹配的字符串中提取出区号和本地号码：
```python
>>> m = re.match(r'^(\d{3})-(\d{3,8})$', '010-12345')
>>> m
<_sre.SRE_Match object; span=(0, 9), match='010-12345'>
>>> m.group(0)
'010-12345'
>>> m.group(1)
'010'
>>> m.group(2)
'12345'
```
注意到group(0)永远是原始字符串，group(1)、group(2)……表示第1、2、……个子串。

### 贪婪匹配
正则匹配默认是贪婪匹配，也就是匹配尽可能多的字符。举例如下，匹配出数字后面的0：
```python
>>> re.match(r'^(\d+)(0*)$', '102300').groups()
('102300', '')
```
由于\d+采用贪婪匹配，直接把后面的0全部匹配了，结果0*只能匹配空字符串了。
必须让\d+采用非贪婪匹配（也就是尽可能少匹配），才能把后面的0匹配出来，加个?就可以让\d+采用非贪婪匹配：
```python
>>> re.match(r'^(\d+?)(0*)$', '102300').groups()
('1023', '00')
```
### 编译
如果一个正则表达式要重复使用几千次，出于效率的考虑，我们可以预编译该正则表达式，接下来重复使用时就不需要编译这个步骤了，直接匹配：
```python
>>> import re
# 编译:
>>> re_telephone = re.compile(r'^(\d{3})-(\d{3,8})$')
# 使用：
>>> re_telephone.match('010-12345').groups()
('010', '12345')
>>> re_telephone.match('010-8086').groups()
('010', '8086')
```

## 常用內建模块
### datetime
#### 获取当前日期和时间
datetime是模块，datetime模块还包含一个datetime类，通过from datetime import datetime导入的才是datetime这个类。
如果仅导入import datetime，则必须引用全名datetime.datetime。
```python
>>> from datetime import datetime
>>> now = datetime.now() # 获取当前datetime
>>> print(now)
2015-05-18 16:28:07.198690
>>> print(type(now))
<class 'datetime.datetime'>
```
datetime.now()返回当前日期和时间，其类型是datetime。

#### 获取指定日期和时间
要指定某个日期和时间，我们直接用参数构造一个datetime：
```python
>>> from datetime import datetime
>>> dt = datetime(2015, 4, 19, 12, 20) # 用指定日期时间创建datetime
>>> print(dt)
2015-04-19 12:20:00
```

#### datetime转换为timestamp
把一个datetime类型转换为timestamp只需要简单调用**timestamp()**方法：
```python
>>> from datetime import datetime
>>> dt = datetime(2015, 4, 19, 12, 20) # 用指定日期时间创建datetime
>>> dt.timestamp() # 把datetime转换为timestamp
1429417200.0
```
注意Python的timestamp是一个浮点数。如果有小数位，小数位表示毫秒数。
某些编程语言（如Java和JavaScript）的timestamp使用整数表示毫秒数，这种情况下只需要把timestamp除以1000就得到Python的浮点表示方法.

#### timestamp转换为datetime
要把timestamp转换为datetime，使用datetime提供的**fromtimestamp()**方法：
```python
>>> from datetime import datetime
>>> t = 1429417200.0
>>> print(datetime.fromtimestamp(t))
2015-04-19 12:20:00
```

#### str转换为datetime
转换方法是通过**datetime.strptime()**实现，需要一个日期和时间的格式化字符串：
```python
>>> from datetime import datetime
>>> cday = datetime.strptime('2015-6-1 18:19:59', '%Y-%m-%d %H:%M:%S')
>>> print(cday)
2015-06-01 18:19:59
```

#### datetime转换为str
如果已经有了datetime对象，要把它格式化为字符串显示给用户，就需要转换为str，转换方法是通过strftime()实现的
```python
>>> from datetime import datetime
>>> now = datetime.now()
>>> print(now.strftime('%a, %b %d %H:%M'))
Mon, May 05 16:28
```

#### datetime加减
对日期和时间进行加减实际上就是把datetime往后或往前计算，得到新的datetime。加减可以直接用+和-运算符，不过需要导入timedelta这个类
```python
>>> from datetime import datetime, timedelta
>>> now = datetime.now()
>>> now
datetime.datetime(2015, 5, 18, 16, 57, 3, 540997)
>>> now + timedelta(hours=10)
datetime.datetime(2015, 5, 19, 2, 57, 3, 540997)
>>> now - timedelta(days=1)
datetime.datetime(2015, 5, 17, 16, 57, 3, 540997)
>>> now + timedelta(days=2, hours=12)
datetime.datetime(2015, 5, 21, 4, 57, 3, 540997)
```
使用timedelta你可以很容易地算出前几天和后几天的时刻

### collections
collections是Python内建的一个集合模块，提供了许多有用的集合类。

 






最后再次感谢廖老师的辛勤劳动。






