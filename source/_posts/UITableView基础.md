

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

#### 设置可以编辑的行
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

#### 设置编辑模式
通过设置`UITableViewCellEditingStyle`可以切换进入的编辑模式是实现插入还是删除操作。这个返回值将作为下面的`commitEditingStyle:forRowAtIndexPath:`方法的入参传入。
```objc
-(UITableViewCellEditingStyle)tableView:(UITableView *)tableView editingStyleForRowAtIndexPath:(NSIndexPath *)indexPath{
    if (具体情况) {
        return UITableViewCellEditingStyleInsert;
    }else{
   		return UITableViewCellEditingStyleDelete;
	}
}
```

#### 设置编辑模式下的操作
实现`tableView:commitEditingStyle:forRowAtIndexPath:`这个统一的回调方法，无论是添加还是删除都会执行，需要自己根据入参区分开。第二个实参是`UITableViewCellEditingStyle`类型的常数(删除表格行时，传入的是`UITableViewCellEditingStyleDelete`;插入表格行时，传入的是`UITableViewCellEditingStyleInsert`)。  
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

### 侧滑菜单

许多 app 提供侧滑某一栏展示菜单的功能。这在 iOS8 中有了默认的实现：

```objc
//设置cell可编辑状态
-(BOOL)tableView:(UITableView *)tableView canEditRowAtIndexPath:(NSIndexPath *)indexPath{
    return YES;
}

//定义编辑样式为删除样式
-(UITableViewCellEditingStyle)tableView:(UITableView *)tableView editingStyleForRowAtIndexPath:(NSIndexPath *)indexPath{
    return UITableViewCellEditingStyleDelete;
}

//设置返回存放侧滑按钮数组
-(NSArray<UITableViewRowAction *> *)tableView:(UITableView *)tableView editActionsForRowAtIndexPath:(NSIndexPath *)indexPath{
    
  //这是iOS8以后的方法
    UITableViewRowAction *deleBtn = [UITableViewRowAction rowActionWithStyle:UITableViewRowActionStyleDestructive title:@"删除" handler:^(UITableViewRowAction * _Nonnull action, NSIndexPath * _Nonnull indexPath) {
       
        [_messageData removeObjectAtIndex:indexPath.row];
        
        [tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationBottom];
        
    }];
    
    
    UITableViewRowAction *moreBtn = [UITableViewRowAction  rowActionWithStyle:UITableViewRowActionStyleDestructive title:@"更多更多" handler:^(UITableViewRowAction * _Nonnull action, NSIndexPath * _Nonnull indexPath) {
       
        NSLog(@"更多，，点了");
        
    }];
    
   
    UITableViewRowAction *upBtn = [UITableViewRowAction rowActionWithStyle:UITableViewRowActionStyleDestructive title:@"置顶" handler:^(UITableViewRowAction * _Nonnull action, NSIndexPath * _Nonnull indexPath) {
       
        [_messageData exchangeObjectAtIndex:indexPath.row withObjectAtIndex:0];
        
        NSIndexPath *firstIndexPath = [NSIndexPath indexPathForRow:0 inSection:indexPath.section];
        
        [tableView moveRowAtIndexPath:indexPath toIndexPath:firstIndexPath];
        
    }];
    
    //设置背景颜色，他们的大小会分局文字内容自适应，所以不用担心
    deleBtn.backgroundColor = [UIColor redColor];
    
    moreBtn.backgroundColor = [UIColor orangeColor];
    
    upBtn.backgroundColor = [UIColor grayColor];
    
    
    return @[deleBtn,moreBtn,upBtn];
    
}
```

还是设置为可编辑，然后设置编辑样式为删除样式。不同的是，需要实现一个侧滑的专用回调方法。需要创建多个 `UITableViewRowAction` 对象，你可以像 button 一样设置它们，不过在初始化的时候需要事先设置好毁掉方法。

### 代码控制的编辑方式

上面说的都是交互情况下的编辑方式。我们可以自己通过代码控制刷新视图，不需要交互。

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

#### beginUpdates和endUpdates

如果要同时进行刷新或者插入删除移动等操作，需要使用 `beginUpdates` 和 `endUpdates` 包裹，作为一个动画组进行。

如果要自己控制动画的时间可以通过动画的方式：

```objc
[UIView animateKeyframesWithDuration:1 delay:0 options:UIViewKeyframeAnimationOptionAutoreverse animations:^{
    } completion:^(BOOL finished) {
    	// 添加Transaction事务
        [CATransaction begin];
        [CATransaction setCompletionBlock:^{
            NSLog(@"动画完成")
        }];
        [self.tableView beginUpdates];
        [self.tableView moveRowAtIndexPath:[NSIndexPath indexPathForRow:0 inSection:0] toIndexPath:[NSIndexPath indexPathForRow:2 inSection:0]];
        NSString *str = self.arrayData[0];
        [self.arrayData removeObjectAtIndex:0];
        [self.arrayData insertObject:str atIndex:2];
        [self.tableView endUpdates];
        [CATransaction commit];
    }];
```

