title: Storyboard 使用方法	
date: 2016/10/13 14:07:12  
categories: iOS 
tags: 
​	- Xcode

---

Storyboard是苹果官方主推的一个代替xib的策略。有必要详细学习下它的使用方法。

<!--more-->

先来看一下思维导图
![storyboard_28](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_28.png?raw=true)

## storyboard基础
### storyboard优势
storyboard能代替nib自然有其优势，一般来说storyboard具有以下几种优点：

- storyboard能将nib汇总统一管理
- storyboard可以描述各种场景之间的过渡，这种过渡被称作`segue`，storyboard 把 view controller叫做：`scene`，可以通过拖拽实现过度，减少代码
- 支持tableview的prototype cell，可以在storyboard中编辑cell，减少代码量

### storyboard的基本使用
#### 启动storyboard
加载storyboard肯定是需要一个主入口的，这个主入口在`info.plist`中：
![storyboard_1](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_1.png?raw=true)

#### 初始化ViewController
确定了哪个storyboard是主入口，那显然也要确定哪个ViewController是storyboard的主入口。需要选中相应ViewController，勾选`Is Initial View Controller`
![storyboard_2](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_2.png?raw=true)
此时，对应ViewController前出现一个箭头
![storyboard_3](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_3.png?raw=true)

#### 创建relationship segue
对于三大container view controller，即Tab Bar Controller，Navigation Controller，Split View Controller ，可以通过拖拽创建设置relationship segue。

如下图的 popup menu 是从tab bar controller 连到navigation controller,松手后的弹出：
![storyboard_4](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_4.png?raw=true)
连接后的图标如下图，表示relationship segue
![storyboard_5](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_5.png?raw=true)

#### 命名tabbar controller的tabbar
并非在tabbar controller里，而是在与其相连、对应的controller里改动，如图：
![storyboard_6](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_6.png?raw=true)
navigation bar 的 title 也是同理。但是，强烈不建议在storyboard里设置navigationbar，因为storyboard是为了简化操作的，但是设置navigationbar太麻烦了，还不如代码方便实用。

#### 设置ViewController对应的类
选中相应ViewController，然后在 Custom Class 内写上相应类名即可。注意，要选中 ViewController 而不是其中的 View，要点击图中的黄色圆形按钮。
![storyboard_7](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_7.png?raw=true)

#### 获取视图控制器
就是通过`UITableViewController`和`UINavigationController`中的`viewControllers`获取：

```objc
UITabBarController *tabBarController = (UITabBarController *)self.window.rootViewController;
UINavigationController *navigationController = [tabBarController viewControllers][0];
PlayersViewController *playersViewController = [navigationController viewControllers][0];
```

#### Prototype cells
选中tableview，设置tableview的 cotent 为 Dynamic Propotype
![storyboard_8](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_8.png?raw=true)
一般在`tableView: cellForRowAtIndexPath:`方法中都像下面：

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
    static NSString *CellIdentifier = @"Cell";

    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:CellIdentifier];
    if (cell == nil) {
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault
                                      reuseIdentifier:CellIdentifier];
    }

    // Configure the cell...
    return cell;
}
```

但是，由于在storyboard中已经创建了cell，那么就可以直接使用了：

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"PlayerCell"];

    // Configure the cell...

    return cell;
}
```

当然，一个tableview里肯定不应该只有一个，可以把上面的 Content 下面的 Prototype Cells 增加cell，然后选择任意cell，如图：
![storyboard_9](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_9.png?raw=true)
设置不同的 identifier 来标识不同的cell。

除了在viewcontroller里直接创建cell，不需要另一个cell的xib的区别外，其他方面和xib无异。也可以选中cell，在右边栏指定对应的Controller的custom class用来控制cell。同样的，也不能直接把cell中的view连线到cell所属的viewController中。

#### Static cells
使用静态的cell，适用在仅有几个确定cell的tableview中，不能重用，设置了几个cell，就显示几个cell。static cells的设置如下图：
![storyboard_10](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_10.png?raw=true)
因为prototype cells究竟怎么显示可以在代码中设置，所以只需要设置有几个可重用的cell就行了，而static cells因为不可重用，那么这里的设置选项就变成了 Sections 设置多少段。

那么怎么设置每个section有多少个cell呢？选中如下图所示的只有static cells才有的蓝色立方体：
![storyboard_11](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_11.png?raw=true)
此时右边栏出现如图所示 Table View Section
![storyboard_12](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_12.png?raw=true)
可以设置数量，表头表尾的title

