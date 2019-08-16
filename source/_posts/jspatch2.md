title: JSPatch 源码解析(二)
date: 2019/8/8 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

这是 JSPatch 源码解析的第二部分。主要解释 JS 与 OC 的相互调用

<!--more-->

## 执行流程

### js 端调用

前面说过，所有 js 中定义的方法调用都会被预处理替换为 `__c()` 的形式。这是方法调用的起点。主要逻辑如下：

1. 判断调用者是否是 bool 类型，是的话直接返回 false
2. 如果调用的方法是重写的方法，那么绑定上下文
3. 如果调用的是父类方法，那么将 `__clsName` 设置为父类方法名，从 `__ocCls` 数组中取出父类的方法调用。
4. 如果调用的方法不是 js 定义的方法那么走到 oc 调用

第一步判断是否是 bool 类型的原因在下面的注释中有细说，主要是为了能让 js 代码在 oc 返回的对象是 nil 的情况下也能调用，不报错。

```js
/// 调用 native 相应的方法
__c: function(methodName) {
  var slf = this
  // 如果 oc 返回了一个空对象，在 js 端会以 false 的形式接受。当这个空对象再调用方法的时候，就会走到这个分支中，直接返回 false，而不会走 oc 的消息转发
  if (slf instanceof Boolean) {
    return function() {
      return false
    }
  }
  if (slf[methodName]) {
    return slf[methodName].bind(slf);
  }

  if (!slf.__obj && !slf.__clsName) {
    throw new Error(slf + '.' + methodName + ' is undefined')
  }

  /// 如果当前调用的父类的方法，那么通过 OC 方法获取该 clsName 的父类的名字
  if (slf.__isSuper && slf.__clsName) {
      slf.__clsName = _OC_superClsName(slf.__obj.__realClsName ? slf.__obj.__realClsName: slf.__clsName);
  }
  var clsName = slf.__clsName

  if (clsName && _ocCls[clsName]) {
    /// 根据 __obj 字段判断是否是实例方法或者类方法
    var methodType = slf.__obj ? 'instMethods': 'clsMethods'
    /// 如果当前方法是提前定义的方法，那么直接走定义方法的调用
    if (_ocCls[clsName][methodType][methodName]) {
      slf.__isSuper = 0;
      return _ocCls[clsName][methodType][methodName].bind(slf)
    }
  }

  /// 当前方法不是在 js 中定义的，那么直接调用 oc 的方法
  return function(){
    var args = Array.prototype.slice.call(arguments)
    return _methodFunc(slf.__obj, slf.__clsName, methodName, args, slf.__isSuper)
  }
}
```

> `__realClsName` 和 `__clsName` 一般情况下是相等的，但是如果要调用的是父类的方法，那么 `__clsName` 会被替换为父类名，而子类名仍然是 `__realClsName`

调用 oc 方法通过 `_methodFunc` 执行：

```js
/// 执行 js 方法, 返回 js 对象
// 把要调用的方法名类名，参数传递给 oc
var _methodFunc = function(instance, clsName, methodName, args, isSuper, isPerformSelector) {
  var selectorName = methodName
  // js 端的方法都是 xxx_xxx 的形式，而 oc 端的方法已经在 defineClass 的时候转为了 xxx:xxx: 的形式。所以一般情况下 js 调用 oc 方法的时候都需要先把方法名转换一下。也就是当 isPerformSelector 为 false 的情况。
  // 那么什么时候这个属性为 true 呢？当 js 端调用 performSelector 这个的方法的时候。这个方法默认需要传入 xxx:xxx: 形式的 OC selector 名。
  // 一般 performSelector 用于从 oc 端动态传来 selectorName 需要 js 执行的时候。没有太多的使用场景
  if (!isPerformSelector) {
    methodName = methodName.replace(/__/g, "-")
    selectorName = methodName.replace(/_/g, ":").replace(/-/g, "_")
    var marchArr = selectorName.match(/:/g)
    var numOfArgs = marchArr ? marchArr.length : 0
    if (args.length > numOfArgs) {
      selectorName += ":"
    }
  }
  var ret = instance ? _OC_callI(instance, selectorName, args, isSuper):
                        _OC_callC(clsName, selectorName, args)
  return _formatOCToJS(ret)
}
```

由于 OC 端的参数都在 defineClass 的时候做了处理，从 `xxx_xxx` 变为了 `xxx:xxx:`，js 端则没有做任何处理仍然是 `xxx_xxx`。因此，就需要在调用前将方法名进行修改。最终根据是否是类方法来决定调用 `_OC_callI` 还是 `_OC_callC`。

### OC 端执行

OC 端执行涉及到的几个方法是 JSPatch 最重要的几个方法了。涉及到方法的调用，方法签名的获取，方法的消息转发

#### OC 执行方法

js 端调用 `_OC_callI` 或者 `_OC_callC` 之后都会来到  OC 的 `callSelector()` 方法中。

1. 特殊情况处理
2. 处理调用父类方法
3. 创建 NSInvocation
4. 处理可变参数方法
5. 设置 NSInvocation 参数
6. 执行并返回结果给 js

##### 处理特殊情况

这一步主要包含两种特殊情况：

