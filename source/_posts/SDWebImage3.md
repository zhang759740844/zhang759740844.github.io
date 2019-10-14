title: SDWebImage 源码解析—下载策略
date: 2018/3/5 14:07:12  
categories: iOS
tags: 

- 学习笔记
- 源码解析

------

上一篇的缓存策略主要说到二级缓存。这一部分将谈及下载。

<!--more-->

## SDWebImageManager 调用

首先看 `SDWebImageManager` 中的方法：

```objc
#pragma mark - 下载图片
- (void)callDownloadProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                    url:(nonnull NSURL *)url
                                options:(SDWebImageOptions)options
                                context:(SDWebImageContext *)context
                            cachedImage:(nullable UIImage *)cachedImage
                             cachedData:(nullable NSData *)cachedData
                              cacheType:(SDImageCacheType)cacheType
                               progress:(nullable SDImageLoaderProgressBlock)progressBlock
                              completed:(nullable SDInternalCompletionBlock)completedBlock {
    // Check whether we should download image from network
    /// 能否下载
    /// options 不是只能从 cache 中加载
    BOOL shouldDownload = !SD_OPTIONS_CONTAINS(options, SDWebImageFromCacheOnly);
    /// 并且不存在缓存的 image 或者 options 必须要刷新缓存的 image
    shouldDownload &= (!cachedImage || options & SDWebImageRefreshCached);
    /// 并且 delegate 没有 imageManager:shouldDownloadImageForURL: 方法或者返回的是 true
    shouldDownload &= (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url]);
    /// SDWebImageDownloader 中实现，对所有费控 url 都是 true
    shouldDownload &= [self.imageLoader canRequestImageForURL:url];
    /// 如果必须要下载
    if (shouldDownload) {
        /// 如果 option 的目的是 refreshCached。那么要重新从服务器下载，让 NSURLCache 记录下新的缓存
        if (cachedImage && options & SDWebImageRefreshCached) {
            [self callCompletionBlockForOperation:operation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
            // Pass the cached image to the image loader. The image loader should check whether the remote image is equal to the cached image.
            SDWebImageMutableContext *mutableContext;
            if (context) {
                mutableContext = [context mutableCopy];
            } else {
                mutableContext = [NSMutableDictionary dictionary];
            }
            mutableContext[SDWebImageContextLoaderCachedImage] = cachedImage;
            context = [mutableContext copy];
        }
        
        // `SDWebImageCombinedOperation` -> `SDWebImageDownloadToken` -> `downloadOperationCancelToken`, which is a `SDCallbacksDictionary` and retain the completed block below, so we need weak-strong again to avoid retain cycle
        @weakify(operation);
        /// 下载的 operation
        operation.loaderOperation = [self.imageLoader requestImageWithURL:url options:options context:context progress:progressBlock completed:^(UIImage *downloadedImage, NSData *downloadedData, NSError *error, BOOL finished) {
            @strongify(operation);
            if (!operation || operation.isCancelled) {
                /// 如果 operation 已经销毁或者已经取消,抛出error
                [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorCancelled userInfo:nil] url:url];
            } else if (cachedImage && options & SDWebImageRefreshCached && [error.domain isEqualToString:SDWebImageErrorDomain] && error.code == SDWebImageErrorCacheNotModified) {
                /// 如果下载的目的是刷新缓存，但是缓存没有变化，那么什么也不做
            } else if ([error.domain isEqualToString:SDWebImageErrorDomain] && error.code == SDWebImageErrorCancelled) {
                /// 过早的被取消
                [self callCompletionBlockForOperation:operation completion:completedBlock error:error url:url];
            } else if (error) {
                /// 各种其他错误
                [self callCompletionBlockForOperation:operation completion:completedBlock error:error url:url];
                /// 判断是否要 block 错误的 url
                BOOL shouldBlockFailedURL = [self shouldBlockFailedURLWithURL:url error:error];
                if (shouldBlockFailedURL) {
                    SD_LOCK(self.failedURLsLock);
                    [self.failedURLs addObject:url];
                    SD_UNLOCK(self.failedURLsLock);
                }
            } else {
                /// 其他成功的情况
                if ((options & SDWebImageRetryFailed)) {
                    SD_LOCK(self.failedURLsLock);
                    [self.failedURLs removeObject:url];
                    SD_UNLOCK(self.failedURLsLock);
                }
                
                /// 进行图片存储
                [self callStoreCacheProcessForOperation:operation url:url options:options context:context downloadedImage:downloadedImage downloadedData:downloadedData finished:finished progress:progressBlock completed:completedBlock];
            }
            
            if (finished) {
                [self safelyRemoveOperationFromRunning:operation];
            }
        }];
    /// 如果不让下载，但是有缓存的图片，b把缓存图片t返回
    } else if (cachedImage) {
        [self callCompletionBlockForOperation:operation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
        [self safelyRemoveOperationFromRunning:operation];
    /// 没有缓存的图片并且也不能下载，直接调用complete 回调把空返回
    } else {
        // Image not in cache and download disallowed by delegate
        [self callCompletionBlockForOperation:operation completion:completedBlock image:nil data:nil error:nil cacheType:SDImageCacheTypeNone finished:YES url:url];
        [self safelyRemoveOperationFromRunning:operation];
    }
}
```

