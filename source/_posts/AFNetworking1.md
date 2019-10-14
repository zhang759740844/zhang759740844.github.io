title: AFNetworking 源码解析 - 网络状态与安全策略
date: 2019/1/5 14:07:12  
categories: iOS
tags: 

 - 学习笔记
 - 源码解析
---

到了 AFNetworking 的最后一篇了。网络状态变更和安全策略是比较独立的两块东西。因此把他们放在最后看完。

<!--more-->

## 网络状态

`AFNetworkReachabilityManager`类提供了对于网络状态变化的监听。AF 预定义了四种网络状态：

```objc
typedef NS_ENUM(NSInteger, AFNetworkReachabilityStatus) {
    /// 未知
    AFNetworkReachabilityStatusUnknown          = -1,
    /// 无网络
    AFNetworkReachabilityStatusNotReachable     = 0,
    /// WWAN 手机自带网络
    AFNetworkReachabilityStatusReachableViaWWAN = 1,
    /// Wifi
    AFNetworkReachabilityStatusReachableViaWiFi = 2,
};
```

### 设置监听

网络状态的监听是一个逻辑很清晰的过程。对于网络状态的改变，我们必须要通过系统的 API 进行。那么在 `AFNetworkReachabilityStatusForFlags` 中就提供了这样的一个方法实施监听：

```objc
- (void)startMonitoring {
    [self stopMonitoring];

    /// 没有创建 networkReachability 实例，直接返回
    if (!self.networkReachability) {
        return;
    }

    /// 网络监听的回调
    __weak __typeof(self)weakSelf = self;
    AFNetworkReachabilityStatusBlock callback = ^(AFNetworkReachabilityStatus status) {
        __strong __typeof(weakSelf)strongSelf = weakSelf;

        strongSelf.networkReachabilityStatus = status;
        if (strongSelf.networkReachabilityStatusBlock) {
            strongSelf.networkReachabilityStatusBlock(status);
        }

    };

    /// 网络监听上下文，注册网络改变的回调
    SCNetworkReachabilityContext context = {0, (__bridge void *)callback, AFNetworkReachabilityRetainCallback, AFNetworkReachabilityReleaseCallback, NULL};
    SCNetworkReachabilitySetCallback(self.networkReachability, AFNetworkReachabilityCallback, &context);
    SCNetworkReachabilityScheduleWithRunLoop(self.networkReachability, CFRunLoopGetMain(), kCFRunLoopCommonModes);

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0),^{
        SCNetworkReachabilityFlags flags;
        if (SCNetworkReachabilityGetFlags(self.networkReachability, &flags)) {
            AFPostReachabilityStatusChange(flags, callback);
        }
    });
}
```

可以看到，通过系统 API 创建了一个上下文 `SCNetworkReachabilityContext` 实例，一个监听回调的 `AFNetworkReachabilityStatusBlock` 实例，以及一个句柄 `networkReachability`。既然是句柄，那么就可以通过它来结束监听：

```objc
- (void)stopMonitoring {
    if (!self.networkReachability) {
        return;
    }

    SCNetworkReachabilityUnscheduleFromRunLoop(self.networkReachability, CFRunLoopGetMain(), kCFRunLoopCommonModes);
}
```

`networkReachability` 在初始化监听类对象的时候创建：

```objc
+ (instancetype)sharedManager {
    static AFNetworkReachabilityManager *_sharedManager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedManager = [self manager];
    });

    return _sharedManager;
}

+ (instancetype)managerForDomain:(NSString *)domain {
    /// 创建 SCNetworkReachabilityRef 对象
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithName(kCFAllocatorDefault, [domain UTF8String]);

    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];
    
    CFRelease(reachability);

    return manager;
}

+ (instancetype)managerForAddress:(const void *)address {
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithAddress(kCFAllocatorDefault, (const struct sockaddr *)address);
    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];

    CFRelease(reachability);
    
    return manager;
}

+ (instancetype)manager
{
#if (defined(__IPHONE_OS_VERSION_MIN_REQUIRED) && __IPHONE_OS_VERSION_MIN_REQUIRED >= 90000) || (defined(__MAC_OS_X_VERSION_MIN_REQUIRED) && __MAC_OS_X_VERSION_MIN_REQUIRED >= 101100)
    struct sockaddr_in6 address;
    bzero(&address, sizeof(address));
    address.sin6_len = sizeof(address);
    address.sin6_family = AF_INET6;
#else
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_len = sizeof(address);
    address.sin_family = AF_INET;
#endif
    return [self managerForAddress:&address];
}
```

初始化方法比较长，主要分为了两种方式监听，一种是监听域名，一种是监听 socket。同时通过系统 API 创建 `networkReachability`。

