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

### 改写修复 js 文件

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

