title: Masonry 源码解析 
date: 2018/4/19 10:07:12  
categories: iOS
tags:

- 源码解析

-----

Masonry 是 iOS 中的一套布局框架，先占个坑学习下源码。

<!--more-->

## 使用

### 添加约束

```objc
UIEdgeInsets padding = UIEdgeInsetsMake(10, 10, 10, 10);
[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(superview.mas_top).with.offset(padding.top); //with is an optional semantic filler
    make.left.equalTo(superview.mas_left).with.offset(padding.left);
    make.bottom.equalTo(superview.mas_bottom).with.offset(-padding.bottom);
    make.right.equalTo(superview.mas_right).with.offset(-padding.right);
}];
```

> 添加约束必须先把视图添加到父视图上，否则会crash

### 约束添加几倍于

```objc
make.width.equalTo(superview.mas_width).multiplieBy(0.5);
```

### 约束不超过或者不小于

```objc
// 不小于
make.left.greaterThanOrEqualTo(label.mas_left);
// 不大于
make.left.lessThanOrEqualTo(label.mas_left);
```

### 优先级

可以设置三种优先级 `.priorityHigh`，`.priorityMedium`，`.priorityLow`。也可以自己设置优先级的大小。默认的优先级为 1000

```objc
make.left.greaterThanOrEqualTo(label.mas_left).with.priorityLow();
make.top.equalTo(label.mas_top).with.priority(600);
```

### 设置边距大小和中心点

#### edgs

```objc
// make top, left, bottom, right equal view2
make.edges.equalTo(view2);

// make top = superview.top + 5, left = superview.left + 10,
//      bottom = superview.bottom - 15, right = superview.right - 20
make.edges.equalTo(superview).insets(UIEdgeInsetsMake(5, 10, 15, 20))
```

#### size

```objc
// make width and height greater than or equal to titleLabel
make.size.greaterThanOrEqualTo(titleLabel)

// make width = superview.width + 100, height = superview.height - 50
make.size.equalTo(superview).sizeOffset(CGSizeMake(100, -50))
```

#### center

```objc
// make centerX and centerY = button1
make.center.equalTo(button1)

// make centerX = superview.centerX - 5, centerY = superview.centerY + 10
make.center.equalTo(superview).centerOffset(CGPointMake(-5, 10))
```

### 更新约束

使用 `mas_updateConstraints` 更新约束

```objc
[self.growingButton mas_updateConstraints:^(MASConstraintMaker *make) {
    make.center.equalTo(self);
    make.width.equalTo(@(self.buttonSize.width)).priorityLow();
    make.height.equalTo(@(self.buttonSize.height)).priorityLow();
    make.width.lessThanOrEqualTo(self);
    make.height.lessThanOrEqualTo(self);
}];
```

### 重新设置约束

使用 `mas_remakeConstraints` 重新设置约束

```objc
[self.button mas_remakeConstraints:^(MASConstraintMaker *make) {
    make.size.equalTo(self.buttonSize);

    if (topLeft) {
        make.top.and.left.offset(10);
    } else {
        make.bottom.and.right.offset(-10);
    }
}];
```

### 多个控件等距、等宽排列

Masonry 提供了两个方法，可以提供等距排列或者等宽排列：

```objc
/**
 *  等距排列
 *
 *  @param axisType     横排还是竖排
 *  @param fixedSpacing 两个控件间隔
 *  @param leadSpacing  第一个控件与边缘的间隔
 *  @param tailSpacing  最后一个控件与边缘的间隔
 */
- (void)mas_distributeViewsAlongAxis:(MASAxisType)axisType withFixedSpacing:(CGFloat)fixedSpacing leadSpacing:(CGFloat)leadSpacing tailSpacing:(CGFloat)tailSpacing;

/**
 *  等宽排列
 *
 *  @param axisType        横排还是竖排
 *  @param fixedItemLength 控件的宽或高
 *  @param leadSpacing     第一个控件与边缘的间隔
 *  @param tailSpacing     最后一个控件与边缘的间隔
 */
- (void)mas_distributeViewsAlongAxis:(MASAxisType)axisType withFixedItemLength:(CGFloat)fixedItemLength leadSpacing:(CGFloat)leadSpacing tailSpacing:(CGFloat)tailSpacing;
```

