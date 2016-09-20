title: runtime原理笔记
date: 2016/8/22 14:07:12  
categories: IOS
tags: [runtime]

---

最近尝试往category中添加实例变量失败，但是可以添加方法。于是，研究了一下runtime的实现和使用。runtime的原理展开来分析的话还是挺复杂的。下载了最新的源码（objc4-680）,发现和现在网上的多数教程还是有出入的，跟进起来很困难。
所以尝试自己整理了一下，加了一些自己的理解，具体细节可能有些小出入。参考文章：[Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)	
<!--more-->

## Runtime基础
在Objective-C中，使用`[receiver message]`语法并不会马上执行`receiver`对象的`message`方法的代码，而是向`receiver`发送一条`message`消息。
其实`[receiver message]`被编译器转化为:
```objc
id objc_msgSend ( id self, SEL op, ... );
```
现在可以看出`[receiver message]`真的不是一个简简单单的方法调用。因为这只是在编译阶段确定了要向接收者发送`message`这条消息，而`receive`将要如何响应这条消息，那就要看运行时发生的情况来决定了。如果消息的接收者能够找到对应的`selector`，那么就相当于直接执行了接收者这个对象的特定方法；否则，消息要么被转发，或是临时向接收者动态添加这个`selector`对应的实现内容，要么就干脆玩完崩溃掉。

### id
`objc_msgSend`第一个参数类型为`id`,它是一个指向`objc_object`结构体指针,它包含一个`Class isa`指针(完整的定义在`objc_private.h`中)：
```objc
typedef struct objc_object *id;

struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

### SEL
`objc_msgSend`函数第二个参数类型为`SEL`，它是`selector`在Objc中的表示类型.`selector`是方法选择器，可以理解为区分方法的 ID，而这个 ID 的数据结构是`SEL`:
```objc
typedef struct objc_selector *SEL;
```

### Class
之所以说`isa`是指针是因为`Class`其实是一个指向`objc_class`结构体的指针：
```objc
typedef struct objc_class *Class;
```

这个结构体在`runtime.h`中有定义:
```objc
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif
} OBJC2_UNAVAILABLE;
```
其中`OBJC2_UNAVAILABLE`表示：在objc2中这些属性已经不在此定义。

在objc2中，`objc_class`的完整定义在`objc-runtime-new.h`中：
```objc
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};

