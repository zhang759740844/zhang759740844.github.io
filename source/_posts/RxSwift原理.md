title: RxSwift 的一些原理解析
date: 2017/11/14 10:07:12  
categories: iOS
tags:
	- 源码解析
	- RxSwift

------

前面两篇讲了 RxSwift 以及 RxCocoa 的使用，这篇探究下 RxSwift 的实现原理。

<!--more-->

这篇将从四个方面探究 RxSwift 的实现原理。包括 

1. rx 的命名空间如何产生的
2. 如何创建 Observable 以及订阅
3. Subject 是如何实现的
4. 如何实现调用 delegate 方法的
5. ​

## rx 的命名空间如何产生

我们都知道，RxCocoa 为我们的基本控件添加了 rx 的命名空间，然后定义了 rx 相关的属性。那么这是怎么做到的呢？

我们点进 rx 的定义：

```swift
/// A type that has reactive extensions.
public protocol ReactiveCompatible {
    /// Extended type
    associatedtype CompatibleType

    /// Reactive extensions.
    static var rx: Reactive<CompatibleType>.Type { get set }

    /// Reactive extensions.
    var rx: Reactive<CompatibleType> { get set }
}
```

可以看到，rx 是定义在这个叫 `ReactiveCompatible` 的协议中的。rx 是 `Reactive` 类型。在协议拓展中，实现了这个协议的默认实现，即返回 `Reactive` 类型，或者返回该类型实例。其中，它将上面的关联类型 `associatedtype`，设置为调用者自身：

```swift
extension ReactiveCompatible {
    /// Reactive extensions.
    public static var rx: Reactive<Self>.Type {
        get {
            return Reactive<Self>.self
        }
        set {
            // this enables using Reactive to "mutate" base type
        }
    }

    /// Reactive extensions.
    public var rx: Reactive<Self> {
        get {
            return Reactive(self)
        }
        set {
            // this enables using Reactive to "mutate" base object
        }
    }
}
```

接着看 `Reactive` 的定义，它是一个结构体，接受一个泛型参数。这个结构体很简单，只有一个泛型类型的属性 `base`。上面方法在创建 `Reactive` 实例的时候，将调用者设置为其 `base` 属性，比如调用 `button.rx`，那么就会将 button 实例设置为其 `base`：

```swift
public struct Reactive<Base> {
    /// Base object to extend.
    public let base: Base

    /// Creates extensions with base object.
    ///
    /// - parameter base: Base object.
    public init(_ base: Base) {
        self.base = base
    }
}
```

因为所有的控件都是继承于 `NSObject`，所以要让所有控件都拥有 rx 这个属性，只需要让 `NSObject` 实现 `ReactiveCompatible` 即可：

```swift
/// Extend NSObject with `rx` proxy.
extension NSObject: ReactiveCompatible { }
```

这样，rx 就和基本控件关联了起来。但是 `Reactive` 这个结构体这么简单，它是怎么拓展出这么多 rx 属性的呢？我们以 UIButton 为例。打开 RxCocoa 中的 `UIButton+Rx.swift`，这是一个对于 button 的 rx 拓展：

```swift
extension Reactive where Base: UIButton {
    
    /// Reactive wrapper for `TouchUpInside` control event.
    public var tap: ControlEvent<Void> {
        return controlEvent(.touchUpInside)
    }
}
```

这里通过模式匹配 where 将泛型 `Base` 限制为 `UIButton`，为 `UIButton` 添加特定的属性。这里只列举一个 `tap` 属性，其实还有一个 `title` 方法。我们先不管这个 `ControlEvent` 是啥，这个后面会讲。至此，控件就获得了 rx 相关的属性了。

> 总之，整个过程就是，通过让 `NSObject` 实现一个协议，使控件获得 rx 属性，然后在 rx 类型的 extension 中，以模式匹配的方式为不同控件的 rx 添加相应的属性。过程如图所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_prin_1.png?raw=true)

## 如何创建 Observable 以及订阅

### 概述

我们知道通过 `create`，`just`，`of` 等方法可以创建一个 Observable。那么这是如何做到的呢？我们以 `create` 创建的订阅为例。这一创建及订阅过程还是比较复杂的，涉及的类以及相互关系很多。所以我先把所有相关类的关系图展示出来：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_prin_2.png?raw=true)

整个过程涉及四个部分：Observable，Observer，Sink，Disposer。

Observable 主要就是保存了事件队列；Observer 主要保存了事件队列的处理方法；Sink 连接前两者，由它来让 Observer 响应 Observable 的事件；Disposer 保存了 Sink 的实例，以便回收。

### 创建

首先我们来看 step 1，调用的 `create` 方法存在于  `Observable` 的一个拓展中，包含了各种 Observable 的创建方法。`create` 方法创建了一个匿名的 Observable `AnonymousObservable`，并将包含各种事件的闭包传入：

```swift
extension Observable {
    public static func create(_ subscribe: @escaping (AnyObserver<E>) -> Disposable) -> Observable<E> {
        return AnonymousObservable(subscribe)
    }
}
```

