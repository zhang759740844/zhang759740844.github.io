title: AFNetworking 源码解析 - 发送请求
date: 2018/12/23 14:07:12  
categories: iOS
tags: 

- 学习笔记
- 源码解析

------

本篇是 AFNetworking 源码解读的第一篇。主要来了解如何通过 AFNetworking 发送网络请求

<!--more-->

## AFHTTPSessionManager

我们在写需求的时候很少碰到直接使用 `NSURLSession` 的情况，都是使用 `AFHTTPSessionManager` 的封装(虽然我们一般还要再封装一层)。

`AFHTTPSessionManager` 是 `AFURLSessionManager` 的子类，属于对外调用的接口，承担了小部分的功能，主要实现还是在 `AFURLSessionManager` 中

### 初始化

`AFHTTPSessionManager` 的初始化方法最终来到这里：

```objc
- (instancetype)initWithBaseURL:(NSURL *)url
           sessionConfiguration:(NSURLSessionConfiguration *)configuration
{
    self = [super initWithSessionConfiguration:configuration];
    if (!self) {
        return nil;
    }

    // Ensure terminal slash for baseURL path, so that NSURL +URLWithString:relativeToURL: works as expected
    if ([[url path] length] > 0 && ![[url absoluteString] hasSuffix:@"/"]) {
        url = [url URLByAppendingPathComponent:@""];
    }

    self.baseURL = url;

    /// 设置请求和返回的序列化对象
    self.requestSerializer = [AFHTTPRequestSerializer serializer];
    self.responseSerializer = [AFJSONResponseSerializer serializer];

    return self;
}
```

参数接收一个 `NSURLSessionConfiguration` 实例，它是创建 `NSURLSession` 需要的配置项。同时它保存了 `baseURL`，并且创建了请求和响应序列化对象 `requestSerializer` 和 `responseSerializer`。

### 几个 set 方法

初始化的时候，创建了默认的 `AFHTTPRequestSerializer` 和 `AFJSONResponseSerializer`。如果有特殊的 content-type 的请求或者相应，那么就需要定义序列化对象，因此提供了两个 set 方法，分别设置请求和响应序列化对象。

还有一个关于安全策略的 set 方法，用来设置 HTTPS 下，证书的校验策略。

```objc
#pragma mark - 设置请求返回序列化对象
- (void)setRequestSerializer:(AFHTTPRequestSerializer <AFURLRequestSerialization> *)requestSerializer {
    NSParameterAssert(requestSerializer);

    _requestSerializer = requestSerializer;
}

- (void)setResponseSerializer:(AFHTTPResponseSerializer <AFURLResponseSerialization> *)responseSerializer {
    NSParameterAssert(responseSerializer);

    [super setResponseSerializer:responseSerializer];
}

@dynamic securityPolicy;

#pragma mark - 设置安全策略
- (void)setSecurityPolicy:(AFSecurityPolicy *)securityPolicy {
    /// 如果SSLPinningMode是有安全策略的，但是请求的 scheme 不是 https 的，那么就要报错。因为安全策略必须要在 https 模式下
    if (securityPolicy.SSLPinningMode != AFSSLPinningModeNone && ![self.baseURL.scheme isEqualToString:@"https"]) {
        NSString *pinningMode = @"Unknown Pinning Mode";
        switch (securityPolicy.SSLPinningMode) {
            case AFSSLPinningModeNone:        pinningMode = @"AFSSLPinningModeNone"; break;
            case AFSSLPinningModeCertificate: pinningMode = @"AFSSLPinningModeCertificate"; break;
            case AFSSLPinningModePublicKey:   pinningMode = @"AFSSLPinningModePublicKey"; break;
        }
        NSString *reason = [NSString stringWithFormat:@"A security policy configured with `%@` can only be applied on a manager with a secure base URL (i.e. https)", pinningMode];
        @throw [NSException exceptionWithName:@"Invalid Security Policy" reason:reason userInfo:nil];
    }

    /// 给 URL 设置安全策略
    [super setSecurityPolicy:securityPolicy];
}
```

### 语义化请求方法

AFN 提供了进行不同方式请求的方法封装，这里进列举 get 和 post 两个方法：

