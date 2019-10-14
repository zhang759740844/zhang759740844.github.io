title: Aspects 源码解析
date: 2019/8/19 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---

Aspects 是一个提供健壮 AOP 能力的库。它的实现原理和 JSPatch 类似。也真是如此，它和 JSPatch 混用时会产生冲突。

<!--more-->

## 使用

Aspects 提供了两个方法来实现三种类型的 hook：

```objc
@interface NSObject (Aspects)

+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;

- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;

@end
```

它能 hook 的类型有：

1. 所有实例对象的特定方法
2. 特定类方法
3. 某个特定实例对象的特定方法

两个方法中，前者处理的是类型 1，2；后者处理的是类型 3。

> 非常有意思，如果你要 hook 1，比如你有一个类 `Test`,你要 hook `Test` 所有实例对象的  `-(void)test` 方法，你要通过 `Test` 去调用。
>
> 如果你要 hook 2，比如你要 hook `Test` 的类方法 `+(void)test` ，你要通过 `object_getClass(Test)` 去调用。

## 涉及的类

Aspects 在实现的过程中，涉及到一些类的使用，提前了解它们有助于我们之后的流程分析。

### AspectsInfo

```objc
@protocol AspectInfo <NSObject>
/// 返回 hook 的对象
- (id)instance;
/// 保存 hook 的原方法的 Invocation
- (NSInvocation *)originalInvocation;
/// 调用方法的所有参数
- (NSArray *)arguments;
@end
```

这是一个协议，当我们要执行 hook 的方法的时候，需要从实现这个协议的对象中拿到执行必要的信息

### AspectIdentifier

```objc
@interface AspectIdentifier : NSObject
/// 创建 AspectIdentifier 实例
+ (instancetype)identifierWithSelector:(SEL)selector object:(id)object options:(AspectOptions)options block:(id)block error:(NSError **)error;
/// 执行 hook 的方法
- (BOOL)invokeWithInfo:(id<AspectInfo>)info;
/// hook 的 SEL
@property (nonatomic, assign) SEL selector;
/// 替换原方法的 block
@property (nonatomic, strong) id block;
/// block 的签名
@property (nonatomic, strong) NSMethodSignature *blockSignature;
/// hook 的对象
@property (nonatomic, weak) id object;
/// hook 的时机
@property (nonatomic, assign) AspectOptions options;
@end

```

`AspectIdentifer` 包含了 hook 需要的绝大多数信息，配合 `AspectsInfo` 就可以实现 hook。

### AspectsContainer

```objc
@interface AspectsContainer : NSObject
/// 保存 hook 的信息
- (void)addAspect:(AspectIdentifier *)aspect withOptions:(AspectOptions)injectPosition;
/// 移除 hook
- (BOOL)removeAspect:(id)aspect;
/// 判断是否存在 hook
- (BOOL)hasAspects;
/// 保存在方法执行前要执行的 hook
@property (atomic, copy) NSArray *beforeAspects;
/// 保存要替换原方法执行的 hook
@property (atomic, copy) NSArray *insteadAspects;
/// 保存在方法执行后要执行的 hook
@property (atomic, copy) NSArray *afterAspects;
@end

```

每一个被 hook 的方法会对应一个 `AspectsContaienr`。`AspectsContainer` 保存了不同时机要执行的 hook 信息 `AspectsIdentifier` 实例。

之后执行的方法判断要执行 hook 的时候，就会拿到 SEL 对应的 `AspectContainer` 实例，从中拿出一个个 `AspectsIdentifier`，再结合通过方法执行时拿到的 NSInvocation 实例创建的 `AspectsInfo` 实例，达到最终 hook 方法的目的。

### AspectTracker

```objc
@interface AspectTracker : NSObject
/// 初始化方法
- (id)initWithTrackedClass:(Class)trackedClass;
/// 对应的 hook 的类
@property (nonatomic, strong) Class trackedClass;
/// 对应的 hook 的类名
@property (nonatomic, readonly) NSString *trackedClassName;
/// 类中被 hook 的方法名
@property (nonatomic, strong) NSMutableSet *selectorNames;
/// 保存某个方法与 hook 该方法的子类的 AspectTracker
@property (nonatomic, strong) NSMutableDictionary *selectorNamesToSubclassTrackers;
/// 把 selectorName 和子类的 sAspectTracker 保存在 selectorNamesToSubclassTrackers 字典中
- (void)addSubclassTracker:(AspectTracker *)subclassTracker hookingSelectorName:(NSString *)selectorName;
/// 把 selectorName 和子类的 sAspectTracker 从 selectorNamesToSubclassTrackers 字典中移除
- (void)removeSubclassTracker:(AspectTracker *)subclassTracker hookingSelectorName:(NSString *)selectorName;
/// 判断该类的子类是否 hook 了 selectorName
- (BOOL)subclassHasHookedSelectorName:(NSString *)selectorName;
/// 拿到 hook 该 selectorName 的子类的 AspectTracker 的集合
- (NSSet *)subclassTrackersHookingSelectorName:(NSString *)selectorName;
@end
```