它会通过其属性 `imageLoader` 去创建一个下载用的 operation。这个 `imageLoader` 属性的定义如下，它会满足 `SDImageLoader` 协议：

```objc
@property (strong, nonatomic, readonly, nonnull) id<SDImageLoader> imageLoader;
```

在初始化 `SDWebImageManager` 的时候对其进行了设置，它默认是 `SDWebImageDownloader` 的实例。

## SDWebImageDownloader

### 初始化方法

`SDWebImageDownloader` 也是一个单例对象，它的初始化方法如下：

```objc
- (instancetype)initWithConfig:(SDWebImageDownloaderConfig *)config {
    self = [super init];
    if (self) {
        if (!config) {
            config = SDWebImageDownloaderConfig.defaultDownloaderConfig;
        }
        _config = [config copy];
        /// 监听 config 中设置的最大并发数
        [_config addObserver:self forKeyPath:NSStringFromSelector(@selector(maxConcurrentDownloads)) options:0 context:SDWebImageDownloaderContext];
        /// 创建一个用于下载的 NSOperationQueue
        _downloadQueue = [NSOperationQueue new];
        /// NSOperationQueue 的最大并发数
        _downloadQueue.maxConcurrentOperationCount = _config.maxConcurrentDownloads;
        _downloadQueue.name = @"com.hackemist.SDWebImageDownloader";
        _URLOperations = [NSMutableDictionary new];
        NSMutableDictionary<NSString *, NSString *> *headerDictionary = [NSMutableDictionary dictionary];
        NSString *userAgent = nil;
        // User-Agent Header; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.43
        /// 设置 UA
        userAgent = [NSString stringWithFormat:@"%@/%@ (%@; iOS %@; Scale/%0.2f)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[UIDevice currentDevice] model], [[UIDevice currentDevice] systemVersion], [[UIScreen mainScreen] scale]];
        if (userAgent) {
            if (![userAgent canBeConvertedToEncoding:NSASCIIStringEncoding]) {
                NSMutableString *mutableUserAgent = [userAgent mutableCopy];
                if (CFStringTransform((__bridge CFMutableStringRef)(mutableUserAgent), NULL, (__bridge CFStringRef)@"Any-Latin; Latin-ASCII; [:^ASCII:] Remove", false)) {
                    userAgent = mutableUserAgent;
                }
            }
            headerDictionary[@"User-Agent"] = userAgent;
        }
        headerDictionary[@"Accept"] = @"image/*,*/*;q=0.8";
        _HTTPHeaders = headerDictionary;
        _HTTPHeadersLock = dispatch_semaphore_create(1);
        _operationsLock = dispatch_semaphore_create(1);
        NSURLSessionConfiguration *sessionConfiguration = _config.sessionConfiguration;
        if (!sessionConfiguration) {
            sessionConfiguration = [NSURLSessionConfiguration defaultSessionConfiguration];
        }
        /**
         *  Create the session for this task
         *  We send nil as delegate queue so that the session creates a serial operation queue for performing all delegate
         *  method calls and completion handler calls.
         */
        /// 创建 NSURLSession
        _session = [NSURLSession sessionWithConfiguration:sessionConfiguration
                                                 delegate:self
                                            delegateQueue:nil];
    }
    return self;
}
```

