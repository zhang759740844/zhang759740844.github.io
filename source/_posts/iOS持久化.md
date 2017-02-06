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
```

这里只列举了几个 set 方法，get 方法类似。

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
NSData *imageData = UIImageJPEGRepresentation(image, 100);//把image归档为NSData
[defaults setObject:imageData forKey:@"image"];
```

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

