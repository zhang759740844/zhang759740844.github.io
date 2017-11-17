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

Subject 是 hot observable，和上面的区别在于，可以自己定义事件触发的时机。那么这个不同是如何体现出来的呢？下面来瞧瞧 Variable 的原理。

### Variable

Variable 是 BehaviorSubject 的封装。来看一下 Variable 中的属性：

```swift
public final class Variable<Element> {

    public typealias E = Element
    
    private let _subject: BehaviorSubject<Element>
    
    private var _lock = SpinLock()
 
    // state
    private var _value: E
  
  	...
}
```

可以看到，很简单，一个 `BehaviorSubject`，一个同步锁，以及一个 `value` 属性。Variable 和 BehaviorSubject 相比，唯一的不同就是 Variable 有一个 `value` 属性， 并在其 set 方法中，自动触发了 next 事件。

现在看一下初始化方法，以及 `value` 的 get set 方法：

```swift
public final class Variable<Element> {

	...
    /// Gets or sets current value of variable.
    ///
    /// Whenever a new value is set, all the observers are notified of the change.
    ///
    /// Even if the newly set value is same as the old value, observers are still notified for change.
    public var value: E {
        get {
            _lock.lock(); defer { _lock.unlock() }
            return _value
        }
        set(newValue) {
            _lock.lock()
            _value = newValue
            _lock.unlock()

            _subject.on(.next(newValue))
        }
    }
    
    /// Initializes variable with initial value.
    ///
    /// - parameter value: Initial variable value.
    public init(_ value: Element) {
        _value = value
        _subject = BehaviorSubject(value: value)
    }

  	...
}

```

`init` 方法就是初始化了一个 `BehaviorSubject` 以及设置了 `value` 的初始值。`value` 的 setter 方法就是更新了 `value` 值，然后触发了 BehaviorSubject 的事件。

这里学习一个加锁解锁的方法，在 getter 方法中通过 `defer` 在 return 之后执行解锁操作。

好了这就是 Variable 的全部了。由于 Variable 并不是继承于 Observable，所以你甚至都不能订阅，只能通过 `asObservable` 方法，返回其中的 `BehaviorSubject` 属性。下面来看下 BehaviorSubject 是如何实现的。

### BehaviorSubject

我们使用 BehaviorSubject 进行订阅的时候，一开始的步骤还是一样的：`Observable` 先创建一个 `AnonymousObserver`，将事件处理方法设置给它的 `eventHandler` 属性。所有的 Observable 订阅，都会进行这样的方法，我们可以将这部分成为订阅的公共部分。

然后到上面的 step 5 开始有区别了。在 `create` 中，由于继承关系调用的是 `Producer` 的 `subscribe`；而 BehaviorSubject 中也实现了自己的 `subscribe` 方法，我们可以称这部分为订阅的私有部分(这两个名字都是我随便起的)。

在 `Producer` 的订阅私有部分，进行的是 step7 以及 step 8，也就是创建一个 `SinkDisposer` 和执行 `run` 方法(执行事件闭包中的事件)，这也就是 `create` 能够自己发出事件的原因。BehaviorSubject 的订阅私有部分做的是将刚创建的 `AnonymousObserver` 保存起来，然后以当前 `value` 值作为事件值，发出一个事件，这就是 BehaviorSubject 订阅就会触发事件的原因。

下面来看代码，首先看看 `BehaviorSubject` 的属性:

```swift
public final class BehaviorSubject<Element>
    : Observable<Element>
    , SubjectType
    , ObserverType
    , SynchronizedUnsubscribeType
    , Disposable {
    public typealias SubjectObserverType = BehaviorSubject<Element>
    typealias DisposeKey = Bag<AnyObserver<Element>>.KeyType
    
    /// Indicates whether the subject has any observers
    public var hasObservers: Bool {
        _lock.lock()
        let value = _observers.count > 0
        _lock.unlock()
        return value
    }
    
    let _lock = RecursiveLock()
    
    // state
    private var _isDisposed = false
    private var _value: Element
    private var _observers = Bag<(Event<Element>) -> ()>()
    private var _stoppedEvent: Event<Element>?

    /// Indicates whether the subject has been disposed.
    public var isDisposed: Bool {
        return _isDisposed
    }
}
```

BehaviorSubject 的属性就这么多，基本上看到都是能一眼看明白的。`_observers` 就是创建的 `AnonymousObserver`，它的类型 `Bag` 是一个字典的封装，它可以为不同的 `AnonymousObserver` 提供不同的键，这样能在取消订阅的时候便于回收响应 `AnonymousObserver`。另外，`_stoppedEvent` 也是需要解释一下的，就是在 `completed` 以及 `error` 事件发生后，保存这个 event，以后当有其他事件触发的时候，发出的是这个 event 的事件。