来到 step 2， `AnonymousObservable` 初始化的过程很简单，就是将传入的包含事件的闭包保存起来。另外该类还提供了一个重写的 `run` 方法：

```swift
final class AnonymousObservable<Element> : Producer<Element> {
	typealias SubscribeHandler = (AnyObserver<Element>) -> Disposable

    let _subscribeHandler: SubscribeHandler

    init(_ subscribeHandler: @escaping SubscribeHandler) {
        _subscribeHandler = subscribeHandler
    }
  
    override func run<O : ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element {
        let sink = AnonymousObservableSink(observer: observer, cancel: cancel)
        let subscription = sink.run(self)
        return (sink: sink, subscription: subscription)
    }
}
```

到这里，创建 Observable 的过程就结束了。简单来说就是内部创建了一个匿名的 Observable，然后将事件闭包保存。下面是订阅过程。

### 订阅

来到 step 3，订阅的时候，我们调用的是 `ObservableType` 提供的订阅方法。`ObservableType` 是一个协议，由 `Observable` 实现。这个方法中创建了一个匿名的 Observer `AnonymousObserver`，也就是 step 4，把事件处理方法的闭包传入。然后就是观察者订阅事件队列，即 step 5：

```swift
extension ObservableType {
    public func subscribe(_ on: @escaping (Event<E>) -> Void)
        -> Disposable {
        let observer = AnonymousObserver { e in
            on(e)
        }
        return self.subscribeSafe(observer)
    }
  
    fileprivate func subscribeSafe<O: ObserverType>(_ observer: O) -> Disposable where O.E == E {
        return self.asObservable().subscribe(observer)
    }
}
```

step 5 的方法中，调用的是入参为 Observer 的 `subscribe` 方法。在 `Observable` 类中定义了该方法。不过这个方法并没有具体实现，它要求其子类实现。否则就会抛出异常：

```swift
public class Observable<Element> : ObservableType {
    /// Type of elements in sequence.
    public typealias E = Element
  
    ...
    public func subscribe<O: ObserverType>(_ observer: O) -> Disposable where O.E == E {
        abstractMethod()
    }
    ...
  
}
```

`Observable` 在本例的子类是 `AnonymousObservable`，不过不是直接子类，这两个类中还夹着一个 `Producer` 类。所以订阅的过程 step 6 在 `Producer` 中：

```swift
class Producer<Element> : Observable<Element> {
    override func subscribe<O : ObserverType>(_ observer: O) -> Disposable where O.E == Element {
        if !CurrentThreadScheduler.isScheduleRequired {
            // The returned disposable needs to release all references once it was disposed.
            let disposer = SinkDisposer()
            let sinkAndSubscription = run(observer, cancel: disposer)
            disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription)

            return disposer
        }
        else {
            return CurrentThreadScheduler.instance.schedule(()) { _ in
                let disposer = SinkDisposer()
                let sinkAndSubscription = self.run(observer, cancel: disposer)
                disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription)

                return disposer
            }
        }
    }
    
    func run<O : ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element {
        abstractMethod()
    }
}
```

step 6 的方法先创建了一个 `SinkDisposer`，即 step 7。要留意这个 `SinkDisposer`，因为最后我们加入 disposebag 的，就是这个 disposer 实例。

下一步调用 `run` 方法。我们看到上面的代码中，`run` 方法也是一个类似的抽象方法。那么 step 8 中，我们就要到它的子类 `AnonymousObservable` 中找，这个上面也提到过。

### 执行

来看一下这个 `run` 方法的实现。它接受刚才创建的匿名 Observer `AnonymousObserver`，以及 `SinkDisposer`：

```swift
final class AnonymousObservable<Element> : Producer<Element> {
	...
    override func run<O : ObserverType>(_ observer: O, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where O.E == Element {
        let sink = AnonymousObservableSink(observer: observer, cancel: cancel)
        let subscription = sink.run(self)
        return (sink: sink, subscription: subscription)
    }
}
```

第一步 step 9，创建了一个 `AnonymousObservableSink` 实例。这个 `AnonymousObservableSink` 即实现了 `ObserverType` 协议，又实现了 `Disposable` 协议，所以它既是一个 Observer，又是一个 Disposer。在它的初始化方法中，调用了父类 `Sink` 的初始化方法，也就是 step 10：

```swift
class Sink<O : ObserverType> : Disposable {
    fileprivate let _observer: O
    fileprivate let _cancel: Cancelable
    fileprivate var _disposed: Bool

    init(observer: O, cancel: Cancelable) {
        _observer = observer
        _cancel = cancel
        _disposed = false
    }
}
```

无关代码删除后，可以看到，`Sink` 中包含了 Observer，也包含了最终的 Disposer 实例。

回到 step 8，在创建完 `AnonymousSink` 后，调用其 `run` 方法，并将返回值赋给 `subscription`。我们来看一下 step 11，这个 `run` 方法做了什么：

