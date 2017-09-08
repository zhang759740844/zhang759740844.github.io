title: 许多 iOS API 的使用方式(持续更新)
date: 2017/2/13 14:07:12  
categories: iOS
tags: 

	- 学习笔记
	- 持续更新

------

很多不常用的简单的 API 的使用方式。分开写的话太短小，就合在一起吧。（这里都是学到的时候，东找一点西找一点拼凑起来的，一开始没有记录出处在哪，如果哪里有侵权的地方，还请尽快告知呀。我会第一时间注明出处的。谢谢啦~）

<!--more-->

## NSJSONSerialization

JSON(也就是特定类型的 `NSString`) 和 `NSDictionary`、`NSArray` 之间的转换可以通过 `NSJSONSerialization` 类进行

### JSON(NSString) => NSDictionary/NSArray

先将 JSON 通过 `dataUSingEncoding:` 转换为 `NSData`，然后再用通过 `NSJSONSerialization` 将 `NSData` 转换为 `NSDictionary`/`NSArray`.

```objc
#import "NSString+JSONCategories.h"
@implementation NSString(JSONCategories)
-(id)JSONValue {
   NSData* data = [self dataUsingEncoding:NSUTF8StringEncoding];
   __autoreleasing NSError* error = nil;
   id result = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:&error];
   if (error != nil) return nil;
   return result;
}
@end
```

使用：

```objc
// 数组
NSString *str = @"[{ \"id\": \"hu\"},{\"blog\": \"damon\"}]";
NSArray *array = (NSArray*)[str JSONValue];

// 字典
NSString *str = @"{ \"id\": \"hu\",\"blog\": \"damon\"}";
NSDictionary *array = (NSDictionary *)[str JSONValue];
```

数组的 JSON 是用 `[]` 包裹起来的。这个例子中每个元素都是单元素的 `NSDictionary`。字典的 JSON 就是 `{}` 括起来的键值对。注意由于返回时是 `id` 类型，不区分具体是 `NSDictionary` 还是 `NSArray`，所以要进行类型转换

### NSDictionary/NSArray => JSON(NSString)

在 `NSObject` 中添加分类，先将 `NSDictionary`/`NSArray` 转换为 `NSData`。注意上面使用的方法是 `JSONObjectWithData:options:error:` 这里是 `dataWithJSONObject:options:error:`。然后通过 `initWithData:encoding:` 将 `NSData` 转为 `NSString`:

```objc
#import "NSObject+JSONCategories.h"
@implementation NSObject (JSONCategories)
-(NSString *)JSONString {
   NSError *error = nil;
   NSData *result = [NSJSONSerialization dataWithJSONObject:self
                                               options:kNilOptions error:&error];
   if (error != nil) return nil;
   return [[NSString alloc] initWithData:result encoding:NSUTF8StringEncoding];
}
@end
```

### NSString 与 NSArray 的互转

上面的 JSON 是一种特殊格式的 `NSString`，所以要借助于 `NSJSONSerialization` 进行解析。但是如果直接的 `NSString` 和 `NSArray` 的互相转换就要简单许多，但是还有注意点.

一般我们把 `NSArray` 转为 `NSString` 是直接通过 `stringWithFormat:` 的形式：

```objc
NSArray *array = [NSArray arrayWithObjects:@"sss",@"mmm",@"lll",@"kkk",@"ppp",@"ooo", nil];
NSString *str1 = [NSString stringWithFormat:@"%@",array];

//输出 str1
str1 = @"(\n    sss,\n    mmm,\n    lll,\n    kkk,\n    ppp,\n    ooo\n)"
```

可以看出，这样的转换是有问题的，中间引入了空格，并且两边还有括号没有消除。👇是正确的方式：

```objc
NSArray *array = [NSArray arrayWithObjects:@"sss",@"mmm",@"lll",@"kkk",@"ppp",@"ooo", nil];
NSString *str2 = [array componentsJoinedByString:@","];

// 输出 str2 
str2 = @"sss,mmm,lll,kkk,ppp,ooo"
```

通过 `NSArray` 的方法，将数组中的元素完全拿了出来。

另一方面， `NSString` 转为 `NSArray`。通过 `NSString` 的 `componentsSeparatedByString:` 方法，识别逗号:

```objc
NSArray *array2 = [str1 componentsSeparatedByString:@","];
NSArray *array3 = [str2 componentsSeparatedByString:@","];
```

比较输出结果可以发现，`str1` 无法重新转回最开始的数组了，所以两者互转一定要用 `str2` 的方式：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/NSString_NSArray.png?raw=true)



## UIView 中的坐标转换

一个 View 的 `frame` 的起点是相当于其所在的 View，即调用 `addSubView:` 方法的 View。如果要判断两个 View 是否是包含关系，由于两者的起点不同，那么肯定是无法进行比较的。

```objc
// rect1和rect2是否有重叠
CGRectContainsRect(<#CGRect rect1#>, <#CGRect rect2#>)
// point是不是在rect上
CGRectContainsPoint(<#CGRect rect#>, <#CGPoint point#>)
// rect1是否包含了rect2
CGRectIntersectsRect(<#CGRect rect1#>, <#CGRect rect2#>)
```

为了统一原点，我们可以使用以下代码：

```objc
- (CGPoint)convertPoint:(CGPoint)point toView:(nullable UIView *)view;
- (CGPoint)convertPoint:(CGPoint)point fromView:(nullable UIView *)view;

- (CGRect)convertRect:(CGRect)rect toView:(nullable UIView *)view;
- (CGRect)convertRect:(CGRect)rect fromView:(nullable UIView *)view;
```

