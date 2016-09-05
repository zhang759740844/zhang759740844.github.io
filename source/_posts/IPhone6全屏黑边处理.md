title: IPhone6全屏黑边处理
date: 2016/9/5 14:07:12  
categories: IOS
tags: 
	- Xcode
	
---

写ios小demo的时候，真机调试出现View不能满屏显示的情况。屏幕上下都有一条粗粗的黑条。

<!--more-->

检查了许久，代码里都是直接设置的`UIScreen`的`bounds`。因此，不可能是代码的问题。那么需要在设置中找原因。

通过和正确的demo进行对比，发现了差异所在。

这是有黑边情况下的设置：

![错误设置](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SetLaunchImage2.png?raw=true)

这是没有黑边情况下的设置：
![正确设置](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/SetLaunchImage1.png?raw=true)

通过比较可以发现：
出现黑边的原因是在设置里设置了`Launch Images Source`为`LaunchImage`,但是在Images.xcassets中并没有`LaunchImage`资源。所以，需要给`LaunchImage`设置图片资源。
也可以像上面的那样设置`Launch Images Source`为`Use Asset Catalog`,并且将`Launch Screen File`设置为`LaunchImage`即可解决。