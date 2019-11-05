title: SDWebImage 源码解析—解码策略
date: 2018/3/7 14:07:12  
categories: iOS
tags: 

- 学习笔记
- 源码解析

------

上一篇讲到了如何下载。这篇将探究下载后是如何解码的。

<!--more-->

## 外部调动

外部会在 disk 中获取到缓存 data 后调用解码方法，也会在 download 到 data 后调用解码方法。两者的解码操作基本一致，但是前者调用的 `SDImageCacheDefine.m` 中的方法，而后者调用的是 `SDImageLoader.m` 中的方法

### Cache 中的解码

在从 disk 中拿到 `NSData` 之后，就会调用 `SDImageCacheDefine.m` 中的 `SDImageCacheDecodeImageData()` 方法，进行解码：

```objc
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
        /// 从 data 转为 image
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
        /// 需要解码就开始解码，把 image 加载到内存中
        if (shouldDecode) {
            /// 是否需要缩放
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

如果是一个动图，并且不只解码第一帧，那么就会调用 `SDAnimatedImage` 的初始化方法把所有帧都解码出来。

如果不是动图，那么就会通过 `SDImageCodersManager` 先用 `NSData` 创建  `UIImage`，再对 `UIImage` 解码。整个过程会在下文详解。

### Download 中的解码

download 方式下的解码方法在 `SDImageLoader.m` 中。有两个方法 `SDImageLoaderDecodeImageData()` 和 `SDImageLoaderDecodeProgressiveImageData()`。

**前者和 Cache 中的方法是一样的**，都是将下载或者内存加载的 `NSData` 转为 `UIImage`，再解码。后者则提供了一种渐进式加载图片的方式：

```objc
UIImage * _Nullable SDImageLoaderDecodeProgressiveImageData(NSData * _Nonnull imageData, NSURL * _Nonnull imageURL, BOOL finished,  id<SDWebImageOperation> _Nonnull operation, SDWebImageOptions options, SDWebImageContext * _Nullable context) {
    NSCParameterAssert(imageData);
    NSCParameterAssert(imageURL);
    NSCParameterAssert(operation);
    
    UIImage *image;
    id<SDWebImageCacheKeyFilter> cacheKeyFilter = context[SDWebImageContextCacheKeyFilter];
    NSString *cacheKey;
    if (cacheKeyFilter) {
        cacheKey = [cacheKeyFilter cacheKeyForURL:imageURL];
    } else {
        cacheKey = imageURL.absoluteString;
    }
    BOOL decodeFirstFrame = SD_OPTIONS_CONTAINS(options, SDWebImageDecodeFirstFrameOnly);
    NSNumber *scaleValue = context[SDWebImageContextImageScaleFactor];
    CGFloat scale = scaleValue.doubleValue >= 1 ? scaleValue.doubleValue : SDImageScaleFactorForKey(cacheKey);
    SDImageCoderOptions *coderOptions = @{SDImageCoderDecodeFirstFrameOnly : @(decodeFirstFrame), SDImageCoderDecodeScaleFactor : @(scale)};
    if (context) {
        SDImageCoderMutableOptions *mutableCoderOptions = [coderOptions mutableCopy];
        [mutableCoderOptions setValue:context forKey:SDImageCoderWebImageContext];
        coderOptions = [mutableCoderOptions copy];
    }
    
    id<SDProgressiveImageCoder> progressiveCoder = objc_getAssociatedObject(operation, SDImageLoaderProgressiveCoderKey);
    if (!progressiveCoder) {
        /// 从 SDImageCodersManager 中找能解码该图片的并把它保存在 operation 中
        for (id<SDImageCoder>coder in [SDImageCodersManager sharedManager].coders.reverseObjectEnumerator) {
            if ([coder conformsToProtocol:@protocol(SDProgressiveImageCoder)] &&
                [((id<SDProgressiveImageCoder>)coder) canIncrementalDecodeFromData:imageData]) {
                progressiveCoder = [[[coder class] alloc] initIncrementalWithOptions:coderOptions];
                break;
            }
        }
        objc_setAssociatedObject(operation, SDImageLoaderProgressiveCoderKey, progressiveCoder, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    /// 没有能解码该图片的，直接返回nil
    if (!progressiveCoder) {
        return nil;
    }
    
    /// 渐进式加载 data
    [progressiveCoder updateIncrementalData:imageData finished:finished];
    /// 不是只decode第一帧
    if (!decodeFirstFrame) {
        // check whether we should use `SDAnimatedImage`
        Class animatedImageClass = context[SDWebImageContextAnimatedImageClass];
        if ([animatedImageClass isSubclassOfClass:[UIImage class]] && [animatedImageClass conformsToProtocol:@protocol(SDAnimatedImage)] && [progressiveCoder conformsToProtocol:@protocol(SDAnimatedImageCoder)]) {
            image = [[animatedImageClass alloc] initWithAnimatedCoder:(id<SDAnimatedImageCoder>)progressiveCoder scale:scale];
            if (image) {
                // Progressive decoding does not preload frames
            } else {
                // Check image class matching
                if (options & SDWebImageMatchAnimatedImageClass) {
                    return nil;
                }
            }
        }
    }
    /// 一般情况下生成 image
    if (!image) {
        image = [progressiveCoder incrementalDecodedImageWithOptions:coderOptions];
    }
    if (image) {
        BOOL shouldDecode = !SD_OPTIONS_CONTAINS(options, SDWebImageAvoidDecodeImage);
        if ([image.class conformsToProtocol:@protocol(SDAnimatedImage)]) {
            // `SDAnimatedImage` do not decode
            shouldDecode = NO;
        } else if (image.sd_isAnimated) {
            // animated image do not decode
            shouldDecode = NO;
        }
        if (shouldDecode) {
            image = [SDImageCoderHelper decodedImageWithImage:image];
        }
        // mark the image as progressive (completionBlock one are not mark as progressive)
        image.sd_isIncremental = YES;
    }
    
    return image;
}
```

和 cache 中做法相同的 `SDImageLoaderDecodeImageData()` 方法会在下载图片完成回调时进行；而渐进式解码 `SDImageLoaderDecodeProgressiveImageData()` 方法则会在下载过程中接收到一边接收数据一边进行。

## 编解码

### SDImageCodersManager：解码器的管理者

`SDImageCodersManager`  是一个单例类，内部在初始化的时候增加了多个实际的编解码类，**用于将 NSData 转化为 UIImage**。每次在使用到的时候都会循环每一个实际的编解码类，来对图片进行 data → image 的转换

> 你可能会有疑问，NDData 到 UIImage 不是系统内置的方法就可以解决嘛？一般情况下是的，对于 png， jpeg 等格式，系统自带了转换方法。但是对于苹果不是默认支持的格式，比如 webp，就需要自己通过解码器将 NSData → UIImage。
>
> 由此可见，**NSData → UIImage 的过程也是一个解码的过程**

#### 初始化方法

初始化方法如下所示。在单例的初始化过程中，默认增加了 `SDImageIOCode`，`SDImageGIFCoder `和 `SDImageAPNGCoder` 三个实际的编解码类。

```objc
+ (nonnull instancetype)sharedManager {
    static dispatch_once_t once;
    static id instance;
    dispatch_once(&once, ^{
        instance = [self new];
    });
    return instance;
}

- (instancetype)init {
    if (self = [super init]) {
        // initialize with default coders
        _imageCoders = [NSMutableArray arrayWithArray:@[[SDImageIOCoder sharedCoder], [SDImageGIFCoder sharedCoder], [SDImageAPNGCoder sharedCoder]]];
        _codersLock = dispatch_semaphore_create(1);
    }
    return self;
}
```

除了这三个内置的编解码类，我们还可以通过它提供的方法动态的添加和移除编解码器相关的类。

#### 能否支持编解码

解码要通过 `NSData` 进行解码。编码则是把 `UIImage` 根据 `SDImageFormat` 表示的图片类型转变为 `NSData`。和上面所说的一样，`SDImageCodersManager` 会调用内部的所有编解码类分别判断：

```objc
/// 遍历所有 coder 看是否能解码
- (BOOL)canDecodeFromData:(NSData *)data {
    NSArray<id<SDImageCoder>> *coders = self.coders;
    for (id<SDImageCoder> coder in coders.reverseObjectEnumerator) {
        if ([coder canDecodeFromData:data]) {
            return YES;
        }
    }
    return NO;
}

/// 遍历所有 coder 看是否能编码
- (BOOL)canEncodeToFormat:(SDImageFormat)format {
    NSArray<id<SDImageCoder>> *coders = self.coders;
    for (id<SDImageCoder> coder in coders.reverseObjectEnumerator) {
        if ([coder canEncodeToFormat:format]) {
            return YES;
        }
    }
    return NO;
}
```

#### 解码：从 NSData → UIImage

在判断能否进行编解码之后，就会调用实际的编解码的类的相关方法：

```objc
#pragma mark - 通过内部 coder 编解码
- (UIImage *)decodedImageWithData:(NSData *)data options:(nullable SDImageCoderOptions *)options {
    if (!data) {
        return nil;
    }
    UIImage *image;
    NSArray<id<SDImageCoder>> *coders = self.coders;
    for (id<SDImageCoder> coder in coders.reverseObjectEnumerator) {
        if ([coder canDecodeFromData:data]) {
            image = [coder decodedImageWithData:data options:options];
            break;
        }
    }
    
    return image;
}

- (NSData *)encodedDataWithImage:(UIImage *)image format:(SDImageFormat)format options:(nullable SDImageCoderOptions *)options {
    if (!image) {
        return nil;
    }
    NSArray<id<SDImageCoder>> *coders = self.coders;
    for (id<SDImageCoder> coder in coders.reverseObjectEnumerator) {
        if ([coder canEncodeToFormat:format]) {
            return [coder encodedDataWithImage:image format:format options:options];
        }
    }
    return nil;
}
```

 以上就是 `SDImageCodersManager` 的全部内容了。本生作为一个 Manager，并没有太多的任务，所有的操作都是遍历内部的编解码类的列表，让它们来完成的。

### SDImageCoderHelper：UIImage 到 bitmap 的转换者

上面 manager 中管理的是从 data 到 image 的解码，这个 helper 则是要把 UIImage 提前转化为 bitmap 以贡显示。

#### SDImageFrame 和 UIImage 的互相转化

`SDImageFrame` 是 GIF 或者 APNG 的每一帧的模型化。每一个 `SDImageFrame` 实例包括一个 `UIImage` 实例和一个时间间隔 `duration`:

```objc
@interface SDImageFrame : NSObject

@property (nonatomic, strong, readonly, nonnull) UIImage *image;
@property (nonatomic, readonly, assign) NSTimeInterval duration;

+ (instancetype _Nonnull)frameWithImage:(UIImage * _Nonnull)image duration:(NSTimeInterval)duration;

@end
```

GIF 和 APNG 不能直接转为 `UIImage` 的 `animatedImages` 的原因是后者的两帧之间的时间间隔是固定的，而前者则是不同的。因此两者相互转化的时候就要找到 GIF 和 APNG 两帧之间时间的最大公约数，以此作为 `UIImage` 的时间间隔。如果 GIF 和 APNG 的间隔较长则需要往其中插帧。

```objc
#pragma mark - 从 SDImageFrame 转为 UIImage
+ (UIImage *)animatedImageWithFrames:(NSArray<SDImageFrame *> *)frames {
    NSUInteger frameCount = frames.count;
    if (frameCount == 0) {
        return nil;
    }
    
    UIImage *animatedImage;
    
    /// 每一帧的持续时间都保存在 durations 中
    NSUInteger durations[frameCount];
    for (size_t i = 0; i < frameCount; i++) {
        durations[i] = frames[i].duration * 1000;
    }
    /// 获取各帧间隔时间的最大公约数作为 duration。间隔长的加帧
    NSUInteger const gcd = gcdArray(frameCount, durations);
    __block NSUInteger totalDuration = 0;
    NSMutableArray<UIImage *> *animatedImages = [NSMutableArray arrayWithCapacity:frameCount];
    [frames enumerateObjectsUsingBlock:^(SDImageFrame * _Nonnull frame, NSUInteger idx, BOOL * _Nonnull stop) {
        UIImage *image = frame.image;
        NSUInteger duration = frame.duration * 1000;
        totalDuration += duration;
        NSUInteger repeatCount;
        if (gcd) {
            /// 要重复 repeatCount 次
            repeatCount = duration / gcd;
        } else {
            repeatCount = 1;
        }
        for (size_t i = 0; i < repeatCount; ++i) {
            [animatedImages addObject:image];
        }
    }];
    
    animatedImage = [UIImage animatedImageWithImages:animatedImages duration:totalDuration / 1000.f];
    
    return animatedImage;
}

#pragma mark - UIImage 转 SDImageFrame
+ (NSArray<SDImageFrame *> *)framesFromAnimatedImage:(UIImage *)animatedImage {
    if (!animatedImage) {
        return nil;
    }
    
    NSMutableArray<SDImageFrame *> *frames = [NSMutableArray array];
    NSUInteger frameCount = 0;
    
    NSArray<UIImage *> *animatedImages = animatedImage.images;
    frameCount = animatedImages.count;
    if (frameCount == 0) {
        return nil;
    }
    
    NSTimeInterval avgDuration = animatedImage.duration / frameCount;
    if (avgDuration == 0) {
        avgDuration = 0.1; // if it's a animated image but no duration, set it to default 100ms (this do not have that 10ms limit like GIF or WebP to allow custom coder provide the limit)
    }
    
    __block NSUInteger index = 0;
    __block NSUInteger repeatCount = 1;
    __block UIImage *previousImage = animatedImages.firstObject;
    [animatedImages enumerateObjectsUsingBlock:^(UIImage * _Nonnull image, NSUInteger idx, BOOL * _Nonnull stop) {
        // ignore first
        if (idx == 0) {
            return;
        }
        if ([image isEqual:previousImage]) {
            /// 如果是重复的帧，只保存一次 SDImageFrame，加间隔
            repeatCount++;
        } else {
            SDImageFrame *frame = [SDImageFrame frameWithImage:previousImage duration:avgDuration * repeatCount];
            [frames addObject:frame];
            repeatCount = 1;
            index++;
        }
        previousImage = image;
        // last one
        if (idx == frameCount - 1) {
            SDImageFrame *frame = [SDImageFrame frameWithImage:previousImage duration:avgDuration * repeatCount];
            [frames addObject:frame];
        }
    }];

    return frames;
}
```

获取最大公约数的过程通过辗转相除的方式完成：

```objc
static NSUInteger gcd(NSUInteger a, NSUInteger b) {
    NSUInteger c;
    while (a != 0) {
        c = a;
        a = b % a;
        b = c;
    }
    return b;
}

/// 获取各帧时间间隔的最大公约数
static NSUInteger gcdArray(size_t const count, NSUInteger const * const values) {
    if (count == 0) {
        return 0;
    }
    NSUInteger result = values[0];
    for (size_t i = 1; i < count; ++i) {
        result = gcd(values[i], result);
    }
    return result;
}
```

#### 获取设备的 rgb空间

这个方法没什么特别的地方，就是 Core Graphic api的调用。它会在创建 Bitmap 的时候使用

```objc
#pragma mark - 获取设备的rgb颜色空间，创建bitmap时候需要
+ (CGColorSpaceRef)colorSpaceGetDeviceRGB {
    static CGColorSpaceRef colorSpace;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        if (@available(iOS 9.0, tvOS 9.0, *)) {
            colorSpace = CGColorSpaceCreateWithName(kCGColorSpaceSRGB);
        } else {
            colorSpace = CGColorSpaceCreateDeviceRGB();
        }
    });
    return colorSpace;
}
```

#### 是否具有alpha通道

判断是否含有 alpha 通道，在创建爱你 bitmap 的时候需要传入的属性：

```objc
#pragma mark - 是否含有 alpha 通道
+ (BOOL)CGImageContainsAlpha:(CGImageRef)cgImage {
    if (!cgImage) {
        return NO;
    }
    CGImageAlphaInfo alphaInfo = CGImageGetAlphaInfo(cgImage);
    BOOL hasAlpha = !(alphaInfo == kCGImageAlphaNone ||
                      alphaInfo == kCGImageAlphaNoneSkipFirst ||
                      alphaInfo == kCGImageAlphaNoneSkipLast);
    return hasAlpha;
}
```

在UI渲染的时候，实际上是把多个图层按像素叠加计算的过程，需要对每一个像素进行 RGBA 的叠加计算。当某个 layer 的是不透明的，也就是 opaque 为 YES 时，GPU 可以直接忽略掉其下方的图层，这就减少了很多工作量。这也是调用 CGBitmapContextCreate 时 bitmapInfo 参数设置为忽略掉 alpha 通道的原因。

#### 解码：从 image 到 bitmap

image 要显示在屏幕上需要转化为 bitmap，这个操作往往是显示的时候在主线程中做的。为了提高效率，可以通过 `decodedImageWithImage` 方法把 image 提前转化为 bitmap，这样这张新图片就不再需要重复渲染了，提高了渲染效率：

```objc
+ (UIImage *)decodedImageWithImage:(UIImage *)image {
    if (![self shouldDecodeImage:image]) {
        return image;
    }
    /// 新建一个 imageRef
    CGImageRef imageRef = [self CGImageCreateDecoded:image.CGImage];
    if (!imageRef) {
        return image;
    }
    /// 重新生成一个 image
    UIImage *decodedImage = [[UIImage alloc] initWithCGImage:imageRef scale:image.scale orientation:image.imageOrientation];
    CGImageRelease(imageRef);
    decodedImage.sd_isDecoded = YES;
    decodedImage.sd_imageFormat = image.sd_imageFormat;
    return decodedImage;
}

