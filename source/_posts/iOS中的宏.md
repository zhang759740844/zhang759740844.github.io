title: iOS 中的宏
date: 2017/6/30 10:07:12  
categories: iOS
tags:
	- 学习笔记
---

学一下如何在 OC 中自定义宏。参考自[官方文档](https://gcc.gnu.org/onlinedocs/cpp/Object-like-Macros.html)

<!--more-->

## 类对象宏

类对象宏是一种会在预处理时被代码片段替代的简单标识。之所以被叫做类对象宏是因为在使用的时候类似于一个对象。最常见的使用方式是用其为一个常量定义一个别名。

### 基本使用

最基本的使用方式：

```c++
// 定义
#define BUFFER_SIZE 1024

// 使用
foo = (char *) malloc (BUFFER_SIZE);
	→ foo = (char *) malloc (1024);
```

一般的，都要使用大写来命名宏。

### 换行

宏定义在 `#define` 那一行结束。如果你要书写多行，需要使用 `\` 换行。在宏展开的时候，会被自动展开为一行：

```c++
#define NUMBERS 1, \
                2, \
                3
int x[] = { NUMBERS };
	→ int x[] = { 1, 2, 3 };
```

### 顺序替换

预处理器顺序的读取程序，因此只有在定义宏之后，宏才能生效：

```c++
foo = X;
#define X 4
bar = X;

//----result----
foo = X;
bar = 4;
```

宏是一个替换过程，在顺序替换的时候，`X` 还没有被定义，所以 `foo = X`，之后宏 `X` 被定义了，所以之后的读取宏的都会被替换为具体内容。

### 嵌套宏

在拓展宏的时候，预处理器会检查宏的内容是否还是一个宏，会对其继续替换：

```c++
#define TABLESIZE BUFSIZE
#define BUFSIZE 1024
TABLESIZE
     → BUFSIZE
     → 1024
```

宏定义以最后生效的定义为准，因此下面的代码 `TABLESIZE` 对应37:

```c++
#define BUFSIZE 1020
#define TABLESIZE BUFSIZE
#undef BUFSIZE
#define BUFSIZE 37
```



## 类函数宏

宏还可以定义函数：

```c++
#define lang_init()  c_init()
lang_init()
     → c_init()
```

注意，`()` 一定要跟在名称后面，否则会被认为是一个类对象宏：

```c++
#define lang_init ()    c_init()
lang_init()
     → () c_init()()
```

预处理器会认为 `lang_init` 是一个整体，`() c_init()` 是一个整体进行替换。



## 宏参数

可以在类函数宏中传入参数：

```c++
#define min(A, B)  ((A) < (B) ? (A) : (B))

x = min(a, b);          →  x = ((a) < (b) ? (a) : (b));
```

如果宏的内容中有字符串，那么不会被宏参数替换：

```c++
#define foo(x) x, "x"
foo(bar)        → bar, "x"
```

## 三个非常重要的注意事项

宏的使用经常会产生错误，下面三点请一定注意，有助于消除大部分的 bug

### 为宏参数添加括号

比如上面的 `min` 的例子，如果我们在宏的内容中不为 `X` 和 `Y` 添加括号，会出现什么样的情况？

```c++
#define MIN(A,B) (A < B ? A : B)
```

一般情况下不会有任何问题，但是**如果 `A` 或 `B` 是一个表达式**，就会出错：

```c++
int a = MIN(3, 4 < 5 ? 4 : 5);
// => int a = (3 < 4 < 5 ? 4 : 5 ? 3 : 4 < 5 ? 4 : 5);  //希望你还记得运算符优先级
//  => int a = ((3 < (4 < 5 ? 4 : 5) ? 3 : 4) < 5 ? 4 : 5);  //为了您不太纠结，我给这个式子加上了括号
//   => int a = ((3 < 4 ? 3 : 4) < 5 ? 4 : 5)
//    => int a = (3 < 5 ? 4 : 5)
//     => int a = 4
```

所以，**一定要为宏参数添加括号，使其作为一个值被使用**。

### 若宏的结果为值，则为整个宏添加括号

还是上面 `min` 的例子，如果此时不在整个红的外面添加括号，会如何呢？

```c++
#define min(A, B)  (A) < (B) ? (A) : (B)
```

同样的，一般不会出现什么问题，但是**如果对这个值做其他的操作**：

```c++
int a = 2 * MIN(3, 4);
// => int a = 2 * 3 < 4 ? 3 : 4;
// => int a = 6 < 4 ? 3 : 4;
// => int a = 4;
```

被替换后，由于优先级的问题，`2*3` 会被率先执行。所以还上面类似的，**要为整个宏添加括号，使其作为一个值被使用**。

### 若宏替换的是代码块，则要为这段代码添加 `do{...}while(0)`

如果宏替换的是某个代码块，比如👇(不用考虑具体代码块表达什么意思)：

```c++
//A wrong version of NSLog
#define NSLog(format, ...)   ...(语句1);  \
							 ...(语句2);
```

一般使用任然不会出错，比如下面这种情况就能正确替换：

```c++
...
NSLog(@"Oops, error happened");
...
```

但是考虑**如果代码块是 `if` 中单条语句的简写形式**：

```c++
if (errorHappend)
    NSLog(@"Oops, error happened");
```

展开后可以知道这样一定编译出错：

```c++
if (errorHappend)
  	...(语句1);
	...(语句2);
```

你可以在宏中为代码块加上 `{}`。但是如果判断中有 `else` 语句，那么即使为宏套上了 `{}`，也不能通过编译：

```c++
if (errorHappend) {
	...(语句1);
	...(语句2);
}; else {
    //Yep, no error, I am happy~ :)
}
```

因此，解决方式是**在定义宏的时候为代码块套上 `do{...}while(0)`**，使整个代码块形成一个整体，相当于一条语句，并且保证了代码块只会执行一次。这样就可以符合 `if` 的简写方式了：

```c++
if (errorHappend) 
	do {
      ...(语句1);
      ...(语句2);
    } while (0);
else {
    //Yep, no error, I am really happy~ :)
}
```

> 为代码块添加 `do{...}while(0)` 是通用写法

## 字符串化

可以在宏参数前添加 `#`，将参数转换为字符串：

```c++
#define WARN_IF(EXP) \
do { if (EXP) \
        fprintf (stderr, "Warning: " #EXP "\n"); } \
while (0)
WARN_IF (x == 0);
     → do { if (x == 0)
           fprintf (stderr, "Warning: " "x == 0" "\n"); } while (0);
```

字符串化会将参数中的所有字符(包括引号)都字符串化，如果中间有很多空格，字符串化后将只有一个空格。

那如果想通过 `#` 来实现参数值的字符串化，而不仅仅是将参数名字符串化呢？这就需要需要使用两层的宏：

```c++
#define xstr(s) str(s)
#define str(s) #s
#define foo 4
str (foo)
     → "foo"
xstr (foo)
     → xstr (4)
     → str (4)
     → "4"
```

在使用 `str` 的时候，`s` 会立即被字符串化，而没有被宏展开。但是如果我们使用另外一个宏 `xstr` 嵌套着，那就会先展开，将值带入后，然后再字符串化。

## 连接

可以使用 `##` 连接两个 token：

```c++
struct command
{
  char *name;
  void (*function) (void);
};

struct command commands[] =
{
  { "quit", quit_command },
  { "help", help_command },
  …
};

#define COMMAND(NAME)  { #NAME, NAME ## _command }

struct command commands[] =
{
  COMMAND (quit),
  COMMAND (help),
  …
};
```

预处理器会将所有注释转为空格，`##` 会将左右的空格都忽略。



## 多参数宏

如果不确定有多少个宏参数，可以使用 `…` 代替，这在很多语言中都有类似的做法。相应的，使用 `__VA_ARGS__` 在具体的宏中，代替 `...`：

```c++
#define eprintf(…) fprintf (stderr, __VA_ARGS__)

eprintf ("%s:%d: ", input_file, lineno)
     →  fprintf (stderr, "%s:%d: ", input_file, lineno)
```

上面的例子中，如果宏参数缺失，那么替换后不就成 `fprintf (stderr,)` 了么？需要稍微修改：

```c++
#define eprintf(…) fprintf (stderr, ##__VA_ARGS__)
```

在加上 `##` 之后，预处理器就会在传入空的时候，删掉前面的 `,` 了。



## 条件编译

### `#if/#elif/#else`

如果 `IOS10` 值为真时，输出 `It is an iOS10 device!；` 若 `IOS10` 值为假且 `IOS9` 值为真时，输出`It is an iOS9 device!；`否则输出`It is a device NOT runing iOS10 or iOS9!`

`#if`  和 `#endif` 配对出现，`#endif` 用于终止 `#if` 预处理指令  

```c++
#if IOS10
        NSLog(@"It is an iOS10 device!");
#elif IOS9
        NSLog(@"It is an iOS9 device!");
#else
        NSLog(@"It is a device NOT runing iOS10 or iOS9!");
#endif 
```

### `#ifdef`

`#ifdef`  等价于 `#if defined`，如果后面跟的宏被定义过，则执行下面的代码。

```
#ifdef RUN
        NSLog(@"defined...");
#endif 
```

### `#ifndef`

和上面相反，如果没有被定义过，则执行下面的代码：

```
#ifndef RUN
        NSLog(@"not defined...");
#endif
```



## 内联函数

在 iOS 的一些框架中，static incline 是经常出现的关键字组合。比如：

```objc
// 直接写在文件内，不要作为类方法
static inline CGFloat CGFloatFromPixel(CGFloat value) {
    return value / YYScreenScale();
}
```

内联函数的标志就是 `incline`。内联函数直接写在文件内，不要作为类方法，使用的时候导入文件，直接调用即可。

内联函数有什么用呢？内联函数类似于宏，会把代码直接嵌入调用代码中，因此没有普通函数调用产生的性能损耗。如果该函数会被经常调用，那么就可以提高性能。它与 `#define` 宏的区别在于：

1. `#define` 定义的格式要有要求，而使用`inline`则就行平常写函数那样，只要加上`inline`即可！ 
2. 使用 `#define` 宏定义的代码，编译器不会对其进行参数有效性检查，仅仅只是对符号表进行替换
3. `#define` 宏定义的代码，其返回值不能被强制转换成可转换的适合的转换类型

在`inline`加上`static`修饰符，只是为了表明该函数只在该文件中可见！也就是说，在同一个工程中，就算在其他文件中也出现同名、同参数的函数也不会引起函数重复定义的错误！

 







## 参考文章

[官方文档](http://gcc.gnu.org/onlinedocs/cpp/Macros.html#Macros)

[iOS开发高级：使用宏定义macros](http://blog.csdn.net/songrotek/article/details/8929963#comments)

[宏定义的黑魔法 - 宏菜鸟起飞手册](https://onevcat.com/2014/01/black-magic-in-macro/)

 