1. 执行方法的是实例对象还是类还是空对象
2. 执行的方法是 `toJS` 方法

调用 `formatJSToOC` 方法，把调用者和参数都转为 OC 的对象，这个方法连同后面将OC结果转JS的方法 `formatOCToJS` 会在后面单独说。

首先判断 JS 传来的调用者的类型。如果 `class_isMetaClass` 得到的是 true，就表示是一个类对象。如果是 Bool 类型，或者是 `_nilObj` 那么直接返回 `{isNil: true}` 这样，JS 端就知道是空对象了。

之后判断调用方法是否是 `toJS`。这个方法用来将 OC 对象转为形如 `{__obj: xxx, clsName: xxx}`  的 js 对象返回。

```objc
static id callSelector(NSString *className, NSString *selectorName, JSValue *arguments, JSValue *instance, BOOL isSuper)
{
    // realClsName 是真正要调用的类，和 clsName 的区别在于如果要调用的方法是父类方法，那么 clsName 会变为父类名，而 realClsName 仍然是子类名
    NSString *realClsName = [[instance valueForProperty:@"__realClsName"] toString];
    // 校验调用对象是否是类名，是否是空对象
    if (instance) {
        instance = formatJSToOC(instance);
        if (class_isMetaClass(object_getClass(instance))) {
            className = NSStringFromClass((Class)instance);
            instance = nil;
        } else if (!instance || instance == _nilObj || [instance isKindOfClass:[JPBoxing class]]) {
            return @{@"__isNil": @(YES)};
        }
    }
    // 把参数列表从 JS 转到 OC
    id argumentsObj = formatJSToOC(arguments);
    /**
     如果要执行的方法是"toJS"，即转化为js类型
     对于NSString/NSNumber/NSData 等可以直接转为 js 对象的直接转为 js 默认对象
     对于普通 oc 对象，转为 {__obj: xxx, __clsName: xxx} 的包裹形式
    **/
    if (instance && [selectorName isEqualToString:@"toJS"]) {
        if ([instance isKindOfClass:[NSString class]] || [instance isKindOfClass:[NSDictionary class]] || [instance isKindOfClass:[NSArray class]] || [instance isKindOfClass:[NSDate class]]) {
            return _unboxOCObjectToJS(instance);
        }
    }
  
  	...
}
```

`toJS` 方法的实现：

```objc
// 把 oc 对象转为 js 对象 （增加了 __obj __clsName 等字段）
static NSDictionary *_wrapObj(id obj)
{
    if (!obj || obj == _nilObj) {
        return @{@"__isNil": @(YES)};
    }
    return @{@"__obj": obj, @"__clsName": NSStringFromClass([obj isKindOfClass:[JPBoxing class]] ? [[((JPBoxing *)obj) unbox] class]: [obj class])};
}

// 如果是 NSString，NSNumber，NSDate，NSBlock 直接返回，
// 如果是 NSArray 或者 NSDictionary 把内部解包
// 否则以 {__obj: xxx, __clsName: xxx} 的形式包裹对象
static id _unboxOCObjectToJS(id obj)
{
    if ([obj isKindOfClass:[NSArray class]]) {
        NSMutableArray *newArr = [[NSMutableArray alloc] init];
        for (int i = 0; i < [(NSArray*)obj count]; i ++) {
            [newArr addObject:_unboxOCObjectToJS(obj[i])];
        }
        return newArr;
    }
    if ([obj isKindOfClass:[NSDictionary class]]) {
        NSMutableDictionary *newDict = [[NSMutableDictionary alloc] init];
        for (NSString *key in [obj allKeys]) {
            [newDict setObject:_unboxOCObjectToJS(obj[key]) forKey:key];
        }
        return newDict;
    }
    if ([obj isKindOfClass:[NSString class]] ||[obj isKindOfClass:[NSNumber class]] || [obj isKindOfClass:NSClassFromString(@"NSBlock")] || [obj isKindOfClass:[NSDate class]]) {
        return obj;
    }
    return _wrapObj(obj);
}
```

##### 处理调用父类方法

如果 js 端调用的是父类的方法，那么模拟 OC 调用父类的过程。OC 中调用父类，调用者还是当前子类，但是调用的 IMP 则是父类方法的 IMP。因此，要模拟 OC 中的方法调用，需要给 OC 添加和父类一样的 IMP 实现。以 `SUPER_XXX` 表示 SEL。