#pragma mark - 相当于复制一个 CGImageRef
+ (CGImageRef)CGImageCreateDecoded:(CGImageRef)cgImage {
    return [self CGImageCreateDecoded:cgImage orientation:kCGImagePropertyOrientationUp];
}

#pragma mark - 把 CGImageRef 转个向
+ (CGImageRef)CGImageCreateDecoded:(CGImageRef)cgImage orientation:(CGImagePropertyOrientation)orientation {
    if (!cgImage) {
        return NULL;
    }
    size_t width = CGImageGetWidth(cgImage);
    size_t height = CGImageGetHeight(cgImage);
    if (width == 0 || height == 0) return NULL;
    size_t newWidth;
    size_t newHeight;
    switch (orientation) {
        case kCGImagePropertyOrientationLeft:
        case kCGImagePropertyOrientationLeftMirrored:
        case kCGImagePropertyOrientationRight:
        case kCGImagePropertyOrientationRightMirrored: {
            // These orientation should swap width & height
            newWidth = height;
            newHeight = width;
        }
            break;
        default: {
            newWidth = width;
            newHeight = height;
        }
            break;
    }
    
    BOOL hasAlpha = [self CGImageContainsAlpha:cgImage];
    // iOS prefer BGRA8888 (premultiplied) or BGRX8888 bitmapInfo for screen rendering, which is same as `UIGraphicsBeginImageContext()` or `- [CALayer drawInContext:]`
    // Though you can use any supported bitmapInfo (see: https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/dq_context/dq_context.html#//apple_ref/doc/uid/TP30001066-CH203-BCIBHHBB ) and let Core Graphics reorder it when you call `CGContextDrawImage`
    // But since our build-in coders use this bitmapInfo, this can have a little performance benefit
    CGBitmapInfo bitmapInfo = kCGBitmapByteOrder32Host;
    /// 在UI渲染的时候，实际上是把多个图层按像素叠加计算的过程，需要对每一个像素进行 RGBA 的叠加计算。当某个 layer 的是不透明的，也就是 opaque 为 YES 时，GPU 可以直接忽略掉其下方的图层，这就减少了很多工作量。这也是调用 CGBitmapContextCreate 时 bitmapInfo 参数设置为忽略掉 alpha 通道的原因。
    bitmapInfo |= hasAlpha ? kCGImageAlphaPremultipliedFirst : kCGImageAlphaNoneSkipFirst;
    /// 创建 CGContextRef
    CGContextRef context = CGBitmapContextCreate(NULL, newWidth, newHeight, 8, 0, [self colorSpaceGetDeviceRGB], bitmapInfo);
    if (!context) {
        return NULL;
    }
    
    // Apply transform
    CGAffineTransform transform = SDCGContextTransformFromOrientation(orientation, CGSizeMake(newWidth, newHeight));
    CGContextConcatCTM(context, transform);
    CGContextDrawImage(context, CGRectMake(0, 0, width, height), cgImage); // The rect is bounding box of CGImage, don't swap width & height
    CGImageRef newImageRef = CGBitmapContextCreateImage(context);
    CGContextRelease(context);
    
    return newImageRef;
}
```

#### 将 UIImage 按比例减小到 bytes 内

每一个像素点由于有 RGBA 四个通道，每个通道占用 1 个字节，整个像素点需要占用 4 个字节。对于一个高分辨率的图片来说，会占用大量内存资源，甚至产生 OOM。

SDWebImage 的做法是，使用分块绘制的方式，读取一部分图片的data绘制成图，再把图绘制到缩放过的 context 中，然后释放掉之前读取的 data。循环往复，将整张图都绘制出来。这样能够解决内存爆炸问题。

```objc
#pragma mark - 将 UIImage 按比例缩小到 bytes 内
/// 加载大图可能会出现 OOM，使用分块绘制的方式，读取一部分图片的data，然后绘制到缩放过的 context 中，然后释放掉之前读取的 data。这样能够解决内存爆炸问题。
+ (UIImage *)decodedAndScaledDownImageWithImage:(UIImage *)image limitBytes:(NSUInteger)bytes {
    if (![self shouldDecodeImage:image]) {
        return image;
    }
    
    /// 如果不需要按比例缩小，那么直接解码
    if (![self shouldScaleDownImage:image limitBytes:bytes]) {
        return [self decodedImageWithImage:image];
    }
    
    CGFloat destTotalPixels;
    CGFloat tileTotalPixels;
    if (bytes > 0) {
        destTotalPixels = bytes / kBytesPerPixel;
        /// 这里 3 可以是任意的值，只是想获取一个较小的分块像素总量
        tileTotalPixels = destTotalPixels / 3;
    } else {
        destTotalPixels = kDestTotalPixels;
        tileTotalPixels = kTileTotalPixels;
    }
    CGContextRef destContext;
    
    @autoreleasepool {
        CGImageRef sourceImageRef = image.CGImage;
        
        CGSize sourceResolution = CGSizeZero;
        sourceResolution.width = CGImageGetWidth(sourceImageRef);
        sourceResolution.height = CGImageGetHeight(sourceImageRef);
        CGFloat sourceTotalPixels = sourceResolution.width * sourceResolution.height;
        // Determine the scale ratio to apply to the input image
        // that results in an output image of the defined size.
        // see kDestImageSizeMB, and how it relates to destTotalPixels.
        /// 面积之比开平方，就是边长之比
        CGFloat imageScale = sqrt(destTotalPixels / sourceTotalPixels);
        CGSize destResolution = CGSizeZero;
        /// 目标图像的宽高
        destResolution.width = (int)(sourceResolution.width * imageScale);
        destResolution.height = (int)(sourceResolution.height * imageScale);
        
        // device color space
        CGColorSpaceRef colorspaceRef = [self colorSpaceGetDeviceRGB];
        BOOL hasAlpha = [self CGImageContainsAlpha:sourceImageRef];
        // iOS display alpha info (BGRA8888/BGRX8888)
        CGBitmapInfo bitmapInfo = kCGBitmapByteOrder32Host;
        bitmapInfo |= hasAlpha ? kCGImageAlphaPremultipliedFirst : kCGImageAlphaNoneSkipFirst;
        
        /// 创建一个目标大小的 context
        destContext = CGBitmapContextCreate(NULL,
                                            destResolution.width,
                                            destResolution.height,
                                            kBitsPerComponent,
                                            0,
                                            colorspaceRef,
                                            bitmapInfo);
        
        if (destContext == NULL) {
            return image;
        }
        /// 设置插值质量为高质量
        CGContextSetInterpolationQuality(destContext, kCGInterpolationHigh);
        
        // Now define the size of the rectangle to be used for the
        // incremental blits from the input image to the output image.
        // we use a source tile width equal to the width of the source
        // image due to the way that iOS retrieves image data from disk.
        // iOS must decode an image from disk in full width 'bands', even
        // if current graphics context is clipped to a subrect within that
        // band. Therefore we fully utilize all of the pixel data that results
        // from a decoding opertion by achnoring our tile size to the full
        // width of the input image.
        CGRect sourceTile = CGRectZero;
        /// sourcetile 的宽度是原图片的宽度
        sourceTile.size.width = sourceResolution.width;
        // The source tile height is dynamic. Since we specified the size
        // of the source tile in MB, see how many rows of pixels high it
        // can be given the input image width.
        /// sourceTile 的宽高乘积是 tileTotalPixels
        sourceTile.size.height = (int)(tileTotalPixels / sourceTile.size.width );
        sourceTile.origin.x = 0.0f;
        // The output tile is the same proportions as the input tile, but
        // scaled to image scale.
        CGRect destTile;
        destTile.size.width = destResolution.width;
        /// sourceTile 的高度 * 缩放比例就是 destTile 的高度
        destTile.size.height = sourceTile.size.height * imageScale;
        destTile.origin.x = 0.0f;
        // The source seem overlap is proportionate to the destination seem overlap.
        // this is the amount of pixels to overlap each tile as we assemble the ouput image.
        /// 目标tile 有 2像素的 overlap。换算为 sourceTile 的overlap
        float sourceSeemOverlap = (int)((kDestSeemOverlap/destResolution.height)*sourceResolution.height);
        CGImageRef sourceTileImageRef;
        // calculate the number of read/write operations required to assemble the
        // output image.
        /// 要迭代的z次数等于源图片高度/原tile的高度
        int iterations = (int)( sourceResolution.height / sourceTile.size.height );
        // If tile height doesn't divide the image height evenly, add another iteration
        // to account for the remaining pixels.
        /// 如果不能整除，那么迭代次数要+1
        int remainder = (int)sourceResolution.height % (int)sourceTile.size.height;
        if(remainder) {
            iterations++;
        }
        // Add seem overlaps to the tiles, but save the original tile height for y coordinate calculations.
        float sourceTileHeightMinusOverlap = sourceTile.size.height;
        sourceTile.size.height += sourceSeemOverlap;
        destTile.size.height += kDestSeemOverlap;
        /// 分块绘制
        for( int y = 0; y < iterations; ++y ) {
            @autoreleasepool {
                sourceTile.origin.y = y * sourceTileHeightMinusOverlap + sourceSeemOverlap;
                destTile.origin.y = destResolution.height - (( y + 1 ) * sourceTileHeightMinusOverlap * imageScale + kDestSeemOverlap);
                sourceTileImageRef = CGImageCreateWithImageInRect( sourceImageRef, sourceTile );
                if( y == iterations - 1 && remainder ) {
                    float dify = destTile.size.height;
                    destTile.size.height = CGImageGetHeight( sourceTileImageRef ) * imageScale;
                    dify -= destTile.size.height;
                    destTile.origin.y += dify;
                }
                CGContextDrawImage( destContext, destTile, sourceTileImageRef );
                CGImageRelease( sourceTileImageRef );
            }
        }
        
        /// 通过 destContext 创建 CGImageRef
        CGImageRef destImageRef = CGBitmapContextCreateImage(destContext);
        CGContextRelease(destContext);
        if (destImageRef == NULL) {
            return image;
        }
        /// 通过destImageRef创建image
        UIImage *destImage = [[UIImage alloc] initWithCGImage:destImageRef scale:image.scale orientation:image.imageOrientation];
        CGImageRelease(destImageRef);
        if (destImage == nil) {
            return image;
        }
        destImage.sd_isDecoded = YES;
        destImage.sd_imageFormat = image.sd_imageFormat;
        return destImage;
    }
}

