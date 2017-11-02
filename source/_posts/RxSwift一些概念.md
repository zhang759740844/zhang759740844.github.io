title: 《RxSiwft Reactive Programming with Swift》的一点笔记
date: 2017/10/26 10:07:12  
categories: iOS
tags:
	- Swift
---

学习 RxSwift，开始冲了个泊学会员看视频，发现看视频还不如直接看泊学文档。之后看到了这本书，果然看文档还不如看书。（泊学视频中的RxSwift 就是照搬了这本书的前两个 Section，例子也是照搬的）

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

### Subject 与 Observer 和 Observable 的区别

这一节是我自己的理解，可能在理解上有偏差。如果看到这里可以自己也思考一下。

#### Subject 与 Observable 的不同

`onNext`，`onError`，`onCompleted` 都是 Observer 中的方法，但是本身并没有具体的实现。Observable 在订阅的时候传入了这几个方法的实现，所以在触发事件的时候，直接内部调用这些闭包响应事件即可。你可以把这个过程想象成 Observable 调用其内部的 Observer。**所以 Observable 必须要能被订阅，订阅有什么用？用来传递事件处理方法**。

Subject 既是 Observable 又是 Observer，我们订阅完，传入事件的实现后，就可以直接调用 `observer.onNext()`。相当于，我们**可以选择主动触发事件的时机**，然后 Observer 响应事件。这很有用，比如处理 UI 交互，就必须要在任意时刻都能主动触发 `observer.onNext()` 方法。而单纯的 Observable 则不行，它是非常被动的，要么直接触发，要么延迟多久或者多少周期触发。

#### Subject 与 Observer 的不同

对于单纯的 Observer，由于没有这些事件方法的具体实现，所以我们也不能像 Subject 一样主动触发事件。但是**有一些特殊的 Observer 本身在初始化的时候就提供了事件处理的方法**，这种 Observer 不需要像 subscribe 一样订阅的时候传入事件处理方法，直接使用 `bindTo()` 绑定即可。

不过也正是因为 Observer 提供了默认的事件处理方法，所以我们不能用其处理 UI 交互。因为 UI 交互的事件处理方法是要根据具体情况而定的。比如点击一个按钮我可能要让计数加一，或者让计数减一。

#### 总结

**所以，在我的理解下，Observable 需要传递事件处理方法，所以可以定制，但不能主动触发；Observer 有默认事件处理方法，所以可以主动触发，但不能定制。**

**因为 Subject 既能主动选择触发事件的时机，又能个性化事件处理的方法，所以一般交互的 UI 控件中的属性都是 Subject 类型。**



## 操作符的最佳实践

### 过滤操作符

#### Igoring operators

##### ignoreElements

ignoreElements 用来忽略所有的 `.next` 事件。所以用来指接受 completed 事件

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_10.png?raw=true)

示例如下：

```swift
let strikes = PublishSubject<String>
strikes.ignoreElements()
	.subscribt { print("You are out") }
	.addDisposableTo(bag)
```

##### elementAt

elementAt 只会获取索引序号的事件，忽略其他的所有 `.next`。索引序号从 0 开始:

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_11.png?raw=true)

代码如下：

```swift
let strikes = PublishSubject<String>()
strikes.elementAt(1)
	.subscribe { print("You are out") }
	.addDisposableTo(bag)
```

##### filter

filter 接受一个断言闭包，入参为当前事件值。接受所有断言正确的事件。注意是接受而不是过滤掉：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_12.png?raw=true)

代码如下：

```swift
Observable.of(1, 2, 3, 4, 5)
	.filter { $0 % 2 == 0}
	.subscribe { print($0.elemetn ?? $0) }
	.addDisposableTo(bag)
```

#### Skipping operators

上面一节是根据条件筛选；这一节是忽略到某个满足条件的，后面的全部接受。

##### skip

skip 用来略过一定数量的 `.next`事件，然后开始接受。从 1 开始计数

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_13.png?raw=true)

代码如下：

```swift
Observable.of(1, 2, 3, 4, 5)
	.skip(2)
	.subscribe { print($0.element ?? $0)}
	.addDisposableTo(bag)
```

##### skipWhile

skipWhile 是略过直到某个满足条件的事件发生。skipWhile 还有一个兄弟方法 skipWhileWithIndex，除了接受事件值，还接受事件序号，index 从 0 开始：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_14.png?raw=true)

