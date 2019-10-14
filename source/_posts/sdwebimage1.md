title: SDWebImage 源码解析--总览
date: 2018/3/1 14:07:12  
categories: iOS
tags: 

 - 学习笔记
 - 源码解析
---

SDWebImage 是我们最常用的框架之一。它是一个比较庞大的库，但是整体逻辑却非常清晰。套用官方 [github](<https://github.com/SDWebImage/SDWebImage>) 中的一张整体的流程图可以看到：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/sdwebimage1.png?raw=true)

<!--more-->

从图中可以看出我们通过 `UIImageView+WebCache` 分类提供的方法调用了 `UIView+WebCache` 中的方法，进而 `SDWebImageManager` 接管了加载的整个流程。根据不同的策略决定是否从 Cache 中查找，以及是从 Memory Cache 中查找还是从 Disk Cache 中查找。最终走到 `SDWebImageDownloader` 中下载图片。下载完成还需要对图片进行缓存。这篇我们先简单看一下 `SDWebImageManager` 之前的调用过程。

## UIView+WebCache

### UIImageView+WebCache 的入口方法

`UIImageView+WebCache` 是我们最常使用的分类，它提供了 SDWebImage 使用的入口方法。一般我们会使用这个方法：

```objc
- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder {
    [self sd_setImageWithURL:url placeholderImage:placeholder options:0 progress:nil completed:nil];
}
```

这个方法省略了配置参数以及进度和完成回调。它会调用 `UIView+WebCache` 的相应方法：

```objc
- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder
                   options:(SDWebImageOptions)options
                   context:(nullable SDWebImageContext *)context
                  progress:(nullable SDImageLoaderProgressBlock)progressBlock
                 completed:(nullable SDExternalCompletionBlock)completedBlock {
    [self sd_internalSetImageWithURL:url
                    placeholderImage:placeholder
                             options:options
                             context:context
                       setImageBlock:nil
                            progress:progressBlock
                           completed:^(UIImage * _Nullable image, NSData * _Nullable data, NSError * _Nullable error, SDImageCacheType cacheType, BOOL finished, NSURL * _Nullable imageURL) {
                               if (completedBlock) {
                                   completedBlock(image, error, cacheType, imageURL);
                               }
                           }];
}
```

### UIView+WebCache 入口方法

这一段代码有点长，它会创建一个 `SDWebImageManager` 实例，用来执行下载和缓存操作。代码中有详尽的注释：

```objc
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                           context:(nullable SDWebImageContext *)context
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDImageLoaderProgressBlock)progressBlock
                         completed:(nullable SDInternalCompletionBlock)completedBlock {
    /// context 其实是一个字典
    context = [context copy]; // copy to avoid mutable object
    /// 获取 Operation 对应的 key
    NSString *validOperationKey = context[SDWebImageContextSetImageOperationKey];
    if (!validOperationKey) {
        /// 如果 context 中没有指定 Operation 对应的 key，那么就指定为当前的类名
        validOperationKey = NSStringFromClass([self class]);
    }
    /// 记录下最近的 Opeartion 的 key。为了方便取消最近一次的 Operation
    self.sd_latestOperationKey = validOperationKey;
    /// 取消之前 validOperationKey 的下载队列
    [self sd_cancelImageLoadOperationWithKey:validOperationKey];
    self.sd_imageURL = url;

    /// 如果没有设置延迟加载占位图，就会先加载占位图
    if (!(options & SDWebImageDelayPlaceholder)) {
        dispatch_main_async_safe(^{
            /// 直接设置展位图
            [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock cacheType:SDImageCacheTypeNone imageURL:url];
        });
    }

    if (url) {
        // reset the progress
        NSProgress *imageProgress = objc_getAssociatedObject(self, @selector(sd_imageProgress));
        if (imageProgress) {
            imageProgress.totalUnitCount = 0;
            imageProgress.completedUnitCount = 0;
        }

        /// 开启加载指示器
        [self sd_startImageIndicator];
        id<SDWebImageIndicator> imageIndicator = self.sd_imageIndicator;

        /// 下载和缓存的真正执行者
        SDWebImageManager *manager = context[SDWebImageContextCustomManager];
        if (!manager) {
            manager = [SDWebImageManager sharedManager];
        }

        /// 更新进度条的回调 block
        SDImageLoaderProgressBlock combinedProgressBlock = ^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
            if (imageProgress) {
                imageProgress.totalUnitCount = expectedSize;
                imageProgress.completedUnitCount = receivedSize;
            }
#if SD_UIKIT || SD_MAC
            if ([imageIndicator respondsToSelector:@selector(updateIndicatorProgress:)]) {
                double progress = 0;
                if (expectedSize != 0) {
                    progress = (double)receivedSize / expectedSize;
                }
                progress = MAX(MIN(progress, 1), 0); // 0.0 - 1.0
                dispatch_async(dispatch_get_main_queue(), ^{
                    [imageIndicator updateIndicatorProgress:progress];
                });
            }
#endif
            if (progressBlock) {
                progressBlock(receivedSize, expectedSize, targetURL);
            }
        };
        @weakify(self);
        /// manager 执行下载和缓存
        id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options context:context progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            @strongify(self);
            /// operation 执行完成的回调
            if (!self) { return; }

            /// 加载完成停止加载指示
            if (finished) {
                [self sd_stopImageIndicator];
            }

/// 如果完成了，或者 options 不允许自动将下载好的图片设置进去（需要手动设置），那么就需要调用外部传进来的 completeBlock
            BOOL shouldCallCompletedBlock = finished || (options & SDWebImageAvoidAutoSetImage);
            /// 有 image，但是不让自动设置，或者没有 image 并且已经加载过占位图了
            BOOL shouldNotSetImage = ((image && (options & SDWebImageAvoidAutoSetImage)) ||
                                      (!image && !(options & SDWebImageDelayPlaceholder)));
            SDWebImageNoParamsBlock callCompletedBlockClojure = ^{
                if (!self) { return; }
                /// 可以设置图片的时候
                if (!shouldNotSetImage) {
                    [self sd_setNeedsLayout];
                }
                /// 能够调用完成回调
                if (completedBlock && shouldCallCompletedBlock) {
                    completedBlock(image, data, error, cacheType, finished, url);
                }
            };
            /// 无法设置图片，直接调用完成回调
            if (shouldNotSetImage) {
                dispatch_main_async_safe(callCompletedBlockClojure);
                return;
            }

            UIImage *targetImage = nil;
            NSData *targetData = nil;
            /// 加载完了，有图片显示图片，没图片看看是否是 SDWebImageDelayPlaceholder。是就表示之前没有显示占位图，那么现在显示展位图
            if (image) {
                // case 2a: we got an image and the SDWebImageAvoidAutoSetImage is not set
                targetImage = image;
                targetData = data;
            } else if (options & SDWebImageDelayPlaceholder) {
                // case 2b: we got no image and the SDWebImageDelayPlaceholder flag is set
                targetImage = placeholder;
                targetData = nil;
            }

#if SD_UIKIT || SD_MAC
            // check whether we should use the image transition
            SDWebImageTransition *transition = nil;
            if (finished && (options & SDWebImageForceTransition || cacheType == SDImageCacheTypeNone)) {
                transition = self.sd_imageTransition;
            }
#endif
            dispatch_main_async_safe(^{
#if SD_UIKIT || SD_MAC
                [self sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];
#endif
                callCompletedBlockClojure();
            });
        }];
        /// 设置 key 对应的 operation
        [self sd_setImageLoadOperation:operation forKey:validOperationKey];
    } else {
        /// URL为空，那么直接报错
        [self sd_stopImageIndicator];
        dispatch_main_async_safe(^{
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}];
                completedBlock(nil, nil, error, SDImageCacheTypeNone, YES, url);
            }
        });
    }
}
```

代码注释非常详尽，主要做了以下几件事：

- 准备加载中与加载完成的回调 block
- 创建 `SDWebImageManager` 实例，并通过它创建满足 `SDWebImageOperation` 协议的 operation
- 将上一步创建的 operation 作为值，默认是当前类名，可通过 options 自定义的字符串作为 key，保存到 `SDOperationsDictionary` 字典中。

现在还要补充以下几点：

#### SDOperationsDictionary 的作用

```objc
typedef NSMapTable<NSString *, id<SDWebImageOperation>> SDOperationsDictionary;
```

我们看到上面的方法会生成一个 `validOperationKey`，并且会将 Manager 生成的 Operation 放入**通过关联对象保存在每个视图对象中**的 `SDOperationsDictionary ` 字典中。那么这个字典存在的意义是什么呢？

绝大多数情况下，我们不需要这个字典。因为一个 UIView 只会下载并展示一张图。但是还是有特殊情况，比如 UIButton，可以设置不同状态下的图片。那么我们就需要一个字典保存不同状态的 Operation 了。

#### SDWebImageContext 的作用

```objc
typedef NSDictionary<SDWebImageContextOption, id> SDWebImageContext;
```

一般情况下，我们不会对 `SDWebImageContext` 做任何操作，直接传入 nil。在上面的方法中：

```objc
/// 获取 Operation 对应的 key
NSString *validOperationKey = context[SDWebImageContextSetImageOperationKey];
if (!validOperationKey) {
    /// 如果 context 中没有指定 Operation 对应的 key，那么就指定为当前的类名
    validOperationKey = NSStringFromClass([self class]);
}
```

拿到字典中的 `SDWebImageContextSetImageOperationKey` 对应的字符串，作为 `SDOperationsDictionary` 中 Operation 的键。用来针对同一 UIView 内的多个图片下载的 Operation 做区分。

#### 设置图片的操作

在设置图片前我们看下两个状态量的设置：

```objc
/// 如果完成了，或者 options 不允许自动将下载好的图片设置进去（需要手动设置），那么就需要调用外部传进来的 completeBlock
BOOL shouldCallCompletedBlock = finished || (options & SDWebImageAvoidAutoSetImage);
/// 有 image，但是不让自动设置，或者没有 image 并且已经加载过占位图了
BOOL shouldNotSetImage = ((image && (options & SDWebImageAvoidAutoSetImage)) ||
                          (!image && !(options & SDWebImageDelayPlaceholder)));
```

看了注释你可能还是很懵。这是在 Operation 执行完成的回调中进行的。其中 `finished` 和 `image` 都是 Operation 的回调返回的。

`shouldCallCompletedBlock` 通过判断是否是 `finished` 之后，或者传入的 `SDWebImageOptions` 是否是禁止自动设置图片。如果是，那么就会调用使用者传入的完成回调。

`shouldNotSetImage` 分为两种情况，一种是下载好了图片，但是 `SDWebImageOptions` 不让自动设置图片 `SDWebImageAvoidAutoSetImage`；或者没有把图片下载下来，并且 `SDWebImageOptions` 设置为需要延迟加载占位图 `SDWebImageDelayPlaceholder`。这两种情况会直接调用完成回调，并且直接就返回了。

如果没有问题，那么就走到了设置图片的方法中：

```objc
- (void)sd_setImage:(UIImage *)image imageData:(NSData *)imageData basedOnClassOrViaCustomSetImageBlock:(SDSetImageBlock)setImageBlock transition:(SDWebImageTransition *)transition cacheType:(SDImageCacheType)cacheType imageURL:(NSURL *)imageURL {
    UIView *view = self;
    SDSetImageBlock finalSetImageBlock;
    if (setImageBlock) {
        /// 如果有自己的设置图片的 block
        finalSetImageBlock = setImageBlock;
    } else if ([view isKindOfClass:[UIImageView class]]) {
        /// 针对 UIImageView 的设置图片的 block
        UIImageView *imageView = (UIImageView *)view;
        finalSetImageBlock = ^(UIImage *setImage, NSData *setImageData, SDImageCacheType setCacheType, NSURL *setImageURL) {
            imageView.image = setImage;
        };
    } else if ([view isKindOfClass:[UIButton class]]) {
        /// 针对 UIButton 的设置图片的 block
        UIButton *button = (UIButton *)view;
        finalSetImageBlock = ^(UIImage *setImage, NSData *setImageData, SDImageCacheType setCacheType, NSURL *setImageURL) {
            [button setImage:setImage forState:UIControlStateNormal];
        };
    }

    /// 如果有自定义的展示动画
    if (transition) {
        [UIView transitionWithView:view duration:0 options:0 animations:^{
            // 0 duration to let UIKit render placeholder and prepares block
            if (transition.prepares) {
                transition.prepares(view, image, imageData, cacheType, imageURL);
            }
        } completion:^(BOOL finished) {
            [UIView transitionWithView:view duration:transition.duration options:transition.animationOptions animations:^{
                if (finalSetImageBlock && !transition.avoidAutoSetImage) {
                    finalSetImageBlock(image, imageData, cacheType, imageURL);
                }
                if (transition.animations) {
                    transition.animations(view, image);
                }
            } completion:transition.completion];
        }];
    } else {
        if (finalSetImageBlock) {
            finalSetImageBlock(image, imageData, cacheType, imageURL);
        }
    }
}
```

## SDWebImageManager

### 入口方法

SDWebImageManager 是一个单例的对象。外部调用的方法如下：

```objc
- (SDWebImageCombinedOperation *)loadImageWithURL:(nullable NSURL *)url
                                          options:(SDWebImageOptions)options
                                          context:(nullable SDWebImageContext *)context
                                         progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                        completed:(nonnull SDInternalCompletionBlock)completedBlock {
    
    if ([url isKindOfClass:NSString.class]) {
        url = [NSURL URLWithString:(NSString *)url];
    }

    // Prevents app crashing on argument type error like sending NSNull instead of NSURL
    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }

    /// 创建一个 Operation 对象，用来保存 Cache Operation 和 Download Operation
    SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
    operation.manager = self;

    /// 是否是已知的失败的URL
    BOOL isFailedUrl = NO;
    if (url) {
        SD_LOCK(self.failedURLsLock);
        isFailedUrl = [self.failedURLs containsObject:url];
        SD_UNLOCK(self.failedURLsLock);
    }

    /// 没有url，或者失败了并且没有让重新下载的情况，直接抛出错误
    if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
        [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}] url:url];
        return operation;
    }

    SD_LOCK(self.runningOperationsLock);
    /// 把当前下载的 operation 加入到 runingOperations 中
    [self.runningOperations addObject:operation];
    SD_UNLOCK(self.runningOperationsLock);
    
    // Preprocess the options and context arg to decide the final the result for manager
    /// 存放 options 和 context 的实例。这里可以传入自己的处理对象，来自定义 options
    SDWebImageOptionsResult *result = [self processedResultForURL:url options:options context:context];
    
    /// 调用缓存方法
    [self callCacheProcessForOperation:operation url:url options:result.options context:result.context progress:progressBlock completed:completedBlock];

    return operation;
}
```

入口方法可说的不多。主要就是判断一下是不是失败的 url，如果不是就走缓存方法。

#### SDWebImageCombinedOperation

从命名上也可以看出来，这是一个复合的 Operation。包含了一个缓存用的 `cacheOperation` 和一个下载用的 `loaderOperation`：

```objc
@interface SDWebImageCombinedOperation : NSObject <SDWebImageOperation>

/**
 Cancel the current operation, including cache and loader process
 */
- (void)cancel;

/**
 The cache operation from the image cache query
 */
@property (strong, nonatomic, nullable, readonly) id<SDWebImageOperation> cacheOperation;

/**
 The loader operation from the image loader (such as download operation)
 */
@property (strong, nonatomic, nullable, readonly) id<SDWebImageOperation> loaderOperation;

@end
```

它的 `cancel` 方法会取消内部的两种 Operation：

```objc
- (void)cancel {
    @synchronized(self) {
        if (self.isCancelled) {
            return;
        }
        self.cancelled = YES;
        if (self.cacheOperation) {
            [self.cacheOperation cancel];
            self.cacheOperation = nil;
        }
        if (self.loaderOperation) {
            [self.loaderOperation cancel];
            self.loaderOperation = nil;
        }
        [self.manager safelyRemoveOperationFromRunning:self];
    }
}
```

前面的代码中我们可以看到，最终方法返回的就是 `SDWebImageCombinedOperation` 的实例。也就是说，前面各个视图实例中通过关联对象方式保存的 `SDOpeartionsDictionary` 字典中保存的就是 `SDWebImageCombinedOperation` 实例对象。

不仅如此，`SDWebImageCombinedOperation` 实例，还被保存在了 `SDWebImageManager` 的 `runningOperations` 属性中。它会强引用 operation 实例。当 cancel 时，从 `runningOperations` 数组中移除的时候，由于 `SDOpeartionsDictionary` 是弱引用的方式保存的 operation，它就会被自动移除。

> 也就是说，UIView 实例和 Manager 都会将 Operation 保存起来。UIView 实例是弱引用，在 Manager 中移除的时候会自动清空其对应 Operation

#### SDWebImageOptionsResult

`SDWebImageOptionResult` 将 options 和 context 进行了整合统一。之后的使用过程中就不会出现 options 和 context 了，而是使用 `SDWebImageOptionResult` 实例：

```objc
@implementation SDWebImageOptionsResult

- (instancetype)initWithOptions:(SDWebImageOptions)options context:(SDWebImageContext *)context {
    self = [super init];
    if (self) {
        self.options = options;
        self.context = context;
    }
    return self;
}

@end
```

### 执行缓存方法

```objc
- (void)callCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                 url:(nonnull NSURL *)url
                             options:(SDWebImageOptions)options
                             context:(nullable SDWebImageContext *)context
                            progress:(nullable SDImageLoaderProgressBlock)progressBlock
                           completed:(nullable SDInternalCompletionBlock)completedBlock {
    /// 校验是否每次只能重新下载，不使用缓存
    BOOL shouldQueryCache = !SD_OPTIONS_CONTAINS(options, SDWebImageFromLoaderOnly);
    if (shouldQueryCache) {
        /// 取出自定义的缓存过滤器，用来通过 url 找到 url 对应的 key 字符串
        id<SDWebImageCacheKeyFilter> cacheKeyFilter = context[SDWebImageContextCacheKeyFilter];
        /// 通过缓存过滤器，根据 url 返回本地用来缓存的 key。如果没有传入缓存过滤器，那么默认为 url 的 absoluteString
        NSString *key = [self cacheKeyForURL:url cacheKeyFilter:cacheKeyFilter];
        @weakify(operation);
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
    } else {
        /// 不允许缓存直接下载
        [self callDownloadProcessForOperation:operation url:url options:options context:context cachedImage:nil cachedData:nil cacheType:SDImageCacheTypeNone progress:progressBlock completed:completedBlock];
    }
}
```

透过名字我们就能知道它是执行缓存的方法。能走缓存走缓存，不能走缓存直接走下载。关于缓存的具体实现请看后篇。

### 执行下载方法

```objc
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
    BOOL shouldDownload = !SD_OPTIONS_CONTAINS(options, SDWebImageFromCacheOnly);
    shouldDownload &= (!cachedImage || options & SDWebImageRefreshCached);
    shouldDownload &= (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url]);
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
                // Image combined operation cancelled by user
                [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorCancelled userInfo:nil] url:url];
            } else if (cachedImage && options & SDWebImageRefreshCached && [error.domain isEqualToString:SDWebImageErrorDomain] && error.code == SDWebImageErrorCacheNotModified) {
                /// 如果只是要刷新缓存，那么不调用回调方法。下载之后就会刷新 NSURLCache 了
            } else if ([error.domain isEqualToString:SDWebImageErrorDomain] && error.code == SDWebImageErrorCancelled) {
                // Download operation cancelled by user before sending the request, don't block failed URL
                [self callCompletionBlockForOperation:operation completion:completedBlock error:error url:url];
            } else if (error) {
                [self callCompletionBlockForOperation:operation completion:completedBlock error:error url:url];
                BOOL shouldBlockFailedURL = [self shouldBlockFailedURLWithURL:url error:error];
                
                if (shouldBlockFailedURL) {
                    SD_LOCK(self.failedURLsLock);
                    [self.failedURLs addObject:url];
                    SD_UNLOCK(self.failedURLsLock);
                }
            } else {
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
    /// 如果有缓存的图片
    } else if (cachedImage) {
        [self callCompletionBlockForOperation:operation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
        [self safelyRemoveOperationFromRunning:operation];
    /// 没有缓存的图片并且也不能下载，直接调用complete 回调
    } else {
        // Image not in cache and download disallowed by delegate
        [self callCompletionBlockForOperation:operation completion:completedBlock image:nil data:nil error:nil cacheType:SDImageCacheTypeNone finished:YES url:url];
        [self safelyRemoveOperationFromRunning:operation];
    }
}
```

从方法名中也能直观理解他是下载相关方法。在下载成功的回调中，会进行图片的存储。

### 执行存储方法

```objc
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

存储的方法主要围绕着是否要对图片进行 transform 展开的。在下载好图片之后，可以自定义的将图片进行一些转化，比如我们下载的头像数据，可能会直接切成圆角存储。避免以后每次使用图片都要设置带来的困扰。它是通过 `SDImageTransformer` 实现的。`SDImageTransformer` 本身是一个协议。SDWebImage 本身实现了一些，可以通过 context 传入。比如 `SDImagePipelineTransformer`，`SDImageRoundTransformer` 等。要注意，transform 之后的图片并不是以原命名保存的，会加上 transformer 提供的后缀，在获取缓存的时候也会在有自定义 transformer 的时候在 key 上加上 transformer 提供的后缀查询。transformer 的种类如下，实际实用过程中是很实用的：

|  **SDImageTransformer类型**   |              **作用**               |
| :---------------------------: | :---------------------------------: |
|  SDImagePipelineTransformer   | 可以传入一个NSArray<id>按顺序做转换 |
| SDImageRoundCornerTransformer |              添加圆角               |
|  SDImageResizingTransformer   |              调整大小               |
|  SDImageCroppingTransformer   |                剪裁                 |
|  SDImageFlippingTransformer   |                翻转                 |
|  SDImageRotationTransformer   |                旋转                 |
|    SDImageTintTransformer     |              添加色彩               |
|    SDImageBlurTransformer     |              添加模糊               |
|   SDImageFilterTransformer    |              添加滤镜               |



至此，SDWebImageManager 的 manager 部分就分析完了。后续会有缓存，下载和解码部分。