初始化了一个 NSURLSession，并且设置了部分 header。

### 外部调用方法

外部调用的方法如下，它先对 options 做了一个简单的处理：

```objc
#pragma mark - 外部调用
- (id<SDWebImageOperation>)requestImageWithURL:(NSURL *)url options:(SDWebImageOptions)options context:(SDWebImageContext *)context progress:(SDImageLoaderProgressBlock)progressBlock completed:(SDImageLoaderCompletedBlock)completedBlock {
    UIImage *cachedImage = context[SDWebImageContextLoaderCachedImage];
    
    SDWebImageDownloaderOptions downloaderOptions = 0;
    /// 低优先级
    if (options & SDWebImageLowPriority) downloaderOptions |= SDWebImageDownloaderLowPriority;
    /// image 会一点一点加载出来
    if (options & SDWebImageProgressiveLoad) downloaderOptions |= SDWebImageDownloaderProgressiveLoad;
    /// 使用 NSURLCache 缓存
    if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
    /// 允许后台下载
    if (options & SDWebImageContinueInBackground) downloaderOptions |= SDWebImageDownloaderContinueInBackground;
    /// cookies 相关
    if (options & SDWebImageHandleCookies) downloaderOptions |= SDWebImageDownloaderHandleCookies;
    /// 允许非法 SSL
    if (options & SDWebImageAllowInvalidSSLCertificates) downloaderOptions |= SDWebImageDownloaderAllowInvalidSSLCertificates;
    /// 高优先级
    if (options & SDWebImageHighPriority) downloaderOptions |= SDWebImageDownloaderHighPriority;
    /// 降低图片的 scale
    if (options & SDWebImageScaleDownLargeImages) downloaderOptions |= SDWebImageDownloaderScaleDownLargeImages;
    /// 禁止解码图片
    if (options & SDWebImageAvoidDecodeImage) downloaderOptions |= SDWebImageDownloaderAvoidDecodeImage;
    /// 值解码第一帧
    if (options & SDWebImageDecodeFirstFrameOnly) downloaderOptions |= SDWebImageDownloaderDecodeFirstFrameOnly;
    /// 预加载所有帧
    if (options & SDWebImagePreloadAllFrames) downloaderOptions |= SDWebImageDownloaderPreloadAllFrames;

    /// 有图但是要刷新缓存
    if (cachedImage && options & SDWebImageRefreshCached) {
        // force progressive off if image already cached but forced refreshing
        downloaderOptions &= ~SDWebImageDownloaderProgressiveLoad;
        // ignore image read from NSURLCache if image if cached but force refreshing
        downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
    }
    
    return [self downloadImageWithURL:url options:downloaderOptions context:context progress:progressBlock completed:completedBlock];
}
```

### 创建 `SDWebImageDownloadToken` 

 `SDWebImageDownloadToken` 类型的实例就是保存在 `SDWebImageCombinedOperation` 中的 `loaderOperation`。先看一下它的实例对象：

```objc
@interface SDWebImageDownloadToken ()

@property (nonatomic, strong, nullable, readwrite) NSURL *url;
@property (nonatomic, strong, nullable, readwrite) NSURLRequest *request;
@property (nonatomic, strong, nullable, readwrite) NSURLResponse *response;
@property (nonatomic, strong, nullable, readwrite) id downloadOperationCancelToken;
@property (nonatomic, weak, nullable) NSOperation<SDWebImageDownloaderOperation> *downloadOperation;
@property (nonatomic, weak, nullable) SDWebImageDownloader *downloader;
@property (nonatomic, assign, getter=isCancelled) BOOL cancelled;

@end
```

可以看到它的内部包含一个 `NSOperation` 和 `cancelled` 实例。可以看出，他是 `NSOperation` 的包装类。来看一下它的创建过程：