```objc
/// get 的正真方法带有 progress 参数
- (NSURLSessionDataTask *)GET:(NSString *)URLString
                   parameters:(id)parameters
                     progress:(void (^)(NSProgress * _Nonnull))downloadProgress
                      success:(void (^)(NSURLSessionDataTask * _Nonnull, id _Nullable))success
                      failure:(void (^)(NSURLSessionDataTask * _Nullable, NSError * _Nonnull))failure
{

    /// 创建 NSURLSessionDataTask
    NSURLSessionDataTask *dataTask = [self dataTaskWithHTTPMethod:@"GET"
                                                        URLString:URLString
                                                       parameters:parameters
                                                   uploadProgress:nil
                                                 downloadProgress:downloadProgress
                                                          success:success
                                                          failure:failure];

    [dataTask resume];

    return dataTask;
}

- (NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(id)parameters
                       success:(void (^)(NSURLSessionDataTask *task, id responseObject))success
                       failure:(void (^)(NSURLSessionDataTask *task, NSError *error))failure
{
    return [self POST:URLString parameters:parameters progress:nil success:success failure:failure];
}

/// post 方法 带 progress 参数的
- (NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(id)parameters
                      progress:(void (^)(NSProgress * _Nonnull))uploadProgress
                       success:(void (^)(NSURLSessionDataTask * _Nonnull, id _Nullable))success
                       failure:(void (^)(NSURLSessionDataTask * _Nullable, NSError * _Nonnull))failure
{
    NSURLSessionDataTask *dataTask = [self dataTaskWithHTTPMethod:@"POST" URLString:URLString parameters:parameters uploadProgress:uploadProgress downloadProgress:nil success:success failure:failure];

    [dataTask resume];

    return dataTask;
}
```

他们调用了另一个方法后，就生成了一个 `NSURLSessionDataTask` 实例对象，并且通过 `resume` 方法开启网络请求。

那么创建 `NSURLSessionDataTask` 的方法做了什么呢？

```objc
/// 不带 constructingBodyWithBlock 创建 NSURLSessionDataTask
- (NSURLSessionDataTask *)dataTaskWithHTTPMethod:(NSString *)method
                                       URLString:(NSString *)URLString
                                      parameters:(id)parameters
                                  uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgress
                                downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgress
                                         success:(void (^)(NSURLSessionDataTask *, id))success
                                         failure:(void (^)(NSURLSessionDataTask *, NSError *))failure
{
    NSError *serializationError = nil;
    /// 通过 requestSerializer 创建 NSMutableURLRequest
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];
    if (serializationError) {
        if (failure) {
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
        }

        return nil;
    }

    __block NSURLSessionDataTask *dataTask = nil;
    /// 通过 request 创建 dataTask. 这是 AFURLSessionManager 中的方法
    dataTask = [self dataTaskWithRequest:request
                          uploadProgress:uploadProgress
                        downloadProgress:downloadProgress
                       completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(dataTask, error);
            }
        } else {
            if (success) {
                success(dataTask, responseObject);
            }
        }
    }];

    return dataTask;
}
```

首先通过前面的请求序列化类创建一个 `NSMutableURLRequest` 然后使用它创建 `NSURLSessionDataTask`，不过创建方法不在本类中，而是在它的父类中。这个将稍后再看。

可以看到，在经过一系列的封装之后，发送一个网络请求需要的 `NSMutableURLRequest`，`NSURLSession`，`NSURLSessionDataTask` 都通过 AFN 创建了，而自己只需要负责传入请求，参数和回调即可。

### 发送 form-data 请求

上面的 get，post 方法无法发送文件，因此，AFN 提供了一个方法用来发送 form-data 类型的请求：

```objc
/// post 方法带 constructingBodyWithBlock 的
/// constructingBodyWithBlock 参数 formData 会被添加到 http body 中
- (NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(id)parameters
     constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                      progress:(nullable void (^)(NSProgress * _Nonnull))uploadProgress
                       success:(void (^)(NSURLSessionDataTask *task, id responseObject))success
                       failure:(void (^)(NSURLSessionDataTask *task, NSError *error))failure
{
    NSError *serializationError = nil;
    /// 初始化一个 request，包含了参数以及生成 http body 中的 constructingBodyWithBlock
    NSMutableURLRequest *request = [self.requestSerializer multipartFormRequestWithMethod:@"POST" URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters constructingBodyWithBlock:block error:&serializationError];
    /// 序列化错误
    if (serializationError) {
        if (failure) {
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
        }

        return nil;
    }

    __block NSURLSessionDataTask *task = [self uploadTaskWithStreamedRequest:request progress:uploadProgress completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(task, error);
            }
        } else {
            if (success) {
                success(task, responseObject);
            }
        }
    }];

    [task resume];

    return task;
}
```