```objc
static id callSelector(NSString *className, NSString *selectorName, JSValue *arguments, JSValue *instance, BOOL isSuper)
{
    ...

    // realClsName 是真正要调用的类，和 clsName 的区别在于如果要调用的方法是父类方法，那么 clsName 会变为父类名，而 realClsName 仍然是子类名
    NSString *realClsName = [[instance valueForProperty:@"__realClsName"] toString];
    
    Class cls = instance ? [instance class] : NSClassFromString(className);
    SEL selector = NSSelectorFromString(selectorName);
    NSString *superClassName = nil;

    if (isSuper) {
        // 创建一个 SUPER_${selectorName} 的 SEL
        NSString *superSelectorName = [NSString stringWithFormat:@"SUPER_%@", selectorName];
        SEL superSelector = NSSelectorFromString(superSelectorName);
        
        // 通过 js 传来的 realClsName 拿到 superCls
        Class superCls;
        if (realClsName.length) {
            Class defineClass = NSClassFromString(realClsName);
            // 如果 realClsName 的父类能找到就用 realClsName 的父类，否则就直接用传过来的 className 转的 cls
            superCls = defineClass ? [defineClass superclass] : [cls superclass];
        } else {
            superCls = [cls superclass];
        }
        
        // 获取 superCls 对应的原始方法的函数指针
        Method superMethod = class_getInstanceMethod(superCls, selector);
        IMP superIMP = method_getImplementation(superMethod);
        
        // 给当前的 class 添加一个 superSelector, 形式为 SUPER_XXX 的形式，这个方法的指针指向父类方法
        // 之所以给子类添加一个指向父类实现的方法是因为 OC 中调用父类方法调用者也是子类，是通过转发调用到父类实现的。
        // 这里就是模拟了这个过程，直接给子类添加父类实现的方法
        class_addMethod(cls, superSelector, superIMP, method_getTypeEncoding(superMethod));
        
        // 因为是给子类动态添加的 ${SUPER_XXX} 的方法，它的实现是指向父类相应的原始方法。如果父类的方法被 JS 重写了，那么 ${SUPER_XXX} 也应该被 JS 重写。所以这里要对 ${SUPER_XXX} 的方法进行 JS 的替换
        NSString *JPSelectorName = [NSString stringWithFormat:@"_JP%@", selectorName];
        JSValue *overideFunction = _JSOverideMethods[superCls][JPSelectorName];
        if (overideFunction) {
            overrideMethod(cls, superSelectorName, overideFunction, NO, NULL);
        }
        selector = superSelector;
        superClassName = NSStringFromClass(superCls);
    }
  
  	...
}
```

##### 创建 NSInvocation

这一段比较简单，就是创建 `NSInvocation` ，并且获取方法签名。其中将缓存存放在 `_JSMethodSignatureCache` 字典中的操作我认为意义不大。

```objc
static id callSelector(NSString *className, NSString *selectorName, JSValue *arguments, JSValue *instance, BOOL isSuper)
{
  	...
      
    // 创建 NSInvocation
    
    NSInvocation *invocation;
    NSMethodSignature *methodSignature;
    // 缓存实例的方法签名,虽然我觉得用处不大
    if (!_JSMethodSignatureCache) {
        _JSMethodSignatureCache = [[NSMutableDictionary alloc]init];
    }
    if (instance) {
        [_JSMethodSignatureLock lock];
        if (!_JSMethodSignatureCache[cls]) {
            _JSMethodSignatureCache[(id<NSCopying>)cls] = [[NSMutableDictionary alloc]init];
        }
        methodSignature = _JSMethodSignatureCache[cls][selectorName];
        if (!methodSignature) {
            methodSignature = [cls instanceMethodSignatureForSelector:selector];
            _JSMethodSignatureCache[cls][selectorName] = methodSignature;
        }
        [_JSMethodSignatureLock unlock];
        if (!methodSignature) {
            _exceptionBlock([NSString stringWithFormat:@"unrecognized selector %@ for instance %@", selectorName, instance]);
            return nil;
        }
        // 创建 NSInvocation
        invocation = [NSInvocation invocationWithMethodSignature:methodSignature];
        [invocation setTarget:instance];
    } else {
        // 如果是类方法，直接获取函数签名
        methodSignature = [cls methodSignatureForSelector:selector];
        if (!methodSignature) {
            _exceptionBlock([NSString stringWithFormat:@"unrecognized selector %@ for class %@", selectorName, className]);
            return nil;
        }
        invocation= [NSInvocation invocationWithMethodSignature:methodSignature];
        [invocation setTarget:cls];
    }
    // 设置 NSInvocation 的 SEL
    [invocation setSelector:selector];
  
  	...
}
```

##### 处理可变参数方法

方法签名的参数个数小于实际传入的参数就是可变参数方法。重写可变参数方法容易，但是直接调用可变参数方法却比较难。因为，通过 `objc_msgSend` 需要确定调用固定参数的个数。

既然 `objc_msgSend` 需要明确知道固定参数的个数，那么我们就把它确定参数个数的方法都实现一遍就好了。比如：

```c
id (*new_msgSend1)(id, SEL, id,...) = (id (*)(id, SEL, id,...)) objc_msgSend;
```

上面就声明了一个 `new_msgSend1`，实现了调用固定参数一个，后续是可变参数的方法。同样的，我们可以声明两个固定参数，乃至 n 个固定参数的方法。但是这样有一个缺陷就是**只能调用所有参数都是 id 类型的方法**，因为无法提前确定参数类型。

各个固定参数个数的 `objc_msgSend` 方法定义好后，就可以根据 JS 端传来的参数个数和方法签名的实际参数个数来决定调用哪个 `objc_msgSend` 了。

