title: AFNetworking 源码解析 - 响应序列化
date: 2018/12/27 14:07:12  
categories: iOS
tags: 

- 学习笔记
- 源码解析

------

本篇是 AFNetworking 源码解读的第三篇了。主要讲解接收到响应后的处理。

<!--more-->

## AFURLResponseSerialization 协议

`AFURLResponseSerialization` 协议只有一个方法需要实现：

```objc
- (nullable id)responseObjectForResponse:(nullable NSURLResponse *)response
                           data:(nullable NSData *)data
                          error:(NSError * _Nullable __autoreleasing *)error NS_SWIFT_NOTHROW;
```

这个协议会拿到响应对象 `NSURLResponse` ，响应数据 `NSData`。可以根据 `NSURLResponse` 返回数据类型标识 `content-type` 选取相应的解析器，来解析 data.

## AFHTTPResponseSerializer 基类

 `AFHTTPResponseSerializer` 是一个基类，提供最基本的属性和方法，针对不同 `content-type` 的响应报文，需要实例化不同的子类进行解析。

`AFHTTPResponseSerializer` 有两个重要的属性：

```objc
/// 设置接收的状态码，不在此内的状态码返回错误
@property (nonatomic, copy, nullable) NSIndexSet *acceptableStatusCodes;
/// 设置接受的content-type
@property (nonatomic, copy, nullable) NSSet <NSString *> *acceptableContentTypes;
```

它们会在初始化方法中赋值：

```objc
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    /// 设置i接受的返回code 接收范围是 200-299
    self.acceptableStatusCodes = [NSIndexSet indexSetWithIndexesInRange:NSMakeRange(200, 100)];
    /// 设置接受的返回content-type 为任何 type
    self.acceptableContentTypes = nil;

    return self;
}
```

因为 `AFHTTPResponseSerializer` 不做任何类型校验，因此其 `acceptableContentTypes` 为 nil，表示所有类型皆可，这个属性会在不同子类中设置为不同类型。

`acceptableStatusCodes` 是一个 `NSIndexSet` 实例，是一个有序的，唯一的，无符号整数的集合。比如：

```objc
NSMutableIndexSet *indexSetM = [NSMutableIndexSet indexSet];
[indexSetM addIndex:6];
[indexSetM addIndex:8];
[indexSetM addIndexesInRange:NSMakeRange(20, 5)];

=> [6,8,20,21,22,23,24]
```

`addIndexesInRange:` 方法可以设定一个范围。因此上面的代码中 `acceptableStatusCodes` 代表的是只接受 200-299 的状态码。

根据上面的有效状态码和有效 content-type，基类中还有一个验证方法。如果验证无效，会抛出 `NSError`：