`AspectTracker` 的目的是要保存继承链上对于某个方法的 hook 的情况。这是为了在 hook 的时候校验并保证**同一个继承链上只能有一个类被 hook**。至于这样的目的，后面再细说。

> 其实这个原因我疑惑了很久，看了很多文章，但是几乎所有对 Aspects 的源码解析都没有对 `AspectTracker` 的意义进行解释。只是照本宣科的说明继承链上只能有一个类被 hook，而没有说明为什么要设计成这样。

## 流程

### 进行 hook

hook 过程涉及的两个方法，一个类方法，一个实例方法都是直接调用 `aspect_add()` 方法：

```objc
/// 添加 hook
static id aspect_add(id self, SEL selector, AspectOptions options, id block, NSError **error) {
    __block AspectIdentifier *identifier = nil;
    /// 使用自旋锁包裹hook过程
    aspect_performLocked(^{
        // 是否可以 hook 方法
        if (aspect_isSelectorAllowedAndTrack(self, selector, options, error)) {
            /// 每个被 hook 的对象会有一个 AspectContainer 实例。取出当前对象的关联对象 AspectsContainer，如果没有就创建一个返回
            AspectsContainer *aspectContainer = aspect_getContainerForObject(self, selector);
            /// 创建一个 AspectIdentifier
            identifier = [AspectIdentifier identifierWithSelector:selector object:self options:options block:block error:error];
            if (identifier) {
                /// 把 aspectIdentifier 添加到 AspectsContainer 中
                [aspectContainer addAspect:identifier withOptions:options];

                /// hook 类和方法
                aspect_prepareClassAndHookSelector(self, selector, error);
            }
        }
    });
    return identifier;
}
```

整个过程分为几步：

1. 判断是否可以 hook
2. 创建 `AspectsContainer` 实例
3. 创建 `AspectsIdentifier` 实例
4. 实现 hook 方法

#### 判断是否可以 hook

hook 前要判断是否可以 hook。hook 的原则有几点：

1. 已经被 hook 的方法不会被重复 hook
2. 不能 hook 包含 retain，release，autorelease，forwardInvocation 的方法
3. hook 方法只能在 dealloc 方法前执行，不能替换 dealloc 方法
4. 不能 hook 不存在的 selector
5. 不能 hook 继承链上已经被 hook 过的同名方法