```objc
- (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url
                                                   options:(SDWebImageDownloaderOptions)options
                                                   context:(nullable SDWebImageContext *)context
                                                  progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                                 completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
    if (url == nil) {
        if (completedBlock) {
            NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}];
            completedBlock(nil, nil, error, YES);
        }
        return nil;
    }
    
    SD_LOCK(self.operationsLock);
    id downloadOperationCancelToken;
    /// 从单例的 URLOperations 池中取出该 url 对应的 operation
    NSOperation<SDWebImageDownloaderOperation> *operation = [self.URLOperations objectForKey:url];
    if (!operation || operation.isFinished || operation.isCancelled) {
        /// 如果 opertion 不存在或者已经结束了，那么就创建一个新的 operation
        operation = [self createDownloaderOperationWithUrl:url options:options context:context];
        /// 没创建成功就报错
        if (!operation) {
            SD_UNLOCK(self.operationsLock);
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidDownloadOperation userInfo:@{NSLocalizedDescriptionKey : @"Downloader operation is nil"}];
                completedBlock(nil, nil, error, YES);
            }
            return nil;
        }
        @weakify(self);
        /// opeartion 完成之后要移除单例中对应的 operation
        operation.completionBlock = ^{
            @strongify(self);
            if (!self) {
                return;
            }
            SD_LOCK(self.operationsLock);
            [self.URLOperations removeObjectForKey:url];
            SD_UNLOCK(self.operationsLock);
        };
        /// 把创建的 operation 加入到单例的字典中
        self.URLOperations[url] = operation;
        // Add operation to operation queue only after all configuration done according to Apple's doc.
        // `addOperation:` does not synchronously execute the `operation.completionBlock` so this will not cause deadlock.
        /// 开启异步任务
        [self.downloadQueue addOperation:operation];
        /// 把 progressBlock 和 completedBlock 放到 Operation 的数组中。
        downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
    } else {
        /// 如果单例中已经存在了这个 opeartion，那么说明对于这个下载任务会有多个监听。因此通过 addHandlersForProgress 将完成回调加入 operation 的数组中
        @synchronized (operation) {
            downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
        }
        /// 存在 operation 并且不是在执行中
        if (!operation.isExecuting) {
            /// 根据 options 设置优先级
            if (options & SDWebImageDownloaderHighPriority) {
                operation.queuePriority = NSOperationQueuePriorityHigh;
            } else if (options & SDWebImageDownloaderLowPriority) {
                operation.queuePriority = NSOperationQueuePriorityLow;
            } else {
                operation.queuePriority = NSOperationQueuePriorityNormal;
            }
        }
    }
    SD_UNLOCK(self.operationsLock);
    
    /// SDWebImageDownloadToken 是 operation 的包装类
    SDWebImageDownloadToken *token = [[SDWebImageDownloadToken alloc] initWithDownloadOperation:operation];
    token.url = url;
    token.request = operation.request;
    /// 通过downloadOperationCancelToken作为标识可以取消某一个opeartion的回调
    token.downloadOperationCancelToken = downloadOperationCancelToken;
    token.downloader = self;
    
    return token;
}
```

`SDWebImageDownloader` 是一个单例对象，它的内部包含了一个 `URLOpeartion` 的字典，保存着 `url → NSOpeartion` 的映射。当重复请求同一个正在请求的 url 的时候，就可以将回调方法保存在 `NSOperation` 的特定数组中，可以通过创建的 `downloadOperationCancelToken` 找到它，在特定的时候销毁或取消。

先不谈 `NSOperation` 是如何创建的，当它被创建之后，会被包裹在 `SDWebImageDownloadToken` 实例中，同样还包括 `downloadOperationCancelToken`。通过 `downloadOperationCancelToken` 可以取消某一个回调。

## SDWebImageDownloaderOperation

`SDWebImageDownloaderOperation` 是 `NSOperation` 的子类，负责异步下载。

### 创建 NSOperation

