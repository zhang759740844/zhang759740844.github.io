title: SDWebImage 源码解析—缓存策略
date: 2018/3/3 14:07:12  
categories: iOS
tags: 

- 学习笔记
- 源码解析

------

上一篇说了整体的 Manager 所做的事情。现在来有针对性的看看缓存策略是怎样的

<!--more-->

## SDImageCache

说到缓存，一般都会想到两级缓存：内存缓存和磁盘缓存。SDWebImage 也不例外，我们先开看一下初始化方法：

### 初始化方法

`SDImageCache` 是一个单例对象。它的初始化方法中会创建磁盘缓存对象 `_diskCache`，和内存缓存对象 `_diskCache`：

```objc
+ (nonnull instancetype)sharedImageCache {
    static dispatch_once_t once;
    static id instance;
    dispatch_once(&once, ^{
        instance = [self new];
    });
    return instance;
}


- (nonnull instancetype)initWithNamespace:(nonnull NSString *)ns
                       diskCacheDirectory:(nullable NSString *)directory
                                   config:(nullable SDImageCacheConfig *)config {
    if ((self = [super init])) {
        NSAssert(ns, @"Cache namespace should not be nil");
        
        // Create IO serial queue
        _ioQueue = dispatch_queue_create("com.hackemist.SDImageCache", DISPATCH_QUEUE_SERIAL);
        
        if (!config) {
            config = SDImageCacheConfig.defaultCacheConfig;
        }
        _config = [config copy];
        
        // Init the memory cache
        NSAssert([config.memoryCacheClass conformsToProtocol:@protocol(SDMemoryCache)], @"Custom memory cache class must conform to `SDMemoryCache` protocol");
        /// 创建 memoryCache
        _memoryCache = [[config.memoryCacheClass alloc] initWithConfig:_config];
        
        // Init the disk cache
        /// 设置缓存目录
        if (directory != nil) {
            _diskCachePath = [directory stringByAppendingPathComponent:ns];
        } else {
            NSString *path = [[[self userCacheDirectory] stringByAppendingPathComponent:@"com.hackemist.SDImageCache"] stringByAppendingPathComponent:ns];
            _diskCachePath = path;
        }
        
        NSAssert([config.diskCacheClass conformsToProtocol:@protocol(SDDiskCache)], @"Custom disk cache class must conform to `SDDiskCache` protocol");
        /// 创建 diskCache 实例
        _diskCache = [[config.diskCacheClass alloc] initWithCachePath:_diskCachePath config:_config];
        
        // Check and migrate disk cache directory if need
        [self migrateDiskCacheDirectory];

        // Subscribe to app events
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(applicationWillTerminate:)
                                                     name:UIApplicationWillTerminateNotification
                                                   object:nil];

        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(applicationDidEnterBackground:)
                                                     name:UIApplicationDidEnterBackgroundNotification
                                                   object:nil];
    }
    return self;
}
```

可以看到，两个缓存的类型都是通过 `config` 获取的。`config` 默认的类型为 `SDMemoryCache` 和 `SDDiskCache`。

#### SDMemoryCache 初始化

SDMemoryCache 初始化方法如下：

```objc
- (void)commonInit {
    SDImageCacheConfig *config = self.config;
    /// 缓存内存缓存的最大值
    self.totalCostLimit = config.maxMemoryCost;
    /// 内存缓存数量上的最大值
    self.countLimit = config.maxMemoryCount;
    
    /// 对 config 的 maxMemoryCost 和 maxMenoryCount 做监听如果变化了，立刻变化内存缓存的相应值
    [config addObserver:self forKeyPath:NSStringFromSelector(@selector(maxMemoryCost)) options:0 context:SDMemoryCacheContext];
    [config addObserver:self forKeyPath:NSStringFromSelector(@selector(maxMemoryCount)) options:0 context:SDMemoryCacheContext];
    
    /// 新建一个弱引用字典
    self.weakCache = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];
    self.weakCacheLock = dispatch_semaphore_create(1);
    
    [[NSNotificationCenter defaultCenter] addObserver:self
                                             selector:@selector(didReceiveMemoryWarning:)
                                                 name:UIApplicationDidReceiveMemoryWarningNotification
                                               object:nil];
}
```

