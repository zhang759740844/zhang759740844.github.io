title: React-Native 初始化与通信原理源码分析
date: 2017/1/17 10:07:12  
categories: React-Native
tags:
	- React-Native
---

RN 是如何启动的，oc 与 js 是如何通信的将是本文探究的重点。虽然网上已经有不少解释 RN 原理的文章，但是到自己读起源码的时候，还是非常累人的。这里，将较为细致的梳理一下 RN 的方法调用过程，希望能对各位有所帮助。

本文主要针对 RN 0.39 版本，不同版本可能会略有不同。不多bb RN 的基本概要了，直接开始（建议读者还是先自行了解下 RN 的基本原理，有助于理解）。

<!--more-->

## Native 初始化过程

先来一张完整的初始化流程图：

![RN 初始化流程图](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/RN启动流程.png?raw=true)

下面将对其中的部分方法做一些注释。

### tag1
```objc
RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
                                                    moduleName:@"AwesomeProject"
                                             initialProperties:nil
                                                 launchOptions:launchOptions];
```

任何 RN 的使用者都应该对这个方法不陌生。在 js 端写的各个 component，都将在 native 转换成 `RCTRootView` 的形式展现出来。

这个方法分为两步，第一步创建一个 `RCTBridge` 的实例，它是 oc 与 js 交互的桥梁，整个初始化的过程就是创建这个 Bridge。第二步通过这个 Bridge 创建一个 `RCTRootView`。

这里需要注意一下。对于一个半 RN 半 native 的应用（即不是通过 RN 的 navigator 跳转，而是通过原生跳转好后，再分别创建 RN 的 view），创建页面时不应该直接调用 `initWithBundleURL:moduleName:initialProperties:launchOptions:` 方法。因为这样每次都要创建 `RCTBridge`，这是一个耗时耗资源的过程。应该事先创建好 `RCTBridge`，在要创建页面的时候调用 `initWithBridge:moduleName:initialProperties:` 方法。

### tag2
一番跳转来到 `setUp` 方法内，一些比较次要的代码比如 `Logger` 我就不分析了：

```objc
- (void)setUp
{
	...
  [self createBatchedBridge];
  [self.batchedBridge start];
	...
}
```

创建并持有了其子类 `RCTBatchedBridge` 的实例。`RCTBridge` 中其实没有太多代码，初始化的大部分逻辑都在 `[self.batchedBridge start]` 中完成。

### tag3
创建 `RCTBatchedBridge` 中最重要的方法就是 `[self.batchedBridge start]` 方法。该方法主要包含以下几步:

1. 读取 js 源码
2. 初始化各个需要给 js 调用的模块
3. 创建一个 JSContext，为 JSContext 设置多个回调方法
4. 将模块信息写入一个字符串中
5. 将字符串传递给 js 端
6. 执行 js 源码

其中创建了两个 GCDGroup，分别为 `initModulesAndLoadSource` 和 `setupJSExecutorAndModuleConfig`。当3、4步完成后才能执行第5步，当1、2、5都完成后才会执行6。

### tag4
异步地读取 jsBundle，传入加载成功和正在加载中的回调方法。没有太多好说的，都是一些系统 api 的调用。

```objc
  [self loadSource:^(NSError *error, NSData *source, __unused int64_t sourceLength) {
    if (error) {
      RCTLogWarn(@"Failed to load source: %@", error);
      dispatch_async(dispatch_get_main_queue(), ^{
        [weakSelf stopLoadingWithError:error];
      });
    }
    sourceCode = source;
    dispatch_group_leave(initModulesAndLoadSource);
  } onProgress:^(RCTLoadingProgress *progressData) {
#ifdef RCT_DEV
    RCTDevLoadingView *loadingView = [weakSelf moduleForClass:[RCTDevLoadingView class]];
    [loadingView updateProgress:progressData];
#endif
  }];
```

其中，`[weakSelf stopLoadingWithError:error]` 就是经常所见的红屏报错。`onProgress` 的回调方法表示，如果在真机上加载，那么在加载时页面上部显示 “Loading from XXXX” 的提示。