#pragma mark - 是否需要按比例缩小
+ (BOOL)shouldScaleDownImage:(nonnull UIImage *)image limitBytes:(NSUInteger)bytes {
    BOOL shouldScaleDown = YES;
    
    CGImageRef sourceImageRef = image.CGImage;
    CGSize sourceResolution = CGSizeZero;
    sourceResolution.width = CGImageGetWidth(sourceImageRef);
    sourceResolution.height = CGImageGetHeight(sourceImageRef);
    float sourceTotalPixels = sourceResolution.width * sourceResolution.height;
    if (sourceTotalPixels <= 0) {
        return NO;
    }
    CGFloat destTotalPixels;
    if (bytes > 0) {
        /// bytes / 每一个像素几个 bytes = 像素数
        destTotalPixels = bytes / kBytesPerPixel;
    } else {
        destTotalPixels = kDestTotalPixels;
    }
    /// 小于1M不需要按比例缩小
    if (destTotalPixels <= kPixelsPerMB) {
        // Too small to scale down
        return NO;
    }
    /// 比例 = 目标像素/现在的image的像素
    float imageScale = destTotalPixels / sourceTotalPixels;
    if (imageScale < 1) {
        shouldScaleDown = YES;
    } else {
        shouldScaleDown = NO;
    }
    
    return shouldScaleDown;
}
```

### SDImageIOCoder：内置图片类型的解码器

`SDImageIOCoder` 支持 png，jpeg ，以及部分机型支持 HEIC HEIF 的解码以及渐进式加载。

#### 能否解码

```objc
- (BOOL)canDecodeFromData:(nullable NSData *)data {
    switch ([NSData sd_imageFormatForImageData:data]) {
        case SDImageFormatWebP:
            // Do not support WebP decoding
            return NO;
        case SDImageFormatHEIC:
            // Check HEIC decoding compatibility
            return [[self class] canDecodeFromHEICFormat];
        case SDImageFormatHEIF:
            // Check HEIF decoding compatibility
            return [[self class] canDecodeFromHEIFFormat];
        default:
            return YES;
    }
}
```

能否解码主要依据与图片的类型是否是支持的类型。获取图片数据格式的方式如下：

```objc
+ (SDImageFormat)sd_imageFormatForImageData:(nullable NSData *)data {
    if (!data) {
        return SDImageFormatUndefined;
    }
    
    // File signatures table: http://www.garykessler.net/library/file_sigs.html
    uint8_t c;
    [data getBytes:&c length:1];
    switch (c) {
        case 0xFF:
            return SDImageFormatJPEG;
        case 0x89:
            return SDImageFormatPNG;
        case 0x47:
            return SDImageFormatGIF;
        case 0x49:
        case 0x4D:
            return SDImageFormatTIFF;
        case 0x52: {
            if (data.length >= 12) {
                //RIFF....WEBP
                NSString *testString = [[NSString alloc] initWithData:[data subdataWithRange:NSMakeRange(0, 12)] encoding:NSASCIIStringEncoding];
                if ([testString hasPrefix:@"RIFF"] && [testString hasSuffix:@"WEBP"]) {
                    return SDImageFormatWebP;
                }
            }
            break;
        }
        case 0x00: {
            if (data.length >= 12) {
                //....ftypheic ....ftypheix ....ftyphevc ....ftyphevx
                NSString *testString = [[NSString alloc] initWithData:[data subdataWithRange:NSMakeRange(4, 8)] encoding:NSASCIIStringEncoding];
                if ([testString isEqualToString:@"ftypheic"]
                    || [testString isEqualToString:@"ftypheix"]
                    || [testString isEqualToString:@"ftyphevc"]
                    || [testString isEqualToString:@"ftyphevx"]) {
                    return SDImageFormatHEIC;
                }
                //....ftypmif1 ....ftypmsf1
                if ([testString isEqualToString:@"ftypmif1"] || [testString isEqualToString:@"ftypmsf1"]) {
                    return SDImageFormatHEIF;
                }
            }
            break;
        }
    }
    return SDImageFormatUndefined;
}
```

通过读取 data 的第一个字节就能知道图片的类型。

判断判断是否支持某一个格式的编码，通过看是否能生成 CGImageDestination 来判断：

```objc
+ (BOOL)canEncodeToFormat:(SDImageFormat)format { // 判断是否支持某一个格式的编码，通过看是否能生成 CGImageDestination 来判断
    NSMutableData *imageData = [NSMutableData data];
    CFStringRef imageUTType = [NSData sd_UTTypeFromImageFormat:format];
    
    // Create an image destination.
    CGImageDestinationRef imageDestination = CGImageDestinationCreateWithData((__bridge CFMutableDataRef)imageData, imageUTType, 1, NULL);
    if (!imageDestination) {
        // Can't encode to HEIC
        return NO;
    } else {
        // Can encode to HEIC
        CFRelease(imageDestination);
        return YES;
    }
}
```

#### 直接解码：完整的 NSData → UIImage

直接解码的方式非常简单粗暴，直接通过 `UIImage` 的方法：

```objc
- (UIImage *)decodedImageWithData:(NSData *)data options:(nullable SDImageCoderOptions *)options {
    if (!data) {
        return nil;
    }
    CGFloat scale = 1;
    NSNumber *scaleFactor = options[SDImageCoderDecodeScaleFactor];
    if (scaleFactor != nil) {
        scale = MAX([scaleFactor doubleValue], 1) ;
    }
    
    UIImage *image = [[UIImage alloc] initWithData:data scale:scale];
    image.sd_imageFormat = [NSData sd_imageFormatForImageData:data];
    return image;
}
```

#### 渐进式解码：部分 NSData → UIImage

渐进式解码提供了边下载边解码的功能。

```objc
#pragma mark - 渐进式更新
- (void)updateIncrementalData:(NSData *)data finished:(BOOL)finished {
    if (_finished) {
        return;
    }
    _finished = finished;
    
    /// 要把刚获得的 data 加入到 imageSource 中
    CGImageSourceUpdateData(_imageSource, (__bridge CFDataRef)data, finished);
    
    if (_width + _height == 0) {
        CFDictionaryRef properties = CGImageSourceCopyPropertiesAtIndex(_imageSource, 0, NULL);
        if (properties) {
            NSInteger orientationValue = 1;
            CFTypeRef val = CFDictionaryGetValue(properties, kCGImagePropertyPixelHeight);
            if (val) CFNumberGetValue(val, kCFNumberLongType, &_height);
            val = CFDictionaryGetValue(properties, kCGImagePropertyPixelWidth);
            if (val) CFNumberGetValue(val, kCFNumberLongType, &_width);
            val = CFDictionaryGetValue(properties, kCGImagePropertyOrientation);
            if (val) CFNumberGetValue(val, kCFNumberNSIntegerType, &orientationValue);
            CFRelease(properties);
            
            // When we draw to Core Graphics, we lose orientation information,
            // which means the image below born of initWithCGIImage will be
            // oriented incorrectly sometimes. (Unlike the image born of initWithData
            // in didCompleteWithError.) So save it here and pass it on later.
            _orientation = (CGImagePropertyOrientation)orientationValue;
        }
    }
}

