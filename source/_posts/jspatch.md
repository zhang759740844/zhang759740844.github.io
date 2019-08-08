outitle: JSPatch 源码解析(一)
date: 2019/8/1 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

JSPatch 虽然被禁，但是它的源码是非常值得学习的。可以说，是我看过的各个库中设计的最巧妙也是知识点最多的开源库。非常有学习价值。

<!--more-->

这是这个源码解析的第一部分

## Demo 演示

JSPatch 的主要功能涵盖于三个文件中：`JPEngine.h`，`JPEngine.m`，`JSPatch.js`。由于语法规则比较多，这里就不进行用法的介绍了。下面跟着 JSPatch 附带的 demo 了解 JSPatch 的修复过程。

### 修复的 js 文件

通过 js 进行热修复，demo 中定义了要修复的类为 `JPTableViewController`，它是一个 UITableViewController。在这个 修复类中，增加了 data 这个 property，并且提供了完整的 UITableView 显示所需要的方法：

```js
defineClass('JPTableViewController : UITableViewController <UIAlertViewDelegate>', ['data'], {
  dataSource: function() {
    var data = self.data();
    if (data) return data;
    var data = [];
    for (var i = 0; i < 20; i ++) {
      data.push("cell from js " + i);
    }
    self.setData(data)
    return data;
  },
  numberOfSectionsInTableView: function(tableView) {
    return 1;
  },
  tableView_numberOfRowsInSection: function(tableView, section) {
    return self.dataSource().length;
  },
  tableView_cellForRowAtIndexPath: function(tableView, indexPath) {
    var cell = tableView.dequeueReusableCellWithIdentifier("cell") 
    if (!cell) {
      cell = require('UITableViewCell').alloc().initWithStyle_reuseIdentifier(0, "cell")
    }
    cell.textLabel().setText(self.dataSource()[indexPath.row()])
    return cell
  },
  tableView_heightForRowAtIndexPath: function(tableView, indexPath) {
    return 60
  },
  tableView_didSelectRowAtIndexPath: function(tableView, indexPath) {
     var alertView = require('UIAlertView').alloc().initWithTitle_message_delegate_cancelButtonTitle_otherButtonTitles("Alert",self.dataSource()[indexPath.row()], self, "OK",  null);
     alertView.show()
  },
  alertView_willDismissWithButtonIndex: function(alertView, idx) {
    console.log('click btn ' + alertView.buttonTitleAtIndex(idx).toJS())
  }
})

```

### 加载修复的 js 文件