### tag5
`initModulesWithDispatchGroup:` 是一个比较复杂的方法，用来初始化模块信息。每一个将要暴露给 js 的模块都会保存为 `RCTModuleData` 的形式，然后被存储为3分配置表。

```objc
NSMutableArray<Class> *moduleClassesByID = [NSMutableArray new];
NSMutableArray<RCTModuleData *> *moduleDataByID = [NSMutableArray new];
NSMutableDictionary<NSString *, RCTModuleData *> *moduleDataByName = [NSMutableDictionary new];

moduleDataByName[moduleName] = moduleData;
[moduleClassesByID addObject:moduleClass];
[moduleDataByID addObject:moduleData];
```

#### RCTRegisterModule
那么，如何找到暴露给 js 的模块呢？RN 提供了 `RCTRegisterModule();` 的宏：

```objc
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }

void RCTRegisterModule(Class moduleClass)
{
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    RCTModuleClasses = [NSMutableArray new];
  });
  
  // Register module
  [RCTModuleClasses addObject:moduleClass];
}
```

这样就生成了两个方法：
1. 在类 `load` 的时候，就会调用 `RCTRegisterModule` 方法，将类自动注册到 `RCTModuleClasses` 数组中。只要遍历该数组，就能取出所有暴露出的模块。
2. `moduleName` 方法，返回 `@#js_name`。这里的 `@#` 的一时是把宏参数 `js_name` 转为字符串，啥也没有，返回的就是空。当然在 `RCTBridgeModuleNameForClass()` 这个获取模块名的方法里，如果 `moduleName` 长度为0，那么就会调用 `NSStringFromClass()` 方法获取类名。

### tag6
在所有要暴露给 js 的类中，`RCTJSCExecutor` 是最特殊的一个类。需要率先创建一个实例并作为instance 保存在一个 `RCTModuleData` 的实例中，以防其他 module 可能需要用到。

在创建 `RCTJSCExecutor` 实例的时候，创建了一个 JSThread，所有的 JS 通信，都是通过  `RCTJSCExecutor` 执行，都是在这个 JSThread 内。

### tag7
除了 `RCTJSCExecutor` 的其他模块实例化的过程都是在 `prepareModulesWithDispatchGroup:` 中完成。实例化每个暴露的模块，并将其设置为 `RCTModuleData` 的 `instance` 属性。在 `setUpMethodQueue` 方法中，为每一个模块都会创建一个自己独有的专属串行队列，保证每个模块内的通信事件都是串行执行的。

### tag8
`gatherConstants` 方法主要功能：

```objc
RCTExecuteOnMainThread(^{
  self->_constantsToExport = [self->_instance constantsToExport] ?: @{};
}, YES);
```

将模块(instance)的一些常量设置给各自的 `RCTModuleData` 实例中。

### tag9
在 `setUp` 方法中，JSThread 内创建了一个 `JSContext`，并且为 Context 设置了不同的 block，如：

```objc
    context[@"nativeFlushQueueImmediate"] = ^(NSArray<NSArray *> *calls){
      RCTJSCExecutor *strongSelf = weakSelf;
      if (!strongSelf.valid || !calls) {
        return;
      }

      RCT_PROFILE_BEGIN_EVENT(RCTProfileTagAlways, @"nativeFlushQueueImmediate", nil);
      [strongSelf->_bridge handleBuffer:calls batchEnded:NO];
      RCT_PROFILE_END_EVENT(RCTProfileTagAlways, @"js_call");
    };
```

这些 block 会在特定的场合调用。之后将有介绍。

### tag10
```objc
- (NSString *)moduleConfig
{
  NSMutableArray<NSArray *> *config = [NSMutableArray new];
  for (RCTModuleData *moduleData in _moduleDataByID) {
    if (self.executorClass == [RCTJSCExecutor class]) {
      [config addObject:@[moduleData.name]];
    } else {
      [config addObject:RCTNullIfNil(moduleData.config)];
    }
  }

  return RCTJSONStringify(@{
    @"remoteModuleConfig": config,
  }, NULL);
}
```

