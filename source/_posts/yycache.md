title: YYCache 源码解析
date: 2018/11/30 14:07:12  
categories: iOS
tags: 

 - 源码解析
	
---



<!--more-->

## 基本使用

### 创建

创建 Cache 实例，之后的所有操作都是通过它。

```objc
YYCache *cache = [YYCache cacheWithName:@"cacheName"];
```

### 使用方法一览

之前创建的 `YYCache *cache` 的实例方法。包含了读取、写入，移除的所有相关方法

```objc
//是否包含某缓存，无回调
- (BOOL)containsObjectForKey:(NSString *)key;
//是否包含某缓存，有回调
- (void)containsObjectForKey:(NSString *)key withBlock:(nullable void(^)(NSString *key, BOOL contains))block;
c
//获取缓存对象，无回调
- (nullable id<NSCoding>)objectForKey:(NSString *)key;
//获取缓存对象，有回调
- (void)objectForKey:(NSString *)key withBlock:(nullable void(^)(NSString *key, id<NSCoding> object))block;

//写入缓存对象，无回调
- (void)setObject:(nullable id<NSCoding>)object forKey:(NSString *)key;
//写入缓存对象，有回调
- (void)setObject:(nullable id<NSCoding>)object forKey:(NSString *)key withBlock:(nullable void(^)(void))block;

//移除某缓存，无回调
- (void)removeObjectForKey:(NSString *)key;
//移除某缓存，有回调
- (void)removeObjectForKey:(NSString *)key withBlock:(nullable void(^)(NSString *key))block;

//移除所有缓存，无回调
- (void)removeAllObjects;
//移除所有缓存，有回调
- (void)removeAllObjectsWithBlock:(void(^)(void))block;
//移除所有缓存，有进度和完成的回调
- (void)removeAllObjectsWithProgressBlock:(nullable void(^)(int removedCount, int totalCount))progress
                                 endBlock:(nullable void(^)(BOOL error))end;
```

### 结构

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/yycache.png?raw=true)

- `YYCache`：最外层接口，调用了YYMemoryCache与YYDiskCache的相关方法。

- `YYMemoryCache`：负责内存缓存

- `_YYLinkedMap`：内存缓存进行 LRU 所使用的双向链表类

- `_YYLinkedMapNode`：是 `_YYLinkedMap` 使用的节点类。

- `YYDiskCache`：负责磁盘缓存

- `YYKVStorage`：`YYDiskCache` 的实现类，用于管理磁盘缓存。

- `YYKVStorageItem`：`YYKVStorage` 内部用于封装某个缓存的类。

## 源码分析

### YYCache

#### 初始化

```objc
- (instancetype)initWithName:(NSString *)name {
    if (name.length == 0) return nil;
    // 根据传入的 cache 名字创建对应路径
    NSString *cacheFolder = [NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) firstObject];
    NSString *path = [cacheFolder stringByAppendingPathComponent:name];
    return [self initWithPath:path];
}

- (instancetype)initWithPath:(NSString *)path {
    if (path.length == 0) return nil;
    // 根据路径创建磁盘缓存
    YYDiskCache *diskCache = [[YYDiskCache alloc] initWithPath:path];
    if (!diskCache) return nil;
    NSString *name = [path lastPathComponent];
    // 新建一个内存缓存
    YYMemoryCache *memoryCache = [YYMemoryCache new];
    memoryCache.name = name;
    
    self = [super init];
    _name = name;
    _diskCache = diskCache;
    _memoryCache = memoryCache;
    return self;
}
```

#### 是否存在

```objc
- (BOOL)containsObjectForKey:(NSString *)key {
    // 先判断是否在内存缓存中，如果不在那么判断是否在磁盘缓存中
    return [_memoryCache containsObjectForKey:key] || [_diskCache containsObjectForKey:key];
}
```

#### 获取缓存