和 prototype cell 一样，static cell可以指定一个专门的Controller。但是不同的是，static cell 的cell以及cell中的控件都相当于确定的view，因此，static cell可以把cell以及cell中的控件连线到cell所属的viewController中。
也就是说，如果在cell的Controller中设置了一个button的点击事件，然后又在cell所属viewController中又设置了一次该button的点击事件，不会报错，两个点击事件都会触发。

所以，方便起见，static cell 直接在viewController中连线设置就可以了。





## 使用segue
### 简介
#### 什么是Segue
Storyboard上每一根用来界面跳转的线，都是一个UIStoryboardSegue对象（简称Segue）

#### Segue的属性
每一个Segue对象，都有3个属性

```objc
// 唯一标识
@property (nonatomic, readonly) NSString *identifier;
// 来源控制器
@property (nonatomic, readonly) id sourceViewController;
// 目标控制器
@property (nonatomic, readonly) id destinationViewController;
```

#### Segue的类型
根据Segue的执行（跳转）时刻，Segue可以分为2大类型:
- 自动型(Action segue)：点击某个控件后（比如按钮），自动执行Segue，自动完成界面跳转。
- 手动型(Manual segue)：需要通过写代码手动执行Segue，才能完成界面跳转。

#### segue执行过程
手动调用`performSegueWithIdentifier:sender:`方法实现跳转。那么这期间发生了什么呢？大致分为三个部分。

1. 根据`identifier`去storyboard中找到对应的线，新建UIStoryboardSegue对象

```objc
- (instancetype)initWithIdentifier:(NSString *)identifier source:(UIViewController *)source destination:(UIViewController *)destination; // Designated initializer
```

其实就是执行了UIStoryboardSegue中`initWithIdentifier:source:destination:`方法,并且`identifier`就是在Storyboard中Segue属性设置的标识. 来源就是连线的头部. 目标就是连线尾

2. 调用sourceViewController的下面方法，做一些跳转前的准备工作并且传入创建好的Segue对象

```objc
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender;
```

所谓跳转前的准备，因为可以拿到Segue(来源控制器,目标控制器)，所以就可以在这里给下一个控制器传递数据。这个方法是系统默认调用，所以只需要实现即可。另外，只能由来源控制器调用,来拿到目标控制器。

3. 调用Segue对象的`perform`方法开始执行界面跳转操作。

### 页面跳转
segue可以实现页面间跳转，除了上面的 relationship segue 还有 Action segue 和 Manual segue，分别对应button跳转和viewController跳转。

#### 跳进
##### 使用storyboard
Action segue 比较简单，就是将button连到要展示的viewController上，当点击时，就会触发。
Manual segue 相对比较麻烦，但是比较灵活。它设置了两个viewController的跳转关系，在你需要的时候出发跳转。

首先，先对两个viewController进行连线：
![storyboard_14](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_14.png?raw=true)
之后点击连线后两个viewController之间产生的箭头，在右边栏可以看到如下：
![storyboard_15](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_15.png?raw=true)
其中参数 `identifier` 就是跳转的标识符，根据这个标识符来确定跳转到是那个页面。下面几个参数，下面再说。

接下来就可以调用方法，在合适的时机加载了

```objc
//根据 segue Identifier跳转界面
[self performSegueWithIdentifier:@"GotoTwo" sender:self];
```

其中的`identifer`自然不用多说，那么`sender`是什么呢？`sender`是参数名称，理论上可以指代任何对象，用来区分是哪个控件触发了segue。比如有两个button都跳转到一个页面，那么这时就可以设置`sender`区分了。引申开来，在设置button点击事件时的`-(IBAction)click:(id)sender;`方法中的`sender`和这里的`sender`是一个作用。


##### 使用纯代码
上面的方法实现效果和平时用的下面两个方法相同：

```objc
//以modal 方式跳转
[self presentViewController:ViewController animated:YES completion:nil];
//压进一个viewcontroller
[self.navigationController pushViewController:ViewController animated:YES completion:nil];
```

不过，既然用了storyboard了，那么实例化viewController时就不能用`initWithNibName`了。在storyboard中，要通过storyboard找到viewController的布局。首先要设置viewcontroller的 storyboardID：
![storyboard_19](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_19.png?raw=true)
那个`use storyboardId`的勾不打也行，不知道干什么用的，

