title: KVO 实践及 FBKVOController 原理
date: 2016/11/17 14:07:12  
categories: iOS
tags: 

	- 源码解析

---

本篇简单学习下如何使用 KVO

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

## KVO 原理

### 实现原理

当一个对象使用 KVO 监听，iOS 会修改对象的 isa 指针，改为指向一个由 runtime 创建的类，这个类的 superclass 为原来的 class 对象。动态创建的类拥有自己的 set 方法实现，内部会依次调用：

```objc
1. - (void)willChangeValueForKey:(NSString *)key;
2. 原来的 setter 方法
3. - (void)didChangeValueForKey:(NSString *)key;
```

在 `didChangeValueForKey` 中会调用监听器的监听方法。所以**直接修改成员变量不会触发 KVO**。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/kvo_1.png?raw=true)

对象的 isa 指向可以通过 `someInstance->isa` 查看

### 如何手动触发 KVO

有时候为了在不改变属性值的情况下，触发监听方法，所以要手动触发 KVO。手动调用：

```objc
- (void)willChangeValueForKey:(NSString *)key;
- (void)didChangeValueForKey:(NSString *)key;
```

### KVC 触发 KVO

KVC 会触发 KVO。即使成员变量没有 get set 方法，KVC 手动调用 `willChangeValueForKey:` 和 `didChangeValueForKey:`。

## KVOController

### KVO 存在的问题

KVO 本身写起来并不友好，存在一些问题：

1. 需要手动移除观察者
2. 处理观察事件需要和注册观察事件割裂开

那么如何解决呢？

没有什么是一个中间变量不能解决的。可以创建一个实例，观察的事件由它分发，在其 dealloc 方法中移除观察者。这样就不用在外部业务方法中移除了。`KVOController` 也是这么做的。

### 使用方式

使用方式很简单，首先创建一个 `KVOController` 实例，然后执行 `observe:keyPath:options:block:` 方法注册观察者：

```objc
// 在 ClockView 类中
FBKVOController *KVOController = [FBKVOController controllerWithObserver:self];
self.KVOController = KVOController;

// clock 是 ClockView 实例的一个属性，data 是 clock 实例的一个属性
[self.KVOController observe:clock keyPath:@"date" options:NSKeyValueObservingOptionInitial|NSKeyValueObservingOptionNew block:^(ClockView *clockView, Clock *clock, NSDictionary *change) {
  clockView.date = change[NSKeyValueChangeNewKey];
}];
```

其中， block 的第一个参数 ClockView 为 observer，第二个参数 Clock 为 target，第三个参数为变化的 keypath 的变化前后值的字典。

### 整体结构

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/kvo_2.png?raw=true)

整体结构如上图所示，`KVOController` 对象有一个观察者 `observer` 还有一个 `NSMapTable` 的 `_objectInfosMap`，它的键是被观察的对象 `object`，值是被观察对象的各个属性的封装的 `NSMutableSet`，Set 中的每一个元素都保存了要执行的 block

### 源码解析

#### 创建 KVOController 实例

各个初始化方法都会来到这个方法：

```objc
- (instancetype)initWithObserver:(nullable id)observer retainObserved:(BOOL)retainObserved
{
  self = [super init];
  
  if (nil != self) {
    _observer = observer;
    // 创建 NSMapTable 需要提供一个选项来决定 key 以及 value 是强引用还是弱引用
    NSPointerFunctionsOptions keyOptions = retainObserved ? NSPointerFunctionsStrongMemory|NSPointerFunctionsObjectPointerPersonality : NSPointerFunctionsWeakMemory|NSPointerFunctionsObjectPointerPersonality;
    _objectInfosMap = [[NSMapTable alloc] initWithKeyOptions:keyOptions valueOptions:NSPointerFunctionsStrongMemory|NSPointerFunctionsObjectPersonality capacity:0];
    pthread_mutex_init(&_lock, NULL);
  }
  return self;
}
```

这里面的 `observer` 在外部会强引用这个 `KVOController`，所以初始化方法中的 `_observer`  默认是弱引用的，不用担心循环引用的问题。

