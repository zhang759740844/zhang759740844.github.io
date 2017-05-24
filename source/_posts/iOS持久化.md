title: iOS 持久化方式
date: 2017/2/3 11:07:12
categories: iOS
tags:

	- 学习笔记
---

开发的时候总是要保存一些东西在磁盘上的，那么有哪些方式来完成持久化呢？

<!--more-->

## NSUserDefaults

对于应用来说，每个用户都有自己的独特偏好设置，而好的应用会让用户根据喜好选择合适的使用方式，把这些偏好记录在应用包的 `plist` 文件中，通过 `NSUserDefaults` 类来访问，这是 `NSUserDefaults` 的常用姿势。如果有一些设置你希望用户即使升级后还可以继续使用，比如玩游戏时得过的最高分、喜好和通知设置、主题颜色甚至一个用户头像，那么你可以使用 `NSUserDefaults` 来存储这些信息。

### 使用方法

`NSUserDefaults` 是 iOS 系统提供的一个单例类，通过类方法 `standardUserDefaults` 可以获取 `NSUserDefaults` 单例:

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
```

`NSUserDefaults` 单例以 key-value 的形式存储了一系列偏好设置，key 是名称，value 是相应的数据。存/取数据时可以使用方法 `objectForKey:` 和 `setObject:forKey: ` 来把对象存储到相应的 `plist` 文件中，或者读取，既然是 `plist` 文件，那么对象的类型则必须是 `plist` 文件可以存储的类型，包括：

- `NSData`
- `NSString`
- `NSNumber`
- `NSDate`
- `NSArray`
- `NSDictionary`

而如果需要存储  `plist` 文件不支持的类型，比如图片，可以先将其归档为 `NSData` 类型，再存入 `plist` 文件，需要注意的是，即使对象是 `NSArray` 或 `NSDictionary` ，他们存储的类型也应该是以上范围包括的。

### 一些方法

`NSUserDefaults` 提供了若干简便方法可以存储某些常用类型的值，例如：

```objective-c
- setBool:forKey:
- setFloat:forKey:
- setInteger:forKey:
- setDouble:forKey:
- setURL:forKey:
- setObject:forKey:
```

这里只列举了几个 set 方法。诸如 `NSString`,`NSData`,`NSArray`,`NSDictionary` 等都使用 `setObject:forKey:` 方法设置。

```objc
- boolForKey:
- floatForKey:
- integerForKey:
- doubleForKey:
- urlForKey:
- stringForKey:
- dataForKey:
- arrayForKey:
- dictionaryForKey:
```

需要注意，这里方法不是一一对应的，上面使用 `setObject:forKey:` 的，在获取的时候，都使用了具体的类型，读取的时候最好使用明确类型的方法，而不是 `objectForKey:`.

> 使用获取方法的时候一定要用明确类型的方法，否则会产生不已排查的意想不到的错误！！！



### 举例

存一个整数、字符串和一张图片：

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
//字符串
[defaults setObject:@"jack" forKey:@"firstName"];
//整数
[defaults setInteger:10 forKey:@"Age"];
//图片
UIImage *image =[UIImage imageNamed:@"somename"];
NSData *imageData = UIImageJPEGRepresentation(image, 1);//把image归档为NSData
[defaults setObject:imageData forKey:@"image"];
```

**UIImageJPEGRepresentation** 有两个实参，一个是 **UIImage**，另一个是浮点数变量，代表压缩质量。压缩质量的值必须在0到1之间，1代表最高质量即不压缩。

对应的读取方法：

```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
//字符串
NSString *firstName = [defaults objectForKey:@"firstName"]
//数字
NSInteger age = [defaults integerForKey:@"Age"];
//图片
NSData *imageData = [defaults dataForKey:@"image"];
UIImage *image = [UIImage imageWithData:imageData];
```

## plist 的使用

上面的 `NSUserDefaults` 是基于 `plist` 的。可以用来存储城市列表等类似数据。那么如何直接操作 `plist` 呢？

