title: CALayer的transform
date: 2016/9/2 14:07:12  
categories: IOS
tags: 
	- Animation
	- UI

---

在[CollectionView的使用](https://zhang759740844.github.io/2016/08/05/UICollectionView/)中，有为cell添加transform属性，实现自定义动画。现在，详细了解下View及其Layer的transform属性。


<!--more-->

## UIView的transform属性
transform是view的一个重要属性,它在矩阵层面上改变view的显⽰状态,能实现view的缩放、旋转、平移等功能。transform是`CGAffineTransform`类型的。

### transform结构
transform是一个`CGAffineTransform`类型，结构如下：
```objc
struct CGAffineTransform {
  CGFloat a, b, c, d;
  CGFloat tx, ty;
};
```

CGAffineTransform实际上是一个矩阵
```objc
| a,  b,  0 |
| c,  d,  0 |
| tx, ty, 1 |
```
由于transform只有两维，需要一个3阶矩阵来表示其缩放以及平移的变化。

坐标变换过程：
```objc
                    | a,  b,  0 |
{x',y',1}={x,y,1} x | c,  d,  0 |
                    | tx, ty, 1 |
                    
==>

xn=ax+cy+tx;
yn=bx+dy+ty;

```

这个矩阵的第三列是固定的，所以每次变换时，只需传入前两列的六个参数[a,b,c,d,tx,ty]即可。

### transform方法
在`CGAffineTransform`的生成函数中，大多是两两对应的，一个带
make字样，一个不带。带make字样的是直接生成一个新的`CGAffineTransform`，不带make字样的则是在一个`CGAffineTransform`的基础上生成新的。函数返回值均是`CGAffineTransform`类型。

多个`CGAffineTransform`对象赋给view，最终只执行最后一个动画，多个动画需要组合在一起。


#### scale
实现的是放大和缩小:
```objc
CGAffineTransformScale(CGAffineTransform t,
  CGFloat sx, CGFloat sy)；
CGAffineTransformMakeScale(CGFloat sx, CGFloat sy)；
```
生成新的transform相当于将`t' = [sx ，0 ，0，sy ，0， 0]`这六个参数代入矩阵中,即改变a和d。

#### rotate
实现的是旋转：
```objc
CGAffineTransformRotate(CGAffineTransform t,
  CGFloat angle)
CGAffineTransformMakeRotation(CGFloat angle)；
```
angle为角度，angle=π则旋转180度。矩阵的六个参数为`t' =  [ cos(angle)，sin(angle)，-sin(angle)，cos(angle) 0，0]；`

#### translate
实现的是平移：
```objc
CGAffineTransformTranslate(CGAffineTransform t,
  CGFloat tx, CGFloat ty)；
CGAffineTransformMakeTranslation(CGFloat tx,
  CGFloat ty)；
```
矩阵的六个参数为`t' = [1，0，0，1，tx，ty] ；`代入公式，`xn=x+tx,yn=y+ty`

#### 复原
```objc
view.transform＝CGAffineTransformIdentity;
```
上述的各种动画的变化都是以原始图像为基准的，而不是在变化后继续变化。`CGAffineTransformIdentity`将view从当前状态复原回view最初始的状态。

## CALayer的transform属性
### transform结构
CALayer的transform是一个`CATransform3D`结构：
```objc
struct CATransform3D
{
  CGFloat m11, m12, m13, m14;
  CGFloat m21, m22, m23, m24;
  CGFloat m31, m32, m33, m34;
  CGFloat m41, m42, m43, m44;
};
```
有别于`CGAffineTransform`,`CATransform3D`是一个三维变化，需要一个4阶矩阵表示。其他类似，再次不表。

### transform方法
CALayer的transform方法和View的transform基本一致。举几点不同：

- CALayer由于有z轴，因此对不同图层使用`CATransform3DMakeTranslation (CGFloat tx, CGFloat ty, CGFloat tz)`方法的时候可以通过改变tz的值，来实现图层的覆盖。对于tz来说，值越大，那么图层就越往外（接近屏幕），值越小，图层越往里（屏幕里）。

- 由于图像是从正面投影，直接绕着x或y轴旋转达不到透视的效果，如果要达到透视效果，需要改变`m34`(其实改变m14，m24也能达到效果，可以自行通过行列式推导。)，再对图层进行旋转。`m34`的值可以根据需要实现的效果推导得到，不直接的方法还是直接试。

- 如果想要直接改变矩阵里的值，可以先使用`CATransform3DIdentity`的方式，初始化一个`CATransform3D`实例，然后再赋值。



