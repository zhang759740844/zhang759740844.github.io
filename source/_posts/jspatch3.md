title: JSPatch 源码解析(三)
date: 2018/8/11 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

这是 JSPatch 源码解析的第三部分，主要对之前遗漏的方法做一些补充，以及提供一些思考问题

<!--more-->

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

针对几种特殊情况做处理，包括类型为 `JPBoxing` 类型，包含 `__obj`,`__clsName`,`__isBlock` 字段：

```objc
static id formatJSToOC(JSValue *jsval)
{
    id obj = [jsval toObject];
    if (!obj || [obj isKindOfClass:[NSNull class]]) return _nilObj;
    
    // 如果是 JPBoxing 类型的，那么解包
    if ([obj isKindOfClass:[JPBoxing class]]) return [obj unbox];
    // 如果是数组类型的，那么把数组里的所有都 format 一遍
    if ([obj isKindOfClass:[NSArray class]]) {
        NSMutableArray *newArr = [[NSMutableArray alloc] init];
        for (int i = 0; i < [(NSArray*)obj count]; i ++) {
            [newArr addObject:formatJSToOC(jsval[i])];
        }
        return newArr;
    }
    // 如果是字典类型的
    if ([obj isKindOfClass:[NSDictionary class]]) {
        // 如果内部有 __obj 字段
        if (obj[@"__obj"]) {
            // 拿到 __obj 对应的对象
            id ocObj = [obj objectForKey:@"__obj"];
            if ([ocObj isKindOfClass:[JPBoxing class]]) return [ocObj unbox];
            return ocObj;
        } else if (obj[@"__clsName"]) {
            // 如果存在 __clsName 对象，那么把 clsName 对应的 Class 拿出
            return NSClassFromString(obj[@"__clsName"]);
        }
        // 如果是 block
        if (obj[@"__isBlock"]) {
            Class JPBlockClass = NSClassFromString(@"JPBlock");
            if (JPBlockClass && ![jsval[@"blockObj"] isUndefined]) {
                return [JPBlockClass performSelector:@selector(blockWithBlockObj:) withObject:[jsval[@"blockObj"] toObject]];
            } else {
                return genCallbackBlock(jsval);
            }
        }
        NSMutableDictionary *newDict = [[NSMutableDictionary alloc] init];
        for (NSString *key in [obj allKeys]) {
            [newDict setObject:formatJSToOC(jsval[key]) forKey:key];
        }
        return newDict;
    }
    return obj;
}
```



#### 将 OC 对象转为 js `formatOCToJS`

将 OC 对象转为 JS 对象，为了方便 JS 调用，提前将 OC 对象的类取出，以 `{__obj: xxx, __clsName: xxx}` 的形式包裹：

```objc
// 把 oc 对象转为 js 对象 （增加了 __obj __clsName 等字段）
static NSDictionary *_wrapObj(id obj)
{
    if (!obj || obj == _nilObj) {
        return @{@"__isNil": @(YES)};
    }
    return @{@"__obj": obj, @"__clsName": NSStringFromClass([obj isKindOfClass:[JPBoxing class]] ? [[((JPBoxing *)obj) unbox] class]: [obj class])};
}

static id formatOCToJS(id obj)
{
    if ([obj isKindOfClass:[NSString class]] || [obj isKindOfClass:[NSDictionary class]] || [obj isKindOfClass:[NSArray class]] || [obj isKindOfClass:[NSDate class]]) {
        return _autoConvert ? obj: _wrapObj([JPBoxing boxObj:obj]);
    }
    if ([obj isKindOfClass:[NSNumber class]]) {
        return _convertOCNumberToString ? [(NSNumber*)obj stringValue] : obj;
    }
    if ([obj isKindOfClass:NSClassFromString(@"NSBlock")] || [obj isKindOfClass:[JSValue class]]) {
        return obj;
    }
    return _wrapObj(obj);
}
```

这里通过 `formatOCToJS` 转换后是一个包裹对象，传到 JS 端后还需要通过 `_formatOCToJS` 拿到 JS 真正想要的东西。

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

### 新建协议