在 `moduleConfig` 方法中，各个模块名将先被加入 config 数组中，然后被转换为 json 字符串的形式。这里只是写入模块名：


![moduleConfig](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/moduleConfig.png?raw=true)

### tag11
这一步将上面的 JSON 字符串通过 `JSExecutor` 传入 JS 作为全局变量。变量名为 `__fbBatchedBridgeConfig`:

```objc
- (void)injectJSONConfiguration:(NSString *)configJSON
                     onComplete:(void (^)(NSError *))onComplete
{
	...
  [_javaScriptExecutor injectJSONText:configJSON
                  asGlobalObjectNamed:@"__fbBatchedBridgeConfig"
                             callback:onComplete];
}
```

### tag 12
当上面所有步骤全部做完后，就开始通过 `executeSourceCode` 执行 js 代码了。


## JS初始化过程
native 端 `injectJSONConfiguration` 只把模块的名称组成的 JSON 字符串置入了 `__fbBatchedBridgeConfig`，那么 JS 端如何拿到模块的方法以及一些常量信息呢？需要我们研究一下 JS 端的初始化过程。

### NativeModule
js 的打包文件如下所示：
![jsbundle](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/jsBundle.png?raw=true)

可以看到，其中导入了 `NativeModule`。它就是用来接收保存 native 端暴露的模块的。

先来看一张整体的初始化流程图：
![JS 初始化](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/获取methods.png?raw=true)

#### tag1
```javascript
  const bridgeConfig = global.__fbBatchedBridgeConfig;
  invariant(bridgeConfig, '__fbBatchedBridgeConfig is not set, cannot invoke native modules');
  (bridgeConfig.remoteModuleConfig || []).forEach((config: ModuleConfig, moduleID: number) => {
    // Initially this config will only contain the module name when running in JSC. The actual
    // configuration of the module will be lazily loaded.
    const info = genModule(config, moduleID);
    if (!info) {
      return;
    }

    if (info.module) {
      NativeModules[info.name] = info.module;
    }
    // If there's no module config, define a lazy getter
    else {
      defineLazyObjectProperty(NativeModules, info.name, {
        get: () => loadModule(info.name, moduleID)
      });
    }
  });
```
这里的 `global.__fbBatchedBridgeConfig` 就是 native 注入的字符串，通常情况下是各个模块名。这里说一般是因为如果打开了 Remote JS Debugging，那么这里得到的是包括方法名的全部配置信息。如下图：

![bridgeConfig](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/bridgeConfig.png?raw=true)

(注：开了 Remote JS Debugging 后 js 代码是可以断点的，但是 Xcode 经常显示 `__nw_connection_get_connected_socket_block_invoke xx Connection has no connected handler` 然后，Xcode 的断点就失灵了。)

关闭 Remote JS Debugging，`info.module` 为 nil，因此一定会运行到 `get: () => loadModule(info.name, moduleID)` 方法，该方法是一个懒加载方法，以此来加快 RN 的初始化速度。

经过这个方法，就将所有暴露出来的方法的执行信息保存在 NativeModule 对象（或者说字典，不是数组。键是module名，值是模块内的方法组成的对象）里了。

#### tag2
进入 `loadModule` 方法：

```javascript
function loadModule(name: string, moduleID: number): ?Object {
  invariant(global.nativeRequireModuleConfig,
    'Can\'t lazily create module without nativeRequireModuleConfig');
  const config = global.nativeRequireModuleConfig(name);
  const info = genModule(config, moduleID);
  return info && info.module;
}
```

可以看到调用了 `global.nativeRequireModuleConfig` 方法，以及 `genModule` 方法。

#### tag3
`global.nativeRequireModuleConfig` 就是 native 流程图的 tag9，`RCTJSCExecutor` 的 `setUp` 方法中，注册的诸多回调中的一个：

