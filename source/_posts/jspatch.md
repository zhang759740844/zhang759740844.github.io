title: JSPatch 源码解析
date: 2018/7/31 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

JSPatch 虽然被禁，但是它的源码是非常值得学习的。可以说，是我看过的各个库中设计的最巧妙也是知识点最多的开源库。非常有学习价值。

<!--more-->

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

#### js 端定义

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
```

主要过程在于 `_formatDefineMethods` 中：

```js

```

## 补充

### js 方法补充

#### `_formatOCToJS`

这个方法用于将 js 端接收到的 OC 对象转换为 js 对象：

```js
/// 把 oc 转化为 js 对象
var _formatOCToJS = function(obj) {
  // 如果 oc 端返回的直接是 undefined 或者 null，那么直接返回 false
  if (obj === undefined || obj === null) return false
  if (typeof obj == "object") {
    // js 传给 oc 时会把自己包裹在 __obj 中。因此，存在 __obj 就可以直接拿到 js 对象
    if (obj.__obj) return obj
    // 如果是空，那么直接返回 false。因为如果返回 null 的话，就无法调用方法了。
    if (obj.__isNil) return false
  if (obj instanceof Array) {
    // 如果是数组，要对每一个 oc 转 js 一下
    var ret = []
    obj.forEach(function(o) {
      ret.push(_formatOCToJS(o))
    })
    return ret
  }
  if (obj instanceof Function) {
    return function() {
      var args = Array.prototype.slice.call(arguments)
      // 如果 oc 传给 js 的是一个函数，那么 js 端调用的时候就需要先把 js 参数转为 oc 对象，调用。
      var formatedArgs = _OC_formatJSToOC(args)
      for (var i = 0; i < args.length; i++) {
        if (args[i] === null || args[i] === undefined || args[i] === false) {
          formatedArgs.splice(i, 1, undefined)
        } else if (args[i] == nsnull) {
          formatedArgs.splice(i, 1, null)
        }
      }
      // 在调用完 oc 方法后，又要 oc 对象转为 js 对象回传给 oc
      return _OC_formatOCToJS(obj.apply(obj, formatedArgs))
    }
  }
  if (obj instanceof Object) {
    // 如果是一个 object 并且没有 __obj，那么把所有的 key 都 format 一遍
    var ret = {}
    for (var key in obj) {
      ret[key] = _formatOCToJS(obj[key])
    }
    return ret
  }
  return obj
}
```

具体的过程做了详尽的注释。这里还要提一点，关于空对象的判断。由于 OC 的动态性，OC 中的 nil 发送消息会不响应任何方法。而 js 端的 undefined 或者 null 调用方法时则会直接报错。

因此，为了能让 js 端在接收到 oc 端的 nil 对象时也能调用，而不报错，针对 undefined， null 以及某些 OC 方法在返回空对象时手动返回的 `{__isNil: true}`， 这个方法都会将其解析为 bollean 值 false。

js 端空对象的调用在不报错后，还要阻止其针对 bollean 值 false 的消息转发。因此，之后在执行 `__c()` 调用 oc 方法时就会提前判断是否是 bollean 类型。如果是就说明是空对象，直接返回 false，而不进行消息转发。