在 JS 端定义协议非常简单，和创建类差不多，传入类名，类方法和实例方法：

```js
global.defineProtocol = function(declaration, instProtos , clsProtos) {
  var ret = _OC_defineProtocol(declaration, instProtos,clsProtos);
  return ret
}
```

来到 OC 中，调用的是 `defineProtocol` 方法。动态创建 protocol，向其中添加方法。

```objc
// 动态创建一个 protocol
static void defineProtocol(NSString *protocolDeclaration, JSValue *instProtocol, JSValue *clsProtocol)
{
    const char *protocolName = [protocolDeclaration UTF8String];
    // runtime 动态创建一个 protocol
    Protocol* newprotocol = objc_allocateProtocol(protocolName);
    if (newprotocol) {
        // 为创建出来的 protocol 添加方法
        addGroupMethodsToProtocol(newprotocol, instProtocol, YES);
        addGroupMethodsToProtocol(newprotocol, clsProtocol, NO);
        objc_registerProtocol(newprotocol);
    }
}
```

从 JS 中传来的 Protocol 是一个字典类型。键是方法名，值是参数返回值的类型，如下：

```json
{
  someFunctionName: {
    paramsType: "param1,param2,param3",
    returnType: "returnType",
    typeEncode: "v@:"
  }
}
```

添加方法这一步最重要的就是要获取方法的 `typeEncode`，如果注册者能够写出正确的 `typeEncode` 那么就可以直接注册了。但是由于 `typeEncode` 很多开发者不能正确写出，因此，又提供一套保底方案，通过 `paramsType` 和 `returnType` 解析出类型：