接下来看一下订阅的私有部分的相关代码：

```swift
    /// Subscribes an observer to the subject.
    ///
    /// - parameter observer: Observer to subscribe to the subject.
    /// - returns: Disposable object that can be used to unsubscribe the observer from the subject.
    public override func subscribe<O : ObserverType>(_ observer: O) -> Disposable where O.E == Element {
        _lock.lock()
        let subscription = _synchronized_subscribe(observer)
        _lock.unlock()
        return subscription
    }

    func _synchronized_subscribe<O : ObserverType>(_ observer: O) -> Disposable where O.E == E {
        if _isDisposed {
            observer.on(.error(RxError.disposed(object: self)))
            return Disposables.create()
        }
        
        if let stoppedEvent = _stoppedEvent {
            observer.on(stoppedEvent)
            return Disposables.create()
        }
        
        let key = _observers.insert(observer.on)
        observer.on(.next(_value))
    
        return SubscriptionDisposable(owner: self, key: key)
    }

```

需要把这部分加上锁，防止在将 Observer 加入字典的时候，产生错误。然后就像上面讲的一样，先看看有没有 `stoppedEvent`，有的话执行该 event，没有的话，将 Observer 加入字典，然后触发这个 Observer 的事件。

最后，返回了一个 `SubscriptionDisposable`，它是一个非常简单的结构体。保存了 self，防止 BehaviorSubject 回收，保存了`key`，用于回收对应的 Observer。

`on` 方法是因为 BehaviorSubject 也实现了 `ObserverType` 协议，再看一下 `on` 做了什么：

```swift
    /// Notifies all subscribed observers about next event.
    ///
    /// - parameter event: Event to send to the observers.
    public func on(_ event: Event<E>) {
        _lock.lock()
        dispatch(_synchronized_on(event), event)
        _lock.unlock()
    }

    func _synchronized_on(_ event: Event<E>) -> Bag<(Event<Element>) -> ()> {
        if _stoppedEvent != nil || _isDisposed {
            return Bag()
        }
        
        switch event {
        case .next(let value):
            _value = value
        case .error, .completed:
            _stoppedEvent = event
        }
        
        return _observers
    }
```

同样的也是将这个过程锁上，避免在执行的时候，新的 Observer 被加入字典中，导致新的 Observer 也响应事件。这里的 `dispatch` 方法主要就是通过一个 for 循环，将 `_observers` 中的每个 Observer 都触发 `event` 事件。下面同步的 `on` 方法也就不多解释了。

这就是 BehaviorSubject 的大致实现过程。

> 其实 Subject 就是典型的观察者模式。`create` 把类大概分成了四个部分。Subject 则结合了 Observable 和 Sink 做的事情。它在内部为每个观察者创建了匿名的 Observer 并保存，然后可以自行调用 `onNext` 调用这些 Observer 的事件处理方法。将创建事件队列和执行事件队列放在了一起。

## 代理转发

前面的文章说到，会详细叙述一下代理转发的流程，那么现在来看一看这方面的源码。我们在 RxCocoa 中找一个需要实现代理的控件类，看一下它的解决方式。比如 UITabBarController。

### 控件的 rx 拓展

关于代理转发需要我们创建的只有两个类，一个是 UITabBarController 的 rx 拓展类，一个是 UITabBarControllerDelegate 的实现类 DelegateProxy。先来看一下 UITabBarController 的 rx 拓展类：

```swift
extension Reactive where Base: UITabBarController {
    /// Reactive wrapper for `delegate`.
    ///
    /// For more information take a look at `DelegateProxyType` protocol documentation.
    public var delegate: DelegateProxy {
        return RxTabBarControllerDelegateProxy.proxyForObject(base)
    }
    
    /// Reactive wrapper for `delegate` message `tabBarController:didSelect:`.
    public var didSelect: ControlEvent<UIViewController> {
        let source = delegate.methodInvoked(#selector(UITabBarControllerDelegate.tabBarController(_:didSelect:)))
            .map { a in
                return try castOrThrow(UIViewController.self, a[1])
        }
        
        return ControlEvent(events: source)
    }
}
```

我们可以看到，这个类中先定义了一个 `DelegateProxy` 类型的 `delegate`，这就是上面所说的 UITabBarControllerDelegate 的实现类。另外定义了一个 `didSelect` 属性，它的类型是 `ControlEvent`，其实就是一个 Observable。我们知道 UITabBarControllerDelegate 中有一个代理方法就是 `didSelect`，所以这里定义的 Observable 其实就是用来替代传统的代理方法的。

