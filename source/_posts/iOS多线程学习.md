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
### 创建

NSOperation 是个抽象类，不能用来封装操作。我们只有使用它的子类来封装操作。我们有三种方式来封装操作。

1. 使用子类 NSInvocationOperation
2. 使用子类 NSBlockOperation
3. 自定义继承自 NSOperation 的子类，通过实现内部相应的方法来封装操作。

在不使用 NSOperationQueue，单独使用 NSOperation 的情况下系统同步执行操作，下面我们学习以下操作的两种创建方式。

> **NSOperation 一大优点就是可以通过 cancel 方法取消**
>
> NSOperation 还可以通过 KVO 监听 `finished` 以及 `executing` 状态

#### NSInvocationOperation

 ```objc
/**
 * 使用子类 NSInvocationOperation
 */
- (void)useInvocationOperation {

    // 1.创建 NSInvocationOperation 对象
    NSInvocationOperation *op = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(task1) object:nil];

    // 2.调用 start 方法开始执行操作
    [op start];
}

/**
 * 任务1
 */
- (void)task1 {
    for (int i = 0; i < 2; i++) {
        [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
        NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
    }
}
 ```

在没有使用 NSOperationQueue、在主线程中单独使用使用子类 NSInvocationOperation 执行一个操作的情况下，**操作是在当前线程执行的，并没有开启新线程**。

#### NSBlockOperation

```objc
/**
 * 使用子类 NSBlockOperation
 */
- (void)useBlockOperation {

    // 1.创建 NSBlockOperation 对象
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 2.调用 start 方法开始执行操作
    [op start];
}
```

在没有使用 NSOperationQueue、在主线程中单独使用 NSBlockOperation 执行一个操作的情况下，**操作是在当前线程执行的，并没有开启新线程**。

但是，NSBlockOperation 还提供了一个方法 `addExecutionBlock:`，通过 `addExecutionBlock:` 就可以为 NSBlockOperation 添加额外的操作。这些操作（包括 blockOperationWithBlock 中的操作）可以在不同的线程中同时（并发）执行。只有当所有相关的操作已经完成执行时，才视为完成。

使用子类 `NSBlockOperation`，并调用方法 `AddExecutionBlock:` 的情况下，`blockOperationWithBlock:`方法中的操作 和 `addExecutionBlock:` 中的操作是在不同的线程中异步执行的。而且，这次执行结果中 `blockOperationWithBlock:`方法中的操作也不是在当前线程（主线程）中执行的。从而印证了`blockOperationWithBlock:` 中的操作也可能会在其他线程（非当前线程）中执行。

### 创建队列

NSOperationQueue 一共有两种队列：主队列、自定义队列。其中自定义队列同时包含了串行、并发功能。下边是主队列、自定义队列的基本创建方法和特点。

- 主队列 
  - 凡是添加到主队列中的操作，都会放到主线程中执行。

```objc
// 主队列获取方法
NSOperationQueue *queue = [NSOperationQueue mainQueue];
复制代码
```

- 自定义队列（非主队列） 
  - 添加到这种队列中的操作，就会自动放到子线程中执行。
  - 同时包含了：串行、并发功能。

```objc
// 自定义队列创建方法
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
复
```

#### 将操作加入到队列中

上边我们说到 NSOperation 需要配合 NSOperationQueue 来实现多线程。

那么我们需要将创建好的操作加入到队列中去。总共有两种方法。

##### - (void)addOperation:(NSOperation *)op;

需要先创建操作，再将创建好的操作加入到创建好的队列中去。

```objc
/**
 * 使用 addOperation: 将操作加入到操作队列中
 */
- (void)addOperationToQueue {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.创建操作
    // 使用 NSInvocationOperation 创建操作1
    NSInvocationOperation *op1 = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(task1) object:nil];

    // 使用 NSInvocationOperation 创建操作2
    NSInvocationOperation *op2 = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(task2) object:nil];

    // 使用 NSBlockOperation 创建操作3
    NSBlockOperation *op3 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"3---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [op3 addExecutionBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"4---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 3.使用 addOperation: 添加所有操作到队列中
    [queue addOperation:op1]; // [op1 start]
    [queue addOperation:op2]; // [op2 start]
    [queue addOperation:op3]; // [op3 start]
}
```

