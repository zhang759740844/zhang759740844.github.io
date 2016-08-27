title: tableview自适应高度
date: 2016/8/26 14:07:12  
categories: IOS
tags: [tableview]

---

由于tableview的cell的高度会随着内部label以及textview字符长度的变化而变化，因此不能一味的设置heightForRowAtIndexPath为定值。

<!--more-->

## 设置布局约束
在UITableViewCell子类中，添加约束，使子视图的边缘与contentView的边缘固定。确保每个子视图在垂直方向上的内容压缩阻力(compression resistance)和内容吸附性约束(content hugging constraints)没有被你添加的更高优先级的约束覆盖，以使得这些子视图的固有内容尺寸(intrinsic content size)来推动contentView的高度。

记住，重点是cell的子视图与contentView要有垂直方向上的连结，让它们能够对contentView“施加压力”，使contentView扩张以适合它们的尺寸。

下面用一个带有一些子视图的cell作为示例，展示了一些必要的约束（没有展示全部的约束）：

![布局示例](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/tableviewautolayout1.png?raw=true)

可以想象，当更多的文本被添加到“Multi-line body”那个label上面后，它就需要垂直地增高以适应文本，这实际上将强迫cell增加高度。

**需要注意的是：**
1. 如果没有设置好约束，比如:缺失了“Multi-line body”那个label到底部的约束，那么高度极有可能变成0；
2. label要显示多行，需要将其`Lines`设置为0。
3. 题外话，如果不是在cell里的label要设置高度包含文字的话，可以少设置一个约束，这样布局的时候会自动撑满整个label。

## cell的自适应
### label的ios8实现
在iOS8上，苹果将许多在之前你比较难实现的东西都内置实现了。为了让cell实现自适应（self-sizing），必须先将tableView的`rowHeight`属性设置为常量`UITableViewAutomaticDimension`。然后，只需将tableView的`estimatedRowHeight`属性设置为一个非零值即可开启行高估算功能,如：
```objc
self.tableView.rowHeight = UITableViewAutomaticDimension;
self.tableView.estimatedRowHeight = 44.0; // 设置为一个接近于行高“平均值”的数值
```

这样就为tableView提供了一个还没有被显示在屏幕上的cell的临时估算的行高。当cell即将滚入屏幕范围内的时候，会计算出真实的高度。为了确定每一行的实际高度，tableView会自动让每个cell基于其contentView的已知固定宽度（tableView的宽度减去其他额外的，像section index或accessoryView这些宽度）和被添加到contentView及其子视图上的布局约束来计算`contentView`的高度。真实的行高被计算出来之后，旧的估算的行高会被更新为这个真实的行高（并且其他任何需要对tableView的contentSize或contentOffset的更改都自动替你完成）。

一般来说，行高的估算值不需要太精确——它只是用来修正tableView中滚动条的尺寸的，当你在屏幕上滑动cell的时候，即使估算值不准确，tableView还是能很好地调节滚动条。将tableView的`estimatedRowHeight`属性设置成（在`viewDidLoad`或类似的方法中）一个接近于行高“平均值”的常量值即可。仅在行高极端变化的时候（比如相差一个数量级），滚动过程中才会产生滚动条的“跳跃”现象。这个时候，你才需要考虑实现`tableView:estimatedHeightForRowAtIndexPath:`方法，为每一行返回一个更精确的估算值。

### label的ios7实现
首先，实例化一个离屏(offscreen)的cell实例，为每个重用标示符实例化一个与之对应的cell实例，这些cell实例严格的仅用于高度计算。（离屏表示cell的引用被存储在view controller的一个属性或实例变量之中，并且这个cell绝对不会被用作`tableView:cellForRowAtIndexPath:`方法的返回值显示在屏幕上。）接下来，这个cell的内容（例如，文本、图片等等）还必须被配置为与显示在table view中的内容完全一样。
**因为要在`heightForRowAtIndexPath`中计算高度，需要一个cell实例，这个离屏cell就是为了这个用的。**

然后，强制cell立即更新子视图的布局，再在cell的`contentView`上调用`systemLayoutSizeFittingSize:`方法以计算出cell所需的高度。使用`UILayoutFittingCompressedSize`参数得到适合cell中所有内容所需的最小尺寸。然后将其高度作为`tableView:heightForRowAtIndexPath:`方法的返回值返回给table view。