主要设置了缓存的最大值和最大数量。另外 `weakCache` 是一个令人注意的地方。它是一个 NSMapTable，它的作用会在后续使用的时候揭晓。

另外，`totalCostLimit` 和 `countLimit` 是 NSCache 的两个属性。`SDMemoryCache` 是 NSCache 的子类。

#### SDDiskCache 初始化

`SDDiskCache` 的初始化方法就简单了很多。就是获取一个 `NSFileManager`

```objc
- (void)commonInit {
    if (self.config.fileManager) {
        self.fileManager = self.config.fileManager;
    } else {
        self.fileManager = [NSFileManager new];
    }
}
```

### 入口方法

前篇说到，Manager 会通过 `SDImageCache` 实例，创建一个缓存的 operation：

```objc
/// 进行缓存相关操作，把返回的 Operation 设置给 CacheOperation
operation.cacheOperation = [self.imageCache queryImageForKey:key options:options context:context completion:^(UIImage * _Nullable cachedImage, NSData * _Nullable cachedData, SDImageCacheType cacheType) {
    @strongify(operation);
    /// 如果执行过程中操作取消，安全移除操作
    if (!operation || operation.isCancelled) {
        // Image combined operation cancelled by user
        [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorCancelled userInfo:nil] url:url];
        [self safelyRemoveOperationFromRunning:operation];
        return;
    }
    /// 前面拿好了缓存，现在就要开始下载
    [self callDownloadProcessForOperation:operation url:url options:options context:context cachedImage:cachedImage cachedData:cachedData cacheType:cacheType progress:progressBlock completed:completedBlock];
}];

```

现在来到具体方法中：

```objc
- (id<SDWebImageOperation>)queryImageForKey:(NSString *)key options:(SDWebImageOptions)options context:(nullable SDWebImageContext *)context completion:(nullable SDImageCacheQueryCompletionBlock)completionBlock {
    SDImageCacheOptions cacheOptions = 0;
    /// 设置 options 选项
    if (options & SDWebImageQueryMemoryData) cacheOptions |= SDImageCacheQueryMemoryData;
    /// 从内存中同步查找
    if (options & SDWebImageQueryMemoryDataSync) cacheOptions |= SDImageCacheQueryMemoryDataSync;
    /// 内存不存在时，从磁盘同步查找
    if (options & SDWebImageQueryDiskDataSync) cacheOptions |= SDImageCacheQueryDiskDataSync;
    /// 将大图压缩到适当大小
    if (options & SDWebImageScaleDownLargeImages) cacheOptions |= SDImageCacheScaleDownLargeImages;
    /// 禁止下载完就将图片解码（立即解码可能会导致内存使用变大）
    if (options & SDWebImageAvoidDecodeImage) cacheOptions |= SDImageCacheAvoidDecodeImage;
    /// 动图只解码第一帧（否则会全部解码）
    if (options & SDWebImageDecodeFirstFrameOnly) cacheOptions |= SDImageCacheDecodeFirstFrameOnly;
    /// 动图提前准备好所有帧（降低多处使用同一动图的 CPU 加载消耗）
    if (options & SDWebImagePreloadAllFrames) cacheOptions |= SDImageCachePreloadAllFrames;

    return [self queryCacheOperationForKey:key options:cacheOptions context:context done:completionBlock];
}
```

这里对于外部传来的 options 做了一些处理。然后传入内部的处理方法。

### 内部方法

内部方法分为两部分，从 Memory 中查找缓存以及从 Disk 中查找缓存。方法很长：

