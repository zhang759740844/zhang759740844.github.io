title: AFNetworking 源码解析 - 请求序列化
date: 2018/12/25 14:07:12  
categories: iOS
tags: 

- 学习笔记
- 源码解析

------

本篇是 AFNetworking 源码解析的第二批，主要讲解如何生成请求 request。请求的序列化过程是非常重要的一个过程，涉及到多种处理和数据结构。自顶向下分析难度较大，因此本篇将先分析最底层的类。

<!--more-->

## 涉及到的类

### AFQueryStringPair

该类用于保存请求要发送的参数，从他的头文件中的属性可以看出：

```objc
@interface AFQueryStringPair : NSObject
@property (readwrite, nonatomic, strong) id field;
@property (readwrite, nonatomic, strong) id value;
@end
```

`field` 和 `value` 就代表着请求参数的键值对。它有一个方法，用于将参数转化为发送请求时的字符串样式以等号连接。它可以在 get 请求时被拼接到 url 中也可以在 post 请求时作为请求体放到 body 中：

```objc
/// field value 用 = 连接
- (NSString *)URLEncodedStringValue {
    if (!self.value || [self.value isEqual:[NSNull null]]) {
        return AFPercentEscapedStringFromString([self.field description]);
    } else {
        return [NSString stringWithFormat:@"%@=%@", AFPercentEscapedStringFromString([self.field description]), AFPercentEscapedStringFromString([self.value description])];
    }
}
```

其中涉及到了一个方法 `AFPercentEscapedStringFromString()`，它被用来将字符串编码。在 url 中，只能存在数字字母以及 `-_.~` 四个特殊符号，其他的符号都需要通过百分号转码，比如 `!` 要被转码为 `%21`，`=` 要被转码为 `%3D`。

` AFPercentEscapedStringFromString()` 会将字符串分割成数个小端，每个小端分别进行编码。编码的工作主要使用的是 NSString 的 `stringByAddingPercentEncodingWithAllowedCharacters:` 方法。还需要注意的是，对于 emoji 表情，一个 emoji 表情将占用两个字符的大小。因此，在分割字符串的时候，还需要对 emoji 在边界的情况做一个处理，防止劈开 emoji 表情：

```objc
#pragma mark - 将字符串编码
NSString * AFPercentEscapedStringFromString(NSString *string) {
    static NSString * const kAFCharactersGeneralDelimitersToEncode = @":#[]@"; // does not include "?" or "/" due to RFC 3986 - Section 3.4
    static NSString * const kAFCharactersSubDelimitersToEncode = @"!$&'()*+,;=";

    NSMutableCharacterSet * allowedCharacterSet = [[NSCharacterSet URLQueryAllowedCharacterSet] mutableCopy];
    /// 从允许的不要编码的字符中移除👆的那些字符
    [allowedCharacterSet removeCharactersInString:[kAFCharactersGeneralDelimitersToEncode stringByAppendingString:kAFCharactersSubDelimitersToEncode]];

    static NSUInteger const batchSize = 50;

    NSUInteger index = 0;
    NSMutableString *escaped = @"".mutableCopy;

    /// 循环每50个字符进行一次编码
    while (index < string.length) {
        NSUInteger length = MIN(string.length - index, batchSize);
        NSRange range = NSMakeRange(index, length);

        // To avoid breaking up character sequences such as 👴🏻👮🏽
        /// 一个 emoji 代表多个字符，因此要通过这个方法纠正，防止劈开 emoji 表情
        range = [string rangeOfComposedCharacterSequencesForRange:range];

        NSString *substring = [string substringWithRange:range];
        /// 将字符串编码
        NSString *encoded = [substring stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];
        [escaped appendString:encoded];

        index += range.length;
    }

	return escaped;
}
```

### AFHTTPBodyPart

一般请求不会用到这个类，当请求类型为 **multipart/form-data** 的时候，会创建多个 `AFHTTPBodyPart` 对象，每一个对象会有一个 `NSInputStream` 输入流，在生成请求的时候导入到 request 中。

#### 头文件

`AFHTTPBodyPart` 保存了 HTTP body的全部的信息。来看它的头部信息：

```objc
@interface AFHTTPBodyPart : NSObject
/// 编码方式
@property (nonatomic, assign) NSStringEncoding stringEncoding;
/// 头
@property (nonatomic, strong) NSDictionary *headers;
/// 边界字符串
@property (nonatomic, copy) NSString *boundary;
/// 主体内容
@property (nonatomic, strong) id body;
/// 主体大小
@property (nonatomic, assign) unsigned long long bodyContentLength;
/// 输入流
@property (nonatomic, strong) NSInputStream *inputStream;

/// 是否有初始边界
@property (nonatomic, assign) BOOL hasInitialBoundary;
@property (nonatomic, assign) BOOL hasFinalBoundary;

/// 是否还能从输入流中读取data
@property (readonly, nonatomic, assign, getter = hasBytesAvailable) BOOL bytesAvailable;
/// 总体的长度
@property (readonly, nonatomic, assign) unsigned long long contentLength;
@end
```

它的类拓展中还提供了几个属性作用于读取数据阶段：

```objc
@interface AFHTTPBodyPart () <NSCopying> {
    /// body 中四大部分的枚举
    AFHTTPBodyPartReadPhase _phase;
    /// 输入流
    NSInputStream *_inputStream;
    /// 读取的当前位置
    unsigned long long _phaseReadOffset;
}
```

其中枚举类型为。将在下面讲解方法的时候看到：

```objc
typedef enum {
    /// 前边界阶段
    AFEncapsulationBoundaryPhase = 1,
    /// 头部阶段
    AFHeaderPhase                = 2,
    /// body阶段
    AFBodyPhase                  = 3,
    /// 后边界阶段
    AFFinalBoundaryPhase         = 4,
} AFHTTPBodyPartReadPhase;
```

#### 方法

##### inputStream

接着依次看 `AFHTTPBodyPart` 的方法。首先是 `inputStream` 的 get 方法：