```objc
- (id<NSCoding>)objectForKey:(NSString *)key {
    // 先从内存缓存获取
    id<NSCoding> object = [_memoryCache objectForKey:key];
    if (!object) {
        // 不在内存缓存中，那么从磁盘缓存中获取
        object = [_diskCache objectForKey:key];
        if (object) {
            // 如果存在于磁盘缓存中，那么存入内存缓存中
            [_memoryCache setObject:object forKey:key];
        }
    }
    return object;
}

```

#### 设置缓存

```objc
- (void)setObject:(id<NSCoding>)object forKey:(NSString *)key {
    // 存入内存缓存
    [_memoryCache setObject:object forKey:key];
    // 存入磁盘缓存
    [_diskCache setObject:object forKey:key];
}
```

#### 移除缓存

```objc
// 先移除内存缓存，再移除磁盘缓存
- (void)removeObjectForKey:(NSString *)key {
    [_memoryCache removeObjectForKey:key];
    [_diskCache removeObjectForKey:key];
}

- (void)removeAllObjects {
    [_memoryCache removeAllObjects];
    [_diskCache removeAllObjects];
}
```

### YYMemoryCache

#### 概览

##### 初始化

```objc
@implementation YYMemoryCache {
    // 🔐实例
    pthread_mutex_t _lock;
    // 内存缓存存放处
    _YYLinkedMap *_lru;
    // 自己创建的队列
    dispatch_queue_t _queue;
}

- (instancetype)init {
    self = super.init;
    // 创建锁
    pthread_mutex_init(&_lock, NULL);
    // 创建双向链表保存内存缓存数据
    _lru = [_YYLinkedMap new];
    // 自己创建的串行队列，主要用在 trim 的时候，保证在后台的操作按顺序执行
    _queue = dispatch_queue_create("com.ibireme.cache.memory", DISPATCH_QUEUE_SERIAL);
    
	... (省略清除数据的策略配置)
        
    // 开启定时清理
    [self _trimRecursively];
    return self;
}

- (void)dealloc {
	...
	// 由于内部缓存内部是双向链表，所以需要手动清空，以防内存泄漏
    [_lru removeAll];
    // 回收锁
    pthread_mutex_destroy(&_lock);
}
```

##### 方法

`YYMemoryCache` 包含两类方法，一类是操作内存缓存的方法，一类是缓存自动清理的方法：

```objc
#pragma mark - Access Methods
// =========== 操作缓存数据接口 =========== 
//是否包含某个缓存
- (BOOL)containsObjectForKey:(id)key;

//获取缓存对象
- (nullable id)objectForKey:(id)key;

//写入缓存对象
- (void)setObject:(nullable id)object forKey:(id)key;

//写入缓存对象，并添加对应的开销
- (void)setObject:(nullable id)object forKey:(id)key withCost:(NSUInteger)cost;

//移除某缓存
- (void)removeObjectForKey:(id)key;

//移除所有缓存
- (void)removeAllObjects;

#pragma mark - Trim
// =========== 缓存清理接口 =========== 
//清理缓存到指定个数
- (void)trimToCount:(NSUInteger)count;

//清理缓存到指定开销
- (void)trimToCost:(NSUInteger)cost;

//清理缓存时间小于指定时间的缓存
- (void)trimToAge:(NSTimeInterval)age;
```

#### _YYLinkedMap

##### 概览

`_YYLinkedMap` 是保存内存缓存的实例。内置了一个字典和一个双向链表用于保存数据，两者分工不同。**字典是用来快速存取节点的，双向链表则是用来进行缓存淘汰的，双向链表有助于快速移动节点。**