```objc
static id callSelector(NSString *className, NSString *selectorName, JSValue *arguments, JSValue *instance, BOOL isSuper)
{
  	...
      
		// 处理可变参数
    
    NSUInteger numberOfArguments = methodSignature.numberOfArguments;
    NSInteger inputArguments = [(NSArray *)argumentsObj count];
    // 针对可变参数个数的方法。参数个数会大于方法签名的参数的个数就是可变参数方法
    if (inputArguments > numberOfArguments - 2) {
        // 只支持参数o都是 id 类型，并且返回类型也是 id 类型。
        id sender = instance != nil ? instance : cls;
        id result = invokeVariableParameterMethod(argumentsObj, methodSignature, sender, selector);
        return formatOCToJS(result);
    }
		...
}
```

(`invokeVariableParameterMethod()` 方法就省略了，用了大量的宏来定义和代用 `objc_msgSend`，有兴趣可以自己查看) 

##### 设置 NSInvocation 参数

设置 NSInvocation 参数也是一个非常冗长的方法。这个方法的起因是 JS 端传来的 JSValue 类型的参数在最开始 `formatJSToOC` 的时候都变为了统一类型 id 类型。设置 NSInvocation 参数的时候要恢复其原来的类型。

以下方法做了详尽的注释：

```objc
static id callSelector(NSString *className, NSString *selectorName, JSValue *arguments, JSValue *instance, BOOL isSuper)
{
  	...
      
		for (NSUInteger i = 2; i < numberOfArguments; i++) {
        const char *argumentType = [methodSignature getArgumentTypeAtIndex:i];
        id valObj = argumentsObj[i-2];
        // 根据 argumentType 表示的类型，设置 NSInvocation 的参数类型
        switch (argumentType[0] == 'r' ? argumentType[1] : argumentType[0]) {
                
                // 判断是否是当前的 type，如果是，就设置 invocation 的第 i 个元素设置为该类型，值为 valObj
                #define JP_CALL_ARG_CASE(_typeString, _type, _selector) \
                case _typeString: {                              \
                    _type value = [valObj _selector];                     \
                    [invocation setArgument:&value atIndex:i];\
                    break; \
                }
                
                JP_CALL_ARG_CASE('c', char, charValue)
                JP_CALL_ARG_CASE('C', unsigned char, unsignedCharValue)
                JP_CALL_ARG_CASE('s', short, shortValue)
                JP_CALL_ARG_CASE('S', unsigned short, unsignedShortValue)
                JP_CALL_ARG_CASE('i', int, intValue)
                JP_CALL_ARG_CASE('I', unsigned int, unsignedIntValue)
                JP_CALL_ARG_CASE('l', long, longValue)
                JP_CALL_ARG_CASE('L', unsigned long, unsignedLongValue)
                JP_CALL_ARG_CASE('q', long long, longLongValue)
                JP_CALL_ARG_CASE('Q', unsigned long long, unsignedLongLongValue)
                JP_CALL_ARG_CASE('f', float, floatValue)
                JP_CALL_ARG_CASE('d', double, doubleValue)
                JP_CALL_ARG_CASE('B', BOOL, boolValue)
                
            // selector类型
            case ':': {
                SEL value = nil;
                if (valObj != _nilObj) {
                    value = NSSelectorFromString(valObj);
                }
                [invocation setArgument:&value atIndex:i];
                break;
            }
            // 结构体类型
            case '{': {
                // 获取结构体名
                NSString *typeString = extractStructName([NSString stringWithUTF8String:argumentType]);
                // 去除 js 的参数
                JSValue *val = arguments[i-2];
                // 如果结构体是给定的类型，那么就把 js 的参数转化为该类型的结构体设置到 invocation 中去
                #define JP_CALL_ARG_STRUCT(_type, _methodName) \
                if ([typeString rangeOfString:@#_type].location != NSNotFound) {    \
                    _type value = [val _methodName];  \
                    [invocation setArgument:&value atIndex:i];  \
                    break; \
                }
                // 校验是否是 Rect，Point，Size，Range 四种结构体
                JP_CALL_ARG_STRUCT(CGRect, toRect)
                JP_CALL_ARG_STRUCT(CGPoint, toPoint)
                JP_CALL_ARG_STRUCT(CGSize, toSize)
                JP_CALL_ARG_STRUCT(NSRange, toRange)
                @synchronized (_context) {
                    // 检查是否是通过 defineStruct 定义的结构体
                    // 结构体包含的键为 types 结构体的类型，keys 结构体的u键
                    NSDictionary *structDefine = _registeredStruct[typeString];
                    if (structDefine) {
                        // 拿到结构体的大小
                        size_t size = sizeOfStructTypes(structDefine[@"types"]);
                        // 创建相应大小的内存地址
                        void *ret = malloc(size);
                        // 把 valObj 里的数据复制到 ret 上
                        getStructDataWithDict(ret, valObj, structDefine);
                        // 设置 ret 为 invocation 的参数
                        [invocation setArgument:ret atIndex:i];
                        // chuan
                        free(ret);
                        break;
                    }
                }
                
                break;
            }
            // 指针类型
            case '*':
            case '^': {
                // 如果是 JPBoxing 类型，那么解包,如果不是 JPBoxing 类型，那么继续往下走
                if ([valObj isKindOfClass:[JPBoxing class]]) {
                    void *value = [((JPBoxing *)valObj) unboxPointer];
                    
                    // ^@ 表示一个指向id类型的指针
                    if (argumentType[1] == '@') {
                        if (!_TMPMemoryPool) {
                            _TMPMemoryPool = [[NSMutableDictionary alloc] init];
                        }
                        // 把 JPBoxing 放到 markArray 数组中
                        if (!_markArray) {
                            _markArray = [[NSMutableArray alloc] init];
                        }
                        memset(value, 0, sizeof(id));
                        [_markArray addObject:valObj];
                    }
                    
                    [invocation setArgument:&value atIndex:i];
                    break;
                }
            }
            // class 类型
            case '#': {
                // 如果是 JPBoxing 类型那么解包，如果不是那么继续往下走
                if ([valObj isKindOfClass:[JPBoxing class]]) {
                    Class value = [((JPBoxing *)valObj) unboxClass];
                    [invocation setArgument:&value atIndex:i];
                    break;
                }
            }
            default: {
                // null 类型
                if (valObj == _nullObj) {
                    valObj = [NSNull null];
                    [invocation setArgument:&valObj atIndex:i];
                    break;
                }
                if (valObj == _nilObj ||
                    ([valObj isKindOfClass:[NSNumber class]] && strcmp([valObj objCType], "c") == 0 && ![valObj boolValue])) {
                    valObj = nil;
                    [invocation setArgument:&valObj atIndex:i];
                    break;
                }
                // block 类型
                if ([(JSValue *)arguments[i-2] hasProperty:@"__isBlock"]) {
                    JSValue *blkJSVal = arguments[i-2];
                    Class JPBlockClass = NSClassFromString(@"JPBlock");
                    if (JPBlockClass && ![blkJSVal[@"blockObj"] isUndefined]) {
                        __autoreleasing id cb = [JPBlockClass performSelector:@selector(blockWithBlockObj:) withObject:[blkJSVal[@"blockObj"] toObject]];
                        [invocation setArgument:&cb atIndex:i];
                        Block_release((__bridge void *)cb);
                    } else {
                        __autoreleasing id cb = genCallbackBlock(arguments[i-2]);
                        [invocation setArgument:&cb atIndex:i];
                    }
                } else {
                    [invocation setArgument:&valObj atIndex:i];
                }
            }
        }
    }
  	
  	...
}
```

