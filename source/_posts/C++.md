title: C++简略语法
date: 2020/1/23 11:07:12
categories: C++
tags:

- 学习笔记

------

假期无事，重学C++，简单摘录了一些 C++ 的基本语法，具体还需要再看 C++ Primer。

<!--more-->

### .cpp 的由来

.cpp 意为 c plus plus

### 用 C 语言的方式去编译
使用场景是如果你要使用别人用 c 写的第三方库的时候，由于 C++ 的函数编译方式和 C 不同。C++ 会默认按照 C++ 的方式去查找函数，这样是无法找到 C 的函数实现的，所以必须先声明该函数是 C 的函数
```c++
// .cpp 中的声明
extern "C" void test(int a);
extern "C" {
	// 引用 c 头文件
	#include "math.h"
}

// .c 中的实现
void test(int a){
    cout << "single";
}
```

> C++ 由于支持函数重载，会在编译函数的时候把参数类型也添加为函数标识中，而 C 不支持重载，不关心函数参数。因此 C++ 中直接调用 C 的函数是找不到的  

### .h 文件中 #ifndef #define #endif的作用
对于 OC 来说没有多大的作用，OC 编译器不会重复引入同一个头文件。但是对于 C 文件来说，通过多次 `#include` 会重复包含声明，因此需要通过 `#ifndef` `#define` 和 `#endif` 避免重复声明。

> 宏 `#pragma once` 也可以达到相同的效果（较新的特性）  

### 引用
引用就相当于变量的别名，引用**不可以修改指向**：
```c++
int age = 10;
int &rage = age; 

int age2 = 20;
// 相当于给 age 赋值，而不是将 rage 指向 age2
rage = age2   // age = 20 
&rage = age2  // ❌
```

> 引用相当于别名，可以理解为之后用到引用的部分都替换为原变量  

### Class 和 Struct 区别
C++ 中 Struct 也能定义函数，和 Class 类似。

C++ 中 Struct 和 Class 的唯一区别在于： Class 默认成员权限是 private，而Struct 默认成员权限是 public

### C++ 函数调用和 OC 的区别
我们知道 C++ 函数调用时直接调用，即在编译的时候对于函数的调用直接转为 call 函数地址。因此， 创建的 C++ 类的实例在内存中只有成员变量。

而 OC 函数调用基于消息，所以需要有一个 isa 指针指向 meta class，以此查找调用的方法。因此 OC 类的实例除了成员变量外，还需要一个 isa 指针。

### C++ 中的 `.` 与 `->`
点语法的左边只能是对象，如果是指针，必须要用 `->`：
```c++
Person person;
person.age = 3;	// √
Person *person2 = &person;
person2.age = 5;	// ×
person2->age = 5;	// √
```

### 构造和析构回收
在创建对象的时候构造函数会自动调用。构造函数和类同名，可以有参数，可以重载。

回收对象通过 `delete 对象` 来进行，调用之后对象回收(不是引用计数减一，是直接回收)。

回收时调用析构函数，析构函数不可以重载。析构函数中要释放内部的对象：
```c++
struct Car {};
struct Person {
	int age;
	Car *car;
	// 构造函数的相互调用
	Person() :Person(12) {
	  cout << "Person build" << endl;
	}
	Person(int age) {
	  this->age = age;
	  this->car = new Car();
	  cont << "has age is" << age << endl;
	}
	~Person() {
	  cout << "destructor" << endl;
	  delete this->car;
	}
};

int main() {
	Person *person = new Person(23);
	delete person;
	return 0;
}
```
> 注意这里的构造函数的相互调用的方式。由于 C 不像 OC，Java 这种构造函数返回自身，而是返回 void，因此，不能在方法中主动调用其他构造函数，而是需要使用如上的语法糖告知编译器  

#### C 语言中的创建与回收
通过 malloc 分配的对象不会调用构造函数和析构函数：
```c
Person *person = (Person *)malloc(sizeof(Person));
person->age = 12;
free(person);
```

### 声明实现分离
C++ 中声明文件和实现文件分离：
```c++
// .h 中
#ifndef Person_hpp
#define Person_hpp

#include <stdio.h>

class Car {};

class Person {
public:
    int age;
    Car *car;
	  void run();
    Person(int age);
    ~Person();
};

#endif /* Person_hpp */
```
声明之后的实现方式如下：
```C++
// .cpp 中
#include "Person.h"
#include <iostream>

Person::Person(int age) {
    this->age = age;
    this->car = new Car();
}
Person::~Person() {
    std::cout << “dealloc” << std::endl;
    delete this->car;
}

void Person::run() {
	cout << "run" << endl;
}
```

> 编译的时候只会编译 cpp 文件，头文件中放入的声明，在编译的时候会被直接写入 cpp 中。声明对应的具体实现会在链接过程中替换为具体的地址。  
>   
> 因此，我们可以得出一个结论，在 .h 中不能直接写函数实现是因为会被多个文件导入，从而导致 duplicate symbol。而 .h 中定义的 class 中的方法可以写函数实现是因为 class 包裹的关系，不会因为多次导入而变成两份实现。  

### 命名空间
使用 `namespace` 关键字包裹。**同名的命名空间会被合并**：
```c++
namespace ZAC {
	class Car {};
}
namespace ZAC {
	class Bike {};
}
```
使用：
```c++
// 直接写明命名空间
ZAC::Car *car = new ZAC::Car();
// 省略命名空间,但是这样会导致全部的命名空间默认都是 ZAC
using namespace ZAC;
int main() {
	Car *car = new Car();
	return 0;
}
```

