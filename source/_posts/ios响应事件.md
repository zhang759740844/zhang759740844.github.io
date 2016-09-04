title: ios响应机制
date: 2016/9/3 10:07:12  
categories: IOS
tags:
	- UIResponder

---

从点击屏幕到系统做出响应，经历了哪些过程？需要详细探究下ios的响应机制。本文参考自[史上最详细的iOS之事件的传递和响应机制
](http://www.jianshu.com/p/2e074db792ba)

<!--more-->

## iOS中的事件
ios中的事件可以被分为三类：**触摸事件**，**加速计事件**，**远程控制事件**。本文讨论的是触摸事件。

### 响应者对象(UIResponder)
在iOS中不是任何对象都能处理事件，只有继承了`UIResponder`的对象才能接受并处理事件，我们称之为“响应者对象”。

以下都是继承自`UIResponder`的，所以都能接收并处理事件:
- UIApplication
- UIViewController
- UIView

那么为什么继承自UIResponder的类就能够接收并处理事件呢？因为UIResponder中提供了以下4个对象方法来处理触摸事件。
```objc
UIResponder内部提供了以下方法来处理事件触摸事件
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event;
- (void)touchesCancelled:(NSSet *)touches withEvent:(UIEvent *)event;
加速计事件
- (void)motionBegan:(UIEventSubtype)motion withEvent:(UIEvent *)event;
- (void)motionEnded:(UIEventSubtype)motion withEvent:(UIEvent *)event;
- (void)motionCancelled:(UIEventSubtype)motion withEvent:(UIEvent *)event;
远程控制事件
- (void)remoteControlReceivedWithEvent:(UIEvent *)event;
```

## 事件的处理
### 触摸事件：
下面以UIView为例来说明触摸事件的处理:
```objc
// UIView是UIResponder的子类，可以覆盖下列4个方法处理不同的触摸事件
// 一根或者多根手指开始触摸view，系统会自动调用view的下面方法
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
// 一根或者多根手指在view上移动，系统会自动调用view的下面方法（随着手指的移动，会持续调用该方法）
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event
// 一根或者多根手指离开view，系统会自动调用view的下面方法- (void)touchesEnded:(NSSet *)touches withEvent:(UIEvent *)event
// 触摸结束前，某个系统事件(例如电话呼入)会打断触摸过程，系统会自动调用view的下面方法
- (void)touchesCancelled:(NSSet *)touches withEvent:(UIEvent *)event
// 提示：touches中存放的都是UITouch对象
```
需要注意的是：
- 以上四个方法是由系统自动调用的，所以可以通过重写该方法来处理一些事件。很重要的一点：**如果想处理UIView的触摸事件，那么就在UIView中重写；如果想处理UIViewController的触摸事件，那么就在UIViewController中重写。**
- 如果两根手指同时触摸一个view，那么view只会调用一次`touchesBegan:withEvent:`方法，`touches`参数中装着2个`UITouch`对象.
- 如果这两根手指一前一后分开触摸同一个view，那么view会分别调用2次`touchesBegan:withEvent:`方法，并且每次调用时的`touches`参数中只包含一个`UITouch`对象.

### UITouch
当用户用一根手指触摸屏幕时，会创建一个与手指相关的UITouch对象。

#### UITouch作用
- 保存着跟手指相关的信息，比如触摸的位置、时间、阶段
- 当手指移动时，系统会更新同一个UITouch对象，使之能够一直保存该手指在的触摸位置
- 当手指离开屏幕时，系统会销毁相应的UITouch对象

#### UITouch属性
```objc
触摸产生时所处的窗口
@property(nonatomic,readonly,retain) UIWindow *window;

触摸产生时所处的视图
@property(nonatomic,readonly,retain) UIView *view
;

短时间内点按屏幕的次数，可以根据tapCount判断单击、双击或更多的点击
@property(nonatomic,readonly) NSUInteger tapCount;

记录了触摸事件产生或变化时的时间，单位是秒
@property(nonatomic,readonly) NSTimeInterval timestamp;

当前触摸事件所处的状态
@property(nonatomic,readonly) UITouchPhase phase;
```

#### UITouch方法
```objc
(CGPoint)locationInView:(UIView *)view;
// 返回值表示触摸在view上的位置
// 这里返回的位置是针对view的坐标系的（以view的左上角为原点(0, 0)）
// 调用时传入的view参数为nil的话，返回的是触摸点在UIWindow的位置

(CGPoint)previousLocationInView:(UIView *)view;
// 该方法记录了前一个触摸点的位置
```

#### 使用UITouch实现UIView的拖拽
```objc
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event{ 
    // 想让控件随着手指移动而移动,监听手指移动 
    // 获取UITouch对象 
    UITouch *touch = [touches anyObject]; 
    // 获取当前点的位置 
    CGPoint curP = [touch locationInView:self]; 
    // 获取上一个点的位置 
    CGPoint preP = [touch previousLocationInView:self]; 
    // 获取它们x轴的偏移量,每次都是相对上一次 
    CGFloat offsetX = curP.x - preP.x; 
    // 获取y轴的偏移量 
    CGFloat offsetY = curP.y - preP.y; 
    // 修改控件的形变或者frame,center,就可以控制控件的位置 
    // 形变也是相对上一次形变(平移) 
    // CGAffineTransformMakeTranslation:会把之前形变给清空,重新开始设置形变参数 
    // make:相对于最原始的位置形变 
    // CGAffineTransform t:相对这个t的形变的基础上再去形变 
    // 如果相对哪个形变再次形变,就传入它的形变 
    self.transform = CGAffineTransformTranslate(self.transform, offsetX, offsetY);}
```
通过UITouch对象获得当前点和上一个点的位置，求得偏移量

## iOS中的事件的产生和传递
### 事件的产生
1. 发生触摸事件后，系统会将该事件加入到一个由UIApplication管理的事件队列中。
2. UIApplication会从事件队列中取出最前面的事件，并将事件分发下去以便处理，通常，先发送事件给应用程序的主窗口（keyWindow）。
3. 主窗口会在视图层次结构中找到一个最合适的视图来处理触摸事件，这也是整个事件处理过程的第一步。
4. 找到合适的视图控件后，就会调用视图控件的touches方法来作具体的事件处理。

触摸事件的传递是从父控件传递到子控件,也就是UIApplication->window->寻找处理事件最合适的view。注意: 如果父控件不能接受触摸事件，那么子控件就不可能接收到触摸事件

### 找到最适合的控件
回到上面的第三部，从主窗口传来的触摸事件，需要找到最合适的视图，那么如何找到最佳视图？
1. 首先判断主窗口（keyWindow）自己是否能接受触摸事件。
2. 判断触摸点是否在自己身上。
3. 子控件数组中从后往前遍历子控件(后添加的view在上面，降低循环次数)，重复前面的两个步骤。
4. 如果触摸点在子控件上，那么重复3.
5. 如果没有符合条件的子控件，那么就认为自己最合适处理这个事件，也就是自己是最合适的view。
6. 找到最合适的view后，就会调用该view的touches方法处理具体的事件。

### UIView不能接受触摸事件的三种情况
- 不允许交互：`userInteractionEnabled = NO`
- 隐藏：如果把父控件隐藏，那么子控件也会隐藏，隐藏的控件不能接受事件
- 透明度：如果设置一个控件的透明度<0.01，会直接影响子控件的透明度。alpha：0.0~0.01为透明。

**注 意**:
- 默认UIImageView不能接受触摸事件，因为不允许交互，即`userInteractionEnabled = NO`，所以如果希望UIImageView可以交互，需要`userInteractionEnabled = YES`。
- 不管视图能不能处理事件，只要点击了视图就都会产生事件，关键看该事件是由谁来处理！也就是说，如果视图不能处理事件，点击视图，还是会产生一个触摸事件，只是该事件不会由被点击的视图处理而已！

### hitTest:withEvent：方法
#### 何时调用？
只要事件一传递给一个控件,这个控件就会调用他自己的`hitTest：withEvent：`方法.

#### 作用
寻找并返回最合适的view(能够响应事件的那个最合适的view)

**注 意：**
- 不管这个控件能不能处理事件，也不管触摸点在不在这个控件上，事件都会先传递给这个控件，随后再调用`hitTest:withEvent:`方法
- `hitTest`的这个`CGPoint point`表示当前手指触摸的点，其值是以方法调用者的坐标系为基准的。也就是说，不同的控件调用这个方法，这个point都是经过计算而产生的不同的值。


#### 拦截事件的处理
- 正因为`hitTest：withEvent：`方法可以返回最合适的view，所以可以通过重写`hitTest：withEvent：`方法，返回指定的view作为最合适的view。
- 不管点击哪里，最合适的view都是`hitTest：withEvent：`方法中返回的那个view。
- 通过重写`hitTest：withEvent：`，就可以拦截事件的传递过程，想让谁处理事件谁就处理事件。

**注 意：**如`hitTest:withEvent:`方法中返回`nil`，那么调用该方法的控件本身和其子控件都不是最合适的view，也就是在自己身上没有找到更合适的view。那么最合适的view就是该控件的父控件。

#### 技巧
想让谁成为最合适的view就重写谁自己的父控件的hitTest:withEvent:方法返回指定的子控件。这里又要注意，最好不要在子控件内`return self`。因为，在遍历子控件的时候，很有可能没有遍历到你真正想要返回的那个控件，就在其他控件已经return了。

例如：whiteView有redView和greenView两个子控件。redView先添加，greenView后添加。如果要求无论点击那里都要让redView作为最合适的view（把事件交给redView来处理）那么只能在whiteView的hitTest:withEvent:方法中return self.subViews[0];这种情况下在redView的hitTest:withEvent:方法中return self;是不好使的！

#### hitTest:withEvent：的底层实现
```objc
#import "WSWindow.h"
@implementation WSWindow
// 什么时候调用:只要事件一传递给一个控件，那么这个控件就会调用自己的这个方法
// 作用:寻找并返回最合适的view
// UIApplication -> [UIWindow hitTest:withEvent:]寻找最合适的view告诉系统
// point:当前手指触摸的点
// point:是方法调用者坐标系上的点
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event{
    // 1.判断下窗口能否接收事件
     if (self.userInteractionEnabled == NO || self.hidden == YES ||  self.alpha <= 0.01) return nil; 
    // 2.判断下点在不在窗口上 
    // 不在窗口上 
    if ([self pointInside:point withEvent:event] == NO) return nil; 
    // 3.从后往前遍历子控件数组 
    int count = (int)self.subviews.count; 
    for (int i = count - 1; i >= 0; i--)     { 
    // 获取子控件
    UIView *childView = self.subviews[i]; 
    // 坐标系的转换,把窗口上的点转换为子控件上的点 
    // 把自己控件上的点转换成子控件上的点 
    CGPoint childP = [self convertPoint:point toView:childView]; 
    UIView *fitView = [childView hitTest:childP withEvent:event]; 
    if (fitView) {
    // 如果能找到最合适的view 
    return fitView; 
    }
    } 
    // 4.没有找到更合适的view，也就是没有比自己更合适的view 
    return self;
    }
    // 作用:判断下传入过来的点在不在方法调用者的坐标系上
    // point:是方法调用者坐标系上的点
    //- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event
    //{
    // return NO;
    //}
    - (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event{ 
    NSLog(@"%s",__func__);
    }
    @end
```

`hit:withEvent:`方法底层会调用`pointInside:withEvent:`方法判断点在不在方法调用者的坐标系上。

#### pointInside:withEvent:方法
`pointInside:withEvent:`方法判断点在不在当前view上（方法调用者的坐标系上）如果返回YES，代表点在方法调用者的坐标系上;返回NO代表点不在方法调用者的坐标系上，那么方法调用者也就不能处理事件。


## 事件的响应
### 事件的传递与响应
前面说到了，当一个事件发生后，事件会从父控件传给子控件，也就是说由UIApplication -> UIWindow -> UIView -> initial view,以上就是事件的传递，也就是寻找最合适的view的过程。

在找到了最合适的view后，接下来就是事件的响应过程。响应过程是传递过程的逆过程。在事件的响应中，如果某个控件实现了`touches...`方法，则这个事件将由该控件来接受，如果调用了`[super touches….]`;就会将事件顺着响应者链条往上传递，传递给上一个响应者；接着就会调用上一个响应者的`touches….`方法。

touches默认做法是把事件顺着响应者链条向上抛：
```objc
#import "WSView.h"
@implementation WSView 
//只要点击控件,就会调用touchBegin,如果没有重写这个方法,自己处理不了触摸事件
// 上一个响应者可能是父控件
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event{ 
// 默认会把事件传递给上一个响应者,上一个响应者是父控件,交给父控件处理
[super touchesBegan:touches withEvent:event]; 
// 注意不是调用父控件的touches方法，而是调用父类的touches方法
// super是父类 superview是父控件 
// 不过super内还是要调用superview的touch的方法来交给上一个响应者的。
}
@end
```

### 一个事件多个对象处理
因为系统默认做法是把事件上抛给父控件，所以可以通过重写自己的touches方法和父控件的touches方法来达到一个事件多个对象处理的目的。

```objc
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event{ 
// 1.自己先处理事件...
NSLog(@"do somthing...");
// 2.再调用系统的默认做法，再把事件交给上一个响应者处理
[super touchesBegan:touches withEvent:event]; 
}
```

### 事件的传递和响应的区别
事件的传递是从上到下（父控件到子控件），事件的响应是从下到上（顺着响应者链条向上传递：子控件到父控件。







