title: React-Native 如何创建原生 View 源码解析
date: 2017/1/24 10:07:12  
categories: React-Native
tags:
	- React-Native
	- 源码解析
---

前一篇，主要讲了 RN 如何初始化一个 `BCTBatchedBridge`，这一篇将进一步研究 RN 是如何通过这个 bridge 创建出真正展示出来的 View 的。

<!--more-->

首先抛出结论：每一个 js `render()` 方法中的 component 都对应于 native 中的一个 `RCTView`。在渲染前，js 会循环取出各个 component 。然后通过 `UIManager` （`NativeModules` 中的一项） 的 `createView` 方法，通过 `BatchedBridge` 调用 native 端的 `RCTUIManager` 的同名方法，创建与 component 相对应的类。对于触摸事件也是类似的。native 端通过 bridge 将触摸事件传给 js，js 在找到对应 component 后，响应各个触摸事件。

下面将分几个阶段解读 RN 是如何显示出来的，依旧是 0.39 版本。最好在理解了前一篇 RN 通信原理后再看。

## native 调用 js

这是整个过程的第一步，从 `RCTRootView *rootView = [[RCTRootView alloc] initWithBridge:_bridge moduleName:moduleName initialProperties:param];` 方法开始。native 告诉 js 我要准备显示 view 了，快点把你这里各个 view 的信息告诉我。先来看一张调用流程图：

![RN native调用js](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/RN_nativecalljs.png?raw=true)

### tag1

这个 `setBridge` 方法是初始化时候执行的方法。还记得前一篇初始化的时候其中的一步设置各个 `RCTModuleData` 的 `instance` 么？其中就会调用 `setBridge` 方法为每个 RCTModuleData 的 `instance` 设置 bridge。在 `RCTUIManager` 类的 `setBridge` 方法中还做了一些其他操作：

```objective-c
- (void)setBridge:(RCTBridge *)bridge{
  ...
      NSMutableDictionary *componentDataByName = [NSMutableDictionary new];
      for (Class moduleClass in _bridge.moduleClasses) {
    	if ([moduleClass isSubclassOfClass:[RCTViewManager class]]) {
          RCTComponentData *componentData = [[RCTComponentData alloc] initWithManagerClass:moduleClass bridge:_bridge];
          componentDataByName[componentData.name] = componentData;
        }
      }
      _componentDataByName = [componentDataByName copy];
  ...
}
```

该方法循环遍历 bridge 中的每一个暴露给 js 的模块名，判断是否是 `RCTViewManager` 的子类。如果是，则为其创建一个 `RCTComponentData` 并将其保存在 `_componentDataByName` 字典中。`RCTViewManager` 是各个 `RCTView` 的控制器的父类，所有 `RCTView` 的方法都在 `RCTViewManager` 的子类里实现。这样就把所有视图控制器保存在了 `RCTUIManager` 中。

### tag2

看到这个方法的名字就知道是在 bundle 加载完成后执行的。 这个方法分为两步。第一步是创建一个 `_contentView` 用来容纳各个 component。第二步就是通过 `BatchedBridge` 调用 js 的方法了。部分代码如下：

```objc
- (void)bundleFinishedLoading:(RCTBridge *)bridge
{
  ...
  _contentView = [[RCTRootContentView alloc] initWithFrame:self.bounds
                                                    bridge:bridge
                                                  reactTag:self.reactTag
                                            sizeFlexiblity:_sizeFlexibility];
  [self runApplication:bridge];
  ...
}
```

其中有一个地方需要说明，就是这里面的 `self.reactTag`。类似于 RN 通信的原理，`reactTag` 是 native 和 js 识别各个 component 的标记。来看一下 `reactTag` 的 getter 方法：

```objc
- (NSNumber *)reactTag
{
  RCTAssertMainQueue();
  if (!super.reactTag) {
    self.reactTag = [_bridge.uiManager allocateRootTag];
  }
  return super.reactTag;
}

- (NSNumber *)allocateRootTag
{
  NSNumber *rootTag = objc_getAssociatedObject(self, _cmd) ?: @1;
  objc_setAssociatedObject(self, _cmd, @(rootTag.integerValue + 10), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
  return rootTag;
}
```

每一个根视图都必须有一个独一无二的 `ractTag`。这个 tag 从1开始，每次自增10，也就是 1，11，21，31...的形式，以此类推。

