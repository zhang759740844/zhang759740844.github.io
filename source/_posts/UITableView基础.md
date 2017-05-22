title: UITableView 基础
date: 2016/8/30 14:07:12  
categories: iOS 
tags: 
	- 基本控件
---

UITableView是最常用的基本控件。此处对其一般用法进行总结。

<!--more-->


## UITableView基础
### UITableView 的样式
1. UITableViewStylePlain		将会保持在顶部直到被顶掉
2. UITableViewStyleGrouped将会随着cell一起滚动

```objc
UITableView *tableView = [[UITableView alloc] initWithFrame:CGRectZero style:UITableViewStyle];
```

其中`CGRectZero`表示`equivalent to CGRectMake(0, 0, 0, 0)`.之后代码会改UITableView的Frame，所以暂且都是0。

### UITableView对象提供数据
UITableView不包含任何数据，需要提供一个数据源.  
我们将BNRAppDelegate实例设置为UITableView对象的数据源。BNRAPPDelegate必须遵循UITableViewDataSource协议。  
在BNRAPPDelegate.h文件中，声明BNRAPPDelegate遵循UITableViewDataSource协议
```objc
@interface BNRAppDelegate: UIResponder <UIApplicationDelegate,UITableViewDataSource>
@property (nonatomic) UITableView *taskTable;
@property (nonatomic) NSMutable Array *tasks;
@end
```

在.m中向UITableView发送`setDataSource`消息，将BNRAPPDelegate实例设置为数据源

```objc
self.taskTable.dataSource = self;
```

UITableViewDataSource设置了两个必须方法：

1. 根据指定的表格索引给出相应表格段包含的行数（`tableView：numberOfRowsInSection：`）
2. 根据指定表格段索引和行索引给出相应的UITableViewCell对象（`tableView：cellForRowAtIndexPath：`）
```objc
@implementation BNRAppDelegate
-(NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section{
	    return [self.tasks count];
}
-(UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
        UITableViewCell *c = [self.taskTable dequeueReusableCellWithIdentifier:@“cell” forIndexPath:indexPath];
	    //重用cell
	    NSString *item = [self.tasks objectAtIndex:indexPath.row];
	    c.textLabel.text = item;
	    return c;
}
```

刷新表格：`[self.taskTable reloadData];`

> 使用 `dequeueReuseableCellWithIdentifier:forIndexPath:` 必须要注册 `registerClass` ，但是不用判断是否非空以及手动创建。
>
> 使用 `dequeueReuseableCellWithIdentifier:` 可以免去注册，但是需要手动判断是否为空，为空的话还要手动创建

### 重用UITableViewCell对象
需要将自定义的cell类和identifier进行关联。  
在ViewController.m中覆盖viewDidLoad方法，向表视图注册应该使用的UITabeViewCell
```objc
-(void)viewDidLoad{
	[super viewDidLoad];
	[self.tableView registerClass:[UITableViewCell class] forCellReuseIdentifier:@“UITableViewCell”];
}
```

这样UITableViewCell类就和这个string关联起来了。也可以使用nib文件关联
```objc
[self.tableView registerNib:[UINib nibWithNibName:@"MineUserInfoCell" bundle:nil]  forCellReuseIdentifier:@"MineUserInfoCellIdentifier"];
```

> 纯代码用 `registerClass`，有 xib 用 `registerNib`.
>
> 代码创建，使用  `registerClass:` 注册, dequeue时会调用 cell 的 `- (id)initWithStyle:withReuseableCellIdentifier:`
>
> 若使用nib，使用  `registerNib:` 注册，dequeue时会调用 cell 的`-(void)awakeFromNib`

### 使用UIViewController创建tabeView

需要自己创建tableview属性：
```objc
@property (nonatomic, strong) UITableView *tableView;
```
这里要用strong ，因为这个tableview是自己创建的。如果是weak那么指向的地方在创建后就被ARC回收了,那么这个tableview指针就成了野指针了。
如果是连线的就可以用weak，weak属性一般用在当前属性是其他类创建，只存一个该属性的引用的时候，为了强引用那个类。

### 创建多个section的tableview
创建多个section的tableview需要实现方法`numberOfSectionsInTableView:`返回tableview中section的个数。其余使用和单个section一样。

在`cellForRowAtIndexPath:`方法中，需要区分不同section返回不同cell。区分section的方式在于该方法传入的参数`(NSIndexPath *)indexPath`。

`NSIndexPath`是一个结构体，具有两个属性`row`和`section`。表示所在section和section内row。
`NSIndexPath`中的属性是只读的，不能直接修改。只能通过重新创建的形式，达到修改属性的目的。
创建方式:
```objc
NSIndexPath *indexPath = [NSIndexPath indexPathForRow:1 inSection:1];
```