`beginUpdates` 和 `endUpdates` 有一个好处就是会调用 `heightForRow` 改变高度，但是不会调用 `cellForRow` 刷新视图。所以，当 tableView 的某一项高度改变的时候，可以直接使用该方法刷新高度。

```objc
// 刷新 tableView 中 cell 的高度
self.tableView.beginUpdates()
self.tableView.endUpdates()
```



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



### 使用 UILocalizedIndexedCollation 对数据源分组

比如一个联系人的 tableView，我们需要把联系人按照首字母分组显示。iOS 就提供了一个 `UILocalizedIndexedCollation` 帮助我们进行分组。

#### 属性与方法

`UILocalizedIndexedCollation` 需要被记住的属性和方法就这么几个：

```objc
// 获得单例 UILocalizedIndexedCollation 的实例
+ (instancetype)currentCollation;
// 字符索引数组(其实内容是固定的就是 A-Z,加上一个代表其他的 #)
@property(nonatomic, readonly) NSArray<NSString *> * sectionTitles;
// 输入对象以及一个返回字符串的对象方法，根据返回的 int 判断字符串要加入哪个分组
- (NSInteger)sectionForObject:(id)object collationStringSelector:(SEL)selector;
// 数组内对象排序
- (NSArray *)sortedArrayFromArray:(NSArray *)array collationStringSelector:(SEL)selector;
```

除了上面的属性和方法还有一个属性和一个方法用不到，不用管它。

#### 实例

比如要实现一个需求是对联系人进行归类，然后以 tableView 的形式展示。特殊的地方时，只展示有值的分组。

##### 初始化数组

```objc
// 单例对象 
UILocalizedIndexedCollation *localIndex = [UILocalizedIndexedCollation curre ntCollation]; 
// 获得当前语言下的所有的indexTitles 
_allIndexTitles = localIndex.sectionTitles; 
// 初始化所有数据的数组 (之后的所有进过分组的数据都保存在这个 _data 里)
_data = [NSMutableArray arrayWithCapacity:_allIndexTitles.count]; 
// 为每一个indexTitle 生成一个可变的数组 
for (int i = 0; i<_allIndexTitles.count; i++) {
	// 初始化数组 (_data 里的元素是各个分组，所以也是数组)
	[_data addObject:[NSMutableArray array]]; 
} 
// 初始化有效的sectionIndexs (之后有值的分组的索引会被保存在这里)
_sectionIndexs = [NSMutableArray arrayWithCapacity:_allIndexTitles.count];
```

这里要强调的是 `_data` 是不管分组有没有数据，都创建并保存到其中了。而 `_sectionIndexs` 只保存有数据的分组的索引。

##### 分组

```objc
SEL nameSelector = @selector(name); 
for (Person *person in persons) { 
  	if (person == nil) continue;
  
	// 获取到这个contact的name的首字母对应的indexTitle 
  	// 注意这里必须使用对象，这个selector也是有要求的 
  	// 必须是这个对象中的selector, 并且不能有参数，必须返回字符串
  	// 所以这里直接使用 name 属性的get方法就可以 
  	NSInteger index = [localIndex sectionForObject:person collationStringSe lector:nameSelector];
	
  	// 处理多音字 例如 "曾" -->> 会被当做 ceng 来处理，其他需要处理的多音字类似 
  	if ([person.name hasPrefix:@"曾"]) { 
      	index = [_allIndexTitles indexOfObject:@"Z"]; 
    } 
  	// 将这个contact添加到对应indexTitle的数组中去 
  	[_data[index] addObject:person];
}
```

这里就通过 `UILocalizedIndexedCollation` 提供的方法将数据源分组了。上面的注释也写到了，传入一个对象以及对象的方法，通过方法返回的字符串获取所在分组的索引。获取到的索引你可以做进一步判断修改。然后通过这个索引将其放到 `_data` 的对应分组内。

##### 遍历分组

```objc
for (int i=0; i<_data.count; i++) { 
	NSArray *temp = _data[i]; 
  	if (temp.count != 0) { 
      	// 取出不为空的部分对应的indexTitle 
      	[_sectionIndexs addObject:[NSNumber numberWithInt:i]]; 
    } 
  	// 排序每一个数组 
  	_data[i] = [localIndex sortedArrayFromArray:temp collationStringSelecto r:nameSelector]; 
}
```

上一节将数据源全都塞到了对应的分组里。现在就要将分组里的数据排序，并且剔除没有数据的分组了。

判断 `_data` 中有数据的分组的索引保存到 `_sectionIndexs` 中。这样以后拿着 `_sectionIndexs` 中保存的索引就可以到 `sectionTitles` 中，找到索引对应的值。

##### 结合字母索引展示

现在我们已经获得了完整的 `_data` 和 `_sectionIndexs`。只需要通过 `_sectionIndexs` 中保存的索引找到 `_data` 中相应的分组即可完成展示。