```objc
/// 检验 aspect 是否可以 hook 该方法。主要规则对于类方法，一个继承链上的同名方法只能被 hook 一次
static BOOL aspect_isSelectorAllowedAndTrack(NSObject *self, SEL selector, AspectOptions options, NSError **error) {
    static NSSet *disallowedSelectorList;
    static dispatch_once_t pred;
    dispatch_once(&pred, ^{
        disallowedSelectorList = [NSSet setWithObjects:@"retain", @"release", @"autorelease", @"forwardInvocation:", nil];
    });

    // Check against the blacklist.
    NSString *selectorName = NSStringFromSelector(selector);
    /// 1.对于包含 retain，release，autorelease，forwardInvocation 的方法是不能被 hook 的
    if ([disallowedSelectorList containsObject:selectorName]) {
        NSString *errorDescription = [NSString stringWithFormat:@"Selector %@ is blacklisted.", selectorName];
        AspectError(AspectErrorSelectorBlacklisted, errorDescription);
        return NO;
    }

    // Additional checks.
    AspectOptions position = options&AspectPositionFilter;
    /// 2.校验 hook dealloc 方法只能在之前执行
    if ([selectorName isEqualToString:@"dealloc"] && position != AspectPositionBefore) {
        NSString *errorDesc = @"AspectPositionBefore is the only valid position when hooking dealloc.";
        AspectError(AspectErrorSelectorDeallocPosition, errorDesc);
        return NO;
    }

    /// 3.如果不存在这个 selector 那么直接报错
    if (![self respondsToSelector:selector] && ![self.class instancesRespondToSelector:selector]) {
        NSString *errorDesc = [NSString stringWithFormat:@"Unable to find selector -[%@ %@].", NSStringFromClass(self.class), selectorName];
        AspectError(AspectErrorDoesNotRespondToSelector, errorDesc);
        return NO;
    }

    // Search for the current class and the class hierarchy IF we are modifying a class object
    /// 如果是类类型
    if (class_isMetaClass(object_getClass(self))) {
        Class klass = [self class];
        NSMutableDictionary *swizzledClassesDict = aspect_getSwizzledClassesDict();
        Class currentClass = [self class];

        AspectTracker *tracker = swizzledClassesDict[currentClass];
        /// 4.子类是否已经 hook 了这个 selector。一个方法如果被子类 hook 过了，父类就不能 hook 了。
        if ([tracker subclassHasHookedSelectorName:selectorName]) {
            NSSet *subclassTracker = [tracker subclassTrackersHookingSelectorName:selectorName];
            NSSet *subclassNames = [subclassTracker valueForKey:@"trackedClassName"];
            NSString *errorDescription = [NSString stringWithFormat:@"Error: %@ already hooked subclasses: %@. A method can only be hooked once per class hierarchy.", selectorName, subclassNames];
            AspectError(AspectErrorSelectorAlreadyHookedInClassHierarchy, errorDescription);
            return NO;
        }
        /// 5.到父类中找，如果一个方法已经在父类中 hook 过了，那么就不能在子类中 hook 了
        do {
            tracker = swizzledClassesDict[currentClass];
            if ([tracker.selectorNames containsObject:selectorName]) {
                if (klass == currentClass) {
                    // Already modified and topmost!
                    return YES;
                }
                NSString *errorDescription = [NSString stringWithFormat:@"Error: %@ already hooked in %@. A method can only be hooked once per class hierarchy.", selectorName, NSStringFromClass(currentClass)];
                AspectError(AspectErrorSelectorAlreadyHookedInClassHierarchy, errorDescription);
                return NO;
            }
        } while ((currentClass = class_getSuperclass(currentClass)));

        // Add the selector as being modified.
        currentClass = klass;
        AspectTracker *subclassTracker = nil;
        /// 如果能 hook，那么把当前的 AspectsTracker 实例加入到父类的 AspectTracker 的 selectorNamesToSubclassTrackers 字典 selectorName 对应的集合中。
        /// 这样在之后 hook 的时候就可以查看继承链上是否有被 hook 过了
        do {
            /// 如果 swizzledClassesDict 中不存在这个  tracker，那么创建这个 AspectTracker，然后添加进去
            tracker = swizzledClassesDict[currentClass];
            /// 递归为每一个当前类的父类创建一个 AspectTracker。
            if (!tracker) {
                tracker = [[AspectTracker alloc] initWithTrackedClass:currentClass];
                swizzledClassesDict[(id<NSCopying>)currentClass] = tracker;
            }
            // 递归把当前类的 AspectTracker 添加到父类的 selectorNamesToSubclassTrackers 中
            if (subclassTracker) {
                [tracker addSubclassTracker:subclassTracker hookingSelectorName:selectorName];
            } else {
                [tracker.selectorNames addObject:selectorName];
            }

            // All superclasses get marked as having a subclass that is modified.
            subclassTracker = tracker;
            /// 递归获取当前类的父类
        }while ((currentClass = class_getSuperclass(currentClass)));
	} else {
    // 实例对象默认是 YES
    // 实例对象 hook 的方法只针对该实例，因此没有同一个继承链上只能 hook 一次同名方法的限制
		return YES;
	}

    return YES;
}
```

前四个规则都容易理解。那么为什么要有**同一个继承链上只能 hook 一次同名方法**？我们来看这样一个例子：

```objc
@interface A : NSObject
- (void)foo;
@end

@implementation A
- (void)foo {
    NSLog(@"%s", __PRETTY_FUNCTION__);
}
@end

@interface B : A @end

@implementation B
- (void)foo {
    NSLog(@"%s", __PRETTY_FUNCTION__);
    [super foo];
}
@end

int main(int argc, char *argv[]) {
    [B aspect_hookSelector:@selector(foo) atPosition:AspectPositionBefore withBlock:^(id object, NSArray *arguments) {
        NSLog(@"before -[B foo]");
    }];
    [A aspect_hookSelector:@selector(foo) atPosition:AspectPositionBefore withBlock:^(id object, NSArray *arguments) {
        NSLog(@"before -[A foo]");
    }];

    B *b = [[B alloc] init];
    [b foo];
}
```