```objc
/// inputStream 的 get 方法
- (NSInputStream *)inputStream {
    if (!_inputStream) {
        /// body 有 NSData NSURL NSInputStream 三种类型
        if ([self.body isKindOfClass:[NSData class]]) {
            _inputStream = [NSInputStream inputStreamWithData:self.body];
        } else if ([self.body isKindOfClass:[NSURL class]]) {
            _inputStream = [NSInputStream inputStreamWithURL:self.body];
        } else if ([self.body isKindOfClass:[NSInputStream class]]) {
            _inputStream = self.body;
        } else {
            _inputStream = [NSInputStream inputStreamWithData:[NSData data]];
        }
    }

    return _inputStream;
}
```

它会把 `body` 转化为 `NSInputStream` ，无论 `body` 原本是 `NSData` 类型，`NSURL` 类型还是 `NSInputStream` 类型。

##### stringForHeaders

`stringForHeaders` 方法将 `headers` 字典转化为字符串：

```objc
/// 根据 self 的 header 字典属性来拼接 header 字符串
- (NSString *)stringForHeaders {
    NSMutableString *headerString = [NSMutableString string];
    for (NSString *field in [self.headers allKeys]) {
        /// 每个头部参数的键值对结束后插入换行 CRLF
        [headerString appendString:[NSString stringWithFormat:@"%@: %@%@", field, [self.headers valueForKey:field], kAFMultipartFormCRLF]];
    }
    /// 所有头部参数结束后，再插入一个换行符 CRLF
    [headerString appendString:kAFMultipartFormCRLF];

    return [NSString stringWithString:headerString];
}
```

> 这里的 header 不是 request header，而是 request body 中的 header。下面会有举例

##### contentLength

`contentLength` 用于计算 HTTP body 的整体长度。body 包含四个部分，分别对应于上面的 `AFHTTPBodyPartReadPhase` 枚举：

```objc
/// 总体的长度
- (unsigned long long)contentLength {
    unsigned long long length = 0;

    /// 前边界读取
    /// 如果数据体是第一个数据体，那么则插入首分隔字符串，否则，就插入中间分隔字符串，后者比前者多了一个回车换行符 \r\n
    NSData *encapsulationBoundaryData = [([self hasInitialBoundary] ? AFMultipartFormInitialBoundary(self.boundary) : AFMultipartFormEncapsulationBoundary(self.boundary)) dataUsingEncoding:self.stringEncoding];
    length += [encapsulationBoundaryData length];

    /// 头部读取
    NSData *headersData = [[self stringForHeaders] dataUsingEncoding:self.stringEncoding];
    length += [headersData length];

    /// 加上 body 的长度
    length += _bodyContentLength;

    /// 尾边界读取
    NSData *closingBoundaryData = ([self hasFinalBoundary] ? [AFMultipartFormFinalBoundary(self.boundary) dataUsingEncoding:self.stringEncoding] : [NSData data]);
    length += [closingBoundaryData length];

    return length;
}
```

这里涉及到以下四个方法：

```objc
/// 生成随机 boundary 字符串
static NSString * AFCreateMultipartFormBoundary() {
    return [NSString stringWithFormat:@"Boundary+%08X%08X", arc4random(), arc4random()];
}

/// 分行符
static NSString * const kAFMultipartFormCRLF = @"\r\n";

/// 生成初始边界字符串
static inline NSString * AFMultipartFormInitialBoundary(NSString *boundary) {
    return [NSString stringWithFormat:@"--%@%@", boundary, kAFMultipartFormCRLF];
}

/// 生成中间边界字符串 和初始字符串相比多了前部的 \r\n
static inline NSString * AFMultipartFormEncapsulationBoundary(NSString *boundary) {
    return [NSString stringWithFormat:@"%@--%@%@", kAFMultipartFormCRLF, boundary, kAFMultipartFormCRLF];
}

/// 生成结束边界字符串
static inline NSString * AFMultipartFormFinalBoundary(NSString *boundary) {
    return [NSString stringWithFormat:@"%@--%@--%@", kAFMultipartFormCRLF, boundary, kAFMultipartFormCRLF];
}
```

用来生成不同情况下的 boundary。具体作用将在 `read:maxLength:` 讲解。

##### hasBytesAvailable

`AFHTTPBodyPart` 中有一个输入流 `NSInputStream` 实例，这个方法就是用来查看输入流中是否还有可用数据的。根据输入流的状态判断：

```objc
/// 是否还有数据可读
- (BOOL)hasBytesAvailable {
    // Allows `read:maxLength:` to be called again if `AFMultipartFormFinalBoundary` doesn't fit into the available buffer
    if (_phase == AFFinalBoundaryPhase) {
        return YES;
    }

    switch (self.inputStream.streamStatus) {
        case NSStreamStatusNotOpen:
        case NSStreamStatusOpening:
        case NSStreamStatusOpen:
        case NSStreamStatusReading:
        case NSStreamStatusWriting:
            return YES;
        case NSStreamStatusAtEnd:
        case NSStreamStatusClosed:
        case NSStreamStatusError:
        default:
            return NO;
    }
}
```

##### transitionToNextPhase

这里用到了上面的枚举对象。根据当前所处阶段，将阶段设置为下一阶段：

```objc
/// 转移到下一阶段
- (BOOL)transitionToNextPhase {
    /// 保证代码在主线程
    if (![[NSThread currentThread] isMainThread]) {
        dispatch_sync(dispatch_get_main_queue(), ^{
            [self transitionToNextPhase];
        });
        return YES;
    }

    switch (_phase) {
        case AFEncapsulationBoundaryPhase:
            _phase = AFHeaderPhase;
            break;
        case AFHeaderPhase:
            /// 打开了一个输入流
            [self.inputStream scheduleInRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
            [self.inputStream open];
            _phase = AFBodyPhase;
            break;
        case AFBodyPhase:
            /// 将输入流关闭
            [self.inputStream close];
            _phase = AFFinalBoundaryPhase;
            break;
        case AFFinalBoundaryPhase:
        default:
            _phase = AFEncapsulationBoundaryPhase;
            break;
    }
    /// 重置 offset
    _phaseReadOffset = 0;

    return YES;
}
```

##### read:maxLength:

这是 `AFHTTPBodyPart` 的核心方法。`AFHTTPBodyPart` 封装了 `multipart/form-data` 情况下 body 中的部分数据。那么在发送的时候一定要把它读出来吧。`read:maxLength:` 的目的就是如此，将数据放入 `buffer` 中：