`delegate` 是一个只读属性，它只是调用了 DelegateProxy 的 `proxyForObject` 方法。DelegateProxy 的继承关系比较复杂，我们先看一下关系图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_prin_3.png?raw=true)

> 强调一下，`delegate` 是 rx 命名空间下的，不会影响到控件本生的 delegate 属性。

### 替换代理类

可以看到 `proxyForObject` 是 `DelegateProxyType` 协议中的方法，它在协议的拓展中提供了默认的实现：

```swift
    public static func proxyForObject(_ object: AnyObject) -> Self {
        MainScheduler.ensureExecutingOnScheduler()

        let maybeProxy = Self.assignedProxyFor(object) as? Self

        let proxy: Self
        if let existingProxy = maybeProxy {
            proxy = existingProxy
        }
        else {
            proxy = Self.createProxyForObject(object) as! Self
            Self.assignProxy(proxy, toObject: object)
            assert(Self.assignedProxyFor(object) === proxy)
        }

        let currentDelegate: AnyObject? = Self.currentDelegateFor(object)

        if currentDelegate !== proxy {
            proxy.setForwardToDelegate(currentDelegate, retainDelegate: false)
            assert(proxy.forwardToDelegate() === currentDelegate)
            Self.setCurrentDelegate(proxy, toObject: object)
            assert(Self.currentDelegateFor(object) === proxy)
            assert(proxy.forwardToDelegate() === currentDelegate)
        }
        
        return proxy
    }
```

#### 创建 proxy

首先通过 `assignedPorxyFor` 这个类方法方法，获取指定控件(object)的 DelegateProxy 的实例。这个方法在 `DelegateProxyType` 协议中定义，在 `DelegateType` 中实现：

```swift
    open class func assignedProxyFor(_ object: AnyObject) -> AnyObject? {
        let maybeDelegate = objc_getAssociatedObject(object, self.delegateAssociatedObjectTag())
        return castOptionalOrFatalError(maybeDelegate.map { $0 as AnyObject })
    }
```

它通过关联对象的方式，从指定控件中取出。如果这个 proxy 存在，就说明之前已经设置过了，如果没有，那么通过 `createProxyForObject` 方法创建。这个创建方法同样是在 `DelegateProxyType` 中定义，在 `DelegateProxy` 中实现。它做的很简单，就是实例化自身：

```swift
    open class func createProxyForObject(_ object: AnyObject) -> AnyObject {
        return self.init(parentObject: object)
    }
```

然后就是将自身与控件关联。总的来说，上面就是要创建一个 DelegateProxy，然后保存为相应控件的属性的一个属性。

#### 获取当前代理对象

接下来通过 `currentDelegateFor` 获取当前控件的实际代理类。这个方法是需要自己在 DelegtateProxy 子类中实现的，因为虽然用做代理，但是可能叫法不同，比如 tableView 的数据源代理就叫做 dataSource。TabBarController 的 DelegateProxy 中仅有的两个方法就是取出和设置自身的代理：

```swift
public class RxTabBarControllerDelegateProxy
    : DelegateProxy
    , UITabBarControllerDelegate
    , DelegateProxyType {
    
    /// For more information take a look at `DelegateProxyType`.
    public class func currentDelegateFor(_ object: AnyObject) -> AnyObject? {
        let tabBarController: UITabBarController = castOrFatalError(object)
        return tabBarController.delegate
    }
    
    /// For more information take a look at `DelegateProxyType`.
    public class func setCurrentDelegate(_ delegate: AnyObject?, toObject object: AnyObject) {
        let tabBarController: UITabBarController = castOrFatalError(object)
        tabBarController.delegate = castOptionalOrFatalError(delegate)
    }
}
```

> 就是说，上面代码中的 delegate 在一些情况下，可能不叫 delegate，所以没法写成一个公共的方法，必须要每个 DelegateProxy 都自己根据情况实现。

#### 用 proxy 替换代理对象

拿到当前的代理对象后，通过一个 if 判断当前代理对象是否是上面得到的 proxy。如果是，说明之前已经设置过了，直接将 proxy 返回即可；如果不是，那就要用 proxy 将代理对象替换掉。这个过程由 `setForwardToDelegate` 方法完成，实现还是在 `DelegateProxy` 中：

```swift
open func setForwardToDelegate(_ delegate: AnyObject?, retainDelegate: Bool) {
    self._setForward(toDelegate: delegate, retainDelegate: retainDelegate)
}
```