```objc
@interface _YYLinkedMap : NSObject {
    @package
    CFMutableDictionaryRef _dic; 	// 用于存放节点
    NSUInteger _totalCost;   		//总开销
    NSUInteger _totalCount;  		//节点总数
    _YYLinkedMapNode *_head;            // 链表的头部结点
    _YYLinkedMapNode *_tail; 		// 链表的尾部节点
    BOOL _releaseOnMainThread; 	        //是否在主线程释放，默认为NO
    BOOL _releaseAsynchronously; 	//是否在子线程释放，默认为YES
}

//在链表头部插入某节点
- (void)insertNodeAtHead:(_YYLinkedMapNode *)node;

//将链表内部的某个节点移到链表头部
- (void)bringNodeToHead:(_YYLinkedMapNode *)node;

//移除某个节点
- (void)removeNode:(_YYLinkedMapNode *)node;

//移除链表的尾部节点并返回它
- (_YYLinkedMapNode *)removeTailNode;

//移除所有节点（默认在子线程操作）
- (void)removeAll;

@end
```

##### 方法实现

`_YYLinkedMap` 中的方法都是用来操作节点的，稍微熟悉数据结构的人都能看懂。这里不详细分析，只举两个例子。

**插入节点**

插入节点需要先把键值对存到字典中，然后将节点插入链表的头部，是一个操作链表的基本操作。

```objc
- (void)insertNodeAtHead:(_YYLinkedMapNode *)node {
    // 插入节点需要先把键值对保存到字典中
    CFDictionarySetValue(_dic, (__bridge const void *)(node->_key), (__bridge const void *)(node));
    //增加开销和总缓存数量
    _totalCost += node->_cost;
    _totalCount++;
    if (_head) {
        node->_next = _head;
        _head->_prev = node;
        _head = node;
    } else {
        _head = _tail = node;
    }
}
```

**移除节点**

移除节点要先把保存的键值对移除，然后减少缓存中的总开销。

```objc
- (void)removeNode:(_YYLinkedMapNode *)node {
    // 将节点的键值对移除
    CFDictionaryRemoveValue(_dic, (__bridge const void *)(node->_key));
    // 减少开销
    _totalCost -= node->_cost;
    _totalCount--;
    if (node->_next) node->_next->_prev = node->_prev;
    if (node->_prev) node->_prev->_next = node->_next;
    if (_head == node) _head = node->_next;
    if (_tail == node) _tail = node->_prev;
}
```

#### 缓存策略

`YYMemoryCache` 的操作类似于 `NSCache`，它们的不同之处在于：

- `YYMemoryCache` 使用 LRU(least-recently-used) 算法来清理使用频率较低的缓存；`NSCache` 的淘汰方式不确定
- `YYMemoryCache` 使用三个维度 count（缓存数量），cost（开销），age（距上一次的访问时间）控制缓存清除；`NSCache` 不确定
- `YYMemoryCache` 可以配置收到内存警告或者进入后台时自动清除缓存。

LRU 算法的思想就是优先保存最近用过的数据。将最近使用的节点放到链表的头部，并且优先淘汰链表尾部的数据。

#### 方法实现

##### 缓存相关方法

缓存相关方法比较类似，我们这里只举两个例子

**获取缓存对象**

```objc
- (id)objectForKey:(id)key {
    if (!key) return nil;
    pthread_mutex_lock(&_lock);
    _YYLinkedMapNode *node = CFDictionaryGetValue(_lru->_dic, (__bridge const void *)(key));
    // 获取到了缓存的对象就需要把节点移动到最前面，并且更新时间
    if (node) {
        node->_time = CACurrentMediaTime();
        [_lru bringNodeToHead:node];
    }
    pthread_mutex_unlock(&_lock);
    return node ? node->_value : nil;
}
```

获取对象以及对链表进行操作的时候注意要加锁

**设置缓存对象**