```objc
- (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key options:(SDImageCacheOptions)options context:(nullable SDWebImageContext *)context done:(nullable SDImageCacheQueryCompletionBlock)doneBlock {
    if (!key) {
        if (doneBlock) {
            doneBlock(nil, nil, SDImageCacheTypeNone);
        }
        return nil;
    }
    
    /// 获取自定义的 transformer
    id<SDImageTransformer> transformer = context[SDWebImageContextImageTransformer];
    if (transformer) {
        /// 将自定义的 transformer 的 key 拼接到原来的 key 上
        NSString *transformerKey = [transformer transformerKey];
        key = SDTransformedKeyForKey(key, transformerKey);
    }
    
    // First check the in-memory cache...
    /// 从内存获取image
    UIImage *image = [self imageFromMemoryCacheForKey:key];
    
    if (image) {
        /// 只解码第一帧
        if (options & SDImageCacheDecodeFirstFrameOnly) {
            // Ensure static image
            Class animatedImageClass = image.class;
            if (image.sd_isAnimated || ([animatedImageClass isSubclassOfClass:[UIImage class]] && [animatedImageClass conformsToProtocol:@protocol(SDAnimatedImage)])) {
                image = [[UIImage alloc] initWithCGImage:image.CGImage scale:image.scale orientation:image.imageOrientation];
            }
        } else if (options & SDImageCacheMatchAnimatedImageClass) {
            // Check image class matching
            Class animatedImageClass = image.class;
            Class desiredImageClass = context[SDWebImageContextAnimatedImageClass];
            if (desiredImageClass && ![animatedImageClass isSubclassOfClass:desiredImageClass]) {
                image = nil;
            }
        }
    }

    /// 只能从 Memory 中读取
    BOOL shouldQueryMemoryOnly = (image && !(options & SDImageCacheQueryMemoryData));
    if (shouldQueryMemoryOnly) {
        if (doneBlock) {
            doneBlock(image, nil, SDImageCacheTypeMemory);
        }
        return nil;
    }
    
    NSOperation *operation = [NSOperation new];
    /// 如果内存中存在图片，并且需要同步获取，或者内存中不存在图片，并且需要同步从磁盘获取
    BOOL shouldQueryDiskSync = ((image && options & SDImageCacheQueryMemoryDataSync) ||
                                (!image && options & SDImageCacheQueryDiskDataSync));
    /// 查找缓存的 block
    void(^queryDiskBlock)(void) =  ^{
        /// 如果是已经取消了，那么直接执行完成回调
        if (operation.isCancelled) {
            if (doneBlock) {
                doneBlock(nil, nil, SDImageCacheTypeNone);
            }
            return;
        }
        
        @autoreleasepool {
            /// 从磁盘读取data
            NSData *diskData = [self diskImageDataBySearchingAllPathsForKey:key];
            UIImage *diskImage;
            SDImageCacheType cacheType = SDImageCacheTypeNone;
            if (image) {
                /// 如果有 image，说明从 Memory 中q读取的。设置它的类型
                diskImage = image;
                cacheType = SDImageCacheTypeMemory;
            } else if (diskData) {
                /// 如果有 diskData，说明是从 disk 中读取的。设置它的类型为磁盘类型
                cacheType = SDImageCacheTypeDisk;
                /// 还需要将data转为image
                diskImage = [self diskImageForKey:key data:diskData options:options context:context];
                /// 如果配置为需要缓存到内存中。那么就将磁盘中读取的 Image 存到内存中
                if (diskImage && self.config.shouldCacheImagesInMemory) {
                    NSUInteger cost = diskImage.sd_memoryCost;
                    [self.memoryCache setObject:diskImage forKey:key cost:cost];
                }
            }
            
            if (doneBlock) {
                /// 是否是同步方法
                if (shouldQueryDiskSync) {
                    doneBlock(diskImage, diskData, cacheType);
                } else {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        doneBlock(diskImage, diskData, cacheType);
                    });
                }
            }
        }
    };
    
    /// isQueue 是一个串行队列。根据 shouldQueryDiskSync 判断是否同步执行
    if (shouldQueryDiskSync) {
        dispatch_sync(self.ioQueue, queryDiskBlock);
    } else {
        dispatch_async(self.ioQueue, queryDiskBlock);
    }
    
    return operation;
}
```

#### 内存查找 image

内存查找方法直接从前面创建的 `SDMemoryCache` 对象 `memoryCache` 对象中获取：

```objc
- (nullable UIImage *)imageFromMemoryCacheForKey:(nullable NSString *)key {
    return [self.memoryCache objectForKey:key];
}
```