该方法会发起一个 form-data 类型的请求。你可以将 `NSData` 放入 param 中，由 AFN 进行序列化。也可以拿到 `constructingBodyWithBlock` 中的 block，通过 `AFMultipartFormData` 协议的方法，向其中添加 fileURL，来告诉 AFN 去序列化该地址下的文件。

## AFURLSessionManager

`AFURLSessionManager` 是 AFN 请求的核心。它遵循了下面几个协议：

- NSURLSessionTaskDelegate
- NSURLSessionDataDelegate
- NSURLSessionDownloadDelegate
- NSURLSessionDelegate

### AFURLSessionManager 的属性

`AFURLSessionManager` 中包含了非常多的实现协议时自定义的 block：

```objc
@interface AFURLSessionManager ()
/// 执行响应的队列
@property (readwrite, nonatomic, strong) NSOperationQueue *operationQueue;
/// task <=> 代理
@property (readwrite, nonatomic, strong) NSMutableDictionary *mutableTaskDelegatesKeyedByTaskIdentifier;
/// session 的 taskDescription 都是相同的
@property (readonly, nonatomic, copy) NSString *taskDescriptionForSessionTasks;
// NSURLSessionDelegate
/// 由于系统原因导致会话失效时的回调
@property (readwrite, nonatomic, copy) AFURLSessionDidBecomeInvalidBlock sessionDidBecomeInvalid;
/// 连接服务器，接收到认证请求时，可以返回指定的认证选项
@property (readwrite, nonatomic, copy) AFURLSessionDidReceiveAuthenticationChallengeBlock sessionDidReceiveAuthenticationChallenge;
/// 当应用进入后台时创建的后台会话的相关任务均执行完毕时，该回调在主队列中被执行
@property (readwrite, nonatomic, copy) AFURLSessionDidFinishEventsForBackgroundURLSessionBlock didFinishEventsForBackgroundURLSession;
// NSURLSessionTaskDelegate
/// 接收到重定向反馈时，该代码块可以指定重定向的链接
@property (readwrite, nonatomic, copy) AFURLSessionTaskWillPerformHTTPRedirectionBlock taskWillPerformHTTPRedirection;
///当数据任务被要求进行证书加密时，该代码块可以指定相关选择项，如使用证书、默认处理、取消请求
@property (readwrite, nonatomic, copy) AFURLSessionTaskDidReceiveAuthenticationChallengeBlock taskDidReceiveAuthenticationChallenge;
/// 任务需要流传递数据给服务端时，该代码块可以返回一个输入流
@property (readwrite, nonatomic, copy) AFURLSessionTaskNeedNewBodyStreamBlock taskNeedNewBodyStream;
/// 当数据上传时，该代码回调可以用来获取本次上传的字节数、已经上传的字节数、该任务需要上传的总字节数
@property (readwrite, nonatomic, copy) AFURLSessionTaskDidSendBodyDataBlock taskDidSendBodyData;
/// 当任务结束时，该代码回调会被执行，可以获取错误信息，如果有的话
@property (readwrite, nonatomic, copy) AFURLSessionTaskDidCompleteBlock taskDidComplete;
// NSURLSessionDataDelegate
/// 当数据任务接收到服务器响应时，该回调可以选择取消或允许等选项
@property (readwrite, nonatomic, copy) AFURLSessionDataTaskDidReceiveResponseBlock dataTaskDidReceiveResponse;
/// 当数据任务将转变为数据下载任务时，该回调可以进行一些处理，回调参数中包含了原任务，和将要转变为的目标任务
@property (readwrite, nonatomic, copy) AFURLSessionDataTaskDidBecomeDownloadTaskBlock dataTaskDidBecomeDownloadTask;
/// 当接收到服务端数据时，该回调被执行
@property (readwrite, nonatomic, copy) AFURLSessionDataTaskDidReceiveDataBlock dataTaskDidReceiveData;
/// 将要缓存响应报文时，可以对返回的响应报文进行一些处理，回调参数中包含了 NSCachedURLResponse 的实例对象，也是即将返回的实例
@property (readwrite, nonatomic, copy) AFURLSessionDataTaskWillCacheResponseBlock dataTaskWillCacheResponse;
// NSURLSessionDownloadDelegate
/// 当下载任务结束时，该回调代码块可以指定下载的缓存数据移动到的目标地址
@property (readwrite, nonatomic, copy) AFURLSessionDownloadTaskDidFinishDownloadingBlock downloadTaskDidFinishDownloading;
/// 在数据下载的过程中，该回调会被调用，其参数包含有本次下载的数据字节数、已经下载的字节数、该任务需要下载的总字节数
@property (readwrite, nonatomic, copy) AFURLSessionDownloadTaskDidWriteDataBlock downloadTaskDidWriteData;
/// 当下载任务再次启动时，该回调执行，回调参数包含有文件的字节偏移量和整个文件的字节长度
@property (readwrite, nonatomic, copy) AFURLSessionDownloadTaskDidResumeBlock downloadTaskDidResume;
@end
```

