title: iOS 中的自动引用计数
date: 2017/5/13 10:07:12  
categories: iOS
tags:

- 学习笔记

---

对这本书做一下《oc 高级编程》的读书笔记

<!--more-->

## 内存管理/引用计数

### 内存管理的思考方式

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

通过 `retain` 方法，非自己生成的对象也可以成为自己持有。

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

由于外部是通过 `allocObject` 生成的，所以内部不用 `[obj autorelease]`

上面的例子把 `allocObject` 中不用 `alloc` 创建，这个时候就会添加 `retain` 了：

```objc
- (id)allocObject{
  //取得非自己生成并持有的对象
  id obj = [NSArray array];
  //取得对象的存在，但自己不持有对象
  [obj retain];
  // 自己持有对象
  return obj；
}

// 取得非自己生成并持有的对象
id obj1 = [obj0 allocObject];
// 自己持有对象
```

由于外部是符合命名规范的，因此，内部不添加 `autorelease` 方法。

> 其实可以这么理解，方法内如果是用 `alloc` 创建对象并返回的，那么不用 `retain`；如果不用 `alloc` 创建对象的，会自动插入 `retain`，所以不管怎样引用计数器必然加一。这个时候，由于方法是带有 `alloc` 等字眼的，编译器不会在方法中的最后添加 `release` 方法。因此，外部就不用再 retain 了，因为内部已经加过一了。

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

> 在 ARC 中，如果不符合上面的命名规范，那么系统会自动添加 autorelease 方法，并且外部就需要再 retain 一次了。
>
> 如此：符合命名规范，既不要内部 release，也不要外部 retain；不符合命名规范，既要内部 release 一次，也要外部 retain 一次。这样成对的操作，才保证了引用计数的正确性。
>
> 那不符合命名规范的时候先 release 在 retain 不是很浪费性能么？其实在这种情况下，oc 做了优化，不会把对象注册到 AutoreleasePool 中，实现方法是使用`objc_retainAutoreleasedReturnValue()` 和 `objc_autoreleaseReturnValue()`，具体在下方。

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

## ARC 中的所有权修饰符

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

如果指定了变量的作用域：

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

中间调用了 `objc_retainAutoreleasedReturnValue()` 方法。它持有的对象应为 返回注册在 `autoreleasepool` 中对象的方法，或是函数的返回值。这个方法是与 `objc_autoreleaseReturnValue()` 方法成对出现的，用于优化程序运行。来看 `[NSMutableArray array]` 方法:

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
  id __weak obj1 = obj;
  id tmp = obj1;
  NSLog(@"1 %@",obj1);
  NSLog(@"2 %@",obj1);
  NSLog(@"3 %@",obj1);
}
```

如果没有 `id tmp = obj1`，`obj1` 就会被注册到 `autoreleasepool` 注册3次，但是如果有这句，就只会注册一次。

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

## oc 对象指针与 c 指针的转换

### `__bridge` `__bridge_transfer` 和 `__bridge_retained` 的区别

我们在将将 c 指针和 oc 对象指针之间做转换的时候会用到上述几个修饰符。**它们都会将 c 指针转为 oc 对象指针**。差别在于：

- `__bridge`：ARC 不会插入 `retain` 和 `release` ，即生命周期和 c 指针一致
- `__bridge_retained` ： ARC 会插入一条 `retain`，不会插入 `release`
- `__bridge_transfer`：ARC 会插入一条 `release`，不会插入 `retain`

### 解决 NSInvocation `getArgument` 引发的 Double Release

从 NSInvocation 中获取参数会这样取：

```objc
id arg;
[invocation getArgument:&arg atIndex:i];
```

一般情况下赋值操作会成对的插入 `retain` 和 `release`：

```objc
- (void)method {
id arg = [SomeClass getSomething];
// [arg retain]
...
// [arg release]  退出作用域前release
}
```

但是 ARC 下由于 arg 不是赋值操作，因此没有加入 `[arg retain]`。但是在结尾的时候还是调用了  `[arg release]` 就会造成 crash：

```objc
id arg;
[invocation getArgument:&arg atIndex:i];
// [arg release];
```

我们有两种方式解决这个问题，一种是通过 `__unsafe_unretained` 修饰符，告诉编译器不要插入 `[arg release]` 和 `[arg retain]`

```objc
__unsafe_unretained id arg;
[invocation getReturnValue:&arg];
```

还有一种方式就是通过上面提到的 `__bridge`，同样告诉编译器不要插入  `[arg release]` 和 `[arg retain]`

```objc
id returnValue;
void *result;
[invocation getReturnValue:&result];
returnValue = (__bridge id)result;
```

> 这种操作指针的还是通过  `__bridge` 比较好

### 解决 NSInvocation 创建对象后 `getReturnValue` 引发的内存泄漏

前面说过，当方法名开头是 alloc / new / copy / mutableCopy 时，返回的对象是 retainCount = 1 的。因此，需要在作用于结束的时候添加 `release` ，释放引用计数。但是通过 `__bridge` 将 c 对象转为 oc 对象的时候会省略 `release` 。因此，要使用 `__bridge_transfer`，仍然插入 `release`：

```objc
id returnValue;
void *result;
[invocation getReturnValue:&result];
if ([selectorName isEqualToString:@"alloc"] || [selectorName isEqualToString:@"new"]) {
    returnValue = (__bridge_transfer id)result;
} else {
    returnValue = (__bridge id)result;
}
```



 