```objc
- (void)setObject:(id)object forKey:(id)key withCost:(NSUInteger)cost {
    if (!key) return;
    // 没有传 object，就直接移除
    if (!object) {
        [self removeObjectForKey:key];
        return;
    }
    // 加锁
    pthread_mutex_lock(&_lock);
    _YYLinkedMapNode *node = CFDictionaryGetValue(_lru->_dic, (__bridge const void *)(key));
    NSTimeInterval now = CACurrentMediaTime();
    if (node) {
        // 存在 node 就更新 node，并且把 node 移到链表最前面
        _lru->_totalCost -= node->_cost;
        _lru->_totalCost += cost;
        node->_cost = cost;
        node->_time = now;
        node->_value = object;
        [_lru bringNodeToHead:node];
    } else {
        // 不存在 node 就新建一个 node，然后存储
        node = [_YYLinkedMapNode new];
        node->_cost = cost;
        node->_time = now;
        node->_key = key;
        node->_value = object;
        [_lru insertNodeAtHead:node];
    }
    // 如果缓存总消耗超过了限制，那么异步淘汰末端缓存
    if (_lru->_totalCost > _costLimit) {
        dispatch_async(_queue, ^{
            [self trimToCost:_costLimit];
        });
    }
    // 如果缓存总量超过了限制，那么清楚最后一个
    if (_lru->_totalCount > _countLimit) {
        _YYLinkedMapNode *node = [_lru removeTailNode];
        // 在主线程或者子线程释放
        if (_lru->_releaseAsynchronously) {
            dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
            dispatch_async(queue, ^{
                [node class]; //hold and release in queue
            });
        } else if (_lru->_releaseOnMainThread && !pthread_main_np()) {
            dispatch_async(dispatch_get_main_queue(), ^{
                [node class]; //hold and release in queue
            });
        }
    }
    pthread_mutex_unlock(&_lock);
}
```

这里有一个关于淘汰的缓存再哪里释放的技巧，即最后在哪个线程调用，就在哪个线程释放。

##### 缓存清理相关方法

缓存清理包含两种情况。一种是主动添加缓存之后的主动触发，还有一种是周期性的后台触发。创建 `YYCache` 时调用的 `_trimRecursively` 方法就会触发周期性的后台触发：

```objc
//YYMemoryCache.m
- (instancetype)init{
    ...
    //开始定期清理
    [self _trimRecursively];
    ...
}


//递归清理，相隔时间为_autoTrimInterval，在初始化之后立即执行
- (void)_trimRecursively {
    __weak typeof(self) _self = self;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(_autoTrimInterval * NSEC_PER_SEC)),   
        dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0), ^{
        __strong typeof(_self) self = _self;
        if (!self) return;
        //在后台进行清理操作
        [self _trimInBackground];
        //调用自己，递归操作
        [self _trimRecursively];
    });
}

//清理所有不符合限制的缓存，顺序为：cost，count，age
- (void)_trimInBackground {
    dispatch_async(_queue, ^{
        [self _trimToCost:self->_costLimit];
        [self _trimToCount:self->_countLimit];
        [self _trimToAge:self->_ageLimit]; 
    });
}
```

缓存清理有三个维度包括 age, count, cost。它们的实现也非常相似，这里只列举其中一个说明：

**cost 清理**

```objc
- (void)_trimToCost:(NSUInteger)costLimit {
    BOOL finish = NO;
    // 无需清理的时候
    pthread_mutex_lock(&_lock);
    if (costLimit == 0) {
        [_lru removeAll];
        finish = YES;
    } else if (_lru->_totalCost <= costLimit) {
        finish = YES;
    }
    pthread_mutex_unlock(&_lock);
    if (finish) return;
    
    // 暂存需要清除的缓存对象数组
    NSMutableArray *holder = [NSMutableArray new];
    while (!finish) {
        // 尝试加锁，如果锁被占用，那么就空转 10ms，达到自旋锁的效果
        if (pthread_mutex_trylock(&_lock) == 0) {
            if (_lru->_totalCost > costLimit) {
                _YYLinkedMapNode *node = [_lru removeTailNode];
                if (node) [holder addObject:node];
            } else {
                finish = YES;
            }
            pthread_mutex_unlock(&_lock);
        } else {
            usleep(10 * 1000); //10 ms
        }
    }
    if (holder.count) {
        dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
        dispatch_async(queue, ^{
            [holder count]; // release in queue
        });
    }
}
```