> 对于**声明实现分离**的类使用命名空间，声明和实现都要包括在 namespace 中  

### 继承
C++ 继承和 OC 继承类似:
```c++
struct Person {};
struct Student : Person {};
```

### 多态
C++ 的函数调用方式是编译时根据指针类型，直接调用函数地址。要实现多态，需要借助虚函数。如果有多态，需要把其析构函数也声明为虚析构函数（如果不设置为虚析构函数，那么就会调用父类的析构函数而不是真正的析构函数）：
```C++
struct Person {
	virtual void walk() {
	  cout << "person walk" << endl;
	}
	virtual ~Person() {}
};

struct Student {
	void walk() {
	  count << "student walk" << endl;
	}
	~Student() {}
};
```

> 使用虚函数后，对象的前4个字节会存一个虚函数表的地址，其他属性的地址会依次向后移动4个字节。  

#### 纯虚函数
没有函数体，且函数初始化为0的叫做纯虚函数。含有纯虚函数的类叫做抽象类，不可以被实例化，只能用来定义接口规范：
```c++
struct Person {
	virtual void walk() = 0;
};
```

### 虚继承
C++ 是多继承的，这样就造成了在菱形继承情况下会同时继承多个基类的同名属性的情况。虚继承则是告知编译器，共用同一个属性：
```c++
struct Person {
	int age;
}

struct Student : virtual Person {
	int score;
};

struct Worker : virtual Person {
	int salary;
};

struct Me : Student, Worker {
	int name;
};
```

### 静态成员变量
C++ 中访问静态成员变量的方式和其他语言不同。它的初始化方式也和其他语言不同：
```c++
class Person {
public:
    static int age;
};
// 在类外部初始化。一般放在 .cpp 中
int Person::age = 12;

int main(int argc, const char * argv[]) {
    Person::age = 12;
    return 0;
}
```

> OC 中由于有 meta class 的存在，静态成员变量和静态方法是存在 meta class 中的。
>
> 而 C++ 中的静态成员变量不同，静态成员变量就相当于是个限制了作用域的**全局变量**。  

### const
C++ 中的 const 有两种场景：修饰值时不能改变对象的属性，修饰指针时不能改变指针的指向。

C++ 中的方法也可以是 const 的，表示该方法不会修改对象的成员变量。在 const 修饰对象的时候，不能调用非 const 的方法：
```c++
class Person {
public:
	  // 这里必须要有 const
    void run() const {
        cout << "run" << endl;
    }
};

int main(int argc, const char * argv[]) {
    const Person *p = new Person();
	  // const 修饰对象，那么一定只能调用 const 的方法
    p->run();
    return 0;
}
```

#### const 的位置标示的结果的不同
这是一个老问题，const 位置的不同所代表的意义也不同：
```c++
// 指向的值不能改变，对象的话不能改变属性
Person const *person = new Person();
// 指针指向不能改变
Person * const person = new Person();
```
> const 永远修饰它的👉，右边是 `*person` 就表示指针指向的值不能变，右边是 `person` 就表示指针不能改变指向。  

### 模板
#### 函数模板
C++ 模板写法如下：
```c++
// 写在 .h 文件中
template <typename T> T add(T a, T b) {
	return a + b;
}

add<double>(1.5, 2);
```

> 编译的时候根据类型生成不同的函数  

C++ 模板在声明实现分离的时候不能在 .cpp 中实现而要直接在 .h 中实现。这是因为模板实现的 .cpp 文件在单独编译的时候不知道被传入了哪些类型，因此无法创建相应类型的函数实现。而写在 .h 中直接被引入其他 .cpp 中后，在编译的时候就能知道其他 .cpp 文件传入了什么类型，进而创建相应的函数。

#### 类模板
类模板写起来非常麻烦，设置了类模板之后，每个方法的实现都要声明一遍模板，否则报错：
```c++
// .h 中
template <class Food>
class Person {
public:
    void eat(Food *a);
    Person();
    Person(int age);
    ~Person();
};

template <class Food>
void Person<Food>::eat(Food *a) {
    cout << "food" << endl;
}

// .cpp 中
template <class Food>
Person<Food>::Person() {
}
template <class Food>
Person<Food>::Person(int age) {
}
template <class Food>
Person<Food>::~Person() {
    std::cout << "dealloc" << std::endl;
}
```

> 类模板的实现声明分离同样还是写在头文件中  

### 自动类型推断
和 dart 类似，可以不用写明变量的类型，通过 `auto` 声明让编译器自行推断：
```c++
auto a = 10;
```

### lamda 表达式
lamda 表达式的一般形式如下，`[]` 表示的是捕获的外部的变量，`()` 内是参数。这种捕获方式是**值捕获**：
```c++
int a = 10;
int (*p)(int) = [a](int param) -> int {
	return param * a;
}
```

如果要捕获的变量会根据外部的修改而变化，并且内部的变化也会影响外部的值，那么就需要获取变量的地址。这种捕获方式是**引用捕获**：
```c++
int a = 10;
int (*p)(int) = [&a](int param) -> int {
	return param * a;
}
```
> 和 OC 中的 `__block` 类似，都是可以内部修改外部的值以及外部可以修改内部的值。只不过 C++ 中是通过指针传递的方式，而 OC 中则是通过生成一个包裹对象的形式。
>
> 另外合 OC 类似的，全局变量不用捕获。  