`SDMemoryCache` 本身继承于 `NSCache`，它有一套系统的缓存方式，会自动在某个时刻进行内存的释放。不过 SDWebImage 重写了 `objectForKey` 方法：

```objc
- (id)objectForKey:(id)key {
    /// NSCache 的 objectforkey 方法
    id obj = [super objectForKey:key];
    /// 如果 config 设置不能自己缓存则直接返回 NSCache 的缓存对象
    if (!self.config.shouldUseWeakMemoryCache) {
        return obj;
    }
    /// 如果 NSCache 中没有找到缓存
    if (key && !obj) {
        // Check weak cache
        SD_LOCK(self.weakCacheLock);
        /// 从自己的NSMapTable 中搜索
        obj = [self.weakCache objectForKey:key];
        SD_UNLOCK(self.weakCacheLock);
        /// 如果从 NSMapTable 中找到了
        if (obj) {
            /// 设置缓存大小
            NSUInteger cost = 0;
            if ([obj isKindOfClass:[UIImage class]]) {
                cost = [(UIImage *)obj sd_memoryCost];
            }
            /// 把从 NSMapTable 中找到的对象再设置回 NSCache
            [super setObject:obj forKey:key cost:cost];
        }
    }
    return obj;
}
```

这里就用到了上面 `SDMemoryCache` 初始化的 `NSMapTable` 了。那么它的作用是什么呢？由于 NSCache 是系统维护回收状态的缓存，可能在任何时候回收。那么就可能出现 image 还存在，但是已经被 NSCache 回收的情况，容易产生**因为内存紧张频繁释放内存导致频繁从磁盘加载图片**的情况。

因此，在 NSCache 本身的缓存回收机制下，再设置一个弱引用的字典。它并不影响缓存的引用计数。但是如果 image 没有被回收，那么也不会像 NSCache 一样可能被回收。

类似的，在把 image 丢到 `SDMemoryCache` 中的时候需要设置到 `NSMapTable` 中：

```objc
- (void)setObject:(id)obj forKey:(id)key cost:(NSUInteger)g {
    [super setObject:obj forKey:key cost:g];
    if (!self.config.shouldUseWeakMemoryCache) {
        return;
    }
    /// 两套方案，一套 NSCache 的缓存，y再通过弱引用一套。
    if (key && obj) {
        // Store weak cache
        SD_LOCK(self.weakCacheLock);
        [self.weakCache setObject:obj forKey:key];
        SD_UNLOCK(self.weakCacheLock);
    }
}
```

#### Disk 中查找 image

```objc
- (nullable NSData *)diskImageDataBySearchingAllPathsForKey:(nullable NSString *)key {
    if (!key) {
        return nil;
    }
    
    /// 从磁盘中读取 Data
    NSData *data = [self.diskCache dataForKey:key];
    if (data) {
        return data;
    }
    
    /// 从自定义的 key → path block 中获取文件路径读取路径
    if (self.additionalCachePathBlock) {
        NSString *filePath = self.additionalCachePathBlock(key);
        if (filePath) {
            data = [NSData dataWithContentsOfFile:filePath options:self.config.diskCacheReadingOptions error:nil];
        }
    }

    return data;
}


```

 `SDDiskCache` 中的查找方法。这是一个比较简单明确的方法：

```objc
- (NSData *)dataForKey:(NSString *)key {
    NSParameterAssert(key);
    /// 拿到 disk 缓存的路径
    NSString *filePath = [self cachePathForKey:key];
    /// 到这个路径上找相应的 data
    NSData *data = [NSData dataWithContentsOfFile:filePath options:self.config.diskCacheReadingOptions error:nil];
    /// 获取到了 data 说明存才 cache，直接返回
    if (data) {
        return data;
    }
    
    data = [NSData dataWithContentsOfFile:filePath.stringByDeletingPathExtension options:self.config.diskCacheReadingOptions error:nil];
    if (data) {
        return data;
    }
    
    return nil;
}
```

#### Disk 中的 NSData 转 UIImage

从 Disk 中拿到的是 NSData 对象。我们需要把它转为 UIImage：