### AFURLSessionManager 的方法

#### 初始化方法

初始化方法中创建了请求需要使用的各种基本属性：

```objc
#pragma mark - 初始化方法
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    self = [super init];
    if (!self) {
        return nil;
    }

    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }

    self.sessionConfiguration = configuration;

    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;

    /// 创建 NSURLSession
    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];

    self.responseSerializer = [AFJSONResponseSerializer serializer];

    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];

    self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];

    self.lock = [[NSLock alloc] init];
    self.lock.name = AFURLSessionManagerLockName;

    /// 为session的所有task设置 delegate
    /// 做清空处理，有一种后台session，当从后台回来的时候，根据 configuration 中的一个 ID，就可以重新恢复这个session，这时候其中就会有之前未完成的task了。这里这么做的目的就是防止一些之前的后台请求任务，导致程序的crash。
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        for (NSURLSessionDataTask *task in dataTasks) {
            [self addDelegateForDataTask:task uploadProgress:nil downloadProgress:nil completionHandler:nil];
        }

        for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
            [self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
        }

        for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
            [self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
        }
    }];

    return self;
}
```

#### 创建 Task

创建的 Task 分为三种，普通的 dataTask，上传的 uploadTask 和下载的 downloadTask。

##### dataTask

在 `AFHTTPSessionManager` 中已经创建好了 `NSURLRequest` 对象，调用以下方法创建 `NSURLSessionDataTask` 对象实例：

```objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                            completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    return [self dataTaskWithRequest:request uploadProgress:nil downloadProgress:nil completionHandler:completionHandler];
}

/// c通过 request 创建 dataTask
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler {

    __block NSURLSessionDataTask *dataTask = nil;
    /// iOS8下可能会创建多个 task
    url_session_manager_create_task_safely(^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

    /// 给 dataTask 创建 delegate
    [self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];

    return dataTask;
}
```

除了简单调用 `NSURLSession` 的 API 生成实例外，`AFURLSessionManager` 还需要给生成的 dataTask 创建一个代理对象。这个下面就会说到。

##### uploadTask

uploadTask 在 `AFHTTPSessionManager` 中没有使用，`AFHTTPSessionManger` 提供的方法中要想实现上传功能，需要自己把文件转为 NSData 后通过参数上传。而 uploadTask 是自己通过输入流的方式将数据写入到了 request 中。使用 uploadTask 则能更加方便的交由 `NSURLSession` 完成相应的任务。只需要传入文件路径，data，或者输入流即可：

```objc
#pragma mark - 生成 uploadTask
/// 通过文件路径上传
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromFile:(NSURL *)fileURL
                                         progress:(void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                                completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    __block NSURLSessionUploadTask *uploadTask = nil;
    url_session_manager_create_task_safely(^{
        uploadTask = [self.session uploadTaskWithRequest:request fromFile:fileURL];
    });

    [self addDelegateForUploadTask:uploadTask progress:uploadProgressBlock completionHandler:completionHandler];

    return uploadTask;
}

/// 通过data上传
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromData:(NSData *)bodyData
                                         progress:(void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                                completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    __block NSURLSessionUploadTask *uploadTask = nil;
    url_session_manager_create_task_safely(^{
        uploadTask = [self.session uploadTaskWithRequest:request fromData:bodyData];
    });

    [self addDelegateForUploadTask:uploadTask progress:uploadProgressBlock completionHandler:completionHandler];

    return uploadTask;
}

/// 通过流上传
- (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request
                                                 progress:(void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                                        completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    __block NSURLSessionUploadTask *uploadTask = nil;
    url_session_manager_create_task_safely(^{
        uploadTask = [self.session uploadTaskWithStreamedRequest:request];
    });

    [self addDelegateForUploadTask:uploadTask progress:uploadProgressBlock completionHandler:completionHandler];

    return uploadTask;
}
```

同样的，每一个 dataTask 都会设置一个 delegate。