看到这个带下划线的方法，应该能猜到，这就是调用的 OC 的方法了。在 `_RXDelegateProxy` 这个 OC 类中，定义了一个 `__forwardToDelegate` 属性。这个属性是弱引用的，它用来保存正真的代理对象：

```objc
-(void)_setForwardToDelegate:(id __nullable)forwardToDelegate retainDelegate:(BOOL)retainDelegate {
    __forwardToDelegate = forwardToDelegate;
    if (retainDelegate) {
        self.strongForwardDelegate = forwardToDelegate;
    }
    else {
        self.strongForwardDelegate = nil;
    }
}
```

我们可以看到有一个 Bool 类型的  `retainDelegate` 入参。它来决定是否将弱引用变成强引用，不过一般情况，我们并不需要强引用。总之，将原本的代理对象安顿好了之后，就通过 `setCurrentDelegate` 将 proxy 设置为代理对象，并且返回了。

### 获取方法的 Observable

回到我们看过的控件的 rx 拓展中，前面说到 `didSelect` 这个计算属性，返回一个 Observable，拦截代理方法的工作也是由它自己完成的：

```swift
public var didSelect: ControlEvent<UIViewController> {
    let source = delegate.methodInvoked(#selector(UITabBarControllerDelegate.tabBarController(_:didSelect:)))
        .map { a in
            return try castOrThrow(UIViewController.self, a[1])
    }
    
    return ControlEvent(events: source)
}
```

这里的主要方法就是这个 `methodInvoked`，它在 `DelegateProxy` 中。（另外插一句，这里的 `a` 代表的是代理方法的各个入参，`a[0]` 表示该 tabBarController，`a[1]` 表示的就是 `didSelect:` 对应的入参。）我们继续看下去：

```swift
open func methodInvoked(_ selector: Selector) -> Observable<[Any]> {
    checkSelectorIsObservable(selector)

    let subject = methodInvokedForSelector[selector]

    if let subject = subject {
        return subject
    }
    else {
        let subject = PublishSubject<[Any]>()
        methodInvokedForSelector[selector] = subject
        return subject
    }
}
```

首先通过 `checkSelectorIsObservable` 检查方法是否已通过非 rx 的代理方式实现了。这个方法不是太重要。之后从 `methodInvokedForSelector` 中取出 selector 对应的 subject 对象。这个属性是一个字典，以 selector 为键，以 subject 为值。如果存在 subject 就说明，已经创建了 selector 对应的 Observable 对象了，直接返回即可；如果没有，那么就创建一个 PublishSubject，并且将其存入 `methodInvokedForSelecotr` 字典中去，最后返回这个 subject。我们就可以使用这个 subject 来进行订阅了。

> 所以获取方法的 Observable 的过程就是从一个字典中取出方法对应 Observable 的过程。

### 拦截方法

那么调用代理方法的过程是如何转变为触发事件的呢？这里就要用到 OC runtime 中的消息转发。在 `_RXDelegateProxy` 中实现了消息转发的方法 `forwardInvocation`。我们将 DelegateProxy 设置为代理类，但是我们一直没有实现代理方法。所以系统企图调用代理方法的时候发现没有代理方法，就执行消息转发的方法：

```objc
-(void)forwardInvocation:(NSInvocation *)anInvocation {
    BOOL isVoid = RX_is_method_signature_void(anInvocation.methodSignature);
    NSArray *arguments = nil;
    if (isVoid) {
        arguments = RX_extract_arguments(anInvocation);
        [self _sentMessage:anInvocation.selector withArguments:arguments];
    }
    
    if (self._forwardToDelegate && [self._forwardToDelegate respondsToSelector:anInvocation.selector]) {
        [anInvocation invokeWithTarget:self._forwardToDelegate];
    }

    if (isVoid) {
        [self _methodInvoked:anInvocation.selector withArguments:arguments];
    }
}
```

首先检查一下代理方法是不是有返回值。为什么要检查这个呢？因为我们订阅 Observable 的处理方法是不会有返回值的，但是一般的代理方法不同，代理方法有一些是要返回处理完的数据的。对于有返回值的代理方法，我们是无法以 Observable 的方式订阅的，只能老老实实的写回调方法。

下面是三个 if 判断。中间的 if 我们看到了熟悉的 `_forwardToDelegate` 属性。我们也说过了，这个属性是用来临时保存原本正真的代理对象的。这个 if 的作用就是，检查一下正真的代理对象有没有实现这个 selector，如果有，那么执行。这样的好处就是既不影响原本代理方法的执行，又能给我们提供用 Observable 处理的余地。