要注意的是，如果是横排，那么还是需要自己设置高度约束；如果是竖排，还是需要设置宽度约束。例：

```objc
// 导入 NSArray 的分类
#import "NSArray+MASAdditions.h"

// 把视图添加到数组中
- (NSMutableArray *)masonryViewArray {
    if (!_masonryViewArray) {
        _masonryViewArray = [NSMutableArray array];
        for (int i = 0; i < 4; i ++) {
            UIView *view = [[UIView alloc] init];
            view.backgroundColor = [UIColor redColor];
            [self.view addSubview:view];
            [_masonryViewArray addObject:view];
        }
    }
    return _masonryViewArray;
}

// 设置约束
- (void)test_masonry_horizontal_fixSpace {
    // 实现masonry水平固定间隔方法
    [self.masonryViewArray mas_distributeViewsAlongAxis:MASAxisTypeHorizontal withFixedSpacing:30 leadSpacing:10 tailSpacing:10];
    
    // 设置array的垂直方向的约束
    [self.masonryViewArray mas_makeConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(150);
        make.height.equalTo(80);
    }];
}

```

> 这个方法在官方的 README 中没有看到，只是在阅读源码的时候才注意到

## 源码解析

### 文件结构

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/masonry_1.png?raw=true)

使用 Masonry 设置约束的时候，主要的几个类如上图所示。最上面的三个是调用者，分别在 `UIView`，`UIViewController` 和 `NSArray` 中添加分类方法。下面的 `MASConstraintMaker` 是约束的创建者，我们在 block 中使用的 `make` 就是这个类的实例。后面的 `MASContraint` 和 `MASViewAdditions` 不对使用者暴露，前者是约束实例的封装，后者是所约束对象的封装。

我们使用的时候主要调用 `View+MASAdditions`中的方法，直接在某个 View 上添加约束。  `NSArray+MASAdditions` 可以将多个视图放在一个数组中，然后对其中的每一个视图加上约束，用的很少。`ViewController+MASAdditions` 主要操作的是`topLayoutGuide` 和 `bottomLayoutGuide`。这两个 View 的属性在 iOS11 中已经被废弃，建议使用 View 中的 `mas_safeAreaLayoutGuide` 替代。因此，几乎不使用

### View+MASAdditions

这个 UIView 的分类提供了三个方法分别用来创建、更新、重设约束。先来看创建约束的方法：

```objc
- (NSArray *)mas_makeConstraints:(void(^)(MASConstraintMaker *))block {
    self.translatesAutoresizingMaskIntoConstraints = NO;
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    block(constraintMaker);
    return [constraintMaker install];
}
```

这个方法中首先把 `translatesAutoresizingMaskIntoConstraints` 设置为 NO，表示要自己添加约束。然后初始化 `MASConstraintMaker`。接着把 `MASConstraintMaker` 作为参数传入 block 执行，配置约束。最后调用`install` 方法，把配置好的越是添加到视图上。

> 这里外部传来的 block 不会被保存，而是直接执行。这就保证了 block 中直接强引用 self 也不会产生循环引用。

再看更新和重设方法：

```objc
- (NSArray *)mas_updateConstraints:(void(^)(MASConstraintMaker *))block {
    self.translatesAutoresizingMaskIntoConstraints = NO;
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    constraintMaker.updateExisting = YES;
    block(constraintMaker);
    return [constraintMaker install];
}

- (NSArray *)mas_remakeConstraints:(void(^)(MASConstraintMaker *make))block {
    self.translatesAutoresizingMaskIntoConstraints = NO;
    MASConstraintMaker *constraintMaker = [[MASConstraintMaker alloc] initWithView:self];
    constraintMaker.removeExisting = YES;
    block(constraintMaker);
    return [constraintMaker install];
}
```

和创建方法类似，只是多加了两个标记 `updateExisting`，`removeExisting`，用来和创建区别开。

