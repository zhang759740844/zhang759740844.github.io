title: 《Objective-C 高级编程》读书笔记
date: 2017/5/13 10:07:12  
categories: iOS
tags:
	- 学习笔记
---

对这本书做一下读书笔记

<!--more-->

# 自动引用计数

## 内存管理/引用计数

### 内存管理的思考方式

**引用计数的思考方式**：

- 自己生成的对象，自己持有
- 非自己生成的对象，自己也能持有
- 不再需要自己持有的对象时释放
- 非自己持有的对象无法释放

**对象操作与OC 方法的对应:**

1. 生成并持有对象：`alloc/new/copy/mutableCopy`
2. 持有对象：`retain`
3. 释放对象：`release`
4. 废弃对象：`dealloc`

`NSObject` 类负担内存管理的职责。

#### 自己生成对象，自己持有

使用以下名称开头的方法意味着自己生成的对象只有自己持有：

- alloc
- new
- copy
- mutableCopy

> 这里的意思是只要**以这几个词开头**，就不需要再调用 `retain` 方法就可以保留了对象了。在 ARC 中，系统不会自动补上 retain 方法

```objc
// 自己生成并持有对象
id obj = [[NSObject alloc] init];
// 自己持有对象
```

这里就不会再调用 `[obj retain]` 了。`obj` 已经持有了对象。

#### 非自己生成的对象，自己也能持有

除了上面的方法取得的对象，因为非自己生成并持有，所以自己不是该对象的持有者。

```objc
// 取得非自己生成并持有的对象
id obj = [NSMutableArray array];
// 取得对象的存在，但自己不持有对象
[obj retain];
// 自己持有对象
```

通过 `retaion` 方法，非自己生成的对象也可以成为自己持有。

#### 不再需要自己持有的对象时释放

#####  release

自己持有的对象，一旦不再需要，持有者可以通过 `release` 方法，释放该对象。

```objc
// 取得非自己生成并持有的对象
id obj = [NSMutableArray array];
// 取得对象的存在，但自己不持有对象
[obj retain];
// 自己持有对象
[obj release];
// 释放对象，对象不可再被访问
```

##### autorelease

自定义一个符合前文命名规范的生成对象的方法：

```objc
- (id)allocObject{
  //自己生成并持有对象
  id obj = [[NSObject alloc] init];
  // 自己持有对象
  return obj；
}

// 取得非自己生成并持有的对象
id obj1 = [obj0 allocObject];
// 自己持有对象
```

由于内部是通过 `alloc` 方法生成的，所以外部调用的时候不需要在 `retain` 了。

> 由于是符合上面的命名规范，所以再 ARC 中，系统不会给 obj1 使用 retain 方法。
>
> 我想 应该是通过 `alloc` 方法，引用计数已经加过一了，而后又没有 `release` 过，最终只有 `obj1` 一个变量会指向该对象，所以就不用再 `retain` 了。

那么如果是类似 ` [NSMutableArray array]` 方式，该如何取得对象呢？以自定义一个 `object` 方法为例：

```objc
- (id)object{
  id obj = [[NSObject alloc] init];
  // 自己持有对象
  [obj autorelease];
  // 取得对象的存在，但自己不持有对象
  return obj;
}

id obj1 = [obj0 object];
// 取得对象的存在，但自己不持有
[obj1 retain]；
// 自己持有对象
```

> 在 ARC 中，如果不符合上面的命名规范，那么系统会自动添加 autorelease 方法
>
> 这里由于 autorelease 了，所以引用计数又为0了，就需要在外部再 retain 一次了

上例中，使用 `autorelease` 方法，取得对象的存在，但是自己不持有对象。`autorelease` 提供这样的功能，**使对象在超出指定的生存范围能够自动并正确的释放**(调用 release 方法)。

#### 无法释放非自己持有的对象

释放非自己持有的对象时会发生崩溃



### alloc/retain/release/dealloc 实现

苹果将对象的引用计数保存在散列表中。这样的好处是：

1. 对象用内存块的分配无须考虑内存块头部
2. 引用计数表各记录中存有内存块地址，可从各个记录追溯到各对象的内存块。

实现规则：