##### downloadTask

downloadTask 分为两个部分，一个是创建，一个是断线续传：

```objc
#pragma mark - 生成 downloadTask

/// 创建下载任务
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                             progress:(void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                                          destination:(NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                    completionHandler:(void (^)(NSURLResponse *response, NSURL *filePath, NSError *error))completionHandler
{
    __block NSURLSessionDownloadTask *downloadTask = nil;
    url_session_manager_create_task_safely(^{
        downloadTask = [self.session downloadTaskWithRequest:request];
    });

    [self addDelegateForDownloadTask:downloadTask progress:downloadProgressBlock destination:destination completionHandler:completionHandler];

    return downloadTask;
}

/// 继续下载
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData
                                                progress:(void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                                             destination:(NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                       completionHandler:(void (^)(NSURLResponse *response, NSURL *filePath, NSError *error))completionHandler
{
    __block NSURLSessionDownloadTask *downloadTask = nil;
    url_session_manager_create_task_safely(^{
        downloadTask = [self.session downloadTaskWithResumeData:resumeData];
    });

    [self addDelegateForDownloadTask:downloadTask progress:downloadProgressBlock destination:destination completionHandler:completionHandler];

    return downloadTask;
}
```

如果你不熟悉 `NSURLSessionDownloadTask` 那么你一定会很疑惑下面这个继续下载为什么只要传一个 `resumeData` 就可以知道下载的所有信息的。其实这个 `resumeData` 并不是下载的数据，而是下载数据的信息的 plist 生成的 `NSData`，其中包含了下载所需要的各种信息，包括本地缓存的路径名，下载的地址，当前下载了多少等。

#### 为 task 创建代理

task 请求过程中的请求中请求完成的回调都是通过 delegate 完成的。delegate 是一个 `AFURLSessionManagerTaskDelegate` 实例。添加代理的过程就是创建实例并设置回调方法的过程：

```objc
/// 设置 task 的代理； 代理包含了 task 的各种回调。
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
    NSParameterAssert(task);
    NSParameterAssert(delegate);

    [self.lock lock];
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    [self addNotificationObserverForTask:task];
    [self.lock unlock];
}

/// 给 datatask 增加代理对象
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    /// 创建一个 dataTask 的代理
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] initWithTask:dataTask];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    /// 设置 taskDescription 有助于区别是否是 AFN 创建的 task
    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    [self setDelegate:delegate forTask:dataTask];

    delegate.uploadProgressBlock = uploadProgressBlock;
    delegate.downloadProgressBlock = downloadProgressBlock;
}

- (void)addDelegateForUploadTask:(NSURLSessionUploadTask *)uploadTask
                        progress:(void (^)(NSProgress *uploadProgress)) uploadProgressBlock
               completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] initWithTask:uploadTask];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    uploadTask.taskDescription = self.taskDescriptionForSessionTasks;

    [self setDelegate:delegate forTask:uploadTask];

    delegate.uploadProgressBlock = uploadProgressBlock;
}

- (void)addDelegateForDownloadTask:(NSURLSessionDownloadTask *)downloadTask
                          progress:(void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                       destination:(NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                 completionHandler:(void (^)(NSURLResponse *response, NSURL *filePath, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] initWithTask:downloadTask];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    if (destination) {
        delegate.downloadTaskDidFinishDownloading = ^NSURL * (NSURLSession * __unused session, NSURLSessionDownloadTask *task, NSURL *location) {
            return destination(location, task.response);
        };
    }

    downloadTask.taskDescription = self.taskDescriptionForSessionTasks;

    [self setDelegate:delegate forTask:downloadTask];

    delegate.downloadProgressBlock = downloadProgressBlock;
}
```

每一个生成的 task 和它的代理都会保存在 manager 的 `mutableTaskDelegatesKeyedByTaskIdentifier` 中。当然其实也可以通过给 task 添加关联对象的方式保存 delegate。

### 协议方法

上面说到了 `AFURLSessionManager` 实现了 `NSURLSession` 相关的四个代理。下面来分别看看：

#### NSURLSessionDelegate

如果有相关的 block 那么就执行相关的 block，没有 block 也可以通过监听相应的 notification ：