```objc
/// 读取 buffer
- (NSInteger)read:(uint8_t *)buffer
        maxLength:(NSUInteger)length
{
    /// 已经读取的字节数
    NSInteger totalNumberOfBytesRead = 0;

    /// 前边界读取
    /// 如果数据体是第一个数据体，那么则插入首分隔字符串，否则，就插入中间分隔字符串，后者比前者多了一个回车换行符 \r\n
    if (_phase == AFEncapsulationBoundaryPhase) {
        NSData *encapsulationBoundaryData = [([self hasInitialBoundary] ? AFMultipartFormInitialBoundary(self.boundary) : AFMultipartFormEncapsulationBoundary(self.boundary)) dataUsingEncoding:self.stringEncoding];
        totalNumberOfBytesRead += [self readData:encapsulationBoundaryData intoBuffer:&buffer[totalNumberOfBytesRead] maxLength:(length - (NSUInteger)totalNumberOfBytesRead)];
    }

    /// 头部信息读取
    if (_phase == AFHeaderPhase) {
        NSData *headersData = [[self stringForHeaders] dataUsingEncoding:self.stringEncoding];
        totalNumberOfBytesRead += [self readData:headersData intoBuffer:&buffer[totalNumberOfBytesRead] maxLength:(length - (NSUInteger)totalNumberOfBytesRead)];
    }

    /// 数据体读取
    /// 调用输入流进行数据读取
    if (_phase == AFBodyPhase) {
        NSInteger numberOfBytesRead = 0;

        numberOfBytesRead = [self.inputStream read:&buffer[totalNumberOfBytesRead] maxLength:(length - (NSUInteger)totalNumberOfBytesRead)];
        if (numberOfBytesRead == -1) {
            return -1;
        } else {
            totalNumberOfBytesRead += numberOfBytesRead;
            /// 如果读到了流的末尾，那么进入下一个读取阶段
            if ([self.inputStream streamStatus] >= NSStreamStatusAtEnd) {
                [self transitionToNextPhase];
            }
        }
    }

    /// 后边界读取
    /// 如果数据体是最后一个数据，那么插入后边界字符串，否则，插入空值
    if (_phase == AFFinalBoundaryPhase) {
        NSData *closingBoundaryData = ([self hasFinalBoundary] ? [AFMultipartFormFinalBoundary(self.boundary) dataUsingEncoding:self.stringEncoding] : [NSData data]);
        totalNumberOfBytesRead += [self readData:closingBoundaryData intoBuffer:&buffer[totalNumberOfBytesRead] maxLength:(length - (NSUInteger)totalNumberOfBytesRead)];
    }

    /// 最后返回读取到的所有数据的字节数
    return totalNumberOfBytesRead;
}

/// 读取 data 到 buffer 中
- (NSInteger)readData:(NSData *)data
           intoBuffer:(uint8_t *)buffer
            maxLength:(NSUInteger)length
{
    NSRange range = NSMakeRange((NSUInteger)_phaseReadOffset, MIN([data length] - ((NSUInteger)_phaseReadOffset), length));
    [data getBytes:buffer range:range];

    _phaseReadOffset += range.length;

    ///如果偏移量大于等于数据的长度，说明数据读取完毕，可以进入下一个阶段
    if (((NSUInteger)_phaseReadOffset) >= [data length]) {
        [self transitionToNextPhase];
    }

    return (NSInteger)range.length;
}
```

这个方法中有涉及到了边界相关的几个方法。当 http 请求 type 为 `multipart/form-data` 时，它会将表单中的数据处理为一条消息，以标签为单元，用分隔符分开。既可以上传键值对，也可以上传文件。当上传的字段是文件时，会有Content-Type来表名文件类型：`content-disposition`，用来说明字段的一些信息。如：

```http
--boundary+004563210AB32145
Content-Disposition: form-data; name="field1"

value1
--boundary+004563210AB32145
Content-Disposition: form-data; name="field2"

value2
--boundary+004563210AB32145
Content-Disposition: form-data; name="text"; filename="file.txt"

data1
--boundary+004563210AB32145
Content-Disposition: form-data; name="pic"; filename="icon.png"

data2
--boundary+004563210AB32145--
```

> 每两个 boundary 之间的数据就是一个 `AFHTTPBodyPart` 实例。也就是 `read:maxLength:` 读取生成的 buffer 的数据 

`--boundary+004563210AB32145` 就是由上面的四个方法生成的。针对 boundary 所处的不同位置，会有不同的添加换行符的规则。

### AFMultipartBodyStream

之前 `AFHTTPBodyPart` 是 form-data 下 body 中的各个部分。现在的 `AFMutipartBodyStream` 则是 `AFHTTPBodyPart` 的集合。

#### 头文件

```objc
@interface AFMultipartBodyStream : NSInputStream <NSStreamDelegate>
@property (nonatomic, assign) NSUInteger numberOfBytesInPacket;
@property (nonatomic, assign) NSTimeInterval delay;
@property (nonatomic, strong) NSInputStream *inputStream;
@property (readonly, nonatomic, assign) unsigned long long contentLength;
@property (readonly, nonatomic, assign, getter = isEmpty) BOOL empty;
@end
  
@interface AFMultipartBodyStream () <NSCopying>
@property (readwrite, nonatomic, assign) NSStringEncoding stringEncoding;
@property (readwrite, nonatomic, strong) NSMutableArray *HTTPBodyParts;
@property (readwrite, nonatomic, strong) NSEnumerator *HTTPBodyPartEnumerator;
@property (readwrite, nonatomic, strong) AFHTTPBodyPart *currentHTTPBodyPart;
@property (readwrite, nonatomic, strong) NSOutputStream *outputStream;
@property (readwrite, nonatomic, strong) NSMutableData *buffer;
@end
```

可以发现 `AFMutipartBodyStream` 是继承于 `NSInputStream` 的。我们需要清楚的是，`HTTPBodyParts` 是以一个数组的形式存在于它的属性中。

#### 方法

##### setInitialAndFinalBoundaries

由于它内内存存在一个 `AFHTTPBodyPart` 的数组，这个方法就是将数组中第一个设置为存在初始边界，最后一个存在末尾边界，中间的则是普通边界的情况：