### MASConstraintMaker

我们在设置约束的时候一般是这么写的：

```objc
[view1 mas_makeConstraints:^(MASConstraintMaker *make) {
    make.top.equalTo(superview.mas_top).with.offset(20); //with is an optional semantic filler
}];
```

我们来看看这个 `top` 方法如何实现的：

```objc
- (MASConstraint *)top {
    return [self addConstraintWithLayoutAttribute:NSLayoutAttributeTop];
}

- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    return [self constraint:nil addConstraintWithLayoutAttribute:layoutAttribute];
}

- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    MASViewAttribute *viewAttribute = [[MASViewAttribute alloc] initWithView:self.view layoutAttribute:layoutAttribute];
    MASViewConstraint *newConstraint = [[MASViewConstraint alloc] initWithFirstViewAttribute:viewAttribute];
		...
    if (!constraint) {
        newConstraint.delegate = self;
        [self.constraints addObject:newConstraint];
    }
    return newConstraint;
}
```

中间省略了一些暂时不涉及的代码。这里先是初始化了 `MASViewAttribute` 对象。由于 `MASViewAttribute` 只是简单的约束作用的视图以及一个枚举属性 `NSLayoutAttribute` (`NSLayoutAttribut` 就是在创建约束的时候标识约束的类型的，比如宽度，高度，上下左右等)的封装，比较简单，这里就直接看一些它的初始化方法：

```objc
- (id)initWithView:(MAS_VIEW *)view layoutAttribute:(NSLayoutAttribute)layoutAttribute {
    self = [self initWithView:view item:view layoutAttribute:layoutAttribute];
    return self;
}

- (id)initWithView:(MAS_VIEW *)view item:(id)item layoutAttribute:(NSLayoutAttribute)layoutAttribute {
    self = [super init];
    if (!self) return nil;
    
    _view = view;
    _item = item;
    _layoutAttribute = layoutAttribute;
    
    return self;
}
```

除了保存 `NSLayoutAttribute` 就是对 view 做了一个弱引用的保存。可以看到，除了 `view` 外，还有一个成员变量 `item`。一般情况下，`view` 和 `item` 是同一个东西。只有当添加的约束是 ViewController 的 `topLayoutGuide` 和 `bottomLayoutGuide` 时才会不同：

```objc
- (MASViewAttribute *)mas_topLayoutGuide {
    return [[MASViewAttribute alloc] initWithView:self.view item:self.topLayoutGuide layoutAttribute:NSLayoutAttributeBottom];
}
```

`MASViewAttribute` 的初始化方法之后，又初始化了 `MASViewConstraint` 对象，并把它加入到 `MASConstraintMaker` 的 `constraints` 约束数组中，并最终返回。 

### MASViewConstraint

```objc
make.top.equalTo(superview.mas_top).with.offset(20); 
```

`make.top` 返回的是 `MASViewConstraint` 对象实例，所以后面的 `equalTo` 方法就是 `MASViewConstraint` 的方法。

先来看它的初始化方法：

```objc
- (id)initWithFirstViewAttribute:(MASViewAttribute *)firstViewAttribute {
    self = [super init];
    if (!self) return nil;
    
    _firstViewAttribute = firstViewAttribute;
    self.layoutPriority = MASLayoutPriorityRequired;
    self.layoutMultiplier = 1;
    
    return self;
}
```

从名字就可以看出，初始化的时候必须传入一个 `MASViewAttribute` 实例。从名字也可以看得出来，这个属性是作为约束中的 `FirstView` 的。

再来看看 `equalTo` 的实现，这也就是这个库的精髓所在:

```objc
- (MASConstraint * (^)(id))equalTo {
    return ^id(id attribute) {
        return self.equalToWithRelation(attribute, NSLayoutRelationEqual);
    };
}

- (MASConstraint * (^)(id, NSLayoutRelation))equalToWithRelation {
    return ^id(id attribute, NSLayoutRelation relation) {
        self.layoutRelation = relation;
        self.secondViewAttribute = attribute;
        return self;
    };
}
```