```objc
// 为 protocol 添加方法
static void addGroupMethodsToProtocol(Protocol* protocol,JSValue *groupMethods,BOOL isInstance)
{
    NSDictionary *groupDic = [groupMethods toDictionary];
    for (NSString *jpSelector in groupDic.allKeys) {
        // 拿到代表每一个方法的字典
        NSDictionary *methodDict = groupDic[jpSelector];
        // 从字典中取出参数列表
        NSString *paraString = methodDict[@"paramsType"];
        // 取出返回类型，如果没有就是 void
        NSString *returnString = methodDict[@"returnType"] && [methodDict[@"returnType"] length] > 0 ? methodDict[@"returnType"] : @"void";
        // 取出方法的typeEncode
        NSString *typeEncode = methodDict[@"typeEncode"];
        // 切割参数列表，参数列表以 ，分割的字符串的形式传递
        NSArray *argStrArr = [paraString componentsSeparatedByString:@","];
        // 获取真正的方法名
        NSString *selectorName = convertJPSelectorString(jpSelector);
        
        // 为了js端的写法好看点，末尾的参数可以不用写 `_`
        if ([selectorName componentsSeparatedByString:@":"].count - 1 < argStrArr.count) {
            selectorName = [selectorName stringByAppendingString:@":"];
        }

        // 如果有 typeEnable，那么直接添加方法
        if (typeEncode) {
            addMethodToProtocol(protocol, selectorName, typeEncode, isInstance);
            
        } else {
            if (!_protocolTypeEncodeDict) {
                _protocolTypeEncodeDict = [[NSMutableDictionary alloc] init];
                // 利用 # 宏，来将  _type 字符串化，键是 type，值是 type 的 encode
                #define JP_DEFINE_TYPE_ENCODE_CASE(_type) \
                    [_protocolTypeEncodeDict setObject:[NSString stringWithUTF8String:@encode(_type)] forKey:@#_type];\

                JP_DEFINE_TYPE_ENCODE_CASE(id);
                JP_DEFINE_TYPE_ENCODE_CASE(BOOL);
                JP_DEFINE_TYPE_ENCODE_CASE(int);
                JP_DEFINE_TYPE_ENCODE_CASE(void);
                JP_DEFINE_TYPE_ENCODE_CASE(char);
                JP_DEFINE_TYPE_ENCODE_CASE(short);
                JP_DEFINE_TYPE_ENCODE_CASE(unsigned short);
                JP_DEFINE_TYPE_ENCODE_CASE(unsigned int);
                JP_DEFINE_TYPE_ENCODE_CASE(long);
                JP_DEFINE_TYPE_ENCODE_CASE(unsigned long);
                JP_DEFINE_TYPE_ENCODE_CASE(long long);
                JP_DEFINE_TYPE_ENCODE_CASE(float);
                JP_DEFINE_TYPE_ENCODE_CASE(double);
                JP_DEFINE_TYPE_ENCODE_CASE(CGFloat);
                JP_DEFINE_TYPE_ENCODE_CASE(CGSize);
                JP_DEFINE_TYPE_ENCODE_CASE(CGRect);
                JP_DEFINE_TYPE_ENCODE_CASE(CGPoint);
                JP_DEFINE_TYPE_ENCODE_CASE(CGVector);
                JP_DEFINE_TYPE_ENCODE_CASE(NSRange);
                JP_DEFINE_TYPE_ENCODE_CASE(NSInteger);
                JP_DEFINE_TYPE_ENCODE_CASE(Class);
                JP_DEFINE_TYPE_ENCODE_CASE(SEL);
                JP_DEFINE_TYPE_ENCODE_CASE(void*);
#if TARGET_OS_IPHONE
                JP_DEFINE_TYPE_ENCODE_CASE(UIEdgeInsets);
#else
                JP_DEFINE_TYPE_ENCODE_CASE(NSEdgeInsets);
#endif

                [_protocolTypeEncodeDict setObject:@"@?" forKey:@"block"];
                [_protocolTypeEncodeDict setObject:@"^@" forKey:@"id*"];
            }
            
            // 设置返回值的 encode
            NSString *returnEncode = _protocolTypeEncodeDict[returnString];
            // 拼接上所有的 encode 成为这个方法的 encode
            if (returnEncode.length > 0) {
                NSMutableString *encode = [returnEncode mutableCopy];
                [encode appendString:@"@:"];
                for (NSInteger i = 0; i < argStrArr.count; i++) {
                    NSString *argStr = trim([argStrArr objectAtIndex:i]);
                    NSString *argEncode = _protocolTypeEncodeDict[argStr];
                    if (!argEncode) {
                        NSString *argClassName = trim([argStr stringByReplacingOccurrencesOfString:@"*" withString:@""]);
                        if (NSClassFromString(argClassName) != NULL) {
                            argEncode = @"@";
                        } else {
                            _exceptionBlock([NSString stringWithFormat:@"unreconized type %@", argStr]);
                            return;
                        }
                    }
                    [encode appendString:argEncode];
                }
                // 拼接好方法签名后给 protocol 增加方法
                addMethodToProtocol(protocol, selectorName, encode, isInstance);
            }
        }
    }
}

// 真正的添加 protocol 方法的地方
static void addMethodToProtocol(Protocol* protocol, NSString *selectorName, NSString *typeencoding, BOOL isInstance)
{
    SEL sel = NSSelectorFromString(selectorName);
    const char* type = [typeencoding UTF8String];
    protocol_addMethodDescription(protocol, sel, type, YES, isInstance);
}
```

至此，协议就添加完成了。之后就可以给类添加这个协议了。

### 执行block

block 本身就是对象，执行 block 也就是执行 block 下的函数指针指向的方法。因此，block 也是可以进行消息转发的。可以通过替换 `forwardInvocation:` 方法执行 js 部分的方法。

#### js 端定义 block

当 OC 端需要 js 提供一个回调函数的时候。不能简单的只是提供一个 function。对于 OC 来说，它们需要知道函数的 `typeEncode`。因此，JSPatch 在 JS 端提供了一个 `block` 方法给使用者，使用者需要提前给出参数的类型：

```objc
// Obj-C
@implementation JPObject
+ (void)request:(void(^)(NSString *content, BOOL success))callback
{
  callback(@"I'm content", YES);
}
@end
```

```js
// JS
require('JPObject').request(block("NSString *, BOOL", function(ctn, succ) {
  if (succ) log(ctn)  //output: I'm content
}))

```

`block` 方法的定义方式如下：