struct objc_class : objc_object {
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() { 
        return bits.data();
    }
};
```

和`runtime.h`中的定义对比，少了许多属性，多出一个`bits`属性。这些属性都包含在了`class_rw_t`的`data`对象中。

注意到`objc_class`中也有一个`isa`对象(继承自`objc_object`)，这是因为一个ObjC类本身同时也是一个对象，为了处理类和对象的关系，runtime库创建了一种叫做 **元类 (Meta Class)**的东西，类对象所属类型就叫做元类，它用来表述类对象本身所具备的元数据。**类方法就定义于此处**，因为这些方法可以理解成类对象的实例方法。每个类仅有一个类对象，而每个类对象仅有一个与之相关的元类。
当你发出一个类似`[NSObject alloc]`的消息时，你事实上是把这个消息发给了一个**类对象 (Class Object)** ，这个类对象必须是一个元类的实例，而这个元类同时也是一个**根元类 (root meta class)**的实例。所有的元类最终都指向根元类为其超类。所有的元类的方法列表都有能够响应消息的类方法。所以当 `[NSObject alloc]`这条消息发给类对象的时候，`objc_msgSend()`会去它的元类里面去查找能够响应消息的方法，如果找到了，然后对这个类对象执行方法调用。
![Class isa and superclass relationship](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/runtime_principle.jpg?raw=true)

### class_rw_t
`bits` 中包含了一个指向 `class_rw_t` 结构体的指针，它的定义如下:
```objc
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;
}
```

其中，`const class_ro_t *ro`包含了所有**成员变量**的一维数组以及**各个基础的(base)属性、方法和协议**
`method_array_t methods`、`property_array_t properties`、`protocol_array_t protocols`则包含了所有**拓展的属性方法和协议**。**注意：这里的属性只是getset方法，不包含成员变量。**
这也就解释了，开篇不能添加成员变量的原因。

具体如何应用可看下一篇。

### 部分类的声明
这部分将上面涉及的类的声明贴出以供参照，可以直接跳过。

#### class_ro_t:
```objc
struct class_ro_t {
	...
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;
    property_list_t *baseProperties;
    ...
};
```
#### method_t
`method_list_t`和`method_array_t`内都是`method_t`类型:
```objc
struct method_t {
    SEL name;
    const char *types;
    IMP imp;
}
```
- 方法名类型为`SEL`
- 方法类型`types`是个`char`指针，其实存储着方法的参数类型和返回值类型。
- `imp`指向了方法的实现，本质上是一个函数指针

#### ivar_t
`ivar_list_t`和`ivar_array_t`内都是`ivar_t`类型:
```objc
struct ivar_t {
    int32_t *offset;
    const char *name;
    const char *type;
}
```

#### property_t
`ivar_list_t`和`ivar_array_t`内都是`ivar_t`类型:
```objc
struct property_t {
    const char *name;
    const char *attributes;
};
```

#### protocol_t
`protocol_list_t`和`protocol_array_t`内都是`protocol_t`类型:
```objc
struct protocol_t : objc_object {
    const char *mangledName;
    struct protocol_list_t *protocols;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    method_list_t *optionalInstanceMethods;
    method_list_t *optionalClassMethods;
    property_list_t *instanceProperties;
    uint32_t size;   // sizeof(protocol_t)
    uint32_t flags;
};
```

#### Ivar,Method,Category,objc_property_t
外部可以通过一系列的方法获得类的信息，诸如：`class_copyIvarList`,`class_copyPropertyList`等方法，可以获得类的实例变量和属性，对应的类型是`Ivar`、`objc_property_t`，它们分别是`ivar_t`,`property_t`类型的指针。

```objc
typedef struct method_t *Method;
typedef struct ivar_t *Ivar;
typedef struct category_t *Category;
typedef struct property_t *objc_property_t;
```

## 消息
### objc_msgSend函数
看起来像是`objc_msgSend`返回了数据，其实`objc_msgSend`从不返回数据而是你的方法被调用后返回了数据。下面详细叙述下消息发送步骤：
1. 检测这个 `selector` 是不是要忽略的。比如有了垃圾回收就不理会 `retain`, `release` 这些函数了。
2. 检测这个 `target` 是不是 `nil` 对象。ObjC 的特性是允许对一个 `nil`对象执行任何一个方法不会 Crash，因为会被忽略掉。
3. 如果上面两个都过了，那就开始查找这个类的 `IMP`，先从 `cache` 里面找，完了找得到就跳到对应的函数去执行。
4. 如果 `cache` 找不到就找一下方法分发表。
5. 如果分发表找不到就到超类的分发表去找，一直找，直到找到`NSObject`类为止。
6. 如果还找不到就要开始进入动态方法解析了.

编译器会根据情况在`objc_msgSend`, `objc_msgSend_stret`, `objc_msgSendSuper`, 或 `objc_msgSendSuper_stret`四个方法中选择一个来调用。如果消息是传递给超类，那么会调用名字带有`Super`的函数；如果消息返回值是数据结构而不是简单值时，那么会调用名字带有`stret`的函数。

当我们调用`[super selector]` 时，Runtime 会调用 `objc_msgSendSuper `方法，`objc_msgSendSuper` 方法有两个参数，`super` 和 `op`，`Runtime` 会把 `selector` 方法选择器赋值给 `op`。而 `super` 是一个 `objc_super` 结构体指针，`objc_super` 结构体定义如下：
```objc
struct objc_super { id receiver; Class class; };
```
需要注意的是：这里的`receiver`仍然是`self`本身。当我们想通过`[super class]`获取超类时，编译器只是将指向`self`的`id`指针和`class`的`SEL`传递给了`objc_msgSendSuper`函数，因为只有在`NSObject`类才能找到`class`方法，然后`class`方法调用`object_getClass()`，接着调用`objc_msgSend(objc_super->receiver, @selector(class))`，传入的第一个参数是指向`self`的`id`指针，与调用`[self class]`相同，所以我们得到的永远都是`self`的类型。

举个栗子，问下面的代码输出什么：

```objc
@implementation Son : Father
- (id)init
{
    self = [super init];
    if (self)
    {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}
@end
```
答案是全部输出 `Son`.

当调用 `[super class]` 时，会转换成 `objc_msgSendSuper` 函数：
1. 先构造 `objc_super` 结构体，结构体第一个成员就是 `self`。第二个成员是 `(id)class_getSuperclass(objc_getClass(“Son”))`.
2. 去 `Father` 这个类里去找 `- (Class)class`，没有，然后去 `NSObject` 类去找，找到了。最后内部是使用 `objc_msgSend(objc_super->receiver, @selector(class))` 去调用，此时已经和 `[self class]` 调用相同了，所以两个输出结果都是 `Son`。

这里调用`class`只是举例，所有调用`super`的方法，最后都变成了`self`的调用。


## Category 的实现原理
### 概述
runtime 依赖于 dyld 动态加载，在 objc-os.mm 文件中可以找到入口，它的调用栈简单整理如下:
```objc
void _objc_init(void)
└──const char \*map_2_images(...)
    └──const char \*map_images_nolock(...)
        └──void _read_images(header_info **hList, uint32_t hCount)
        	└──static void remethodizeClass(Class cls)
        		└──static void attachCategories(Class cls, category_list *cats, bool flush_caches)
```
### Category 相关的数据结构
首先来了解一下一个 Category 是如何存储的，在 objc-runtime-new.h 中可以看到如下定义：
```objc
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
};
```
从category的定义也可以看出category的可为（可以添加实例方法，类方法，甚至可以实现协议，添加属性）和不可为（无法添加实例变量）。

### 处理 Category
对 Category 中方法的解析并不复杂，首先来看一下 attachCategories 的简化版代码:
```objc
static void attachCategories(Class cls, category_list *cats, bool flush_caches) {
    if (!cats) return;
    bool isMeta = cls->isMetaClass();

    method_list_t **mlists = (method_list_t **)malloc(cats->count * sizeof(*mlists));
    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int i = cats->count;
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);
}
```

首先，通过 `while` 循环，我们遍历所有的 category，也就是参数 `cats` 中的 `list` 属性。对于每一个 category，得到它的方法列表 `mlist` 并存入 `mlists` 中。换句话说，我们将所有 category 中的方法拼接到了一个大的二维数组中，数组的每一个元素都是装有一个 category 所有方法的容器。

在 `while` 循环外，我们得到了拼接成的方法，此时需要与类原来的方法合并:
```objc
auto rw = cls->data();
rw->methods.attachLists(mlists, mcount);
```
`rw` 是一个 `class_rw_t`类型的结构体指针。根据 runtime 中的数据结构，它有一个 `methods` 结构体成员，并从父类继承了 `attachLists` 方法，用来合并 category 中的方法:
```objc
void attachLists(List* const * addedLists, uint32_t addedCount) {
    if (addedCount == 0) return;
    uint32_t oldCount = array()->count;
    uint32_t newCount = oldCount + addedCount;
    setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
    array()->count = newCount;
    memmove(array()->lists + addedCount, array()->lists, oldCount * sizeof(array()->lists[0]));
    memcpy(array()->lists, addedLists, addedCount * sizeof(array()->lists[0]));
}
```
需要注意的是，无论执行哪种逻辑，参数列表中的方法都会被添加到二维数组的前面。因此 category 中定义的同名方法不会替换类中原有的方法，但是对原方法的调用实际上会调用 category 中的方法。