```objc
context[@"nativeRequireModuleConfig"] = ^NSArray *(NSString *moduleName) {
  RCTJSCExecutor *strongSelf = weakSelf;
  if (!strongSelf.valid) {
    return nil;
  }

  RCT_PROFILE_BEGIN_EVENT(RCTProfileTagAlways, @"nativeRequireModuleConfig", @{ @"moduleName": moduleName });
  NSArray *result = [strongSelf->_bridge configForModuleName:moduleName];
  RCT_PROFILE_END_EVENT(RCTProfileTagAlways, @"js_call,config");
  return RCTNullIfNil(result);
};
```

`methods` 方法会拿到对应 `RCTModuleData` 的所有方法，然后循环找到以 `__rct_export__` 开头的方法(为什么是以 `__rct_export__` 开头后面再讲)。

`JSMethodName` 方法会拿到方法的字符串，并截取第一个冒号前的字符，作为 JS 简写方法名。

最后将各个方法常量等组成一个数组，即需要返回的 `config`:

```objc
  NSArray *config = @[
    self.name,
    RCTNullIfNil(constants),
    RCTNullIfNil(methods),
    RCTNullIfNil(promiseMethods),
    RCTNullIfNil(syncMethods)
  ];
```

#### tag4 
来看一下 `genModule` 方法：

```javascript
function genModule(config: ?ModuleConfig, moduleID: number): ?{name: string, module?: Object} {
  ...
  methods && methods.forEach((methodName, methodID) => {
    const isPromise = promiseMethods && arrayContains(promiseMethods, methodID);
    const isSync = syncMethods && arrayContains(syncMethods, methodID);
    invariant(!isPromise || !isSync, 'Cannot have a method that is both async and a sync hook');
    const methodType = isPromise ? 'promise' : isSync ? 'sync' : 'async';
    module[methodName] = genMethod(moduleID, methodID, methodType);
  });
  ...
}
```

拿到了通过 `global.nativeRequireModuleConfig` 方法获得的完整 `config` 信息。在该方法中通过一个循环，将所有 `method` 拿出，存入一个 `module` 对象中去，键是 method 名，值是通过 `genMethod` 方法生成的 function。

#### tag5
再跟进到 `genMethod` 方法中去：

```javascript
function genMethod(moduleID: number, methodID: number, type: MethodType) {
  let fn = null;
  if (type === 'promise') {
	...
  } else {
    fn = function(...args: Array<any>) {
      const lastArg = args.length > 0 ? args[args.length - 1] : null;
      const secondLastArg = args.length > 1 ? args[args.length - 2] : null;
      const hasSuccessCallback = typeof lastArg === 'function';
      const hasErrorCallback = typeof secondLastArg === 'function';
      hasErrorCallback && invariant(
        hasSuccessCallback,
        'Cannot have a non-function arg after a function arg.'
      );
      const onSuccess = hasSuccessCallback ? lastArg : null;
      const onFail = hasErrorCallback ? secondLastArg : null;
      const callbackCount = hasSuccessCallback + hasErrorCallback;
      args = args.slice(0, args.length - callbackCount);
      BatchedBridge.enqueueNativeCall(moduleID, methodID, args, onFail, onSuccess);
    };
  }
  fn.type = type;
  return fn;
}
```

我们现在只看 else 的情况。主要就是将 `moduleID`，`methodID`，参数以及失败和成功的回调函数传入 `BatchedBridge` 的 `enqueuenativeCall` 方法。这里的 `moduleID`，`methodID`， 也就是一般讨论 RN 原理时经常看到的 “通过模块、方法id找到对应模块和方法”，其实就是**对应数组中的下标**。

另外，关于取出成功和失败的回调方法，是通过判断最后两个参数是否是方法来得到的，默认情况下，最后一个是成功的回调方法，倒数第二个是失败的回调方法。因此，如果需要设置回调方法，那么必须放在最后，且回调方法不能超过两个。

### BatchedBridge
`BatchedBridge` 是个啥:

```javascript
const MessageQueue = require('MessageQueue');
const BatchedBridge = new MessageQueue();

...

Object.defineProperty(global, '__fbBatchedBridge', {
  configurable: true,
  value: BatchedBridge,
});
```

