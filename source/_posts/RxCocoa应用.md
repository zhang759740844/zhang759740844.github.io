title: RxCocoa 的应用
date: 2017/11/3 10:07:12  
categories: iOS
tags:
	- Swift
	- RxSwift
---

继续 RxSwift 的学习。这章的难度明显大了许多，很多东西，你知道要这么用没有用，你得知道为什么要这么用，这就需要阅读源码了。本文还是按照书中的章节进行。知其所以然会在后面的文章中补全。

<!--more-->

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_1.png?raw=true)

## 使用 RxCocoa 的应用

### 开始使用 RxCocoa

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
let search = searchCityName.rx.text
	.filter { ($0 ?? "").characters.count > 0}
	.flatMapLatest { text in
		return ApiController.shared.currentWeather(city: text ?? "Error")
                    .catchErrorJustReturn(ApiController.Weather.empty)
	}.observeOn(MainScheduler.instance)
// 订阅
search.subscribe(
    	...
    ).addDisposableTo(bag)
```

这段代码做了什么，这段代码将 textfield 的输入框和处理逻辑绑定了起来，输入框一有输入，那么就触发事件。先将空输入过滤掉，然后将 textfield 中输入的值通过 flatMapLatest 再转换为新的 Observable，即 city 转 Weather 的过程。其中如果 `currentWeather` 的转换产生了错误，就通过 `catchErrorJustReturn` 使用默认的事件值，空 Weather。

通过这个例子，可以再复习一下 flatMapLatest 和 flatMap 的区别。这个例子中 flatMapLatest 和 flatMap 是一样的，因为 `currentWeather` 返回的 Observable 只有一个元素，所以只订阅最后的 Observable 和 订阅每一个 Observable 没有区别，因为 Observable 不会再有事件发生了。但是，如果这里 `currentWeather` 是一个异步的操作，比如去网上拉取数据，那就必须使用 flatMapLatest 了。试想一下，在上一个网络请求还没有完成的时候，输入变化，这样又触发了一次网络请求。使用 flatMap 就会在每一次网络请求结束时都做响应，但这个并不符合逻辑，应该只响应最后一次网络请求，否则会产生请求覆盖。所以一定要用 flatMapLatest。 

以上的数据流程如图所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_41.png?raw=true)

#### 绑定 Observable

绑定是一种单向的数据流。RxCocoa 中的绑定通过 `bindTo(_:)` 实现。被绑定的对象的类型必须是 `ObserverType`，也就是 Observer 对象。

> 书上这个地方我认为是有问题的，书上的原话是：
>
> The fundamental function of binding is bindTo(_:). To bind an observable to another entity, the receiver must conform to ObserverType. This entity has been explained in previous chapters: it’s a Subject which can process values, but can also be written to manually.
>
> 书上说 ObserverType 就是一种 Subject，这样说是没有道理的。Subject 的范围要比 ObserverType 大，Subject 可以被订阅，提供自定义事件处理方法，但是 `bindTo()` 并不需要，它只需要有一个默认的事件处理方法即可。所以我认为书上这么说至少是不严谨的。

`bindTo(_:)` 是一种特殊的 subscribe，也是在 Observable 触发事件的时候，调用 Observer 的事件处理方法。但是原本需要在 subscribe 的时候传入的事件处理方法现在要求被绑定的 Observer 自己提供。所以像 `UIBindingObserver` 这样有默认事件处理方法的 Observer 或者一个已经被订阅过的 Subject 都是可以作为绑定对象的。

```swift
let search = searchCityName.rx.text
	.filter { ($0 ?? "").characters.count > 0}
	.flatMapLatest { text in
		return ApiController.shared.currentWeather(city: text ?? "Error")
                    .catchErrorJustReturn(ApiController.Weather.empty)
	}.observeOn(MainScheduler.instance)
// 绑定
search.map {"\($0.temperature)°C"}
	.bindTo(tempLabel.rx.text)
	.addDisposableTo(bag)