- 调用 `alloc` 或者 `reatain` 后，引用计数值加1
- 调用 `release` 后，引用计数减1
- 引用计数值为0后，调用 `dealloc` 方法废弃对象

### autorelease

`autorelease` 使对象超出作用域后，对象实例的 `release` 实例方法被调用。其具体使用方法如下：

1. 生成并持有 `NSAutoreleasePool` 对象
2. 调用已分配对象的 `autorelease` 实例方法
3. 废弃 `NSAutoreleasePool` 对象

对于所有掉用过 `autorelease` 实例方法的对象，在废弃 `NSAutoreleasePool` 对象时，都将调用 `release` 实例方法。

> `NSRunLoop` 会自动完成 `NSAutoreleasePool` 的生成、持有和废弃处理，不一定非要应用开发者手动使用 `NSAutoreleasePool`。不过如果存在大量 `autorelease` 的对象的话，还是建议自己生成和废弃 `NSAutoreleasePool` 的。

### autorelease 实现

`autorelease` 实例方法的本质是调用 `NSAutoreleasePool` 对象的 `addObject` 类方法。

其实就是将要释放的对象添加到 `NSAutoreleasePool` 中的数组中去。当 `NSAutoreleasePool` 将要销毁时，对数组中的所有对象调用 `release` 方法。

## ARC规则

### 所有权修饰符

ARC 有效时，对象类型上必须附加所有权修饰符：

- __strong
- __weak
- __unsafe_unretained
- __autoreleasing

#### __strong 修饰符

__strong 修饰符是默认的所有权修饰符，也就是说，下面的 id 变量，实际上被附加了所有权修饰符

```objc
id obj = [[NSObject alloc] init];
=> id __strong obj = [[NSObject alloc] init];
```

如果指定了变量的作用于：

```objc
{
  id __strong obj = [[NSObject alloc] init];
}
// 相当于下面👇
{
  id obj = [[NSObject alloc] init];
  [obj release];
}
```

`__strong` 修饰符**表示对对象的强引用**，持有强引用的对象，在超出其作用域时被废弃，随着强引用的失效，引用的对象会随之释放。

#### __weak修饰符

`__strong` 修饰符容易引起引用循环，使用 `__weak` 修饰符可以避免循环引用。

```objc
id __weak obj = [[NSObject alloc] init];
```

上面的代码存在一个问题，由于使用弱引用，新建 `NSObject` 对象在创建后会立刻释放(这也是平时使用 `weak` 时需要注意的)。需要修改成下面

```objc
id __strong obj0 = [[NSObject alloc] init];
id __weak obj1 = obj0;
```

举个例子：

```objc
id __weak obj1 = nil;
{
  id __strong obj0 = [[NSObject alloc] init];
  obj1 = obj0;
  NSLog(@"A: %@",obj1);
}
NSLog(@"B: %@",obj1);
```

由于弱引用，在括号后范围外，`obj0` 被回收，`obj1` 为置为 `nil`

#### __unsafe_unretained修饰符

已经被废弃的修饰符，这个修饰符和 `__weak` 的差别在于，`__weak` 修饰符在对象被回收后会置为 `nil`，而该修饰符则仍指向原有内存地址，访问被废弃的对象将可能会产生崩溃。

#### __autoreleasing 修饰符

> autorelease 其实就是将 release 方法延迟一段时间执行

ARC 中无法直接使用 `autorelease`。在 ARC 无效时会像下面来使用：

```objc
// ARC 无效
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
id obj = [[NSObject alloc] init];
[obj autorelease];
[pool drain];
```

ACR 有效时，将源代码写成这样:

```objc
@autoreleasepool{
  id __autoreleasing obj = [[NSObject alloc] init];
}
```

在 ARC 有效时，用 `@autoreleaseing` 块代替 `NSAutoreleasePool`类，用附有 `__autoreleasing` 修饰符的变量替代 `autorelease` 方法。

## ARC 的实现

### __strong修饰符

```objc
{
  id __strong obj = [[NSObject alloc] init];
}
编译器模拟的代码 =>
{
  id obj = objc_msgSend(NSObject, @selector(alloc));
  objc_msgSend(obj, @selector(init));
  objc_release(obj);
}
```

