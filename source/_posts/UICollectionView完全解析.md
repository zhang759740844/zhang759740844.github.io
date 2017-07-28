title: 你真的会使用 UICollectionView 了吗？
date: 2017/7/27 14:07:12  
categories: iOS
tags: 
	- 基本控件	
---

UICollectionView 功能强大，方法众多。实现一个简单的布局并不难，但是你真的敢说自己会使用 UICollectionView 了吗？

本篇是 《iOS UICollectionView:The Complete Guide》 的读书笔记，与大家分享。

![书名](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/UICollectionViewBookName.jpeg?raw=true)

<!--more-->

## 基本展示

### UIScrollView 概述

UICollectionView 继承于 UIScrollView，`UICollectionViewDelegate` 协议继承于 `UIScrollViewDelegate` 协议。所以在使用 UICollectionView 的时候，可以直接使用 UIScrollView 的各个属性方法。

UIScrollView 中有几个重要的属性，`contentSize` 用来标识 UIScrollView 的滚动范围；`contentOffset` 用来设置视图原点与可视区域左上角的距离；`contentInset` 可用通过 `UIEdgeInsetMake(10,10,10,10)` 的方法设置一个内边框，这个值可以是负的，能使视图超出可视的滚动范围。

我们可以通过 `setContentOffset:animated:` 将视图滚动到某一个位置，也可以使用 `scrollRectToVisible:animated:` 将某块 rect 滚动到可视区域内（如果已经可见则不会滚动）。

一些有用的 UIScrollViewDelegate 的方法：

- `scrollViewDidScroll:`：在任何 `contentOffset` 发生改变的时刻调用。可以用来自定义一个下拉刷新控件。
- `scrollViewWillBeginDragging:`：在 scrollView 被拖拽的时候调用。可以在此时暂停一些复杂操作以保证滑动的流畅性。
- `scrollViewWillEndDragging:withVelocity:targetContentOffset:`：在用户结束拖拽并抬起手指后调用。第二个参数表示结束拖动时的速度。第三个参数表示最终结束滚动时的 `contentOffset` 值，这里是一个 `CGPoint` 的指针，我们可以让其指向一个新的 `CGPoint` 以自由控制最终的 `contentOffset` 大小。除了让我们改变最终停止的位置，这个方法还可以使我们能够对要展示区域的数据提前进行准备。
- `scrollViewShouldScrollToTop:`：在用户点击状态栏的时候回调，返回 `YES` 允许 scrollView 滚动到顶端。
- `scrollViewDidScrollToTop:`：在点击顶部状态栏并且滚动到顶端后回调。
- `scrollViewDidEndDecelerating:`：在结束减速动画的时候调用。可以用来继续之前暂停的复杂操作。



### 如何重用 UICollectionViewCell

可以通过 `UINib` 以及 `class` 两种方式注册 cell：

- `registerClass:forCellWithReuserIdentifier:`
- `registerNib:forCellWithReuseIdentifer:`

当 `dequeueReuseableCellWithReuseIdentifirer:forIndexPath:` 被调用的时候，你将获得一个被重用的 cell，如果此时没有可用的 cell，系统会帮你创建一个。



### 展现内容

#### 准备一个 cell 类

自定义一个类继承于 `UICollectionViewCell`:

```objc
@interface SimpleCollectionViewCell : UICollectionViewCell

@property (nonatomic,copy) NSString *num;

- (void)refreshData;

@end
```

UICollectionViewCell 提供了一个重用前清空 cell 内数据的方法 `prepareForReuse`，重写它，：

```objc
- (void)prepareForReuse {
    [super prepareForReuse];
    self.num = @"";
}
```

> 一定要调用 super 方法

#### 设置 ViewController

先添加一个 CollectionView，然后注册，先定义一个 static 的 Identifier：

```objc
static NSString *SIMPLECELLIDENTIFIER = @"Simple Cell Identifier";

- (void)viewDidLoad {
    [super viewDidLoad];
    // 设置 flowLayout
    UICollectionViewFlowLayout *flowLayout = [[UICollectionViewFlowLayout alloc] init];
    flowLayout.minimumInteritemSpacing = 40;
    flowLayout.minimumLineSpacing = 40;
    flowLayout.sectionInset = UIEdgeInsetsMake(10, 10, 10, 10);
    flowLayout.itemSize = CGSizeMake(50, 50);
    

    // 添加 collectionView，记得要设置 delegate 和 dataSource 的代理对象
    _collectionView = [[UICollectionView alloc] initWithFrame:self.view.frame collectionViewLayout:flowLayout];
    _collectionView.delegate = self;
    _collectionView.dataSource = self;
    _collectionView.backgroundColor = [UIColor redColor];
    [self.view addSubview:_collectionView];
    
    // 注册 cell
    [_collectionView registerClass:[SimpleCollectionViewCell class] forCellWithReuseIdentifier:SIMPLECELLIDENTIFIER];
}
```