加载修复的 js 文件是在应用启动完毕的回调中完成的：

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // JSPatch 初始化
    [JPEngine startEngine];
    NSString *sourcePath = [[NSBundle mainBundle] pathForResource:@"demo" ofType:@"js"];
    NSString *script = [NSString stringWithContentsOfFile:sourcePath encoding:NSUTF8StringEncoding error:nil];
  	// 执行修复脚本
    [JPEngine evaluateScript:script];
  
  	...
}
```

先调用 `startEngine` 初始化 JSPatch，这一步的作用主要是创建一个 JSContext，并在其中注入提供给 JS 调用的方法。随后加载提供修复的 js 文件。通过刚才创建的 JSContext 执行。

当执行到 `JPTableViewController` 的时候，修复方法替换了原本的各种 UITableView 回调执行。

## 执行流程

### 初始化 JSContext

初始化 JSPatch 通过 `startEngine` 方法完成，这个方法为 JSContext 注入了多种方法，并且初始化了 OC 端的各个属性。方法非常长，因此做了非常详尽的注释：

```objc
// SECTION 开始JSPatch
+ (void)startEngine
{
    // 创建 JSContext 实例
    JSContext *context = [[JSContext alloc] init];
    
#ifdef DEBUG
    // 将 lldb 中 po 打印出来的字符串传递给 js
    context[@"po"] = ^JSValue*(JSValue *obj) {
        id ocObject = formatJSToOC(obj);
        return [JSValue valueWithObject:[ocObject description] inContext:_context];
    };
    // js 中打印调用堆栈，类似于 lldb 中的 bt
    context[@"bt"] = ^JSValue*() {
        return [JSValue valueWithObject:_JSLastCallStack inContext:_context];
    };
#endif
    // 在 OC 中定义 Class， 输入类名，实例方法数组 和 类方法数组
    context[@"_OC_defineClass"] = ^(NSString *classDeclaration, JSValue *instanceMethods, JSValue *classMethods) {
        return defineClass(classDeclaration, instanceMethods, classMethods);
    };
    // 在 OC 中定义协议，输入协议名
    context[@"_OC_defineProtocol"] = ^(NSString *protocolDeclaration, JSValue *instProtocol, JSValue *clsProtocol) {
        return defineProtocol(protocolDeclaration, instProtocol,clsProtocol);
    };
    // 调用 oc 的实例方法
    context[@"_OC_callI"] = ^id(JSValue *obj, NSString *selectorName, JSValue *arguments, BOOL isSuper) {
        return callSelector(nil, selectorName, arguments, obj, isSuper);
    };
    // 调用 oc 的类方法
    context[@"_OC_callC"] = ^id(NSString *className, NSString *selectorName, JSValue *arguments) {
        return callSelector(className, selectorName, arguments, nil, NO);
    };
    // 将 JS 对象转为 OC 对象
    context[@"_OC_formatJSToOC"] = ^id(JSValue *obj) {
        return formatJSToOC(obj);
    };
    // 将 OC 对象转为 JS对象
    context[@"_OC_formatOCToJS"] = ^id(JSValue *obj) {
        return formatOCToJS([obj toObject]);
    };
    // 获取 js 定义 class 的时候添加的自定义的 props
    context[@"_OC_getCustomProps"] = ^id(JSValue *obj) {
        id realObj = formatJSToOC(obj);
        return objc_getAssociatedObject(realObj, kPropAssociatedObjectKey);
    };
    // 设置 js 定义 class 时候自定义的 props
    context[@"_OC_setCustomProps"] = ^(JSValue *obj, JSValue *val) {
        id realObj = formatJSToOC(obj);
        objc_setAssociatedObject(realObj, kPropAssociatedObjectKey, val, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    };
    // js 对象弱引用, 主要用于 js 对象会传递给 OC 的情况
    context[@"__weak"] = ^id(JSValue *jsval) {
        id obj = formatJSToOC(jsval);
        return [[JSContext currentContext][@"_formatOCToJS"] callWithArguments:@[formatOCToJS([JPBoxing boxWeakObj:obj])]];
    };
    // js 对象强引用，将 js 弱引用r对象转为强引用
    context[@"__strong"] = ^id(JSValue *jsval) {
        id obj = formatJSToOC(jsval);
        return [[JSContext currentContext][@"_formatOCToJS"] callWithArguments:@[formatOCToJS(obj)]];
    };
    // 获取传入类的父类名
    context[@"_OC_superClsName"] = ^(NSString *clsName) {
        Class cls = NSClassFromString(clsName);
        return NSStringFromClass([cls superclass]);
    };
    // 设置标识位，在 formatOCToJS 的时候是否自动将 NSString  NSArray 等对象自动转为 js 中的 string，array 等。
    // 如果设置为 false，则会当成一个对象，使用 JPBoxing 包裹
    context[@"autoConvertOCType"] = ^(BOOL autoConvert) {
        _autoConvert = autoConvert;
    };
    // 设置标识位，在 formatOCToJS 的时候是否直接将 OC 的 NSNumber 类型转为 js 的 string 类型
    context[@"convertOCNumberToString"] = ^(BOOL convertOCNumberToString) {
        _convertOCNumberToString = convertOCNumberToString;
    };
    // 在 js 中调用 include 方法，可以在一个 js 文件中加载其他 js 文件
    context[@"include"] = ^(NSString *filePath) {
        NSString *absolutePath = [_scriptRootDir stringByAppendingPathComponent:filePath];
        if (!_runnedScript) {
            _runnedScript = [[NSMutableSet alloc] init];
        }
        if (absolutePath && ![_runnedScript containsObject:absolutePath]) {
            [JPEngine _evaluateScriptWithPath:absolutePath];
            [_runnedScript addObject:absolutePath];
        }
    };
    // 提供一个文件名返回完整的文件路径
    context[@"resourcePath"] = ^(NSString *filePath) {
        return [_scriptRootDir stringByAppendingPathComponent:filePath];
    };
    // 延时在主线程中执行。
    context[@"dispatch_after"] = ^(double time, JSValue *func) {
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(time * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            [func callWithArguments:nil];
        });
    };
    // 在主线程中异步执行
    context[@"dispatch_async_main"] = ^(JSValue *func) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [func callWithArguments:nil];
        });
    };
    // 在主线程中同步执行
    context[@"dispatch_sync_main"] = ^(JSValue *func) {
        // 先判断是不是主线程，是就直接执行，不是就用 dispatch_sync
        if ([NSThread currentThread].isMainThread) {
            [func callWithArguments:nil];
        } else {
            dispatch_sync(dispatch_get_main_queue(), ^{
                [func callWithArguments:nil];
            });
        }
    };
    // 异步执行
    context[@"dispatch_async_global_queue"] = ^(JSValue *func) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            [func callWithArguments:nil];
        });
    };
    // 释放二级指针指向的对象
    context[@"releaseTmpObj"] = ^void(JSValue *jsVal) {
        if ([[jsVal toObject] isKindOfClass:[NSDictionary class]]) {
            void *pointer =  [(JPBoxing *)([jsVal toObject][@"__obj"]) unboxPointer];
            id obj = *((__unsafe_unretained id *)pointer);
            @synchronized(_TMPMemoryPool) {
                [_TMPMemoryPool removeObjectForKey:[NSNumber numberWithInteger:[(NSObject*)obj hash]]];
            }
        }
    };
    // js 调用 oc 方法，打印 js 对象到 oc 控制台
    context[@"_OC_log"] = ^() {
        NSArray *args = [JSContext currentArguments];
        for (JSValue *jsVal in args) {
            id obj = formatJSToOC(jsVal);
            NSLog(@"JSPatch.log: %@", obj == _nilObj ? nil : (obj == _nullObj ? [NSNull null]: obj));
        }
    };
    // js 捕捉到的 exception 打印在 oc 中
    context[@"_OC_catch"] = ^(JSValue *msg, JSValue *stack) {
        _exceptionBlock([NSString stringWithFormat:@"js exception, \nmsg: %@, \nstack: \n %@", [msg toObject], [stack toObject]]);
    };
    context.exceptionHandler = ^(JSContext *con, JSValue *exception) {
        NSLog(@"%@", exception);
        _exceptionBlock([NSString stringWithFormat:@"js exception: %@", exception]);
    };
    // js 中的 nsnull 就是 oc 中的一个普通对象
    _nullObj = [[NSObject alloc] init];
    context[@"_OC_null"] = formatOCToJS(_nullObj);
    
    _context = context;
    
    // 各种初始化
    _nilObj = [[NSObject alloc] init];
    _JSMethodSignatureLock = [[NSLock alloc] init];
    _JSMethodForwardCallLock = [[NSRecursiveLock alloc] init];
    _registeredStruct = [[NSMutableDictionary alloc] init];
    _currInvokeSuperClsName = [[NSMutableDictionary alloc] init];
    
    // 找到 JSPatch.js 的路径
    NSString *path = [[NSBundle bundleForClass:[self class]] pathForResource:@"JSPatch" ofType:@"js"];
    if (!path) _exceptionBlock(@"can't find JSPatch.js");
    // 加载路径上的文件
    NSString *jsCore = [[NSString alloc] initWithData:[[NSFileManager defaultManager] contentsAtPath:path] encoding:NSUTF8StringEncoding];
    
    // 执行 JSPatch.js 
    if ([_context respondsToSelector:@selector(evaluateScript:withSourceURL:)]) {
        [_context evaluateScript:jsCore withSourceURL:[NSURL URLWithString:@"JSPatch.js"]];
    } else {
        [_context evaluateScript:jsCore];
    }
}
```

注入 JSContext 的方法会在之后用到的时候详细解释。在注入方法完成后，执行了 JSPatch js 端的初始化代码。

### 初始化 js 部分

JSPatch 的 js 端初始化部分主要做了两件事：

- 为 js 中的 Object 对象增加方法
- 为 JSContext 的全局对象增加各种属性与方法

```js
// Object 增加的几个方法：
function __c (methodName) {...}
function super () {...}
function performSelectorInOC () {...}
function performSelector () {...}