#pragma mark - 渐进式更新后生成image
- (UIImage *)incrementalDecodedImageWithOptions:(SDImageCoderOptions *)options {
    UIImage *image;
    
    if (_width + _height > 0) {
        // Create the image
        CGImageRef partialImageRef = CGImageSourceCreateImageAtIndex(_imageSource, 0, NULL);
        
        if (partialImageRef) {
            CGFloat scale = _scale;
            NSNumber *scaleFactor = options[SDImageCoderDecodeScaleFactor];
            if (scaleFactor != nil) {
                scale = MAX([scaleFactor doubleValue], 1);
            }
            UIImageOrientation imageOrientation = [SDImageCoderHelper imageOrientationFromEXIFOrientation:_orientation];
            image = [[UIImage alloc] initWithCGImage:partialImageRef scale:scale orientation:imageOrientation];
            CGImageRelease(partialImageRef);
            CFStringRef uttype = CGImageSourceGetType(_imageSource);
            image.sd_imageFormat = [NSData sd_imageFormatFromUTType:uttype];
        }
    }
    
    return image;
}
```

主要还是使用 Core Graphic 的 API。前一个方法通过 `CGImageSourceUpdateData()` 往 `CGImageSourceRef` 中增加 data，后一个方法把 `CGImageSourceRef` 生成为 `UIImage`

#### 编码

编码是从 `UIImage` 到 `NSData` 的转换：

```objc
- (NSData *)encodedDataWithImage:(UIImage *)image format:(SDImageFormat)format options:(nullable SDImageCoderOptions *)options {
    if (!image) {
        return nil;
    }
    
    if (format == SDImageFormatUndefined) {
        BOOL hasAlpha = [SDImageCoderHelper CGImageContainsAlpha:image.CGImage];
        /// 含有 alpha 通道就是 PNG 没有就是 JPEG
        if (hasAlpha) {
            format = SDImageFormatPNG;
        } else {
            format = SDImageFormatJPEG;
        }
    }
    
    NSMutableData *imageData = [NSMutableData data];
    CFStringRef imageUTType = [NSData sd_UTTypeFromImageFormat:format];
    
    // Create an image destination.
    CGImageDestinationRef imageDestination = CGImageDestinationCreateWithData((__bridge CFMutableDataRef)imageData, imageUTType, 1, NULL);
    if (!imageDestination) {
        // Handle failure.
        return nil;
    }
    
    NSMutableDictionary *properties = [NSMutableDictionary dictionary];
    CGImagePropertyOrientation exifOrientation = [SDImageCoderHelper exifOrientationFromImageOrientation:image.imageOrientation];
    properties[(__bridge NSString *)kCGImagePropertyOrientation] = @(exifOrientation);
    double compressionQuality = 1;
    if (options[SDImageCoderEncodeCompressionQuality]) {
        compressionQuality = [options[SDImageCoderEncodeCompressionQuality] doubleValue];
    }
    properties[(__bridge NSString *)kCGImageDestinationLossyCompressionQuality] = @(compressionQuality);
    
    // Add your image to the destination.
    CGImageDestinationAddImage(imageDestination, image.CGImage, (__bridge CFDictionaryRef)properties);
    
    // Finalize the destination.
    if (CGImageDestinationFinalize(imageDestination) == NO) {
        // Handle failure.
        imageData = nil;
    }
    
    CFRelease(imageDestination);
    
    return [imageData copy];
}
```

`UIImageJPEGRepresentation()` 类似的 API 也可以将 `UIImage` 转为 `NSData`，但是上面这种通过 imageIO api 的方式生成的 NSData 无论是从内存使用还是物理内存的占用上都有更好的表现。

## 总结

从本篇的解码过程中我们可以知道，不论是从 cache 还是 download，都需要通过 data 转化为通过 bitmap 展示的 image。其中解码过程会分为两步：

1. 从 data → image
2. 从 image → bitmap 形式的 image

第一步由 `SDImageCodersManager` 管理，它通过内置的多个编解码器完成；第二部由 `SDImageCoderHelper` 完成，它会创建一个 bitmap，并把前面的  image 画到上面去生成一个新的 image。

在编解码阶段我们能获取到的知识点有如下：

- 获取图片的类型可以读取其 data 的第一个字节判断
- GIF 的 duration 是不固定的，转为 iOS 中显示时要插帧
- 如果图片没有 alpha 通道，创建 bitmap 的时候就不要创建有 alpha 通道的。没有 alpha 通道能增快渲染效率
- UIImage 只有在显示的时候才会在主线程中解码，浪费时间。可以通过在子线程中创建 bitmap 的方式提前将图片加载到内存中。
- 对于超大的 UIImage，为了降低内存消耗，可以按比例缩放。生成缩放图片的时候可以通过分片渲染的方式，每加载一部分，渲染一部分到缩放图片上。
- 渐进式解码主要使用 Core Graphic API，获取一部分数据就往 `CGImageSourceRef` 中丢一部分，然后生成 UIImage
- 使用 `UIImageJPEGRepresentation()` 类似 API 将  UIImage → NSData 不如使用 ImageIO 的 API 效率高，占用内存小。