使用 NSOperation 子类创建操作，并使用 `addOperation:` 将操作加入到操作队列后**能够开启新线程，进行并发执行**。

##### - (void)addOperationWithBlock:(void (^)(void))block;

无需先创建操作，在 block 中添加操作，直接将包含操作的 block 加入到队列中

```objc
/**
 * 使用 addOperationWithBlock: 将操作加入到操作队列中
 */

- (void)addOperationWithBlockToQueue {
    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.使用 addOperationWithBlock: 添加操作到队列中
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"2---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    [queue addOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"3---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
}
```

使用 addOperationWithBlock: 将操作加入到操作队列后**能够开启新线程，进行并发执行**.

### NSOperationQueue 控制串行执行、并发执行

这里有个关键属性 `maxConcurrentOperationCount`，叫做**最大并发操作数**。用来控制一个特定队列中可以有多少个操作同时参与并发执行。

> 注意：这里 `maxConcurrentOperationCount` 控制的不是并发线程的数量，而是一个队列中同时能并发执行的最大操作数。而且一个操作也并非只能在一个线程中运行。

最大并发操作数：

```
maxConcurrentOperationCount
```

- `maxConcurrentOperationCount` 默认情况下为-1，表示不进行限制，可进行并发执行。
- `maxConcurrentOperationCount` 为1时，队列为串行队列。只能串行执行。
- `maxConcurrentOperationCount` 大于1时，队列为并发队列。操作并发执行，当然这个值不应超过系统限制，即使自己设置一个很大的值，系统也会自动调整为 min{自己设定的值，系统设定的默认最大值}。

### NSOperation操作依赖

NSOperation、NSOperationQueue 最吸引人的地方是它能添加操作之间的依赖关系。通过操作依赖，我们可以很方便的控制操作之间的执行先后顺序。NSOperation 提供了3个接口供我们管理和查看依赖。

- `- (void)addDependency:(NSOperation *)op;` 添加依赖，使当前操作依赖于操作 op 的完成。
- `- (void)removeDependency:(NSOperation *)op;` 移除依赖，取消当前操作对操作 op 的依赖。
- `@property (readonly, copy) NSArray<NSOperation *> *dependencies;` 在当前操作开始执行之前完成执行的所有操作对象数组。

当然，我们经常用到的还是添加依赖操作。现在考虑这样的需求，比如说有 A、B 两个操作，其中 A 执行完操作，B 才能执行操作。

如果使用依赖来处理的话，那么就需要让操作 B 依赖于操作 A。具体代码如下：

```objc
/**
 * 操作依赖
 * 使用方法：addDependency:
 */
- (void)addDependency {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc] init];

    // 2.创建操作
    NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"2---%@", [NSThread currentThread]); // 打印当前线程
        }
    }];

    // 3.添加依赖
    [op2 addDependency:op1]; // 让op2 依赖于 op1，则先执行op1，在执行op2

    // 4.添加操作到队列中
    [queue addOperation:op1];
    [queue addOperation:op2];
}
```

通过添加操作依赖，无论运行几次，其结果都是 op1 先执行，op2 后执行。

### NSOperation 优先级

NSOperation 提供了`queuePriority`（优先级）属性，`queuePriority`属性适用于同一操作队列中的操作，不适用于不同操作队列中的操作。默认情况下，所有新创建的操作对象优先级都是`NSOperationQueuePriorityNormal`。但是我们可以通过`setQueuePriority:`方法来改变当前操作在同一队列中的执行优先级。

## 其他

### NSNotification 与 多线程

`NSNotification` 在哪个线程 post，最终就会在哪个线程执行。如果我们不是在主线程 post 的，但是却在主线程接收的，而且我们期望 selector 在主线程执行。这时候我们需要注意下，在 selector 需要 dispatch 到主线程才可以

