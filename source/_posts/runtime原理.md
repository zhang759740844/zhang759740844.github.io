title: runtime原理笔记
date: 2016/8/22 14:07:12  
categories: IOS
tags: [runtime]

---

最近尝试往category中添加实例变量失败，但是可以添加方法。于是，研究了一下runtime的实现和使用。runtime的原理展开来分析的话还是挺复杂的。下载了最新的源码（objc4-680）,发现和现在网上的多数教程还是有出入的，跟进起来很困难。
所以尝试自己整理了一下，加了一些自己的理解，具体细节可能有些小出入。	
<!--more-->

## Runtime基础
在Objective-C中，使用`[receiver message]`语法并不会马上执行`receiver`对象的`message`方法的代码，而是向`receiver`发送一条`message`消息。
其实`[receiver message]`被编译器转化为:
```objc
id objc_msgSend ( id self, SEL op, ... );
```

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
和runtime.h中的定义对比，少了许多属性，多出一个`bits`属性。这些属性都包含在了`class_rw_t`的`data`对象中。

注意到`objc_class`中也有一个`isa`对象，这是因为一个ObjC类本身同时也是一个对象，为了处理类和对象的关系，runtime库创建了一种叫做元类 (Meta Class) 的东西，类对象所属类型就叫做元类，它用来表述类对象本身所具备的元数据。类方法就定义于此处，因为这些方法可以理解成类对象的实例方法。每个类仅有一个类对象，而每个类对象仅有一个与之相关的元类。
当你发出一个类似[NSObject alloc]的消息时，你事实上是把这个消息发给了一个类对象 (Class Object) ，这个类对象必须是一个元类的实例，而这个元类同时也是一个根元类 (root meta class) 的实例。所有的元类最终都指向根元类为其超类。所有的元类的方法列表都有能够响应消息的类方法。所以当 [NSObject alloc] 这条消息发给类对象的时候，objc_msgSend()会去它的元类里面去查找能够响应消息的方法，如果找到了，然后对这个类对象执行方法调用。
![Class isa and superclass relationship](http://7xwxux.com1.z0.glb.clouddn.com/runtime_principle.jpg)

### class_rw_t
bits 中包含了一个指向 class_rw_t 结构体的指针，它的定义如下:
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

其中，`const class_ro_t *ro`包含了所有成员变量的一维数组`ivar_list_t * ivars`
`method_array_t methods`则包含了所有方法的二维数组。
具体如何应用可看下一篇。

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
首先，通过 while 循环，我们遍历所有的 category，也就是参数 cats 中的 list 属性。对于每一个 category，得到它的方法列表 mlist 并存入 mlists 中。换句话说，我们将所有 category 中的方法拼接到了一个大的二维数组中，数组的每一个元素都是装有一个 category 所有方法的容器。

在 while 循环外，我们得到了拼接成的方法，此时需要与类原来的方法合并:
```objc
auto rw = cls->data();
rw->methods.attachLists(mlists, mcount);
```
rw 是一个 class_rw_t 类型的结构体指针。根据 runtime 中的数据结构，它有一个 methods 结构体成员，并从父类继承了 attachLists 方法，用来合并 category 中的方法:
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