这里要吐槽一下 facebook 的程序员。没想到外国的程序员也会把单词拼错。上面方法中 `sizeFlexiblity` 少了个 `i`。这是要输入法背锅么？

### tag3 

该方法首先为 `rootView` 新建了 `RCTTouchHandler` 实例，并设置为其 `GestureRecognizer`。其次执行了 `registerRootView:withSizeFlexibility:` 方法。这个方法将 `rootView` 保存在 `RCTUIManager` 的 `_viewRegistry` 字典中，其中键是 `rootView` 的 `reactTag`。此时，`RCTUIManager` 中设置并保存好了两个比较重要的字典：`_viewRegistry`,`_componentDataByName`。

### tag4

这个方法就比较直观了：

```objc
- (void)runApplication:(RCTBridge *)bridge
{
  NSString *moduleName = _moduleName ?: @"";
  NSDictionary *appParameters = @{
    @"rootTag": _contentView.reactTag,
    @"initialProps": _appProperties ?: @{},
  };

  RCTLogInfo(@"Running application %@ (%@)", moduleName, appParameters);
  [bridge enqueueJSCall:@"AppRegistry"
                 method:@"runApplication"
                   args:@[moduleName, appParameters]
             completion:NULL];
}
```

就是调用 `enqueueJSCall:method:args:completion:` 方法。如果前一篇看懂了的话，就会知道，这个方法调用的是 js 端 `AppRegistry.js` 的 `runApplication` 方法，需要创建的 `rootView` 以 String 的方式通过 `moduleName` 传入。

到这一步，Native 调用 JS 就结束了，接下来就是 js 端的执行。

## js 调用 native

js 端的方法调用过程还是比较清晰的。来看一下下面的流程图：

![RN native调用js](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/RN_jscallnative.png?raw=true)

### tag1

在 native 调用 `enqueueJSCall:method:args:completion` 后，我们来看看 `AppRegistry` 的 `runApplication` 方法。

```javascript
  runApplication: function(appKey: string, appParameters: any): void {
	...
    runnables[appKey].run(appParameters);
  },
```

`appKey` 就是上面的 `moduleName`，通过它拿到 `runnables` 字典中对应的对象，执行其 `run` 方法。这个字典怎么来的呢？往下看。

### tag2

RN 的官方文档中第一篇就说明了，在定义好一个 component 后，需要在 `index.ios.js` 中使用 `AppRegistry.registerComponent()` 方法注册:

```javascript
import HelloWorldView from './app/view/HelloWorldView';
AppRegistry.registerComponent('HelloWorldView', () => HelloWorldView);
```

那么 `registerComponent` 做了什么呢？

```javascript
  registerComponent: function(appKey: string, getComponentFunc: ComponentProvider): string {
    runnables[appKey] = {
      run: (appParameters) =>
        renderApplication(getComponentFunc(), appParameters.initialProps, appParameters.rootTag)
    };
    return appKey;
  }
```

这里就设置了前面用到的 `runnables` 字典，并且为字典内对象创建了 `run` 方法。前面执行的 `run` 方法，其实就是执行 `renderApplication` 方法。`renderApplication` 在 RootView 外层又嵌套了一层 `AppContainer`。

### tag3

这个方法内部就比较复杂了。其中:

```javascript
var instance = instantiateReactComponent(nextWrappedElement, false);
```