```objc
#pragma mark - 创建下载 Operation
- (nullable NSOperation<SDWebImageDownloaderOperation> *)createDownloaderOperationWithUrl:(nonnull NSURL *)url
                                                                                  options:(SDWebImageDownloaderOptions)options
                                                                                  context:(nullable SDWebImageContext *)context {
    /// 请求超时时间
    NSTimeInterval timeoutInterval = self.config.downloadTimeout;
    if (timeoutInterval == 0.0) {
        timeoutInterval = 15.0;
    }
    
    // In order to prevent from potential duplicate caching (NSURLCache + SDImageCache) we disable the cache for image requests if told otherwise
    /// 是否使用 NSURLCache
    NSURLRequestCachePolicy cachePolicy = options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData;
    /// 创建一个 request
    NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:cachePolicy timeoutInterval:timeoutInterval];
    mutableRequest.HTTPShouldHandleCookies = SD_OPTIONS_CONTAINS(options, SDWebImageDownloaderHandleCookies);
    mutableRequest.HTTPShouldUsePipelining = YES;
    SD_LOCK(self.HTTPHeadersLock);
    /// 设置 request 的 http 头
    mutableRequest.allHTTPHeaderFields = self.HTTPHeaders;
    SD_UNLOCK(self.HTTPHeadersLock);
    
    /// 自定义对于请求做一些修改
    id<SDWebImageDownloaderRequestModifier> requestModifier;
    if ([context valueForKey:SDWebImageContextDownloadRequestModifier]) {
        requestModifier = [context valueForKey:SDWebImageContextDownloadRequestModifier];
    } else {
        requestModifier = self.requestModifier;
    }
    NSURLRequest *request;
    if (requestModifier) {
        NSURLRequest *modifiedRequest = [requestModifier modifiedRequestWithRequest:[mutableRequest copy]];
        // If modified request is nil, early return
        if (!modifiedRequest) {
            return nil;
        } else {
            request = [modifiedRequest copy];
        }
    } else {
        request = [mutableRequest copy];
    }
    
    /// 有自定义 operationClass 用 operationClass，没有就用默认的 SDWebImageDownloaderOperation
    Class operationClass = self.config.operationClass;
    if (operationClass && [operationClass isSubclassOfClass:[NSOperation class]] && [operationClass conformsToProtocol:@protocol(SDWebImageDownloaderOperation)]) {
        // Custom operation class
    } else {
        operationClass = [SDWebImageDownloaderOperation class];
    }
    NSOperation<SDWebImageDownloaderOperation> *operation = [[operationClass alloc] initWithRequest:request inSession:self.session options:options context:context];
    
    /// 有鉴权执行鉴权
    if ([operation respondsToSelector:@selector(setCredential:)]) {
        if (self.config.urlCredential) {
            operation.credential = self.config.urlCredential;
        } else if (self.config.username && self.config.password) {
            operation.credential = [NSURLCredential credentialWithUser:self.config.username password:self.config.password persistence:NSURLCredentialPersistenceForSession];
        }
    }
        
    /// 最小回调进度
    if ([operation respondsToSelector:@selector(setMinimumProgressInterval:)]) {
        operation.minimumProgressInterval = MIN(MAX(self.config.minimumProgressInterval, 0), 1);
    }
    
    /// 下载优先级
    if (options & SDWebImageDownloaderHighPriority) {
        operation.queuePriority = NSOperationQueuePriorityHigh;
    } else if (options & SDWebImageDownloaderLowPriority) {
        operation.queuePriority = NSOperationQueuePriorityLow;
    }
    
    /// 如果是后进先出的，那么通过dependency确定执行顺序
    if (self.config.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
        // Emulate LIFO execution order by systematically adding new operations as last operation's dependency
        [self.lastAddedOperation addDependency:operation];
        self.lastAddedOperation = operation;
    }
    
    return operation;
}
```

SDWebImage 中充分利用了协议代替继承。这里需要创建的是一个实现了 `SDWebImageDownloaderOperation` 协议的 `NSOperation`:

