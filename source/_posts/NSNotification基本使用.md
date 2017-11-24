title: NSNotification的使用
date: 2016/9/13 14:07:12  
categories: iOS
tags: 
	- NSNotification
---

本篇是对ios的通知使用方法的一个简单罗列。


<!--more-->

## NSNotificationCenter 

`NSNotificationCenter`就是一个消息通知机制，类似广播。观察者只需要向消息中心注册感兴趣的东西，当有地方发出这个消息的时候，通知中心会发送给注册这个消息的对象。这样也起到了多个对象之间解耦的作用。苹果给我们封装了这个`NSNotificationCenter`，让我们可以很方便的进行通知的注册和移除。

### 注册

#### addObserver:selector:name:object:

```objc
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(test) name:@"test" object:nil];
```

参数：
- `addObserver`:貌似都是`controller`因为还需要`selector`来执行
- `name`:通知名。当发送该通知名的通知是，观察者收到消息。
- `object`:缩小接受通知的范围，只接受相对应的`object`对象发来的通知。`nil`则全部接收。

#### addObserverForName:object:queue:usingBlock:

`NSNotificationCenter`消息的接受线程是基于发送消息的线程的,也就是**同步**的，因此，有时候，你发送的消息可能不在主线程，而大家都知道操作UI必须在主线程，不然会出现不响应的情况。所以，在你收到消息通知的时候，注意选择你要执行的线程。
```objc
id observer = [[NSNotificationCenter defaultCenter] addObserverForName:notification object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification *note){...}];
```

参数：
- `queue`:`block`执行所在的线程。如果是`nil`那么就在当前线程(posting thread)
- `block`:该种方式不用`selector`而使用`block`回调。

### 删除

只要往`NSNotificationCenter`注册了，就必须有`remove`的存在。

#### removeObserver:

```objc
- (void)removeObserver:(id)notificationObserver
```
对应`addObserverForName`方式，直接把上面的`id observer`作为参数传入即可。
对应`addObserver`方式，删除`controller`中全部通知.

#### removeObserver:name:object:

```objc
[[NSNotificationCenter defaultCenter] removeObserver:self name:NotificationCitySelect object:nil]
```
删除指定`name`的通知。

### 发送

有三种发送方式：

```objc
- (void)postNotification:(NSNotification *)notification;
- (void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject;
- (void)postNotificationName:(NSNotificationName)aName object:(nullable id)anObject userInfo:(nullable NSDictionary *)aUserInfo;
```

---

其中 `aUserInfo` 是发送的通知需要传递给接受者的信息。那么这个信息怎么取得呢？我们看一下 `NSNotification` 的数据结构：

```objc
@interface NSNotification : NSObject 
 
@property (readonly, copy) NSNotificationName name;
@property (nullable, readonly, retain) id object;
@property (nullable, readonly, copy) NSDictionary *userInfo;
```

上面 `addObserver` 的时候，无论是用 `selector` 还是 `block`，都将 `NSNotification` 实例传入了，可以直接获取 `userInfo` 信息。

---

```objc
[[NSNotificationCenter defaultCenter] postNotificationName:NotificationCitySelectChanged object:@{NotificationCitySelect_ObjectLocationChange:@(YES)}]
```
发送指定`objcet`的通知，只有注册时注册了该对象的通知才可以接收（即注册的时候的 `object` 要写的和这里的一样。`name` 和 `object` 都是用来取区分通知的）。

### 关于 Notification 在哪设置

Notification 最好在 `viewDidAppear` 中注册，在 `viewDidDisappear` 中移除，**即在页面显示的时候接收通知**。实在需要页面不显示的时候也能接到通知的话就只能在 `init` 和 `dealloc` 中做添加删除了（`dealloc` 中 remove 不会造成内存泄漏，Notification 中的 Observer 不是 strong，所以不会让引用+1，但是必须 remove 不然会产生野指针导致崩溃）由于 `viewDidAppear` 和 `viewDidDisappear` 不是成对出现的，需要在 `viewDidAppear` 注册通知前先移除一下该通知。

通过手势操作 VC 可以让 `viewDidAppear` 调用无数次，但 `viewDidDisappear` 一次也不触发。

## NSNotificationCenter的实现

`NSNotificationCenter` 的实现原理是观察者模式，用于监听操作。我们可以简单地认为 `NSNotificationCenter` 就是维护一个 `NSMutableDictionary`，key 是由 `name` 和 `sender` 生成的 notification object，比如 `NSNotification`，value 是向 observer 发送的 `selector`。发送通知就是在字典中找到相应方法执行。

这就解释了为什么一定要在对象回收时 `removeObserver`。因为在 `notificationCenter` 中，对象都是弱引用的。当对象回收后，`notificationCenter` 中保存的引用就会置为 `nil`，就会发生通知发向一个 `nil` 类的情况，导致 App 崩溃。