### 写入

通过 `writeToFile:atomically:` 上面提到的 `plist` 能够存储的类型都有这个方法。

```objc
    //获取应用程序沙盒的Documents目录  
    NSArray *paths=NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,NSUserDomainMask,YES);  
    NSString *plistPath1 = [paths objectAtIndex:0];  

    //得到完整的文件名  
    NSString *filename=[plistPath1 stringByAppendingPathComponent:@"test.plist"];  

    //输入写入  data 是个 dictionary
    [data writeToFile:filename atomically:YES];  
```

### 读取

通过 `initWithContentsOfFile:` 方法初始化读取，上面提到的类型都有该方法：

```objc
//那怎么证明我的数据写入了呢？读出来看看  
//这里的 filename 是完整的路径和文件名
NSMutableDictionary *data1 = [[NSMutableDictionary alloc] initWithContentsOfFile:filename]; 
```

## 归档

### 协议

归档可以实现对非上述类型对象的存储。但是必须遵守 `NSCoding` 协议。该协议有两个方法：

```objc
@protocol NSCoding

- (void)encodeWithCoder:(NSCoder *)aCoder;
- (id)initWithCoder:(NSCoder *)aDecoder;

@end
```

实现实例：

```objc
//.h文件
#import <Foundation/Foundation.h>

// 给自定义类归档，首先要遵守NSCoding协议。
@interface Person : NSObject<NSCoding>

@property (nonatomic,strong) NSString *name;
@property (nonatomic,assign) NSInteger age;
@property (nonatomic,strong) NSString *gender;

@end
  
  
//.m文件
#import "Person.h"

@implementation Person

// 归档方法
- (void)encodeWithCoder:(NSCoder *)aCoder
{
    [aCoder encodeObject:self.name forKey:@"name"];
    [aCoder encodeInteger:self.age forKey:@"age"];
    [aCoder encodeObject:self.gender forKey:@"gender"];
}
// 反归档方法
- (instancetype)initWithCoder:(NSCoder *)aDecoder
{
    self = [super init];

    if (self != nil) {
        self.name = [aDecoder decodeObjectForKey:@"name"];
        self.age = [aDecoder decodeIntegerForKey:@"age"];
        self.gender = [aDecoder decodeObjectForKey:@"gender"];
    }
    return self;
}

@end
```

这里可以在使用的时候，通过代码提示选取合适的 `encode` 和 `decode` 方法。

如此就能对 `Person` 对象进行归档了。当然这种归档方法非常傻瓜。因为每一个属性都要设置一次 `encode` 和 ` decode`。我们可以通过 runtime，实现自动归档。具体的实现方法在 “runtime 应用” 一篇有说明。

### 使用

对于所有系统类型（**NSString、NSDictionary、NSArray、NSNumber等**）以及所有实现了上面协议的类型，通过下面两个方法编码（NSKeyedArchiver）和解码（NSKeyedUnarchiver）

```objc
BOOL success = [NSKeyedArchiver archiveRootObject:data toFile:filePath];
id data = [NSKeyedUnarchiver unarchiveObjectWithFile:filePath];
```

实现实例：

```objc
- (void)viewDidLoad {
    [super viewDidLoad];

    Person *person = [[Person alloc]init];

    person.name = @"BigBaby";
    person.age = 16;
    person.gender = @"男";

    // 归档，调用归档方法
    NSString *filePath = [NSHomeDirectory() stringByAppendingString:@"person.plist"];
    BOOL success = [NSKeyedArchiver archiveRootObject:person toFile:filePath];
    NSLog(@"%d",success);

    // 反归档，调用反归档方法
    Person *per = [NSKeyedUnarchiver unarchiveObjectWithFile:filePath];
    NSLog(@"%@",per);

    // Do any additional setup after loading the view, typically from a nib.
}
```

## Realm

最强大的跨平台移动端数据库，但是暂时用不到，用到再说。