两次调用 `objc_msgSend` 方法，变量作用域结束时通过 `objc_release` 释放对象。虽然 ARC 有效时不能使用 `release` 方法，但是编译器会自动插入。

如果使用其他的创建方法：

```objc
{
  id __strong obj = [NSMutableArray array];
}
编译器模拟的代码 =>
{
  id obj = objc_msgSend(NSMutableArray, @selector(array));
  objc_retainAutoreleasedReturnValue(obj);
  objc_release(obj);
}
```

中间调用了 `objc_retainAutoreleasedReturnValue()` 方法。它持有的对象应为 返回注册在 `autoreleasepool` 中对象的方法，或是函数的返回值。这个方法是与 `objcautoreleaseReturnValue()` 方法成对出现的，用于优化程序运行。来看 `[NSMutableArray array]` 方法:

```objc
+ (id)array{
  return [[NSMutableArray alloc] init];
}
编译器模拟的代码 =>
+ (id)array{
  id obj = objc_msgSend(NSMutableArray, @selector(alloc));
  objc_msgSend(obj, @selector(init));
  return objc_autoreleaseReturnValue(obj);
}
```

这样做就不需要再把 `obj` 对象注册到 `NSAutoreleasePool` 中了，而是直接强引用这个对象。如下图：

![优化](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/autoreleasepool1.png.jpeg?raw=true)

### __weak

假设变量 `obj` 附加 `__strong` 修饰符且对象被赋值:

```objc
{
  id __weak obj1 = obj;
}
编译器模拟代码 =>
{
  id obj1;
  objc_initWeak(&obj1,obj);
  objc_destroyWeak(&obj1);
}
```

其中 `objc_initWeak` 函数，会将附有 `__weak` 修饰符的变量初始化为0后，调用 `objc_storeWeak` 函数。`objc_destroyWeak` 函数会将0作为参数调用 `objc_storeWeak` 函数。所以，上面等效于下面：

```objc
编译器模拟代码 =>
{
  id obj1;
  obj1 = 0;
  objc_storeWeak(&obj1, obj);
  objc_storeWeak(&obj1, 0);
}
```

`objc_storeWeak` 函数把第二参数的赋值对象的地址作为键值，将第一参数的附有`__weak` 修饰符的变量的地址注册到 weak 表中。如果第二个参数为0，则把变量的地址从 weak 表中删除。

weak 表和引用计数器表相同，作为散列表被实现。如果使用 weak 表，将废弃对象的地址作为键值进行检索，就能高速地获取对应的附有 `__weak` 修饰符的变量的地址。

假设考虑到 `obj1` 是被加入到 `autoreleasepool` 中的。那么编译器模拟代码又是什么呢？

```objc
{
  id __weak obj1 = obj;
  NSLog(@"%@",obj1);
}
编译器模拟代码 =>
{
  id obj1;
  objc_initWeak(&obj1,obj);
  id tmp = objc_loadWeakRetained(&obj1);
  objc_autorelease(tmp);
  NSLog(@"%@",tmp);
  objc_destroyWeak(&obj1);
}
```

与被赋值相比，增加了`objc_loadWeakRetained` 和 `objc_autorelease` 方法的调用：

1. `objc_loadWeakRetained` 取出了附有 `__weak` 修饰符变量所引用的对象并 `retain`
2. `objc_autorelease` 将对象注册到 `autoreleasepool` 中

> 也就是说在使用 weak 变量的代码块里，临时的产生了强引用

如果大量地使用附有 `__weak` 修饰符的变量，注册到 `autoreleasepool` 的对象会大量地增加，即会大量创建 `tmp`。所以，使用附有 `__weak` 修饰符的变量，最好先暂时给附有 `__strong` 修饰符的变量后再使用：

```objc
{
  id __weak o = obj;
  id tmp = o;
  NSLog(@"1 %@",obj1);
  NSLog(@"2 %@",obj1);
  NSLog(@"3 %@",obj1);
}
```

如果没有 `id tmp = o`，`o` 就会被注册到 `autoreleasepool` 注册3次，但是如果有这句，就只会注册一次。

### __autoreleasing

