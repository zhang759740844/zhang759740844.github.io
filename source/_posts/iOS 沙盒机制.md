title: iOS 文件下载与处理相关
date: 2017/3/30 16:07:12  
categories: iOS
tags:

	- 学习笔记
---

iOS 文件下载与处理相关

<!--more-->

## iOS 沙盒机制

每个应用程序的活动范围都限定在自己的沙盒里。不能随意跨越自己的沙盒去访问别的应用程序沙盒中的内容。

### 沙盒中的目录类型

#### 详解

就像 url 一样，每个手机中的 app 都有一个独一无二的路径，通过这个路径，我们能拿到 app 中保存的资源。在根路径下保存着几个特定的文件夹，下面将介绍这几个文件夹：

- Document:保存应用运行时生成的需要持久化的数据。建议将在应用程序中**只有用户生成的文件**、**应用程序不能重新创建的文件**保存在该文件夹。iCloud 会自动备份这个文件夹。
- Library:该文件夹下又有两个子文件夹。
  - Caches:**可以重新下载**或者**重新生成的数据**应该保存在该目录下，该文件夹下的文件不会因为退出而清除。iCloud 不会自动备份该文件夹
  - Preferences:保存应用程序的所有偏好设置 iOS 的 Settings，我们不应该直接在这里创建文件，而是需要通过 `NSUserDefault` 这个类来访问应用程序的偏好设置。
- Tmp:保存临时使用的数据，其中的数据会在应用退出后清除。

根路径一般形式如下，各个文件夹都保存在该目录下:

```
/var/mobile/Containers/Data/Application/一串随机字符用以和其他app区分开
```

除了上面的提到的根目录，随着 app 一起打包的资源文件都会被放在一个 bundle 目录下，路径一般为：

```
/var/containers/Bundle/Application/随机字符串，和上面的不同/应用名.app
```

资源文件都是通过这个根路径拼接获得的，比如要得到一个 `www` 目录下的 `test.js` 文件，那么路径就是

```
/var/containers/Bundle/Application/随机字符串，和上面的不同/应用名.app/www/test.js
```

#### 注意

如果你做个记事本的app，那么用户写了东西，总要把东西存起来。那么这个文件则是用户自行生成的，就放在documents文件夹里面。

如果你有一个app，需要和服务器配合，经常从服务器下载东西，展示给用户看。那么这些下载下来的东西就放在 library/cache。

> 在 cache 目录下的文件在存储空间不足的情况下，会被清空。所以一些重要的文件最好不要放在 cache 中。

apple对这个很严格，放错了就会被拒。主要原因是ios的icloud的同步问题。

### 获取沙盒路径

#### 获取沙盒的 Home 目录

```objc
//获取根目录 
NSString *homePath = NSHomeDirectory(); 
NSLog(@"Home目录：%@",homePath);
```

#### 获取沙盒的 Documents 目录

```objc
NSString *filePath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,NSUserDomainMask,YES) firstObject];
```

#### 获取 Library 文件路径

```objc
NSString *filePath = [NSSearchPathForDirectoriesInDomains(NSLibraryDirectory,NSUserDomainMask,YES) firstObject];
```

#### 获取 Caches 文件路径

```objc
NSString *filePath = [NSSearchPathForDirectoriesInDomains(NSCachesDirectory,NSUserDomainMask,YES) firstObject];
```

#### 获取 Tmp 文件路径

```objc
NSString *filePath = NSTemporaryDirectory();
```



其中 `NSUserDomainMask` 表示在当前沙盒范围内查找，`YES` 表示展开路径，`NO` 表示不展开路径。**里面的所有文件夹都可以通过 Home 目录拼接而成**。

沙盒里还有很多其他的文件夹，上面只是列举了常用的几个。

####  获取 Bundle 文件路径

```objc
NSString *bundle = [NSBundle mainBundle];
```



## 文件下载

### 文件操作

#### 创建文件夹

在 Document 目录下创建 test 文件夹：

```objc
//创建文件夹  
-(void)createDir{  
    NSString *documentsPath =[self dirDoc];  
    NSFileManager *fileManager = [NSFileManager defaultManager];  
    NSString *testDirectory = [documentsPath stringByAppendingPathComponent:@"test"];  
    // 创建目录  
    BOOL res=[fileManager createDirectoryAtPath:testDirectory withIntermediateDirectories:YES attributes:nil error:nil];  
    if (res) {  
        NSLog(@"文件夹创建成功");  
    }else  
        NSLog(@"文件夹创建失败");  
 }  
```

`NSFileManger`  是一个文件处理的类。 `self dirDoc` 是自定义的获得 Document 文件路径的方法，上面说到过。

#### 创建文件