```objc
- (nullable UIImage *)diskImageForKey:(nullable NSString *)key data:(nullable NSData *)data options:(SDImageCacheOptions)options context:(SDWebImageContext *)context {
    if (data) {
      	/// NSData 转 UIImage
        UIImage *image = SDImageCacheDecodeImageData(data, key, [[self class] imageOptionsFromCacheOptions:options], context);
        return image;
    } else {
        return nil;
    }
}

UIImage * _Nullable SDImageCacheDecodeImageData(NSData * _Nonnull imageData, NSString * _Nonnull cacheKey, SDWebImageOptions options, SDWebImageContext * _Nullable context) {
    UIImage *image;
    /// 判断是否只要解码第一帧
    BOOL decodeFirstFrame = SD_OPTIONS_CONTAINS(options, SDWebImageDecodeFirstFrameOnly);
    NSNumber *scaleValue = context[SDWebImageContextImageScaleFactor];
    /// 获取解码的 scale
    CGFloat scale = scaleValue.doubleValue >= 1 ? scaleValue.doubleValue : SDImageScaleFactorForKey(cacheKey);
    SDImageCoderOptions *coderOptions = @{SDImageCoderDecodeFirstFrameOnly : @(decodeFirstFrame), SDImageCoderDecodeScaleFactor : @(scale)};
    if (context) {
        SDImageCoderMutableOptions *mutableCoderOptions = [coderOptions mutableCopy];
        [mutableCoderOptions setValue:context forKey:SDImageCoderWebImageContext];
        coderOptions = [mutableCoderOptions copy];
    }
    
    /// 如果不仅仅解码第一帧，则从 context 中取出 AnimatedImageClass，默认使用的是 SDAnimatedImage，然后进行图片数据的解码，根据需要还可以预解码所有帧
    if (!decodeFirstFrame) {
        Class animatedImageClass = context[SDWebImageContextAnimatedImageClass];
        // check whether we should use `SDAnimatedImage`
        if ([animatedImageClass isSubclassOfClass:[UIImage class]] && [animatedImageClass conformsToProtocol:@protocol(SDAnimatedImage)]) {
            image = [[animatedImageClass alloc] initWithData:imageData scale:scale options:coderOptions];
            if (image) {
                // Preload frames if supported
                if (options & SDWebImagePreloadAllFrames && [image respondsToSelector:@selector(preloadAllFrames)]) {
                    [((id<SDAnimatedImage>)image) preloadAllFrames];
                }
            } else {
                // Check image class matching
                if (options & SDWebImageMatchAnimatedImageClass) {
                    return nil;
                }
            }
        }
    }
    if (!image) {
        /// 解码出 UIImage
        image = [[SDImageCodersManager sharedManager] decodedImageWithData:imageData options:coderOptions];
    }
    if (image) {
        /// 查看是否需要解码
        BOOL shouldDecode = !SD_OPTIONS_CONTAINS(options, SDWebImageAvoidDecodeImage);
        if ([image.class conformsToProtocol:@protocol(SDAnimatedImage)]) {
            // `SDAnimatedImage` do not decode
            /// 动图不需要解码
            shouldDecode = NO;
        } else if (image.sd_isAnimated) {
            // animated image do not decode
            shouldDecode = NO;
        }
        /// 需要解码就开始解码
        if (shouldDecode) {
            BOOL shouldScaleDown = SD_OPTIONS_CONTAINS(options, SDWebImageScaleDownLargeImages);
            if (shouldScaleDown) {
                image = [SDImageCoderHelper decodedAndScaledDownImageWithImage:image limitBytes:0];
            } else {
                image = [SDImageCoderHelper decodedImageWithImage:image];
            }
        }
    }
    
    return image;
}
```

具体解码相关键后续文章。

#### 返回 NSOperation

我们可以看到内部方法中通过 `NSOperation *operation = [NSOperation new];` 创建了一个 NSOperation 实例，然后就直接返回了。没有对这个 NSOperation 做任何其他的处理。唯一使用到的地方就是在 Disk 查询的时候判断是否已经取消。那么为什么要这样做呢？