`__autoreleasing` 修饰符的变量等同于 ARC 无效时调用对象的 autorelease 方法：

```objc
@autoreleasepool{
  id __autoreleasing obj = [[NSObject alloc] init];
}
编译器模拟编码 =>
{
  id pool = objc_autoreleasePoolPush();
  id obj = objc_msgSend(NSObject, @selector(alloc));
  objc_msgSend(obj,@selector(init));
  objc_autorelease(obj);
  objc_autoreleasePoolPop(pool);
}
```

和苹果 `autorelease` 实现中的说明完全相同。



# Blocks

## Blocks 模式

### Block 语法

Block的格式:

> ^ 返回值类型 （参数列表） {表达式}

其中，返回值类型和参数列表可以省略。

###  Block 类型变量

先来看一下 C语言的函数指针：

```c
int (*funcptr)(int)
```

声明 Block 类型变量和 C语言几乎一致，除了将 `*` 改为 `^`，示例如下：

```objc
int (^blk)(int)
```

Block 可以作为右值，返回类型，参数类型。可以使用 `typedef` 定义：

```objc
typedef int (^blk_t)(int);
=>
blk_t blk;
```

这样，`blk_t` 就**从变量名，变成了变量类型**。

### 截获自动变量值

Block 表达式截获所使用的自动变量的值，即保存该自动变量的瞬间值。因为 Block 表达式保存了自动变量的值，所以在执行 Block 语法后，即使改写 Block 中使用的自动变量的值也不会影响 Block 执行时自动变量的值。这就是自动变量值的截获。

### __block说明符

自动变量值截获只能保存执行 Block 语法瞬间的值。保存后就不能改写该值。如果尝试改写截获的自动变量值，会产生编译错误。

若想在 Block 语法的表达式中将值赋给在 Block 语法外声明的自动变量，需要在该自动变量上附加 `__block` 说明符。

对于截获的 OC 对象，我们可以修改对象的内容，但是不能修改对象的地址指向，否则还是需要使用 `__block`。

## Blocks 的实现

### Block的实质

clang(LLVM 编译器)通过 “-rewrite-objc” 选项可以将含有 Block 语法的源代码变换为 C语言源代码：

```objc
clang -rewrite-objc 源代码文件名
```

比如一个最简单的 Block ：

```objc
int main(){
  void (^blk)(void) = ^{printf("Block\n");};
  blk();
  return 0;
}
```

其中，`^{printf("Block\n"};` 可以转化为：

```objc
static void __main_block_func_0(struct __main_block_impl_0 *__cself){
  printf("Block\n");
}
```

通过 Blocks 使用的匿名函数实际上被作为简单的 C语言函数来处理。另外，根据 Block 语法所属的函数名和该 Block 语法在该函数出现的顺序值(此处为0)来给经 clang 变换的函数命名。

该函数的参数 `__cself` 相当于 OC 中的 `self`，是 `__main_block_impl_0` 结构体的指针：

```objc
struct __main_block_impl_0{
  struct __block_impl impl;
  struct __main_block_desc_0 *Desc;
}
//其中：
struct __block_impl{
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
}  
struct __main_block_desc_0{
  unsigned long reserved;
  unsigned long Block_size;
}
```

来看一下该实例的构造方法：

```objc
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flag=0){
  impl.isa = &_NSConcreteStackBlock;
  impl.Flags = flags;
  impl.FuncPtr = fp;
  Desc =desc;
}
```

第一个参数是由 Block 语法转换的 C 语言函数指针，本例中将会传入上面的 `__main_block_func_0`，第二个参数是作为静态全局变量初始化的 `__main_block_desc_0` 结构体实例指针，该指针内保存了 `__main_block_impl_0` 结构体实例的大小。

观察上面的构造方法 `impl.isa = &_NSConcreteStackBlock;` ，其 `isa` 指针指向了 `&NSConcreteStackBlock`，说明 **Block 的实质是 Objective-C 的对象**。

通过上面的构造方法，我们看 `void (^blk)(void) = ^{printf("Block\n");};` 的转换后的代码：

```objc
struct __main_block_impl_0 tmp = (void (*)(void))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA);
void (*blk)(void) = &tmp;
```

