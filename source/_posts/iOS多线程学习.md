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



## 锁

### 自旋锁

#### 定义

自旋锁不会引起调用者睡眠，而是不停的循环，直到锁被释放。适用于多核。

#### OSSpinLock

iOS 中的自旋锁为 **OSSpinLock**。但是不建议使用。因为会产生优先级反转的现象。

#### 优先级反转

由于线程存在优先级，即根据优先级来分配 CPU 执行时间。但是会产生问题：如果一个低优先级的线程获得锁并访问共享资源，这时一个高优先级的线程也尝试获得这个锁，它会处于 spin lock 的忙等状态从而占用大量 CPU。此时低优先级线程无法与高优先级线程争夺 CPU 时间，从而导致任务迟迟完不成、无法释放 lock。

#### 特点

由于不停循环，避免了线程上下文切换的时间损耗。在能快速获取资源的多线程环境中速度最快。

### 互斥锁

#### 定义

顾名思义，等待锁的线程会处于休眠状态。

#### 常见类型

 iOS 使用以 `pthread_mutex` 为基础的各种封装。常见三种类型：

- PTHREAD_MUTEX_NORMAL
- PTHREAD_MUTEX_ERRORCHECK
- PTHREAD_MUTEX_RECURSIVE 

#### NSLock

针对第二种类型，iOS 封装了 `NSLock`。它会损失一定性能换来错误提示。并简化直接使用 pthread_mutex 的定义。

```objc
//主线程中
NSLock *lock = [[NSLock alloc] init];
    
//线程1
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [lock lock];
        NSLog(@"线程1");
        sleep(2);
        [lock unlock];
        NSLog(@"线程1解锁成功");
    });

    //线程2
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        sleep(1);//以保证让线程2的代码后执行
        [lock lock];
        NSLog(@"线程2");
        [lock unlock];
    });

2016-08-19 14:23:09.659 ThreadLockControlDemo[1754:129663] 线程1
2016-08-19 14:23:11.663 ThreadLockControlDemo[1754:129663] 线程1解锁成功
2016-08-19 14:23:11.665 ThreadLockControlDemo[1754:129659] 线程2
```

#### NSReursiveLock 和 @synchronized

针对第三种类型，iOS 封装了 `NSRecursiveLock` 和 `@synchronized`。它们是**递归锁**，也就是说同一个线程可以重复获取递归锁，不会死锁。`NSRecursiveLock` 和 `NSLock` 使用类似。

##### @synchronized 实现原理

`@synchronized` 中传入的object的内存地址，被用作key，系统创建了一个递归锁，作为值，保存在一个 hash map 中。每当再次遇到 `@synchronized` 关键字的时候，就会到 hash map 中得到这个锁，并且尝试获取这个锁，失败则挂起。

所以，如果object 被外部访问变化，`@synchronized` 就失去了锁的作用。因此一定要注意，不能改变 object 的地址。

> 这是一个考点，`@synchronized` 如何实现的

#### NSConditionLock

另外还有一种**条件锁**`NSConditionLock`。基于 `pthread_cond_t` 实现。只有 condition 参数与初始化时候的 condition 相等，lock 才能正确进行加锁操作。

```objc
//主线程中
NSConditionLock *lock = [[NSConditionLock alloc] initWithCondition:0];

//线程1
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    [lock lockWhenCondition:1];
    NSLog(@"线程1");
    sleep(2);
    [lock unlock];
});

//线程2
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    if ([lock tryLockWhenCondition:0]) {
        NSLog(@"线程2");
        [lock unlockWithCondition:1];
        NSLog(@"线程2解锁成功");
    } else {
        NSLog(@"线程2尝试加锁失败");
    }
});
```

### 信号量

#### 定义

信号量的初始值，可以用来控制线程并发访问的最大数量。信号量的初始值为1，代表同时只允许1条线程访问资源，保证线程同步

#### dispatch_semaphore

```objc
dispatch_semaphore_t signal = dispatch_semaphore_create(1);
// 如果信号量的值 > 0，就让信号量的值减1，然后继续往下执行代码
// 如果信号量的值 <= 0，就会休眠等待，直到信号量的值变成>0，就让信号量的值减1，然后继续往下执行代码
dispatch_semaphore_wait(signal, DISPATCH_TIME_FOREVER);
// 让信号量的值+1
dispatch_semaphore_signal(signal);
```

### 信号量和互斥锁的区别

虽然 Semaphore=1时可以看成互斥锁，但是它们真正的使用场景是有差别的。

锁是服务于共享资源的；而semaphore是服务于多个线程间的执行的逻辑顺序的。比如，a 和 b 执行完了再执行 c，就可以通过信号量实现，但是无法通过互斥锁实现。

###  速度比较

OSSpinLock > dispatch_semaphore > pthread_mutex > NSLock > NSRecursiveLock > NSConditionLock > @synchronized



## 其他

### 问题

1. i++ 在两个线程中分别执行100次，不加锁，最后 i 的取值范围？

2-200，200 的情况不用多说，2 的情况是：

1. 两个线程同时读取了初始值 0。
2. 线程1执行了 99 次，写回。此时内存为 99。
3. 线程2执行了一次，写回。此时内存为 1。
4. 两个线程同时读取值 1。
5. 线程2执行 99 次，写回。此时内存为 99
6. 线程1执行1次，写回。此时内存为 2

问题的关键在于，一个线程的写入与读取这两个操作之间，可以穿插别的线程的写入操作。