通过 `instantiateReactComponent` 方法将整个 View 的 JSX 转换为 `ReactCompositeComponentWrapper` 实例。这也就是 Reactjs 的精髓之一。好吧。并没有花时间去细读，臣妾做不到啊。有兴趣的话可以找找类似于 [reactjs源码分析-上篇（首次渲染实现原理）](http://purplebamboo.github.io/2015/09/15/reactjs_source_analyze_part_one/) 的文章学习一下。之后就是长长的调用栈了，循环递归每一个 component 执行 `createView`。

### tag4

在 `ReactNativeBaseComponent.js` 中调用了 `UIManager.createView()`，但是这个方法点不进去，`UIManager.js` 中没有这个方法。开始我比较迷惑，后来细读了一下发现：

```javascript
const { UIManager } = NativeModules;
```

也就是说，这个 `UIManager` 是 native 中的一个 module，这个 `createView()` 是 native 中暴露出来的一个方法。

```javascript
    UIManager.createView(
      tag,
      this.viewConfig.uiViewClassName,
      nativeTopRootTag,
      updatePayload
    );
```

这个方法会将当前 component 的标记，名字，根标记，属性方法等一起传给 native。其中 `tag` 是 `ReactNativeTagHandles.allocateTag()` 生成的。每次调用该方法，`tag` 都会加一（会略过1结尾的，那些是留给根 component 的）。

至此，js 调用 native 的部分就结束了。回顾一下，主要就是循环每一个 component，为每个 component 分配一个独有的 `tag` 然后调用 `createView` 方法。

## Native createView 的过程

来到 native，`RCTUIManager` 暴露出了许多方法供 js 调用。来看一下刚才 js 调用的 `createView` 方法：

```objc
RCT_EXPORT_METHOD(createView:(nonnull NSNumber *)reactTag
                  viewName:(NSString *)viewName
                  rootTag:(__unused NSNumber *)rootTag
                  props:(NSDictionary *)props)
{
  RCTComponentData *componentData = _componentDataByName[viewName];
  ...
  __weak RCTUIManager *weakManager = self;
  dispatch_async(dispatch_get_main_queue(), ^{
    RCTUIManager *uiManager = weakManager;
    if (!uiManager) {
      return;
    }
    UIView *view = [componentData createViewWithTag:reactTag];
    if (view) {
      [componentData setProps:props forView:view]; // Must be done before bgColor to prevent wrong default
      ...
      uiManager->_viewRegistry[reactTag] = view;
    }
  });
}
```

`_componentDataByName` 上面介绍过了，可以拿到前面保存的 js 将要创建的 view 的 `RCTComponentData`。然后在主线程中使用 `createViewWithTag:` 方法创建 View。通过 `RCTComponentData` 中保存的 `_managerClass` 可以拿到对应 View 的 `manager`，然后调用其 `view` 方法创建 view。

```objc
- (UIView *)createViewWithTag:(NSNumber *)tag
{
  UIView *view = [self.manager view];
  view.reactTag = tag;
	...
  return view;
}

- (RCTViewManager *)manager
{
  if (!_manager) {
    _manager = [_bridge moduleForClass:_managerClass];
  }
  return _manager;
}
```



通过 `setProps:forView:` 方法将每个 view 的 props 保存下来。这里又要用到反射的方式，还是比较复杂的。具体就不说了。只要清楚，它会生成一个 block，每次都用这个 block，都会**调用对应属性的 setter 方法**将值赋给该属性即可。

最后，把设置好的 view 保存到 `_viewRegistry` 中。js 和 native 端都会保存好这个 `tag` 和 view 的映射表。这里的 `_viewRegistry` 就是 native 端保存的字典。

这里只是介绍了如何通过 js 传来的 `tag` 和 `moduleName` 创建出对应的 native 端的 view 的。当然，创建和展示中间还隔着 measure、layout之类的步骤，这也是 React 需要做的，这个我就不聊了，太深入了聊不动，容易聊爆。

![聊爆啦](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/RN_liaobaole.png?raw=true)

## 自建组件

现在我们知道 view 是如何创建出来的了。facebook 封装的组件和自建的组件原理上是一致的。现在来研究一下自建组件的创建过程。

### 基本设置

前面说到拿到 view 的 `manager` 调用其 `view` 方法。那么 `manager` 是个什么呢？`manager` 相当于一个 Controller，负责创建和控制 view。文档上关于创建 `RCTViewManager` 主要有三步：

1. 创建一个子类
2. 添加 `RCT_EXPORT_MODULE()` 标记宏
3. 实现 `-(UIView *)view` 方法


这三步是最基本的三步，作用也挺明显的。继承 `RCTViewManager` 作为一个标记，这样 `RCTUIManager` 在设置 `bridge`  的时候就能够将该 `manager` 放入 `_componentDataByName` 数组中了。添加 `RCT_EXPORT_MODULE()` 标记宏后才能将该类暴露给 js，才能让 js 调用该类方法。实现 `-(UIView *)view` 方法就是上面 `[self.manager view]` 调用的方法。

### 设置属性

上面只是创建了一个 view，更复杂一点的情况就是需要控制 view 的属性。RN 提供了两个宏：

- `RCT_EXPORT_VIEW_PROPERTY(name,type)`
- `RCT_CUSTOM_VIEW_PROPERTY(name,type,viewClass){...}`

来看一下宏定义的实现：

```objc
// 常规导出
#define RCT_EXPORT_VIEW_PROPERTY(name, type) \
+ (NSArray<NSString *> *)propConfig_##name { return @[@#type]; }
// 自定义导出
#define RCT_CUSTOM_VIEW_PROPERTY(name, type, viewClass) \
RCT_REMAP_VIEW_PROPERTY(name, __custom__, type)         \
- (void)set_##name:(id)json forView:(viewClass *)view withDefaultView:(viewClass *)defaultView

#define RCT_REMAP_VIEW_PROPERTY(name, keyPath, type) \
+ (NSArray<NSString *> *)propConfig_##name { return @[@#type, @#keyPath]; }
```

关于 `##` 和 `@#` 前一篇也有涉及。操作符 `##` 用来实现宏中 token 的连接， `@#` 实现将宏中的参数转化为字符串。比如 `RCT_EXPORT_VIEW_PROPERTY(placeholder, NSString)` 这样一个属性，将会被转化为下面这个方法：

```objc
+ (NSArray<NSString *> *)propConfig_placeholder { return @[@"NSString"]; }
```

但如果要对属性进行也谢操作的话，比如：

```objc
RCT_CUSTOM_VIEW_PROPERTY(fontSize, NSNumber, RCTTextView)
{
  view.font = [RCTFont updateFont:view.font withSize:json ?: @(defaultView.font.pointSize)];
}
```

这样的自定义属性，则会转化为如下方法：

```objc
+ (NSArray<NSString *> *)propConfig_fontSize { return @[@"NSNumber", @"__custom__"]; }
- (void)set_fontSize:(id)json forView:(RCTTextView *)view withDefaultView:(RCTTextView *)defaultView{
  view.font = [RCTFont updateFont:view.font withSize:json ?: @(defaultView.font.pointSize)];
}
```

其中 `json` 代表了 JS 中传递的尚未解析的原始值。`view` 代表访问的对应的视图实例。`defaultView` 在 JS 发送 null 的时候，可以把视图的这个属性重置回默认值。

这里说一下这个 `defaultView`，这是 `RCTComponentData` 中的一个属性，不用在意也不用自己设置它。在 `json` 为空的时候，就会去自己创建一个当前 view 的实例作为 `defaultView`：

```objc
//RCTComponentData.m propBlockForKey:inDictionary
if (!json && !strongSelf->_defaultView) {
  // Only create default view if json is null
  strongSelf->_defaultView = [strongSelf createViewWithTag:nil];
}
```

宏定义的方法在上面说过的 `setProps:forView:` 方法里调用，目的是验证从 js 端传递过来的属性是否在 native 里注册过。

至于还有一些非常规的需要用到 `RCTConvert` 以及事件的使用方式可以参考文档，不深入了。

### 创建 View

前面说过了如何创建 Controller，现在说一下这个 view。

```objc
- (UIView *)view
{
  return [[MKMapView alloc] init];
}
```

这是官方文档里的例子，在 `manager` 中的 `view` 方法里创建了一个 `MKMapView` 的视图。那么 RN 是如何设置 `MKMapView` 的 frame 的呢？不用担心，RN 实现了 `UIView` 的范畴 `UIView(React)`，其中就实现了 `reactSetFrame` 方法，会自己将计算好的大小设置给 view，如果你想在 view 设置好大小后做什么事，你就可以通过重写这个方法。另外，当 js 端的一些属性变化时，就会触发 native 相应 view 的相应属性的 setter 方法，可以自己实现这些 setter 方法，来做一些别的处理，比如在 `RCTImageView` 里，只要任意属性发生变化，都会触发 `reloadImage` 方法。

## 终

总算通过这两篇将 RN 稍微梳理了一下，跟进起来不容易啊，都是泪。本来想研究一下，写一篇 “RN view 显示原理” 的。但是读着读着发现 Reactjs 并不能看懂，那还写个蛋的显示原理啊喂。最终就忽略了 view 的层次关系以及大小位置等等的转换原理，写了这么一篇。虽然写的水平不高，但是我相信如果你想更深入的了解一下 react native，读一下本篇还是能有些许帮助的。毕竟为了看懂，我也读了好久好久~~~~~~~

也许会有下一篇，也许会挑一个 RN 封装的组件具体看看它是如何实现的~