```objc
@protocol SDWebImageDownloaderOperation <NSURLSessionTaskDelegate, NSURLSessionDataDelegate>
@required
/// 初始化方法，保存 NSURLRequest 和 NSURLSession 实例对象
- (nonnull instancetype)initWithRequest:(nullable NSURLRequest *)request
                              inSession:(nullable NSURLSession *)session
                                options:(SDWebImageDownloaderOptions)options;

- (nonnull instancetype)initWithRequest:(nullable NSURLRequest *)request
                              inSession:(nullable NSURLSession *)session
                                options:(SDWebImageDownloaderOptions)options
                                context:(nullable SDWebImageContext *)context;

/// 保存进度回调以及完成回调
- (nullable id)addHandlersForProgress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                            completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock;

/// 根据 token 取消 NSOperation 中的某个进度及完成回调
- (BOOL)cancel:(nullable id)token;

@property (strong, nonatomic, readonly, nullable) NSURLRequest *request;
@property (strong, nonatomic, readonly, nullable) NSURLResponse *response;

@optional
@property (strong, nonatomic, readonly, nullable) NSURLSessionTask *dataTask;
@property (strong, nonatomic, nullable) NSURLCredential *credential;
@property (assign, nonatomic) double minimumProgressInterval;

@end
```

在 SDWebImage 中，默认的实现类是 `SDWebImageDownloaderOperation`。

### 开始执行

当 `NSOperation` 通过 `addOpeartion` 被添加到 `NSOperationQueue` 的时候，就会触发 `NSOperation` 的 `start` 方法：

```objc
#pragma mark - 开启下载
/// 通过 addOperation 开始的
- (void)start {
    @synchronized (self) {
        /// 如果已经被取消了
        if (self.isCancelled) {
            self.finished = YES;
            // Operation cancelled by user before sending the request
            [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorCancelled userInfo:nil]];
            [self reset];
            return;
        }

        Class UIApplicationClass = NSClassFromString(@"UIApplication");
        BOOL hasApplication = UIApplicationClass && [UIApplicationClass respondsToSelector:@selector(sharedApplication)];
        /// 如果 options 要求在后台仍需要执行任务的话
        if (hasApplication && [self shouldContinueWhenAppEntersBackground]) {
            __weak typeof(self) wself = self;
            UIApplication * app = [UIApplicationClass performSelector:@selector(sharedApplication)];
            self.backgroundTaskId = [app beginBackgroundTaskWithExpirationHandler:^{
                /// 取消所有的下载任务
                [wself cancel];
            }];
        }
        NSURLSession *session = self.unownedSession;
        if (!session) {
            NSURLSessionConfiguration *sessionConfig = [NSURLSessionConfiguration defaultSessionConfiguration];
            sessionConfig.timeoutIntervalForRequest = 15;
            
            /**
             *  Create the session for this task
             *  We send nil as delegate queue so that the session creates a serial operation queue for performing all delegate
             *  method calls and completion handler calls.
             */
            /// 如果不存在 session 那么创建一个 session
            session = [NSURLSession sessionWithConfiguration:sessionConfig
                                                    delegate:self
                                               delegateQueue:nil];
            self.ownedSession = session;
        }
        
        /// 如果要无视 NSURLCache 的缓存
        if (self.options & SDWebImageDownloaderIgnoreCachedResponse) {
            // Grab the cached data for later check
            NSURLCache *URLCache = session.configuration.URLCache;
            if (!URLCache) {
                URLCache = [NSURLCache sharedURLCache];
            }
            NSCachedURLResponse *cachedResponse;
            // NSURLCache's `cachedResponseForRequest:` is not thread-safe, see https://developer.apple.com/documentation/foundation/nsurlcache#2317483
            @synchronized (URLCache) {
                cachedResponse = [URLCache cachedResponseForRequest:self.request];
            }
            if (cachedResponse) {
                self.cachedData = cachedResponse.data;
            }
        }
        
        /// 通过 session 创建 dataTask
        self.dataTask = [session dataTaskWithRequest:self.request];
        self.executing = YES;
    }

    /// 如果创建了 dataTask
    if (self.dataTask) {
        /// 根据 options 设置优先级
        if (self.options & SDWebImageDownloaderHighPriority) {
            self.dataTask.priority = NSURLSessionTaskPriorityHigh;
        } else if (self.options & SDWebImageDownloaderLowPriority) {
            self.dataTask.priority = NSURLSessionTaskPriorityLow;
        }
        /// 开启 dataTask
        [self.dataTask resume];
        /// 执行 progressBlock
        for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
            progressBlock(0, NSURLResponseUnknownLength, self.request.URL);
        }
        __block typeof(self) strongSelf = self;
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStartNotification object:strongSelf];
        });
    } else {
        /// 没有创建玩报错
        [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidDownloadOperation userInfo:@{NSLocalizedDescriptionKey : @"Task can't be initialized"}]];
        [self done];
    }
}
```