代码如下：

```swift
Observable.of(1, 2, 3, 4, 5)
	.skipWhile { $0 % 2 == 0 }
	.subscribe { print($0.element ?? $0)}
	.addDisposableTo(bag)
```

##### skipUntil

skipUntil 是略过直到某个事件发生。就是当前 Observable 和另外一个 Observable 相关联，当特定的 Observable 的 `.next` 发生的时候，才开始接受事件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_15.png?raw=true)

代码如下：

```swift
let subject = PublishSubject<String>()
let trigger = PublishSubject<String>()

subject.skipUntil(trigger)
	.subscribe { print( $0.element ?? $0) }
	.addDisposableTo(bag)
// ... 当前时刻虽然订阅了，但是发送事件是无反应的
trigger.onNext("a")
// ... 由于 trigger 发送了 onNext 事件，现在 subject 可以接收到 next 事件了
```

 #### Taking operators

本小节和上一小结正好相反，这一小节是接受某个事件之前的所有事件，之后的都不接受。

##### take

take 和 skip 正好相反。take 是直到一定数量的事件发生后才开始取，而不是取到一定数量的时候停

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_16.png?raw=true)

代码如下：

```swift
Observable.of(1, 2, 3, 4, 5)
	.take(2)
	.subscribe { print($0.element ?? $0) }
	.addDisposableTo(bag)
```

##### takeWhile

takeWhile 和 skipWhile 正好相反。takeWhile 是取到某个不满足条件的事件。takeWhile 还有一个兄弟方法 takeWhileWithIndex，除了事件值 value，还接受事件的序号 index：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_17.png?raw=true)

代码如下：

```swift
Observable.of(1, 2, 3, 4, 5)
	.takeWhileWithIndex { v, i in 
		v > 1 && i >1
	}.subscribe { print($0.element ?? $0) }
	.addDisposableTo(bag)
```

##### takeUntil

takeUntil 和 skipUntil 相反。表示接受事件知道某个 Observable 的事件触发：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_18.png?raw=true)

代码如下：

```swift
let subject = PublishSubject<String>()
let trigger = PublishSubject<String>()

subject.takeUntil(trigger)
	.subscribe { print($0.element ?? $0 )}
	.addDisposableTo(bag)
// ... 此时一直接受 next 事件
trigger.onNext("x")
// ... 现在忽略所有的 next 事件了
```

#### Distinct operators

本小节的操作符可以防止重复事件

##### distinctUntilChanged

如果当前事件和前一个事件的事件值相同，那么忽略这个事件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_19.png?raw=true)

代码如下：

```swift
Observable.of(1, 2, 2, 1)
	.distinctUntilChanged()
	.subscribe { print($0.elemet ?? $0) }
	.addDisposableTo(bag)
```

distinctUntilChanged 还接受一个闭包，闭包入参为相邻事件的事件值，闭包返回值为 true 则忽略当前事件。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_20.png?raw=true)

代码如下，当前一个事件值为 1，当前事件值为 2 的时候忽略当前事件：

```swift
Observable.of(1, 2, 2, 1)
	.distinctUntilChanged { a, b in
		if a == 1 && b == 2 {
            return true
        }
		return false
    }
```

### 转换操作符

#### 转换元素

##### toArray

将事件序列的元素转换成一个数组，然后将这个数组作为事件值，触发 `.next` 事件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_21.png?raw=true)

代码示例：

```swift
Observable.of("A", "B", "C")
	.toArray()
	.subscribe(onNext: { print($0) })
	.addDisposableTo(bag)					// ["A", "B", "c"]
```

##### map

map 和数组中的 map 无异：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_22.png?raw=true)

代码示例：

```swift
Observable.of(1, 2, 3)
	.map{ $0 * 2}
	.subscribe{ print($0.element ?? $0) }
	.addDisposableTo(bag)
```

map 还有一个兄弟方法 mapWithIndex 带有一个索引：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_23.png?raw=true)

代码示例：

```swift
Observable.of(1, 2, 3)
	.mapWithIndex{ v, i in
		i > 1 ? v * 2 : v
	}.subscribe{ print($0.element ?? $0) }
	.addDisposableTo(bag)
```

#### 转换内部 Observables

##### flatMap