`BatchedBridge` 是一个 `MessageQueue` 实例。其将自身写入全局变量 `__fbBatchedBridge` 上，这样 Native 可以通过 `__fbBatchedBridge` ，访问 JSBridge对象，比如在 `RCTJSCExecutor.mm` 的 `_executeJSCall:arguments:unwrapResult:callback` 方法中。

### MessageQueue
在前面的 tag5 中我们看到，`genModule` 方法中生成的 function 调用了 `BatchedBridge` 的 `enqueuenativeCall`。这个方法就定义在 `MessageQueue.js` 中。

代码太长我就就贴部分吧，大致分为 个部分：

首先，在 `MessageQueue` 中定义了一个从0开始的计数的 `_callbackID`，用来标识 js 端的回调函数。将 `callbackID` push 进参数数组 `params` 里。然后，将这个 callback 保存在本地的 `_callbacks` 数组中的对应 `callbackID` 位置。这样，当 native 传来回调的 `callbackID` 的时候，就能在 `_callbacks` 数组中找到并执行相应方法了。

```javscript
onFail && params.push(this._callbackID);
this._callbacks[this._callbackID++] = onFail;
onSucc && params.push(this._callbackID);
this._callbacks[this._callbackID++] = onSucc;
```

设置好回调之后，将 `moduleID`，`methodID`，`params` 分别 push 进 `_queue`这个数组的各个位置。下面的 `MODULE_IDS`，`METHOD_IDS `，`PARAMS`分别代表0，1，2。
```javascript
this._queue[MODULE_IDS].push(moduleID);
this._queue[METHOD_IDS].push(methodID);
this._queue[PARAMS].push(params);
```



为什么要把方法参数放进一个 `_queue` 数组里呢？因为 js 不能主动调用除了上面设置 `JSContext` 的时候设置的那些回调方法外的 native 方法，所以 js 端想调用 native 端代码的时候，必须将想要调用的模块、方法、参数放在一个数组里，等待 native 来获取这个数组，也就是这个 `_queue`。什么时候 native 回来要一次这个数组呢？比如在完成 native 调用 js 方法后：

```objc
  [_javaScriptExecutor callFunctionOnModule:module
                                     method:method
                                  arguments:args
                                   callback:^(id json, NSError *error) {
                                     [weakSelf _processResponse:json error:error];
                                   }];
```

上面的这个方法就是 native 调用 js 的方法，具体的调用流程在后面会说明。可以看到，在 `callback` 的 block 内，有一个叫做 `json` 的入参，这个 `json` 就是上面的 `_queue`。在 native 调用完 js 的方法后，native 就会来处理 js 是否需要调用 native 的什么方法，并且刷新 `_queue` 数组。

![json示例](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/json示例.png?raw=true)

继续刚才的 `enqueueNativeCall` 方法往下走执行到下面的代码：

```javascript
if (global.nativeFlushQueueImmediate && now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS) {
    global.nativeFlushQueueImmediate(this._queue);
    this._queue = [[], [], [], this._callID];
    this._lastFlush = now;
}
```

这里的 `global.nativeFlushQueueImmediate` 就是 `JSContext` 设置的几个回调方法中的一个，供 JS 主动调用。来看一下这个方法：

```objc
    context[@"nativeFlushQueueImmediate"] = ^(NSArray<NSArray *> *calls){
      RCTJSCExecutor *strongSelf = weakSelf;
      if (!strongSelf.valid || !calls) {
        return;
      }

      RCT_PROFILE_BEGIN_EVENT(RCTProfileTagAlways, @"nativeFlushQueueImmediate", nil);
      [strongSelf->_bridge handleBuffer:calls batchEnded:NO];
      RCT_PROFILE_END_EVENT(RCTProfileTagAlways, @"js_call");
    };
```

调用了 `nativeFlushQueueImmediate` 方法后就会执行 `handleBuffer:batcheEnded:` 方法来强制 native 来执行 js 需要调用的 native 方法，并且刷新 `_queue` 数组。