在 `start` 方法中，创建并保存了 `NSURLSession` 实例，然后通过它创建并开启了 `NSURLSessionTask`。

`beginBackgroundTaskWithExpirationHandler` 不是意味着立即执行后台任务，它相当于**注册了一个后台任务**，而之后的 `handler` 表示 **App 在直到后台运行的时机到来后在运行其中的 block 代码逻辑**。

### NSURLSessionDataDelegate

`SDWebImageDownloaderOperation`  是 `NSURLSession` 的 `NSURLSessionDataDelegate`。首先看第一次收到数据后的回调方法：

```objc
/// 收到数据后第一次回调
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler {
    NSURLSessionResponseDisposition disposition = NSURLSessionResponseAllow;
    /// 拿到预计的图片的大小
    NSInteger expected = (NSInteger)response.expectedContentLength;
    expected = expected > 0 ? expected : 0;
    self.expectedSize = expected;
    self.response = response;
    /// 拿到返回的状态码
    NSInteger statusCode = [response respondsToSelector:@selector(statusCode)] ? ((NSHTTPURLResponse *)response).statusCode : 200;
    /// 200 - 400 之间的是有效的
    BOOL valid = statusCode >= 200 && statusCode < 400;
    /// 无效的状态码直接报错
    if (!valid) {
        self.responseError = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidDownloadStatusCode userInfo:@{SDWebImageErrorDownloadStatusCodeKey : @(statusCode)}];
    }
    //'304 Not Modified' is an exceptional one
    //URLSession current behavior will return 200 status code when the server respond 304 and URLCache hit. But this is not a standard behavior and we just add a check
    if (statusCode == 304 && !self.cachedData) {
        valid = NO;
        self.responseError = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorCacheNotModified userInfo:nil];
    }
    
    if (valid) {
        for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
            progressBlock(0, expected, self.request.URL);
        }
    } else {
        /// 如果是 invalid ，那么取消这次请求
        disposition = NSURLSessionResponseCancel;
    }
    __block typeof(self) strongSelf = self;
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadReceiveResponseNotification object:strongSelf];
    });
    
    if (completionHandler) {
        completionHandler(disposition);
    }
}
```

首先是获取期望的数据大小 `expectedContentLength`，随后通过 `statusCode` 判断是否可以接收 data。调用了 `completionHander` 之后，进入下一个代码方法：

```objc
/// 收到数据
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data {
    if (!self.imageData) {
        /// 创建一个 expectedSize 大小的 NSMutableData
        self.imageData = [[NSMutableData alloc] initWithCapacity:self.expectedSize];
    }
    /// 把 data append 到 imageData 中
    [self.imageData appendData:data];
    /// 计算已接收数据大小
    self.receivedSize = self.imageData.length;
    if (self.expectedSize == 0) {
        // Unknown expectedSize, immediately call progressBlock and return
        for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
            /// 回调 progressBlock
            progressBlock(self.receivedSize, self.expectedSize, self.request.URL);
        }
        return;
    }
    
    /// 如果收到的 data 大于期望的 data，那么直接表示完成
    BOOL finished = (self.receivedSize >= self.expectedSize);
    double currentProgress = (double)self.receivedSize / (double)self.expectedSize;
    double previousProgress = self.previousProgress;
    double progressInterval = currentProgress - previousProgress;
    /// 本次接收到的数据的大小小于最小的回调间隔，那么取消回调
    if (!finished && (progressInterval < self.minimumProgressInterval)) {
        return;
    }
    self.previousProgress = currentProgress;

    /// 如果是渐进式的加载方式
    if (self.options & SDWebImageDownloaderProgressiveLoad) {
        NSData *imageData = [self.imageData copy];
        
        /// 在 coderQueue 队列中，渐进式的 decode
        dispatch_async(self.coderQueue, ^{
            @autoreleasepool {
                UIImage *image = SDImageLoaderDecodeProgressiveImageData(imageData, self.request.URL, finished, self, [[self class] imageOptionsFromDownloaderOptions:self.options], self.context);
                if (image) {
                    // We do not keep the progressive decoding image even when `finished`=YES. Because they are for view rendering but not take full function from downloader options. And some coders implementation may not keep consistent between progressive decoding and normal decoding.
                    
                    [self callCompletionBlocksWithImage:image imageData:nil error:nil finished:NO];
                }
            }
        });
    }
    
    for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
        progressBlock(self.receivedSize, self.expectedSize, self.request.URL);
    }
}
```