> 一定要先将 collectionView 的 delegate 和 dataSource 设置为 self 哦

#### 实现代理

实现两个必须的代理，一个返回 `item` 的数量，一个返回具体的 `cell`：

```objc
- (NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section {
    return _count;
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath {
    SimpleCollectionViewCell *cell = (SimpleCollectionViewCell *)[collectionView dequeueReusableCellWithReuseIdentifier:SIMPLECELLIDENTIFIER forIndexPath:indexPath];
    cell.num = [NSString stringWithFormat:@"%ld",(long)indexPath.item];
    [cell refreshData];
    return cell;
}
```

#### 批量处理

可以将这些操作通过 `performBatchUpdates:completion:` 方法进行批量处理：

```objc
- (void)addItem:(id)sender {
    [self.collectionView performBatchUpdates:^{
        [self.collectionView deleteItemsAtIndexPaths:@[[NSIndexPath indexPathForItem:2 inSection:0]]];
        [self.collectionView insertItemsAtIndexPaths:@[[NSIndexPath indexPathForItem:5 inSection:0]]];
    } completion:nil];
}
```



### Cell 的视图层级

在 cell 中添加视图的时候，我们要将视图添加到 `contentView` 上，而不是 `self`。我们可以观察 cell 的视图层级：

![视图层级](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/collectionview_cell.png?raw=true)

在 `contentView` 和 `UICollectionViewCell` 之间还存在着 `selectedBackgroundView` 和 `backgroundView`。一般情况下，这两个 view 都是 nil 的。如果你希望他们能显示出来，那么需要 alloc 出来，不需要手动设置 frame 大小，系统会自己撑满整个 cell：

```objc
self.backgroundView = [UIView alloc] init];
```



##  更进一步

上面一节讲了如何展示一个最基本的 CollectionView，本节要在展示的基础上拓展一些操作方法。



### Supplementary Views

Supplementary View 是 `UICollectionViewLayout` 提供的，可以随着 CollectionView 滚动，并且是数据驱动，可以展示信息的。

在 `UICollectionViewFlowLayout` 中提供了两种 Supplementary View：头部视图和尾部视图。但是这只是在这种 Layout 方式下的特殊类型，之后我们会学习如何自定义 Supplementary View。只要记住，**任何有数据展示的并且不是 cell 的视图，都可以用 Supplementary View 展示**。

Supplementary View 和 cell 类似，也需要注册 `Nib` 或者 `Class`，也会被一直重用。



#### 定义一个 Supplementary View 类

自定义的 Supplementary View 继承于 `UICollectionReusableView`。这个类提供了一些和 `UICollectionViewCell` 相同的方法，但是比其更轻量级，比如无法像 Cell 一样，处理点击以及高亮等。

```objc
@interface SimpleReusableView : UICollectionReusableView

@property (nonatomic,strong) NSString *num;

- (void)refreshData;

@end
```

同样实现以下 `prepareForReuse` 方法：

```objc
- (void)prepareForReuse {
    [super prepareForReuse];
    self.num = @"";
}
```

#### 设置 ViewController

还是在 `viewDidLoad` 方法中注册，同样先定义一个 Identifier：

```objc
static NSString *SIMPLESUPPLEMENTARYIDENTIFIER = @"Simple Supplementary Identifier";
static NSString *SIMPLECELLIDENTIFIER = @"Simple Cell Identifier";

- (void)viewDidLoad {
    [super viewDidLoad];
  	...
      
    // 设置大小
    flowLayout.headerReferenceSize = CGSizeMake(self.view.bounds.size.width, 30);
  
  	// 注册 Supplementary View
  	[_collectionView registerClass:[SimpleReusableView class] forSupplementaryViewOfKind:UICollectionElementKindSectionHeader withReuseIdentifier:SIMPLESUPPLEMENTARYIDENTIFIER];
}
```

这里注册的是 header:`UICollectionElementKindSectionHeader`，类似的还可以注册 footer:`UICollectionElementKindSectionFooter`

另外，一定要设置 `UICollectionViewFlowLayout` 的 `headerReferenceSize` 属性，其默认是 `CGSizeZero` 的，如果不手动设置，Supplementary View 就无法显示。



#### 实现代理：

同样实现两个代理，一个返回 `section` 的数量，一个返回 `Supplementary View` ：