```swift
final class AnonymousObservableSink<O: ObserverType> : Sink<O>, ObserverType {
    typealias E = O.E
    typealias Parent = AnonymousObservable<E>

    func on(_ event: Event<E>) {
        switch event {
        case .next:
            if _isStopped == 1 {
                return
            }
            forwardOn(event)
        case .error, .completed:
            if AtomicCompareAndSwap(0, 1, &_isStopped) {
                forwardOn(event)
                dispose()
            }
        }
    }

    func run(_ parent: Parent) -> Disposable {
        return parent._subscribeHandler(AnyObserver(self))
    }
}
```

这个方法中总算执行了之前保存在 `AnonymousObservable` 对象中的事件闭包。我们可以会奇怪为什么它要用 `AnyObserver(self)` 来代替之前创建的 `AnonymousObserver`，这是为了擦除类型，反正现在需要的就是一个有事件响应方法的 Observer。

执行事件闭包发出事件，也就是调用 Observer 相应的响应方法。`AnyObserver` 稍微做了个转换，响应方法调用的是上面 `AnonymousObservableSink` 的 `on` 方法。这一步是 step 12。

在 `on` 方法中，将事件进行转发，来到 step 13， `sink` 中的 `forwardOn` 中：

```swift
class Sink<O : ObserverType> : Disposable {
   	...
    final func forwardOn(_ event: Event<O.E>) {
        if _disposed {
            return
        }
        _observer.on(event)
    }
}   
```

这里的 `_observer` 应该很熟悉，就是刚才保存的 `AnonymousObserver`。不过 `AnonymousObserver` 这个类很简陋，并没有 `on` 方法啊：

```swift
final class AnonymousObserver<ElementType> : ObserverBase<ElementType> {
    typealias Element = ElementType
    
    typealias EventHandler = (Event<Element>) -> Void
    
    private let _eventHandler : EventHandler
    
    init(_ eventHandler: @escaping EventHandler) {
#if TRACE_RESOURCES
        let _ = Resources.incrementTotal()
#endif
        _eventHandler = eventHandler
    }

    override func onCore(_ event: Event<Element>) {
        return _eventHandler(event)
    }
    
#if TRACE_RESOURCES
    deinit {
        let _ = Resources.decrementTotal()
    }
#endif
}
```

到父类 `ObserverBase` 中查看：

```swift
class ObserverBase<ElementType> : Disposable, ObserverType {
    typealias E = ElementType

    private var _isStopped: AtomicInt = 0

    func on(_ event: Event<E>) {
        switch event {
        case .next:
            if _isStopped == 0 {
                onCore(event)
            }
        case .error, .completed:

            if !AtomicCompareAndSwap(0, 1, &_isStopped) {
                return
            }

            onCore(event)
        }
    }

    func onCore(_ event: Event<E>) {
        abstractMethod()
    }
}
```

`ObserverBase` 的 `on` 中，最终还是走了子类的 `onCore`。我的天，这一通折腾。处理完事件，最终返回给前面的 `subscription` 的就是在事件队列中创建的 Disposer。

### 回收

现在事件都已经处理完了，回到 step 6，继续往下走，执行 step 16：

```swift
fileprivate final class SinkDisposer: Cancelable {
    private var _state: AtomicInt = 0
    private var _sink: Disposable? = nil
    private var _subscription: Disposable? = nil
  
    func setSinkAndSubscription(sink: Disposable, subscription: Disposable) {
        _sink = sink
        _subscription = subscription

        let previousState = AtomicOr(DisposeState.sinkAndSubscriptionSet.rawValue, &_state)
        if (previousState & DisposeStateInt32.sinkAndSubscriptionSet.rawValue) != 0 {
            rxFatalError("Sink and subscription were already set")
        }

        if (previousState & DisposeStateInt32.disposed.rawValue) != 0 {
            sink.dispose()
            subscription.dispose()
            _sink = nil
            _subscription = nil
        }
    }
}
```

它把 `AnonymousObservableSink` 以及 `subscription` 保存了起来。也就是说最终返回的 `SinkDisposer` 中会包含两个 Disposer。如果执行了 `dispose()`，那么这两个 Disposer 都会执行各自的 `dispose()`，并且释放。这样就是为什么我们在 `subscribeHandle` 中随便创建的一个 Disposer，也能起作用的原因。

`SinkDisposer` 引用了 `AnonymousObservableSink` 以及 `subscription`。`AnonymousObservableSink` 引用了 `SinkDisposer` 以及 `AnonymousObserver`。这是一个引用循环，为的是各自调用各自的 `dispose()` 的时候，都能触发对方的 `dispose()` ，将对方也回收。

> 其实在没看源码前大概的想法也是，在 Observable 内部创建保存一个不被外界知道的 Observer，进行事件处理。实际上呢是添加了一个 `Sink` 类转梦用来保存 Observer，并且进行处理。
>
> 明白了 Disposer 的引用关系，也就明白了如何将订阅回收的了。另外，将 `Sink` 设计成 `Disposable` 也使得回收更加方便



## Subject 的创建与订阅



