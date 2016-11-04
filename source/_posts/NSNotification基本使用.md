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

## 注册
### addObserver:selector:name:object:
```objc
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(test) name:@"test" object:nil];
```

参数：
- `addObserver`:貌似都是`controller`因为还需要`selector`来执行
- `name`:通知名。当发送该通知名的通知是，观察者收到消息。
- `object`:缩小接受通知的范围，只接受相对应的`object`对象发来的通知。`nil`则全部接收。

### addObserverForName:object:queue:usingBlock:
`NSNotificationCenter`消息的接受线程是基于发送消息的线程的,也就是**同步**的，因此，有时候，你发送的消息可能不在主线程，而大家都知道操作UI必须在主线程，不然会出现不响应的情况。所以，在你收到消息通知的时候，注意选择你要执行的线程。
```objc
id observer = [[NSNotificationCenter defaultCenter] addObserverForName:notification object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(){...}];
```

参数：
- `queue`:`block`执行所在的线程。如果是`nil`那么就在当前线程(posting thread)
- `block`:该种方式不用`selector`而使用`block`回调。

## 删除
只要往`NSNotificationCenter`注册了，就必须有`remove`的存在。

### removeObserver:
```objc
- (void)removeObserver:(id)notificationObserver
```
对应`addObserverForName`方式，直接把上面的`id observer`作为参数传入即可。
对应`addObserver`方式，删除`controller`中全部通知.

### removeObserver:name:object:
```objc
[[NSNotificationCenter defaultCenter] removeObserver:self name:NotificationCitySelect object:nil]
```
删除指定`name`的通知。

## 发送
```objc
[[NSNotificationCenter defaultCenter] postNotificationName:NotificationCitySelectChanged object:@{NotificationCitySelect_ObjectLocationChange:@(YES)}]
```
发送指定`objcet`的通知，只有注册时注册了该对象的通知才可以接收。