```objc
- (NSInteger)numberOfSectionsInCollectionView:(UICollectionView *)collectionView {
    return _sectionCount;
}

- (UICollectionReusableView *)collectionView:(UICollectionView *)collectionView viewForSupplementaryElementOfKind:(NSString *)kind atIndexPath:(NSIndexPath *)indexPath {
    SimpleReusableView *reuseableView = (SimpleReusableView *)[collectionView dequeueReusableSupplementaryViewOfKind:kind withReuseIdentifier:SIMPLESUPPLEMENTARYIDENTIFIER forIndexPath:indexPath];
    reuseableView.num = [NSString stringWithFormat:@"%ld",(long)indexPath.section];
    [reuseableView refreshData];
    return reuseableView;
}
```



### 响应交互

实现 `CollectionViewDelegate` 的方法 `collectionView:didSelectedAtIndexPath:`

```objc
- (void)collectionView:(UICollectionView *)collectionView didSelectItemAtIndexPath:(NSIndexPath *)indexPath {
    // 取消选中
    [collectionView deselectItemAtIndexPath:indexPath animated:YES];
    // 获取当前点击的 cell
    SimpleCollectionViewCell *cell = (SimpleCollectionViewCell *)[collectionView cellForItemAtIndexPath:indexPath];
	...
}
```

 `deselectItemAtIndexPath:animted:` 方法取消选中。

`cellForItemAtIndexPath:` 方法通过 `indexPath` 获取当前 cell。



### 对 item 和 section 的操作

系统提供了以下方法来对 item 和 section 进行插入、删除、重载、移动操作：

- `insertSections:`
- `deleteSections:`
- `reloadSections:`
- `moveSection:toSection:`
- `insertItemsAtIndexPaths:`
- `deleteItemsAtIndexPaths:`
- `reloadItemsAtIndexPaths:`
- `moveItemAtIndexPath:toIndexPath:`

注意一定要先改变 item 或者 section 的数量，然后才能对 item 或者 section 进行插入和删除。

这些方法可以放在 `performBatchUpdates:completion:` 中进行批量处理。

### 改变 item 和 section 的大小

前面，我们通过 `UICollectionViewFlowLayout`  的 `itemSize` 和 `headerReferenceSize` (包括 `footerReferenceSize`) 统一设置了 item 和 section 的大小。

如何每个 item 和 section 的大小不规则呢？ 我们可以实现 `UICollectionViewDelegateFlowLayout` 协议中的以下方法实现：

- `collectionView:layout:sizeForItemAtIndexPath:`
- `collectionView:layout:referenceSizeForHeaderInSection:`
- `collectionView:layout:referenceSizeForFooterInSection:`

实现这些方法，并在根据 `indexPath` 返回响应的 `CGSize`.

`UICollectionViewDelegateFlowLayout `协议继承于 `UICollectionViewDelegate `协议，所以仍然将 CollectionView 的 `delegate` 设置为 ViewController 就行了。

### 是否可以选中

实现 `UICollectionViewDelegate` 协议中的以下方法可以实现控制 item 是否可以高亮，选中，取消选中：

- `collectionView:shouldHighlightItemAtIndexPath:`
- `collectionView:shouldSelectItemAtIndexPath:`
- `collectionView:shouldDeselectedItemAtIndexPath:`

### 复制粘贴操作

有时候我们想复制一些 cell 中的东西，实现下面方法就可以复制：

```objc
// 能否选中
- (BOOL)collectionView:(UICollectionView *)collectionView shouldSelectItemAtIndexPath:(NSIndexPath *)indexPath {
    return YES;
}

// 选中后能否弹出菜单
- (BOOL)collectionView:(UICollectionView *)collectionView shouldShowMenuForItemAtIndexPath:(NSIndexPath *)indexPath {
    return YES;
}

// 能否执行菜单里的某个操作
- (BOOL)collectionView:(UICollectionView *)collectionView canPerformAction:(SEL)action forItemAtIndexPath:(NSIndexPath *)indexPath withSender:(id)sender {
    return YES;
}

// 棘突执行菜单选项的操作
- (void)collectionView:(UICollectionView *)collectionView performAction:(SEL)action forItemAtIndexPath:(NSIndexPath *)indexPath withSender:(id)sender {
    if ([NSStringFromSelector(action) isEqualToString:@"copy:"]) {
        UIPasteboard *pasteboard = [UIPasteboard generalPasteboard];
        [pasteboard setString:((SimpleCollectionViewCell *)[collectionView cellForItemAtIndexPath:indexPath]).num];
    }
}
```



## 使用 UICollectionViewFlowLayout

### 创建 UICollectionViewFlowLayout 的子类









