由于参数 `retainObserverd` 默认是 `YES`，所以创建的 `NSMapTable` 的 key 的选项默认是：

```objc
NSPointerFunctionsStrongMemory|NSPointerFunctionsObjectPointerPersonality
```

选项的具体含义在下面介绍 `NSMapTable` 的时候会提及。总之，这里创建的 `NSMapTable` 默认键和值都是强引用的，所以如果被观察的对象是观察者 `observer` 本身，就要注意要传入弱引用，否则会产生循环引用。

最后创建了一个 `pthread_mutex_t`

####  注册观察方法

注册观察方法外部调用的方法：

```objc
- (void)observe:(nullable id)object keyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options context:(nullable void *)context {
  // create info
  _FBKVOInfo *info = [[_FBKVOInfo alloc] initWithController:self keyPath:keyPath options:options context:context];

  // observe object with info
  [self _observe:object info:info];
}
```

外部调用的时候会创建一个 `FBKVOInfo` 对象，这个对象保存了观察回调所需的各种要素，随后执行内部的注册方法：

```objc
- (void)_observe:(id)object info:(_FBKVOInfo *)info {
  // lock
  pthread_mutex_lock(&_lock);

  /// 拿到 KVOController 中所有的 object 的 KVOInfo 的集合
  NSMutableSet *infos = [_objectInfosMap objectForKey:object];

  /// 如果存在这个 info 那么就直接返回。 Info 主要是监听的 keyPath 和要执行的 block 信息，Infos 就是所有要监听的 keyPath 的集合
  _FBKVOInfo *existingInfo = [infos member:info];
  if (nil != existingInfo) {
    // unlock and return
    pthread_mutex_unlock(&_lock);
    return;
  }

  /// 如果没有这个 infos，那么就自己创建一个
  if (nil == infos) {
    infos = [NSMutableSet set];
    [_objectInfosMap setObject:infos forKey:object];
  }

  // add info and oberve
  [infos addObject:info];

  // unlock prior to callout
  pthread_mutex_unlock(&_lock);

  [[_FBKVOSharedController sharedController] observe:object info:info];
}
```

可以看到，这里把要观察的对象 `object` 和刚刚创建的保存回调要素的 `KVOInfo` 对象保存在了 `_onjectInfosMap` 这个 `NSMapTable` 中。

所以，一个被观察的对象 `object` 的所有注册观察的 `keyPath` 都会保存在一个 `NSMutableSet` 中，再把 `object` 和 `NSMutableSet` 作为键值对，保存在 `NSMapTable` 中。

最后调用的 `_FBKVOSharedController` 才是真正注册观察者的地方。

#### 真正用来注册观察者的 `KVOSharedController`

`KVOSharedController` 中还用到了 `NSHashTable`，但是和 `KVOController` 中 `NSMapTable` 不同的是，这里都是通过弱引用保存的。原因是因为外部的 `KVOController` 都保存确保不会被回收了，这里就不需要再强引用了。

```objc
@implementation _FBKVOSharedController
{
  NSHashTable<_FBKVOInfo *> *_infos;
  pthread_mutex_t _mutex;
}

+ (instancetype)sharedController
{
  static _FBKVOSharedController *_controller = nil;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    _controller = [[_FBKVOSharedController alloc] init];
  });
  return _controller;
}

- (instancetype)init
{
  self = [super init];
  if (nil != self) {
    NSHashTable *infos = [NSHashTable alloc];
    _infos = [infos initWithOptions:NSPointerFunctionsWeakMemory|NSPointerFunctionsObjectPointerPersonality capacity:0];
    pthread_mutex_init(&_mutex, NULL);
  }
  return self;
}
```

`KVOSharedController` 是个单例，所有的注册和回调逻辑都是通过这个实例完成的.

注册方法：

```objc
- (void)observe:(id)object info:(nullable _FBKVOInfo *)info
{
  if (nil == info) {
    return;
  }

  // register info
  pthread_mutex_lock(&_mutex);
  [_infos addObject:info];
  pthread_mutex_unlock(&_mutex);

  // add observer
  [object addObserver:self forKeyPath:info->_keyPath options:info->_options context:(void *)info];
}
```