该源码将 **栈上（因为这里的所有结构体并没有 malloc 内存，所以都是保存在栈上的）**生成的 `__main_block_impl_0` 结构体实例的指针，赋值给 `__main_block_impl_0` 结构体指针类型的变量 `blk`。

继续查看 `blk()` 的转换后的代码：

```objc
((void (*)(struct __block_impl *))(struct __block_impl *)blk)->FuncPtr)((struct __block_impl *)blk);
简化后 =>
(*blk->impl.FuncPtr)(blk);
```

就是调用了 `blk` 中保存的函数指针，并将 `blk` 作为 `__cself` 传入。

### 截获自动变量值

前面的 Block 中没有用到任何自动变量，所以`__main_block_impl_0` 中只有两个成员：`__block_impl` 和 `__main_block_desc_0`。如果 Block 中用到了自动变量，将会添加相应参数，如在 Block 中用到了 `char *fmt` 和 `int val` ，那么结构体，构造函数等都要相应添加：

```objc
struct __main_block_impl_0{
  struct __block_impl impl;
  struct __main_block_desc_0 *Desc;
  const char *fmt;
  int val;
}
```

所谓“截获自动变量值”，意味着在执行 Block 语法所使用的自动变量值被保存到 Block 的结构体实例（即 Block 自身）中

### __block 说明符

上面说到 Block 会截取自动变量值，并且 Block 内部不能修改自动变量值，否则会抛出异常。如果想要在 Block 中修改外部变量的值，需要在变量上添加 `__block` 说明符，加上这个说明符后，这就不再是个简单的变量了，它为自身创建了一个结构体。比如一个 `int` 类型的变量 `val`,它变成了一个结构体实例 `__Block_byref_val_0`:

```objc
struct __Block_byref_val_0{
  void *__isa;
  __Block_byref_val_0 *__forwarding;
  int __flags;
  int __size;
  int val;
};
```

其中 `__forwarding` 持有指向自身的指针，`val` 保存了 `int` 的值。

```objc
__block int val = 10;
转换后=>
__Block_byref_val_0 val = {
  0,
  &val,
  0,
  sizeof(__Block_byref_val_0),
  10
};
```

比如一个给 `__block` 变量赋值的代码将会转换成下面：

```objc
^(val = 1;)
=>
static void __main_block_func_0(struct __main_block_impl_0 *__cself){
  __Block_byref_val_0 *val = __cself->val;
  (val->__forwarding->val) = 1;
}
```

### Block 存储域

Block 转换为 Block 结构体类型的自动变量，`__block` 变量转换为 `__block` 变量的结构体类型的自动变量。**所谓结构体类型的自动变量，即栈上生成的该结构体的实例。**

Block 也是 OC 对象。将 Block 当做 OC 对象来看时，该 Block 类为 `_NSConcreteStackBlock`，类似的还有：

- _NSConcreteStackBlock
- _NSConcreteGlobalBlock 
- _NSConcreteMallocBlock

设置在栈上的 Block，如果其所属的变量作用域结束，该 Block 就被废弃了。由于 `__block` 变量也配置在栈上，同样的，如果其所属的变量作用域结束，则该 `__block` 变量也会被废弃。Blocks 提供了将 Block 和 `__block` 从栈上复制到堆上的方法来解决这个问题。复制到堆上的 Block 将 `_NSConcreteMallocBlock` 类对象写入 Block 的结构体实例的成员变量 isa：

```objc
impl.isa = &_NSConcreteMallocBlock;
```

通过 `copy` 方法，将 Block 从栈上复制到堆上：

```objc
[^{NSLog(@"haha")} copy];
```

### __block 变量存储域

前面说到 `__block` 产生的结构体 `__Block_byref_val_0` 中会有一个 `forwarding` 的指针，一般指向自己。那么这个指针存在的意义是什么呢？

对于一个 `__block` 和 Block 变量，原本是保存在栈上的。当 Block 被复制到堆上后，其中使用到的 `__block` 变量也会从栈中复制到堆上，并被 Block 持有。如下图：

![复制](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/MallocBlock.JPG?raw=true)