flatMap 主要就是**将一个 Observable 中的每个元素都转换为一个 Observable，并且订阅**。flatMap 需要一个闭包，传入当前 Observable 的事件值，返回一个新的 Observable。下面图示的例子中 Observable 的事件值类型为 Variable。传入一个 Variable，将其 value 属性扩大十倍：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_24.png?raw=true)

代码示例：

```swift
let outter = PublishSubject<Variable<Int>>
outter.asObservable()
	.flatMap { 
		$0.value *= 10
  		return $0.asObservable()
	}.subscribe(onNext: { print($0) })
	.addDisposableTo(bag)

outter.onNext(Variable(1))
outter.onNext(Variable(2))
```

##### flatMapLatest

flatMapLatest 是 flatMap 和 switchlatest 的合体。在使用上和 flatMap 一致，但是它值订阅最新的 Observable：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_25.png?raw=true)

看出区别了么？当有新的订阅产生的时候，旧的 Observable 就取消订阅了，所以这里由于订阅了绿色的 Observable，所以蓝色变为 30，并不会触发订阅；由于订阅了橙色的 Observable，所以绿色变为 50，也不会触发订阅。而 flatMap 则是全部都触发了订阅的。

### 关联操作符

#### 前缀与串联

##### startWith

startWith 接受一个事件值，将其插到当前事件序列的最前面，返回一个新的 Observable：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_26.png?raw=true)

代码如下：

```swift
let numbers = Observable.of(2, 3, 4)
let observable = numbers.startWith(1)
observable.subscribe{ print($0.element ?? $0) }
	.addDisposableTo(bag)
```

##### concat

concat 连接两个事件序列，生成一个新的事件序列：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_27.png?raw=true)

代码如下，注意要用 `[]` 将事件序列当成一个数组：

```swift
let first = Observable.of(1, 2, 3)
let second = Observable.of(4, 5, 6)
let observable = Observable.concat([first, second])

observable.subscribe{ print($0.element ?? $0) }
	.addDisposableTo(bag)
```

连个事件序列的泛型类型一定要相同，否则崩溃给你看😖

#### 合并

##### merge

merge 将元素为事件序列的事件序列自动拆开，成为一个新的事件序列。其实你也可以分开来写，让它们分别订阅，merge 主要就是用来减少事件序列的订阅的：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_28.png?raw=true)

代码如下：

```swift
let left = PublishSubject<String>()
let right = PublishSubject<String>()
// 将两个事件序列作为事件值
let source = Observable.of(left.asObservable(), right.asObservable())
// 将新的事件序列的元素合并，返回一个新的事件序列
let observable = source.merge()

observable.subscribe{ print($0.element ?? $0) }
	.addDisposableTo(bag)
```

注意，只有当内部的时间序列都 completed 后，merge 产生的事件序列才会 completed。

#### 关联元素

##### combineLatest 

当 combineLatest 中的子序列中的任意一个发出事件的时候，将会调用一个你提供的闭包。这个闭包将子序列的最近的事件值作为入参传入，得到的返回值作为事件值执行订阅的方法。主要用在同时监控多个源的状态。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_29.png?raw=true)

上面图示中，当事件 1 触发的时候，由于另外一个序列没有事件发生过，所以不触发订阅，直到那一个序列发生了事件 4，才触发了订阅。

代码如下：

```swift
let left = PublishSubject<String>()
let right = PublishSubject<String>()
let observable = Observable.combineLatest(left, right, resultSelector: {
    lastLeft, lastRight in
  	"\(lastLeft) \(lastRight)"
})
observable.subscribe(onNext: { value in 
		print(value)
	}).addDisposableTo(bag)
```

另外需要说明的就是只有两个子序列都 completed，外部序列才会 completed。如果其中一个子序列先结束了，当另外一个序列触发事件的时候，使用的是结束的那个子序列结束前最后一次事件的事件值。其实上面的图中也有展示，right 先结束了，此时left 触发了事件 3，所以最终是将 3，6 的值作为事件值的。

##### zip

和上面的 combineLatest 不同，zip 要求必须每个子序列都有新消息的时候，才触发事件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_30.png?raw=true)

可以看到，left 和 right 都必须有新的消息最终才能产生事件。由于 right 已经结束了，所以 sunny 永远不会接收到。

代码如下：