因此，`enqueueNativeCall` 方法中的这段代码表示，当上次刷新 `_queue` 数组的时间和当前时间相比超过了 `MIN_TIME_BETWEEN_FLUSHES_MS` 即5ms，那么就会主动调用 native 的 `nativeFlushQueueImmediate` 方法，强制执行，刷新 `_queue`。

注意到，这里清空的操作：`this._queue = [[], [], [], this._callID];`，将 `_queue` 的第四项设置为 `_callID`。这个 `_callID` 每次 `enqueueNativeCall` 的时候都会自增一次，应该只是一个标记，暂时没看出有什么太特别的用处。

### RCT_EXPORT_METHOD()
RN 是如何将方法前加上 `__rct_export__` 的呢？通过 `RCT_EXPORT_METHOD()` 方法。

```objc
#define RCT_EXPORT_METHOD(method) \
  RCT_REMAP_METHOD(, method)

#define RCT_REMAP_METHOD(js_name, method) \
  RCT_EXTERN_REMAP_METHOD(js_name, method) \
  - (void)method

#define RCT_EXTERN_REMAP_METHOD(js_name, method) \
  + (NSArray<NSString *> *)RCT_CONCAT(__rct_export__, \
    RCT_CONCAT(js_name, RCT_CONCAT(__LINE__, __COUNTER__))) { \
    return @[@#js_name, @#method]; \
  }
```

中间的宏为方法补全了 `-(void)` 恢复了完整 OC 方法的定义。这样才能使得 `RCT_EXPORT_METHOD(xxx)` 这样的写法编译器不报错。

下面的宏中 `RCT_CONCAT` 是一个拼接的宏。大致为每一个 `RCT_EXPORT_METHOD` 生成了唯一识别的数字 tag 与 js_name 拼接，然后在前面加上一个 `__rct_export__`，生成了一个返回一个数组的方法。