注意，如果期待的入参的 typeencoding 是 `^@`，即方法需要一个二级指针作为参数的时候。会创建一个 `_TMPMemoryPool` 字典，以及一个 `_markArray` 数组，并把参数 push 到数组中。这两个对象会在执行完成是用到，下面再说。

##### 执行并返回结果给 js

执行的时候会判断是否是父类方法，如果执行的是父类方法，那么会把它预存在 `_currInvokeSuperClsName` 字典中。因为前面我们处理父类方法的时候是给子类添加 `SUPER_XXX` 的 SEL。在后面消息转发时，在 `_JSOverideMethods` 字典中保存的则是 `XXX` 的 SEL。因此，保存在 `_currInvokeSuperClsName` 字典中就是要说明这是调用的父类的方法，要把 `SUPER_` 前缀去掉，到父类的重写方法中找。

```objc
static void overrideMethod(Class cls, NSString *selectorName, JSValue *function, BOOL isClassMethod, const char *typeDescription)
{
  	...
      
		// 如果执行的是子类，那么在 _currInvokeSuperClsName 中保存
    if (superClassName) _currInvokeSuperClsName[selectorName] = superClassName;
    // 执行方法
    [invocation invoke];
    // 执行完方法后从 _currInvokeSuperClsName 移除
    if (superClassName) [_currInvokeSuperClsName removeObjectForKey:selectorName];
		
		...
}
```

在方法执行完成后，会对 `_TMPMemoryPool` 和 `_markArray` 进行处理。具体场景再补充中说，这里只是添加注释：

```objc
static void overrideMethod(Class cls, NSString *selectorName, JSValue *function, BOOL isClassMethod, const char *typeDescription)
{
  	...
  	
    if ([_markArray count] > 0) {
        for (JPBoxing *box in _markArray) {
            // pointer 是一个二级指针
            // 执行完方法后，二级指针会被指向一个指针
            void *pointer = [box unboxPointer];
            // 让 obj 指向二级指针pointer指向的指针指向的值
            id obj = *((__unsafe_unretained id *)pointer);
            if (obj) {
                @synchronized(_TMPMemoryPool) {
                    // 如果二级指针指向的地址确实存在，那么就把 obj 暂时保存起来，防止被回收了。
                    // 因为上面的 obj 是 unsafe_unretained 的
                    [_TMPMemoryPool setObject:obj forKey:[NSNumber numberWithInteger:[(NSObject*)obj hash]]];
                }
            }
        }
    }
 	   
    ...
}
```

最后对 OC 执行返回的结果 JS 化。这里存在一个内存泄漏的问题。即当动态调用的方法是  `alloc`，`new`，`copy`，`mutableCopy` 时，ARC 不会在尾部插入 `release` 语句，即多了一次引用计数，需要通过 `(__bridge_transfer id)` 将 c 对象转为 OC 对象，并自动释放一次引用计数，以此来达到引用计数的平衡：

