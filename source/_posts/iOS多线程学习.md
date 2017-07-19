title: iOS中的多线程学习笔记
date: 2016/11/1 14:07:12  
categories: iOS
tags: 
	- 学习笔记
---

在学习 RunLoop 的时候，碰到了一些不太理解的东西，查阅资料后发现是多线程的相关方法。因此在完成 RunLoop 的笔记前，先学习下多线程的使用方法。
<!--more-->

可以通过三种方式实现 iOS 的多线程：
- NSThread
- GCD
- NSOperation&NSOperationQueue

## NSThread
### 创建并启动
#### 先创建线程类，再启动

```objc
// 创建
NSThread *thread = [[NSThread alloc] initWithTarget:self selector:@selector(run:) object:nil];
// 启动
[thread start];
```

其中，`run:` 是即将执行的方法，`object` 是 `run:` 方法的参数。规定 `run:` 方法最多可有一个参数，且返回类型必须是 `void`。

#### 创建并自动启动

```objc
[NSThread detachNewThreadSelector:@selector(run:) toTarget:self withObject:nil];
```

#### 使用 NSObject 的方法创建并自动启动

```objc
[self performSelectorInBackground:@selector(run:) withObject:nil];
```

### 其他方法
除了创建启动外，NSThread 还以很多方法，下面我列举一些常见的方法，当然我列举的并不完整，更多方法可以去类的定义里去看。

```objc
//取消线程
- (void)cancel;

//启动线程
- (void)start;

//判断某个线程的状态的属性
@property (readonly, getter=isExecuting) BOOL executing;
@property (readonly, getter=isFinished) BOOL finished;
@property (readonly, getter=isCancelled) BOOL cancelled;

//设置和获取线程名字
-(void)setName:(NSString *)n;
-(NSString *)name;

//获取当前线程信息
+ (NSThread *)currentThread;

//获取主线程信息
+ (NSThread *)mainThread;

//使当前线程暂停一段时间，或者暂停到某个时刻
+ (void)sleepForTimeInterval:(NSTimeInterval)time;
+ (void)sleepUntilDate:(NSDate *)date;
```

## GCD
该部分前一篇关于 GCD 的文章已经较为详细的研究过了。

## NSOperation
暂时没有时间看，先占个坑

## 其他
### 线程同步
#### 互斥锁
使用 `@synchronized`:

```objc
@synchronized(self) {
    //需要执行的代码块
}
```

#### 同步执行
把多个线程都要执行此段代码添加到同一个串行队列，这样就实现了线程同步的概念。



### 从其他线程回到主线程的方法

在其他线程操作完成后必须到主线程更新UI

#### NSThread

```objc
[self performSelectorOnMainThread:@selector(run) withObject:nil waitUntilDone:NO];
```

#### GCD

```objc
dispatch_async(dispatch_get_main_queue(), ^{
  ...
});
```