```objc
/// 一个 AFMultipartBodyStream 中有多个 HTTPBodyParts，最前面的设置 initialBoundary 最后面的设置 finalBoundary
- (void)setInitialAndFinalBoundaries {
    if ([self.HTTPBodyParts count] > 0) {
        for (AFHTTPBodyPart *bodyPart in self.HTTPBodyParts) {
            bodyPart.hasInitialBoundary = NO;
            bodyPart.hasFinalBoundary = NO;
        }

        [[self.HTTPBodyParts firstObject] setHasInitialBoundary:YES];
        [[self.HTTPBodyParts lastObject] setHasFinalBoundary:YES];
    }
}

```

##### appendHTTPBodyPart:

既然有一个数组，那么数组元素当然要一个个添加进来啦。这个方法就是用来添加元素的：

```objc
/// 增加HTTPBodyPart实例到数组中
- (void)appendHTTPBodyPart:(AFHTTPBodyPart *)bodyPart {
    [self.HTTPBodyParts addObject:bodyPart];
}
```

##### read:maxLength:

这个方法在 `AFHTTPBodyPart` 中也有，其实他是 `NSInputStream` 的方法，在 `AFMultipartBodyStream` 中被重写了：

```objc
/// 读取 HTTPBodyPart 中的数据到 buffer 中
- (NSInteger)read:(uint8_t *)buffer
        maxLength:(NSUInteger)length
{
    if ([self streamStatus] == NSStreamStatusClosed) {
        return 0;
    }

    /// 已读数据
    NSInteger totalNumberOfBytesRead = 0;
    /// 遍历读取数据
    while ((NSUInteger)totalNumberOfBytesRead < MIN(length, self.numberOfBytesInPacket)) {
        ///  如果当前读取的body不存在或者body没有可读字节
        if (!self.currentHTTPBodyPart || ![self.currentHTTPBodyPart hasBytesAvailable]) {
            ///把下一个body赋值给当前的body 如果下一个为nil 就退出循环
            if (!(self.currentHTTPBodyPart = [self.HTTPBodyPartEnumerator nextObject])) {
                break;
            }
        /// 当前body存在
        } else {
            // 剩余可读文件的大小
            NSUInteger maxLength = MIN(length, self.numberOfBytesInPacket) - (NSUInteger)totalNumberOfBytesRead;
            /// 把当前的body的数据读入到buffer中
            NSInteger numberOfBytesRead = [self.currentHTTPBodyPart read:&buffer[totalNumberOfBytesRead] maxLength:maxLength];
            if (numberOfBytesRead == -1) {
                self.streamError = self.currentHTTPBodyPart.inputStream.streamError;
                break;
            } else {
                /// 读完之后增加已读长度
                totalNumberOfBytesRead += numberOfBytesRead;

                /// 读完之后是否需要有 delay
                if (self.delay > 0.0f) {
                    [NSThread sleepForTimeInterval:self.delay];
                }
            }
        }
    }

    return totalNumberOfBytesRead;
}
```

简而言之，就是调用每一个 `AFHTTPBodyPart` 实例的相应方法，把他们的所有 buffer 汇集在一起。

> buffer 读完之后有什么用，后面再说

##### contentLength

将所有 `AFHTTPBodyPart` 的长度都累加起来成为最终的 content-length

```objc
- (unsigned long long)contentLength {
    unsigned long long length = 0;
    for (AFHTTPBodyPart *bodyPart in self.HTTPBodyParts) {
        length += [bodyPart contentLength];
    }

    return length;
}
```

##### 开闭 NSInputStream

这也是 `NSInputStream`中的方法，在读取 buffer 前后要分别 open 以及 close

```objc
/// 重写 NSInputStream 的方法
- (void)open {
    if (self.streamStatus == NSStreamStatusOpen) {
        return;
    }

    self.streamStatus = NSStreamStatusOpen;

    [self setInitialAndFinalBoundaries];
    self.HTTPBodyPartEnumerator = [self.HTTPBodyParts objectEnumerator];
}

- (void)close {
    self.streamStatus = NSStreamStatusClosed;
}
```

### AFStreamingMultipartFormData

前面 `AFMultipartBodyStream` 包含了多个 `AFHTTPBodyPart` 实例，且可以获取到所有的 buffer，他们准确的说就是model类，没有更多的操作。而此处的 `AFStreamingMultipartFormData` 则更像是一种 manager，可以管理 `AFMultipartBodyStream`。

#### 头文件

```objc
@interface AFStreamingMultipartFormData ()
/// NSMutableURLRequest
@property (readwrite, nonatomic, copy) NSMutableURLRequest *request;
@property (readwrite, nonatomic, assign) NSStringEncoding stringEncoding;
@property (readwrite, nonatomic, copy) NSString *boundary;
/// AFMultipartBodyStream 实例
@property (readwrite, nonatomic, strong) AFMultipartBodyStream *bodyStream;
@end

```

头文件中主需要关注两个属性，一个是 `NSMutableURLRequest` 实例，还有一个是 `SFMultipartBodyStream` 实例。它将请求和输入流封装了起来。

#### 方法

##### 初始化

初始化方法很简单，就是对头文件中属性的设置。它要传入一个 `NSMutableURLRequest` 实例。

```objc
- (instancetype)initWithURLRequest:(NSMutableURLRequest *)urlRequest
                    stringEncoding:(NSStringEncoding)encoding
{
    self = [super init];
    if (!self) {
        return nil;
    }

    self.request = urlRequest;
    self.stringEncoding = encoding;
    self.boundary = AFCreateMultipartFormBoundary();
    self.bodyStream = [[AFMultipartBodyStream alloc] initWithStringEncoding:encoding];

    return self;
}
```

##### 通过文件路径增加 AFHTTPBodyPart

如果要上传一个本地文件，那么就把本地文件的路径设置给新创建的 `SFHTTPBodyPart` 实例中。

