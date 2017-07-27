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

## 用 UICollectionView 展示内容

### UIScrollView 概述

UICollectionView 继承于 UIScrollView，UICollectionViewDelegate 继承于 UIScrollViewDelegate。所以在使用 UICollectionView 的时候，可以直接使用 UIScrollView 的各个属性方法。

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

```objc
@interface SimpleCollectionViewCell : UICollectionViewCell

@property (nonatomic,copy) NSString *num;

- (void)refreshData;

@end
```

UICollectionViewCell 提供了一个重用前清空 cell 内数据的方法 `prepareForReuse`，重写它：

```objc
- (void)prepareForReuse {
    [super prepareForReuse];
    self.num = @"";
}
```



#### 在 VC 中添加 UICollectionView

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



#### 实现代理

实现两个必须的代理：

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

通过 `performBatchUpdates:completion:` 方法可以批量增删：

```objc
- (void)addItem:(id)sender {
    [self.collectionView performBatchUpdates:^{
        [self.collectionView deleteItemsAtIndexPaths:@[[NSIndexPath indexPathForItem:2 inSection:0]]];
        [self.collectionView insertItemsAtIndexPaths:@[[NSIndexPath indexPathForItem:5 inSection:0]]];
    } completion:nil];
}
```



#### Cell 的视图层级

在 cell 中添加视图的时候，我们要将视图添加到 `contentView` 上，而不是 `self`。我们可以观察 cell 的视图层级：

![视图层级](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/collectionview_cell.png?raw=true)

在 `contentView` 和 `UICollectionViewCell` 之间还存在着 `selectedBackgroundView` 和 `backgroundView`。一般情况下，这两个 view 都是 nil 的。如果你希望他们能显示出来，那么需要 alloc 出来，不需要手动设置 frame 大小，系统会自己撑满整个 cell：

```objc
self.backgroundView = [UIView alloc] init];
```









