```swift
let left = PublishSubject<String>()
let right = PublishSubject<String>()
let observable = Observable.zip(left, right) {
    lastLeft, lastRight in
  	"\(lastLeft) \(lastRight)"
})
observable.subscribe(onNext: { value in 
		print(value)
	}).addDisposableTo(bag)
```

需要注意的是，zip 不需要所有内部序列都完成，只要有一个 completed，整个事件序列就结束了。

(确实这个挺像拉链的，名字起得很形象)

#### 触发器

##### withLatestFrom

当一个 Observable 触发的时候，获取另一个 Observable 的最新的事件值。很常用，比如点击按钮的时候要获取 textfield 的最新值：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_31.png?raw=true)

text 不管怎么修改，当 button 点击的时候，都获得最新的 text 的值。

代码如下：

```swift
let button = PublishSubject<Void>()
let textField = PublishSubject<String>()

button.WithLatestFrom(textField)
	.subScribe { print($0.element ?? $0) }
	.addDisposableTo(bag)
```

##### simple

触发某个 Observable 获取另一个 Observable 的最新值。但是和 withLatestFrom 不同的是，当再次触发这个 Observable 的时候，如果另一个 Observable 没有更新值，那么不会触发事件，类似于 distinctUntilChanged：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_32.png?raw=true)

代码如下：

```swift
let Observable = textField.sample(button)
```

一定要注意这里啊，前面是 `button.withLatestFrom(textField)`，这里是 `textField.sample(button)`。

#### 开关

##### amb

当两个 Observable 中的任意一个触发的时候，取消订阅另一个，以后只接受当前 Observable 的事件。如图所示，由于 right 先出法，所以就取消了 left 的订阅，以后就只能接收到 right 的事件了：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_33.png?raw=true)

代码如下：

```swift
let left = PublishSubject<String>()
let right = PublishSubject<String>()

left.amb(right)
	.subscribe { print($0.element ?? $0) }
	.addDisposableTo(bag)
```

##### switchLatest

前面那个是被动的哪个 Observable 最先触发就一直订阅哪一个。这个是可以自己控制当前想要订阅那个 Observable：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_34.png?raw=true)

如图所示，source 在选择 one 的时候，只接受 one 的事件，在选择 two 的时候，只接受 two 的事件。

```swift
let one = PublishSubject<String>()
let two = PublishSubject<String>()
let three = PublishSubject<String>()

// source 的事件值类型是 Observable 类型
let source = PublishSubject<Observable<String>>()

let observable = source.switchLatest()
let disposable = observable.subscribe(onNext: { value in print(value) })

// 选择Observable one
source.onNext(one)
one.onNext("emit") 				// emit
two.onNext("emit")				// 没有 emit
// 选择Observable two
source.onNext(two)
two.onNext("emit")				// emit
```

还记得 flatMapLatest 吗？之前说过 flatMapLatest，就是 map + switchLatest

#### 元素和序列的结合

##### reduce

Rx 中的 reduce 类似于 Swift 中的 reduce，将一个序列的所有事件值通过运算，得到一个最终的事件值，并触发事件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_35.png?raw=true)

代码如下：

```swift
Observable.of(1, 2, 3)
	.reduce(0) { summary, newValue in
        return summary + newValue
    }.subscribe { print($0.element && $0) }
	.addDeposiableAt(bag)
```

##### scan

scan 和 reduce 的不同在于，reduce 是一锤子买卖，scan 每次接收到事件值时都会触发一个事件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_36.png?raw=true)

代码如下：

```swift
Observable.of(1, 2, 3)
	.scan(0, accumulator: +)
	.subscribe(onNext: { value in print(value) })
	.addDeposiableTo(bag)
```

### 基于时间的操作符

#### 缓存操作符

##### replay

这个操作符是针对**一个 Observable，多个订阅者**的。为 Observable 设置 replay ，当有新的订阅者订阅的时候，会立即触发最近的几个事件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_37.png?raw=true)

上面的是 Observable，每隔 1s 发出一次事件。下面是一个订阅者，在第 4s 的时候开始订阅。由于设置了 replay 的数量为 1，所以立刻重现之前的事件 3，再加上当前事件 4 的触发，所以再时刻 4，有两个事件一起触发了。

代码如下：

```swift
let interval = Observable<Int>.interval(1,
    scheduler:MainScheduler.instance).replay(1)

_ = interval.connect()

delay(3) {
    _ = interval.subscribe(onNext: {
        print("Subscriber 2: Event - \($0) at \(stamp())")
    })
}
```