```objc
- (BOOL)appendPartWithFileURL:(NSURL *)fileURL
                         name:(NSString *)name
                        error:(NSError * __autoreleasing *)error
{
    NSParameterAssert(fileURL);
    NSParameterAssert(name);

    NSString *fileName = [fileURL lastPathComponent];
    NSString *mimeType = AFContentTypeForPathExtension([fileURL pathExtension]);

    return [self appendPartWithFileURL:fileURL name:name fileName:fileName mimeType:mimeType error:error];
}

/// 将 fileURL 下的文件包装为 AFHTTPBodyPart 实例，保存下来
- (BOOL)appendPartWithFileURL:(NSURL *)fileURL
                         name:(NSString *)name
                     fileName:(NSString *)fileName
                     mimeType:(NSString *)mimeType
                        error:(NSError * __autoreleasing *)error
{
    NSParameterAssert(fileURL);
    NSParameterAssert(name);
    NSParameterAssert(fileName);
    NSParameterAssert(mimeType);

    if (![fileURL isFileURL]) {
        NSDictionary *userInfo = @{NSLocalizedFailureReasonErrorKey: NSLocalizedStringFromTable(@"Expected URL to be a file URL", @"AFNetworking", nil)};
        if (error) {
            *error = [[NSError alloc] initWithDomain:AFURLRequestSerializationErrorDomain code:NSURLErrorBadURL userInfo:userInfo];
        }

        return NO;
    /// 检查 url 下的文件是否存在
    } else if ([fileURL checkResourceIsReachableAndReturnError:error] == NO) {
        NSDictionary *userInfo = @{NSLocalizedFailureReasonErrorKey: NSLocalizedStringFromTable(@"File URL not reachable.", @"AFNetworking", nil)};
        if (error) {
            *error = [[NSError alloc] initWithDomain:AFURLRequestSerializationErrorDomain code:NSURLErrorBadURL userInfo:userInfo];
        }

        return NO;
    }

    /// 通过 NSFileManager 提取 fileURL 所示文件的信息
    NSDictionary *fileAttributes = [[NSFileManager defaultManager] attributesOfItemAtPath:[fileURL path] error:error];
    if (!fileAttributes) {
        return NO;
    }

    NSMutableDictionary *mutableHeaders = [NSMutableDictionary dictionary];
    /// 设置 header
    [mutableHeaders setValue:[NSString stringWithFormat:@"form-data; name=\"%@\"; filename=\"%@\"", name, fileName] forKey:@"Content-Disposition"];
    [mutableHeaders setValue:mimeType forKey:@"Content-Type"];

    AFHTTPBodyPart *bodyPart = [[AFHTTPBodyPart alloc] init];
    bodyPart.stringEncoding = self.stringEncoding;
    bodyPart.headers = mutableHeaders;
    bodyPart.boundary = self.boundary;
    bodyPart.body = fileURL;
    /// 获取文件的大小
    bodyPart.bodyContentLength = [fileAttributes[NSFileSize] unsignedLongLongValue];
    /// 把创建的 AFHTTPBodyPart 放到 AFMultipartBodyStream 属性中
    [self.bodyStream appendHTTPBodyPart:bodyPart];

    return YES;
}
```

##### 通过输入流增加 AFHTTPBodyPart

把传入的输入流作为 AFHTTPBodyPart 的 body 保存

```objc
///  把 inputStream 封装为 AFHTTPBodyPart，保存
- (void)appendPartWithInputStream:(NSInputStream *)inputStream
                             name:(NSString *)name
                         fileName:(NSString *)fileName
                           length:(int64_t)length
                         mimeType:(NSString *)mimeType
{
    NSParameterAssert(name);
    NSParameterAssert(fileName);
    NSParameterAssert(mimeType);

    NSMutableDictionary *mutableHeaders = [NSMutableDictionary dictionary];
    [mutableHeaders setValue:[NSString stringWithFormat:@"form-data; name=\"%@\"; filename=\"%@\"", name, fileName] forKey:@"Content-Disposition"];
    [mutableHeaders setValue:mimeType forKey:@"Content-Type"];

    AFHTTPBodyPart *bodyPart = [[AFHTTPBodyPart alloc] init];
    bodyPart.stringEncoding = self.stringEncoding;
    bodyPart.headers = mutableHeaders;
    bodyPart.boundary = self.boundary;
    bodyPart.body = inputStream;

    bodyPart.bodyContentLength = (unsigned long long)length;

    [self.bodyStream appendHTTPBodyPart:bodyPart];
}

```

##### 通过 NSData 增加 AFHTTPBodyPart

如果直接传入了 NSData，那么直接把他保存到 `AFHTTPBodyPart` 的 body 中：

```objc
- (void)appendPartWithFileData:(NSData *)data
                          name:(NSString *)name
                      fileName:(NSString *)fileName
                      mimeType:(NSString *)mimeType
{
    NSParameterAssert(name);
    NSParameterAssert(fileName);
    NSParameterAssert(mimeType);

    NSMutableDictionary *mutableHeaders = [NSMutableDictionary dictionary];
    [mutableHeaders setValue:[NSString stringWithFormat:@"form-data; name=\"%@\"; filename=\"%@\"", name, fileName] forKey:@"Content-Disposition"];
    [mutableHeaders setValue:mimeType forKey:@"Content-Type"];

    [self appendPartWithHeaders:mutableHeaders body:data];
}

- (void)appendPartWithFormData:(NSData *)data
                          name:(NSString *)name
{
    NSParameterAssert(name);

    NSMutableDictionary *mutableHeaders = [NSMutableDictionary dictionary];
    [mutableHeaders setValue:[NSString stringWithFormat:@"form-data; name=\"%@\"", name] forKey:@"Content-Disposition"];

    [self appendPartWithHeaders:mutableHeaders body:data];
}

/// 把 NSData 封装为 AFHTTPBodyPart 保存
- (void)appendPartWithHeaders:(NSDictionary *)headers
                         body:(NSData *)body
{
    NSParameterAssert(body);

    AFHTTPBodyPart *bodyPart = [[AFHTTPBodyPart alloc] init];
    bodyPart.stringEncoding = self.stringEncoding;
    bodyPart.headers = headers;
    bodyPart.boundary = self.boundary;
    bodyPart.bodyContentLength = [body length];
    bodyPart.body = body;

    [self.bodyStream appendHTTPBodyPart:bodyPart];
}
```

##### 返回 NSMutableURLRequest 实例

前面提到过，读取的 buffer 有什么用呢？答案就在这个方法中：

```objc
- (NSMutableURLRequest *)requestByFinalizingMultipartFormData {
    if ([self.bodyStream isEmpty]) {
        return self.request;
    }

    // Reset the initial and final boundaries to ensure correct Content-Length
    [self.bodyStream setInitialAndFinalBoundaries];
    /// 把 bodyStream 设置给 request
    [self.request setHTTPBodyStream:self.bodyStream];

    /// 设置 content-type 和 content-length
    [self.request setValue:[NSString stringWithFormat:@"multipart/form-data; boundary=%@", self.boundary] forHTTPHeaderField:@"Content-Type"];
    [self.request setValue:[NSString stringWithFormat:@"%llu", [self.bodyStream contentLength]] forHTTPHeaderField:@"Content-Length"];

    return self.request;
}
```