```objc
    char returnType[255];
    strcpy(returnType, [methodSignature methodReturnType]);
    
    id returnValue;
    // 不是 void 类型
    if (strncmp(returnType, "v", 1) != 0) {
        // 是 id 类型需要判断是不是创建新对象
        if (strncmp(returnType, "@", 1) == 0) {
            void *result;
            // 拿到返回值
            [invocation getReturnValue:&result];
            
            //For performance, ignore the other methods prefix with alloc/new/copy/mutableCopy
            if ([selectorName isEqualToString:@"alloc"] || [selectorName isEqualToString:@"new"] ||
                [selectorName isEqualToString:@"copy"] || [selectorName isEqualToString:@"mutableCopy"]) {
                // 针对 alloc 等方法，需要通过 __bridge_transfer 减去引用计数
                returnValue = (__bridge_transfer id)result;
            } else {
                returnValue = (__bridge id)result;
            }
            // 将 OC 转为 JS 返回
            return formatOCToJS(returnValue);
            
        } else {
            // 其他的各种类型 返回 JSValue
            switch (returnType[0] == 'r' ? returnType[1] : returnType[0]) {
                    
                #define JP_CALL_RET_CASE(_typeString, _type) \
                case _typeString: {                              \
                    _type tempResultSet; \
                    [invocation getReturnValue:&tempResultSet];\
                    returnValue = @(tempResultSet); \
                    break; \
                }
                    
                JP_CALL_RET_CASE('c', char)
                JP_CALL_RET_CASE('C', unsigned char)
                JP_CALL_RET_CASE('s', short)
                JP_CALL_RET_CASE('S', unsigned short)
                JP_CALL_RET_CASE('i', int)
                JP_CALL_RET_CASE('I', unsigned int)
                JP_CALL_RET_CASE('l', long)
                JP_CALL_RET_CASE('L', unsigned long)
                JP_CALL_RET_CASE('q', long long)
                JP_CALL_RET_CASE('Q', unsigned long long)
                JP_CALL_RET_CASE('f', float)
                JP_CALL_RET_CASE('d', double)
                JP_CALL_RET_CASE('B', BOOL)

                case '{': {
                    NSString *typeString = extractStructName([NSString stringWithUTF8String:returnType]);
                    #define JP_CALL_RET_STRUCT(_type, _methodName) \
                    if ([typeString rangeOfString:@#_type].location != NSNotFound) {    \
                        _type result;   \
                        [invocation getReturnValue:&result];    \
                        return [JSValue _methodName:result inContext:_context];    \
                    }
                    JP_CALL_RET_STRUCT(CGRect, valueWithRect)
                    JP_CALL_RET_STRUCT(CGPoint, valueWithPoint)
                    JP_CALL_RET_STRUCT(CGSize, valueWithSize)
                    JP_CALL_RET_STRUCT(NSRange, valueWithRange)
                    @synchronized (_context) {
                        NSDictionary *structDefine = _registeredStruct[typeString];
                        if (structDefine) {
                            size_t size = sizeOfStructTypes(structDefine[@"types"]);
                            void *ret = malloc(size);
                            [invocation getReturnValue:ret];
                            NSDictionary *dict = getDictOfStruct(ret, structDefine);
                            free(ret);
                            return dict;
                        }
                    }
                    break;
                }
                case '*':
                case '^': {
                    void *result;
                    [invocation getReturnValue:&result];
                    returnValue = formatOCToJS([JPBoxing boxPointer:result]);
                    if (strncmp(returnType, "^{CG", 4) == 0) {
                        if (!_pointersToRelease) {
                            _pointersToRelease = [[NSMutableArray alloc] init];
                        }
                        [_pointersToRelease addObject:[NSValue valueWithPointer:result]];
                        CFRetain(result);
                    }
                    break;
                }
                case '#': {
                    Class result;
                    [invocation getReturnValue:&result];
                    returnValue = formatOCToJS([JPBoxing boxClass:result]);
                    break;
                }
            }
            return returnValue;
        }
    }
    return nil;
}
```

#### OC 消息转发

jspatch 想要 hook 任意的方法，就需要让所有被 hook 的方法走一个统一的入口。这个统一的入口通过让所有被替换的方法走消息转发，并且 hook 消息转发的 `forwardInvocation()` 方法实现，无论这个方法是 JS 调用的还是 OC 调用的。

主要过程如下：

1. 获取被重写的方法的 JS 实现
   1. 如果没有重写的 JS 方法，那么执行原来的方法实现
2. 准备提供给 JS 的参数
   1. 执行上下文，方法的实际调用者 self
   2. 方法的实际参数
3. 针对调用父类方法的特殊处理
4. 执行 JS 重写方法，并对将返回结果设回 NSInvocation
5. 针对 dealloc 方法的特殊处理

##### 获取被重写方法的 JS 实现

`JPForwardInvocation` 是一个静态方法，它的前两个参数分别是 `self` 和 `selector`。主要通过 `getJSFunctionInObjectHierachy()` 方法获取 JS 替换方法的实现

