title: UICollectionView 使用方法总结
date: 2016/8/5 14:07:12  
categories: iOS
tags: 
	- 基本控件	
---

`UICollectionView` 最强大、同时显著超出 `UITableView` 的特色就是其完全灵活的布局结构。本篇将对 `UICollectionView` 的基本使用方法进行总结。

<!--more-->

包含 `UICollectionView` 的 controller 需要实现三个 delegate：`UICollectionViewDelegate`,`UICollectionViewDataSource`,`UICollectionViewDelegateFlowLayout`

## 视图
`UICollectionView` 上面显示内容的视图有三种**Cell**视图、**Supplementary View**和**Decoration View**。
- `Cell` 视图
  `CollectionView` 中主要的内容都是由它展示的，它是从数据源对象获取的。
- `Supplementary View`
  它展示了每一组当中的信息，与 `cell` 类似，它是从数据源方法当中获取的，但是与 `cell` 不同的是，它并不是强制需要的。
  例如 `flow layout` 当中的 `headers` 和 `footers` 就是可选的 `Supplementary View`。
- `Decoration View`
  这个视图是一个装饰视图，它没有什么功能性，它不跟数据源有任何关系，它完全属于layout对象。

## 注册与重用
### 注册
在使用数据源返回 `cell` 或者 `Supplementary View` 给 `collectionView` 之前，我们必须先要注册。
- registerClass: forCellWithReuseIdentifier:
- registerNib: forCellWithReuseIdentifier:
- registerClass: forSupplementaryViewOfKind: withReuseIdentifier:
- registerNib: forSupplementaryViewOfKind: withReuseIdentifier:

前面两个方法是注册 `cell`，后两个方法注册 `Supplementary View`。注册 `Supplementary View` 时需要指定是`headerview` 还是 `footer`。

### 重用
注册后，调用一下方法进行重用:
- dequeueReusableCellWithReuseIdentifier:forIndexPath:
- dequeueReusableSupplementaryViewOfKind:withReuseIdentifier:forIndexPath:

### 实例

下面列举一个简单的使用实例：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    UICollectionViewFlowLayout * layout = [[UICollectionViewFlowLayout alloc]init];
    //设置布局方向为垂直流布局
    layout.scrollDirection = UICollectionViewScrollDirectionVertical;
    //设置每个item的大小为100*100
    layout.itemSize = CGSizeMake(100, 100);
    //创建collectionView 通过一个布局策略layout来创建
    UICollectionView * collect = [[UICollectionView alloc]initWithFrame:self.view.frame collectionViewLayout:layout];
    //代理设置
    collect.delegate=self;
    collect.dataSource=self;
    //注册item类型 这里使用系统的类型
    [collect registerClass:[UICollectionViewCell class] forCellWithReuseIdentifier:@"cellid"];
    [self.view addSubview:collect];
}