还是能比较容易想到这个方法做了什么的。注意这里把 `KVOInfo` 实例作为 `context` 传了进去。

回调方法：

```objc
- (void)observeValueForKeyPath:(nullable NSString *)keyPath ofObject:(nullable id)object
                        change:(nullable NSDictionary<NSKeyValueChangeKey, id> *)change
                       context:(nullable void *)context
{
  _FBKVOInfo *info;

  {
    pthread_mutex_lock(&_mutex);
    /// 之前
    info = [_infos member:(__bridge id)context];
    pthread_mutex_unlock(&_mutex);
  }

  if (nil != info) {

    // take strong reference to controller
    FBKVOController *controller = info->_controller;
    if (nil != controller) {

      // take strong reference to observer
      id observer = controller.observer;
      if (nil != observer) {

        if (info->_block) {
          NSDictionary<NSKeyValueChangeKey, id> *changeWithKeyPath = change;
          // add the keyPath to the change dictionary for clarity when mulitple keyPaths are being observed
          if (keyPath) {
            /// 把 Keypath 也加入到了字典中去。
            NSMutableDictionary<NSString *, id> *mChange = [NSMutableDictionary dictionaryWithObject:keyPath forKey:FBKVONotificationKeyPathKey];
            [mChange addEntriesFromDictionary:change];
            changeWithKeyPath = [mChange copy];
          }
          info->_block(observer, object, changeWithKeyPath);
        }
      }
    }
  }
}
```

也是顺利成章的，接收到回调事件后，执行 block 里面的内容。刚才通过 `context` 把 `KVOInfo` 传入，现在再通过 `context`，把 `KVOInfo` 取出来。(说实话，这里的操作，包括下面的各种判空，其实有点多余。因为本身外部是强引用的，不会出现为空的情况)

#### 取消注册

取消注册在 `KVOController` 中存在三种情况：

1. 取消所有的监听
2. 取消关于某一个 object 的所有的监听
3. 取消某一个 object 内的某个 keypath 的监听

```objc
// 对应第三种情况
- (void)_unobserve:(id)object info:(_FBKVOInfo *)info
{
  // lock
  pthread_mutex_lock(&_lock);

  // get observation infos
  NSMutableSet *infos = [_objectInfosMap objectForKey:object];

  // lookup registered info instance
  _FBKVOInfo *registeredInfo = [infos member:info];

  if (nil != registeredInfo) {
    [infos removeObject:registeredInfo];

    // remove no longer used infos
    if (0 == infos.count) {
      [_objectInfosMap removeObjectForKey:object];
    }
  }

  // unlock
  pthread_mutex_unlock(&_lock);

  // unobserve
  [[_FBKVOSharedController sharedController] unobserve:object info:registeredInfo];
}

// 对应第二种情况
- (void)_unobserve:(id)object
{
  // lock
  pthread_mutex_lock(&_lock);

  NSMutableSet *infos = [_objectInfosMap objectForKey:object];

  // remove infos
  [_objectInfosMap removeObjectForKey:object];

  // unlock
  pthread_mutex_unlock(&_lock);

  // unobserve
  [[_FBKVOSharedController sharedController] unobserve:object infos:infos];
}

// 对应第一种情况
- (void)_unobserveAll
{
  // lock
  pthread_mutex_lock(&_lock);

  NSMapTable *objectInfoMaps = [_objectInfosMap copy];

  // clear table and map
  [_objectInfosMap removeAllObjects];

  // unlock
  pthread_mutex_unlock(&_lock);

  _FBKVOSharedController *shareController = [_FBKVOSharedController sharedController];

  for (id object in objectInfoMaps) {
    // unobserve each registered object and infos
    NSSet *infos = [objectInfoMaps objectForKey:object];
    [shareController unobserve:object infos:infos];
  }
}
```

这三种情况中，第一种删除所有 Object 是第二种删除某一个 Object 的特殊情况，第二种又是第一种删除某一个 Object 的某个 keyPath 的特殊情况。总的来说，都是操作 `_objectInfosMap`，将强引用的对象删除，然后交给 `KVOSharedController` 操作。

这三种情况最后都归结到 `KVOSharedController` 中去：