```objc
/// 验证服务器返回的数据，是否符合接收的状态码和 content-type
- (BOOL)validateResponse:(NSHTTPURLResponse *)response
                    data:(NSData *)data
                   error:(NSError * __autoreleasing *)error
{
    /// 默认响应都是有效的
    BOOL responseIsValid = YES;
    NSError *validationError = nil;

    if (response && [response isKindOfClass:[NSHTTPURLResponse class]]) {
        /// 如果设置了接受的 content-type 并且响应的 content-type 不在其之内，并且 response 的 MIMEType 和 data 都不为空
        if (self.acceptableContentTypes && ![self.acceptableContentTypes containsObject:[response MIMEType]] &&
            !([response MIMEType] == nil && [data length] == 0)) {

            /// 生成错误信息
            if ([data length] > 0 && [response URL]) {
                NSMutableDictionary *mutableUserInfo = [@{
                                                          NSLocalizedDescriptionKey: [NSString stringWithFormat:NSLocalizedStringFromTable(@"Request failed: unacceptable content-type: %@", @"AFNetworking", nil), [response MIMEType]],
                                                          NSURLErrorFailingURLErrorKey:[response URL],
                                                          AFNetworkingOperationFailingURLResponseErrorKey: response,
                                                        } mutableCopy];
                if (data) {
                    mutableUserInfo[AFNetworkingOperationFailingURLResponseDataErrorKey] = data;
                }

                validationError = AFErrorWithUnderlyingError([NSError errorWithDomain:AFURLResponseSerializationErrorDomain code:NSURLErrorCannotDecodeContentData userInfo:mutableUserInfo], validationError);
            }

            responseIsValid = NO;
        }

        /// 校验响应状态码
        if (self.acceptableStatusCodes && ![self.acceptableStatusCodes containsIndex:(NSUInteger)response.statusCode] && [response URL]) {
            NSMutableDictionary *mutableUserInfo = [@{
                                               NSLocalizedDescriptionKey: [NSString stringWithFormat:NSLocalizedStringFromTable(@"Request failed: %@ (%ld)", @"AFNetworking", nil), [NSHTTPURLResponse localizedStringForStatusCode:response.statusCode], (long)response.statusCode],
                                               NSURLErrorFailingURLErrorKey:[response URL],
                                               AFNetworkingOperationFailingURLResponseErrorKey: response,
                                       } mutableCopy];

            if (data) {
                mutableUserInfo[AFNetworkingOperationFailingURLResponseDataErrorKey] = data;
            }

            validationError = AFErrorWithUnderlyingError([NSError errorWithDomain:AFURLResponseSerializationErrorDomain code:NSURLErrorBadServerResponse userInfo:mutableUserInfo], validationError);

            responseIsValid = NO;
        }
    }

    if (error && !responseIsValid) {
        *error = validationError;
    }

    return responseIsValid;
}
```

## AFJSONResponseSerializer

解析 JSON 的序列化类。它的 `content-type` 为 `@"application/json", @"text/json", @"text/javascript"`

```objc
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    /// 接受的 content-type 为 @"application/json", @"text/json", @"text/javascript"
    self.acceptableContentTypes = [NSSet setWithObjects:@"application/json", @"text/json", @"text/javascript", nil];

    return self;
}
```

序列化方法通过 `NSJSONSerialization` 将 `NSData` 转为 json 对象，一般是一个字典或者数组：

```objc
- (id)responseObjectForResponse:(NSURLResponse *)response
                           data:(NSData *)data
                          error:(NSError *__autoreleasing *)error
{
    /// 验证 response
    if (![self validateResponse:(NSHTTPURLResponse *)response data:data error:error]) {
        if (!error || AFErrorOrUnderlyingErrorHasCodeInDomain(*error, NSURLErrorCannotDecodeContentData, AFURLResponseSerializationErrorDomain)) {
            return nil;
        }
    }
    BOOL isSpace = [data isEqualToData:[NSData dataWithBytes:" " length:1]];
    
    if (data.length == 0 || isSpace) {
        return nil;
    }
    
    NSError *serializationError = nil;
    
    id responseObject = [NSJSONSerialization JSONObjectWithData:data options:self.readingOptions error:&serializationError];

    if (!responseObject)
    {
        if (error) {
            *error = AFErrorWithUnderlyingError(serializationError, *error);
        }
        return nil;
    }
    
    /// 移除空的 key
    if (self.removesKeysWithNullValues) {
        return AFJSONObjectByRemovingKeysWithNullValues(responseObject, self.readingOptions);
    }

    return responseObject;
}
```

最后还会删除数据中值为 `NSNull` 的 key：