场景很简单，父类 hook 了某一个方法，子类中调用父类的该方法。熟悉 runtime 的都知道，调用父类方法时，调用者是子类本身，方法实现是执行父类的 IMP。这个时候由于父类进行了 hook，IMP 指向 `msg_forward`，因此会进行消息转发。消息转发时，`NSInvocation` 只知道调用者是子类，并不知道其实调用的是父类方法。因此，还会执行子类方法。也就是说，变成了子类中该方法又调用了自身，进而形成**无限循环**。

因此，Aspects 的作者禁止了同一个继承链上的多次 hook。当然这是为了解决这种无限循环的无奈之举。禁用某些操作以达到安全性，当然这也限制了一些实际需求的实现。

> JSPatch 在执行到 super 方法的时候会判断 super 方法是否被重写，如果被重写了那么执行重写的 JS 方法。而不是像 Aspects 中非常武断的只是不让一个继承链 hook 一次。

#### 创建 `AspectsContainer` 实例

前面说到，每一个被 hook 的实例或者对象的每一个方法都会有一个 `AspectsContainer` 以关联对象的方式存储。即方法 `aspect_getContainerForObject`

```objc
static AspectsContainer *aspect_getContainerForObject(NSObject *self, SEL selector) {
    NSCParameterAssert(self);
    /// 给 SEL 增加 aspects_ 前缀
    SEL aliasSelector = aspect_aliasForSelector(selector);
    /// 从当前类或者对象中通过关联对象取出该 SEL 对应的 AspectsContainer
    AspectsContainer *aspectContainer = objc_getAssociatedObject(self, aliasSelector);
    if (!aspectContainer) {
        aspectContainer = [AspectsContainer new];
        objc_setAssociatedObject(self, aliasSelector, aspectContainer, OBJC_ASSOCIATION_RETAIN);
    }
    return aspectContainer;
}
```

方法很简单，就是先查看这个对象上有没有方法名对应的关联对象，有就取出，没有就创建。

> **这就说明了，类对象和元类对象都是可以设置关联对象的**

#### 创建 `AspectsIdentifier` 实例

`AspectsIdentifier` 将 hook 的信息保存起来，它会做两件事：

1. 获取 block 的签名
2. 比较 block 的签名和 hook 的方法签名是否一致

```objc
/// 创建一个 AspectIdentifier
+ (instancetype)identifierWithSelector:(SEL)selector object:(id)object options:(AspectOptions)options block:(id)block error:(NSError **)error {
    NSCParameterAssert(block);
    NSCParameterAssert(selector);
    /// 拿到block的signature
    NSMethodSignature *blockSignature = aspect_blockMethodSignature(block, error); // TODO: check signature compatibility, etc.
    /// 比较 block signature 和 method signature 是否一致
    if (!aspect_isCompatibleBlockSignature(blockSignature, object, selector, error)) {
        return nil;
    }

    /// 创建一个  AspectIndentifier 设置 selector  block  signature  options  object
    AspectIdentifier *identifier = nil;
    if (blockSignature) {
        identifier = [AspectIdentifier new];
        identifier.selector = selector;
        identifier.block = block;
        identifier.blockSignature = blockSignature;
        identifier.options = options;
        identifier.object = object; // weak
    }
    return identifier;
}
```

##### 获取 block 签名

获取 block 签名的方式在 jspatch 中有类似的操作。通过 block 的地址，参考 block 结构体的内存分布，将指针移动到 signature 的位置。区别在于 JSPatch hook 的 block 全都是全局 block，因此 block 结构中不会存在 copy 和 dispose 两个方法。而 Aspects 中的 block 有可能是堆 block。当是堆 block 的时候，要移动两个方法指针的大小：

```objc
/// 获取 block 的签名
static NSMethodSignature *aspect_blockMethodSignature(id block, NSError **error) {
    // 把 block 转为 AspectBlockRef 类型
    AspectBlockRef layout = (__bridge void *)block;
    // 没有签名
	if (!(layout->flags & AspectBlockFlagsHasSignature)) {
        NSString *description = [NSString stringWithFormat:@"The block %@ doesn't contain a type signature.", block];
        AspectError(AspectErrorMissingBlockSignature, description);
        return nil;
    }
	void *desc = layout->descriptor;
    // 移动两个 int 的位置。
	desc += 2 * sizeof(unsigned long int);
    // 有 copy 和 dispose 函数。那么 要再移动两个指针的位置。堆 block 有 copy 和 dispose 两个函数
	if (layout->flags & AspectBlockFlagsHasCopyDisposeHelpers) {
		desc += 2 * sizeof(void *);
    }
	if (!desc) {
        NSString *description = [NSString stringWithFormat:@"The block %@ doesn't has a type signature.", block];
        AspectError(AspectErrorMissingBlockSignature, description);
        return nil;
    }
	const char *signature = (*(const char **)desc);
	return [NSMethodSignature signatureWithObjCTypes:signature];
}
```