// JSContext 全局对象增加的方法：
function _formatOCToJS (obj) {...}
function require () {...}
function defineClass (declaration, properties, instMethods, clsMethods) {...}
function defineProtocol (declaration, instProtos, clsProtos) {...}
function block (args, cb) {...}
function defineJSClass (declaration, instMethods, clsMethods) {...}

// JSContext 全局对象增加的属性
global.YES = 1
global.NO = 0
global.nsnull = _OC_null (JSContext 传入，上面有提及)
```

### 改写修复文件

现在开始执行修复文件，拿到 js 代码后调用了`[JPEngine evaluteScript:script];` 方法。这个方法兜兜转转来到了 JPEngine 的  `_evaluteScript:withSourceURL:` 方法中。方法实现很简单，但是却是实现修复的第一个重点：

```objc
// 修改 js 修复文件方法的调用方式
+ (JSValue *)_evaluateScript:(NSString *)script withSourceURL:(NSURL *)resourceURL
{
    if (!_regex) {
        _regex = [NSRegularExpression regularExpressionWithPattern:_regexStr options:0 error:nil];
    }
    // 使用正则表达式替换方法调用方式，将 alloc() 这样的函数调用，替换为 __c("alloc")() 形式
    NSString *formatedScript = [NSString stringWithFormat:@";(function(){try{\n%@\n}catch(e){_OC_catch(e.message, e.stack)}})();", [_regex stringByReplacingMatchesInString:script options:0 range:NSMakeRange(0, script.length) withTemplate:_replaceStr]];
    // 执行处理后的修复方法
    @try {
        if ([_context respondsToSelector:@selector(evaluateScript:withSourceURL:)]) {
            return [_context evaluateScript:formatedScript withSourceURL:resourceURL];
        } else {
            return [_context evaluateScript:formatedScript];
        }
    }
    @catch (NSException *exception) {
        _exceptionBlock([NSString stringWithFormat:@"%@", exception]);
    }
    return nil;
}
```

在 js 中执行形如 `UIView.alloc().init()` 这样的调用方式需要预先将要执行的方法定义在对象中，否则一定会报错：`UIView.alloc is not a function`。但是预先定义的方式工作量巨大且初始化的效率很低，不适用于涉及到大量类的修复。

换一种思路，由于 OC 动态性的便利，使得我们只需要从 js 端传递要执行的对象，方法名和参数，就可以通过动态调用的方式执行相应方法。而传递这三个要素，只要一个通用方法就可以实现。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/jspatch_1.png?raw=true)

JSPatch 作者考虑到开发者的习惯，仍然保留了 `UIView.alloc().init()` 这样的链式写法。但是在初始化的时候对实现内容作了正则替换，将匹配到的方法名取出，改为调用一个通用方法 `__c()`，并将方法名作为参数传入。因此，做了正则替换后的实现变为：`UIView.__c('alloc')().__c('init')()`。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/jspatch_2.png?raw=true)

### 定义要重写的类方法与实例方法

#### js 端定义类

要修复bug那么肯定要定位到要修复的类的某个方法。js 全局对象下的方法 `defineClass` 提供了这个入口。方法比较长，总共做了几件事，如下所示：

1. 设置关联属性
2. hook 定义的方法，并将修改后的方法传给 oc
3. 将该类的方法实现保存到 `__ocCls` 中
4. 创建全局类对象

接下来我们一个一个看。

##### 设置关联属性

```js
global.defineClass = function(declaration, properties, instMethods, clsMethods) {
  ...
 	// 如果存在 properties，在实例方法列表中增加各种 get set 方法
  if (properties) {
    properties.forEach(function(name){
      // 设置属性的 get 方法
      if (!instMethods[name]) {
        // 将 get 方法设置到实例方法中
        instMethods[name] = _propertiesGetFun(name);
      }
      // 设置属性的 set 方法
      var nameOfSet = "set"+ name.substr(0,1).toUpperCase() + name.substr(1);
      if (!instMethods[nameOfSet]) {
        // 将 set 方法设置到实例方法中
        instMethods[nameOfSet] = _propertiesSetFun(name);
      }
    });
  }
  ...
}
```

oc 的属性都需要提供其 get set 方法。因此，这一段代码主要就是查看实例方法中有没有相应的 get set 方法，没有的话就添加。关联属性的 get set 方法通过 `_propertiesGetFun` 和 `_propertiesSetFun` 完成：

```js
// 返回属性的 get 方法
var _propertiesGetFun = function(name){
  return function(){
    var slf = this;
    if (!slf.__ocProps) {
      // 获取 oc 的关联属性
      var props = _OC_getCustomProps(slf.__obj)
      if (!props) {
        props = {}
        _OC_setCustomProps(slf.__obj, props)
      }
      // 将 oc 的关联属性赋给 js 端对象的 __ocProps
      slf.__ocProps = props;
    }
    return slf.__ocProps[name];
  };
}

