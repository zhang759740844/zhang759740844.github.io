title: GCD队列 学习与整理
date: 2016/8/2 14:07:12  
categories: iOS
tags: [GCD]

---

Grand Central Dispatch或者GCD，是一套低层API，提供了一种新的方法来进行并发程序编写。从基本功能上讲，GCD有点像NSOperationQueue，他们都允许程序将任务切分为多个单一任务然后提交至工作队列来并发地或者串行地执行。GCD比之NSOpertionQueue更底层更高效，并且它不是Cocoa框架的一部分。

<!--more-->

### 基本概念
- **Serial vs. Concurrent 串行 vs. 并发**
这些术语描述当任务相对于其它任务被执行，任务串行执行就是每次只有一个任务被执行，任务并发执行就是在同一时间可以有多个任务被执行。
- **Synchronous vs. Asynchronous 同步 vs. 异步**
在 GCD 中，这些术语描述当一个函数相对于另一个任务完成，此任务是该函数要求 GCD 执行的。一个同步函数只在完成了它预定的任务后才返回。一个异步函数，刚好相反，会立即返回，预定的任务会完成但不会等它完成。因此，一个异步函数不会阻塞当前线程去执行下一个函数。

### 队列分类
1. **Serial Queues 串行队列**
这些任务的执行时机受到 GCD 的控制；唯一能确保的事情是 GCD 一次只执行一个任务，并且按照我们添加到队列的顺序来执行。
由于在串行队列中不会有两个任务并发运行，因此不会出现同时访问临界区的风险；相对于这些任务来说，这就从竞态条件下保护了临界区，实现了锁的功能。
2. **Concurrent Queues 并发队列**
在并发队列中的任务能得到的保证是它们会按照被添加的顺序开始执行，但这就是全部的保证了。任务可能以任意顺序完成，你不会知道何时开始运行下一个任务，或者任意时刻有多少 Block 在运行。再说一遍，这完全取决于 GCD 。

### 队列类型
1. **The main queue**
与主线程功能相同。实际上，提交至main queue的任务会在主线程中执行。因为main queue是与主线程相关的，所以这是一个串行队列。由于是系统默认生成的，所以无法调用dispatch_resume()和dispatch_suspend()来控制执行继续或中断。这是在一个并发队列上完成任务后更新 UI 的共同选择。
2. **Global queues**
全局队列是并发队列，并由整个进程共享。进程中存在三个全局队列：高、中（默认）、低三个优先级队列。同样无法控制主线程dispatch队列的执行继续或中断。需要注意的是，三个队列不代表三个线程，可能会有更多的线程。并发队列可以根据实际情况来自动产生合理的线程数。
3. **用户队列**
用户自己创建的队列。可以创建单线程的串行队列，也可以创建多线程的并行队列。

### 队列创建方式
1. **dispatch_queue_t queue = dispatch_queue_create("com.dispatch.serial", DISPATCH_QUEUE_SERIAL);**
生成一个串行队列。第一个参数是队列的名称，在调试程序时会非常有用，所有尽量不要重名。第二个参数表示生成的队列是串行的，如果传入 null 默认是串行的。
2. **dispatch_queue_t queue = dispatch_queue_create("com.dispatch.concurrent", DISPATCH_QUEUE_CONCURRENT);**
生成一个并发执行队列。
3. **dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);**
获得全局队列。
4. **dispatch_queue_t queue = dispatch_get_main_queue()**
获得主线程队列。

### 提交 Job
向一个队列提交Job很简单：调用dispatch_async或dispatch_sync函数，传入一个队列和一个block。队列会在轮到这个block执行时执行这个block的代码。

1. **dispatch_async**
dispatch_async 函数会立即返回, block会在后台异步执行。
```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self goDoSomethingLongAndInvolved];
        NSLog(@"Done doing something long and involved");
});
```
	在典型的Cocoa程序中，你很有可能希望在任务完成时更新界面，这就意味着需要在主线程中执行一些代码。你可以简单地完成这个任务——使用嵌套的dispatch，在外层中执行后台任务，在内层中将任务dispatch到main queue：
```objc
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [self goDoSomethingLongAndInvolved];
        dispatch_async(dispatch_get_main_queue(), ^{
            [textField setStringValue:@"Done doing something long and involved"];
        });
});
```
2. **dispatch_sync**
dispatch_sync 同步执行 block，函数不返回，一直等到 block 执行完毕。一般情况下是在当前线程中完成，因为派发同步任务，本身就要等到任务完成才能继续执行，那么就没有必要再开一个线程去专门执行这个同步任务，执行完后，再返回该线程了。但是如果在其他线程里往主队列里派发同步任务，那么这个同步任务还是会在主线程里执行，当前线程阻塞。

实际编程经验告诉我们，尽可能避免使用 dispatch_sync，嵌套使用同一个队列时极易产生程序死锁，比如嵌套调用主线程：

![gcd_死锁](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/gcd_死锁.png?raw=true)