`AspectBlockRef` 的结构如下：

```objc
typedef struct _AspectBlock {
    __unused Class isa;
    AspectBlockFlags flags;
    __unused int reserved;
    void(__unused *invoke)(struct _AspectBlock *block, ...);
    struct {
        unsigned long int reserved;
        unsigned long int size;
        // requires AspectBlockFlagsHasCopyDisposeHelpers
        void (*copy)(void *dst, const void *src);
        void (*dispose)(const void *);
        // requires AspectBlockFlagsHasSignature
        const char *signature;
        const char *layout;
    } * descriptor;
    // imported variables
} * AspectBlockRef;
```

##### 比较 block 的签名和 hook 的方法签名

```objc
/// 比较 block 参数类型和方法参数类型是否一致
static BOOL aspect_isCompatibleBlockSignature(NSMethodSignature *blockSignature, id object, SEL selector, NSError **error) {
    BOOL signaturesMatch = YES;
    // 获取 selector 的签名
    NSMethodSignature *methodSignature = [[object class] instanceMethodSignatureForSelector:selector];
    if (blockSignature.numberOfArguments > methodSignature.numberOfArguments) {
        // block 参数数量大于方法的参数数量就是不匹配
        signaturesMatch = NO;
    }else {
        if (blockSignature.numberOfArguments > 1) {
            // Aspects 规定，如果有参数，第一个必须是 AspectInfo
            const char *blockType = [blockSignature getArgumentTypeAtIndex:1];
            if (blockType[0] != '@') {
                signaturesMatch = NO;
            }
        }
        // Argument 0 is self/block, argument 1 is SEL or id<AspectInfo>. We start comparing at argument 2.
      	// index 为 0 的参数是 self 或者 block，index 为 1 的参数，是 SEL 或者 Aspects 规定的 AspectInfo 类型，因此从 index 为 2 的参数开始比较
        // The block can have less arguments than the method, that's ok.
        // 一个一个参数对比
        if (signaturesMatch) {
            for (NSUInteger idx = 2; idx < blockSignature.numberOfArguments; idx++) {
                const char *methodType = [methodSignature getArgumentTypeAtIndex:idx];
                const char *blockType = [blockSignature getArgumentTypeAtIndex:idx];
                // Only compare parameter, not the optional type data.
                if (!methodType || !blockType || methodType[0] != blockType[0]) {
                    signaturesMatch = NO; break;
                }
            }
        }
    }

    /// 不匹配直接报错
    if (!signaturesMatch) {
        NSString *description = [NSString stringWithFormat:@"Block signature %@ doesn't match %@.", blockSignature, methodSignature];
        AspectError(AspectErrorIncompatibleBlockSignature, description);
        return NO;
    }
    return YES;
}
```

比较两者的签名主要是比较入参的类型要一致。block 和普通方法的不同在于 block 的签名中不存在 SEL。普通方法的签名 index 为 0 的参数是调用者，即 `@`，index 为 1 的参数是 SEL，即 `:`，而 block 由于不存在 SEL，其 index 为 0 的参数还是调用者，即 `@`，而 index 为 1 的参数就是真正的参数了。Aspects 为了将 block 签名和普通方法签名一致，所以做了限制，只要有参数，第一个一定是一个无关的 `AspectInfo` 实例。

在创建好 `AspectsIdentifier` 后，将实例存放到 `AspectContainer` 中。

#### 实现 hook 方法

hook 方法的真正地方在 `aspect_prepareClassAndHookSelector` 方法中。分为两步：

1. 替换类的 `forwardInvocation` 方法为自己的实现
2. 替换方法的实现为 `_objc_msgForward`

