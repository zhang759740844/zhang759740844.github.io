title: oc 中的泛型与nullability
date: 2016/9/28 10:07:12  
categories: iOS
tags:
	- Objective-C
---

oc中添加了一些关键字，主要作用于编译期，对程序没有任何影响，相当于是一个提示。下面将结合[Objective—C语言的新魅力——Nullability、泛型集合与类型延拓](https://my.oschina.net/u/2340880/blog/514804)对oc的这些新特性进行研究。

<!--more-->

## Nullability检测的支持
可以使用`nullable`关键字，表明该对象可以是空值;使用`nonnull`关键字表明不可以是空对象。

```objc
@property (nullable, nonatomic, strong) ObjectType firstObject;
@property (nullable, nonatomic, strong) ObjectType lastObject;
```

这是NSArray中的两个属性，其中`nullable`关键字说明了这里可能返回空的值。

如果仅仅是在返回值中给开发者一些提示，你可能觉得应用并不大，是的，对开发者最大的帮助是这一特性可以用于函数的参数中，这样我们在调用函数时起到的提示作用，将是非常重要的,例如：

```objc
-(void)setValue:(NSNumber * _Nonnull )number{
	
}
```

如果传入了空值，编译器会警告：

![oc_new_1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/oc_new_1.png?raw=true)

## 泛型的使用
这一特性和Nullability一样，只作用于编译期，是为我们开发者服务的另一重要特性。

### 有类型约定的集合
在Xcode7中，我们可以给集合类型添加一个泛型的约定，如下：

```objc
NSMutableArray<NSString *> *array = [[NSMutableArray alloc]init];
```

声明了这样一个数组后，就告诉了编译器，这个数组中的数据类型都是NSString*类型的，可以使用该类型的方法，如：

![oc_new_2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/oc_new_2.png?raw=true)

在我们向这个数组中追加元素的时候，编译器将元素的类型提示了出来，并且将FromArray方法中需要的元素类型也提示了出来:

![oc_new_3](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/oc_new_3.png?raw=true)

如果我们向这个数组中追加类型不匹配的元素,编译器会给我们一个这样的警告：

![oc_new_4](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/oc_new_4.png?raw=true)

### 类型通配符
iOS的系统类中，大量使用了`ObjectType`关键字，如下是系统的`NSMutableArray`的头文件：

```objc
@interface NSMutableArray<ObjectType> : NSArray<ObjectType>
- (void)addObject:(ObjectType)anObject;
- (void)insertObject:(ObjectType)anObject atIndex:(NSUInteger)index;
- (void)removeLastObject;
- (void)removeObjectAtIndex:(NSUInteger)index;
- (void)replaceObjectAtIndex:(NSUInteger)index withObject:(ObjectType)anObject;
- (instancetype)init NS_DESIGNATED_INITIALIZER;
- (instancetype)initWithCapacity:(NSUInteger)numItems NS_DESIGNATED_INITIALIZER;
- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder NS_DESIGNATED_INITIALIZER;
@end
```

这个`ObjectType`就是泛型的类型标识符,我们可以用它自己定义一个集合类，如包装一个NSSarray：

```objc
@interface MyArray<Type> : NSObject
@property(nonatomic,strong,nonnull)NSMutableArray<Type> *array;
-(void)addObject:(nonnull Type)obj;
@end
```

实现如下：

```objc
- (instancetype)init
{
    self = [super init];
    if (self) {
        _array = [[NSMutableArray alloc]init];
    }
    return self;
}
- (void)addObject:(id)obj{
    [_array addObject:obj];
}
```

**注意**：
- 这个类型通配符只能在interfave里使用，作用域为@interface到@end之间。
- 实现时，对于定义的`Type`类型的参数，使用`id`代替。
- 泛型`Type`不用写成`Type *`。
- 对于多参的集合，将参数类型用“,”隔开即可，如`NSMultableDictionary<NSString * ,NSString *> *dic = [[NSMultableDictionary alloc] init];`

### 泛型和协议

#### 声明

泛型和协议都是用尖括号括起来的，可能会很困扰，编译器如何知道哪个是泛型哪个是协议。其实它们是有规则的。简单看一下例子：

```objc
@protocol proc
@end
// .h
@interface Test<ObjectClass> : NSObject<Proc>
@property (nonatomic,strong) ObjectClass param;
@end
```

我们看到，所有的类都是要有一个基类，所以泛型的尖括号是在基类之前的，基类之后可以写上协议，使用的时候就可以 `Test<SomeType *> test` 这样用啦。那如果基类也有泛型需要声明呢？比如 `NSMutableArray`:

```objc
@interface NSMutableArray<ObjectType> : NSArray<ObjectType>
```

类似于这种的泛型，我们就不能直接申明协议了。但是我们可以变通一下，可以为这个类创建一个分类，在分类中声明协议啊。

另外还有一点，协议中是不能使用泛型的，如果实在想用，拥抱 Swift 吧。

#### 使用

使用的时候一般情况下都是表示泛型，比如：

```objc
Test<NSString *> test = [[Test alloc] init];
```

但是也有表示协议的，不过局限在 id 类型，比如：

```objc
id<SomeProtocol> prop;
```

表示任意一个实现了某个协议的类型。

### __kindof
在小类向大类赋值时，往往没有任何问题。但是大类向小类赋值时，就会产生警告。譬如`UIView`的实例`view`和`UIButton`的实例`button`,可以`view = button`，但是最好不要`button = view`。如果想要使用，又不希望产生警告怎么办？使用`__kindof`。
```objc
UIButton *b = [[UIButton alloc] init];
__kindof UIView *view = [[UIButton alloc]init];
b = view;
```

这里，由于`view`使用`__kindof`表示`UIView`的子类，不会产生警告。否则要进行强转`b = (UIButton *)view;`





