```objc
@implementation BLPostNotification

- (void)postNotification {
    dispatch_queue_t queue = dispatch_queue_create("com.bool.post.notification", DISPATCH_QUEUE_SERIAL);
    dispatch_async(queue, ^{
        // 从非主线程发送通知 （通知名字最好定义成一个常量）
        [[NSNotificationCenter defaultCenter] postNotificationName:@"downloadImage" object:nil];
    });
}
@end

@implementation ImageViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(show) name:@"downloadImage" object:nil];
}

- (void)showImage {
    // 需要 dispatch 到主线程更新 UI
    dispatch_async(dispatch_get_main_queue(), ^{
        // update UI
    });
}
@end
```



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

#### NSOperation

```objc
/**
 * 线程间通信
 */
- (void)communication {

    // 1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc]init];

    // 2.添加操作
    [queue addOperationWithBlock:^{
        // 异步进行耗时操作
        for (int i = 0; i < 2; i++) {
            [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
            NSLog(@"1---%@", [NSThread currentThread]); // 打印当前线程
        }

        // 回到主线程
        [[NSOperationQueue mainQueue] addOperationWithBlock:^{
            // 进行一些 UI 刷新等操作
            for (int i = 0; i < 2; i++) {
                [NSThread sleepForTimeInterval:2]; // 模拟耗时操作
                NSLog(@"2---%@", [NSThread currentThread]); // 打印当前线程
            }
        }];
    }];
}
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
    if ([lock lockWhenCondition:0]) {
        NSLog(@"线程2");
        [lock unlockWithCondition:1];
        NSLog(@"线程2解锁成功");
    } else {
        NSLog(@"线程2尝试加锁失败");
    }
});
```

`NSConditionLock` 就针对于多个线程在复制场景下的同步。

### 信号量

#### 定义

信号量的初始值，可以用来控制线程并发访问的最大数量。信号量的初始值为1，代表同时只允许1条线程访问资源，保证线程同步。

信号量是负几，就表示有几个线程在等待资源。

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

> 二元信号量和互斥锁的区别在《程序员的自我修养》中提及，信号量可以在非当前线程释放，而互斥锁只能在当前线程释放。
>
> 但是 NSLock 实测是可以在不同线程释放的。所以上述结论也不准确。只要记住信号量可以允许 N 个信号量允许 N 个线程并发地执行任务。

### 条件变量和信号量区别

每个信号量有一个与之关联的值，发出时+1，等待时-1，任何线程都可以发出一个信号，即使没有线程在等待该信号量的值。

可是对于条件变量，例如 pthread_cond_signal 发出信号后，没有任何线程阻塞在 pthread_cond_wait 上，那这个条件变量上的信号会直接丢失掉。

###  速度比较

OSSpinLock > dispatch_semaphore > pthread_mutex > NSLock > NSRecursiveLock > NSConditionLock > @synchronized

### 如何使用互斥锁实现读写锁？

```java
class readwrite_lock
{
public:
	readwrite_lock()
		: read_cnt(0)
	{
	}

	void readLock()
	{
		read_mtx.lock();
		if (++read_cnt == 1)
			write_mtx.lock();

		read_mtx.unlock();
	}

	void readUnlock()
	{
		read_mtx.lock();
		if (--read_cnt == 0)
			write_mtx.unlock();

		read_mtx.unlock();
	}

	void writeLock()
	{
		write_mtx.lock();
	}

	void writeUnlock()
	{
		write_mtx.unlock();
	}

private:
	mutex read_mtx;
	mutex write_mtx;
	int read_cnt; // 已加读锁个数
};
```



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



## 参考

[谈 iOS 的锁](<https://juejin.im/post/5a8fdb1c5188257a856f55a8#heading-26>)

[iOS多线程：『NSOperation、NSOperationQueue』详尽总结](<https://juejin.im/post/5a9e57af6fb9a028df222555#heading-9>)