```js
global.block = function(args, cb) {
  var that = this
  var slf = global.self
  if (args instanceof Function) {
    cb = args
    args = ''
  }
  var callback = function() {
    var args = Array.prototype.slice.call(arguments)
    global.self = slf
    return cb.apply(that, _formatOCToJS(args))
  }
  var ret = {args: args, cb: callback, argCount: cb.length, __isBlock: 1}
  return ret
}
```

它最终返回的任然是一个对象。并且在对象中提供了一个标识 `__isBlock` 。这个标识在之前的源码解析中都有涉及，只不过当时都略过了。下面来看看哪些地方对这个标识做了判断，且进行了处理。

#### OC 定义 block

在将 js 对象转为 OC 对象的时候会调用 OC 的 `formatJSToOC` 方法，其中会判断传来的是否是 js 的 block。如果是，则要生成回调 block：

```objc
// js 转为 oc 的对象
static id formatJSToOC(JSValue *jsval)
{
  ...
    // 如果是 block
    if (obj[@"__isBlock"]) {
        Class JPBlockClass = NSClassFromString(@"JPBlock");
        if (JPBlockClass && ![jsval[@"blockObj"] isUndefined]) {
          	// 此处是 JPBlock 拓展
            return [JPBlockClass performSelector:@selector(blockWithBlockObj:) withObject:[jsval[@"blockObj"] toObject]];
        } else {
          	// 生成 block
            return genCallbackBlock(jsval);
        }
    }
	...
}   
```

普通的 block 走的是 `genCallbackBlock` 方法。它主要做了三件事：

1. 创建空的 block 实例
2. 根据参数类型，生成 block 函数的签名，并设置给空 block
3. 通过关联对象将函数实现保存到空的 block 中
4. 替换 `NSBlock` 的函数指针 invoke 为 `msgForwardIMP`
5. 替换 `NSBlock` 的消息转发方法 `forwardInvocation:` 和获取签名方法  `methodSignatureForSelector:`