```objc
/// hook 方法的地方
static void aspect_prepareClassAndHookSelector(NSObject *self, SEL selector, NSError **error) {
    NSCParameterAssert(selector);
    /// 返回 hook 后的类
    /// 实例对象就返回一个新类，类对象替换 forwardInvocation 后返回自身
    Class klass = aspect_hookClass(self, error);
    /// 获取该类要 hook 的方法
    Method targetMethod = class_getInstanceMethod(klass, selector);
    IMP targetMethodIMP = method_getImplementation(targetMethod);
    /// 如果该方法不是指向 _objc_msgForward，说明没有被 hook 过，要 hook 一下
    if (!aspect_isMsgForwardIMP(targetMethodIMP)) {
        // Make a method alias for the existing method implementation, it not already copied.
        /// 获取方法的签名
        const char *typeEncoding = method_getTypeEncoding(targetMethod);
        /// 获取添加了前缀的方法名
        SEL aliasSelector = aspect_aliasForSelector(selector);
        /// 添加方法然后 replace
        if (![klass instancesRespondToSelector:aliasSelector]) {
            __unused BOOL addedAlias = class_addMethod(klass, aliasSelector, method_getImplementation(targetMethod), typeEncoding);
            NSCAssert(addedAlias, @"Original implementation for %@ is already copied to %@ on %@", NSStringFromSelector(selector), NSStringFromSelector(aliasSelector), klass);
        }

        // We use forwardInvocation to hook in.
        class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);
        AspectLog(@"Aspects: Installed hook for -[%@ %@].", klass, NSStringFromSelector(selector));
    }
}
```

##### 替换类的 `forwardInvocation`  实现

替换在 `aspect_hookClass` 中实现。这是一个比较重要也比较有技巧性的方法。如注释上写的，对于 hook 对象主要区分为三种情况：

1. hook 的是类对象(包括元类对象，以下都简称为类对象)
2. hook 的是 kvo 对象
3. hook 的是实例对象

```objc
/// hook 对象
/// 如果传入的是实例对象，那么就新建一个类，并把实例对象的 isa 指针指向它
/// 如果传入的是类对象，那么就直接把类对象的 forwardInvocation 替换掉
/// 如果是 KVO 实例对象，也是直接把 KVO 实例对象的 forwardInvocation 替换掉
static Class aspect_hookClass(NSObject *self, NSError **error) {
    NSCParameterAssert(self);

    /// .class 当 target 是 Instance 则返回 Class，当 target 是 Class 则返回自身
    /// object_getClass 返回 isa 指针的指向
    /// 对于实例对象，.class object_getClass 返回的对象是一样的
    /// 对于类对象，.class 返回自身，object_getClass 返回 meta class
    
    /// 但是实例对象中有一个特殊情况，对于 KVO 对象，.class 返回的如果是 ObjectClass，那么 object_getClass 返回的就是 NSKVONotifing_ObjectClass
    /// 这是因为，KVO 悄悄创建了一个新类，并且重写了 .class 方法
    
	Class statedClass = self.class;
	Class baseClass = object_getClass(self);
	NSString *className = NSStringFromClass(baseClass);

    // Already subclassed
	if ([className hasSuffix:AspectsSubclassSuffix]) {
        /// 如果有 _Aspects_ 前缀，说明已经 hook 过了,直接返回 class
		return baseClass;

        // We swizzle a class object, not a single object.
	}else if (class_isMetaClass(baseClass)) {
        /// 如果是类方法,替换这个类的 forwardInvocation 方法，然后返回该类
        return aspect_swizzleClassInPlace((Class)self);
        // Probably a KVO'ed class. Swizzle in place. Also swizzle meta classes in place.
    }else if (statedClass != baseClass) {
        /// KVO 对象也要替换 KVO 类的 forwardInvocation 方法，然后返回 KVO 的类
        /// KVO 对象要 hook NSKVONotifing_ObjectClass
        return aspect_swizzleClassInPlace(baseClass);
    }

    // Default case. Create dynamic subclass.
    /// 上面已经把类对象过滤掉了，下面的逻辑都是实例对象的
    /// 动态创建当前类的子类，_Aspects_xxx
	const char *subclassName = [className stringByAppendingString:AspectsSubclassSuffix].UTF8String;
	Class subclass = objc_getClass(subclassName);
	if (subclass == nil) {
		subclass = objc_allocateClassPair(baseClass, subclassName, 0);
		if (subclass == nil) {
            NSString *errrorDesc = [NSString stringWithFormat:@"objc_allocateClassPair failed to allocate class %s.", subclassName];
            AspectError(AspectErrorFailedToAllocateClassPair, errrorDesc);
            return nil;
        }

        /// 替换这个子类的  forwardInvocation 实现
		aspect_swizzleForwardInvocation(subclass);
        /// 替换新创建的子类的 class 方法实现，返回原类要返回的类
		aspect_hookedGetClass(subclass, statedClass);
		aspect_hookedGetClass(object_getClass(subclass), statedClass);
        /// 创建这个类
		objc_registerClassPair(subclass);
	}

    /// 设置当前实例的 isa 指向新创建的 class
	object_setClass(self, subclass);
	return subclass;
}
```

`[xxx class]` 方法和 `object_getClass(xxx)` 的区别是要了解的：