```objc
/// 因为异常终止请求
- (void)URLSession:(NSURLSession *)session
didBecomeInvalidWithError:(NSError *)error
{
    if (self.sessionDidBecomeInvalid) {
        self.sessionDidBecomeInvalid(session, error);
    }

    [[NSNotificationCenter defaultCenter] postNotificationName:AFURLSessionDidInvalidateNotification object:session];
}

/// 证书校验
- (void)URLSession:(NSURLSession *)session
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler
{
    /// 默认的处理方式
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    __block NSURLCredential *credential = nil;

    if (self.sessionDidReceiveAuthenticationChallenge) {
        /// 自己的处理方式
        disposition = self.sessionDidReceiveAuthenticationChallenge(session, challenge, &credential);
    } else {
        /// 需要验证证书
        if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
            /// serverTrust 是服务端的证书链
            if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
                /// 验证通过
                credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
                if (credential) {
                    disposition = NSURLSessionAuthChallengeUseCredential;
                } else {
                    disposition = NSURLSessionAuthChallengePerformDefaultHandling;
                }
            } else {
                /// 验证不通过取消请求
                disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
            }
        } else {
            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
        }
    }

    if (completionHandler) {
        /// 返回验证解果
        completionHandler(disposition, credential);
    }
}

/// 后台任务完成后的回调
- (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session {
    if (self.didFinishEventsForBackgroundURLSession) {
        dispatch_async(dispatch_get_main_queue(), ^{
            self.didFinishEventsForBackgroundURLSession(session);
        });
    }
}
```

#### NSURLSessionTaskDelegate

```objc
/// 重定向操作
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
willPerformHTTPRedirection:(NSHTTPURLResponse *)response
        newRequest:(NSURLRequest *)request
 completionHandler:(void (^)(NSURLRequest *))completionHandler
{
    NSURLRequest *redirectRequest = request;

    if (self.taskWillPerformHTTPRedirection) {
        redirectRequest = self.taskWillPerformHTTPRedirection(session, task, response, request);
    }

    if (completionHandler) {
        completionHandler(redirectRequest);
    }
}

/// 使用方法和 session 的那个一致
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge
 completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential *credential))completionHandler
{
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    __block NSURLCredential *credential = nil;

    if (self.taskDidReceiveAuthenticationChallenge) {
        disposition = self.taskDidReceiveAuthenticationChallenge(session, task, challenge, &credential);
    } else {
        if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
            if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
                disposition = NSURLSessionAuthChallengeUseCredential;
                credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
            } else {
                disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
            }
        } else {
            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
        }
    }

    if (completionHandler) {
        completionHandler(disposition, credential);
    }
}

/// 请求需要一个全新的，未打开的数据时调用。特别是请求一个body失败时，可以通过这个方法给一个新的body
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
 needNewBodyStream:(void (^)(NSInputStream *bodyStream))completionHandler
{
    NSInputStream *inputStream = nil;

    if (self.taskNeedNewBodyStream) {
        inputStream = self.taskNeedNewBodyStream(session, task);
    } else if (task.originalRequest.HTTPBodyStream && [task.originalRequest.HTTPBodyStream conformsToProtocol:@protocol(NSCopying)]) {
        inputStream = [task.originalRequest.HTTPBodyStream copy];
    }

    if (completionHandler) {
        completionHandler(inputStream);
    }
}

/// 上传数据那一刻调用
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
   didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend
{

    /// 数据的总长度
    int64_t totalUnitCount = totalBytesExpectedToSend;
    if(totalUnitCount == NSURLSessionTransferSizeUnknown) {
        NSString *contentLength = [task.originalRequest valueForHTTPHeaderField:@"Content-Length"];
        if(contentLength) {
            totalUnitCount = (int64_t) [contentLength longLongValue];
        }
    }
    
    /// 获取 task 对应的 delegate
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];
    
    if (delegate) {
        /// 调用 delegate 的回调方法，设置已经上传了的长度和总共需要上传的长度
        [delegate URLSession:session task:task didSendBodyData:bytesSent totalBytesSent:totalBytesSent totalBytesExpectedToSend:totalBytesExpectedToSend];
    }

    if (self.taskDidSendBodyData) {
        self.taskDidSendBodyData(session, task, bytesSent, totalBytesSent, totalUnitCount);
    }
}

/// 上传完成时
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];

    // delegate may be nil when completing a task in the background
    if (delegate) {
        [delegate URLSession:session task:task didCompleteWithError:error];

        [self removeDelegateForTask:task];
    }

    if (self.taskDidComplete) {
        self.taskDidComplete(session, task, error);
    }
}
```

在发送请求的时候，将请求的总数据和已发送的数据通过 delegate 的相应方法保存下来：