```objc
// 自己的替换方法, 可以看到调用方法前两个参数一个是 self，一个是 selecter， 对应于方法签名的  @:
static void JPForwardInvocation(__unsafe_unretained id assignSlf, SEL selector, NSInvocation *invocation)
{
    BOOL deallocFlag = NO;
    id slf = assignSlf;
    BOOL isBlock = [[assignSlf class] isSubclassOfClass : NSClassFromString(@"NSBlock")];
    
    NSMethodSignature *methodSignature = [invocation methodSignature];
    NSInteger numberOfArguments = [methodSignature numberOfArguments];
    NSString *selectorName = isBlock ? @"" : NSStringFromSelector(invocation.selector);
    NSString *JPSelectorName = [NSString stringWithFormat:@"_JP%@", selectorName];
    // 判断 JSPSEL 是否有对应的 js 函数的实现，如果没有就原始方法的消息转发的流程
    // （被 defineClass 过的类会被替换 forwardInvocation 方法。如果有方法没有被实现，且没有被 JS重写。那么就会走原始的 forwardInvocation 要找到的是 ORIGforwardInvocation 方法）
    JSValue *jsFunc = isBlock ? objc_getAssociatedObject(assignSlf, "_JSValue")[@"cb"] : getJSFunctionInObjectHierachy(slf, JPSelectorName);
    if (!jsFunc) {
        JPExecuteORIGForwardInvocation(slf, selector, invocation);
        return;
    }
  
  	...
}
```

`getJSFunctionInObjectHierachy()` 方法查找当前类是否有 JS 实现，如果没有，就从父类中查找。如果父类中也都没有实现，那么就要走原本的消息转发逻辑了。

```objc
// 判断这个方法是否有 js 方法的实现。 后续通过这个判断结果走原始转发流程还是走 js 方法的调用
// 这是一个递归方法
static JSValue *getJSFunctionInObjectHierachy(id slf, NSString *selectorName)
{
    Class cls = object_getClass(slf);
    // 如果是正在在 js 中调用 oc 一个类中的 super 的方法，那么就通过 _currInvokeSuperClsName 记录下来。因为在调用的过程中由于是 super 方法，selector 会有一个 SUPER_ 的前缀。消息转发到这里的时候需要知道当前调用的其实是一个 super 的方法，需要把 SUPER_ 去除。
    if (_currInvokeSuperClsName[selectorName]) {
        cls = NSClassFromString(_currInvokeSuperClsName[selectorName]);
        selectorName = [selectorName stringByReplacingOccurrencesOfString:@"_JPSUPER_" withString:@"_JP"];
    }
    JSValue *func = _JSOverideMethods[cls][selectorName];
    // 遍历父类找这个方法
    while (!func) {
        cls = class_getSuperclass(cls);
        if (!cls) {
            return nil;
        }
        func = _JSOverideMethods[cls][selectorName];
    }
    return func;
}
```

原本的 `forwardInvocation:` 指向了 SEL `ORIDforwardInvocation:`。拿到它对应的函数指针，并且执行：

```objc
// 方法的原本的 forward 流程
static void JPExecuteORIGForwardInvocation(id slf, SEL selector, NSInvocation *invocation)
{
    // 拿到原始的被替换的 forwardInvocation： ORIDforwardInvocation, 然后调用原始的转发方法
    SEL origForwardSelector = @selector(ORIGforwardInvocation:);
    
    if ([slf respondsToSelector:origForwardSelector]) {
        NSMethodSignature *methodSignature = [slf methodSignatureForSelector:origForwardSelector];
        if (!methodSignature) {
            _exceptionBlock([NSString stringWithFormat:@"unrecognized selector -ORIGforwardInvocation: for instance %@", slf]);
            return;
        }
        // 调用原始的转换方法
        NSInvocation *forwardInv= [NSInvocation invocationWithMethodSignature:methodSignature];
        [forwardInv setTarget:slf];
        [forwardInv setSelector:origForwardSelector];
        [forwardInv setArgument:&invocation atIndex:2];
        [forwardInv invoke];
    } else {
        // 如果不存在原始的转发方法，就调用父类的转发方法
        // 这里应该是保底逻辑，一般来说，不会出现调用没有 forwardInvocation 方法的情况
        Class superCls = [[slf class] superclass];
        Method superForwardMethod = class_getInstanceMethod(superCls, @selector(forwardInvocation:));
        void (*superForwardIMP)(id, SEL, NSInvocation *);
        superForwardIMP = (void (*)(id, SEL, NSInvocation *))method_getImplementation(superForwardMethod);
        superForwardIMP(slf, @selector(forwardInvocation:), invocation);
    }
}
```

##### 准备传给 JS 的参数

先要把 `self` 取出来，放到参数数组中。类对象就用一个包含 `__clsName` 的对象表示，实例对象则用 `JPBoxing` 包裹。另外，对于 `dealloc` 方法要注意不能使用 weak 修饰。在 `dealloc` 期间，不能使用 weak：

