title: UICollectionView 使用方法总结
date: 2016/8/5 14:07:12  
categories: iOS
tags: 
	- 基本控件	

---

UICollectionView最强大、同时显著超出UITableView的特色就是其完全灵活的布局结构。本篇将对UICollectionView的基本使用方法进行总结。

<!--more-->

## 视图
UICollectionView上面显示内容的视图有三种**Cell**视图、**Supplementary View**和**Decoration View**。
- Cell视图
CollectionView中主要的内容都是由它展示的，它是从数据源对象获取的。
- Supplementary View
它展示了每一组当中的信息，与cell类似，它是从数据源方法当中获取的，但是与cell不同的是，它并不是强制需要的。
例如flow layout当中的headers和footers就是可选的Supplementary View。
- Decoration View
这个视图是一个装饰视图，它没有什么功能性，它不跟数据源有任何关系，它完全属于layout对象。

## 注册与重用
### 注册
在使用数据源返回cell或者Supplementary View给collectionView之前，我们必须先要注册。
- registerClass: forCellWithReuseIdentifier:
- registerNib: forCellWithReuseIdentifier:
- registerClass: forSupplementaryViewOfKind: withReuseIdentifier:
- registerNib: forSupplementaryViewOfKind: withReuseIdentifier:

前面两个方法是注册cell，后两个方法注册Supplementary View。注册Supplementary View时需要指定是headerview还是footer。

### 重用
注册后，调用一下方法进行重用:
- dequeueReusableCellWithReuseIdentifier:forIndexPath:
- dequeueReusableSupplementaryViewOfKind:withReuseIdentifier:forIndexPath:

## 数据源方法
### 基本方法
数据源方法与UITableView类似，主要有：
- numberOfSectionsInCollectionView:
- collectionView: numberOfItemsInSection:
- collectionView: cellForItemAtIndexPath:
- collectionView: viewForSupplementaryElementOfKind: atIndexPath:

### 添加头部和尾部视图
collection view 额外管理着两种视图：supplementary views ， Supplementary views 相当于 table view 的 section header 和 footer views。像cells一样，他们的内容都由数据源对象驱动。
```objc
-(UICollectionReusableView *)collectionView:(UICollectionView *)collectionView viewForSupplementaryElementOfKind:(NSString *)kind atIndexPath:(NSIndexPath *)indexPath{
    if ([kind isEqual:UICollectionElementKindSectionFooter] ) {
        MineTicketListReusableView *mineTicketListReusableView = [collectionView dequeueReusableSupplementaryViewOfKind:UICollectionElementKindSectionFooter withReuseIdentifier:MineTicketListReusableViewIdentifier forIndexPath:indexPath];
        mineTicketListReusableView.delegate = self;
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

//collectionView不像tableView可以设置setEditing属性切换是否能够编辑，需要声明长按手势
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
```

## FlowLayout代理方法
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

## UICollectionViewLayout子类
## 基本方法
UICollectionView和UITableView最重要的区别就是UICollectionView并不知道如何布局，它把布局机制委托给了UICollectionViewLayout子类，默认的布局方式是UICollectionFlowViewLayout类提供的流式布局。不过也可以创建自己的布局方式，通过继承UICollectionViewLayout。

子类需要覆盖父类以下3个方法：
- prepareLayout
- layoutAttributesForElementsInRect:(CGRect)rect
- collectionViewContentSize

---
应用场景示例：
为了实现大小不同的cell，可以应用在上文实现UICollectionViewDelegateFlowLayout的协议方法collectionView:layout:sizeForItemAtIndexPath:。但是它会计算每排的最大高度，并不是一个流式视图。
因此就需要继承UICollectionViewLayout来自己实现一个流式布局的效果。

---

### -(void)prepareLayout
初始化参数

### -(CGSize)collectionViewContentSize
布局首先要提供的信息就是滚动区域大小，这样collection view才能正确的管理滚动。布局对象必须在此时计算它内容的总大小，包括supplementary views和decoration views。

### -(NSArray *)layoutAttributesForElementsInRect:(CGRect)rect
实现必须返回一个包含**UICollectionViewLayoutAttributes**对象的数组.其中包括：中心(center)，尺寸(size)，透明度(alpha)，层级(zIndex)，动画效果(transform3D),隐藏(hidden)等。UICollectionViewLayoutAttributes对象决定了cell的摆设位置（frame）。

传入参数rect是一个包含要显示区域的块。这个块的大小一般为2倍的collectionview的长度，只有滑动即将超过rect的范围，rect才会改变数值，保证了显示范围内的所有元素都在rect内。这个参数只是框定了显示范围，自定义layout的时候并不需要用到这个参数，而是使用contentOffset等属性，设置位置信息。

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

具体使用方法以后再写。

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