```objc
//创建文件  
-(void)createFile{  
    NSString *documentsPath =[self dirDoc];  
    NSString *testDirectory = [documentsPath stringByAppendingPathComponent:@"test"];  
    NSFileManager *fileManager = [NSFileManager defaultManager];  
    NSString *testPath = [testDirectory stringByAppendingPathComponent:@"test.txt"];  
    BOOL res=[fileManager createFileAtPath:testPath contents:nil attributes:nil];  
    if (res) {  
        NSLog(@"文件创建成功: %@" ,testPath);  
    }else  
        NSLog(@"文件创建失败");  
}  
```

#### 写数据到文件

```objc
//写文件  
-(void)writeFile{  
    NSString *documentsPath =[self dirDoc];  
    NSString *testDirectory = [documentsPath stringByAppendingPathComponent:@"test"];  
    NSString *testPath = [testDirectory stringByAppendingPathComponent:@"test.txt"];  
    NSString *content=@"测试写入内容！";  
    BOOL res=[content writeToFile:testPath atomically:YES encoding:NSUTF8StringEncoding error:nil];  
    if (res) {  
        NSLog(@"文件写入成功");  
    }else  
        NSLog(@"文件写入失败");  
}
```

#### 从文件读数据

```objc
//读文件  
-(void)readFile{  
    NSString *documentsPath =[self dirDoc];  
    NSString *testDirectory = [documentsPath stringByAppendingPathComponent:@"test"];  
    NSString *testPath = [testDirectory stringByAppendingPathComponent:@"test.txt"];  
//    NSData *data = [NSData dataWithContentsOfFile:testPath];  
//    NSLog(@"文件读取成功: %@",[[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding]);  
    NSString *content=[NSString stringWithContentsOfFile:testPath encoding:NSUTF8StringEncoding error:nil];  
    NSLog(@"文件读取成功: %@",content);  
}  
```

#### 文件大小

```objc
+(float)fileSizeAtPath:(NSString *)path{
    NSFileManager *fileManager=[NSFileManager defaultManager];
    if([fileManager fileExistsAtPath:path]){
        long long size=[fileManager attributesOfItemAtPath:path error:nil].fileSize;
        return size/1024.0/1024.0;
    }
    return 0;
}
```

这个方法获得文件的各个属性，其中一个属性是 `fileSize`。一般清除缓存的时候需要用到计算文件大小。

#### 删除文件

```objc
//删除文件  
-(void)deleteFile{  
    NSString *documentsPath =[self dirDoc];  
    NSString *testDirectory = [documentsPath stringByAppendingPathComponent:@"test"];  
    NSFileManager *fileManager = [NSFileManager defaultManager];  
    NSString *testPath = [testDirectory stringByAppendingPathComponent:@"test.txt"];     
    BOOL res=[fileManager removeItemAtPath:testPath error:nil];  
    if (res) {  
        NSLog(@"文件删除成功");  
    }else  
        NSLog(@"文件删除失败");     
    NSLog(@"文件是否存在: %@",[fileManager isExecutableFileAtPath:testPath]?@"YES":@"NO");  
} 
```

###  NSData 使用

下载得到的文件，一般是以 `NSData` 的形式出现，使用时需要将其转换成相应的类，比如 `NSString`,`UIImage` 等类的形式。

#### NSData 与 NSString

NSData 转 NSString：

```objc
NSString *newStr = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
```

NSString 转 NSData：

```objc
NSData *data = [str dataUsingEncoding:NSUTF8StringEncoding]; 
```

#### NSData 与 UIImage

NSData 转 UIImage：

```objc
UIImage *aimage = [UIImage imageWithData: imageData];
```

UIImage 转 NSData：

```objc
NSData *imageData = UIImagePNGRepresentation(aImage);
```



### 下载

可以通过 `NSURLConnection `类创建下载，不过该类在 iOS9 中被废弃了，取而代之的是 `NSURLSession` 类。我们下面只学习下该类的使用方式和技巧。

#### 普通的 get 请求

创建任务后，不会自动发送请求，需要手动开始任务：

```objc
 	// 1.得到session对象
    NSURLSession* session = [NSURLSession sharedSession];
    NSURL* url = [NSURL URLWithString:@""];

    // 2.创建一个task，任务
    NSURLSessionDataTask* dataTask = [session dataTaskWithURL:url completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        // data 为返回数据
		dispatch_async(dispatch_get_main_queue(), ^{
			[_image setImage:[UIImage imageWithData:data]];
        });
    }];

    // 3.开始任务
    [dataTask resume];
```

> 注意，completionHandler 这个回调是在后台线程中的，如果想要改变UI，就必须dispatch到主线程中去，否则会等待很长时间才能显示出来

#### 下载文件

使用 `NSURLSession` 就不需要边下载边写入等问题，苹果做好了封装：