```

看到上面的代码，通过一个 map 将整个 `Weather` 对象剥离出来，只要其中的 `temperature` 属性。然后将其绑定给 `tempLabel.rx.text` 这个 Observer（这种通过 map 取出部分属性再进行绑定是一个很好的方式）。这就是前面说到的 `UIBindingObserver` 类型，它提供了一个默认的事件处理方法，将外部传来的字符串，设置为自己的 text 属性。

#### 使用 Units

Units 是一类专门用来使 UI 绑定更简单的 Observables。它不会产生异常，并且默认在主线程中。它有以下两大类：

1. `ControlProperty` 和 `ControlEvent`
2. `Driver`

`ControlProperty` 前面已经看到过了，是一种 Subject，用来将 UI 组件和数据绑定，例子中的 `searchCityName.rx.text` 就是一个  `ControlProperty`。`ControlEvent` 从命名上也能推测出，它用来监听 `UIControlEvents`。`Drivers `就是刚才说的不会产生异常，并且一定在主线程中的 Observable。如果不使用 Units，你可能会忘记调用 `.observeOn(MainScheduler.instance)`，最终在其他线程中更新 UI 导致崩溃。

```swift
let search = searchCityName.rx.text
	.filter { ($0 ?? "").characters.count > 0}
	.flatMapLatest { text in
		return ApiController.shared.currentWeather(city: text ?? "Error")
	}.asDriver(onErrorJustReturn: ApiController.Weather.empty)

search.map { "\($0/temerature)°C" }
	.drive(tempLabel.rx.text)
	.addDisposableTo(bag)
```

可以看到，使用 `asDriver(onErrorJustReturn:)` 代替了 `observeOn(MainSchedule.instance)`。另外，用 `drive` 代替了 `bindTo`。

我们现在绑定了一个 textfield，在每次其输入的时候都做了响应，这样其实是没有必要的。一般可以使用 `throttle` 操作符，这个是用来控制每隔多久发送一次事件的。还有更好的方法是监听用户点击返回按钮，当点击的时候获取搜索框中的输入。需要对上面的代码做一些变化：

```swift
let search = searchCityName.rx.controlEvent(.editingDidEndOnExit).asObservable()
	.map { self.searchCityName.text }
	.filter { ($0 ?? "").characters.count > 0}
	.flatMapLatest { text in
		return ApiController.shared.currentWeather(city: text ?? "Error")
	}.asDriver(onErrorJustReturn: ApiController.Weather.empty)
```

所以现在的数据流图变成了：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_44.png?raw=true)

### 进一步拓展

上一章中，完成了一个输入城市，然后显示城市天气的功能。这一章将对其进行拓展。

#### 加载等待状态

首先是给加载天气这个过程中加载一个等待的菊花图。这个菊花图的展示与否有两个条件：用户点击搜索时加载，获取到搜索结果后隐藏。点击搜索和获取搜索结果的 Observable 分别定义为如下：

```swift
let searchInput = 
searchCityName.rx.controlEvent(.editingDidEndOnExit).asObservable()
	.map { self.searchCityName.text }
	.filter { ($0 ?? "").characters.count > 0}

let search = searchInput.flatMapLatest { text in
	return ApiController.shared.currentWeather(city: text ?? "Error")
		.catchErrorJustReturn(ApiController.Weather.dummy)
}.asDriver(onErrorJustReturn: ApiController.Weather.dummy)
```

什么时候满足这两个条件？当 `searchInput` 事件发生的时候显示，当 `search` 事件发生的时候隐藏。所以我们的等待状态需要分别订阅这两个 Observable。但是这样的做法并不优雅，上一篇我们学过如何合并两个 Observable 的事件了，就是用 `merge`:

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_45.png?raw=true)

```swift
let running = Observable.from([
  searchInput.map { _ in true}.asObservable(), 
  search.map { _ in false}.asObservable()
]).merge()
.startWith(true)
.asDriver(onErrorJustReturn: false)

running.skip(1)
	.drive(activityIndicator.rx.isAnimating)
	.addDisposableTo(bag)

running.drive(tempLabel.rx.isHidden)
	.addDisposableTo(bag)
