title: JSPatch 源码解析(二)
date: 2019/8/8 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

这是 JSPatch 源码解析的第二部分。

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

如果 js 端调用的是父类的方法，那么模拟 OC 调用父类的过程。OC 中调用父类，调用者还是当前子类，但是调用的 IMP 则是父类方法的 IMP。因此，要模拟 OC 中的方法调用，需要给 OC 添加和父类一样的 IMP 实现。以 `SUPER_XXX` 的表示 SEL。

这里最后还加了一层保险。如果父类方法也被 JS 重写了，一般情况下父类的该方法的 IMP 是指向 `_objc_msgForward` 的。现在又做了一次 `overrideMethod` 的尝试，确保个子类的 `SUPER_XXX` 与 ``_objc_msgForward`  绑定：

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

最后对 OC 执行返回的结果 JS 化。这里存在一个内存泄漏的问题。即当动态调用的方法是 `alloc`，`new`，`copy`，`mutableCopy` 时，ARC 不会在尾部插入 `release` 语句，即多了一次引用计数，需要通过 `(__bridge_transfer id)` 将 c 对象转为 OC 对象，并自动释放一次引用计数，以此来达到引用计数的平衡：

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





## 补充

### js 方法补充

#### 将 OC 对象转为 js `_formatOCToJS`

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

#### 定义 js 中的类 `defineJSClass`

`defineClass` 会在 oc 生成对应的类。而一些不需要继承 OC，和 OC 没有联系，只会在 JS 中使用的类，比如数据层的 dataSource/manager，直接使用 JS
原生类就可以。因此添加了一个 `defineJSClass` 方法，可以减少转化为 OC 类时的性能损耗。

```js
global.defineJSClass = function(declaration, instMethods, clsMethods) {
  var o = function() {},
      a = declaration.split(':'),
      clsName = a[0].trim(),
      superClsName = a[1] ? a[1].trim() : null
  o.prototype = {
    init: function() {
      if (this.super()) this.super().init()
      return this;
    },
    super: function() {
      return superClsName ? _jsCls[superClsName].prototype : null
    }
  }
  var cls = {
    alloc: function() {
      return new o;
    }
  }
  for (var methodName in instMethods) {
    o.prototype[methodName] = instMethods[methodName];
  }
  for (var methodName in clsMethods) {
    cls[methodName] = clsMethods[methodName];
  }
  global[clsName] = cls
  _jsCls[clsName] = o
}
```

代码实现很简单，还是在全局域下创建一个相应类名的对象。然后把所有类方法和实例方法添加至对象中。由于是通过原型实现的，不了解 js 原型的同学可能稍难理解。

这个 `defineJSClass` 方法和 `defineClass` 创建的类对象基本一致。但是有两点不同：

1. 不用 self 代替 this。
2. 没有 property

由于是纯 js 实现，因此不需要用 self 代替 this，也不需要通过关联属性定义 property，直接把属性放在 js 端就可以了。

### OC 方法补充

#### 将 js 对象转为 OC 对象 `formatJSToOC`



#### 将 OC 对象转为 js `formatOCToJS`



#### 获取 protocol 中的某个方法的签名 `methodTypesInProtocol`

可以说是一个方法模板了，没有特别的技巧：

```objc
// 获取 protocol 中相应方法的函数签名
static char *methodTypesInProtocol(NSString *protocolName, NSString *selectorName, BOOL isInstanceMethod, BOOL isRequired)
{
    // 获取 protocol
    Protocol *protocol = objc_getProtocol([trim(protocolName) cStringUsingEncoding:NSUTF8StringEncoding]);
    unsigned int selCount = 0;
    // 复制 protocol 的方法列表
    struct objc_method_description *methods = protocol_copyMethodDescriptionList(protocol, isRequired, isInstanceMethod, &selCount);
    // 遍历 protocol 的方法列表，找到和目标方法同名的方法，然后通过 c 方法复制出来返回。否则返回 NULL
    for (int i = 0; i < selCount; i ++) {
        if ([selectorName isEqualToString:NSStringFromSelector(methods[i].name)]) {
            char *types = malloc(strlen(methods[i].types) + 1);
            strcpy(types, methods[i].types);
            free(methods);
            return types;
        }
    }
    free(methods);
    return NULL;
}
```

## 问题

### 整体调用流程是什么？

### 如何处理调用父类方法的？

### OC 与 js 对象之间的传递方式是怎样的？

### 如何处理可变参数？

### JS 端对于空对象调用方法如何处理的？

### `__bridge_transfer`,`__bridge`,`__bridge_retain` 的区别？

### NSInvocation 的 `getArgument` 引发的 double release 如何处理？

### JPBoxing 的作用？