`NSMutableURLRequest` 有一个属性是 `HTTPBodyStream`，它是 `NSInputStream` 类型，在生成请求的时候会自动调用其 `HTTPBodyStream` 实例的 `read:maxLength:` 方法。获取 buffer 数据写入请求体中。

> `NSMutableURLRequest` 有两个属性用来设置请求体：
>
> 1. `NSInputStream` 类型的 `HTTPBodyStream` ，也就是上面说的，是以数据流的形式生成请求体，需要实现 `NSInputStream` 的 `read:maxLength:` 方法，返回 buffer。主要使用在 `content-type` 为 `multipart/form-data` 的情况。
> 2. `NSData` 类型的 `HTTPBody`，直接将 `NSData` 放入请求中。主要用在非 `multipart/form-data` 的情况，包括 `application/json`，`application/x-www-form-urlencoded` 等

## AFHTTPRequestSerializer

### 初始化

初始化方法比较长。做了以下几件事：

- 创建了一个并行队列，用于实现读写锁
- 设置请求头，包裹 UA 和 Accept-Language
- 给部分属性添加 KVO

```objc
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.stringEncoding = NSUTF8StringEncoding;

    self.mutableHTTPRequestHeaders = [NSMutableDictionary dictionary];
    /// 请求头读写队列，以此并行队列实现读写锁
    self.requestHeaderModificationQueue = dispatch_queue_create("requestHeaderModificationQueue", DISPATCH_QUEUE_CONCURRENT);

    // Accept-Language HTTP Header; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.4
    NSMutableArray *acceptLanguagesComponents = [NSMutableArray array];
    /// 设置 Accept-Language
    [[NSLocale preferredLanguages] enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
        float q = 1.0f - (idx * 0.1f);
        [acceptLanguagesComponents addObject:[NSString stringWithFormat:@"%@;q=%0.1g", obj, q]];
        *stop = q <= 0.5f;
    }];
    [self setValue:[acceptLanguagesComponents componentsJoinedByString:@", "] forHTTPHeaderField:@"Accept-Language"];

    /// 设置 useragent
    NSString *userAgent = nil;
    // User-Agent Header; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.43
    userAgent = [NSString stringWithFormat:@"%@/%@ (%@; iOS %@; Scale/%0.2f)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[UIDevice currentDevice] model], [[UIDevice currentDevice] systemVersion], [[UIScreen mainScreen] scale]];
    if (userAgent) {
        if (![userAgent canBeConvertedToEncoding:NSASCIIStringEncoding]) {
            NSMutableString *mutableUserAgent = [userAgent mutableCopy];
            if (CFStringTransform((__bridge CFMutableStringRef)(mutableUserAgent), NULL, (__bridge CFStringRef)@"Any-Latin; Latin-ASCII; [:^ASCII:] Remove", false)) {
                userAgent = mutableUserAgent;
            }
        }
        [self setValue:userAgent forHTTPHeaderField:@"User-Agent"];
    }

    // HTTP Method Definitions; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html
    /// 以下三种请求会把 param 放在 uri 中
    self.HTTPMethodsEncodingParametersInURI = [NSSet setWithObjects:@"GET", @"HEAD", @"DELETE", nil];

    /// 用来记录哪些 keyPaths 改变了
    self.mutableObservedChangedKeyPaths = [NSMutableSet set];
    /// 给 AFHTTPRequestSerializerObservedKeyPaths 中的属性添加 KVO
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self respondsToSelector:NSSelectorFromString(keyPath)]) {
            [self addObserver:self forKeyPath:keyPath options:NSKeyValueObservingOptionNew context:AFHTTPRequestSerializerObserverContext];
        }
    }

    return self;
}
```

### 添加 KVO

添加 KVO 的属性由以下方法提供：

```objc
#pragma mark - AFHTTPRequestSeri 通过KVO监听以下的几个属性
static NSArray * AFHTTPRequestSerializerObservedKeyPaths() {
    static NSArray *_AFHTTPRequestSerializerObservedKeyPaths = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _AFHTTPRequestSerializerObservedKeyPaths = @[NSStringFromSelector(@selector(allowsCellularAccess)), NSStringFromSelector(@selector(cachePolicy)), NSStringFromSelector(@selector(HTTPShouldHandleCookies)), NSStringFromSelector(@selector(HTTPShouldUsePipelining)), NSStringFromSelector(@selector(networkServiceType)), NSStringFromSelector(@selector(timeoutInterval))];
    });

    return _AFHTTPRequestSerializerObservedKeyPaths;
}
```

包括 是否允许使用蜂窝网络，缓存策略，是否使用 cookies，是否使用管线连接，网络服务类型，超时时间。当他们被重新设置后，会触发 KVO 方法：

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(__unused id)object
                        change:(NSDictionary *)change
                       context:(void *)context
{
    if (context == AFHTTPRequestSerializerObserverContext) {
        if ([change[NSKeyValueChangeNewKey] isEqual:[NSNull null]]) {
            [self.mutableObservedChangedKeyPaths removeObject:keyPath];
        } else {
            [self.mutableObservedChangedKeyPaths addObject:keyPath];
        }
    }
}
```

也就是说，如果改变了就会被加入到 `mutableObservedChangedKeyPaths` 数组中。这样，在创建 request 的时候就可以通过这个数组获取到变化的信息。

### 设置头部信息

设置 header 本来就是简单的两个 get set 方法，但是还是要拿出来说一下是因为它通过一个并行队列以及 `dispatch_barrier_async` 实现了一个读写锁：

```objc
- (void)setValue:(NSString *)value
forHTTPHeaderField:(NSString *)field
{
    dispatch_barrier_async(self.requestHeaderModificationQueue, ^{
        [self.mutableHTTPRequestHeaders setValue:value forKey:field];
    });
}

