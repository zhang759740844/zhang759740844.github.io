title: 《RxSiwft Reactive Programming with Swift》的一点笔记
date: 2017/10/26 10:07:12  
categories: iOS
tags:
	- Swift
---

学习 RxSwift，开始冲了个泊学会员看视频，发现看视频还不如直接看泊学文档。之后看到了这本书，果然看文旦还不如看书。

<!--more-->

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_1.png?raw=true)

## 着手 RxSwift

### 简介

RxSwift 本质上是通过对新数据的顺序处理，来简化异步程序的开发。

#### RxSwift 基础

RxSwift 由三部分组成：Observables，Operators 和 Schedulers。

`Observables<T>` 帮助一个程序产生一个带有数据的事件序列，允许一个或多个 Observers 响应事件并处理新的数据。Observables 能发送三种类型的事件：next 事件，completed 事件，error 事件。使用事件序列是最终的解耦方式，不再需要使用 delegate 或者闭包。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_2.png?raw=true)

Operators 指的是有着异步的输入，然后产生没有副作用的输出的方法。它们能够组合在一起，实现复杂的逻辑。比如下图的 `filter` 和 `map` 的过程：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_3.png?raw=true)

Schedulers 是 Rx 中的调度队列。我们可以通过 RxSwift 将同一个订阅在不同的调度队列中完成以达到最佳的性能：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_4.png?raw=true)

#### App 架构

Rx 适合所有的架构。当然最匹配的还是 MVVM。MVVM 中有一层 ViewModel 能够暴露出各个 Observables 属性，你可以在 ViewController 中直接将其和 UI 绑定。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_5.png?raw=true)

#### RxCocoa

RxSwift 提供了通用的 Rx API。开发的时候还需要 RxCocoa 配合 RxSwift 使用 UIKit 和 Cocoa。比如 RxCocoa 给 UISwitch 添加了一个 `rx.isOn` 属性，你可以订阅它来监听 UISwitch 的事件队列。

 ### Observable

#### Observable 的生命周期

一个 Observable 不发出任何事件，直到有订阅发生。订阅者发送 error 或者 completed 事件能结束订阅。当订阅结束，也就不会再发出任何事件了。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_6.png?raw=true)

事件 event 是个枚举类型：

```swift
public enum Event<Element> {
  /// Next element is produced. 
  case next(Element)
  /// Sequence terminated with an error. 
  case error(Swift.Error)
  /// Sequence completed successfully. 
  case completed
}
```

#### 创建 Observables

`.just` 可以用来创建一个只有一个元素的 Observable：

```swift
let observable = Observable<Int>.just(one)
```

`.of` 接收多个元素形成一个 Observable：

```swift
let observable2 = Observable.of(1, 2, 3)
```

`.from` 接收一个数组，将数组元素生成一个 Observable：

```swift
let observable3 = Observable.from([1, 2, 3])
```

`.empty` 提供 0 个元素的 Observable，只会触发 completed 事件。尽管 empty 不触发 next 事件，但是还需要明确泛型类型，所以使用 void 是最好的选择：

```swift
let observable4 = Observable<Void>.empty()
```

`.never` 创建的 Observable 不会触发 next，也不会触发 completed：

```swift
let observable5 = Obnservable<Any>.never()
```

`.create` 创建 Observable，需要传入一个入参为 subscribe，返回一个 Disposable 实例的闭包。在这个闭包里定义了所有将要发送的事件：

```swift
Observable<String>.create { observer in 
	// 1 
	observer.onNext("1")
	// 2
	observer.onCompleted()
	// 3 
	observer.onNext("?")
	// 4 
	return Disposables.create()
}
```

注意前面生命周期中提及的，`onCompleted` 发生后，订阅结束。所以后面的 `observer.onNext("?")` 事件是无法接收到的。

#### 订阅 Observables

通过 `.subScribe` 订阅 Observable，返回一个 `Disposable` 对象。

下面是传入一个入参为事件 event 闭包的例子：

```swift
let disposable = observable.subScribe { event in
	if let element = event.element { 
    	print(element) 
    }
}
```

`.subScribe` 还可以接受各种事件的回调闭包，比如 `onNext`,`onCompleted`,`onError`：

```swift
let disposable2 = observable.subscribe(onNext: { element in 
	print(element) 
})
```

#### 取消订阅 Observables

如果不取消订阅，将会将会产生内存泄漏，取消订阅可以通过 `.dispose` 直接取消：

```swift
disposable.dispose()
```

但是这种手动的取消非常的麻烦，RxSwift 提供了一个 `DisposeBag` 类型，用于自动回收，把订阅后返回的 `disposable` 对象直接加入 `DisposeBag`:

```swift
let disposeBag = DisposeBag()
disposable.addDisposableTo(disposeBag)
```

当 `disposeBag` 回收的时候，会自动取消订阅。

