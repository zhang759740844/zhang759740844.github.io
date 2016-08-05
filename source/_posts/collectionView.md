title: UICollectionView 使用方法总结
date: 2016/8/5 14:07:12  
categories: IOS
tags: [UICollectionView]

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
collection view 额外管理着两种视图：supplementary views ， Supplementary views 相当于 table view 的 section header 和 footer views。像cells一样，他们的内容都由数据源对象驱动。然而和 table view 中用法不一样，supplementary view 并不一定会作为 header 或 footer view；他们的数量和放置的位置完全由布局控制。
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

### 设置高度
```objc
-(CGSize)collectionView:(UICollectionView *)collectionView layout:(UICollectionViewLayout *)collectionViewLayout sizeForItemAtIndexPath:(NSIndexPath *)indexPath{
    return CGSizeMake(APP_WIDTH, 130);
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



## UICollectionViewLayout子类
UICollectionView和UITableView最重要的区别就是UICollectionView并不知道如何布局，它把布局机制委托给了UICollectionViewLayout子类，默认的布局方式是UICollectionFlowViewLayout类提供的流式布局。不过也可以创建自己的布局方式，通过继承UICollectionViewLayout。