### 总结
**队列是串行或并发的，操作队列的函数是同步或者异步执行的（也就是在当前线程执行完返回和立即返回另开线程执行的区别）。串行队列其实就相当于加了一个资源锁，无论在多少个线程里，只有当队列中前一个元素执行完，后一个元素的代码块才能继续执行，并行队列则没有任何要求，有需要执行就立即执行。** 

比如下面写在主线程里的示例的四种组合：

```objc
for (int i = 1; i < 10; i++) {  
    dispatch_(a)sync(queue, ^{  
        NSLog(@"%d___%@",i, [NSThread currentThread]);  
    });  
} 

NSLog(@"over");
```

结论:
- 串行同步队列：运行在主线程里，先依次打印 `i` 后，再打印 `over`。
- 串行异步队列：先打印 `over`，再依次打印 `i`。由于是异步的，`over` 执行在主线程里毋庸置疑，打印 `i` 时，新建了一个线程。为什么是一个呢？因为串行队列，代码块依次执行。创建新线程，执行，销毁，再创建新线程的操作太耗时。所以编译器优化后，仅创建了一个新线程。
- 并行同步队列：运行在主线程里，先依次打印 `i` 后，再打印 `over`。现在看起来和串行同步队列一样对不对？那么什么时候才不同呢？假如你手动开了一个线程，并且在那个线程里，又运行了一遍上面的代码。串行同步队列由于有锁，执行当前代码块的时候，另一个线程处于阻塞状态，只能等到当前代码块执行完毕才能跳到另一个线程；并行同步队列没有锁，可能代码块没有执行完，由于线程的时间片用完了，就立即跳到另外一个线程上去执行了。
- 并行异步队列：先打印 `over`，然后瞎JB打印`i`。








### 常用方法
#### dispatch_apply
重复执行block，需要注意的是这个方法是同步返回，也就是说等到所有block执行完毕才返回。多个block的运行是否并发或串行执行也依赖queue的是否并发或串行。

``` objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_apply([array count], queue, ^(size_t index){
    [self doSomethingIntensiveWith:[array objectAtIndex:index]];
});
[self doSomethingWith:array];
```

如果需要异步执行这些代码，只需要用dispatch_async方法，将所有代码推至后台。

```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_async(queue, ^{
    dispatch_apply([array count], queue, ^(size_t index){
        [self doSomethingIntensiveWith:[array objectAtIndex:index]];
    });
    [self doSomethingWith:array];
});
```

那何时才适合用 dispatch_apply 呢？

- 自定义串行队列：串行队列会完全抵消 dispatch_apply 的功能；你还不如直接使用普通的 for 循环。
- 主队列（串行）：与上面一样，在串行队列上不适合使用 dispatch_apply 。还是用普通的 for 循环吧。
- 并发队列：对于并发循环来说是很好选择，特别是当你需要追踪任务的进度时。


#### dispatch_after
延迟提交 block：

```objc
double delayInSeconds = 1.0; 
dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC));
dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
     // code to be executed on the main queue after delay
});
```

dispatch_after 是延迟提交，不是延迟运行，**不是在特定的时间后立即运行！**：

```objc
//创建串行队列
dispatch_queue_t queue = dispatch_queue_create("me.tutuge.test.gcd", DISPATCH_QUEUE_SERIAL);

//立即打印一条信息
NSLog(@"Begin add block...");

//提交一个block
dispatch_async(queue, ^{
    //Sleep 10秒
    [NSThread sleepForTimeInterval:10];
    NSLog(@"First block done...");
});

//5 秒以后提交block
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), queue, ^{
    NSLog(@"After...");
});
```

结果如下:

```objc
2015-03-31 20:57:27.122 GCDTest[45633:1812016] Begin add block...
2015-03-31 20:57:37.127 GCDTest[45633:1812041] First block done...
2015-03-31 20:57:37.127 GCDTest[45633:1812041] After...
```

对于这个串行的队列，先 async 执行了阻塞10秒，此时，添加一个5秒的延时任务。由于5秒前一个任务还未返回，所以延迟任务不能立刻执行。当前一个任务返回后，延迟任务执行时发现已经过了预定时间，那么立即执行。

#### dispatch_once
保证在APP运行期间，block中的代码只执行一次
```objc
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
    // code to be executed once
});
```

一定要注意的是 `dispatch_once_t` **必须是全局或 static 变量**。否则使用时会导致非常不好排查的 bug。

#### dispatch_group
一个dispatch group可以用来将多个block组成一组以监测这些Block全部完成或者等待全部完成时发出的消息。
- *dispatch_group_create*创建一个调度任务组
- *dispatch_group_async* 把一个任务异步提交到任务组里
- *dispatch_group_notify* 用来监听任务组事件的执行完毕
- *dispatch_group_wait* 设置等待时间，在等待时间结束后，如果还没有执行完任务组，则返回。返回0代表执行成功，非0则执行失败
```objc
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
for(id obj in array)
    dispatch_group_async(group, queue, ^{
        [self doSomethingIntensiveWith:obj];
    });
dispatch_group_notify(group, queue, ^{
    [self doSomethingWith:array];
});
```


还有一个参考文章：
[GCD使用经验与技巧浅谈](http://tutuge.me/2015/04/03/something-about-gcd/)