因为获取缓存的方法中要根据 config 的不同，对缓存进行同步，或者异步的读取。即要进行 `dispatch_sync` 和 `dispatch_async` 的切换。这用一个 NSOperation 不方便完成。但是对于外部来说，外部希望有一个和 download 统一的获取缓存的 NSOperation 实例用于取消。因此，就在内部的 GCD 中加入了 NSOperation 的 cancel，来动态取消执行。

### 保存相关方法

按理说，保存应该放在下载模块，不过 SDWebImage 中将保存放在了 Cache 模块中。因此，也一起说完。

#### 外部调用方法

下面就是 Manager 中会调用的缓存方法：

```objc
#pragma mark - 图片存储
- (void)callStoreCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                      url:(nonnull NSURL *)url
                                  options:(SDWebImageOptions)options
                                  context:(SDWebImageContext *)context
                          downloadedImage:(nullable UIImage *)downloadedImage
                           downloadedData:(nullable NSData *)downloadedData
                                 finished:(BOOL)finished
                                 progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                completed:(nullable SDInternalCompletionBlock)completedBlock {
    /// 获取存储类型,默认是既存在 Memory 中又存在 disk 中
    SDImageCacheType storeCacheType = SDImageCacheTypeAll;
    if (context[SDWebImageContextStoreCacheType]) {
        storeCacheType = [context[SDWebImageContextStoreCacheType] integerValue];
    }
    /// 如果图片被 transform 过了，那么这个设置时用来针对原始图像的
    SDImageCacheType originalStoreCacheType = SDImageCacheTypeNone;
    if (context[SDWebImageContextOriginalStoreCacheType]) {
        originalStoreCacheType = [context[SDWebImageContextOriginalStoreCacheType] integerValue];
    }
    /// 获取自定义的 url → string 的转换器
    id<SDWebImageCacheKeyFilter> cacheKeyFilter = context[SDWebImageContextCacheKeyFilter];
    NSString *key = [self cacheKeyForURL:url cacheKeyFilter:cacheKeyFilter];
    /// 获取自定义的 transformer
    id<SDImageTransformer> transformer = context[SDWebImageContextImageTransformer];
    id<SDWebImageCacheSerializer> cacheSerializer = context[SDWebImageContextCacheSerializer];
    
    /// 是否要转换 image 取决于 1. 是否下载好了图片 2. 是否有转化器 3. 动图默认不会 transform，但是如果设置了 SDWebImageTransformAnimatedImage 标识位，那么就会转换
    BOOL shouldTransformImage = downloadedImage && (!downloadedImage.sd_isAnimated || (options & SDWebImageTransformAnimatedImage)) && transformer;
    /// 是否需要缓存原图
    BOOL shouldCacheOriginal = downloadedImage && finished;
    
    /// 如果需要缓存原图
    if (shouldCacheOriginal) {
        /// 获取转换原图的 type
        SDImageCacheType targetStoreCacheType = shouldTransformImage ? originalStoreCacheType : storeCacheType;
        /// 如果有自定义的NSData序列化方式
        if (cacheSerializer && (targetStoreCacheType == SDImageCacheTypeDisk || targetStoreCacheType == SDImageCacheTypeAll)) {
            dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                @autoreleasepool {
                    NSData *cacheData = [cacheSerializer cacheDataWithImage:downloadedImage originalData:downloadedData imageURL:url];
                    /// 具体的缓存方式
                    [self.imageCache storeImage:downloadedImage imageData:cacheData forKey:key cacheType:targetStoreCacheType completion:nil];
                }
            });
        /// 没有自定义的序列化方法，直接存储图片
        } else {
            [self.imageCache storeImage:downloadedImage imageData:downloadedData forKey:key cacheType:targetStoreCacheType completion:nil];
        }
    }
    /// 如果需要对原图进行 transform
    if (shouldTransformImage) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
            @autoreleasepool {
                /// transform 原图
                UIImage *transformedImage = [transformer transformedImageWithImage:downloadedImage forKey:key];
                if (transformedImage && finished) {
                    NSString *transformerKey = [transformer transformerKey];
                    NSString *cacheKey = SDTransformedKeyForKey(key, transformerKey);
                    BOOL imageWasTransformed = ![transformedImage isEqual:downloadedImage];
                    NSData *cacheData;
                    // pass nil if the image was transformed, so we can recalculate the data from the image
                    if (cacheSerializer && (storeCacheType == SDImageCacheTypeDisk || storeCacheType == SDImageCacheTypeAll)) {
                        cacheData = [cacheSerializer cacheDataWithImage:transformedImage  originalData:(imageWasTransformed ? nil : downloadedData) imageURL:url];
                    } else {
                        cacheData = (imageWasTransformed ? nil : downloadedData);
                    }
                    /// 具体的缓存方式
                    [self.imageCache storeImage:transformedImage imageData:cacheData forKey:cacheKey cacheType:storeCacheType completion:nil];
                }
                
                [self callCompletionBlockForOperation:operation completion:completedBlock image:transformedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
            }
        });
    } else {
        [self callCompletionBlockForOperation:operation completion:completedBlock image:downloadedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
    }
}
```

