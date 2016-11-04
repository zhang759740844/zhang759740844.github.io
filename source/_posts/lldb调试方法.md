title: lldb 调试方法
date: 2016/9/21 14:07:12  
categories: iOS
tags:
	- Debug

---

偶然看到使用 lldb 里的 watchpoint 调试，感觉很神奇，就详细了解了下 lldb 调试的使用方法。 查阅了许多文章[与调试器共舞 - LLDB 的华尔兹](https://objccn.io/issue-19-2/)这篇文章帮助最大。

<!--more-->
Xcode中内嵌了LLDB控制台，在Xcode中代码的下方，我们可以看到LLDB控制台。

LLDB控制台平时会输出一些log信息(Debugger output中)。如果我们想输入命令调试，必须让程序进入暂停状态。让程序进入暂停状态的方式主要有2种：
1. 断点或者watchpoint: 在代码中设置一个断点（watchpoint），当程序运行到断点位置的时候，会进入stop状态
2. 直接暂停，控制台上方有一个暂停按钮，上图红框已标出，点击即可暂停程序

不过最好还是不要直接暂停，因为这种情况下的当前作用域没有任何变量，即使进入了 lldb 也做不了什么事。


## expression
expression的几个命令是最常用的，能提升debug效率的命令。

### print
打印一个对象:
![print](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_1.png?raw=true)
可以直接缩写`p`达到同样的效果。

结果中有个`$0`，可以使用它指向这个结果。lldb 中可以创建对象，但是这个对象的命名必须要以`$`开头,表示命名空间是lldb，以免和外部代码中的名字冲突，如：
![print2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_2.png?raw=true)

### expression
#### 使用
如果想要改变一个值，或者执行一个方法。可以使用`expression`或者`e`：
![expression](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_3.png?raw=true)

print其实是特殊的expression，比如`p count =18`其结果和`expression count = 18`结果一样。事实上，`print`是 `expression --` 的缩写。

#### 应用
由于`expression`可以改变值，我们可以动态的改变View的颜色，而不需要重新编译：
```objc
// 改变颜色
(lldb) expression self.view.backgroundColor = [UIColor redColor]
// 刷新界面
(lldb) expression [CATransaction flush]
```
不只是改变颜色，frame，animation等都能改变，非常有用。

如最开始所说，不能直接暂停，必须要在当前界面中触发断点，比如重写touch方法添加断点等。

### po
`print`输出比较啰嗦，而且当尝试打印复杂对象的时候，有时候会很蛋疼：
```objc
(lldb) p @[ @"foo", @"bar" ]

(NSArray *) $8 = 0x00007fdb9b71b3e0 @"2 objects" 
```

这个时候可以使用`po`(print object 的缩写)，相当于调用对象的`description`方法。
```objc
(lldb) po $8
<__NSArrayI 0x7fdb9b71b3e0>(
foo,
bar
)
(lldb) po @"lunar"
lunar
```

## thread
### bt
可以打栈中的所有帧(frame)。这些frame和左边红框里的堆栈是一致的。平时我们看到的左边的堆栈就是frame:
![frame](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_4.png?raw=true)

### thread return
这也是比较有用的一个命令。Debug的时候，也许会因为各种原因，我们不想让代码执行某个方法，或者要直接返回一个想要的值。这时候就该thread return上场了。

它有一个可选参数，在执行时它会把可选参数加载进返回寄存器里，然后**立刻执行返回命令，跳出当前栈帧**。这意味这函数剩余的部分**不会被执行**。这会给 ARC 的引用计数造成一些问题，或者会使函数内的清理部分失效。但是**在函数的开头执行这个命令**，是个非常好的隔离这个函数，伪造返回值的方式 。

比如下图所示：
![return](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_5.png?raw=true)
正常通过`isOdd`方法，一定是`return NO`的，但是通过`(lldb) thread return YES`，跳过了`isOdd`方法里的方法块，直接`return YES`

正如上面所述，必须要在断点进行到如图所示的地方才可以使用`thread return`，因为:
1. 此时当前栈帧是`isOdd`
2. 不会造成引用问题

## breakpoint
断点当然是调试必不可少的东西，下面不会说明怎样添加断点，而是将列举一些断点的使用技巧.

### 流程控制
当插入断点后，程序到达断点时会就会停止运行。调试条上会出现如下四个按钮:
![debug](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_6.png?raw=true)
意义都很明确，不多说了，列举下对应的命令：
1. continue按钮 => `continue`,缩写`c`
2. step over按钮 => `next`，缩写`n`
3. step in按钮 => `step`,缩写`s`
4. step out按钮 => `finish`

### 设置断点
在断点处右击，选择`Edit Breakpoint`弹出如下设置框：
![breakpoint](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_7.png?raw=true)

#### 蓝色的勾
表示enable和disable断点。
#### Condition
指的是条件表达式，该项允许我们对断点生效设置条件，表示当满足某一特定条件的前提下，该断点才生效。（该条件的录入，**不能够识别预处理的宏定义，也不能识别断点作用域之外的变量和方法**）。
#### Ignore
忽略次数。它指定了在断点生效，应用暂停之前，代码忽略断点的次数。你如果希望应用运行一段时间后断点才生效，那么就可以使用这个选项。比如说在调试某一循环体的时候。
#### Action
它表示当断点生效时，Xcode作出反应后的行为动作。点击右边的Add Action选项会弹出如下菜单：
![menu](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_8.jpg?raw=true)
默认的是Debugger Command。还有以下几种动作供选择，下面将介绍两个常用的：
1. **Debugger Command**：默认的选项，可以让断点执行LLDB调试命令。
2. **Log Message**：可以生成消息队列，将相关的消息输出到控制台上.随便写什么都能输出，输出的对象需要使用`@exp@`形式。

#### Options
一个选项，勾上可以在运行到断点后不暂停，自动向下运行，配合Action相当于打了断点。
### 断点类型
#### 异常断点
异常断点是代码出现问题导致编译器抛出异常时触发的断点。它在断点导航器中设置。点击＋号，选择Exception Breakpoint选项。如下图：
![exception](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_9.png?raw=true)

#### 符号断点
他可以中断某个方法的调用，可谓是异常强大，在断点导航器界面，点击＋号，选择Add Symbolic Breakpoint选项，然后会弹出如图所示的对话框:
![exception](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_10.png?raw=true)

它比普通断点的自定义设置界面多出了两个内容
- **Symbol**：用来设置当前断点作用域所能识别的方法，这里面既可以是自定义的方法，也可以是系统的API方法。（注意必须表明是类方法还是成员方法）例如:
	```objc
	-[MyViewController viewDidAppear:]
	+[MyViewController sharedInstance]
	```
- **Module**：模组的意思，用来限制满足符号的方法，编译器将只会在断点满足这个模组的符号的时候才回暂停。

需要注意的是，如果一个子类没有重写父类的方法，那么设置子类这个方法的符号断点是不会被触发的。

### watchpoint
有时候，由于不涉及到方法，无法使用断点，但是我们仍要监视某一个值是否变化，这个时候就可以用`watchpoin`t来监视一个指针的指向。

当指针指向变化时，`watchpoint`会触发:
![watchpoint](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_11.png?raw=true)
#### set
添加watchpoint的方式如上图所示
```objc
(lldb) watchpoint set variable xxx
```
这里要说明一下，watchpoint要监听的对象必须是**当前类的对象**：
![watchpoint2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_12.png?raw=true)
由于`self.view`是从`UIViewController`中继承过来的，因此无法监听。

另外，上面的`addr = 0x1556747f0`**不是指`_myView`在栈中的索引**，而是指这个Watchpoint在堆中的地址。

Watchpoint的**原理**应该是在这个地址中写入了原来`_myView`指向的地址，然后对新`_myView`指向的地址和Watchpoint指向的地址是否相同来做出判断的。

#### disable/delete/enable
watchpoint资源也是比较有限的，对于不需要监听的对象要及时释放。每个watchpoint都有一个序号，操作对应的需要即可：
![watchpoint3](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/lldb_13.png?raw=true)

## 其他技巧
### run
这是我比较喜欢的方法，所以单独列出来。

调试的时候经常需要重新启动程序。但是如果重新Run程序，需要重新编译，非常浪费时间。可以在 lldb 中输入`run`就能直接让程序重新加载了。

### 通过地址操作
比如expression,print都可以通过地址来访问对象，但是实验下来，感觉没有太大用处。

本来想要监控`self.view.backgroundColor`的改变情况，但是不能通过变量名，就尝试能否通过地址，结果发现也不行。

因此，可以大体上认为，能通过地址访问的，那也能通过变量名访问，并且通过变量名访问更清晰迅速，通过地址访问没有优势，故不推荐使用。

### 代替NSLog
看`NSLog`的文档，第一句话就说：`Logs an error message to the Apple System Log facility.`，所以首先，`NSLog`就不是设计作为普通的`debug log`的，而是`error log`；每次`NSLog`都会进行很多耗时工作，因此，非重要参数尽量不要用，使用调试代替。