这种一个 Observable，多个订阅者的情况叫做可连接 Observable，一般的 Observable 类型为 `Observable<E>`，这种类型为 `ConnectableObservable<E>`。所以需要使用 `.connect` 方法来表示 Observable 开始运行。

如果要所有元素都重现，那么可以使用 `.replayAll()`

##### buffer

buffer 用于将事件缓存，在某个条件下，一并发出。`buffer(timeSpan:count:scheduler:)` 接受一个最大时间跨度 timeSpan，一个事件最大发生数量 count。处理逻辑在于，当最大时间跨度内事件数量没有到时，发送一个事件，其事件值为当前事件跨度内发出的时间的事件值组成的数组，重置时间跨度；当时间跨度内发生的事件超过了最大发生数量时，立即发送一个事件值为这些事件值所组成的数组的事件，重置时间跨度：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_38.png?raw=true)

如图，时间跨度为4，最大事件值为2。最开始什么时间也没有发生，所以发送一个 0 个元素的数组。之后某个时刻发出了三个事件，所以将前两个事件值合成为一个数组发送，剩余一个事件，重置时间跨度。后面事件数量没有到最大值，但是时间跨度到了，所以也发送一个事件的数组，重置时间跨度。最后又没有事件发生，发送 0 个元素的数组的事件。

```swift
let interval = Observable<String>.interval(1, scheduler: MainScheduler.instance)
					.buffer(timeSpan: 4, count: 2, scheduler: MainScheduler.instance)

_ = interval.subscribe(onNext: $0)

interval.onNext("🐈")
interval.onNext("🐈")
interval.onNext("🐈")
```

#### 时间平移操作符

##### delaySubscription

延迟订阅，在正式订阅前发生的事件都会被忽略：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_39.png?raw=true)

如图，在 1 时刻开始订阅，由于延迟了 1.5s，所以前两个事件被忽略了。

```swift
Observable.of(1, 2, 3, 4, 5)
	.delaySubscription(RxTimeInterval(delayInSeconds), scheduler: MainSchedular.instance)
	.subscribe{ print($0.element ?? $0) }
```

##### delay

delay 则将序列中的所有事件延迟执行，所以并不会忽略掉事件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_40.png?raw=true)

差别就在于，上面的忽略了1，2，而这里则仍是从事件 1 开始。

```swift
Observable.of(1, 2, 3, 4, 5)
	.delay(RxTimeInterval(delayInSeconds), scheduler: MainSchedular.instance)
	.subscribe{ print($0.element ?? $0) }
```

#### 定时操作符

##### interval

Rx 中的定时不需要使用 NSTimer，也不需要使用 DispatchSource。interval 的使用非常简单，比如一个1s的定时器：

```swift
Observable<Int>.interval(1, scheduler: MainScheduler.instance)
```

事件值默认是从 0 开始发送，依次递增。如果你不想要从 0 开始，可以使用 map。不过一般我们不需要使用这个事件值。

##### timer

`timer(_:period:scheduler:)` 和 interval 的区别在于，可以设置一个重复次数 period。如果不设置，默认只执行一次：

```swift
Observable<Int>.timer(3, period:3, scheduler: MainScheduler.instance)
```

## 使用 RxCocoa 的应用

###开始使用 RxCocoa

这一章节主要是一个例子。讲的是如何通过 Rx 将天气信息展示出来。

#### 初印象

打开 `UITextField+Rx.swift`，其中只有一个类型为 `ControlProperty<String?>` 的 `text` 属性：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_42.png?raw=true)

`ControlProperty` 是一个特殊的 subject，前面也提到过了，和 UI 控件交互的属性都是 subject 类型的。更具体一点，Rxcocoa 中，和 UI 交互的属性都是 `ControlProperty` 类型。

再打开 `UILabel+Rx.swift`，它的两个属性都是 `UIBindingObserver` 类型：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_43.png?raw=true) 

`UIBindingObserver` 是一个 Observer。它就是在 Subject 与 Observer 一节中提到的特殊的 Observer。它有一个 `binding` 属性，用来保存默认的事件处理方法，在初始化的时候设置。

前面也提到过，这种 Observer，没有自定义的事件处理方法，所以只能做诸如设置 UILabel 的 text 等无须 UI 交互的事。

#### 使用 RxCocoa 的 UIKit