```objc
static id genCallbackBlock(JSValue *jsVal)
{
  	// 创建空的 block 实例
    void (^block)(void) = ^(void){};
    // 拿 p 指向刚刚创建的 block
    uint8_t *p = (uint8_t *)((__bridge void *)block);
    // 根据 block 的内存分布，增加了一个 void* 和 2个 int 之后， p 指向的是 invoke 方法
    p += sizeof(void *) + sizeof(int32_t) *2;
    // 新建一个二级指针 invoke 指向 p 的当前位置
    void(**invoke)(void) = (void (**)(void))p;
    // p 在增加一个 void* 和两个 ptr 的大小，p指向的的是 signature
  	// hook 的 block 一定是个不使用外部变量的全局 block，所以没有 copy 和 dispose 函数。因此直接满足下面的公式
    p += sizeof(void *) + sizeof(uintptr_t) * 2;
    // 新建一个二级指针 signature 指向 p 的当前位置
    const char **signature = (const char **)p;
    
    static NSMutableDictionary *typeSignatureDict;
    if (!typeSignatureDict) {
        typeSignatureDict  = [NSMutableDictionary new];
        // 把各个类型的签名都存放到 typeSignatureDict 字典中
        #define JP_DEFINE_TYPE_SIGNATURE(_type) \
        [typeSignatureDict setObject:@[[NSString stringWithUTF8String:@encode(_type)], @(sizeof(_type))] forKey:@#_type];\

        JP_DEFINE_TYPE_SIGNATURE(id);
        JP_DEFINE_TYPE_SIGNATURE(BOOL);
        JP_DEFINE_TYPE_SIGNATURE(int);
        JP_DEFINE_TYPE_SIGNATURE(void);
        JP_DEFINE_TYPE_SIGNATURE(char);
        JP_DEFINE_TYPE_SIGNATURE(short);
        JP_DEFINE_TYPE_SIGNATURE(unsigned short);
        JP_DEFINE_TYPE_SIGNATURE(unsigned int);
        JP_DEFINE_TYPE_SIGNATURE(long);
        JP_DEFINE_TYPE_SIGNATURE(unsigned long);
        JP_DEFINE_TYPE_SIGNATURE(long long);
        JP_DEFINE_TYPE_SIGNATURE(unsigned long long);
        JP_DEFINE_TYPE_SIGNATURE(float);
        JP_DEFINE_TYPE_SIGNATURE(double);
        JP_DEFINE_TYPE_SIGNATURE(bool);
        JP_DEFINE_TYPE_SIGNATURE(size_t);
        JP_DEFINE_TYPE_SIGNATURE(CGFloat);
        JP_DEFINE_TYPE_SIGNATURE(CGSize);
        JP_DEFINE_TYPE_SIGNATURE(CGRect);
        JP_DEFINE_TYPE_SIGNATURE(CGPoint);
        JP_DEFINE_TYPE_SIGNATURE(CGVector);
        JP_DEFINE_TYPE_SIGNATURE(NSRange);
        JP_DEFINE_TYPE_SIGNATURE(NSInteger);
        JP_DEFINE_TYPE_SIGNATURE(Class);
        JP_DEFINE_TYPE_SIGNATURE(SEL);
        JP_DEFINE_TYPE_SIGNATURE(void*);
        JP_DEFINE_TYPE_SIGNATURE(void *);
    }
    
    // 拿到创建 block 时传过来的参数数组
    NSString *types = [jsVal[@"args"] toString];
    // 传过来的参数数组以 ，分割
    NSArray *lt = [types componentsSeparatedByString:@","];
    // block 的函数签名不是 @: 而是 @?
    NSString *funcSignature = @"@?0";
    
    NSInteger size = sizeof(void *);
    // for 循环，把 args 中的类型对应的签名都拿到，然后设置到 funcSignature 中。
    for (NSInteger i = 1; i < lt.count;) {
        NSString *t = trim(lt[i]);
        NSString *tpe = typeSignatureDict[typeSignatureDict[t] ? t : @"id"][0];
        if (i == 0) {
            funcSignature  =[[NSString stringWithFormat:@"%@%@",tpe, [@(size) stringValue]] stringByAppendingString:funcSignature];
            break;
        }
        
        funcSignature = [funcSignature stringByAppendingString:[NSString stringWithFormat:@"%@%@", tpe, [@(size) stringValue]]];
        size += [typeSignatureDict[typeSignatureDict[t] ? t : @"id"][1] integerValue];
        
        i = (i != lt.count - 1) ? i + 1 : 0;
    }
    
    IMP msgForwardIMP = _objc_msgForward;
  	// 将当前 block 的方法实现指针指向 _objc_msgForward
    *invoke = (void *)msgForwardIMP;
    
    const char *fs = [funcSignature UTF8String];
    char *s = malloc(strlen(fs));
    strcpy(s, fs);
  	// 将 block 的签名指向新生成的签名
    *signature = s;
    
  	// 将 js 端传来的函数实现以 _JSValue 的关联属性的方式保存在 block 中
    objc_setAssociatedObject(block, "_JSValue", jsVal, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 获取 NSBlock 的 class
        Class cls = NSClassFromString(@"NSBlock");
        // 要 hook block 就要替换 NSBlock 的 forwardInvocation 和 methodSignatureForSelector 方法
#define JP_HOOK_METHOD(selector, func) {Method method = class_getInstanceMethod([NSObject class], selector); \
BOOL success = class_addMethod(cls, selector, (IMP)func, method_getTypeEncoding(method)); \
if (!success) { class_replaceMethod(cls, selector, (IMP)func, method_getTypeEncoding(method));}}
        
        JP_HOOK_METHOD(@selector(methodSignatureForSelector:), block_methodSignatureForSelector);
        JP_HOOK_METHOD(@selector(forwardInvocation:), JPForwardInvocation);
    });
    
    return block;
}
```