### TableView表头视图
表头视图是指UITableView对象可以在其表格上方显示的特定视图，是和放置针对某个表格段或整张表格的标题和控件。表头视图可以是任意UIView对象。表格视图有两种，分别针对表格段和表格。类似的，还有表尾视图。
```objc
UIView *headerView = [[[NSBundle mainBundle] loadNibNamed:@"HotelReviewsHeaderView" owner:nil options:nil]lastObject];
```

**loadNibNamed:owner:options:**返回的是个数组，保存了xib中的各个view。xib中有几个view，数组元素就是几。因此，可以将多个自定义的view或者cell放在一个xib中，通过数组的方式获取想要的view。`initWithNibName`的实现和该方法类似，其中也会用到该方法。不过`initWithNibName`用在获取Controller的xib中。

```objc
- (void)viewDidLoad{
	UIView *header = self.headerView;
	[self.tableView setTableHeaderView:header];
}
```
加载完headerView后，将其设置为UITableView对象的表头视图。
也可以在 **(UIView \*)tableView:viewForHeaderInSection:**方法中设置，当只有一个section时效果相同。
```objc
- (UIView *)tableView:(UITableView *)tableView viewForHeaderInSection:(NSInteger)section{
    UIView *view = [[[NSBundle mainBundle] loadNibNamed:@"HeaderView" owner:nil options:nil] lastObject];
    return view;
}
```

### tableview的字母索引
实现`sectionIndexTitlesForTableView`方法，返回一个字符串数组：
```objc
- (NSArray<NSString *> *)sectionIndexTitlesForTableView:(UITableView *)tableView{
    NSMutableArray *indexs = [[NSMutableArray alloc]init];
    [indexs addObject:@"我"];
    [indexs addObject:@"是"];
    return indexs;
}
```
这个返回数组的内容和各个section对应，点击索引，就能跳转到对应section。

### 点击cell中button获取所属indexpath
button点击事件中有一个event对象，记录了当前点击坐标，然后通过坐标能够找到所属的cell。可以在cell中为点击事件设置代理，然后在viewcontroller中实现。当然，更推荐的是直接在viewcontroller中添加和实现点击事件。
```objc
[cell.btn addTarget:self action:@selector(cellBtnClicked:event:) forControlEvents:UIControlEventTouchUpInside];

- (void)cellBtnClicked:(id)sender event:(id)event
{
    NSSet *touches =[event allTouches];
    UITouch *touch =[touches anyObject];
    CGPoint currentTouchPosition = [touch locationInView:_tableView];
    NSIndexPath *indexPath= [_tableView indexPathForRowAtPoint:currentTouchPosition];
    if (indexPath!= nil)
    {
        // do something
    }
}
```

可以通过`event`拿到在tableView中的位置`cure`，再通过`indexPathForRowAtPoint:`方法获取`NSIndexPath`。

另外一种方法是，在cell中使用代理，直接将cell传给viewcontroller：
```objc
//cell.h中的声明
- (IBAction)buttonPressed :(id)；

//cell.m中的实现，设置代理
- (void)buttonPressed:(id)sender{
    [self.delegate buttonPressed:self event:event];
}

//viewcontroller中的实现
- (void)buttonPressed:(TableViewCell1 *)cell{
    NSIndexPath *indexPath2 = [_tableView indexPathForCell:cell];
    NSLog(@"所属行数：%ld",(long)indexPath2.row+1);
}
```

通过设置delegate，将button的点击事件交给viewController完成。

> 注意，如果是那种点击 button 后删除的需求，在调用 `deleteItemsAtIndexPaths:` 之前，一定要先删除数据源里的数据，因为`deleteItemsAtIndexPaths:` 调用后，会刷新部分数据。