// 返回属性的 set 方法
var _propertiesSetFun = function(name){
  return function(jval){
    var slf = this;
    if (!slf.__ocProps) {
      var props = _OC_getCustomProps(slf.__obj)
      if (!props) {
        props = {}
        _OC_setCustomProps(slf.__obj, props)
      }
      slf.__ocProps = props;
    }
    slf.__ocProps[name] = jval;
  };
}
```

oc 端设置关联对象的方法如下：

```objc
context[@"_OC_getCustomProps"] = ^id(JSValue *obj) {
    id realObj = formatJSToOC(obj);
    return objc_getAssociatedObject(realObj, kPropAssociatedObjectKey);
};
context[@"_OC_setCustomProps"] = ^(JSValue *obj, JSValue *val) {
    id realObj = formatJSToOC(obj);
    objc_setAssociatedObject(realObj, kPropAssociatedObjectKey, val, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
};
```

可以看到，无论 js 端设置了多少 property，在 oc 端，都作为一个属性保存在 key 为 `kPropAssociatedObjectKey` 的关联属性中。oc 端拿到了这个关联属性后返回给 js 端，js 端的对象则以 `_ocProps` 属性接收。两者指向的地址相同，因此，之后对于 property 的改变只需直接修改 js 端对象的 `_ocProps` 属性就行。

##### hook 方法并传给 oc

oc 中实现 aop 非常麻烦，而 js 端直接操作函数指针就可以完成。

```js
global.defineClass = function(declaration, properties, instMethods, clsMethods) {
  ...
  // realClsName 是 js 中直接截取的类名
  var realClsName = declaration.split(':')[0].trim()
  // 预处理要定义的方法，对方法进行切片，处理参数
  _formatDefineMethods(instMethods, newInstMethods, realClsName)
  _formatDefineMethods(clsMethods, newClsMethods, realClsName)
  // 在 OC 中定义这个类，返回的值类型为 {cls: xxx, superCls: xxx}
  var ret = _OC_defineClass(declaration, newInstMethods, newClsMethods)
  // className 是从 OC 中截取的 cls 的名字。本质上和 realClsName 是一致的
  var className = ret['cls']
  var superCls = ret['superCls']
  ...
}
```

主要过程在于 `_formatDefineMethods` 中：

```js
// 对 js 端定义的 method 进行预处理，取出方法的参数个数。hook 方法，预处理方法的参数，将其转为 js 对象。
var _formatDefineMethods = function(methods, newMethods, realClsName) {
  for (var methodName in methods) {
    if (!(methods[methodName] instanceof Function)) return;
    (function(){
      var originMethod = methods[methodName]
      // 把原来的 method 拿出来，新的 method 变成了一个数组，第一个参数是原来方法的调用参数的个数，第二个参数是
      // 因为runtime修复类的时候无法直接解析js实现函数，也就无法知道参数个数，但方法替换的过程需要生成方法签名，所以只能从js端拿到js函数的参数个数，并传递给OC。
      newMethods[methodName] = [originMethod.length, function() {
        try {
          // js 端执行的方法，需要先把参数转为 js 的类型
          var args = _formatOCToJS(Array.prototype.slice.call(arguments))
          // 暂存之前的 self 对象
          var lastSelf = global.self
          // oc 调用 js 方法的时候，默认第一个参数是 self
          global.self = args[0]
          if (global.self) global.self.__realClsName = realClsName
          // oc 调用方法的时候前两个参数分别为 caller 和 Selecter。真正调用 js 方法需要把这两个参数先去除
          args.splice(0,1)
          // 调用 js 方法
          var ret = originMethod.apply(originMethod, args)
          // 恢复 原始的 self 指向
          global.self = lastSelf
          return ret
        } catch(e) {
          _OC_catch(e.message, e.stack)
        }
      }]
    })()
  }
}
```

这里存在三个要点：

1. 这个方法是要添加到 oc 端的，oc 端需要知道参数个数，但是 oc 端无法直接获取，只能通过解析方法名。因此就把解析参数个数的工程放在了 js 中进行。js 端在将方法传给 oc 前，先把参数个数拿到，然后以数组形式传递。
2. oc 调用 js 方法时传递的参数需要预处理，比如调用对象原本是 js 传递过去的，就会被 `{__obj: xxx}` 包裹，再比如 oc 的空对象的处理。
3. oc 传来的参数第一个是方法的调用上下文，第二个是 Selector。因此需要把调用上下文设置给全聚的 self，以便在方法中使用。在调用方法前，还需要把这个上下文和 Selector 从参数列表中取出，因为 js 调用的时候是不需要这两个参数的。

js 方法预处理完成后，就会调用 `_OC_defineClass` 方法，在 OC 中添加相应方法。这个方法是 JSPatch 中最重要的方法了，在后面会解释，先跳过。

##### 将该类的方法实现保存到 `__ocCls` 中

对于定义好的方法，js 端需要把这些方法保存起来。毕竟真正的实现还是在 js 端进行的。因此定义了一个所有类的方法的暂存地：`__ocCls`。所有的类的所有方法都会被保存在这个对象中。 

```js
global.defineClass = function(declaration, properties, instMethods, clsMethods) {
  ...
  // 初始化该类的类方法和实例方法到 _ocCls 中
  _ocCls[className] = {
    instMethods: {},
    clsMethods: {},
  }
  // 如果父类被 defineClass 过，那么要把父类的方法扔到子类中去。子类调用父类中实现的方法的时候，直接调用
  if (superCls.length && _ocCls[superCls]) {
    for (var funcName in _ocCls[superCls]['instMethods']) {
      _ocCls[className]['instMethods'][funcName] = _ocCls[superCls]['instMethods'][funcName]
    }
    for (var funcName in _ocCls[superCls]['clsMethods']) {
      _ocCls[className]['clsMethods'][funcName] = _ocCls[superCls]['clsMethods'][funcName]
    }
  }
  // 把方法存到 _ocCls 对应的类中。和 _formatDefineMethods 的差别在于这个方法不需要把参数个数提取出来
  _setupJSMethod(className, instMethods, 1, realClsName)
  _setupJSMethod(className, clsMethods, 0, realClsName)
  ...
}
```

如果父类实现了某些方法，那么子类中需要先把这些方法保存起来。这样如果调用了子类没有实现的这些方法的时候就可以直接调用父类相应的实现。

这个想法是对的，但是我认为有点问题在于，这样的话就需要先对父类 `defineClass` 才能定义子类，否则子类就没法拿到父类实现的方法了。因此，定义类的时候一定要注意，先定义父类的 class，再定义子类的。

`_setupJSMethod` 其实也是一个很简单的方法，其实就是把上下文名从 this 切换为 self：

```js
// 替换 this 为 self
var _wrapLocalMethod = function(methodName, func, realClsName) {
  return function() {
    var lastSelf = global.self
    global.self = this
    this.__realClsName = realClsName
    var ret = func.apply(this, arguments)
    global.self = lastSelf
    return ret
  }
}

// 保存方法到 _ocCls 中
var _setupJSMethod = function(className, methods, isInst, realClsName) {
  for (var name in methods) {
    var key = isInst ? 'instMethods': 'clsMethods',
        func = methods[name]
    _ocCls[className][key][name] = _wrapLocalMethod(name, func, realClsName)
  }
}
```

> js 端定义这个 `__ocCls` 的目的是在 JS 端执行定义的方法并且调用到其他定义方法的时候，可以不用重走一遍 OC 的消息转发，而是直接调用 js 端方法，加快方法执行速度。

##### 通过 require 方法创建全局类对象

`defineClass` 的最终返回了一个 `require()` 方法产生的对象：

```js
global.defineClass = function(declaration, properties, instMethods, clsMethods) {
  ...
  return require(className)
}
```

在 `require(xxx)` 某一个类后，会在 js 的全局对象上增加该类的对象，比如：

```js
// js 中 require 一个 UIViewController
require('UIViewController')

// 会在全局下创建一个 UIViewController 对象
global: {
  UIViewController: {
    __clsName: 'UIViewController'
  }
}
```

具体的实现如下：

```js
var _require = function(clsName) {
  if (!global[clsName]) {
    global[clsName] = {
      __clsName: clsName
    }
  }
  return global[clsName]
}

// 全局创建对象的方法，直接为 require 的类创建一个它的对象
global.require = function() {
  var lastRequire
  for (var i = 0; i < arguments.length; i ++) {
    arguments[i].split(',').forEach(function(clsName) {
      lastRequire = _require(clsName.trim())
    })
  }
  return lastRequire
}
```

至此，js 段定义类结束

#### OC 端定义类

前面 js 预处理过后就会来到 OC 的 `defineClass` 方法中。这个方法也比较长，

1. 将类名、父类名、协议名取出
2. 创建类以及给类添加协议
3. 添加和重写方法

##### 取出类名协议名

```objc
static NSDictionary *defineClass(NSString *classDeclaration, JSValue *instanceMethods, JSValue *classMethods)
{
    // 扫描字符串做匹配
    NSScanner *scanner = [NSScanner scannerWithString:classDeclaration];
    NSString *className;
    NSString *superClassName;
    NSString *protocolNames;
    // 扫描到 : 了，把之前的放到 className 中
    [scanner scanUpToString:@":" intoString:&className];
    // 如果没有扫描到底，那么就说明有父类或者协议
    if (!scanner.isAtEnd) {
        scanner.scanLocation = scanner.scanLocation + 1;
        // 扫描出来父类
        [scanner scanUpToString:@"<" intoString:&superClassName];
        if (!scanner.isAtEnd) {
            scanner.scanLocation = scanner.scanLocation + 1;
            // 扫描协议名
            [scanner scanUpToString:@">" intoString:&protocolNames];
        }
    }
    // 如果不存在父类，那么父类就是 NSObject
    if (!superClassName) superClassName = @"NSObject";
    // 修改一下类名和父类名，把前后的空白字符去掉
    className = trim(className);
    superClassName = trim(superClassName);
    // 把 protocol 切开拆分成数组
    NSArray *protocols = [protocolNames length] ? [protocolNames componentsSeparatedByString:@","] : nil;
  	...
}
```

其实就是解析传进来的 `NSString *classDeclaration` 字符串。其实这个直接在 js 解析就可以了。这种 js 解析一遍，OC 又解析一遍的做法有点累赘。

##### 创建类以及给类添加协议

解析好类名和协议名之后就是创建了：

```objc
static NSDictionary *defineClass(NSString *classDeclaration, JSValue *instanceMethods, JSValue *classMethods)
{
  	...
    // 反射为类名
    Class cls = NSClassFromString(className);
    if (!cls) {
        // 如果子类没有实例化成功，那么实例化父类
        Class superCls = NSClassFromString(superClassName);
        // 如果父类也没有实例化成功
        if (!superCls) {
            // 直接报错
            _exceptionBlock([NSString stringWithFormat:@"can't find the super class %@", superClassName]);
            return @{@"cls": className};
        }
        // 存在父类，不存在子类，那么创建一个子类时
        cls = objc_allocateClassPair(superCls, className.UTF8String, 0);
        objc_registerClassPair(cls);
    }
    
    // 如果有协议，那么拿到所有的协议名，给类增加协议
    if (protocols.count > 0) {
        for (NSString* protocolName in protocols) {
            Protocol *protocol = objc_getProtocol([trim(protocolName) cStringUsingEncoding:NSUTF8StringEncoding]);
            class_addProtocol (cls, protocol);
        }
    }
  	...
}
```

##### 添加和重写方法

```objc
static NSDictionary *defineClass(NSString *classDeclaration, JSValue *instanceMethods, JSValue *classMethods)
{
  	...
    // 增加方法
    for (int i = 0; i < 2; i ++) {
        BOOL isInstance = i == 0;
        JSValue *jsMethods = isInstance ? instanceMethods: classMethods;
        // 如果是类方法那么要取出 cls 的 metaClass，否则直接拿出 cls
        Class currCls = isInstance ? cls: objc_getMetaClass(className.UTF8String);
        // 把 JSValue 转化成 字典
        NSDictionary *methodDict = [jsMethods toDictionary];
        
        for (NSString *jsMethodName in methodDict.allKeys) {
            // 通过 methodName 拿到 method 实例
            // method 实例是一个数组，第一个元素表示 method 有几个入参，第二个c元素表示方法实例
            JSValue *jsMethodArr = [jsMethods valueForProperty:jsMethodName];
            int numberOfArg = [jsMethodArr[0] toInt32];
            // 把方法中的 _ 都转为  :
            NSString *selectorName = convertJPSelectorString(jsMethodName);
            
            // 如果尾部没有：，那么添加：
            if ([selectorName componentsSeparatedByString:@":"].count - 1 < numberOfArg) {
                selectorName = [selectorName stringByAppendingString:@":"];
            }
            
            JSValue *jsMethod = jsMethodArr[1];
            if (class_respondsToSelector(currCls, NSSelectorFromString(selectorName))) {
                // TODO: 如果当前类实现了 selectorName 的方法，那么用现在的方法代替原来的方法
                overrideMethod(currCls, selectorName, jsMethod, !isInstance, NULL);
            } else {
                // 如果当前类没有实现 selectorName 方法，那么添加这个方法
                BOOL overrided = NO;
                for (NSString *protocolName in protocols) {
                    char *types = methodTypesInProtocol(protocolName, selectorName, isInstance, YES);
                    if (!types) types = methodTypesInProtocol(protocolName, selectorName, isInstance, NO);
                    if (types) {
                        overrideMethod(currCls, selectorName, jsMethod, !isInstance, types);
                        free(types);
                        overrided = YES;
                        break;
                    }
                }
                if (!overrided) {
                    if (![[jsMethodName substringToIndex:1] isEqualToString:@"_"]) {
                        NSMutableString *typeDescStr = [@"@@:" mutableCopy];
                        for (int i = 0; i < numberOfArg; i ++) {
                            [typeDescStr appendString:@"@"];
                        }
                        overrideMethod(currCls, selectorName, jsMethod, !isInstance, [typeDescStr cStringUsingEncoding:NSUTF8StringEncoding]);
                    }
                }
            }
        }
    }
  	...
}
```

首先是对方法进行预处理，js 端定义的方法都是以下划线为分割的，形如：`tableView_didSelectRowAtIndexPath`，在 OC 中要转的 `tableView:didSelectRowAtIndexPath:` 的形式。这个过程在 `convertJSSelectorString()` 方法中完成：

```objc
// 获取真正的方法名
// 总的来说就是方法中可能存在两种情况，一种是 __ 一种是 _
// _ 本地转为了 :，__ 被转为了 _
static NSString *convertJPSelectorString(NSString *selectorString)
{
    // 用 - 代替 __
    NSString *tmpJSMethodName = [selectorString stringByReplacingOccurrencesOfString:@"__" withString:@"-"];
    // 用 ： 代替 _
    NSString *selectorName = [tmpJSMethodName stringByReplacingOccurrencesOfString:@"_" withString:@":"];
    // 用 _ 代替 -
    return [selectorName stringByReplacingOccurrencesOfString:@"-" withString:@"_"];
}
```

接下来就要给类添加方法了，规则是：

- 对于已经存在的方法：直接获取方法签名，然后传入 `overrideMethod`，修改原来的方法实现的函数指针。

- 对于不存在的的方法：
  - 是某个协议中的方法：获取协议中该方法的函数签名，并传入 `overrideMethod` 实现方法。
  - 不是某个协议的方法：即 js 新增的方法，自定义方法签名为 `@@:{参数个数个 @}` 的形式，表示所有入参和返回值都是 id 类型。传入 `overrideMethod`，实现方法。

由此可见，最重要的方法是 `overrideMethod`。

##### 重写方法 `overrideMethod`

`overrideMethod` 实现了 js 方法对 oc 方法的实现和替换。它的主要做了三件事：

1. 替换目标类的消息转发方法 `forwardInvocation:` 为自定义方法  `JPForwardInvocation:`
2. 各个类的方法命名为 `_JP${方法名}` 保存到 `_JSOverideMethods` 字典中的对应类中
3. 替换原方法实现为 `msgForward`， 保存原方法实现为 `ORIG${原方法名}`

通过替换方法实现为 `mgsForward` 实现消息转发，通过替换 `forwardInvocation:` 实现在消息转发时调用自己的方法，完成 hook。

>  OC 端定义的字典 `_JSOverideMethods`保存函数指针。消息转发的时候可以找到 js 端方法

```objc
// 将方法替换为 msgForwardIMP
static void overrideMethod(Class cls, NSString *selectorName, JSValue *function, BOOL isClassMethod, const char *typeDescription)
{
    // 通过字符串获取SEL
    SEL selector = NSSelectorFromString(selectorName);
    
    // 没有类型签名的时候获取原来的方法的类型签名
    if (!typeDescription) {
        Method method = class_getInstanceMethod(cls, selector);
        typeDescription = (char *)method_getTypeEncoding(method);
    }
    
    // 获取原来方法的 IMP
    IMP originalImp = class_respondsToSelector(cls, selector) ? class_getMethodImplementation(cls, selector) : NULL;
    
    // 获取消息转发处理的系统函数实现 IMP
    IMP msgForwardIMP = _objc_msgForward;

    // 将cls中原来 forwardInvocaiton: 的实现替换成 JPForwardInvocation:函数实现.
    if (class_getMethodImplementation(cls, @selector(forwardInvocation:)) != (IMP)JPForwardInvocation) {
        IMP originalForwardImp = class_replaceMethod(cls, @selector(forwardInvocation:), (IMP)JPForwardInvocation, "v@:@");
        // 为cls添加新的SEL(ORIGforwardInvocation:)，指向原始 forwardInvocation: 的实现IMP.
        if (originalForwardImp) {
            class_addMethod(cls, @selector(ORIGforwardInvocation:), originalForwardImp, "v@:@");
        }
    }

    // 添加一个新的方法 ORIG${原方法名}
    if (class_respondsToSelector(cls, selector)) {
        NSString *originalSelectorName = [NSString stringWithFormat:@"ORIG%@", selectorName];
        SEL originalSelector = NSSelectorFromString(originalSelectorName);
        if(!class_respondsToSelector(cls, originalSelector)) {
            class_addMethod(cls, originalSelector, originalImp, typeDescription);
        }
    }
    
    NSString *JPSelectorName = [NSString stringWithFormat:@"_JP%@", selectorName];
    
    // 初始化 _JSOverideMethods 字典
    _initJPOverideMethods(cls);
    // 记录新SEL对应js传过来的待替换目标方法的实现.
    _JSOverideMethods[cls][JPSelectorName] = function;
    
    // 把原来的方法替换为 msgForward 的实现，实现方法转发
    class_replaceMethod(cls, selector, msgForwardIMP, typeDescription);
}
```

(太长了，编辑器太卡了，转至下一篇。。。)