### 监听回调

设置好监听那么就设置监听回调以处理网络状态变化信息咯。我们来查看上文中注册的 `AFNetworkReachabilityCallback` 回调是怎样的：

```objc
static void AFNetworkReachabilityCallback(SCNetworkReachabilityRef __unused target, SCNetworkReachabilityFlags flags, void *info) {
    AFPostReachabilityStatusChange(flags, (__bridge AFNetworkReachabilityStatusBlock)info);
}

static void AFPostReachabilityStatusChange(SCNetworkReachabilityFlags flags, AFNetworkReachabilityStatusBlock block) {
    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusForFlags(flags);
    dispatch_async(dispatch_get_main_queue(), ^{
        /// 有回调函数调用回调函数
        if (block) {
            block(status);
        }
        /// 发送通知
        NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
        NSDictionary *userInfo = @{ AFNetworkingReachabilityNotificationStatusItem: @(status) };
        [notificationCenter postNotificationName:AFNetworkingReachabilityDidChangeNotification object:nil userInfo:userInfo];
    });
}
```

当网络状态变更的时候，通过 `AFNetworkReachabilityStatusForFlags()` 方法进行基本状况的处理：

```objc
#pragma mark - 根据SCNetworkReachabilityFlags这个网络标记来转换成我们在开发中经常使用的网络状态
static AFNetworkReachabilityStatus AFNetworkReachabilityStatusForFlags(SCNetworkReachabilityFlags flags) {
    /// 能否到达
    BOOL isReachable = ((flags & kSCNetworkReachabilityFlagsReachable) != 0);
    /// 联网之前需要建立连接
    BOOL needsConnection = ((flags & kSCNetworkReachabilityFlagsConnectionRequired) != 0);
    /// 是否可以自动连接
    BOOL canConnectionAutomatically = (((flags & kSCNetworkReachabilityFlagsConnectionOnDemand ) != 0) || ((flags & kSCNetworkReachabilityFlagsConnectionOnTraffic) != 0));
    /// 是否可以在不需要用户手动设置的前提下连接
    BOOL canConnectWithoutUserInteraction = (canConnectionAutomatically && (flags & kSCNetworkReachabilityFlagsInterventionRequired) == 0);
    /// 是否可以联网：1.能够到达 2.不需要建立连接或者不需要用户手动设置连接就能连接到网络
    BOOL isNetworkReachable = (isReachable && (!needsConnection || canConnectWithoutUserInteraction));

    /// 设置网络状态
    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusUnknown;
    if (isNetworkReachable == NO) {
        status = AFNetworkReachabilityStatusNotReachable;
    }
#if	TARGET_OS_IPHONE
    else if ((flags & kSCNetworkReachabilityFlagsIsWWAN) != 0) {
        status = AFNetworkReachabilityStatusReachableViaWWAN;
    }
#endif
    else {
        status = AFNetworkReachabilityStatusReachableViaWiFi;
    }

    return status;
}
```

处理完成后的结果，可以通过两种方式拿到。一种是通过回调函数，这个回调函数就是上文中在 `startMonitoring()` 中创建 `SCNetworkReachabilityContext` 传入的 callback，可以自己将 block 设置给监听类的单例对象。还有一种就是通过通知，提前注册以下两种通知，可以获取到改变时间以及改变的情况：

```objc
#pragma mark - 通知的字符串
/// 网络环境改变时候接收通知
NSString * const AFNetworkingReachabilityDidChangeNotification = @"com.alamofire.networking.reachability.change";
/// 改变后接收通知，携带一个信息
NSString * const AFNetworkingReachabilityNotificationStatusItem = @"AFNetworkingReachabilityNotificationStatusItem";
```

以上就是网络监听相关的方法。

## 安全策略

`AFSecurityPolicy` 类提供了对于 HTTPS 通信过程中证书的校验，可以有效防止中间人攻击。它的校验模式分一下三种：

```objc
#pragma mark - 安全级别
typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
    /// 信任所有证书
    AFSSLPinningModeNone,
    /// 校验证书的公钥
    AFSSLPinningModePublicKey,
    /// 校验整个证书
    AFSSLPinningModeCertificate,
};

```

### 创建 AFSecurityPolicy

#### 初始化

`AFSecurityPolicy` 并不是一个单例对象，每个 `AFEscurityPolicy` 实例都可以保存自己的验证模式，校验证书集合，以及一些配置属性。它的初始化方法如下：

```objc
+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode {
    return [self policyWithPinningMode:pinningMode withPinnedCertificates:[self defaultPinnedCertificates]];
}

+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode withPinnedCertificates:(NSSet *)pinnedCertificates {
    AFSecurityPolicy *securityPolicy = [[self alloc] init];
    securityPolicy.SSLPinningMode = pinningMode;

    [securityPolicy setPinnedCertificates:pinnedCertificates];

    return securityPolicy;
}

- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.validatesDomainName = YES;

    return self;
}
```

