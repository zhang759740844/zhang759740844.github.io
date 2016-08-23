title: python 3.5 面向对象编程
date: 2016/8/9 10:07:12  
categories: python
tags:
	- 读书笔记

---


当时学的是2.x版本，有点过时了，现在看了[廖雪峰的博客](http://www.liaoxuefeng.com/)的3.5版本的语法介绍，做一些摘录。

<!--more-->


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

最后再次感谢廖老师的辛勤劳动。