`ApiController.swift` 用来提供数据，在其中定义了一个方法 `currentWeather(city:)`，用来提供默认的天气数据：

```swift
func currentWeather(city: String) -> Observable<Weather> {
    return Observable.just(
    	Weather(
        	cityName: city,
        	temperature: 20,
        	humidity: 90)
    )
}
```

这个方法接受一个参数：城市city，返回一个只有一个元素的 Observable，事件值类型为天气Weather类型。

接下来就可以在 ViewController 中订阅了。在 `viewDidLoad` 中设置:

```swift
ApiController.shared.currentWeather(city: "Shanghai")
	.observeOn(MainScheduler.instance)
	.subscribe( onNext: { data in
		self.tempLabel.text = "\(data.temperature)°C"
		self.cityNameLabel.text = data.cityName
		self.humidityLabel.text = "\(data.humidity)%"
	}).addDisposableTo(bag)
```

其中 `shared` 用来获取单例，`observeOn(MainScheduler.instance)` 用来将操作限制在主线程中，因为这是在操作 UI。

那么如何获取到 city 的呢？我们可以有一个 textfield `searchCityName` 用来接受用户的输入。RxCocoa 通过协议拓展的方式为大多数 UIKit 的成员都添加了 `rx` 的相关属性。比如 textfield，我们可以输入 `searchCityName.rx.` 就可以查看到很多可以使用的 Rx 相关的属性和方法。这里我们要使用的是 textfield 的 `text` 属性：

```swift
searchCityName.rx.text
	.filter { ($0 ?? "").characters.count > 0}
	.flatMapLatest { text in
		return ApiController.shared.currentWeather(city: text ?? "Error")
                    .catchErrorJustReturn(ApiController.Weather.empty)
	}.observeOn(MainScheduler.instance)
	.subscribe(
    	...
    ).addDisposableTo(bag)
```

这段代码做了什么，这段代码将 textfield 的输入框和处理逻辑绑定了起来，输入框一有输入，那么就触发事件。先将空输入过滤掉，然后将 textfield 中输入的值通过 flatMapLatest 再转换为新的 Observable，即 city 转 Weather 的过程。其中如果 `currentWeather` 的转换产生了错误，就通过 `catchErrorJustReturn` 使用默认的事件值，空 Weather。

通过这个例子，可以再复习一下 flatMapLatest 和 flatMap 的区别。这个例子中 flatMapLatest 和 flatMap 是一样的，因为 `currentWeather` 返回的 Observable 只有一个元素，所以只订阅最后的 Observable 和 订阅每一个 Observable 没有区别，因为 Observable 不会再有事件发生了。但是，如果这里 `currentWeather` 是一个异步的操作，比如去网上拉取数据，那就必须使用 flatMapLatest 了。试想一下，在上一个网络请求还没有完成的时候，输入变化，这样又触发了一次网络请求。使用 flatMap 就会在每一次网络请求结束时都做响应，但这个并不符合逻辑，应该只响应最后一次网络请求，否则会产生请求覆盖。所以一定要用 flatMapLatest。 

以上的数据流程如图所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_41r.png?raw=true)

#### 绑定 Observable

绑定是一种单向的数据流。RxCocoa 中的绑定通过 `bindTo(_:)` 实现。被绑定的对象的类型必须是 `ObserverType`，也就是 Observer 对象。

> 书上这个地方我认为是有问题的，书上的原话是：
>
> The fundamental function of binding is bindTo(_:). To bind an observable to another entity, the receiver must conform to ObserverType. This entity has been explained in previous chapters: it’s a Subject which can process values, but can also be written to manually.
>
> 书上说 ObserverType 就是一种 Subject，这样说是没有道理的。Subject 的范围要比 ObserverType 大，Subject 可以被订阅，提供自定义事件处理方法，但是 `bindTo()` 并不需要，它只需要有一个默认的事件处理方法即可。所以我认为书上这么说至少是不严谨的。

`bindTo(_:)` 是一种特殊的 subscribe，也是在 Observable 触发事件的时候，调用 Observer 的事件处理方法。但是原本需要在 subscribe 的时候传入的事件处理方法现在要求被绑定的 Observer 自己提供。所以像 `UIBindingObserver` 这样有默认事件处理方法的 Observer 或者一个已经被订阅过的 Subject 都是可以作为绑定对象的。