#### 设置校验证书

上面的代码中，通过 `[self defaultPinnedCertificates]` 拿到证书列表，并且再 set 给 `AFSecurityPolicy` 的过程如下：

```objc
#pragma mark - 取出 bundler 中的证书
+ (NSSet *)certificatesInBundle:(NSBundle *)bundle {
    /// bundle 中以 .cer 结尾的所有路径
    NSArray *paths = [bundle pathsForResourcesOfType:@"cer" inDirectory:@"."];

    NSMutableSet *certificates = [NSMutableSet setWithCapacity:[paths count]];
    for (NSString *path in paths) {
        NSData *certificateData = [NSData dataWithContentsOfFile:path];
        [certificates addObject:certificateData];
    }

    return [NSSet setWithSet:certificates];
}

#pragma mark - 获取当前 bundle 下的证书
+ (NSSet *)defaultPinnedCertificates {
    static NSSet *_defaultPinnedCertificates = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        NSBundle *bundle = [NSBundle bundleForClass:[self class]];
        _defaultPinnedCertificates = [self certificatesInBundle:bundle];
    });

    return _defaultPinnedCertificates;
}
```

获取证书的过程就是从 bundle 中加载 `.cer` 文件的过程。设置证书给实例的时候做了一个操作，会到证书中获取到公钥信息：

```objc
#pragma mark - 在证书中获取公钥
static id AFPublicKeyForCertificate(NSData *certificate) {
    id allowedPublicKey = nil;
    SecCertificateRef allowedCertificate;
    SecPolicyRef policy = nil;
    SecTrustRef allowedTrust = nil;
    SecTrustResultType result;

    /// 根据二进制的certificate生成SecCertificateRef类型的证书
    allowedCertificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificate);
    __Require_Quiet(allowedCertificate != NULL, _out);

    /// 新建 policy 为 x.509
    policy = SecPolicyCreateBasicX509();
    /// 设置信任的证书
    __Require_noErr_Quiet(SecTrustCreateWithCertificates(allowedCertificate, policy, &allowedTrust), _out);
    __Require_noErr_Quiet(SecTrustEvaluate(allowedTrust, &result), _out);

    /// 在SecTrustRef对象中取出公钥
    allowedPublicKey = (__bridge_transfer id)SecTrustCopyPublicKey(allowedTrust);

_out:
    if (allowedTrust) {
        CFRelease(allowedTrust);
    }

    if (policy) {
        CFRelease(policy);
    }

    if (allowedCertificate) {
        CFRelease(allowedCertificate);
    }

    return allowedPublicKey;
}

- (void)setPinnedCertificates:(NSSet *)pinnedCertificates {
    _pinnedCertificates = pinnedCertificates;

    if (self.pinnedCertificates) {
        NSMutableSet *mutablePinnedPublicKeys = [NSMutableSet setWithCapacity:[self.pinnedCertificates count]];
        /// 从证书中取出公钥
        for (NSData *certificate in self.pinnedCertificates) {
            id publicKey = AFPublicKeyForCertificate(certificate);
            if (!publicKey) {
                continue;
            }
            [mutablePinnedPublicKeys addObject:publicKey];
        }
        self.pinnedPublicKeys = [NSSet setWithSet:mutablePinnedPublicKeys];
    } else {
        self.pinnedPublicKeys = nil;
    }
}
```

### 执行校验