现在，需要在 Block 内外修改 `__block` 的值，即 `^{++val;}` 和 `++val;` 都能够修改值，这就体现了 `forwarding` 这个指针的目的了：



![复制](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/forwardingBlock.jpg?raw=true)

在变换 Block 语法的函数中，该变量 val 为复制到堆上的 `__block` 变量用结构体实例，而使用的与 Block 无关的变量 val，为复制前栈上的 `__block` 变量用结构体实例。

栈上的 `__block` 变量用结构体实例在 `__block` 变量从栈复制到堆上时，会将成员变量 `__forwarding` 的值替换为复制目标堆上的 `__block` 变量用结构体实例的地址。通过该功能，无论在 Block 语法中、Block 语法外使用 `__block` 变量，还是 `__block` 变量配置在栈上还是堆上，都可以顺利地访问同一个 `__block` 变量。

### 截获对象

比较下面两段代码中， Block 块中的打印日志：

```objc
// 代码段一
blk_t blk;
{
  id array = [[NSMutableArray alloc] init];
  blk = [^(id obj){
    [array addObject:obj];
    NSLog(@"array count = %ld",[array count]);
  } copy];
}
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);

// 代码段二
blk_t blk;
{
  id array = [[NSMutableArray alloc] init];
  blk = ^(id obj){
    [array addObject:obj];
    NSLog(@"array count = %ld",[array count]);
  };
}
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
```

输出：

```objc
// 代码段一
array count = 1
array count = 1
array count = 1
  
// 代码段二
强制结束
```

因为下面的代码没有 copy 到堆上，所以 array 也就仍然存储在栈上，即使 Block 截获了对象，它也会随着变量作用域的结束而被废弃。

所以除了下面的情况，推荐调用 Block 的 copy 方法：

- Block 作为函数返回值返回时
- 将 Block 赋值给类的附有 `__strong` 修饰符的 `id` 类型或 Block 类型成员变量时
- 向方法命中含有 `usingBlock` 的 Cocoa 框架方法或 GCD 的 API 中传递 Block 时

### __block 变量和对象

上面的例子如果改成下面将会输出什么结果呢：

```objc
// 代码段一
blk_t blk;
{
  id array = [[NSMutableArray alloc] init];
  id __weak array2 = array;
  blk = [^(id obj){
    [array2 addObject:obj];
    NSLog(@"array2 count = %ld",[array2 count]);
  } copy];
}
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);

// 代码段二
blk_t blk;
{
  id array = [[NSMutableArray alloc] init];
  __block id __weak array2 = array;
  blk = [^(id obj){
    [array2 addObject:obj];
    NSLog(@"array2 count = %ld",[array2 count]);
  } copy];
}
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
blk([[NSObject alloc] init]);
```

输出：

```objc
// 代码段一
array2 count = 0
array2 count = 0
array2 count = 0
// 代码段二
array2 count = 0
array2 count = 0
array2 count = 0
```

虽然都用了 copy 方法，但是代码段一由于弱引用了 `array`，导致 `array` 在作用域外被回收，`array2` 被置为 `nil`。同样的，即使增加了 `__block` 修饰符，也只是改变 `array2` 的结构，引用计数该回收的时候还是回收了，所以对结果没有任何影响。

### Block 循环引用

Block 有可能会引起循环引用是被熟知的了，一般是使用 `__weak` 说明符的方式打破引用循环，如：

```objc
- (id)init{
  self = [super init];
  id __weak tmp = self;
  blk = ^{NSLog(@"self = %@", tmp);};
  return self;
}
```

除了使用 `__weak` 还可以使用 `__block`，在 Block 内使用完后，手动将变量置为 `nil`:

```objc
- (id)init{
  self = [super init];
  __block id tmp = self;
  blk = ^{
    NSLog(@"self = %@", tmp);
    tmp = nil;
  };
  return self;
}
```

使用上面方法的优点是可以手动控制对象的持有时间，而缺点是必须要执行 Block，否则会产生引用循环。

# GCD

API 的使用在我另外一篇 GCD 的总结中有详细的记述[GCD队列](https://zhang759740844.github.io/2016/08/02/GCD队列/)，不详细描述了。





























 