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

我们知道通过 `create`，`just`，`of` 等方法可以创建一个 Observable。那么这是如何做到的呢？

我们以 `create` 创建的订阅为例。首先调用的 `create` 方法存在于  `Observable` 的一个拓展中，包含了各种 Observable 的创建方法。`create` 方法创建了一个匿名的 Observable `AnonymousObservable`，并将包含各种事件的闭包传入：

```swift
extension Observable {
    public static func create(_ subscribe: @escaping (AnyObserver<E>) -> Disposable) -> Observable<E> {
        return AnonymousObservable(subscribe)
    }
}
```

 `AnonymousObservable` 初始化的过程很简单，就是将传入的包含事件的闭包保存起来。另外该类还提供了一个重写的 `run` 方法：

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

订阅的时候，我们调用的是 `ObservableType` 提供的订阅方法。`ObservableType` 是一个协议，由 `Observable` 实现。这个方法中创建了一个匿名的 Observer `AnonymousObserver`，把事件处理方法的闭包传入。然后就是观察者订阅事件队列：

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

这个订阅方法调用的是入参为 Observer 的 `subscribe` 方法。在 `Observable` 类中定义了该方法。不过这个方法并没有具体实现，它要求其子类实现。否则就会抛出异常：

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

`Observable` 在本例的子类是 `AnonymousObservable`，不过不是直接子类，这两个类中还夹着一个 `Producer` 类：

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

就是在 `Producer` 中实现的 `subscribe`。  



