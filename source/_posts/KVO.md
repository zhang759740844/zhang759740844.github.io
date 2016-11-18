title: KVO 简介
date: 2016/11/17 14:07:12  
categories: iOS
tags: 
	- 学习笔记
	
---

本篇简单学习下如何使用 KVO。

<!--more-->

## KVO是什么？ 
KVO 是 Object-C 中定义的一个通知机制，其定义了一种对象间监控对方状态的改变，并做出反应的机制。对象可以为自己的属性注册观察者，当这个属性的值发生了改变，系统会对这些注册的观察者做出通知。其用途十分广泛，比方说，你的下载进度条是根据下载百分比决定的，那么，可以通过观察下载百分比的改变，刷新进度条的样式，来直观的反应下载进度等等。 

## KVO的用法 
### 为对象的属性注册观察者

```objc
- (void)addObserver:(NSObject *)observer  
         forKeyPath:(NSString *)keyPath  
            options:(NSKeyValueObservingOptions)options  
            context:(void *)context  
```

- observer: 观察者对象. 其必须实现方法 `observeValueForKeyPath:ofObject:change:context:`.
- keyPath: 被观察的属性，其不能为 `nil`.
- options: 设定通知观察者时传递的属性值，是传改变前的呢，还是改变后的，通常设置为 `NSKeyValueObservingOptionNew`。
- context: 一些其他的需要传递给观察者的上下文信息，通常设置为 `nil`。

### 观察者接收通知

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath  
                      ofObject:(id)object  
                        change:(NSDictionary *)change  
                       context:(void *)context  
```

- keyPath: 被观察的属性，其不能为 `nil`.
- object: 被观察者的对象.
- change: 属性值，根据上面提到的 `Options` 设置，给出对应的属性值。
- context: 上面传递的 `context` 对象。

### 清除观察者

```objc
- (void)removeObserver:(NSObject *)anObserver forKeyPath:(NSString *)keyPath  
```

### 注意事项

使用KVO消息传递机制有两个要求：
1. 观察者必须知道被观察对象，即在同一作用域。
2. 观察者还需要知道被观察对象的生命周期，因为在销毁发送者对象之前，需要取消观察者的注册。 