链式编程的核心在于每次调用的时候都返回自身。OC 的点语法相当于调用属性的 get 方法，因此是无法传参的。`make.top.equalTo(superview.mas_top)` 相当于通过 `equalTo` 的 get 方法，返回了一个 block 实例，然后再执行这个 block，返回自身的实例。

> 链式编程的一般使用场景在于设置一个对象的多个属性。
>
> 比如你要设置一个 button，你要设置它的多个属性就需要在多行里分别设置。但是如果 button 的每个属性的设置都返回 button 自身，那么就可以在一行中链式的调用。

同样的，`offset` 方法也是一样的道理。

### 设置约束

一条链式调用会创建一个 `MASViewConstraint` 对象。每一个约束都会保存到 `MASConstraintMaker` 的数组中。在设置完约束后，就来到了 `[constraintMaker install]` 添加约束：

```objc
// MASConstraintMaker
- (NSArray *)install {
    if (self.removeExisting) {
        NSArray *installedConstraints = [MASViewConstraint installedConstraintsForView:self.view];
        for (MASConstraint *constraint in installedConstraints) {
            [constraint uninstall];
        }
    }
    NSArray *constraints = self.constraints.copy;
    for (MASConstraint *constraint in constraints) {
        constraint.updateExisting = self.updateExisting;
        [constraint install];
    }
    [self.constraints removeAllObjects];
    return constraints;
}
```

这里之前的两个标记就派上用场了，如果要重设就会先移除约束再重新添加，而如果是更新，就会把标记传入每一个约束中。`MASViewConstraint` 中添加约束：

```objc
// MASViewConstraint
- (void)install {
    /// 如果该约束已经被 install 过，那么直接 return。
    if (self.hasBeenInstalled) {
        return;
    }
    
    if ([self supportsActiveProperty] && self.layoutConstraint) {
        self.layoutConstraint.active = YES;
        /// 把约束保存到 view 上，这样再 uninstall 的时候就可以找到并移除了。
        [self.firstViewAttribute.view.mas_installedConstraints addObject:self];
        return;
    }
    
    /// 拿到施加约束的第一个视图
    MAS_VIEW *firstLayoutItem = self.firstViewAttribute.item;
    NSLayoutAttribute firstLayoutAttribute = self.firstViewAttribute.layoutAttribute;
    /// 拿到施加约束的第二个视图
    MAS_VIEW *secondLayoutItem = self.secondViewAttribute.item;
    NSLayoutAttribute secondLayoutAttribute = self.secondViewAttribute.layoutAttribute;

    /// 不存在第二个Attribute的时候设置为第一个属性的父级元素
    if (!self.firstViewAttribute.isSizeAttribute && !self.secondViewAttribute) {
        secondLayoutItem = self.firstViewAttribute.view.superview;
        secondLayoutAttribute = firstLayoutAttribute;
    }
    
    /// 创建真正的约束对象，他是 NSLayoutConstraint 的子类
    MASLayoutConstraint *layoutConstraint
        = [MASLayoutConstraint constraintWithItem:firstLayoutItem
                                        attribute:firstLayoutAttribute
                                        relatedBy:self.layoutRelation
                                           toItem:secondLayoutItem
                                        attribute:secondLayoutAttribute
                                       multiplier:self.layoutMultiplier
                                         constant:self.layoutConstant];
    
    layoutConstraint.priority = self.layoutPriority;
    layoutConstraint.mas_key = self.mas_key;
    
    if (self.secondViewAttribute.view) {
        /// 如果存在第二个 View，那么约束要加在这两个 View 的公共祖先视图上。因此要查找公共祖先
        MAS_VIEW *closestCommonSuperview = [self.firstViewAttribute.view mas_closestCommonSuperview:self.secondViewAttribute.view];
        NSAssert(closestCommonSuperview,
                 @"couldn't find a common superview for %@ and %@",
                 self.firstViewAttribute.view, self.secondViewAttribute.view);
        /// installedVi表示约束要添加到的视图
        self.installedView = closestCommonSuperview;
    } else if (self.firstViewAttribute.isSizeAttribute) {
        /// 如果没有第二个视图，并且是操作第一个视图的 size，那么就直接把约束加在这个视图上
        self.installedView = self.firstViewAttribute.view;
    } else {
        /// 如果没有第二个视图，并且不是操作第一个视图的 size，那么就表示添加的约束是第一个视图的位置的，那么需要把约束加在第一个视图的父视图上
        self.installedView = self.firstViewAttribute.view.superview;
    }


    MASLayoutConstraint *existingConstraint = nil;
    /// 如果外部标识是要更新约束
    if (self.updateExisting) {
        /// 判断这个约束是否存在，判断方式是拿这个约束的每一个属性和 view 保存的所有约束的每一个属性对比，如果完全相等就表示存在
        existingConstraint = [self layoutConstraintSimilarTo:layoutConstraint];
    }

    if (existingConstraint) {
        /// 如果这个约束存在 更新约束
        existingConstraint.constant = layoutConstraint.constant;
        self.layoutConstraint = existingConstraint;
    } else {
        /// 如果约束不存在，给视图增加约束
        [self.installedView addConstraint:layoutConstraint];
        self.layoutConstraint = layoutConstraint;
        [firstLayoutItem.mas_installedConstraints addObject:self];
    }
}
```