现在就可以在代码中找到特定storyboard的viewcontroller了：

```objc
- (IBAction)tapButton:(id)sender {
    //获取storyboard: 通过bundle根据storyboard的名字来获取我们的storyboard,
    UIStoryboard *story = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
    //由storyboard根据myView的storyBoardID来获取我们要切换的视图
    UIViewController *myView = [story instantiateViewControllerWithIdentifier:@"myView"];
    //显示ViewController
    [self presentViewController:myView animated:YES completion:nil];
}
```

#### 跳出
能跳进当然也要能跳出，可以使用 exit segue 跳转至任意连线的位置，也可以使用代码跳转。

##### exit segue 
跳出和跳进的方法类似，略有区别，比如要从界面2跳转回界面1：

先打开需要返回到的界面ViewController1.m,加上下面方法，返回类型一定是`IBAction`，参数类型**一定**是`UIStoryboardSegue`，名称随便（这个方法一定要加，返回时调用的）

```objc
//其他界面返回到此界面调用的方法
- (IBAction)ViewController1UnwindSegue:(UIStoryboardSegue *)unwindSegue {
}
```

右键2界面上方的Exit（下图中画绿圈的）弹出菜单中可以看到刚才在1界面中加的那个方法的名称（下图中红色圈里），然后如下图一样连线，弹出菜单选择`manual`，这里连接自己表示要在当前viewcontroller中用代码的方式回退。
![storyboard_17](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_17.png?raw=true)

给2视图的unwind segue取一个名字叫`from2to1`的identifier如下图:
![storyboard_18](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_18.png?raw=true)

现在就可以在界面2中的任意时候调用方法回退了：

```objc
- (IBAction)back:(id)sender {
　　//执行segue跳页的方法
    [self performSegueWithIdentifier:@"from2to1" sender:nil];
}
```

使用的仍是跳进时用的方法，不过第一步的操作已经告诉xcode这是一个回退操作了。可以从上图看到，这个 Unwind Segue 绑定了回退到的界面的一个方法，因此，执行跳转后会执行绑定的方法：

```objc
//其他界面返回到此界面调用的方法
- (IBAction)ViewController1UnwindSegue:(UIStoryboardSegue *)unwindSegue {
    if ([unwindSegue.identifier isEqualToString:@"from2to1"]) {
        _lbShowMessage.text = @"从2退到1";
    } else if ([unwindSegue.identifier isEqualToString:@"from3to1"]) {
        _lbShowMessage.text = @"从3退到1";
    }
}
```

这里就看出上面为什么说类型一定是`UIStoryboardSegue`了，因为可以接收一个该类型的对象，以此判断是从哪个页面的回退。

使用 exit segue 的**好处**是可以跳转到任意打开过的界面比如从3->1，而不是只能返回上级界面从2->1。


##### 一般跳出方法
也可以使用代码根据是model类型还是push类型选择：

```objc
//弹出一个viewcontroller  相当与返回上一个界面
[self.navigationController popViewControllerAnimated:YES];   
// 以 modal跳转的返回方法
[self dismissViewControllerAnimated:YES];
```

### 跳转的方式
在进行跳转连线后会出现如下窗口： 
![storyboard_13](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_13.png?raw=true)
共有三种跳转方式，也就是上面右边栏的`Kind`属性

#### push
Push类型必须用在NavigationController中，否则报错。是在navigation View Controller中下一级时使用的那种从右侧划入的方式。

#### model
最常用的场景，新的场景完全盖住了旧的那个。用户无法再与上一个场景交互，除非他们先关闭这个场景。可以在右边栏的`Presentation`选择需要展示的动画效果。

#### custom
自定义类型，需要继承UIStoryboardSegue类，然后重写Perform方法,然后在Storyboard上将类设置为自定义的类。
![storyboard_16](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_16.png?raw=true)
这段代码的作用是创建从中心渐变充满屏幕的动画:

```objc
-(void)perform{
    UIViewController * svc = self.sourceViewController;
    UIViewController * dvc = self.destinationViewController;
    [svc.view addSubview:dvc.view];
    [dvc.view setFrame:svc.view.frame];
    [dvc.view setTransform:CGAffineTransformMakeScale(0.5, 0.5)];
    [dvc.view setAlpha:0.0];
    [UIView animateWithDuration:1.0
                     animations:^{
                         [dvc.view setTransform:CGAffineTransformMakeScale(1.0, 1.0)];
                         [dvc.view setAlpha:1.0];
                     }
                     completion:^(BOOL finished) {
                         [svc presentViewController:dvc animated:NO completion:nil];
                     }];
}
```