```objc
static void JPForwardInvocation(__unsafe_unretained id assignSlf, SEL selector, NSInvocation *invocation)
{
  	...
      
//  从NSInvocation中获取调用的参数，把self与相应的参数都转换成js对象并封装到一个集合中
//  js端重写的函数，传递过来是JSValue类型，用callWithArgument:调用js方法，参数也要是js对象.
    NSMutableArray *argList = [[NSMutableArray alloc] init];
    if (!isBlock) {
        if ([slf class] == slf) {
            // 如果调用的是类方法，那么给入参列表的第一个参数就是一个包含  __clsName 的 object
            [argList addObject:[JSValue valueWithObject:@{@"__clsName": NSStringFromClass([slf class])} inContext:_context]];
        } else if ([selectorName isEqualToString:@"dealloc"]) {
            // 对于被释放的对象，使用 assign 来保存 self 的指针
            // 因为在 dealloc 的时候，系统不让将 self 赋值给一个 weak 对象。（在 dealloc 的时候应该会有一些操作 weak 字典的步骤，所以不能再这个阶段再操作 weak）
            // assign 和 weak 的区别在于 assign 在指向的对象销毁的时候不会把当前指针置为 nil
            // 所以这里最终要自己确保不会在 dealloc 后调用 slf 的方法
            [argList addObject:[JPBoxing boxAssignObj:slf]];
            deallocFlag = YES;
        } else {
            // 否则用 weak 包裹
            [argList addObject:[JPBoxing boxWeakObj:slf]];
        }
    }
  
  	...
}
```

之后就是取出 `NSInvocation` 中的各个参数了。这个过程是 `callSelector` 中设置 `NSInvocation` 的逆过程。基本一致，所以不重复贴代码了。

##### 调用父类方法的特殊处理

```objc
static void JPForwardInvocation(__unsafe_unretained id assignSlf, SEL selector, NSInvocation *invocation)
{
  	...
     
    // 如果当前调用的方法是 js 引起的，并且 js 调用了一个 super 的方法。那么会在 _currInvokeSuperClsName 中保存一个调用的方法名。这个方法名被加上了前缀 SUPER_。 因此真正调用的时候要把这个前缀替换为 _JP。这样才能找到保存在 JSOverrideMethods 字典中的相应方法
    if (_currInvokeSuperClsName[selectorName]) {
        Class cls = NSClassFromString(_currInvokeSuperClsName[selectorName]);
        NSString *tmpSelectorName = [[selectorName stringByReplacingOccurrencesOfString:@"_JPSUPER_" withString:@"_JP"] stringByReplacingOccurrencesOfString:@"SUPER_" withString:@"_JP"];
        // 如果父类没有重写相应的方法
        if (!_JSOverideMethods[cls][tmpSelectorName]) {
            NSString *ORIGSelectorName = [selectorName stringByReplacingOccurrencesOfString:@"SUPER_" withString:@"ORIG"];
            [argList removeObjectAtIndex:0];
            // 如果父类没有重写这个方法那么就是调用 oc 的方法，oc 直接调用父类的相应方法
            id retObj = callSelector(_currInvokeSuperClsName[selectorName], ORIGSelectorName, [JSValue valueWithObject:argList inContext:_context], [JSValue valueWithObject:@{@"__obj": slf, @"__realClsName": @""} inContext:_context], NO);
            id __autoreleasing ret = formatJSToOC([JSValue valueWithObject:retObj inContext:_context]);
            [invocation setReturnValue:&ret];
            return;
        }
    }
  
  	...
}     
```

##### 执行 JS 方法，并将结果设置到 NSInvocation

准备好参数后，只要调用 `callWithArguments` 就可以同步获取到执行结果。由于是消息转发，因此要将得到的结果还要通过 `setReturnValue:` 设置给 NSInvocation。

```objc
static void JPForwardInvocation(__unsafe_unretained id assignSlf, SEL selector, NSInvocation *invocation)
{
  	...
    // 转化为 js 的参数形式，将对象包裹为 {__obj: obj, __clsName: xxx} 的形式
    NSArray *params = _formatOCToJSList(argList);
    char returnType[255];
    // 获取方法的返回参数的签名
    strcpy(returnType, [methodSignature methodReturnType]);
  
  	// 判断 returnType 的符号签名
    switch (returnType[0] == 'r' ? returnType[1] : returnType[0]) {
        // 先调用 js 重写的方法，得到 返回值 jsval

        // 如果 jsval 不是空，并且需要 isPerformInOC，那么获取其 callback 方法，
        // 执行完后拿到数据转为 js 后调用 callback 方法

        // 如果 jsval 没有 isPerformInOC，那么就是执行完 js 方法后直接往下走
				...
        (省略了各种类型判断)
    }
}
```

##### 针对 dealloc 方法的特殊处理

如果是一般的方法，用 JS 替换了原方法执行就完事了。但是 dealloc 方法则有点特殊，它的原始方法必须要执行，不然怎么完成资源回收。因此，在方法执行的最后会判断当前执行的是不是 dealloc 方法，如果是，那么默认调用它的实现：

```objc
static void JPForwardInvocation(__unsafe_unretained id assignSlf, SEL selector, NSInvocation *invocation)
{
  	...
      
    // 如果这个方法是 dealloc 方法
    if (deallocFlag) {
        slf = nil;
        Class instClass = object_getClass(assignSlf);
        // 拿到 dealloc 方法实现
        Method deallocMethod = class_getInstanceMethod(instClass, NSSelectorFromString(@"ORIGdealloc"));
        void (*originalDealloc)(__unsafe_unretained id, SEL) = (__typeof__(originalDealloc))method_getImplementation(deallocMethod);
        // 调用
        originalDealloc(assignSlf, NSSelectorFromString(@"dealloc"));
    }
}
```

至此，OC 端的消息转发过程结束。