> 取消订阅有两种方式，一种是此处说的 disposebag，还有一种方式是订阅者调用 onCompleted 方法。比如线面的 Subjects 就可以直接调用取消：`subject.onCompleted()`

### Subjects

#### 什么是 subjects？

subjects 既是 Observable 又是 Observer。有四种类型：

- PublishSubject：订阅后接受事件
- BehaviorSubject：有初始值，新的订阅者会获得订阅前最新的事件
- ReplySubject：有一个 buffer，新的订阅者会获得 buffer 中缓存的事件
- Variable：包装了的 BehaviorSubject。把当前值作为状态，值改变时触发事件。新的订阅者同样会获得订阅前最新的事件。

#### PublishSubjects

PublishSubject 只有在订阅后才能接收到事件。流程如图所示，在事件1时并没有进行订阅，所以并没有收到事件；在事件1和事件2之间进行订阅，能收到事件2和事件3；在事件2和事件3之间订阅，能收到事件3：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_7.png?raw=true)

示例代码如下，要注意初始化的时候要设置好泛型类型：

```swift
let disposeBag = DisposeBag()
// 创建 PublishSubject
let subject = PublishSubject<Int>()
// 订阅
subject.subscribe { print($0.element ?? $0) }
	.addDisposableTo(disposeBag)
// 发送事件
subject.onNext(1)							//1
// 结束订阅
subject.onCompleted()						//completed
// 再次订阅
subject.subscribe { print($0.element ?? $0) }
	.addDisposableTo(disposeBag)			//completed
// 发送事件
subject.onNext(2)							//什么也没有发生
```

要注意一点，当一个 subject 取消订阅后，再订阅，会直接受到一个 completed 事件，之后再发送 next 事件是没有任何响应的。

#### BehaviorSubjects

BehaviorSubjects 在订阅之后可以接到最近一次的事件。并且接受一个初始值，当订阅前没有事件时，将初始值作为事件值发送。如果所示，事件1和事件2之间订阅的时候，还是能收到事件1；事件2和事件3之间订阅的时候，还是能收到事件2：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_8.png?raw=true)

示例代码如下，由于有初始值的类型推断，就不需要设置泛型了：

```swift
let disposeBag = DisposeBag()
// 创建 BehaviorSubject
let subject = BehaviorSubject(value: "Initial value")
// 订阅
subject.subscribe { print($0.element ?? $0) }
	.addDisposableTo(bag)					// Initial Value
// 发送事件
subject.onNext("X")							// X
// 错误事件
subject.onError(MyError.anError)			// anError
// 再次订阅
subject.subscribe { print($0.element ?? $0) }
	.addDisposableTo(bag)					// anError
// 发送事件
subject.onNext("X")							// 没有任何事情发生
```

和 PublishSubject 类似的，当 subject 完成或者错误时，再订阅，就只会发出完成和错误事件，并且忽略后面的 next 事件。

#### ReplaySubjects

ReplaySubject 有一个 buffer 用来暂存先前的事件值，初始化的时候设置缓存的大小。当有订阅时，将缓存的值发送。如图所示，当在事件2和事件3之间产生订阅的时候，缓存的事件1和事件2都触发了：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_9.png?raw=true)

示例代码如下，还是要注意设置泛型，并且初始化方法是 `create(bufferSize:)`

```swift
let disposeBag = DisposeBag()
// 创建 ReplaySubject
let subject = ReplaySubject<String>.create(bufferSize: 2)
// 发送事件
subject.onNext("1")
subject.onNext("2")
// 订阅
subject.subscribe { print($0.element ?? $0) }
	.addDisposableTo(bag)				// 1    2
// 发送错误
subject.onError(MyError.annError)		// annError
// 再次订阅
subject.subscribe { print($0.element ?? $0) }
	.addDisposableTo(bag)				// 1	2	annError
// 发送事件
subject.onNext("3")						// 没有任何反应
```

注意，和前面都相同的是，由于之前 subject 已经发出错误事件，再次订阅的时候会立刻收到错误事件。不同的是，由于 ReplaySubject 有 buffer，所以仍然会先响应 buffer 中缓存的事件。

#### Variables

Variable 是 BehaviorSubject 的包装，将其值作为状态。当值变化时，触发事件，不再需要 `onNext()` 方法触发事件。Variable 不会出现 error，也不需要手动触发 completed 事件。

代码示例如下：

```swift
let disposeBag = DisposeBag()
// 创建 Variable
var variable = Variable("Initial value")
// 更改值
variable.value = "New Value"
// 订阅
variable.asObservable()
	.subscribe {
        print($0.element)
    }.addDisposableTo(bag)			// New Value
```

Variable 需要使用 `asObservable()` 方法将其转换为 Observable，值是其 `value` 属性。另外，由于是 BehaviorSubject 的包装，因此，订阅的时候会打印最后一次的事件值。

## 操作符的最佳实践

### 过滤操作符





