```

上面的代码可以看到，每当 `searchInput` 触发的时候，发出一个事件值为 true 的事件，当 `search` 触发的时候，发出一个事件值为 false 的事件。但是为什么要设置第一个事件为 true，又略过第一个事件呢？因为展示的 label 的 `isHidden` 属性和等待状态的 `isAnumating` 是相反的，即展示 label 的时候隐藏等待状态，展示等待状态的时候隐藏 label。但是最开始的时候既要 label 隐藏，又要等待状态隐藏，所以要略过第一个事件值。

#### 获取当前位置

这一章主要讲的是如何用 RxSwift 实现一个代理方法，要理解起来挺有难度的，这里先只是讲步骤，后面会转门写一篇代理转发的源码解析。

获取位置使用的是 `CCLocationManager` 类，这个类有一个代理 `CLLocationManagerDelegate`。我们需要做的就是在 `CCLocationManager` 回调代理方法的时候触发事件，将回调的参数作为事件值。

怎么做到的呢？这里不做详细展开，只介绍一下大概思路。简单来说，就是首先将我们创建的基于 Rx 的代理设置为 `CCLocationManager` 的 delegate。其次，由于我们不是没有写真实的响应方法嘛，当 `CCLocationManager` 要调用代理方法的时候一般是会产生异常的。但是我们可以通过消息转发，将所有调用代理方法的行为转化为发出特定事件。我们只需要通过方法名获取到相应的 Observable 并且订阅，即可达到响应式的效果。

因此，我们创建一个基于 rx 的代理类 `RxCLLocationManagerDelegateProxy`。这个 Delegate 要继承于 `DelegateProxy`，并且实现 `CLLocationManagerDelegate` 和 `DelegateProxyType` 协议。实现 `CLLocationManagerDelegate` 协议很好理解，因为我们要把自己设为 `CLLocationManager` 的代理。另外两个就是实现响应式的关键了：

```swift
class RxCLLocationManagerDelegateProxy: DelegateProxy, CLLocationManager, DelegateProxyType {
}
```

我们要提供两个方法用来设置和获取当前 Delegate。这是 `DelegateProxyType`中要求的，有了它，订阅回调方法的时候，就可以由 Rx 自动把当前类设置为 Delegate 了，而不需要我们显示的设置：

```swift
class RxCLLocationManagerDelegateProxy: DelegateProxy, CLLocationManager, DelegateProxyType {
	class func setCurrentDelegate(_ delegate: AnyObject?, toObject object: AnyObject) {
        let locationManager: CLLocationManager = object as! CLLocationManager
      	locationManager.delegate = delegate as? CLLocationManagerDelegate
    }
  
  	class func currentDelegateFor(_ object: AnyObject) -> AnyObject? {
        let locationManager: CLLocationManager = object as! CLLocationManager
      	return locationManager.delegate
    }
}
```

现在我们要 `CLLocationManager` 添加一个 rx 的 delegate 属性。关于 rx 的属性都是添加在 `Reactive` 类的拓展中，具体为什么会在以后的文章中解释。这是一个计算属性，每次都会调用 `proxyForObject` 方法。这个方法是在 `DelegateProxy` 中实现的，作用就是创建 `RxCLLocationManagerDelegateProxy` 实例，并且将其设置为 `CLLocationManager` 的代理

```swift
extension Reactive where Base: CLLocationManager {
    var delegate: DelegateProxy {
        return RxCLLocationManagerDelegateProxy.proxyForObject(base)
    }
}
```

现在设置到代理就可以添加监听代理方法调用的 Observable 了：

```swift
extension Reactive where Base: CLLocationManager {
    var delegate: DelegateProxy {
        return RxCLLocationManagerDelegateProxy.proxyForObject(base)
    }
  