- (NSString *)valueForHTTPHeaderField:(NSString *)field {
    NSString __block *value;
    dispatch_sync(self.requestHeaderModificationQueue, ^{
        value = [self.mutableHTTPRequestHeaders valueForKey:field];
    });
    return value;
}
```

### 各种创建 NSMutableURLRequest 的方式

#### application/x-www-form-urlencoded

`application/x-www-form-urlencoded` 的 request 将参数拼接到 url 上或者放在 body 中：

```objc
/// 生成普通 urlrequest，param 设置到body或者uri上
- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(id)parameters
                                     error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(method);
    NSParameterAssert(URLString);

    NSURL *url = [NSURL URLWithString:URLString];

    NSParameterAssert(url);

    NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
    mutableRequest.HTTPMethod = method;

    ///将request的各种属性循环遍历
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        /// 如果观察到的属性改变了
        if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
            /// 将属性设置给 request
            [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
        }
    }

    /// 将 param 拼到 uri 或者放到 body 中
    mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];

	return mutableRequest;
}
```

调用 `requestBySerializingRequest:withparameters:error:` 处理参数：

```objc
/// 将 param 拼到 uri 或放在 body
- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(request);

    NSMutableURLRequest *mutableRequest = [request mutableCopy];

    /// 设置header给 request
    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];

    NSString *query = nil;
    if (parameters) {
        /// 如果有自定义的序列化方法，直接调用
        if (self.queryStringSerialization) {
            NSError *serializationError;
            query = self.queryStringSerialization(request, parameters, &serializationError);

            if (serializationError) {
                if (error) {
                    *error = serializationError;
                }

                return nil;
            }
        } else {
            /// 没有自定义的序列化方法调用默认序列化方法
            switch (self.queryStringSerializationStyle) {
                case AFHTTPRequestQueryStringDefaultStyle:
                    query = AFQueryStringFromParameters(parameters);
                    break;
            }
        }
    }

    /// 是否是 get delete header 三种请求中第一个
    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        if (query && query.length > 0) {
            /// 如果是的话，直接把 query信息拼到 uri 后面
            mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
        }
    } else {
        // #2864: an empty string is a valid x-www-form-urlencoded payload
        if (!query) {
            query = @"";
        }
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
        }
        /// 如果不是上述几种，把 query 信息放到 http body 里
        [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
    }

    return mutableRequest;
}
```

如果是 get，delete，header 中的一个，那么就会直接把 query 信息拼到 uri 上，其他的都会塞到 body 中。

如果有自定义序列化方法会调用自定义的方法将 param 序列化成字符串。一般情况下我们可能不需要自定义的序列化方法，不过有些时候，比如说想通过 get 请求传递一个数组，服务端有自己的解析逻辑，那么数组如何展开就要通过自定义的序列化方式进行了：

```objc
- (void)setQueryStringSerializationWithBlock:(NSString *(^)(NSURLRequest *, id, NSError *__autoreleasing *))block {
    self.queryStringSerialization = block;
}
```

没有就通过默认的方式，即 `AFQueryStringFromParametors()` 方法：

```objc
/// 将传入的 parameters 字典转为数组，数组中的保存的是包含 key value 的 AFQueryStringPair 对象。
/// 注意如果字典中包含字典，那么 key 的形式是 “a[b]” 的方式保存；如果字典中包含数组，那么 key 的形式是 “a[]” 的方式保存
NSArray * AFQueryStringPairsFromDictionary(NSDictionary *dictionary) {
    return AFQueryStringPairsFromKeyAndValue(nil, dictionary);
}