```objc
/// 移除空的 key
static id AFJSONObjectByRemovingKeysWithNullValues(id JSONObject, NSJSONReadingOptions readingOptions) {
    /// 如果是数组
    if ([JSONObject isKindOfClass:[NSArray class]]) {
        NSMutableArray *mutableArray = [NSMutableArray arrayWithCapacity:[(NSArray *)JSONObject count]];
        for (id value in (NSArray *)JSONObject) {
            /// 递归调用
            [mutableArray addObject:AFJSONObjectByRemovingKeysWithNullValues(value, readingOptions)];
        }

        return (readingOptions & NSJSONReadingMutableContainers) ? mutableArray : [NSArray arrayWithArray:mutableArray];
    /// 如果字典
    } else if ([JSONObject isKindOfClass:[NSDictionary class]]) {
        NSMutableDictionary *mutableDictionary = [NSMutableDictionary dictionaryWithDictionary:JSONObject];
        /// 循环字典中所有的 key
        for (id <NSCopying> key in [(NSDictionary *)JSONObject allKeys]) {
            id value = (NSDictionary *)JSONObject[key];
            /// 如果 value 是 NSNull
            if (!value || [value isEqual:[NSNull null]]) {
                /// removeObjectForKey
                [mutableDictionary removeObjectForKey:key];
            /// 如果 value 还是数组或者字典类型
            } else if ([value isKindOfClass:[NSArray class]] || [value isKindOfClass:[NSDictionary class]]) {
                /// 递归调用
                mutableDictionary[key] = AFJSONObjectByRemovingKeysWithNullValues(value, readingOptions);
            }
        }

        return (readingOptions & NSJSONReadingMutableContainers) ? mutableDictionary : [NSDictionary dictionaryWithDictionary:mutableDictionary];
    }

    return JSONObject;
}
```

> 注意，数组中的空对象是不会被删除的，只会删除字典中的值为 nil 的键

## AFXMLParserResponseSerializer

解析 xml 的序列化类。它的 `content-type` 为 `@"application/xml", @"text/xml"`:

```objc
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.acceptableContentTypes = [[NSSet alloc] initWithObjects:@"application/xml", @"text/xml", nil];

    return self;
}
```

通过系统类 `NSXMLParser` 解析：

```objc
- (id)responseObjectForResponse:(NSHTTPURLResponse *)response
                           data:(NSData *)data
                          error:(NSError *__autoreleasing *)error
{
    if (![self validateResponse:(NSHTTPURLResponse *)response data:data error:error]) {
        if (!error || AFErrorOrUnderlyingErrorHasCodeInDomain(*error, NSURLErrorCannotDecodeContentData, AFURLResponseSerializationErrorDomain)) {
            return nil;
        }
    }

    return [[NSXMLParser alloc] initWithData:data];
}
```

## AFPropertyListResponseSerializer

解析 plist 的序列化类。它的 content-type 为 `@"application/x-plist"`

```objc
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.acceptableContentTypes = [[NSSet alloc] initWithObjects:@"application/x-plist", nil];

    return self;
}
```

通过 `NSPropertyListSerialization` 解析：

```objc
- (id)responseObjectForResponse:(NSURLResponse *)response
                           data:(NSData *)data
                          error:(NSError *__autoreleasing *)error
{
    if (![self validateResponse:(NSHTTPURLResponse *)response data:data error:error]) {
        if (!error || AFErrorOrUnderlyingErrorHasCodeInDomain(*error, NSURLErrorCannotDecodeContentData, AFURLResponseSerializationErrorDomain)) {
            return nil;
        }
    }

    if (!data) {
        return nil;
    }
    
    NSError *serializationError = nil;
    
    id responseObject = [NSPropertyListSerialization propertyListWithData:data options:self.readOptions format:NULL error:&serializationError];
    
    if (!responseObject)
    {
        if (error) {
            *error = AFErrorWithUnderlyingError(serializationError, *error);
        }
        return nil;
    }

    return responseObject;
}
```

## AFImageResponseSerializer

解析 image 的序列化类。它的 content-type 为 `@"image/tiff", @"image/jpeg", @"image/gif", @"image/png", @"image/ico", @"image/x-icon", @"image/bmp", @"image/x-bmp", @"image/x-xbitmap", @"image/x-win-bitmap"`。

它还有一个属性 `automaticallyInflatesResponseImage` 用来设置是否自动解码：