来举两个例子，注意不同情况下 `compareView` 和 `outerView` 的参数位置：

```objc
CGRect newRect = [self.compareView convertRect:self.innerFrame fromView:self.outerView];
CGRect newRect = [self.outerView convertRect:self.innerFrame toView:self.compareView];
```

得到的就是 `innerFrame` 在 `compareView` 中的位置。







## UIVisualEffectView 实现高斯模糊

如果想要给一个 view  添加一个高斯模糊的效果，只要在那个 view 上添加一个 `UIVisualEffectView` 即可。

高斯模糊有三种效果，从浅入深的 style 依次是：

- UIBlurEffectStyleExtraLight
- UIBlurEffectStyleLight
- UIBlurEffectStyleDark



使用的基本实例：

```objc
// 要添加模糊的view
UIImageView *imageview = [[UIImageView alloc] init];
imageview.frame = CGRectMake(10, 100, 300, 300);
imageview.image = [UIImage imageNamed:@"2"];
imageview.contentMode = UIViewContentModeScaleAspectFit;
imageview.userInteractionEnabled = YES;
[self.view addSubview:imageview];

// 高斯模糊的view
UIBlurEffect *blur = [UIBlurEffect effectWithStyle:UIBlurEffectStyleLight];
UIVisualEffectView *effectview = [[UIVisualEffectView alloc] initWithEffect:blur];
effectview.frame = CGRectMake(0, 0, imageview.size.width/2, 300);

// 添加高斯模糊
[imageview addSubview:effectview];
```



## UIInterpolatingMotionEffect 视图运动

`UIInterpolatingMotionEffect` 可以通过陀螺仪监测手机的倾斜情况。我们可以通过它设置视图响应的运动。

其中有两个属性：`minimumRelativeValue` 和 `maximumRelativeValue`。这两个属性控制图像运动的最大范围。

实例：

```objc
CGFloat effectOffset = 100.f;
// 设置x方向上的移动量
UIInterpolatingMotionEffect *effectX = [[UIInterpolatingMotionEffect alloc] initWithKeyPath:@"center.x" type:UIInterpolatingMotionEffectTypeTiltAlongHorizontalAxis];
effectX.maximumRelativeValue = @(effectOffset);
effectX.minimumRelativeValue = @(-effectOffset);

// 设置y方向上的移动量
UIInterpolatingMotionEffect *effectY = [[UIInterpolatingMotionEffect alloc] initWithKeyPath:@"center.y" type:UIInterpolatingMotionEffectTypeTiltAlongVerticalAxis];
effectY.maximumRelativeValue = @(effectOffset);
effectY.minimumRelativeValue = @(-effectOffset);

// 将移动量添加到数组中
UIMotionEffectGroup *group = [[UIMotionEffectGroup alloc] init];
group.motionEffects = @[effectX, effectY];

// 设置给 view
[self.view addMotionEffect:group];
```



## UISearchController 实现搜索

iOS8 之后，苹果提供了 `UISearchController` 统一了搜索方式。

首先，需要创建一个用来显示搜索结果的视图 `SearchResultsController`，它需要实现 `UISearchResultsUpdating` 协议，在 `UISearchController` 中的搜索关键字变化的时候，会回调 `UISearchResultsUpdating` 中的 `updateSearchResultsForSearchController:` 方法执行数据的筛选搜索。所以，`SearchResultController` 中还需要两个属性，待搜索的所有数据集合和搜索出的数据集合：

```swift
class SearchResultsController: UITableViewController,UISearchResultsUpdating {
    //搜索出和待搜索的数据
  	var names:[String,[String]] = [String:[String]]()
  	var keys: [String] = []
  	var filteredNames: [String] = []
  	
  	func updateSearchResultsForSearchController(searchController: UISearchController) {
        //搜索过程
      	...
      	//筛选出数据后刷新列表
      	tableView.reloadData()
    }
}
```

 

现在要定义一个跳转前的页面。在某个 `ViewController` 中保存一个 `UISearchController` 的实例。在初始化 ` ViewController` 的同时，初始化 `UISearchController` 并拿到 `UISearchController` 中的 `searchBar` 的实例，将其添加到 `ViewController` 的视图中去（为了点击后跳转时，searchBar 的动画效果）。初始化 `UISearchController` 的时候，将搜索结果展示页 `resultsController` 传入，并将其赋给 `searchResultsUpdater` 属性(通过这个属性调用的 `updateSearchResultsForSearchController:` 方法):

```swift
class ViewController: UIViewController {
    var searchController: UISearchController!
  	...
	override func viewDidLoad() {
        super.viewDidLoad()
      	...
      	let resultsController = SearchResultsController()
      	resultsController.names = names
      	resultsController.keys = keys
      	searchController = UISearchController(searchResultsController: resultsController)
      	let searchBar = searchController.searchBar
		searchBar.scopeButtonTitles = ["All","Short"]
      	searchBar.placeholder = "Enter a search item"
      	searchBar.sizeToFit()
      	tableView.tableHeaderView = searchBar
      	searchController.searchResultsUpdater = resultsController
    }
}
```

其中 `scopeButtonTitles` 是可选的，在 `searchBar` 下显示用来进一步筛选的，可以通过 `searchController.searchBar.selectedScopeButtonIndex` 来获取筛选信息。另外，`searchController.searchBar` 要确定通过 `sizeToFit()` 方法确定了大小后，加入到 `ViewController` 中。

当然，这是最基本的一个流程，还有一些自定义的操作以及一些代理方法