[iOS高级开发——CollectionView的动态增删cell及使用模型重构](http://doc.okbase.net/CHENYUFENG1991/archive/193953.html)

## 编辑UITableView
### 编辑模式下的UITableView
#### 进入编辑模式
通过调用`[_tableView setEditing:!_tableView.isEditing animated:true]`进入编辑模式,可实现添加，删除，移动操作。
默认是删除，即cell左边出现一个红色的减号，点击可以删除该行。

#### TableView 编辑行
使用**setEditing:animated:**方法让tableView进入编辑模式.可以使用**tableView:canEditRowAtIndexPath**方法筛选能进入编辑模式的行：
```objc
- (BOOL)tableView:(UITableView *)tableView canEditRowAtIndexPath:(NSIndexPath *)indexPath {
    if(indexPath.row == (10 | 12 | 14)){
        return NO;
    }else{
        return YES;
    }
}
```
如果不实现该方法，默认为YES。

#### 编辑模式
通过设置`UITableViewCellEditingStyle`可以切换进入的编辑模式是实现插入还是删除操作。这个返回值将作为下面的`commitEditingStyle:forRowAtIndexPath:`方法的入参传入。
```objc
-(UITableViewCellEditingStyle)tableView:(UITableView *)tableView editingStyleForRowAtIndexPath:(NSIndexPath *)indexPath{
    if (condition) {
        return UITableViewCellEditingStyleInsert;
    }else{
   		return UITableViewCellEditingStyleDelete;
	}
}
```

#### 编辑模式下的插入和删除行
实现`tableView:commitEditingStyle:forRowAtIndexPath:`方法。传入三个参数。  
第一个实参是发送该消息的UITableView对象。  
第二个实参是`UITableViewCellEditingStyle`类型的常数(删除表格行时，传入的是`UITableViewCellEditingStyleDelete`;插入表格行时，传入的是`UITableViewCellEditingStyleInsert`)。  
第三个实参是一个NSIndexPath对象。
```objc
- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        [self.dataSource removeObjectAtIndex:indexPath.row];
        [self.tableView deleteRowsAtIndexPaths:[NSArray arrayWithObject:indexPath] withRowAnimation:UITableViewRowAnimationFade];
    }else if (editingStyle == UITableViewCellEditingStyleInsert){
    	[self.dataSource insertObject:@"我是新来的" atIndex:indexPath.row];
        [_tableView insertRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationBottom];
    }
}
```
使用`deleteRowsAtIndexPaths`和`insertRowsAtIndexPaths`可以进行局部刷新，节省资源，并且还能添加指定动画。
delete操作可以不在编辑模式的情况下，通过左滑cell直接触发。插入则必须进入编辑模式。

#### cell的移动
进入编辑模式后
实现`tableView:moveRowAtIndexPath:`方法
```objc
- (void)tableView:(UITableView *)tableView moveRowAtIndexPath:(NSIndexPath *)sourceIndexPath toIndexPath:(NSIndexPath *)destinationIndexPath{
    if(sourceIndexPath == destinationIndexPath){
        return ;
    }else{
        Comment *comment = [self.dataSource objectAtIndex:sourceIndexPath.row];
        [self.dataSource removeObjectAtIndex:sourceIndexPath.row];
        [self.dataSource insertObject:comment atIndex:destinationIndexPath.row];
    }
}
```
一定要对数据源进行正确操作。

#### TableView 修改删除按钮
```objc
- (NSString *)tableView:(UITableView *)tableView titleForDeleteConfirmationButtonForRowAtIndexPath:(NSIndexPath *)indexPath{
    return @"删除";
}
```

### 不在编辑模式下的编辑方式
#### 刷新方式
简单总结一些UITableView的刷新方法：
- reloadData									刷新整个表格
- reloadRowsAtIndexPaths:withRowAnimation:刷新indexPath指向的cell
- reloadSections:withRowAnimation:                刷新NSIndexSet内包含的Section

这三个分别刷新tableview的各个部分
第一个没有动画效果。
第二个可以传入一个数组
```objc
[_tableView reloadRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationTop];
```
第三个可以传入一个NSIndexSet集合
```objc
[self.tableView reloadSections:[NSIndexSet indexSetWithIndexesInRange:NSMakeRange(0, 2)] withRowAnimation:UITableViewRowAnimationFade];
[_tableView reloadSections:[NSIndexSet indexSetWithIndex:indexPath.section]withRowAnimation:UITableViewRowAnimationLeft];
```

#### 插入删除
在编辑模式下的插入删除方式，可以直接用在任何想用的场景下，只需要调用一下方法，并且保持dataSource正确即可。
- deleteRowsAtIndexPaths:withRowAnimation:
- insertRowsAtIndexPaths:withRowAnimation:

就不举例了。同上面一样。

## 小技巧
### 当cell未能填满tableview时，怎么响应空白部分点击事件
当 tableview 太大，cell 太少，以至于不能填满tableview的时候，那么空白部分的点击事件该怎么设置呢？只要给 tableview 添加一个 footerview，这个 footerview 的大小是整个 tableview 的大小，然后设置这个 footerview 的点击事件即可。

为什么能这样呢？因为 cell 没有填充满的部分都用 footerview 填充了。

### UITableView 取消点击cell的选中背景颜色

#### 方法一

在选中的回调方法中取消选中：

```objc
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:NO];
}
```

#### 方法二

设置 cell 的选中属性：

```objc
-(UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{
  ...
  cell.selectionStyle = UITableViewCellSelectionStyleNone;
  ...
}
```

####  cell 的点击事件不响应

有可能是在 cell 里设置了 `UIButton`，由于响应机制，先响应 `UIButton` 的点击事件。所以就忽略了 cell 的 `didSelect` 方法。

所以要做的就是将 `UIButton` 的 `UserInteractionEnabled ` 设置为 No。