```objc
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.acceptableContentTypes = [[NSSet alloc] initWithObjects:@"image/tiff", @"image/jpeg", @"image/gif", @"image/png", @"image/ico", @"image/x-icon", @"image/bmp", @"image/x-bmp", @"image/x-xbitmap", @"image/x-win-bitmap", nil];

    self.imageScale = [[UIScreen mainScreen] scale];
    self.automaticallyInflatesResponseImage = YES;

    return self;
}
```

它的解析方式如下：

```objc
- (id)responseObjectForResponse:(NSURLResponse *)response
                           data:(NSData *)data
                          error:(NSError *__autoreleasing *)error
{
    if (![self validateResponse:(NSHTTPURLResponse *)response data:data error:error]) {
        if (!error || AFErrorOrUnderlyingErrorHasCodeInDomain(*error, NSURLErrorCannotDecodeContentData, AFURLResponseSerializationErrorDomain)) {
            return nil;
        }
    }

    if (self.automaticallyInflatesResponseImage) {
        return AFInflatedImageFromResponseWithDataAtScale((NSHTTPURLResponse *)response, data, self.imageScale);
    } else {
        return AFImageWithDataAtScale(data, self.imageScale);
    }
    return nil;
}
```

其中涉及到两个方法 `AFInflatedImageFromResponseWithDataAtScale` 和 `AFImageWithDataAtScale`。先来看后者，它通过 NSData 创建 image：

```objc
/// 创建 image
+ (UIImage *)af_safeImageWithData:(NSData *)data {
    UIImage* image = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        imageLock = [[NSLock alloc] init];
    });
    
    [imageLock lock];
    image = [UIImage imageWithData:data];
    [imageLock unlock];
    return image;
}

@end

/// data scale 创建 image
static UIImage * AFImageWithDataAtScale(NSData *data, CGFloat scale) {
    UIImage *image = [UIImage af_safeImageWithData:data];
    if (image.images) {
        return image;
    }
    
    return [[UIImage alloc] initWithCGImage:[image CGImage] scale:scale orientation:image.imageOrientation];
}
```

上面创建图片的方式比较普通，不太能理解为什么要加锁。然后是 `AFInflatedImageFromResponseWithDataAtScale` ：