优于 OSSpinLock 的优先级反转问题，不建议使用。因此，作者使用 tryLock 的方式尝试获得锁。

### YYDiscCache

`YYDiscCache` 和  `YYMemoryCache` 的操作非常相似。不同之处在于，`YYMemoryCache` 将结果保存在内存中的字典中，用双向链表体现操作时间的远近。而 `YYDiscCache` 将结果保存在数据库中，通过表中最近更新时间体现操作时间的远近。

#### 初始化

```objc
- (instancetype)initWithPath:(NSString *)path {
    // 默认保存在数据库中文件大小阈值为 1024*20
    return [self initWithPath:path inlineThreshold:1024 * 20]; // 20KB
}

- (instancetype)initWithPath:(NSString *)path
             inlineThreshold:(NSUInteger)threshold {
    self = [super init];
    if (!self) return nil;
    
	// 保存到全局的 DiscCache 中
    YYDiskCache *globalCache = _YYDiskCacheGetGlobal(path);
    if (globalCache) return globalCache;
    
    YYKVStorageType type;
	// 根据阈值来设置存储的类型
    if (threshold == 0) {
        type = YYKVStorageTypeFile;
    } else if (threshold == NSUIntegerMax) {
        type = YYKVStorageTypeSQLite;
    } else {
        type = YYKVStorageTypeMixed;
    }
    
	// 操作数据库的实际对象实例
    YYKVStorage *kv = [[YYKVStorage alloc] initWithPath:path type:type];
    if (!kv) return nil;
    _kv = kv
    // 初始化 semaphore 锁
    _lock = dispatch_semaphore_create(1);
    // 
    _queue = dispatch_queue_create("com.ibireme.cache.disk", DISPATCH_QUEUE_CONCURRENT);

	...

    [self _trimRecursively];
    _YYDiskCacheSetGlobal(self);
    
	...

    return self;
}
```

初始化方法。根据阈值来设置存储的 type。分为三种 type：

- YYKVStorageTypeFile: 阈值为0的时候，纯文件形式的存储。
- YYKVStorageTypeSQLite: 阈值为无穷大的时候，纯数据库的存储
- YYKVStorageTypeMixed: 阈值为特定值的时候，混合存储。大于阈值的时候用文件存储，小于阈值的时候使用数据库存储。一般情况下，都是数据库存储。

初始化的过程中，在初始化 `YYKVStorage` 的时候，开启了数据库。同时还创建了 `dispatch_semaphore` 形式的锁，以及全局处理队列。

#### 方法

缓存策略和内存缓存相似。同样的，各个磁盘缓存的方法与内存缓存的方法类似。所以这里还是只分析一个存储的方法。

`YYDiskCache` 中的方法，在传入要保存的键值对后，先将值归档为 `NSData`，然后判断其和阈值的大小，如果超过了阈值并且 type 不是只能使用数据库存储，那么生成一个文件名，之后就会以这个文件名存储。

```objc
// YYDiskCache.m
- (void)setObject:(id<NSCoding>)object forKey:(NSString *)key {
    if (!key) return;
    if (!object) {
        [self removeObjectForKey:key];
        return;
    }
    
    NSData *extendedData = [YYDiskCache getExtendedDataFromObject:object];
    NSData *value = nil;
    if (_customArchiveBlock) {
        value = _customArchiveBlock(object);
    } else {
        @try {
            // 通过归档方法，将要保存的对象转为 NSData 对象
            value = [NSKeyedArchiver archivedDataWithRootObject:object];
        }
        @catch (NSException *exception) {
            // nothing to do...
        }
    }
    if (!value) return;
    NSString *filename = nil;
    if (_kv.type != YYKVStorageTypeSQLite) {
        // 要保存的 value 大于阈值的之后，要创建一个 fileName，如果有 fileName，那么就文件保存，如果没有，就 sql 保存
        if (value.length > _inlineThreshold) {
            filename = [self _filenameForKey:key];
        }
    }
    
    Lock();
    [_kv saveItemWithKey:key value:value filename:filename extendedData:extendedData];
    Unlock();
}
```