通过`systemLayoutSizeFittingSize:`方法计算出的高度还需要加1才是真正的高度，这个1pt高度是cell分割线的高度。

iOS7中，你可以（也绝对应当）使用table view的`estimatedRowHeight`属性。这样会为还不在屏幕范围内的cell提供一个临时估算的行高值。然后，当这些cell即将要滚入屏幕范围内的时候，真实的行高值会被计算出来（通过`tableView:heightForRowAtIndexPath:`方法），估算的行高会被替换掉。

一般来说，行高的估算值不需要太精确——它只是用来修正tableView中滚动条的尺寸的，当你在屏幕上滑动cell的时候，即使估算值不准确，tableView还是能很好地调节滚动条。将tableView的`estimatedRowHeight`属性设置成（在viewDidLoad或类似的方法中）一个接近于行高“平均值”的常量值即可。仅在行高极端变化的时候（比如相差一个数量级），滚动过程中才会产生滚动条的“跳跃”现象。这个时候，你才需要考虑实现`tableView:estimatedHeightForRowAtIndexPath:`方法，为每一行返回一个更精确的估算值。

这样就是将第一次加载时时间消耗降低，但是在加载每个view的时候都要动态计算每一个cell的高度，即滑动时比较耗时，需要使用一个缓存将高度存取，同时这个缓存要能应对cell高度的变化进行修改。

示例：
```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath{
    _cell.label.text = [_data objectAtIndex:indexPath.row];
    CGSize textHeight = [_cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize];
    CGFloat height = textHeight.height+1>90?textHeight.height+1:90;
    return height;
}
```

### textView的自适应
textViewz在tableview中用的不多，但是其自适应高度方式和label有所不同。因为调用`systemLayoutSizeFittingSize`方法返回的高度并不包含textview的高度，需要调用`sizeThatFits`方法获得textview的完整高度，然后将两个高度相加才能得到cell真正的高度。

示例：
```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath{
    _cell.textView.text = [_data objectAtIndex:indexPath.row];
    CGSize textHeight = [_cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize];
    CGSize textViewSize = [_cell.textView sizeThatFits:CGSizeMake(_cell.textView.frame.size.width, FLT_MAX)];
    CGFloat h  = textHeight.height + textViewSize.height;
    CGFloat height = h+1>90?h+1:90;
    return height;
}
```

## UITableView+FDTemplateLayoutCell使用简介
这是一个由国人团队开发的优化计算 UITableViewCell 高度的轻量级框架。主要也是通过`systemLayoutSizeFittingSize`方法计算高度，但是该框架通过**缓存**及**预加载**将效率大幅的地提高。

这里就不具体分析该框架的实现原理了。可以参考[框架学习](http://blog.qiji.tech/archives/9538?utm_source=tuicool&utm_medium=referral)跟进。

这里主要列举下使用方法,其实在[forkingdog的github](https://github.com/forkingdog/UITableView-FDTemplateLayoutCell)上都有，我这里只是摘录下。

### 基本使用(不带cache)
```objc
#import "UITableView+FDTemplateLayoutCell.h"

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
{
    return [tableView fd_heightForCellWithIdentifier:@"reuse identifer" configuration:^(id cell) {
        // Configure this cell with data, same as what you've done in "-tableView:cellForRowAtIndexPath:"
        // Like:
        //    cell.entity = self.feedEntities[indexPath.row];
    }];
}
```

使用的时候需要将一个实例化后的cell作为block传入。如之前讲到的，`systemLayoutSizeFittingSize:`所必须的。

### 带cache的方法
#### 以indexPath区分
```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    return [tableView fd_heightForCellWithIdentifier:@"identifer" cacheByIndexPath:indexPath configuration:^(id cell) {
        // configurations
    }];
}
```
缓存下每个indexPath对应的高度。

#### 以key区分
```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    Entity *entity = self.entities[indexPath.row];
    return [tableView fd_heightForCellWithIdentifier:@"identifer" cacheByKey:entity.uid configuration:^(id cell) {
        // configurations
    }];
}
```
用户将要展示的cell分类，然后将分类的key传入。

### Frame layout mode
对于`Auto layout mode `和`Frame layout mode`。框架提供了两种方式，默认是自动布局。 

### 再次注意
需要再次提醒的是，如上文所说，一个好的自适应cell，一定要将cell内subview的上下左右约束设置好。