上下两个 if 判断就都是用来触发事件的。一上一下分别在正真的代理方法执行前后触发。前文中，我们有通过 `methodInvoked` 方法获取 selector 对应的 Observable，其实还有一个 `sendMessage` 方法也起着一样的作用，原理也是相同的。是不是很像 AOP。

`_methodInvoked` 方法将代理方法的参数作为事件值触发事件：

```swift
open override func _methodInvoked(_ selector: Selector, withArguments arguments: [Any]) {
    methodInvokedForSelector[selector]?.on(.next(arguments))
}
```

> 如何获得触发事件的入口？答案就是通过消息转发。所有没有实现的代理方法走消息转发，达到统一触发事件的目的。

### 另一种替换代理类的方式

前面替换代理类的方式是 `proxyForObject`方法，它会把该 Object 的 delegate 替换为 DelegateProxy，把原本的 delegate 对象设置为 DelegateProxy 的 forwardDelegate 属性。

除此之外，还有一个替换代理的方法：

```swift
public static func installForwardDelegate(_ forwardDelegate: AnyObject, retainDelegate: Bool, onProxyForObject object: AnyObject) -> Disposable {
    weak var weakForwardDelegate: AnyObject? = forwardDelegate

    let proxy = Self.proxyForObject(object)
    
    assert(proxy.forwardToDelegate() === nil, "This is a feature to warn you that there is already a delegate (or data source) set somewhere previously. The action you are trying to perform will clear that delegate (data source) and that means that some of your features that depend on that delegate (data source) being set will likely stop working.\n" +
        "If you are ok with this, try to set delegate (data source) to `nil` in front of this operation.\n" +
        " This is the source object value: \(object)\n" +
        " This this the original delegate (data source) value: \(proxy.forwardToDelegate()!)\n" +
        "Hint: Maybe delegate was already set in xib or storyboard and now it's being overwritten in code.\n")

    proxy.setForwardToDelegate(forwardDelegate, retainDelegate: retainDelegate)
    
    // refresh properties after delegate is set
    // some views like UITableView cache `respondsToSelector`
    Self.setCurrentDelegate(nil, toObject: object)
    Self.setCurrentDelegate(proxy, toObject: object)
    
    assert(proxy.forwardToDelegate() === forwardDelegate, "Setting of delegate failed:\ncurrent:\n\(String(describing: proxy.forwardToDelegate()))\nexpected:\n\(forwardDelegate)")
    
    return Disposables.create {
        MainScheduler.ensureExecutingOnScheduler()
        
        let delegate: AnyObject? = weakForwardDelegate
        
        assert(delegate == nil || proxy.forwardToDelegate() === delegate, "Delegate was changed from time it was first set. Current \(String(describing: proxy.forwardToDelegate())), and it should have been \(proxy)")
        
        proxy.setForwardToDelegate(nil, retainDelegate: retainDelegate)
    }
}
```

这个方法会直接让你传入 forwardDelegate，并且你要是之前设置过 delegate，它还不高兴了。所以这个方法最好在你没有设置代理的时候用。该方法中也调用了 `proxyForObject`。除此之外，这个方法返回一个 Disposer，并在 dispose 的时候，将 forwardDelegate 正式设置为 delegate。因此，用这种方式的话，你需要将其放入 disposableBag 中。这样的功效就是如果 DelegateProxy 回收了，那么 forwardDelegate 就顺势升值了。

其实这也就是 Disposer 给我们带来的便利。我们不需要再 dealloc 的时候做设置了，直接全部扔到 disposableBag 中去。

> 整个过程就是，我们通过  proxyForObject 将 delegate 设置为 DelegateProxy。DelegateProxy 为每一个无返回值的代理方法都创建了 subject。我们通过 DelegateProxy 提供的 methodInvoked 获取这些 subject，并订阅。不在 DelegateProxy 中实现这些代理方法，使其触发消息转发。消息转发中统一执行方法，获取 selector 的 subject，并触发事件。
>
> 我们需要做的是：
>
> 1. 在 rx 拓展中，添加一个只读的代理属性(不一定叫做 delegate)。读取方法中调用相应 DelegateProxy 的 proxyForObject 并返回。
> 2. 在 rx 拓展中，添加只读的属性。读取方法中调用 DelegateProxy 的 methodInvoked 获取相应 selector 的 subject
> 3. 在相应 DelegateProxy 中添加 `currentDelegateFor`  和 `setCurrentDelegate` 方法，返回，设置类的原始代理属性(不一定是 delegate)。
> 4. 在相应 DelegateProxy 中添加返回值非空的代理方法的实现。



不想在写了，累的兔血，暂时就这样吧。