生成文件名的方式是通过把 key 通过 MD5 转为 16 位的 hash 值。

```objc
- (NSString *)_filenameForKey:(NSString *)key {
    NSString *filename = nil;
    if (_customFileNameBlock) filename = _customFileNameBlock(key);
    if (!filename) filename = _YYNSStringMD5(key);
    return filename;
}

/// String's md5 hash.
static NSString *_YYNSStringMD5(NSString *string) {
    if (!string) return nil;
    NSData *data = [string dataUsingEncoding:NSUTF8StringEncoding];
    unsigned char result[CC_MD5_DIGEST_LENGTH];
    CC_MD5(data.bytes, (CC_LONG)data.length, result);
    return [NSString stringWithFormat:
                @"%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x",
                result[0],  result[1],  result[2],  result[3],
                result[4],  result[5],  result[6],  result[7],
                result[8],  result[9],  result[10], result[11],
                result[12], result[13], result[14], result[15]
            ];
}
```

真正存储的方法是在 `YYStorage` 中。如果有文件名，那么优先存文件，如果不成功再尝试存入数据库。

```objc
// YYStorage.m
// 保存键值对到数据库或者本地文件的实际方法
- (BOOL)saveItemWithKey:(NSString *)key value:(NSData *)value filename:(NSString *)filename extendedData:(NSData *)extendedData {
    if (key.length == 0 || value.length == 0) return NO;
    // 如果 type 为文件存储，但是确没有 filename，那么返回失败
    if (_type == YYKVStorageTypeFile && filename.length == 0) {
        return NO;
    }
    
    if (filename.length) {
        // 如果文件名不为空，那么把 data 存入文件中
        if (![self _fileWriteWithName:filename data:value]) {
            return NO;
        }
        if (![self _dbSaveWithKey:key value:value fileName:filename extendedData:extendedData]) {
            [self _fileDeleteWithName:filename];
            return NO;
        }
        return YES;
    } else {
        if (_type != YYKVStorageTypeSQLite) {
            NSString *filename = [self _dbGetFilenameWithKey:key];
            if (filename) {
                [self _fileDeleteWithName:filename];
            }
        }
        return [self _dbSaveWithKey:key value:value fileName:nil extendedData:extendedData];
    }
}
```

```objc
// 将data写到文件
- (BOOL)_fileWriteWithName:(NSString *)filename data:(NSData *)data {
    NSString *path = [_dataPath stringByAppendingPathComponent:filename];
    return [data writeToFile:path atomically:NO];
}
```

存入数据库中调用的方法中，先在缓存中查找 `sqlite3_stmt`。然后绑定参数，执行 sql。