```objc
/// AFURLSessionManagerTaskDelegate 中
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
   didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend{
    
    self.uploadProgress.totalUnitCount = task.countOfBytesExpectedToSend;
    self.uploadProgress.completedUnitCount = task.countOfBytesSent;
}
```

在请求完成时，又调用了 delegate 的相应方法：

```objc
/// AFURLSessionManagerTaskDelegate 中
- (void)URLSession:(__unused NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
    __strong AFURLSessionManager *manager = self.manager;

    __block id responseObject = nil;

    __block NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
    userInfo[AFNetworkingTaskDidCompleteResponseSerializerKey] = manager.responseSerializer;

    //Performance Improvement from #2672
    NSData *data = nil;
    if (self.mutableData) {
        data = [self.mutableData copy];
        //We no longer need the reference, so nil it out to gain back some memory.
        self.mutableData = nil;
    }

    if (self.downloadFileURL) {
        userInfo[AFNetworkingTaskDidCompleteAssetPathKey] = self.downloadFileURL;
    } else if (data) {
        userInfo[AFNetworkingTaskDidCompleteResponseDataKey] = data;
    }

    if (error) {
        userInfo[AFNetworkingTaskDidCompleteErrorKey] = error;

        dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
            if (self.completionHandler) {
                self.completionHandler(task.response, responseObject, error);
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
            });
        });
    } else {
      	/// 请求成功，异步处理返回的数据
        dispatch_async(url_session_manager_processing_queue(), ^{
            NSError *serializationError = nil;
            responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];

            if (self.downloadFileURL) {
                responseObject = self.downloadFileURL;
            }

            if (responseObject) {
                userInfo[AFNetworkingTaskDidCompleteSerializedResponseKey] = responseObject;
            }

            if (serializationError) {
                userInfo[AFNetworkingTaskDidCompleteErrorKey] = serializationError;
            }

            dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
                if (self.completionHandler) {
                    self.completionHandler(task.response, responseObject, serializationError);
                }

                dispatch_async(dispatch_get_main_queue(), ^{
                    [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
                });
            });
        });
    }
}
```

这里 AFN 除了调用回调以及发送通知外，我们还可以发现使用了 `dispatch_group_async`。它可以帮助我们做一些多个请求同步的操作。`url_session_manager_completion_group()` 方法提供了一个 group：

```objc
static dispatch_group_t url_session_manager_completion_group() {
    static dispatch_group_t af_url_session_manager_completion_group;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_session_manager_completion_group = dispatch_group_create();
    });

    return af_url_session_manager_completion_group;
}
```

我们可以在外部通过 AFN 发起多个请求，然后通过 `url_session_manager_completion_group()` 方法提供的 group，在请求完成后统一执行。当然其实我们亦可以在各个请求的完成回调中通过信号量的方式执行。不过那样就有点不是那么优雅了。

#### NSURLSessionDataDelegate

```objc
#pragma mark - NSURLSessionDataDelegate

/// 收到响应时调用，可以选择取消或者允许响应
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler
{
    NSURLSessionResponseDisposition disposition = NSURLSessionResponseAllow;

    if (self.dataTaskDidReceiveResponse) {
        disposition = self.dataTaskDidReceiveResponse(session, dataTask, response);
    }

    if (completionHandler) {
        completionHandler(disposition);
    }
}

/// 当NSURLSessionDataTask变为NSURLSessionDownloadTask调用，之后NSURLSessionDataTask将不再接受消息
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
didBecomeDownloadTask:(NSURLSessionDownloadTask *)downloadTask
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:dataTask];
    if (delegate) {
        /// 移除 dataTask
        [self removeDelegateForTask:dataTask];
        /// 设置 downloadTask
        [self setDelegate:delegate forTask:downloadTask];
    }

    if (self.dataTaskDidBecomeDownloadTask) {
        self.dataTaskDidBecomeDownloadTask(session, dataTask, downloadTask);
    }
}

/// 收到数据后调用
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:dataTask];
    /// 调用 delegate 把 data 收集起来
    [delegate URLSession:session dataTask:dataTask didReceiveData:data];

    if (self.dataTaskDidReceiveData) {
        self.dataTaskDidReceiveData(session, dataTask, data);
    }
}

/// 缓存报文时调用
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
 willCacheResponse:(NSCachedURLResponse *)proposedResponse
 completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler
{
    NSCachedURLResponse *cachedResponse = proposedResponse;

    if (self.dataTaskWillCacheResponse) {
        cachedResponse = self.dataTaskWillCacheResponse(session, dataTask, proposedResponse);
    }

    if (completionHandler) {
        completionHandler(cachedResponse);
    }
}
```