套用[RN 源码解读（二）](http://awhisper.github.io/2016/07/02/ReactNative源码分析2/)中的一个例子：

假设我们写 `RCT_EXPORT_METHOD(nativeAlert:xxx)` 的时候，`__LINE__` 与 `__COUNTER__` 组合起来的数字 tag 如果是 123456，那么这个内二层宏还会自动生成一个这样的函数：

```objc
+ (NSArray<NSString *> *)__rct_export__123456{ 
    return @[@"", @"nativeAlert:xxx"]; 
}
```

换句话说，一行 `RCT_EXPORT_METHOD(xxxx)`，等于生成了2个函数的实现:
- `-(void)nativeAlert:(NSString *)content withButton:(NSString *)name`
- `+(NSArray<NSString *> *)__rct_export__123456`


## Native 与 JS 间的通信
现在开始讲到 native 和 JS 之间的通信。首先是 native 调用 JS 方法。在 `RCTEventDispatcher.m` 中，我们可以看到各种各样的调用方法：
- `sendAppEventWithName:body:`
- `sendDeviceEventWithName:body:`
- `sendInputEventWithName:body:`

这些方法都在内部调动了 `RCTBatchedBridge` 的 `enqueueJSCall:method:args:completion:` 方法，下面是参考了[React Native通信原理解析(IOS)](http://i.dotidea.cn/2016/05/react-native-communication-principle-for-ios/)画的一张完整的调用流程图：

![native js 通信](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/native_js_通信.png?raw=true)

### tag1
```objc
- (void)_actuallyInvokeAndProcessModule:(NSString *)module
                                 method:(NSString *)method
                              arguments:(NSArray *)args
{
  RCTAssertJSThread();

  __weak __typeof(self) weakSelf = self;
  [_javaScriptExecutor callFunctionOnModule:module
                                     method:method
                                  arguments:args
                                   callback:^(id json, NSError *error) {
                                     [weakSelf _processResponse:json error:error];
                                   }];
}
```

这个方法其实也没什么特别的，主要看一下 `callFunctionOnModule` 方法的 callback，在 callback 中调用了 `_processResponse` 方法，用来处理执行 js 端调用 native 的方法。

### tag2

```objc
- (void)_callFunctionOnModule:(NSString *)module
                       method:(NSString *)method
                    arguments:(NSArray *)args
                  returnValue:(BOOL)returnValue
                 unwrapResult:(BOOL)unwrapResult
                     callback:(RCTJavaScriptCallback)onComplete
{
  // TODO: Make this function handle first class instead of dynamically dispatching it. #9317773
  NSString *bridgeMethod = returnValue ? @"callFunctionReturnFlushedQueue" : @"callFunctionReturnResultAndFlushedQueue";
  [self _executeJSCall:bridgeMethod arguments:@[module, method, args] unwrapResult:unwrapResult callback:onComplete];
}
```

在这个方法中，设置了调用 js 端 batchedBridge 的 `callFunctionReturnFlushedQueue` 或者 `callFunctionReturnResultAndFlushedQueue` 方法。并将 `module`，`method`，`args` 作何成一个数组作为 `arguments` 传入。**由于 native 端需要主动调用的 js 端的方法都是 native 所熟知的模块和方法，所以这里的各个参数都是 string**（只有那些自定义的模块或者方法的调用才会用 id）。

### tag3
在执行完 js 端的方法后，回到 `RCTJSCExecutor` 内执行 `onComplete` 回调:

```objc
- (void)_executeJSCall:(NSString *)method
             arguments:(NSArray *)arguments
          unwrapResult:(BOOL)unwrapResult
              callback:(RCTJavaScriptCallback)onComplete
{
  [self executeBlockOnJavaScriptQueue:^{
		//执行js方法
		...
    onComplete(objcValue, error);
  }];
}
```

这个 `onComplete` 就是上面 tag1 中的 `callback`。

### tag4
js 调用 oc 可以有很多个方法，但是最后方法一定会走到 `handleBuffer:` 方法中去。这个方法在上面的 `MessageQueue` 中也有提到过。入参是一个包含模块、方法、参数的数组。

现在就用到了之前保存的 `_moduleDataByID` 数组。

```objc
RCTModuleData *moduleData = self->_moduleDataByID[moduleID.integerValue];
dispatch_queue_t queue = moduleData.methodQueue;
```

通过传入的 `moduleID` 就可以在数组中找到对应的 `RCTModuleData` 实例，拿出每个 `RCTModuleData` 的 gcd 队列。之后就是在各自队列里执行了。由于是串行队列，同一个 `RCTModuleData` 的方法必须是顺序执行的。

### tag5
这个就是正真通过反射的方式执行 native 代码的方法。由于也是 iOS 菜鸟，对于 invocation 也不是很了解，所以这段代码不是很能看得懂，就不细聊了。各位大神可以仔细跟进。

不过要注意其中的 `processMethodSignature` 方法。这个方法中，RN 会对反射的 selector 进行分析，分析有几个参数，是什么类型等等。其中就在 `addBlockArgument` 这个 block 中，设置了一个输入参数为数组的回调方法：

```objc
RCT_BLOCK_ARGUMENT(^(NSArray *args) {
  [bridge enqueueCallback:json args:args];
});
```

这个 `RCT_BLOCK_ARGUMENT` 宏是用来保存这个回调方法的。因为 `NSInvocation` 不会持有这个 block。其中的 `enqueueCallback:args:` 就是用来回调前面保存的 `_callback` 的。

后面所有的方法调用，大致过程都和 native 调用 js 一样，只不过前者传入的是 `callbackID`，后者传入的是各个模块方法参数名。所以我也就不再展开了。


















## 参考文章
[React Native 从入门到原理](http://www.jianshu.com/p/978c4bd3a759) 看完可以对 RN 有一个初步的认识
[ReactNative iOS源码解析](http://awhisper.github.io/2016/06/24/ReactNative流程源码分析/) native 端讲解的非常详细，需要仔细地跟着走一遍
[React Native通信原理解析(IOS)](http://i.dotidea.cn/2016/05/react-native-communication-principle-for-ios/) native 和 js 都讲解的较为详细，前面那篇看懂了，再来看这篇，可以加深理解。(部分内容有一些区别)

 

