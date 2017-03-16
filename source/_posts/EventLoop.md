title: 从 setTimeout() 看js的 Event Loop 执行过程 
date: 2016/12/28 10:07:12 
categories: JavaScript
tags:
	- NodeJs
---

通过 `setTimeout()` 方法，来了解 js 中的代码执行。本文参照[干货 | 原来你是这样的 setTimeout](https://mp.weixin.qq.com/s?__biz=MzI1MTE2NTE1Ng==&mid=2649515867&idx=1&sn=971a3e41da08ddf2da200d9d07af0fb0&chksm=f1efe7d0c6986ec688a746ece15f52c8df78bca37ca2609e75199f5c3fbbabd3fbcc00179885&scene=0&key=564c3e9811aee0abcc036cb111e6e7bdbe3938a8756b5bf3b98a1696b2f16c1e6e3a1b4af159d1ae1dd3e71ee5fae4e0b6655bd9f37cc81efb1174bf3ef39b43f874bc6a0482348422cc5245dfae917f&ascene=0&uin=MzIxNTY1NTU%3D&devicetype=iMac+MacBookPro11%2C1+OSX+OSX+10.12.1+build(16B2555)&version=12010210&nettype=WIFI&fontScale=100&pass_ticket=g24dIjS%2F70EF4QPCYwRMInMa218z6XagvevxLr5Mbzc%3D)

<!--more-->

## Event Loop
Js 引擎是单线程的，在某一个特定的时间内只能执行一个任务，并阻塞其他任务的执行，也就是说这些任务是串行的，但是实际开发中我们却可以使用异步代码来解决。Js 为了引入异步的特性，引申出一个重要的东西，Event Loop（事件循环）。

当异步方法，比如 `setTimeout()` 执行的时候，会交由浏览器内核的其他模块去管理。当异步的方法满足触发条件后，该模块就会将方法推入到一个任务队列（task queue）中，当主线程代码执行完毕处于空闲状态的时候，就会去检查任务队列，将队列中第一个任务入栈执行，完毕后继续检查任务队列，如此循环。前提条件是主线程处于空闲状态，这就是事件循环的模型。

## 第一个 setTimeout 例子
### 例子
```javascript
console.log('start');

setTimeout(() => {
    console.log('hello');
},300);

setTimeout(()=>{
    console.log('world');
},200);

console.log('end');
```

运行结果为：
```
start
end
world
hello
```

### 过程
首先第一个 `console.log()` 入栈执行，执行完毕控制台打印 `start` 后出栈，紧接着执行到 `setTimeout` 定时器，此时 JS 引擎会将定时器交给浏览器的另一个模块去管理（为方便理解这里把它叫做 Timer 模块），然后主线程继续向下执行，紧接着将第二个定时器也交给 Timer 模块，然后执行到第二个 `console.log()`，控制台打印 `end`，执行完毕后清空执行栈。但是并没有结束，在主线程执行的同时，Timer 模块会检查其中的异步代码，**一旦满足触发条件，就会将它添加到任务队列中**(注意，是满足触发条件了再放到任务队列中)。Timer2 延迟 200ms，所以会早于 Timer1 被添加到队列排头。而主线程此时处于空闲状态，所以会检查任务队列是否有待执行的任务。此时会将 Timer2 回调中的 `console.log()` 执行，控制台打印 `world`，然后执行栈空闲后继续检查任务队列，将 Timer1 的代码压入执行栈中执行，控制台打印 `hello`，清空执行栈，此时任务队列为空，执行结束。

##第二个 setTimeout 例子
### 例子
```javascript
console.log('start');

setTimeout(() => {
    console.log('hello');
},300);

setTimeout(()=>{
    console.log('world');
},200);

for (var i=0;i<100000;i++){
    console.log(1)
}

setTimeout(()=>{
    console.log('i am run');
},100);

console.log('end');
```

结果：
```
start
...
end
hello
world
I am run
```

### 过程
Timer3 仅仅延迟了 100ms，反而在另外两个 Timer 之后执行了。其实这里原因很简单，因为在 Timer1 和 Timer2 加入到执行队列中后，主线程依然还在执行for循环中的代码，处于阻塞状态。队列中的 Timer1 和 Timer2 并不会得以执行。**当for循环结束，这时才将 Timer3 交由 Timer 模块去管理**(注意，代码顺序执行，for循环的时候， Timer3 的 `setTimeout()` 还没有执行，所以也就还没有被 Timer 模块管理)，继续执行后续代码打印 `end`，清空执行栈。虽然在这里 Timer3 的延迟时间最短，但是**加入任务队列后**还是会排在 Timer1 和 Timer2 的后面，所以此时按**顺序执行任务队列中的代码**，依次打印。

需要注意的是，执行完 `console.log('end');` 后，立刻执行了 Timer1 和 Timer2，但是 Timer3 执行时间约在之后 100ms，这是因为for循环执行的时间超过了300ms。如果for循环在100ms以内完成，那么 `console.log('i am run');` 仍然是最先执行的。如果for循环在100ms到200ms之间，那么 `console.log('i am run');` 在 `console.log('world');` 之后，`console.log('hello');` 之间执行。