JSPatch 中的 block 的处理应该是参考了 Aspects 中对于 block 的处理方式。将 block 的 IMP 指向 ``_objc_msgForward`，并且替换了 NSBlock 的消息转发方法，所有相关 block 的执行都会走到 `JPForwardInvocation` 方法中：

```objc
static void JPForwardInvocation(__unsafe_unretained id assignSlf, SEL selector, NSInvocation *invocation)
{
	...
    
		JSValue *jsFunc = isBlock ? objc_getAssociatedObject(assignSlf, "_JSValue")[@"cb"] : getJSFunctionInObjectHierachy(slf, JPSelectorName);
  
  ...
}
```

判断如果是一个 block，就从关联属性中拿到 callback，然后执行。

## 总结

再来总结一下 JSPatch 的执行流程。

### 定义方法

1.  创建一个 JSContext 并把各种方法通过 JSContext 提供给 js
2.  将 js 重写的方法方法通过正则表达式改写为 __c() 的形式
3.  预处理提供给 OC 调用的各个方法，hook每一个方法，将方法的第一个参数取出，并赋给 `global.self`。这就是 OC 调用提供的上下文。OC 将会把这些方法保存在 `_JSOverideMethods` 字典中
4.  预处理提供给 OC 调用的各个方法，hook每一个方法，将 this 赋给 global.self 。这就是 JS 调用时的上下文。JS 会把这些方法保存在 _ocCls 中
5.  OC 端解析 js 传来的类和方法，将方法实现替换为 `_objc_msgForward`，替换定义类的消息转发方法为 `JPForwardInvocation:`

### 执行方法

分为两种情况：

1. js 调用 js 重写的方法。这种情况下，在 `_ocCls` 字典中找到相应的方法直接执行
2. oc 调用 js 重写的方法。分为两步：
   1. oc 调用方法，主要是创建 `NSInvocation`，并根据 SEL 的函数函数签名将 js 传来的参数根据类型设置到 `NSInvocation` 上。然后执行
   2. 来到 `JPForwardInvocation` 实际执行方法，在 `_JSOverideMethods` 中找到对应的 js 实现，如果没有走原始的消息转发。将结果设置到 `NSInvocation` 的 returnValue 中

## 问题

### JSPatch 是如何做到 hook 任意方法的

hook 一个方法在 iOS 中有两种方式，一种是 method swizzling，一种是将原方法的实现 IMP 指向 `_objc_msgforward`，然后在 `forwardInvocation` 中处理

JSPatch 就是使用的后者

### 如何实现 js 端链式方法调用？

js 端的方法在执行 `a.xxx(param)`的时候都会变为 `a.__c(xxx)(param)` 的形式，只需全局对象添加 `__c` 方法即可。执行的方法教给 OC 动态的去解析拿到。

### 如何给实例对象添加属性

所有 js 添加的属性都会统一保存在一个属性对象中，然后在 js 端把这个属性对象挂载在要添加的实例对象下。之后属性对象传给 OC，OC 以关联对象的形式保存。关联对象保证了 js  创建的属性的生命周期和实例对象一致。

之后对于属性的读写都直接通过 js 端保存的属性对象，不需要再走 oc 的 bridge。

### js 中的 self 是如何实现的

self 被定义在 global 下，因此可以直接使用 self。在 oc 调用该方法的时候，会在第一个参数传递调用者。js 端拿到第一个参数把它赋给 self 即可在后面以 self 调用方法或获取属性。

### 为什么 OC 端要保存一份定义的方法在 `_JSOverideMethods` 中，js 端也要要存一份方法在 `_ocCls` 中？

因为前者是给 OC 调用的，后者是给 js 调用的。两者调用上下文的获取方式不同。前者通过方法的第一个参数传递，后者直接可以通过 this 拿到。

### js 执行时如果遇到 OC 的 null 怎么办？

如果 OC 返回的是 null，再调用 js 方法会直接出现异常。所以，对于 OC 返回的 null，需要做一层包装。可以返回 false，表示回传的是 null。

### 怎样解决 NSNumber NSMutableArray 之类在转为 JS 后，不能调用其自有方法？

### 如何处理调用父类方法的？

调用 `self.super()` 的时候会给 js 对象加上一个 `__isSuper` 标记。OC 在发现 `__isSuper` 标记后会拿到当前类的父类的相应方法的 IMP，并将其以 `SUPER_xxx` 的 SEL 添加到子类上，由子类调用。

### 如何处理可变参数？

当传参个数大于函数签名所需参数个数的时候就是可变参数方法。OC 中通过 `va_list` 获取参数列表。JSPatch 中事先定义了不同参数个数的 `objc_msgSend` 方法，通过查看传参个数来决定调用哪个 `objc_msgSend` 方法。

由于不同参数个数的 `objc_msgSend` 方法需要预先定义，不能预知调用的方法的实际参数类型。因此预先定义的 `objc_msgSend` 的所有参数都是 id 类型的。这就要求重写的可变参数的方法的函数签名中的已知参数也都是 id 类型的。

### NSInvocation 的 `getArgument` 引发的内存引用计数错误如何解决？

使用 `__bridge` 或者 `__unsafe_unretained` 修饰符告诉编译器不要插入 `retain` 或 `release`。不要改变对象原来的生命周期：

```objc
// __bridge
id returnValue;
void *result;
[invocation getReturnValue:&result];
returnValue = (__bridge id)result;