```objc
 	NSURL* url = [NSURL URLWithString:@"http://dlsw.baidu.com/sw-search-sp/soft/9d/25765/sogou_mac_32c_V3.2.0.1437101586.dmg"];

    // 得到session对象
    NSURLSession* session = [NSURLSession sharedSession];

    // 创建任务
    NSURLSessionDownloadTask* downloadTask = [session downloadTaskWithURL:url completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {

    }];
    // 开始任务
    [downloadTask resume];
```

这里回调参数没有了 `NSData`，多了一个 `location`，这个就是下载好的文件写入沙盒的地址，打印后发现下载好的文件被写入了 temp 文件夹下。

所以我们需要将 temp 目录下的文件转移到不会被删除的 caches 目录下。

```objc
NSString *caches = [NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) lastObject];
// response.suggestedFilename ： 建议使用的文件名，一般跟服务器端的文件名一致
NSString *file = [caches stringByAppendingPathComponent:response.suggestedFilename];

// 将临时文件剪切或者复制Caches文件夹
NSFileManager *mgr = [NSFileManager defaultManager];

// AtPath : 剪切前的文件路径
// ToPath : 剪切后的文件路径
[mgr moveItemAtPath:location.path toPath:file error:nil];
```

 #### 监听下载进度

上面的方法可以下载，但是无法监听下载的进度。想要监听下载进度，需要通过 delegate，要遵循协议`NSURLSessionDownloadDelegate` ：

```objc
/*
 * Messages related to the operation of a task that writes data to a
 * file and notifies the delegate upon completion.
 */
@protocol NSURLSessionDownloadDelegate <NSURLSessionTaskDelegate>

/* Sent when a download task that has completed a download.  The delegate should 
 * copy or move the file at the given location to a new location as it will be 
 * removed when the delegate message returns. URLSession:task:didCompleteWithError: will
 * still be called.
 *  下载完毕会调用
 *
 *  @param location     文件临时地址
 */
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
                              didFinishDownloadingToURL:(NSURL *)location;

@optional
/* Sent periodically to notify the delegate of download progress. 
 *  每次写入沙盒完毕调用
 *  在这里面监听下载进度，totalBytesWritten/totalBytesExpectedToWrite
 *
 *  @param bytesWritten              这次写入的大小
 *  @param totalBytesWritten         已经写入沙盒的大小
 *  @param totalBytesExpectedToWrite 文件总大小
 */
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
                                           didWriteData:(int64_t)bytesWritten
                                      totalBytesWritten:(int64_t)totalBytesWritten
                              totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite;

/* Sent when a download has been resumed. If a download failed with an
 * error, the -userInfo dictionary of the error will contain an
 * NSURLSessionDownloadTaskResumeData key, whose value is the resume
 * data. 
 *  恢复下载后调用，
 */
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask
                                      didResumeAtOffset:(int64_t)fileOffset
                                     expectedTotalBytes:(int64_t)expectedTotalBytes;

@end
```

设置 delegate 回调时，`NSURLSession` 和 `NSURLSessionDownloadTask` 调用方法和上面下载文件中的方法略有不同：

```objc
	// 得到session对象
    NSURLSessionConfiguration* cfg = [NSURLSessionConfiguration defaultSessionConfiguration]; // 默认配置

    NSURLSession* session = [NSURLSession sessionWithConfiguration:cfg delegate:self delegateQueue:[NSOperationQueue mainQueue]];

    // 创建任务
    NSURLSessionDownloadTask* downloadTask = [session downloadTaskWithURL:url];

    // 开始任务
    [downloadTask resume];
```

#### 断点下载

文件的暂停和恢复可以通过 `resume` 和 `suspend` 方法实现。但是这样做，程序退出后再开启就不能接着下载了。一般会使用 `cancelByProducingResumeData:` 方法:

```objc
__weak typeof(self) selfVc = self;
[self.downloadTask cancelByProducingResumeData:^(NSData *resumeData) {
    //  resumeData : 包含了继续下载的开始位置\下载的url
    selfVc.resumeData = resumeData;
    selfVc.downloadTask = nil;
}];
```

调用该取消方法后会回调一个 block，并传入 `resumeData`，该参数包含了继续下载文件的位置信息，你可以将其转换为 `NSString`，打印出来可以看到其实是 plist 的形式，你可以将其保存为 plist 文件。上面 `selfVc.downloadTask = nil` 直接将 `downloadTask` 直接置为了 `nil`，因为后面继续下载的时候会重新创建一个。

继续下载的时候需要重新创建一个与 `resumeData` 相关的`downloadTask`：

```objc
// 传入上次暂停下载返回的数据，就可以恢复下载
self.downloadTask = [self.session downloadTaskWithResumeData:self.resumeData];
[self.downloadTask resume]; // 开始任务
```

具体下载的方法和示例详见：[iOS开发网络篇之文件下载、大文件下载、断点下载](http://www.jianshu.com/p/f65e32012f07) [demo](https://github.com/zhang759740844/MyOCDemo/tree/develop/HFDownLoad-master)











 