-  `[xxx class]` 当 `xxx` 是实例对象的时候返回的是类对象，当是类对象的时候，返回的是自身
- `object_getClass(xxx)` 返回的是 isa 指向的对象。

所以 `class_isMetaClass(object_getClass(self))` 如果是 true，那就说明 self 是类对象。直接通过 `aspect_swizzleClassInPlace` 方法替换其 `forwardInvocation` 实现。

对于类对象来说，一般情况下 `[xxx class]` 和 `object_getClass(xxx)` 返回的结果是一致的。但有一种特殊情况就是 KVO 对象，KVO 会动态生成一个类型，但是会重写修改 class 方法返回的结果。因此，`[xxx class] !== object_getClass(xxx)` 的时候就说明是 KVO 生成的对象。KVO 对象要 hook `NSKVONotifing_xxx` 的 `forwardInvocation` 方法。

前面把类对象都排除了，只剩下实例对象 hook 某个方法的情况。那么如何既不影响其他实例，又能 hook 它的一个的方法呢？我们可以拷贝这个类创建一个新类，然后只修改这个新类的这个方法的实现，就和 Linux 中创建新进程一样，也是实现 KVO 的做法。对于新创建的类，我们修改其 `forwardInvocation` 方法的指向，并通过 `object_setClass` 将当前实例的 isa 指向新创建的 class。

##### 替换方法的实现为 `_objc_msgForward`

这一段的过程在上面的注释中已经很清楚了。主要就是添加一个有前缀的方法，将原来的 IMP 指向它，然后将 `_objc_msgForward` 指向原来的 SEL 。

至此，hook 方法全部完成。

### 执行

方法执行时会直接进入消息转发流程:

```objc
// This is a macro so we get a cleaner stack trace.
#define aspect_invoke(aspects, info) \
for (AspectIdentifier *aspect in aspects) {\
    [aspect invokeWithInfo:info];\
    if (aspect.options & AspectOptionAutomaticRemoval) { \
        aspectsToRemove = [aspectsToRemove?:@[] arrayByAddingObject:aspect]; \
    } \
}

// This is the swizzled forwardInvocation: method.
/// msgForward 执行的方法
static void __ASPECTS_ARE_BEING_CALLED__(__unsafe_unretained NSObject *self, SEL selector, NSInvocation *invocation) {
    NSCParameterAssert(self);
    NSCParameterAssert(invocation);
    SEL originalSelector = invocation.selector;
	SEL aliasSelector = aspect_aliasForSelector(invocation.selector);
    /// invocation 的 selector 转为原方法
    invocation.selector = aliasSelector;
    /// 获取这个对象对应的 AspectsContainer
    AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
    /// 获取这个对象的 isa 对应的 AspectContainer，如果没有就一直到父类找
    AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
    AspectInfo *info = [[AspectInfo alloc] initWithInstance:self invocation:invocation];
    NSArray *aspectsToRemove = nil;

    // Before hooks.
    /// 执行 container 相应生命周期的所有 aspect
    aspect_invoke(classContainer.beforeAspects, info);
    aspect_invoke(objectContainer.beforeAspects, info);

    // Instead hooks.
    BOOL respondsToAlias = YES;
    /// 如果替换数组存在，那么替换，否则执行原方法
    if (objectContainer.insteadAspects.count || classContainer.insteadAspects.count) {
        aspect_invoke(classContainer.insteadAspects, info);
        aspect_invoke(objectContainer.insteadAspects, info);
    }else {
        Class klass = object_getClass(invocation.target);
        do {
            if ((respondsToAlias = [klass instancesRespondToSelector:aliasSelector])) {
                [invocation invoke];
                break;
            }
        }while (!respondsToAlias && (klass = class_getSuperclass(klass)));
    }

    // After hooks.
    aspect_invoke(classContainer.afterAspects, info);
    aspect_invoke(objectContainer.afterAspects, info);

    // If no hooks are installed, call original implementation (usually to throw an exception)
    /// 调用原方法
    if (!respondsToAlias) {
        invocation.selector = originalSelector;
        SEL originalForwardInvocationSEL = NSSelectorFromString(AspectsForwardInvocationSelectorName);
        if ([self respondsToSelector:originalForwardInvocationSEL]) {
            ((void( *)(id, SEL, NSInvocation *))objc_msgSend)(self, originalForwardInvocationSEL, invocation);
        }else {
            [self doesNotRecognizeSelector:invocation.selector];
        }
    }

    // Remove any hooks that are queued for deregistration.
    [aspectsToRemove makeObjectsPerformSelector:@selector(remove)];
}
```