NSArray * AFQueryStringPairsFromKeyAndValue(NSString *key, id value) {
    NSMutableArray *mutableQueryStringComponents = [NSMutableArray array];

    NSSortDescriptor *sortDescriptor = [NSSortDescriptor sortDescriptorWithKey:@"description" ascending:YES selector:@selector(compare:)];

    if ([value isKindOfClass:[NSDictionary class]]) {
        NSDictionary *dictionary = value;
        // Sort dictionary keys to ensure consistent ordering in query string, which is important when deserializing potentially ambiguous sequences, such as an array of dictionaries
        for (id nestedKey in [dictionary.allKeys sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
            id nestedValue = dictionary[nestedKey];
            if (nestedValue) {
                [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue((key ? [NSString stringWithFormat:@"%@[%@]", key, nestedKey] : nestedKey), nestedValue)];
            }
        }
    } else if ([value isKindOfClass:[NSArray class]]) {
        NSArray *array = value;
        for (id nestedValue in array) {
            [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue([NSString stringWithFormat:@"%@[]", key], nestedValue)];
        }
    } else if ([value isKindOfClass:[NSSet class]]) {
        NSSet *set = value;
        for (id obj in [set sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
            [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue(key, obj)];
        }
    } else {
        [mutableQueryStringComponents addObject:[[AFQueryStringPair alloc] initWithField:key value:value]];
    }

    return mutableQueryStringComponents;
}
```

这种序列化方式是最常见的方式数组以 `a[]=xxx&a[]=yyy&a[]=zzz` 的形式提交，字典以`a[b]=xxx&a[c]=yyy&a[d]=zzz` 的方式提交。

> 不过这种一般以 json 的方式提交会更方便。

#### multipart/form-data

这种类型的请求上面已经讲了很多了，需要一个 `NSInputStream`:

```objc
- (NSMutableURLRequest *)multipartFormRequestWithMethod:(NSString *)method
                                              URLString:(NSString *)URLString
                                             parameters:(NSDictionary *)parameters
                              constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                                                  error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(method);
    NSParameterAssert(![method isEqualToString:@"GET"] && ![method isEqualToString:@"HEAD"]);

    /// 生成普通的 urlrequest
    NSMutableURLRequest *mutableRequest = [self requestWithMethod:method URLString:URLString parameters:nil error:error];

    __block AFStreamingMultipartFormData *formData = [[AFStreamingMultipartFormData alloc] initWithURLRequest:mutableRequest stringEncoding:NSUTF8StringEncoding];

    if (parameters) {
      	/// 循环拿到参数，判断 value 是否是 NSData，然后创建 AFHTTPBodyPart 实例，把NSData转为NSInputStream
        for (AFQueryStringPair *pair in AFQueryStringPairsFromDictionary(parameters)) {
            NSData *data = nil;
            if ([pair.value isKindOfClass:[NSData class]]) {
                data = pair.value;
            } else if ([pair.value isEqual:[NSNull null]]) {
                data = [NSData data];
            } else {
                data = [[pair.value description] dataUsingEncoding:self.stringEncoding];
            }

            if (data) {
                [formData appendPartWithFormData:data name:[pair.field description]];
            }
        }
    }

    if (block) {
        block(formData);
    }

    return [formData requestByFinalizingMultipartFormData];
}

- (NSMutableURLRequest *)requestByFinalizingMultipartFormData {
    if ([self.bodyStream isEmpty]) {
        return self.request;
    }

    // Reset the initial and final boundaries to ensure correct Content-Length
    [self.bodyStream setInitialAndFinalBoundaries];
    /// 把 bodyStream 设置给 request
    [self.request setHTTPBodyStream:self.bodyStream];

    /// 设置 content-type 和 content-length
    [self.request setValue:[NSString stringWithFormat:@"multipart/form-data; boundary=%@", self.boundary] forHTTPHeaderField:@"Content-Type"];
    [self.request setValue:[NSString stringWithFormat:@"%llu", [self.bodyStream contentLength]] forHTTPHeaderField:@"Content-Length"];

    return self.request;
}
```

它会把所有的参数都转为 `NSData` 的形式，然后转为 `NSInputStream`，设置给 request。

#### 特殊的 request

```objc
- (NSMutableURLRequest *)requestWithMultipartFormRequest:(NSURLRequest *)request
                             writingStreamContentsToFile:(NSURL *)fileURL
                                       completionHandler:(void (^)(NSError *error))handler
{
    NSParameterAssert(request.HTTPBodyStream);
    NSParameterAssert([fileURL isFileURL]);

    NSInputStream *inputStream = request.HTTPBodyStream;
    NSOutputStream *outputStream = [[NSOutputStream alloc] initWithURL:fileURL append:NO];
    __block NSError *error = nil;

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [inputStream scheduleInRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];
        [outputStream scheduleInRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];

        [inputStream open];
        [outputStream open];

        while ([inputStream hasBytesAvailable] && [outputStream hasSpaceAvailable]) {
            uint8_t buffer[1024];

            NSInteger bytesRead = [inputStream read:buffer maxLength:1024];
            if (inputStream.streamError || bytesRead < 0) {
                error = inputStream.streamError;
                break;
            }

            NSInteger bytesWritten = [outputStream write:buffer maxLength:(NSUInteger)bytesRead];
            if (outputStream.streamError || bytesWritten < 0) {
                error = outputStream.streamError;
                break;
            }

            if (bytesRead == 0 && bytesWritten == 0) {
                break;
            }
        }

        [outputStream close];
        [inputStream close];

        if (handler) {
            dispatch_async(dispatch_get_main_queue(), ^{
                handler(error);
            });
        }
    });

    NSMutableURLRequest *mutableRequest = [request mutableCopy];
    mutableRequest.HTTPBodyStream = nil;

    return mutableRequest;
}
```

这个方法有点特殊，它把 request 的输入流直接连向了本地的输出流。将输入流里的数据，直接写到了本地的 `fileURL` 中。不太明白其中的用意，以及使用场景。

## AFHTTPRequestSerializer 的两个子类

### AFJSONRequestSerializer

该类用于发送 `application/json`  请求。它仅重写了 `AFHTTPRequestSerializer` 的一个方法：

```objc
- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(request);

    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        return [super requestBySerializingRequest:request withParameters:parameters error:error];
    }

    NSMutableURLRequest *mutableRequest = [request mutableCopy];

    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];

    if (parameters) {
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
        }

        if (![NSJSONSerialization isValidJSONObject:parameters]) {
            if (error) {
                NSDictionary *userInfo = @{NSLocalizedFailureReasonErrorKey: NSLocalizedStringFromTable(@"The `parameters` argument is not valid JSON.", @"AFNetworking", nil)};
                *error = [[NSError alloc] initWithDomain:AFURLRequestSerializationErrorDomain code:NSURLErrorCannotDecodeContentData userInfo:userInfo];
            }
            return nil;
        }

        NSData *jsonData = [NSJSONSerialization dataWithJSONObject:parameters options:self.writingOptions error:error];
        
        if (!jsonData) {
            return nil;
        }
        
        [mutableRequest setHTTPBody:jsonData];
    }

    return mutableRequest;
}
```

将传入的参数转为了 JSON 的形式，设置为 request 的 `HTTPBody`

### AFPropertyListRequestSerializer

它和上面的类似，也是重写了同样的方法，用于发送 `application/x-plist` 请求：

```objc
- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(request);

    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        return [super requestBySerializingRequest:request withParameters:parameters error:error];
    }

    NSMutableURLRequest *mutableRequest = [request mutableCopy];

    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];

    if (parameters) {
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/x-plist" forHTTPHeaderField:@"Content-Type"];
        }

        NSData *plistData = [NSPropertyListSerialization dataWithPropertyList:parameters format:self.format options:self.writeOptions error:error];
        
        if (!plistData) {
            return nil;
        }
        
        [mutableRequest setHTTPBody:plistData];
    }

    return mutableRequest;
}
```

它通过 `NSPropertyListSerialization` 将参数转为了 plist 的形式。

## 总结

本篇讲了创建 `NSMutableURLRequest` 的过程。request 的 `content-type` 分为四种形式：

- application/x-www-form-urlencoded
- application/json
- application/x-plist
- multipart/form-data

前三种都是将参数以 `NSData` 的形式设置给 request 的 `HTTPBody`，最后一种则是通过 `NSInputStream` 设置给 request 的 `HTTPBodyStream` 。

第一种将参数以 `a=x&b=y&c=z` 的形式连接，需要对生成的字符串转码。

第二种将参数转为 json 再转为 NSData，直接设置给 request 就可以了。

第三种将参数通过 `NSPropertyListSerialization` 直接转为 NSData，设置给 request

第四种将参数以符合特定规则的 boundary 分割，可以传递文件。用 `AFHTTPBodyPart` 作为每一个参数的模型，`AFMultipartBodyStream` 作为输入流，包含所有参数，重写 `read:maxLength:` 方法，提供 buffer 给 request。`AFStreamingMultipartFormData` 作为 `AFMultipartBodyStream` 的管理者，将 `AFHTTPBodyPart` 设置给它，并且是它和 request 的桥梁。

其中只有第四种方式需要自己手动计算 `content-Length`，其他都由框架通过 `NSData` 自己获取。