// __unsafe_unretained
__unsafe_unretained id arg;
[invocation getReturnValue:&arg];
```

### NSInvocation 创建对象后 `getReturnValue` 引发的内存泄漏怎么解决？

使用 `__bridge_transfer` 释放一次引用计数：

```objc
id returnValue;
void *result;
[invocation getReturnValue:&result];
if ([selectorName isEqualToString:@"alloc"] || [selectorName isEqualToString:@"new"]) {
    returnValue = (__bridge_transfer id)result;
} else {
    returnValue = (__bridge id)result;
}
```

### 如何新增方法

新增方法需要知道方法的函数签名。在 `defineClass` 中增加的方法由于方法签名都是从 OC 获取的，因此只能增加所有参数为 id 类型的方法

JSPatch 中新增了另一个方法 `defineProtocol`，对于要新增的方法可以先通过该方法定义一个协议，并将协议添加到相应类上。增加协议方法的时候是需要提供参数类型的。

### 如何重写 dealloc

调用 dealloc 的方法的 SEL 必须是 `dealloc`，这是 OC 内部的校验。否则 ARC 不认为这是 dealloc 方法。

因此，要实现正确调用 dealloc 方法，就必须要必须拿到替换的 IMP，并且调用的时候传入原始的 SEL：

```objc
if (deallocFlag) {
    slf = nil;
    Class instClass = object_getClass(assignSlf);
    // 拿到 dealloc 方法实现
    Method deallocMethod = class_getInstanceMethod(instClass, NSSelectorFromString(@"ORIGdealloc"));
    void (*originalDealloc)(__unsafe_unretained id, SEL) = (__typeof__(originalDealloc))method_getImplementation(deallocMethod);
    // 调用
    originalDealloc(assignSlf, NSSelectorFromString(@"dealloc"));
}

```

### 关于 block 的处理？

block 的调用类似 aspects 中的处理：

1. 创建一个空的 block
2. 修改其函数指针指向 `_objc_msgForward`
3. 根据 js 传来的类型修改 block 的函数签名
4. 通过关联对象保存其函数实现
5. 修改 NSBlock 的消息转发方法为 `JPForwardInvocation`

执行的时候在 `JPForwardInvocation` 中判断是否是一个 block，是的话就从关联对象中取出函数实现执行即可。

### 如何调用 C 函数

C 函数的调用通过 JSContext 注入



## 参考链接

[iOS 热更新解读（二）—— JSPatch 源码解析](<https://zltunes.github.io/2019/04/16/jspatch-sourcecode/>)

[JSPatch 实现原理详解系列](<https://mp.weixin.qq.com/s?__biz=MzIzNTQ2MDg2Ng==&mid=2247483662&idx=1&sn=c7d9ee27eff35688180bdc840d31120b&scene=4#wechat_redirect>)

[探究Block之MethodSignature](<https://www.jianshu.com/p/1849068b7833>)(文章中关于 block 方法签名处有结论性的错误，堆 block 中含有 copy 和 dispose 方法，而全局 block 没有这两个方法，它的验证试验的结论也是错的。)