整个过程就是从关联对象中取出 `AspectContainer` 实例，然后执行其中各个时间点的回调。你可能会很疑惑它获取关联对象和执行的语句：

```objc
// 获取
AspectsContainer *objectContainer = objc_getAssociatedObject(self, aliasSelector);
AspectsContainer *classContainer = aspect_getContainerForClass(object_getClass(self), aliasSelector);
// 执行
aspect_invoke(classContainer.afterAspects, info);
aspect_invoke(objectContainer.afterAspects, info);
```

为什么既有从对象中获取关联对象，又有从对象的 isa 指向中获取关联对象的情况呢？对于 hook 类来说，实例的执行都应该是通过后者，从类中获取的。但是对于 hook 实例来说，他们的关联对象是存在实例中的。如果既 hook 了实例的某个方法，又 hook 了实例所在的类的同一个方法，那么两份 hook 都应该执行。

如果 hook 时候传入的 options 为 `AspectOptionAutomaticRemoval`，那么会在执行完毕后调用 `AspectIdentifier` 实例的 `remove` 方法移除 hook。

方法执行过程在 `invokeWithInfo` 方法中：

```objc
/// 执行这个 NSInvocation
- (BOOL)invokeWithInfo:(id<AspectInfo>)info {
    /// 创建一个 blockInvocation
    NSInvocation *blockInvocation = [NSInvocation invocationWithMethodSignature:self.blockSignature];
    /// 拿出原始的 Invocation
    NSInvocation *originalInvocation = info.originalInvocation;
    NSUInteger numberOfArguments = self.blockSignature.numberOfArguments;

    // Be extra paranoid. We already check that on hook registration.
    /// 在创建的时候已经校验过了，这里再校验一次
    if (numberOfArguments > originalInvocation.methodSignature.numberOfArguments) {
        AspectLogError(@"Block has too many arguments. Not calling %@", info);
        return NO;
    }

    /// 设置block中的上下文
    if (numberOfArguments > 1) {
        /// block 比较特殊，设置它的 NSInvocation 时， index 为 0 的 argument 为 self, index 从 1 开始就是正常入参了
        /// 普通的 NSInvocation，index 为 0 的 argument 是 self，index 为 1 的 argument 是 SEL
        /// Aspects 为了让 block 的参数也从 2 开始，默认将参数 1 设置为 AspectInfo
        [blockInvocation setArgument:&info atIndex:1];
    }
    
	void *argBuf = NULL;
    /// 依次从原始的 Invocation 中将参数取出设置到  blockInvocation 中
    for (NSUInteger idx = 2; idx < numberOfArguments; idx++) {
        const char *type = [originalInvocation.methodSignature getArgumentTypeAtIndex:idx];
		NSUInteger argSize;
		NSGetSizeAndAlignment(type, &argSize, NULL);
        
		if (!(argBuf = reallocf(argBuf, argSize))) {
            AspectLogError(@"Failed to allocate memory for block invocation.");
			return NO;
		}
        
		[originalInvocation getArgument:argBuf atIndex:idx];
		[blockInvocation setArgument:argBuf atIndex:idx];
    }
    
    /// 执行这个 block
    [blockInvocation invokeWithTarget:self.block];
    
    if (argBuf != NULL) {
        free(argBuf);
    }
    return YES;
}
```

和之前获取参数的时候类似，这里把参数一个个从 NSInvocation 中设置进去。最后通过 `invokeWithTarget` 执行 block。

## 总结

Aspects 的核心原理和 JSPatch 是一致的。当然，它比 JSPatch 还是简单了许多。如果你理解了 JSPatch 中消息转发的过程，Aspects 理解起来就很简单了。



## 思考题

1. 如何 hook 类方法？
2. 如何 hook 实例对象的方法？
3. block 的方法签名和普通方法的方法签名有什么不同？
4. 如何拿到 block 的方法签名？
5. 为什么 Aspects 中一个方法同一个继承链上只能 hook 一次？可以怎么改进？
6. 怎样判断一个实例对象是否进行过 kvo？
7. aspect 中怎么判断一个对象的 `forwardInvocation` 是否已经被 hook 过？
8. 如何判断一个对象是类对象还是实例对象？
9. `[self class]` 和 `object_getClass(self)` 的区别是什么？
10. 关联对象能否作用于类和元类？

## 参考

[静下心来读源码之Aspects](<https://juejin.im/post/5ab1b8996fb9a028e25d6c37#heading-17>)

[从 Aspects 源码中我学到了什么？](https://lision.me/aspects/)(写的太简略了)