代码有点长，不过注释很清楚。这里的逻辑主要用于处理是否需要缓存原图以及是否需要对原图进行 transform 上。

#### 内部存储方法

内部存储方法在 `SDImageCache` 中：

```objc
- (void)storeImage:(UIImage *)image imageData:(NSData *)imageData forKey:(nullable NSString *)key cacheType:(SDImageCacheType)cacheType completion:(nullable SDWebImageNoParamsBlock)completionBlock {
    switch (cacheType) {
        case SDImageCacheTypeNone: {
            [self storeImage:image imageData:imageData forKey:key toMemory:NO toDisk:NO completion:completionBlock];
        }
            break;
        case SDImageCacheTypeMemory: {
            [self storeImage:image imageData:imageData forKey:key toMemory:YES toDisk:NO completion:completionBlock];
        }
            break;
        case SDImageCacheTypeDisk: {
            [self storeImage:image imageData:imageData forKey:key toMemory:NO toDisk:YES completion:completionBlock];
        }
            break;
        case SDImageCacheTypeAll: {
            [self storeImage:image imageData:imageData forKey:key toMemory:YES toDisk:YES completion:completionBlock];
        }
            break;
        default: {
            if (completionBlock) {
                completionBlock();
            }
        }
            break;
    }
}
```

Manager 会把下载下来的 image 或者 data 传入。然后根据缓存策略进行内存或者磁盘缓存：

```objc
- (void)storeImage:(nullable UIImage *)image
         imageData:(nullable NSData *)imageData
            forKey:(nullable NSString *)key
          toMemory:(BOOL)toMemory
            toDisk:(BOOL)toDisk
        completion:(nullable SDWebImageNoParamsBlock)completionBlock {
    if (!image || !key) {
        if (completionBlock) {
            completionBlock();
        }
        return;
    }
    // if memory cache is enabled
    /// 可以保存就保存到 memory 中
    if (toMemory && self.config.shouldCacheImagesInMemory) {
        NSUInteger cost = image.sd_memoryCost;
        [self.memoryCache setObject:image forKey:key cost:cost];
    }
    
    /// 如果可以保存到 disk 中
    if (toDisk) {
        dispatch_async(self.ioQueue, ^{
            @autoreleasepool {
                NSData *data = imageData;
                if (!data && image) {
                    // If we do not have any data to detect image format, check whether it contains alpha channel to use PNG or JPEG format
                    SDImageFormat format;
                    if ([SDImageCoderHelper CGImageContainsAlpha:image.CGImage]) {
                        format = SDImageFormatPNG;
                    } else {
                        format = SDImageFormatJPEG;
                    }
                    data = [[SDImageCodersManager sharedManager] encodedDataWithImage:image format:format options:nil];
                }
                [self _storeImageDataToDisk:data forKey:key];
            }
            
            if (completionBlock) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    completionBlock();
                });
            }
        });
    } else {
        if (completionBlock) {
            completionBlock();
        }
    }
}
```

写入磁盘的时候一直使用的是一个串行队列 ，这样防止了多线程情况下可能会产生的 crash：

```objc
 _ioQueue = dispatch_queue_create("com.hackemist.SDImageCache", DISPATCH_QUEUE_SERIAL);
```



缓存部分到此为止。

