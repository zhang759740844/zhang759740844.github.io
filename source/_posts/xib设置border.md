title: 使用xib设置view的border的宽度和颜色
date: 2016/8/10 14:07:12  
categories: IOS
tags: 
	- Runtime
	- UI
	
---

使用User Defined Runtime Attributes可以配置一些在interface builder 中不能配置的属性。有助于编写更加轻量级的viewcontroller。

<!--more-->

写一个button的时候经常会需要设置其cornerRadious，borderWidth和borderColor三个属性。如果在代码中实现，需要设置一个button属性，再与xib进行关联。
可以使用User Defined Runtime Attributes在xib中进行设置。

对一个view进行设置主要还是设置它的layer。因此，设置的key,type依次是：
```objc
layer.cornerRadius				Number
layer.borderWidth				Number
layer.borderColor				Color
```

但是，进过设置后会发现borderColor属性设置并不成功。这是因为这里设置的颜色类型是UIColor而borderColor是CGColor因此显示不出来。

需要使用category定义一个CALayer的方法
```objc
#import "CALayer+Additions.h"
#import <UIKit/UIKit.h>
@implementation CALayer (Additions)
- (void)setBorderColorFromUIColor:(UIColor *)color{
    self.borderColor = color.CGColor;
}
@end
```
通过这个方法可以把Color设置为borderColor。
xib中的key需要改成此方法名
**layer.borderColorFromUIColor**

在xib中设置borderColorFromUIColor的时候，不需要将新建的CALayer+Additions.h头文件import入任何类，编译的时候会自动调用。

设置masksToBounds为YES（也就是xib中的clip bounds）可以将图层里面东西截取，否则，超出父布局的子视图也将全部显示。