不断的接收 data，并且将 data append 到 `imageData` 之后。如果是渐进式的加载方式，还需要变下载边解码。

下载好之后，会有一个回调判断是否要通过 `NSURLCache` 缓存数据：

```objc
/// 判断是否可以缓存 response 的回调
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
 willCacheResponse:(NSCachedURLResponse *)proposedResponse
 completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler {
    
    NSCachedURLResponse *cachedResponse = proposedResponse;

    if (!(self.options & SDWebImageDownloaderUseNSURLCache)) {
        // Prevents caching of responses
        cachedResponse = nil;
    }
    if (completionHandler) {
        completionHandler(cachedResponse);
    }
}
```

### NSURLSessionTaskDelegate

在下载完成后会回调 `NSURLSessionTaskDelegate` 的完成方法。在这里可以将获取的 data 通过回调函数回传给 Manager：

```objc
/// 完成回调
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
    // If we already cancel the operation or anything mark the operation finished, don't callback twice
    if (self.isFinished) return;
    
    @synchronized(self) {
        self.dataTask = nil;
        __block typeof(self) strongSelf = self;
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStopNotification object:strongSelf];
            if (!error) {
                [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadFinishNotification object:strongSelf];
            }
        });
    }
    
    if (error) {
        /// 存在 error 回调抛出异常
        if (self.responseError) {
            error = self.responseError;
        }
        [self callCompletionBlocksWithError:error];
        [self done];
    } else {
        /// 调用成功回调
        if ([self callbacksForKey:kCompletedCallbackKey].count > 0) {
            NSData *imageData = [self.imageData copy];
            self.imageData = nil;
            if (imageData) {
                /// 如果 options 是无视缓存的 response 的，并且下载下来的 image 和 URLCache 一致
                if (self.options & SDWebImageDownloaderIgnoreCachedResponse && [self.cachedData isEqualToData:imageData]) {
                    /// 抛出异常，表示没有改变或
                    self.responseError = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorCacheNotModified userInfo:nil];
                    // call completion block with not modified error
                    [self callCompletionBlocksWithError:self.responseError];
                    [self done];
                } else {
                    dispatch_async(self.coderQueue, ^{
                        @autoreleasepool {
                            /// 在 coderQueue 中解码
                            UIImage *image = SDImageLoaderDecodeImageData(imageData, self.request.URL, [[self class] imageOptionsFromDownloaderOptions:self.options], self.context);
                            CGSize imageSize = image.size;
                            if (imageSize.width == 0 || imageSize.height == 0) {
                                [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorBadImageData userInfo:@{NSLocalizedDescriptionKey : @"Downloaded image has 0 pixels"}]];
                            } else {
                                [self callCompletionBlocksWithImage:image imageData:imageData error:nil finished:YES];
                            }
                            [self done];
                        }
                    });
                }
            } else {
                [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorBadImageData userInfo:@{NSLocalizedDescriptionKey : @"Image data is nil"}]];
                [self done];
            }
        } else {
            [self done];
        }
    }
}
```

至此，下载部分就结束了

## 总结





