//返回分区个数
-(NSInteger)numberOfSectionsInCollectionView:(UICollectionView *)collectionView{
    return 1;
}
//返回每个分区的item个数
-(NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section{
    return 10;
}
//返回每个item
-(UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath{
    UICollectionViewCell * cell  = [collectionView dequeueReusableCellWithReuseIdentifier:@"cellid" forIndexPath:indexPath];
    cell.backgroundColor = [UIColor colorWithRed:arc4random()%255/255.0 green:arc4random()%255/255.0 blue:arc4random()%255/255.0 alpha:1];
    return cell;
}

```

效果如下：

![效果图](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/collectionView_demo1.png?raw=true)

## 常用方法和属性汇总

```objc
//通过一个布局策略初识化CollectionView
- (instancetype)initWithFrame:(CGRect)frame collectionViewLayout:(UICollectionViewLayout *)layout;

//获取和设置collection的layout
@property (nonatomic, strong) UICollectionViewLayout *collectionViewLayout;

//数据源和代理
@property (nonatomic, weak, nullable) id <UICollectionViewDelegate> delegate;
@property (nonatomic, weak, nullable) id <UICollectionViewDataSource> dataSource;

//从一个class或者xib文件进行cell(item)的注册
- (void)registerClass:(nullable Class)cellClass forCellWithReuseIdentifier:(NSString *)identifier;
- (void)registerNib:(nullable UINib *)nib forCellWithReuseIdentifier:(NSString *)identifier;

//下面两个方法与上面相似，这里注册的是头视图或者尾视图的类
//其中第二个参数是设置 头视图或者尾视图 系统为我们定义好了这两个字符串
//UIKIT_EXTERN NSString *const UICollectionElementKindSectionHeader NS_AVAILABLE_IOS(6_0);
//UIKIT_EXTERN NSString *const UICollectionElementKindSectionFooter NS_AVAILABLE_IOS(6_0);
- (void)registerClass:(nullable Class)viewClass forSupplementaryViewOfKind:(NSString *)elementKind withReuseIdentifier:(NSString *)identifier;
- (void)registerNib:(nullable UINib *)nib forSupplementaryViewOfKind:(NSString *)kind withReuseIdentifier:(NSString *)identifier;

//这两个方法是从复用池中取出cell或者头尾视图
- (__kindof UICollectionViewCell *)dequeueReusableCellWithReuseIdentifier:(NSString *)identifier forIndexPath:(NSIndexPath *)indexPath;
- (__kindof UICollectionReusableView *)dequeueReusableSupplementaryViewOfKind:(NSString *)elementKind withReuseIdentifier:(NSString *)identifier forIndexPath:(NSIndexPath *)indexPath;

//设置是否允许选中 默认yes
@property (nonatomic) BOOL allowsSelection;

//设置是否允许多选 默认no
@property (nonatomic) BOOL allowsMultipleSelection;

//获取所有选中的item的位置信息
- (nullable NSArray<NSIndexPath *> *)indexPathsForSelectedItems; 

//设置选中某一item，并使视图滑动到相应位置，scrollPosition是滑动位置的相关参数，如下：
/*
typedef NS_OPTIONS(NSUInteger, UICollectionViewScrollPosition) {
    //无
    UICollectionViewScrollPositionNone                 = 0,
    //垂直布局时使用的 对应上中下
    UICollectionViewScrollPositionTop                  = 1 << 0,
    UICollectionViewScrollPositionCenteredVertically   = 1 << 1,
    UICollectionViewScrollPositionBottom               = 1 << 2,
    //水平布局时使用的  对应左中右
    UICollectionViewScrollPositionLeft                 = 1 << 3,
    UICollectionViewScrollPositionCenteredHorizontally = 1 << 4,
    UICollectionViewScrollPositionRight                = 1 << 5
};
*/
- (void)selectItemAtIndexPath:(nullable NSIndexPath *)indexPath animated:(BOOL)animated scrollPosition:(UICollectionViewScrollPosition)scrollPosition;

//将某一item取消选中
- (void)deselectItemAtIndexPath:(NSIndexPath *)indexPath animated:(BOOL)animated;

//重新加载数据
- (void)reloadData;

//下面这两个方法，可以重新设置collection的布局，后面的方法多了一个布局完成后的回调，iOS7后可以用
//使用这两个方法可以产生非常炫酷的动画效果
- (void)setCollectionViewLayout:(UICollectionViewLayout *)layout animated:(BOOL)animated;
- (void)setCollectionViewLayout:(UICollectionViewLayout *)layout animated:(BOOL)animated completion:(void (^ __nullable)(BOOL finished))completion NS_AVAILABLE_IOS(7_0);

//下面这些方法更加强大，我们可以对布局更改后的动画进行设置
//这个方法传入一个布局策略layout，系统会开始进行布局渲染，返回一个UICollectionViewTransitionLayout对象
//这个UICollectionViewTransitionLayout对象管理动画的相关属性，我们可以进行设置
- (UICollectionViewTransitionLayout *)startInteractiveTransitionToCollectionViewLayout:(UICollectionViewLayout *)layout completion:(nullable UICollectionViewLayoutInteractiveTransitionCompletion)completion NS_AVAILABLE_IOS(7_0);
//准备好动画设置后，我们需要调用下面的方法进行布局动画的展示，之后会调用上面方法的block回调
- (void)finishInteractiveTransition NS_AVAILABLE_IOS(7_0);
//调用这个方法取消上面的布局动画设置，之后也会进行上面方法的block回调
- (void)cancelInteractiveTransition NS_AVAILABLE_IOS(7_0);

//获取分区数
- (NSInteger)numberOfSections;

//获取某一分区的item数
- (NSInteger)numberOfItemsInSection:(NSInteger)section;

//下面两个方法获取item或者头尾视图的layout属性，这个UICollectionViewLayoutAttributes对象
- (nullable UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath;
- (nullable UICollectionViewLayoutAttributes *)layoutAttributesForSupplementaryElementOfKind:(NSString *)kind atIndexPath:(NSIndexPath *)indexPath;

//获取某一点所在的indexpath位置
- (nullable NSIndexPath *)indexPathForItemAtPoint:(CGPoint)point;

//获取某个cell所在的indexPath
- (nullable NSIndexPath *)indexPathForCell:(UICollectionViewCell *)cell;

//根据indexPath获取cell
- (nullable UICollectionViewCell *)cellForItemAtIndexPath:(NSIndexPath *)indexPath;

//获取所有可见cell的数组
- (NSArray<__kindof UICollectionViewCell *> *)visibleCells;

//获取所有可见cell的位置数组
- (NSArray<NSIndexPath *> *)indexPathsForVisibleItems;

//下面三个方法是iOS9中新添加的方法，用于获取头尾视图
- (UICollectionReusableView *)supplementaryViewForElementKind:(NSString *)elementKind atIndexPath:(NSIndexPath *)indexPath NS_AVAILABLE_IOS(9_0);
- (NSArray<UICollectionReusableView *> *)visibleSupplementaryViewsOfKind:(NSString *)elementKind NS_AVAILABLE_IOS(9_0);
- (NSArray<NSIndexPath *> *)indexPathsForVisibleSupplementaryElementsOfKind:(NSString *)elementKind NS_AVAILABLE_IOS(9_0);

//使视图滑动到某一位置，可以带动画效果
- (void)scrollToItemAtIndexPath:(NSIndexPath *)indexPath atScrollPosition:(UICollectionViewScrollPosition)scrollPosition animated:(BOOL)animated;

//下面这些方法用于动态添加，删除，移动某些分区获取items
- (void)insertSections:(NSIndexSet *)sections;
- (void)deleteSections:(NSIndexSet *)sections;
- (void)reloadSections:(NSIndexSet *)sections;
- (void)moveSection:(NSInteger)section toSection:(NSInteger)newSection;

- (void)insertItemsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths;
- (void)deleteItemsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths;
- (void)reloadItemsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths;
- (void)moveItemAtIndexPath:(NSIndexPath *)indexPath toIndexPath:(NSIndexPath *)newIndexPath;
```



## 数据源方法
### 基本方法
数据源方法与 `UITableView` 类似，主要有：
- numberOfSectionsInCollectionView:
- collectionView: numberOfItemsInSection:
- collectionView: cellForItemAtIndexPath:
- collectionView: viewForSupplementaryElementOfKind: atIndexPath:

### 添加头部和尾部视图
collection view 额外管理着两种视图：supplementary views ， Supplementary views 相当于 table view 的 section header 和 footer views。像cells一样，他们的内容都由数据源对象驱动。
```objc
static NSString *MineTicketListReusableViewIdentifier = @"MineTicketListReusableViewIdentifier";

-(UICollectionReusableView *)collectionView:(UICollectionView *)collectionView viewForSupplementaryElementOfKind:(NSString *)kind atIndexPath:(NSIndexPath *)indexPath{
    if ([kind isEqual:UICollectionElementKindSectionFooter] ) {
      // 这里的 MineTicketListReusableView 就是一个普通的 view
        MineTicketListReusableView *mineTicketListReusableView = [collectionView dequeueReusableSupplementaryViewOfKind:UICollectionElementKindSectionFooter withReuseIdentifier:MineTicketListReusableViewIdentifier forIndexPath:indexPath];
        return mineTicketListReusableView;
    }else{
    	return nil;
    }
}
```

## 部分代理方法
###  移动cell
```objc
//返回YES允许其item移动
- (BOOL)collectionView:(UICollectionView *)collectionView canMoveItemAtIndexPath:(NSIndexPath *)indexPath{
    return YES;
}

//移动item时回调
- (void)collectionView:(UICollectionView *)collectionView moveItemAtIndexPath:(NSIndexPath *)sourceIndexPath toIndexPath:(NSIndexPath*)destinationIndexPath {
}

//collectionView不像tableView可以设置setEditing属性切换是否能够编辑，需要声明长按手势在 viewDidload 方法中
UILongPressGestureRecognizer *longGesture = [[UILongPressGestureRecognizer alloc] initWithTarget:self action:@selector(handlelongGesture:)];
    [self.collectionView addGestureRecognizer:longGesture];
    
//再实现手势操作
- (void)handlelongGesture:(UILongPressGestureRecognizer *)longGesture {
    //判断手势状态
    switch (longGesture.state) {
        case UIGestureRecognizerStateBegan:{
            //判断手势落点位置是否在路径上
            NSIndexPath *indexPath = [self.collectionView indexPathForItemAtPoint:[longGesture locationInView:self.collectionView]];
            if (indexPath == nil) {
                break;
            }
            //在路径上则开始移动该路径上的cell
            [self.collectionView beginInteractiveMovementForItemAtIndexPath:indexPath];
        }
            break;
        case UIGestureRecognizerStateChanged:
            //移动过程当中随时更新cell位置
            [self.collectionView updateInteractiveMovementTargetPosition:[longGesture locationInView:self.collectionView]];
            break;
        case UIGestureRecognizerStateEnded:
            //移动结束后关闭cell移动
            [self.collectionView endInteractiveMovement];
            break;
        default:
            [self.collectionView cancelInteractiveMovement];
            break;
    }
}
```

### 点击cell高亮

高亮就是一般选中的时候会放大什么的标记效果

```objc
// 允许选中时，高亮
-(BOOL)collectionView:(UICollectionView *)collectionView shouldHighlightItemAtIndexPath:(NSIndexPath *)indexPath{
    return YES;
}

// 高亮完成后回调  
// 放大缩小效果
-(void)collectionView:(UICollectionView *)collectionView didHighlightItemAtIndexPath:(NSIndexPath *)indexPath{
    UICollectionViewCell *selectedCell = [collectionView cellForItemAtIndexPath:indexPath];
    [UIView animateWithDuration:kAnimationDuration animations:^{
        selectedCell.transform = CGAffineTransformMakeScale(2.0f, 2.0f);
    }];
}

// 由高亮转成非高亮完成时的回调  
-(void)collectionView:(UICollectionView *)collectionView didUnhighlightItemAtIndexPath:(NSIndexPath *)indexPath{
    UICollectionViewCell *selectedCell = [collectionView cellForItemAtIndexPath:indexPath];
    [UIView animateWithDuration:kAnimationDuration animations:^{
        selectedCell.transform = CGAffineTransformMakeScale(1.0f, 1.0f);
    }];
}
```

### 点击cell选中
```objc
//设置Cell多选，如果不是yes点击其他的上一个就会被取消选中
self.collectionView.allowsMultipleSelection = YES;

// 设置是否允许选中  
- (BOOL)collectionView:(UICollectionView *)collectionView shouldSelectItemAtIndexPath:(NSIndexPath *)indexPath {  
  NSLog(@"%s", __FUNCTION__);  
  return YES;  
}  
  
// 设置是否允许取消选中  
- (BOOL)collectionView:(UICollectionView *)collectionView shouldDeselectItemAtIndexPath:(NSIndexPath *)indexPath {  
  NSLog(@"%s", __FUNCTION__);  
  return YES;  
}  
  
// 选中操作  
- (void)collectionView:(UICollectionView *)collectionView didSelectItemAtIndexPath:(NSIndexPath *)indexPath {  
  NSLog(@"%s", __FUNCTION__);  
}  
  
// 取消选中操作  
- (void)collectionView:(UICollectionView *)collectionView didDeselectItemAtIndexPath:(NSIndexPath *)indexPath {  
  NSLog(@"%s", __FUNCTION__);  
}  

//下面两个方法是主动调用的
- (void)selectItemAtIndexPath:(nullable NSIndexPath *)indexPath animated:(BOOL)animated scrollPosition:(UICollectionViewScrollPosition)scrollPosition;

- (void)deselectItemAtIndexPath:(NSIndexPath *)indexPath animated:(BOOL)animated;
```

这里要几点：

- 对于选中操作，由于 `collectionView` 有复用机制，所以不能只在上面的选中方法对 `cell` 设置一下就不管了，需要在 `dataSource` 中设置一个选中的字段，每次 `cell` 显示的时候都要通过判断显示。
- 如果要实现那种一键全选的效果，除了要上面一条的基础上还要把当前显示的 cell 都设置为已选状态。可以通过 `NSArray *visiblePaths = [self.collectionView indexPathsForVisibleItems];` 方法，拿到当前显示的所有 `indexPath` 然后手动调用 `[self collectionView:self.collectionView didSelectItemAtIndexPath:indexPath];` 方法。
- 上面提到的 `selectItemAtIndexPath:animated:scrollPosition:` 方法不会触发与选择相关的代理方法，即不会触发 `collectionView:didSelectItemAtIndexPath:`。所以除了能滚动外，好像没啥用啊。

## FlowLayout代理方法

### 设置布局方向

- @property(nonatomic) UICollectionViewScrollDirection scrollDirection;

### 设置每个item大小
- @property (CGSize)itemSize
- -collectionView:ayout:sizeForItemAtIndexPath:

可以设置全局属性也可以对某个cell定制尺寸。


### item间隔
- @property (CGSize) minimumInteritemSpacing
- @property (CGSize) minimumLineSpacing
- -collectionView:layout:minimumInteritemSpacingForSectionAtIndex:
- -collectionView:layout:minimumLineSpacingForSectionAtIndex:

同itemsize，可以设置全局属性也可以定制。

### 缩进（padding）
- @property UIEdgeInsets sectionInset;
- -collectionView:layout:insetForSectionAtIndex:

同上。

### 设置headerview的layout
- @property (CGSize) headerReferenceSize
- @property (CGSize) footerReferenceSize
- -collectionView:layout:referenceSizeForHeaderInSection:
- -collectionView:layout:referenceSizeForFooterInSection:

同上。

但是上面的设置仅适用于图形规则的情况，不能实现例如流式布局等酷炫的布局情况，需要自定义 `UICollectionViewLayout` 的子类。

## UICollectionViewLayout子类
###  基本方法

`UICollectionView` 和 `UITableView` 最重要的区别就是 `UICollectionView` 并不知道如何布局，它把布局机制委托给了 `UICollectionViewLayout` 子类，默认的布局方式是 `UICollectionFlowViewLayout` 类提供的流式布局。不过也可以创建自己的布局方式，通过**继承 `UICollectionViewLayout`**。

子类需要覆盖父类以下3个方法：
- prepareLayout
- layoutAttributesForElementsInRect:(CGRect)rect
- collectionViewContentSize

### - (void)prepareLayout

初始化参数,只会调动一次，可以设置每个块的属性，可设置的属性包括：

```objc
//配置item的布局位置
@property (nonatomic) CGRect frame;
//配置item的中心
@property (nonatomic) CGPoint center;
//配置item的尺寸
@property (nonatomic) CGSize size;
//配置item的3D效果
@property (nonatomic) CATransform3D transform3D;
//配置item的bounds
@property (nonatomic) CGRect bounds NS_AVAILABLE_IOS(7_0);
//配置item的旋转
@property (nonatomic) CGAffineTransform transform NS_AVAILABLE_IOS(7_0);
//配置item的alpha
@property (nonatomic) CGFloat alpha;
//配置item的z坐标
@property (nonatomic) NSInteger zIndex; // default is 0
//配置item的隐藏
@property (nonatomic, getter=isHidden) BOOL hidden; 
//item的indexpath
@property (nonatomic, strong) NSIndexPath *indexPath;
//获取item的类型
@property (nonatomic, readonly) UICollectionElementCategory representedElementCategory;
@property (nonatomic, readonly, nullable) NSString *representedElementKind; 

//一些创建方法
+ (instancetype)layoutAttributesForCellWithIndexPath:(NSIndexPath *)indexPath;
+ (instancetype)layoutAttributesForSupplementaryViewOfKind:(NSString *)elementKind withIndexPath:(NSIndexPath *)indexPath;
+ (instancetype)layoutAttributesForDecorationViewOfKind:(NSString *)decorationViewKind withIndexPath:(NSIndexPath *)indexPath;
```

### - (CGSize)collectionViewContentSize
布局首先要提供的信息就是滚动区域大小，这样collection view才能正确的管理滚动。布局对象必须在此时计算它内容的总大小，包括supplementary views和decoration views。

### - (NSArray *)layoutAttributesForElementsInRect:(CGRect)rect
实现必须返回一个包含**UICollectionViewLayoutAttributes**对象的数组.其中包括：中心(center)，尺寸(size)，透明度(alpha)，层级(zIndex)，动画效果(transform3D),隐藏(hidden)等。UICollectionViewLayoutAttributes对象决定了cell的摆设位置（frame）。

传入参数rect是一个包含要显示区域的块。这个块的大小一般为2倍的collectionview的长度，只有滑动即将超过rect的范围，rect才会改变数值，保证了显示范围内的所有元素都在rect内。这个参数只是框定了显示范围，自定义layout的时候并不需要用到这个参数，而是使用contentOffset等属性，设置位置信息。

## 一个流式布局实例

```objc
@interface MyLayout : UICollectionViewFlowLayout
@property(nonatomic,assign)int itemCount;
@end
  
@interface MyLayout()
@property (nonatomic,assign) CGFloat maxHeight;
@end
@implementation MyLayout
{
    //这个数组就是我们自定义的布局配置数组
    NSMutableArray * _attributeAttay;
}
//数组的相关设置在这个方法中
//布局前的准备会调用这个方法
-(void)prepareLayout{
    _attributeAttay = [[NSMutableArray alloc]init];
    [super prepareLayout];
    //演示方便 我们设置为静态的2列
    //计算每一个item的宽度
    float WIDTH = ([UIScreen mainScreen].bounds.size.width-self.sectionInset.left-self.sectionInset.right-self.minimumInteritemSpacing)/2;
    //定义数组保存每一列的高度
    //这个数组的主要作用是保存每一列的总高度，这样在布局时，我们可以始终将下一个Item放在最短的列下面
    CGFloat colHight[2]={self.sectionInset.top,self.sectionInset.bottom};
    //itemCount是外界传进来的item的个数 遍历来设置每一个item的布局
    for (int i=0; i<_itemCount; i++) {
        //设置每个item的位置等相关属性
        NSIndexPath *index = [NSIndexPath indexPathForItem:i inSection:0];
        //创建一个布局属性类，通过indexPath来创建
        UICollectionViewLayoutAttributes * attris = [UICollectionViewLayoutAttributes layoutAttributesForCellWithIndexPath:index];
        //随机一个高度 在40——190之间
        CGFloat hight = arc4random()%150+40;
        //哪一列高度小 则放到那一列下面
        //标记最短的列
        int width=0;
        if (colHight[0]<colHight[1]) {
            //将新的item高度加入到短的一列
            colHight[0] = colHight[0]+hight+self.minimumLineSpacing;
            width=0;
        }else{
            colHight[1] = colHight[1]+hight+self.minimumLineSpacing;
            width=1;
        }
        
        //设置item的位置
        attris.frame = CGRectMake(self.sectionInset.left+(self.minimumInteritemSpacing+WIDTH)*width, colHight[width]-hight-self.minimumLineSpacing, WIDTH, hight);
        [_attributeAttay addObject:attris];
        
    }
    if (colHight[0]>colHight[1]) {
        _maxHeight = colHight[0];
    }else{
        _maxHeight = colHight[1];
    }
    
}

-(CGSize)collectionViewContentSize{
    return CGSizeMake(100, _maxHeight);
}
//这个方法中返回我们的布局数组
-(NSArray<UICollectionViewLayoutAttributes *> *)layoutAttributesForElementsInRect:(CGRect)rect{
    return _attributeAttay;
}
@end
```

## 其他设置
### 对齐 
- **-(CGPoint)targetContentOffsetForProposedContentOffset:withScrollingVelocity：**
  当手放开时调用，返回值以及参数proposedContentOffset决定了collectionview停止滚动时的偏移量。velocity是滚动速率，有x和y两个分量，正表示向右或向下运动。
  每次移动整数个view的长度，示例代码：
```objc
- (CGPoint)targetContentOffsetForProposedContentOffset:(CGPoint)proposedContentOffset withScrollingVelocity:(CGPoint)velocity {
    CGFloat index = roundf((self.scrollDirection == UICollectionViewScrollDirectionVertical ? proposedContentOffset.y : proposedContentOffset.x)/ _itemHeight);
    if (self.scrollDirection == UICollectionViewScrollDirectionVertical) {
        proposedContentOffset.y = _itemHeight * index;
    } else {
        proposedContentOffset.x = _itemHeight * index;
    }
    return proposedContentOffset;
}
```

### 对item缩放翻转
在layoutAttributesForElementsInRect:(CGRect)rect方法中设置**transform3D**的值。transform3D是一个4*4的矩阵，用来控制view的形状。其中：
- m11主要负责x轴缩放
- m22主要负责y轴缩放
- m33主要负责z轴缩放
- 其余值综合控制旋转与翻折

具体使用方法在 [CALayer的transform](https://zhang759740844.github.io/2016/09/02/CALayer的transform属性/)有说明。

### 是否刷新布局
```objc
-(BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds
{
    return !CGRectEqualToRect(newBounds, self.collectionView.bounds);
}
```
划出范围就会调用该方法，控制是否刷新视图。如果返回no，那么久不会再调用prepareLayout以及layoutAttributesForElementsInRect等方法。

### 删除item
```
-(void)collectionView:(UICollectionView *)collectionView didSelectItemAtIndexPath:(NSIndexPath *)indexPath
{
    [self.dataArr removeObjectAtIndex:indexPath.item];
    //TODO:  这个方法 特别注意 删除item的方法
    [self.myCollectionView deleteItemsAtIndexPaths:@[indexPath]];
}
```



## 实例：圆环布局

布局代码：

```objc
@implementation MyLayout
{
    NSMutableArray * _attributeAttay;
}
-(void)prepareLayout{
    [super prepareLayout];
    //获取item的个数
    _itemCount = (int)[self.collectionView numberOfItemsInSection:0];
    _attributeAttay = [[NSMutableArray alloc]init];
    //先设定大圆的半径 取长和宽最短的
    CGFloat radius = MIN(self.collectionView.frame.size.width, self.collectionView.frame.size.height)/2;
    //计算圆心位置
    CGPoint center = CGPointMake(self.collectionView.frame.size.width/2, self.collectionView.frame.size.height/2);
    //设置每个item的大小为50*50 则半径为25
    for (int i=0; i<_itemCount; i++) {
        UICollectionViewLayoutAttributes * attris = [UICollectionViewLayoutAttributes layoutAttributesForCellWithIndexPath:[NSIndexPath indexPathForItem:i inSection:0]];
        //设置item大小
        attris.size = CGSizeMake(50, 50);
        //计算每个item的圆心位置
        /*
         .
         . .
         .   . r
         .     .
         .........
         */
        //计算每个item中心的坐标
        //算出的x y值还要减去item自身的半径大小
        float x = center.x+cosf(2*M_PI/_itemCount*i)*(radius-25);
        float y = center.y+sinf(2*M_PI/_itemCount*i)*(radius-25);
     
        attris.center = CGPointMake(x, y);
        [_attributeAttay addObject:attris];
    }
}
//设置内容区域的大小
-(CGSize)collectionViewContentSize{
    return self.collectionView.frame.size;
}
//返回设置数组
-(NSArray<UICollectionViewLayoutAttributes *> *)layoutAttributesForElementsInRect:(CGRect)rect{
    return _attributeAttay;
}
```

Controller 代码：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    MyLayout * layout = [[MyLayout alloc]init];
     UICollectionView * collect  = [[UICollectionView alloc]initWithFrame:CGRectMake(0, 0, 320, 400) collectionViewLayout:layout];
    collect.delegate=self;
    collect.dataSource=self;
    
    [collect registerClass:[UICollectionViewCell class] forCellWithReuseIdentifier:@"cellid"];
    [self.view addSubview:collect];
}

-(NSInteger)numberOfSectionsInCollectionView:(UICollectionView *)collectionView{
    return 1;
}
-(NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section{
    return 10;
}
-(UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath{
    UICollectionViewCell * cell  = [collectionView dequeueReusableCellWithReuseIdentifier:@"cellid" forIndexPath:indexPath];
    cell.layer.masksToBounds = YES;
    cell.layer.cornerRadius = 25;
    cell.backgroundColor = [UIColor colorWithRed:arc4random()%255/255.0 green:arc4random()%255/255.0 blue:arc4random()%255/255.0 alpha:1];
    return cell;
}
```

这样就能实现一个圆环，随着 item 的增多和减少，布局会自动调整。

![圆环效果图](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/collectionView_demo2.png?raw=true)



## 实例：滚轮控件布局

之前的示例中我们的布局都是静态的，即在初始状态内设置完布局属性就不会再变了。现在要实现的这个布局会根据滚动的状态动态改变每项的属性。实现一个类似于日期选择页的滚轮控件布局。所以这次，我们采用动态配置的方式，在`layoutAttributesForItemAtIndexPath` 方法中进行每个 item 的布局属性设置。

### 显示

viewController 的代码如下：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    MyLayout * layout = [[MyLayout alloc]init];
    UICollectionView * collect  = [[UICollectionView alloc]initWithFrame:CGRectMake(0, 0, 320, 400) collectionViewLayout:layout];
    collect.delegate=self;
    collect.dataSource=self;
    [collect registerClass:[UICollectionViewCell class] forCellWithReuseIdentifier:@"cellid"];
    [self.view addSubview:collect];
}

-(NSInteger)numberOfSectionsInCollectionView:(UICollectionView *)collectionView{
    return 1;
}
-(NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section{
    return 10;
}
-(UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath{
    UICollectionViewCell * cell  = [collectionView dequeueReusableCellWithReuseIdentifier:@"cellid" forIndexPath:indexPath];
    cell.backgroundColor = [UIColor colorWithRed:arc4random()%255/255.0 green:arc4random()%255/255.0 blue:arc4random()%255/255.0 alpha:1];
    UILabel * label = [[UILabel alloc]initWithFrame:CGRectMake(0, 0, 250, 80)];
    label.text = [NSString stringWithFormat:@"我是第%ld行",(long)indexPath.row];
    [cell.contentView addSubview:label];
    return cell;
}
```

上面我创建了10个Item，并且在每个Item上添加了一个标签，标写是第几行。

在我们自定义的布局类中重写 `layoutAttributesForElementsInRect`，在其中返回我们的布局数组：

```objc
-(NSArray<UICollectionViewLayoutAttributes *> *)layoutAttributesForElementsInRect:(CGRect)rect{
    NSMutableArray * attributes = [[NSMutableArray alloc]init];
    //遍历设置每个item的布局属性
    for (int i=0; i<[self.collectionView numberOfItemsInSection:0]; i++) {
        [attributes addObject:[self layoutAttributesForItemAtIndexPath:[NSIndexPath indexPathForItem:i inSection:0]]];
    }
    return attributes;
}
```

之后，在我们布局类中重写 `layoutAttributesForItemAtIndexPath` 方法：

```objc
-(UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath{
    //创建一个item布局属性类
    UICollectionViewLayoutAttributes * atti = [UICollectionViewLayoutAttributes layoutAttributesForCellWithIndexPath:indexPath];
    //获取item的个数
    int itemCounts = (int)[self.collectionView numberOfItemsInSection:0];
    //设置每个item的大小为260*100
    atti.size = CGSizeMake(260, 100);
    //设置中心
    atti.center = CGPointMake(self.collectionView.frame.size.width/2, self.collectionView.frame.size.height/2+self.collectionView.contentOffset.y);

    //创建一个transform3D类
    //CATransform3D是一个类似矩阵的结构体
    //CATransform3DIdentity创建空得矩阵
    CATransform3D trans3D = CATransform3DIdentity;
    //这个值设置的是透视度，影响视觉离投影平面的距离
    trans3D.m34 = -1/900.0;
    //这个是3D滚轮的半径
    CGFloat radius = 50/tanf(M_PI*2/itemCounts/2);
    //获取当前的偏移量
    float offset = self.collectionView.contentOffset.y;
    //在角度设置上，添加一个偏移角度
    float angleOffset = offset/self.collectionView.frame.size.height;
    //计算每个item应该旋转的角度
    CGFloat angle = (float)(indexPath.row+angleOffset)/itemCounts*M_PI*2;
    //这个方法返回一个新的CATransform3D对象，在原来的基础上进行旋转效果的追加
    //第一个参数为旋转的弧度，后三个分别对应x，y，z轴，我们需要以x轴进行旋转
    trans3D = CATransform3DRotate(trans3D, angle, 1.0, 0, 0);
    //这个方法也返回一个transform3D对象，追加平移效果，后面三个参数，对应平移的x，y，z轴，我们沿z轴平移
    trans3D = CATransform3DTranslate(trans3D, 0, 0, radius);
    atti.transform3D = trans3D;
    
    return atti;
}
```

还差一步，实现 `shouldInvalidateLayoutForBoundsChange:` 方法：

```objc
//返回yes，则一有变化就会刷新布局
-(BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds{
    return YES;   
}
```

这样一个滚轮就显示出来了。

现在来解释一下，首先，设置中心的时候要加上 `contentOffset`，否则图像就不能显示在屏幕的正中心。其次，设置旋转角度的时候也要加上一个与 `contentOffset` 有关的值，否则不能滚动，这个与 `contentOffset` 有关的值可大可小，控制着旋转的速度。这个时候的样子是这样的：

![滚轮效果图](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/collectionView_demo4.png?raw=true)

现在就是最后一步 `trans3D = CATransform3DTranslate(trans3D, 0, 0, radius);` 了。这个操作可以把挤在一起的各个 item 沿着 z 轴，也就是垂直于当前面的轴平移。平移的距离通过三角函数确定，适当的时候就能连接在一起。

最后再提醒一下，`collectionViewContentSize:` 的设置也很重要。它和上面的旋转角度共同决定了滚动到底，能滚动几圈。

效果图如下：

![滚轮效果图](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/collectionView_demo3.png?raw=true)

### 无限滚动

如何能让这个滚轮无限滚动？就是要让滚轮在即将滚动到极限的时候，立刻改变 collectionView 的 `contentOffset`。但是这个补偿不是随便脑补出来的，需要计算，否则无法实现无缝对接。回到之前设置偏移角度的代码：

```objc
    //根据偏移量 改变角度
    float offset = self.collectionView.contentOffset.y;
    float angleOffset = offset/self.collectionView.frame.size.height;
    CGFloat angle = (float)(indexPath.row+angleOffset-1)/itemCounts*M_PI*2;
```

实现无缝对接就是要在改变 `contentOffset` 后，`angle` 还是整数个 2π。由于 `itemCounts` 为 10，`collectionView.frame.size.height` 为 400，所以就要设置偏移量为 4000。

因为在 viewController 中我们已经设置了 `collect.delegate=self;`，而且 `UICollectionView` 又是 `UIScrollView` 的子类。所以直接在 viewController 中添加代理方法：

```objc
-(void)scrollViewDidScroll:(UIScrollView *)scrollView{
    if (scrollView.contentOffset.y<200) {
        scrollView.contentOffset = CGPointMake(0, scrollView.contentOffset.y+10*400);
    }else if(scrollView.contentOffset.y>11*400){
        scrollView.contentOffset = CGPointMake(0, scrollView.contentOffset.y-10*400);
    }
}
```

同理，每次转动一个 item 需要移动 400 的距离。可以设置偏移量，来控制最前面显示的 item 是谁。比如要在显示的时候将下一个 item 放置在最前面，那么改动代码：

```objc
//一开始将collectionView的偏移量设置为1屏的偏移量
collect.contentOffset = CGPointMake(0, 400);
//将计算的具体item角度向前递推一个
CGFloat angle = (float)(indexPath.row+angleOffset-1)/itemCounts*M_PI*2;
```

这样一个无限滚轮就做出来了。

[参考文章:iOS流布局UICollectionView系列](https://my.oschina.net/u/2340880/blog/523341)