```objc
/// 将png和jpeg图片解码
static UIImage * AFInflatedImageFromResponseWithDataAtScale(NSHTTPURLResponse *response, NSData *data, CGFloat scale) {
    if (!data || [data length] == 0) {
        return nil;
    }

    CGImageRef imageRef = NULL;
    CGDataProviderRef dataProvider = CGDataProviderCreateWithCFData((__bridge CFDataRef)data);

    if ([response.MIMEType isEqualToString:@"image/png"]) {
        imageRef = CGImageCreateWithPNGDataProvider(dataProvider,  NULL, true, kCGRenderingIntentDefault);
    } else if ([response.MIMEType isEqualToString:@"image/jpeg"]) {
        imageRef = CGImageCreateWithJPEGDataProvider(dataProvider, NULL, true, kCGRenderingIntentDefault);

        if (imageRef) {
            CGColorSpaceRef imageColorSpace = CGImageGetColorSpace(imageRef);
            CGColorSpaceModel imageColorSpaceModel = CGColorSpaceGetModel(imageColorSpace);

            // CGImageCreateWithJPEGDataProvider does not properly handle CMKY, so fall back to AFImageWithDataAtScale
            if (imageColorSpaceModel == kCGColorSpaceModelCMYK) {
                CGImageRelease(imageRef);
                imageRef = NULL;
            }
        }
    }

    CGDataProviderRelease(dataProvider);

    UIImage *image = AFImageWithDataAtScale(data, scale);
    if (!imageRef) {
        if (image.images || !image) {
            return image;
        }

        imageRef = CGImageCreateCopy([image CGImage]);
        if (!imageRef) {
            return nil;
        }
    }

    size_t width = CGImageGetWidth(imageRef);
    size_t height = CGImageGetHeight(imageRef);
    size_t bitsPerComponent = CGImageGetBitsPerComponent(imageRef);

    /// 如果图片太大就不解码了。因为解码的图片很占内存
    if (width * height > 1024 * 1024 || bitsPerComponent > 8) {
        CGImageRelease(imageRef);

        return image;
    }

    // CGImageGetBytesPerRow() calculates incorrectly in iOS 5.0, so defer to CGBitmapContextCreate
    /// 解码相关方法
    size_t bytesPerRow = 0;
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    CGColorSpaceModel colorSpaceModel = CGColorSpaceGetModel(colorSpace);
    CGBitmapInfo bitmapInfo = CGImageGetBitmapInfo(imageRef);

    if (colorSpaceModel == kCGColorSpaceModelRGB) {
        uint32_t alpha = (bitmapInfo & kCGBitmapAlphaInfoMask);
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wassign-enum"
        if (alpha == kCGImageAlphaNone) {
            bitmapInfo &= ~kCGBitmapAlphaInfoMask;
            bitmapInfo |= kCGImageAlphaNoneSkipFirst;
        } else if (!(alpha == kCGImageAlphaNoneSkipFirst || alpha == kCGImageAlphaNoneSkipLast)) {
            bitmapInfo &= ~kCGBitmapAlphaInfoMask;
            bitmapInfo |= kCGImageAlphaPremultipliedFirst;
        }
#pragma clang diagnostic pop
    }

    CGContextRef context = CGBitmapContextCreate(NULL, width, height, bitsPerComponent, bytesPerRow, colorSpace, bitmapInfo);

    CGColorSpaceRelease(colorSpace);

    if (!context) {
        CGImageRelease(imageRef);

        return image;
    }

    CGContextDrawImage(context, CGRectMake(0.0f, 0.0f, width, height), imageRef);
    CGImageRef inflatedImageRef = CGBitmapContextCreateImage(context);

    CGContextRelease(context);

    UIImage *inflatedImage = [[UIImage alloc] initWithCGImage:inflatedImageRef scale:scale orientation:image.imageOrientation];

    CGImageRelease(inflatedImageRef);
    CGImageRelease(imageRef);

    return inflatedImage;
}
```

这个方法能解析 png 和 jpeg 图片，会将图片解码，这样能加速图片的显示速度。

## AFCompoundResponseSerializer

复合类型的序列化器。在创建的时候需要传入上面的各种序列化方式的实例：

```objc
+ (instancetype)compoundSerializerWithResponseSerializers:(NSArray *)responseSerializers {
    AFCompoundResponseSerializer *serializer = [[self alloc] init];
    serializer.responseSerializers = responseSerializers;

    return serializer;
}
```

在获取到响应信息的时候会调用内部的各种序列化器尝试转换：

```objc
- (id)responseObjectForResponse:(NSURLResponse *)response
                           data:(NSData *)data
                          error:(NSError *__autoreleasing *)error
{
    for (id <AFURLResponseSerialization> serializer in self.responseSerializers) {
        if (![serializer isKindOfClass:[AFHTTPResponseSerializer class]]) {
            continue;
        }

        NSError *serializerError = nil;
        id responseObject = [serializer responseObjectForResponse:response data:data error:&serializerError];
        if (responseObject) {
            if (error) {
                *error = AFErrorWithUnderlyingError(serializerError, *error);
            }

            return responseObject;
        }
    }

    return [super responseObjectForResponse:response data:data error:error];
}
```

##  总结

总体来说，响应的序列化类不像请求那样涉及到输入流，因此还是比较清晰简单的。只要针对不同的 `content-type` 选取不同的系统类进行解析即可。

针对图片格式，AFNetworking 做了一部分图片解码转换的功能，但是和 SDWebImage 相比还是简单了很多。