主要关系受到数据后的回调方法，它把接收到的 data 传给了 delegate：

```objc
/// AFURLSessionManagerTaskDelegate 中
- (void)URLSession:(__unused NSURLSession *)session
          dataTask:(__unused NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    self.downloadProgress.totalUnitCount = dataTask.countOfBytesExpectedToReceive;
    self.downloadProgress.completedUnitCount = dataTask.countOfBytesReceived;

    [self.mutableData appendData:data];
}
```

前面请求完成时传给 `responseSerializer` 的就是这里 `mutableData` 拼接好的数据。它把每次收到的数据都拼接起来，等待请求完成。

#### NSURLSessionDownloadDelegate

```objc
#pragma mark - NSURLSessionDownloadDelegate

/// 下载完成调用
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:downloadTask];
    if (self.downloadTaskDidFinishDownloading) {
        /// 获取下载数据要保存的地址
        NSURL *fileURL = self.downloadTaskDidFinishDownloading(session, downloadTask, location);
        if (fileURL) {
            delegate.downloadFileURL = fileURL;
            NSError *error = nil;
            
            /// 移动下载的数据到指定地址
            if (![[NSFileManager defaultManager] moveItemAtURL:location toURL:fileURL error:&error]) {
                [[NSNotificationCenter defaultCenter] postNotificationName:AFURLSessionDownloadTaskDidFailToMoveFileNotification object:downloadTask userInfo:error.userInfo];
            }

            return;
        }
    }

    if (delegate) {
        [delegate URLSession:session downloadTask:downloadTask didFinishDownloadingToURL:location];
    }
}

/// 下载中调用
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
      didWriteData:(int64_t)bytesWritten
 totalBytesWritten:(int64_t)totalBytesWritten
totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite
{
    
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:downloadTask];
    
    if (delegate) {
        [delegate URLSession:session downloadTask:downloadTask didWriteData:bytesWritten totalBytesWritten:totalBytesWritten totalBytesExpectedToWrite:totalBytesExpectedToWrite];
    }

    if (self.downloadTaskDidWriteData) {
        self.downloadTaskDidWriteData(session, downloadTask, bytesWritten, totalBytesWritten, totalBytesExpectedToWrite);
    }
}

/// 恢复下载调用
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
 didResumeAtOffset:(int64_t)fileOffset
expectedTotalBytes:(int64_t)expectedTotalBytes
{
    
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:downloadTask];
    
    if (delegate) {
        [delegate URLSession:session downloadTask:downloadTask didResumeAtOffset:fileOffset expectedTotalBytes:expectedTotalBytes];
    }

    if (self.downloadTaskDidResume) {
        self.downloadTaskDidResume(session, downloadTask, fileOffset, expectedTotalBytes);
    }
}
```

`NSURLSession` 中的下载由系统自动做了边下边存的过程。因此，我们要在下载完成之后把缓存的路径上的文件保存到实际的路径上。所以下载的时候要给 manager 传入一个 `downloadTaskDidFinishDownloading` 的 block，告知下载完成时候安放的路径。

## 总结

`AFHTTPSessionManager` 和 `AFURLSessionManager` 是我们使用 AFN 的入口。`AFHTTPSessionManager` 作为子类，封装了简单 dataTask 请求。它还负责将请求参数拼接以及转码。

对于上传文件的操作，仍然可以使用 `AFHTTPSessionManager` 创建 content-type 为 form-data 的请求，但是这就需要自己把文件转为 `NSData` 或者 `NSInputStream` 类型，设置给 request。如果想要更方便的方式完成，可以使用 `AFURLSessionManager` 提供的 upload 方法创建的 uploadTask，只需要给出 fileURL 即可。

`AFURLSessionManager` 负责创建 `NSURLSession`，并承担了发送请求需要实现的所有协议。针对每一个 task，为其增加一个代理对象。当 `AFURLSessionManager` 接收到`NSURLSessionDataDelegate` 的接收 data 的回调的时候，就会把 data 交给 task 对应的 delegate 保存。当接收到 `NSURLSessionTaskDelegate` 的完成回调的时候，就会告知 task 对应 delegate，让其执行回调方法。

同时 `AFURLSessionManager` 实现的 `NSURLSessionDelegate` 还包含了 HTTPS 证书校验的策略，通过它验证证书信息。