```objc
- (BOOL)_dbSaveWithKey:(NSString *)key value:(NSData *)value fileName:(NSString *)fileName extendedData:(NSData *)extendedData {
    NSString *sql = @"insert or replace into manifest (key, filename, size, inline_data, modification_time, last_access_time, extended_data) values (?1, ?2, ?3, ?4, ?5, ?6, ?7);";
    sqlite3_stmt *stmt = [self _dbPrepareStmt:sql];
    if (!stmt) return NO;
    
    int timestamp = (int)time(NULL);
    sqlite3_bind_text(stmt, 1, key.UTF8String, -1, NULL);
    sqlite3_bind_text(stmt, 2, fileName.UTF8String, -1, NULL);
    sqlite3_bind_int(stmt, 3, (int)value.length);
    if (fileName.length == 0) {
        sqlite3_bind_blob(stmt, 4, value.bytes, (int)value.length, 0);
    } else {
        sqlite3_bind_blob(stmt, 4, NULL, 0, 0);
    }
    sqlite3_bind_int(stmt, 5, timestamp);
    sqlite3_bind_int(stmt, 6, timestamp);
    sqlite3_bind_blob(stmt, 7, extendedData.bytes, (int)extendedData.length, 0);
    
    int result = sqlite3_step(stmt);
    if (result != SQLITE_DONE) {
        if (_errorLogsEnabled) NSLog(@"%s line:%d sqlite insert error (%d): %s", __FUNCTION__, __LINE__, result, sqlite3_errmsg(_db));
        return NO;
    }
    return YES;
}

- (sqlite3_stmt *)_dbPrepareStmt:(NSString *)sql {
    if (![self _dbCheck] || sql.length == 0 || !_dbStmtCache) return NULL;
    sqlite3_stmt *stmt = (sqlite3_stmt *)CFDictionaryGetValue(_dbStmtCache, (__bridge const void *)(sql));
    if (!stmt) {
        int result = sqlite3_prepare_v2(_db, sql.UTF8String, -1, &stmt, NULL);
        if (result != SQLITE_OK) {
            if (_errorLogsEnabled) NSLog(@"%s line:%d sqlite stmt prepare error (%d): %s", __FUNCTION__, __LINE__, result, sqlite3_errmsg(_db));
            return NULL;
        }
        CFDictionarySetValue(_dbStmtCache, (__bridge const void *)(sql), stmt);
    } else {
        sqlite3_reset(stmt);
    }
    return stmt;
}
```

## 几个问题

### 为什么 `YYMemoryCache` 使用互斥锁 `pthread_mutex`，而 `YYDiskCache` 使用信号量 `dispatch_semaphore`

作者通过 `pthread_mutex` 的 `pthread_mutex_trylock()` 和忙等 `usleep()` 来替代 `OSSpinLock`，这是因为使用互斥锁，没有拿到锁的线程会被挂起。当释放锁激活其他线程的时候，就唤醒挂起的线程。需要上下文切换，信号发送等开销，效率低于自旋锁。所以使用这种方式替代自旋锁。

因为 `YYDiskCache` 在写入比较大的缓存时，可能会有比较长的等待时间，而`dispatch_semaphore` 在这个时候是不消耗CPU资源的，所以比较适合。

### 为什么 `YYMemoryCache` 使用双向链表

双向链表的优势在于在头尾插入或者删除的时候时间复杂度最低。而缓存的存入和清理需要频繁的头部插入，尾部删除。所以使用双向链表最为合适。

### `YYMemoryCache` 内容使用什么数据结构保存数据的

通过一个双向链表和一个字典保存。字典的作用是 O(1) 的时间内找到数据，链表的作用是为了进行 LRU 的缓存清理。

### 如何执行定时任务来定时清理缓存

作者使用的是 **`dispatch_after` 的延时 + 递归**的方式进行定时执行操作，而不是使用 NSTimer 的 repeat

### 子线程释放

一个对象最后在哪个线程中使用，那么最终就会在哪个线程的 runloop 中销毁。

```objc
_YYLinkedMapNode *node = [_lru removeTailNode];
if (_lru->_releaseAsynchronously) {
    dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
    dispatch_async(queue, ^{
        [node class]; //hold and release in queue
    });
} else if (_lru->_releaseOnMainThread && !pthread_main_np()) {
    dispatch_async(dispatch_get_main_queue(), ^{
        [node class]; //hold and release in queue
    });
}
```

对象的销毁虽然消耗资源不多，但累积起来也是不容忽视的。通常当容器类持有大量对象时，其销毁时的资源消耗就非常明显。同样的，如果对象可以放到后台线程去释放，那就挪到后台线程去。这里有个小 Tips：把对象捕获到 block 中，然后扔到后台队列去随便发送个消息以避免编译器警告，就可以让对象在后台线程销毁了。





