```objc
- (void)unobserve:(id)object info:(nullable _FBKVOInfo *)info {
  if (nil == info) { return; }

  // unregister info
  pthread_mutex_lock(&_mutex);
  [_infos removeObject:info];
  pthread_mutex_unlock(&_mutex);

  [object removeObserver:self forKeyPath:info->_keyPath context:(void *)info];
}
```



#### dealloc 时候的自动取消注册

由于观察者把整个过程交给了 `KVOController`，所以在观察者销毁的时候，`KVOController` 也会一起执行 `dealloc` 方法，来清除所有的监听。

```objc
- (void)dealloc
{
  [self unobserveAll];
  pthread_mutex_destroy(&_lock);
}
```

#### 总结

整个库的流程是：`KVOController` 把观察的对象作为其 `NSMapTable` 属性 `_objectInfomap` 的键，把整个回调环境组成的对象 `KVOInfo` 作为值保存起来。同时通过一个单例的 `KVOSharedController` 执行具体的注册于监听方法。

在我看来，本身 `KVOSharedController` 这个单例其实是没有多大意义的，在每个 `KVOController` 中其实就可以处理这些监听与回调了。不过这么写可能是为了更好的职责分离吧。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/kvo_3.png?raw=true)

最后还要强调一点，我认为 KVO 自动取消监听的**核心在于让 `KVOController` 这类的中间类的生命周期和被监听的 `object` 同步，而不是和 `Observer` 同步**。因为，只有在被监听对象回收的时候取消监听才能真正避免 crash 的危险。

事实上，我们一般在 VC 和 VM 中做监听，监听的对象是 VC 或者 VM 的属性，这种变相的把观察者 `Observer` 和被观察对象 `object` 的生命周期同步了，因而把 `KVOController` 作为监听者的属性也不会有问题。但是这会造成使用者的误解。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/kvo_4.png?raw=true)

所以我认为 KVOController 这种一个 `KVOController` 监听多个对象的做法其实是有问题的，也是不应该被鼓励的。真正应该的是像我图上画的，给每个被观察的对象绑定一个 `KVOController` 实例。

### NSMapTable

NSMapTable 相比较 NSDictionary 的优势有：

1. NSDictionary 必须是 key-obj 的形式，key 必须是满足 NSCopying 协议的；NSMapTable 则是 obj-obj 的形式
2. NSDictionary 的 obj 是强引用；NSMapTable 的 key 和 value 都可以自己决定是强引用还是弱引用。如果弱引用回收后，会自动删除。

#### 创建

创建 NSMapTable 的时候，需要指定键和值的选项：

```objc
NSMapTable *keyToObjectMapping = [NSMapTable mapTableWithKeyOptions:NSMapTableCopyIn valueOptions:NSMapTableStrongMemory];
```

上面创建的 NSMapTable 将和 NSDictionary 一模一样，复制 `key`，并对它的 `object` 引用计数 +1。

#### NSMapTable 的选项

- NSMapTableStrongMemory (a “memory option”)
- NSMapTableWeakMemory (a “memory option”)
- NSMapTableObjectPointerPersonality (a “personality option”)
- NSMapTableCopyIn (a “copy option”)

其中前两个 memory option 就是控制是对 obj 进行强引用还是弱引用。

personality option 在我的理解中是针对 key 的，它决定是否使用对象的指针来进行 hash （NSDictionary 中使用 NSString 来进行 hash，决定 object 的存储位置）。如果不指定这个选项，默认是使用整个对象进行 hash，那么在存储的过程中，作为 key 的这个对象是不能被改变的（对象变了，hash 值变了，自然就找不到 object 了）。

copy option 选项表明是否执行对象的 copy 方法，深拷贝一个新的对象进行存储。

#### NSArray 和 NSPointerArray 的区别

NSPointerArray 可以保存 NULL，因此，NSPointerArray 中的对象可以是 weak 的，销毁直接将该位置职位 null

#### NSHashTable 和 NSSet 区别

NSHashTable 是可变的，且 NSHashTable 可以放弱引用对象

## 参考

[如何优雅的使用KVO](<https://draveness.me/`KVOController`>)