源码比较长，但是思想还是很明确的，就是调用 iOS 提供的 `NSLayoutConstraint` 方法生成约束，然后找到两个视图的公共父节点，将约束添加到公共父节点上。

### MASCompositeConstraint

最后，来瞧一下 `MASViewConstraint` 的子类 `MASCompositeConstraint`。之前我们省略的代码都是针对它的。简单的说，`MASCompositeConstraint` 就是 `MASViewConstraint` 的集合。

我们看一下这样设置约束的场景：

```objc
make.height.and.width.equaltTo(@20)
```

我们在 `make.height` 的时候，已经创建了一个 `MASViewConstraint` 实例。所以执行到 `.width` 的时候，调用的是 `MASViewConstraint` 的 `width` 方法。我们来对比一下两者的不同：

```objc
// MASConstraintMaker
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    return [self constraint:nil addConstraintWithLayoutAttribute:layoutAttribute];
}

// MASViewConstraint
- (MASConstraint *)addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    return [self.delegate constraint:self addConstraintWithLayoutAttribute:layoutAttribute];
}
```

两者的不同在于，用 `MASConstraintMaker` 创建的约束传入的是 nil，而通过 `MASViewConstraint` 创建的约束传入的是自身。接着来看下面的处理方法：

```objc
- (MASConstraint *)constraint:(MASConstraint *)constraint addConstraintWithLayoutAttribute:(NSLayoutAttribute)layoutAttribute {
    MASViewAttribute *viewAttribute = [[MASViewAttribute alloc] initWithView:self.view layoutAttribute:layoutAttribute];
    MASViewConstraint *newConstraint = [[MASViewConstraint alloc] initWithFirstViewAttribute:viewAttribute];
    if ([constraint isKindOfClass:MASViewConstraint.class]) {
        //replace with composite constraint
        NSArray *children = @[constraint, newConstraint];
        MASCompositeConstraint *compositeConstraint = [[MASCompositeConstraint alloc] initWithChildren:children];
        compositeConstraint.delegate = self;
        [self constraint:constraint shouldBeReplacedWithConstraint:compositeConstraint];
        return compositeConstraint;
    }
   ....
}
```

当 `constraint` 不为 nil 的时候，就会把原来的约束和现在的约束作为一个数组，创建一个 `MASCompositeConstraint` 实例。随后替换 `MASConstraintMaker` 中相应的值。

相应的，`install` 的时候，需要对 `MASCompositeConstraint` 中的所有约束调用 `install` 方法：

```objc
// MASCompositeConstraint
- (void)install {
    for (MASConstraint *constraint in self.childConstraints) {
        constraint.updateExisting = self.updateExisting;
        [constraint install];
    }
}
```

## 总结

约束的创建本身涉及到很多的属性设置，Masonry 使用链式语法的方式精简了大量的代码。整体而言，Masonry 并不是一个太难理解的库。





















 