其实实质还是`presentViewController`，但是不用系统带的`animation`，而是先将`destinationViewController`的页面用动画加载后，直接`present`。

### 跳转传值
前面说到，`- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender `方法会在跳转时自动触发。跳转传值就在这个方法内完成。

我们可以对Segue的标识进行判断，一般有以下两种方法：

```objc
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender
{
    if ([segue.identifier  isEqual: @"login2index"]) {
        // 需要执行的代码
    }
}
```

```objc
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender
{
    if ([segue.destinationViewController isKindOfClass:[IndexTableViewController class]]) {
        //  执行代码
    }
}
```

第一种需要在设置标识的值,并且匹配。第二种却是通过目标控制器判断。个人感觉还是第一种靠谱一些。

接下来就可以对`destination`进行赋值了：

```objc
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender
{
    
    NSLog(@"触发该场景切换的sender对象的类型是:%@",[sender class]);
    
    //方法一,使用KVC给B 也就是目标场景传值
    UIViewController *destinationController=[segue destinationViewController];
    [destinationController setValue:@"119" forKey:@"number"];
    
    //方法2,使用属性传值,需导入相关的类.h
    //BViewController *bController=[segue destinationViewController];
    //bController.number=@188;
}
```

跳转传值不仅可以用`prepareForSegue:sender:`实现，也可以通过代理、通知的方式，不过这样挺麻烦的，不推荐。具体参见[使用storyboard实现页面跳转，简单的数据传递](http://blog.csdn.net/mad1989/article/details/7919504#comments)

## storyboard reference
iOS9中，苹果引入了 storyboard reference 用以减小storyboard的体积，方便管理（并不知道iOS9之前怎么用多个storyboard）。

### 简化现有storyboard
如下图，是做上面练习时创建的一个storyboard，界面已经有点多了。可以使用storyboard reference简化，将一部分viewcontroller拆分到其他storyboard里。

![storyboard_23](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_23.png?raw=true)

做法其实很简单，选中想要拆分的viewcontroller，然后在菜单栏干中“Editor->Refactor to Storyboard”，如下图所示。然后命名新的storyboard即可。
![storyboard_24](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_24.png?raw=true)

![storyboard_25](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_25.png?raw=true)

### 加载storyboard中一个特定viewController
和拖拽其他控件一样，找到storyboard控件，拖拽到storyboard上：
![storyboard_26](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_26.png?raw=true)

然后设置storyboard：
![storyboard_27](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/storyboard_27.png?raw=true)
这里面`storyboard`填的是目标storboard的文件名；`Reference ID`是啥？从它的提示也就才出来了，用来确定联结的是那个viewController，填的是目标storyboard中目标viewController的`Storyboard ID`，具体在哪设，上面也说过。

这样，一个简洁的storyboard就能创建出来了。


## 参考链接
[【Storyboard】Storyboard介绍及使用](http://www.jianshu.com/p/2ec2c19f183e)

[UIStoryboardSegue讲解](http://birdmichael.com/?p=180);

[iOS-prepareForSegue场景切换,KVC传值](http://blog.csdn.net/yangbingbinga/article/details/43704269);

[(4.4.1)使用storyboard实现页面跳转，简单的数据传递](http://blog.csdn.net/mad1989/article/details/7919504#comments);

[【iOS界面处理】使用storyboard实现页面跳转，简单的数据传递](http://blog.csdn.net/xuqiang918/article/details/17023737)

[iOS9 Day-by-Day :: Day 3 :: Storyboard References](https://www.shinobicontrols.com/blog/ios9-day-by-day-day3-storyboard-references);

[基于Storyboard的创建多分支NavigationController的方法](http://www.cnblogs.com/shanpow/p/4149462.html);

[iOS 9 Storyboard 教程(一下)](http://blog.csdn.net/yangmeng13930719363/article/details/49886901);

[10 Practical Tips for iOS Developers Using Storyboards](http://www.xmcgraw.com/10-practical-tips-for-ios-developers-using-storyboards/);

还有一些看完随手就关了，没有记录。

水平有限，如有错误，多多指正~