    var didUpdateLocations: Observable<[CLLocation]> {
      return delegate.methodInvoked(#selector(CLLocationManagerDelegate.locationManager(_:didUpdateLocations:)))
    	.map { paramters in
             return parameters[1] as! [CLLocation]
        }
  	}
}
```

`methodInvoked` 方法同样是 `DelegateProxy` 中实现的方法。它获取相应 selector 所对应的 Observable。这个 selector 和 Observable 的关系是以字典的方式保存在 `DelegateProxy` 中的。当调用代理方法的时候，就会变为触发相应 selector 的 Observable。Observable 的事件值就是代理方法的参数，上面的 `parameters[1]` 就表示 `didUpdateLocations:` 的参数。

到此，rx 实现的代理方法就完成了。

最后如何监听按钮开始定位我就不再赘述了。



## 中级 RxSwift/RxCocoa

### 错误处理

错误处理部分比较简单，一般有两种方式：捕获或者重试。

#### 捕获

捕获有两种，一种是直接传递一个默认值，也就是前面常见的:

```swift
func catchErrorJustReturn(_ element:) -> RxSwift.Observable<Self.E>
```



它会自动将这个默认值作为事件值发出一个新的事件。另外还有一种，接受一个闭包，然后返回一个新的 Observable：

```swift
func catchError(_ handler:) -> RxSwift.Observable<Self.E>
```

示例：

```swift
.catchError { error in
    if let text = text, let cachedData = self.cache[text] {
        return Observable.just(cachedData)
    } else {
        return Observable.just(ApiController.Weather.empty)
    }
}
```

#### 重试

重试就是在 Observable 触发 error 的时候，重新尝试发出事件。使用方式很简单，在捕获错误前即可，参数为做大尝试次数：

```swift
return ApiController.shared.currentWeather(city: text ?? "Error") 
	.do(onNext: { data in 
		if let text = text { 
          	self.cache[text] = data 
        } 
	}).retry(3) 
	.catchError { error in 
		if let text = text, let cachedData = self.cache[text] { 
          	return Observable.just(cachedData) 
        } else { 
          	return Observable.just(ApiController.Weather.empty) 
        } 
}
```

还有一种 `retryWhen()` 方法：

```swift
.retryWhen { e in 
	return e.flatMapWithIndex { (error, attempt) -> Observable<Int> in 
		if attempt >= maxAttempts - 1 { 
          	return Observable.error(error) 
        } 
		return Observable<Int>.timer(Double(attempt + 1), scheduler: MainScheduler.instance).take(1) 
	} 
}
```

这个方法中，在未达到最多尝试次数前都不报错。每隔一秒钟重试一次。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_45.png?raw=true)

### 单元测试

RxSwift 的官方提供了 RxTest 和 RxBlocking 两个库帮助我们进行单元测试。下面来学习一下。

#### RxTest

RxSwift 的单元测试的主要过程就是：自建一个 Observable，然后给定几个默认的值，以检测订阅方法的正确性。

Observable 主要分为 Cold Observable 和 Hot Observable。Rx 认为，这两者都是 Observable，所以并不怎么做区分。但是单元测试的时候还是需要了解一下的。简单的说，通过 `create` 等创建的普通的 Observable 就是 Cold Observable，特点就是当订阅发生的时候立刻发出事件；各种 Subject 就是 Hot Observable，特点就是无论是否订阅都会发出事件，订阅之后才能收到事件。

我们来看一个例子：

```swift
override func setUp() {
    super.setUp()
  	scheduler = TestScheduler(initialClock: 0)
}

func testFilter() {
  	// 1
    let observer = scheduler.createObserver(Int.self)
  	
  	// 2
  	let observable = scheduler.createHotObservable([
        next(100, 1),
      	next(200, 2),
      	next(300, 3),
      	next(400, 2),
      	next(500, 1)
    ])
  
  	// 3
  	let filterObservable = observable.filter {
        $0 < 3
    }
  
  	// 4
  	schedyler.scheduleAt(0) {
        self.subscription = filterObservable.subscribe(observer)
    }
  
  	// 5
  	scheduler.start()
  
  	// 6
  	let results = observer.events.map {
        $0.value.element@
    }
  
  	// 7
    XCTAssertEqual(results, [1, 2, 2, 3])
}

override func tearDown() {
    scheduler.scheduleAt(1000) {
        self.subsciprtion.dispose()
    }
  	super.tearDown()
}
```

下面来解释一下这个例子。首先在 `setUp` 方法中，创建了 `TestScheduler` 实例。所有的订阅事件都会在这个 Scheduler 中进行。然后就是测试方法。测试方法分为七个步骤：

1. 用 Scheduler 创建 Observer，并且设置 Observable 事件值类型
2. 用 Scheduler 创建 Hot Observable，并且设置了延迟多久发出事件，以及发出的事件值
3. 这一步的过滤就是被测试的方法
4. 用 Scheduler 在某个时刻开始订阅上面创建的 Observable
5. 开启 Scheduler，不开启是不会有事件发生的哦
6. 用一个对象收集 Observer 获得的事件值
7. 断言判断 Observer 获得的事件值是否和预期的一样

#### RxBlocking

上面的 RxTest 是一个同步测试。如果有网络请求之类的异步事件该如何呢？我们可以使用 XCTest 中提供的 expectation，不过这样就显得很啰嗦了。RxBlocking 可以简化这一过程。下面是一个使用实例：

```swift
func testRgbIs010() {
    let rgbObservable = viewModel.rgb.asObservable().subscribeOn(scheduler)
  	viewModel.hexString.value = "#00ff00"
  	let result = try! rgbObservable.toBlocking(timeout: 1.0).first()!
  
  	XCTAssertEqual(0 * 255, result.0)
  	XCTAssertEqual(1 * 255, result.1)
  	XCTAssertEqual(0 * 255, result.2)
}
```

看一下这个例子，这里 `rgb` 是一个 `Driver` 类型，我没有把它的定义写出来。`rgbObservable` 是一个异步事件的 Observable。如果不适用 `toBlocking()`，那么程序顺序执行，订阅完了也就结束了。但是这里通过对 Observable 使用 `toBlocking` 方法，就将 Observable 阻塞住了。它会等待事件的到来，直到定时结束。



### Scheduler 的介绍

对 Scheduler 的通常误解是认为 Scheduler 就是 thread。其实 Scheduler 应该类比 dispatch queue。一个 Schelduler 可能在多个线程中，多个 Scheduler 也可能在一个线程中：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_47.png?raw=true)

#### 基本使用

我们可以通过 `subscribeOn()` 以及 `observerOn()` 来将订阅和执行设置不同 scheduler：

```swift
let globalScheduler = ConcurrentDispatchQueueScheduler(queue: DispatchQueue.global())

observable.subscribeOn(globalScheduler)
	.subsribeOn(gloibalScheduler)
	.dump()
	.observeOn(MainScheduler.instance)
	.dumpingSubscription()
	.addDisposableTo(bag)
```

#### 冷热Observable 对 Schedulers 的影响

这里主要介绍了一个注意点，就是对于 hot Observable，它订阅所在的线程就是其发出事件所在的线程，所以使用 `subscribeOn()` 方法控制是无效的。

### 自定义 Rx 拓展

这里是一个拓展 `URLSession`，使其通过 rx 获取数据以及处理数据的例子：

```swift
extensiong Reactive where Base: URLSession {
    func response(request: URLRequest) -> Observable<(HTTPURLResponse, Data)> {
        return Observable.create { observer in 
			let tast = self.base.dataTask(with: request) { (data, response, error) in
				guard let response = response, let data = data else {
                    observer.on(.error(error ?? RxURLSessionError.unKnown))
                  	return
                }
				guard let httpResponse = response as? HTTPURLResponse else {
                    observer.on(.error(RxURLSessionError.invalidResponse(response: response)))
                  	return
                }
				observer.on(.next(httpResponse, data))
				observer.on(.completed)
			}
			task.resume()
			
			return Disponsables.create(with: task.cancel)
		}
    }
  
  	func data(request: URLRequest) -> Observable<Data> {
        return response(requset: request).map { (response, data) -> Data in
			if 200 ..< 300 ~= response.statusCode {
                return data
            } else {
                throw RxURLSessionError.requestFailed(response: response, data: data)
            }
        }
    }
  
  	func image(request: URLResquest) -> Observable<UIImage> {
        return data(request: request).map { d in
        	return UIImage(data: d) ?? UIImage()                                  
		}
    }
}
```

以上就是一个获取网络图片的的一个基本过程，先是一个 `response` 方法，因为是 rx 嘛，必须返回一个 Observable 以供订阅。在方法中通过 `.create()` 创建一个 Observable，然后在其中开始网络请求，也就是 `dataTask` 方法。在网络请求的回调中，将获取的数据作为事件值，发出一个事件，之后结束整个 Observable。另外的两个方法是在网络请求的基础上，通过 `map` 将事件值进行进一步处理。这个网络请求流程如图：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_48.png?raw=true)

更进一步的，我们可以为这个网络请求设置一个缓存，即将获取的数据做一个接口层面的缓存。我们可以这样做：

```swift
// 全局的或者某个单例,用来保存缓存
fileprivate var internalCache = [String: Data]()

// 缓存方法
extension ObservableType where E == (HTTPURLResponse, Data) {
    func cache() -> Observable<E> {
      	return self.do(onNext: {(response, data) in 
        	if let url = response.url?.absoluteString, 200 ..< 300 ~= response.statusCode {
            	internalCache[url] = data
        	}
		})
    }
}

```

先创建一个全局的保存缓存的地方，键是 url，值是数据。然后在 `ObservableType` 中添加一个针对网络请求的缓存方法，所有 Observable 都是 `ObservableType` 类型的。这个 `cache` 方法做了一件事就是通过 `.do`给 Observable 添加了一个 `onNext` 方法，每次触发事件的时候都会调用。使用上和 `map` 类似。现在将之前的 `data`，以及 `image` 方法修改一下即可：

```swift
return response(request: request).cache().map { (response, data) -> Data in
	//...
}
```

由于有了缓存，所以需要在请求数据前加一步验证。如果有缓存，直接返回一个 Observable：

```swift
if let url = request.url?.absoluteString, let data = internalCache[url] {
    return Observable.just(data)
}
```

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/rx_49.png?raw=true)