```objc
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain
{
    /// 自签名证书需要通过验证
    if (domain && self.allowInvalidCertificates && self.validatesDomainName && (self.SSLPinningMode == AFSSLPinningModeNone || [self.pinnedCertificates count] == 0)) {
        NSLog(@"In order to validate a domain name for self signed certificates, you MUST use pinning.");
        return NO;
    }

    NSMutableArray *policies = [NSMutableArray array];
    if (self.validatesDomainName) {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
    } else {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
    }

    /// 设置要校验的域名
    SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);

    /// 在安全级别为不校验的情况，只要通过了CA的校验或者用户允许自签名证书，那么就是成功的
    if (self.SSLPinningMode == AFSSLPinningModeNone) {
        return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
    /// 如果安全级别是校验证书并且没有通过CA的校验且不允许自签名证书，那么直接返回 NO
    } else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
        return NO;
    }

    /// 有安全校验，且证书也是合法的情况下
    switch (self.SSLPinningMode) {
        case AFSSLPinningModeNone:
        default:
            return NO;
        /// 如果是校验证书的级别
        case AFSSLPinningModeCertificate: {
            NSMutableArray *pinnedCertificates = [NSMutableArray array];
            for (NSData *certificateData in self.pinnedCertificates) {
                [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
            }
            
            /// 将本地读取的证书设置为serverTrust的锚点证书
            /// 锚点证书，通过SecTrustSetAnchorCertificates设置了参与校验锚点证书之后，假如验证的数字证书是这个锚点证书的子节点，即验证的数字证书是由锚点证书对应CA或子CA签发的，或是该证书本身，则信任该证书），具体就是调用SecTrustEvaluate来验证。
            SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);

            if (!AFServerTrustIsValid(serverTrust)) {
                return NO;
            }

            // obtain the chain after being validated, which *should* contain the pinned certificate in the last position (if it's the Root CA)
          	/// 验证整个证书链
            NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);
            
            /// 比较服务端证书是否包含在本地证书内
            for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
                if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
                    return YES;
                }
            }
            
            return NO;
        }
        /// 如果只校验公钥，那么只需要比较公钥即可。
        case AFSSLPinningModePublicKey: {
            NSUInteger trustedPublicKeyCount = 0;
            NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);

            for (id trustChainPublicKey in publicKeys) {
                for (id pinnedPublicKey in self.pinnedPublicKeys) {
                    if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
                        trustedPublicKeyCount += 1;
                    }
                }
            }
            return trustedPublicKeyCount > 0;
        }
    }
    
    return NO;
}
```

以上是校验的逻辑，其中涉及到好几个方法：

```objc
#pragma mark - 返回服务器是否可信
static BOOL AFServerTrustIsValid(SecTrustRef serverTrust) {
    BOOL isValid = NO;
    SecTrustResultType result;
    /// 校验是否是可信的证书
    __Require_noErr_Quiet(SecTrustEvaluate(serverTrust, &result), _out);

    // kSecTrustResultUnspecified 证书验证成功，但是用户没有明确指出信任此证书。这是最常见的返回值。
    // kSecTrustResultProceed 用户选择信任此证书。
    isValid = (result == kSecTrustResultUnspecified || result == kSecTrustResultProceed);

_out:
    return isValid;
}

#pragma mark - 取出服务器证书的信任链
static NSArray * AFCertificateTrustChainForServerTrust(SecTrustRef serverTrust) {
    CFIndex certificateCount = SecTrustGetCertificateCount(serverTrust);
    NSMutableArray *trustChain = [NSMutableArray arrayWithCapacity:(NSUInteger)certificateCount];

    for (CFIndex i = 0; i < certificateCount; i++) {
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);
        [trustChain addObject:(__bridge_transfer NSData *)SecCertificateCopyData(certificate)];
    }

    return [NSArray arrayWithArray:trustChain];
}

#pragma mark - 取出服务器返回证书的信任链上的所有公钥放到数组中
static NSArray * AFPublicKeyTrustChainForServerTrust(SecTrustRef serverTrust) {
    SecPolicyRef policy = SecPolicyCreateBasicX509();
    CFIndex certificateCount = SecTrustGetCertificateCount(serverTrust);
    NSMutableArray *trustChain = [NSMutableArray arrayWithCapacity:(NSUInteger)certificateCount];
    for (CFIndex i = 0; i < certificateCount; i++) {
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);

        SecCertificateRef someCertificates[] = {certificate};
        CFArrayRef certificates = CFArrayCreate(NULL, (const void **)someCertificates, 1, NULL);

        SecTrustRef trust;
        __Require_noErr_Quiet(SecTrustCreateWithCertificates(certificates, policy, &trust), _out);

        SecTrustResultType result;
        __Require_noErr_Quiet(SecTrustEvaluate(trust, &result), _out);

        [trustChain addObject:(__bridge_transfer id)SecTrustCopyPublicKey(trust)];

    _out:
        if (trust) {
            CFRelease(trust);
        }

        if (certificates) {
            CFRelease(certificates);
        }

        continue;
    }
    CFRelease(policy);

    return [NSArray arrayWithArray:trustChain];
}
```

总的概括一下就是：

- 如果只验证证书的合法性的话，就直接通过苹果预置的根证书校验。
- 如果要校验证书是否是特定的证书，就要
  - 把服务端返回的证书的签发证书放在 native 中
  - 把 native 中的证书设置为服务端返回证书的锚点证书
  - 取出服务端的证书的信任链上的所有证书并和 native 中的证书做比较。如果相等就说明可以用 native 中保存的证书对服务端的证书解密，即是要信任的证书。如果没有，就说明收到的证书不是服务端签发的，不可信
- 如果要校验证书中的公钥，则是在上一步的基础上，取出证书中的公钥信息，进行比较。













