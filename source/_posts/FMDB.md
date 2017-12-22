title: FMDB 及其封装框架的实现过程
date: 2017/11/30 10:07:12  
categories: iOS
tags:
	- 学习笔记
---

iOS 上的数据库用的不多，看一看主流的 FMDB 以及其封装框架是怎么实现的。

<!--more-->

## FMDB

### 基本使用

FMDB 是 sqlite 的简单封装，主要用来执行 sql 语句，并取出数据。主要有三个类：

- `FMDatabase` : FMDB 最重要的类，用来保存 database 的实例，并对这个 db 执行 sql 语句。
- `FMResultSet`：保存了 sql 语句的执行结果。
- `FMDatabaseQueue`：在多线程环境下执行 sql。

下面这个例子就是 FMDB 的基本使用过程。

```objc
NSString* dir = [NSSearchPathForDirectoriesInDomains( NSDocumentDirectory, NSUserDomainMask, YES) lastObject];

NSString* dbPath = [dir stringByAppendingPathComponent:@"district.sqlite"];

FMDatabase* db = [FMDatabase databaseWithPath:dbPath];

[db open];

FMResultSet *rs = [db executeQuery: @"select * from country"];

while ([rs next]) {
    NSLog(@"%@ %@",
        [rs stringForColumn:@"city"], 
        [rs stringForColumn:@"province"]);
}

[db close];
```

### 源码解析

#### 初始化 FMDatabase

初始化使用的是 `+ [FMDatabase databaseWithPath:@"path"]` 方法，它会在内部调用 `initWithPath:` 方法。它其实就是创建了一个 `FMDatabase` 的实例。

```objc
- (instancetype)initWithPath:(NSString*)aPath {
    ...
      
    self = [super init];
    
    if (self) {
        _databasePath               = [aPath copy];
        _openResultSets             = [[NSMutableSet alloc] init];
        _db                         = nil;
        _logsErrors                 = YES;
        _crashOnErrors              = NO;
        _maxBusyRetryTimeInterval   = 2;
    }
    
    return self;
}
```

要求输入一个数据库的路径，并保存在 `_databasePath` 中。这一步里还没有打开 db，所以 `_db` 还是 nil。

#### 打开 db

`- [FMDatabase open]` 方法打开了 db。

```objc
- (BOOL)open {
	...
    
    int err = sqlite3_open([self sqlitePath], (sqlite3**)&_db );
    if(err != SQLITE_OK) {
        NSLog(@"error opening!: %d", err);
        return NO;
    }
    
    if (_maxBusyRetryTimeInterval > 0.0) {
        // set the handler
        [self setMaxBusyRetryTimeInterval:_maxBusyRetryTimeInterval];
    }
    
    return YES;
}
```

如果已经打开了就直接返回。否则调用 sqlite 提供的的 `sqlite3_open()` 方法，获得了打开的数据库，保存在 `_db` 属性中。最后还有一个 `setMaxBusyRetryTimeInterval:` 的方法，这个方法的主要目的就是当其他线程正在使用数据库的时候，设置一个当前线程的等待时间：

```objc
- (void)setMaxBusyRetryTimeInterval:(NSTimeInterval)timeout {
    
    _maxBusyRetryTimeInterval = timeout;
    
	...
      
    if (timeout > 0) {
        sqlite3_busy_handler(_db, &FMDBDatabaseBusyHandler, (__bridge void *)(self));
    }
    else {
        // turn it off otherwise
        sqlite3_busy_handler(_db, nil, nil);
    }
}
```

它是通过 sqlite 提供的 `sqlite3_busy_handler` 完成的，在这里的意思就是当 `_db` 正忙的时候，调用 `[self FMDBDatabaseBusyHandler]` 方法。这个 `FMDBDatabaseBusyHandler` 方法做了什么呢？

```objc
static int FMDBDatabaseBusyHandler(void *f, int count) {
    FMDatabase *self = (__bridge FMDatabase*)f;
    
    if (count == 0) {
        self->_startBusyRetryTime = [NSDate timeIntervalSinceReferenceDate];
        return 1;
    }
    
    NSTimeInterval delta = [NSDate timeIntervalSinceReferenceDate] - (self->_startBusyRetryTime);
    
    if (delta < [self maxBusyRetryTimeInterval]) {
        int requestedSleepInMillseconds = (int) arc4random_uniform(50) + 50;
        int actualSleepInMilliseconds = sqlite3_sleep(requestedSleepInMillseconds);
        if (actualSleepInMilliseconds != requestedSleepInMillseconds) {
            NSLog(@"WARNING: Requested sleep of %i milliseconds, but SQLite returned %i. Maybe SQLite wasn't built with HAVE_USLEEP=1?", requestedSleepInMillseconds, actualSleepInMilliseconds);
        }
        return 1;
    }
    
    return 0;
}
```

这个方法做的就是不停地通过 `sqlite3_sleep()`，让当前线程 sleep。 直到设置的最大等待时间到来。

#### 创建 select sql

这一节主要讲如何创建 select 的 sql，select 会从数据库中获取数据，所以需要创建一个专门的类用来拿数据，也就是下面的的 `FMResultSet`

##### 创建 sqlite3_stmt

所有 sql 语句都会被转化为 `sqlite3_stmt` 类型。由于这一过程比较耗时，所以一般将转化好的 `sqlite3_stmt` 保存到 `_cachedStatements` 字典中，以便相同 sql 反复使用：

```objc
- (FMResultSet *)executeQuery:(NSString *)sql withArgumentsInArray:(NSArray*)arrayArgs orDictionary:(NSDictionary *)dictionaryArgs orVAList:(va_list)args {
    
  	...
    
    int rc                  = 0x00;
    sqlite3_stmt *pStmt     = 0x00;
    FMStatement *statement  = 0x00;
    FMResultSet *rs         = 0x00;
    
    if (_shouldCacheStatements) {
        statement = [self cachedStatementForQuery:sql];
        pStmt = statement ? [statement statement] : 0x00;
        [statement reset];
    }
    
    if (!pStmt) {
        
        rc = sqlite3_prepare_v2(_db, [sql UTF8String], -1, &pStmt, 0);
        
        if (SQLITE_OK != rc) {
			...
            
            sqlite3_finalize(pStmt);
            _isExecutingStatement = NO;
            return nil;
        }
    }
   
  	...
}
```

`cachedStatementForQuery:` 就是在字典中查找 sql 对应的 `sqlite3_stmt` 的方法，它返回的 `FMStatement` 是 `sqlite3_stmt` 的封装。如果存在对应的 `sqlite3_stmt` 则通过 `reset` 方法调用 `sqlite3_reset()` 重置这个 `sqlite3_stmt`。

如果不存在呢，就是使用 `sqlite3_prepare_v2()`。在创建不成功的情况下，通过 `sqlite3_finalize()` 释放 `sqlite3_stmt` 数据结构。

##### 绑定参数

参数随着方法一起传了进来，一般参数有两种，一种是字典类型的，根据 sql 中的参数名插入，还有一种是数组型的，依次替换 sql 中的占位符：

```objc
- (FMResultSet *)executeQuery:(NSString *)sql withArgumentsInArray:(NSArray*)arrayArgs orDictionary:(NSDictionary *)dictionaryArgs orVAList:(va_list)args {
    
  	...
      
    id obj;
    int idx = 0;
    int queryCount = sqlite3_bind_parameter_count(pStmt); // pointed out by Dominic Yu (thanks!)
    
    // If dictionaryArgs is passed in, that means we are using sqlite's named parameter support
    if (dictionaryArgs) {
        
        for (NSString *dictionaryKey in [dictionaryArgs allKeys]) {
            
            // Prefix the key with a colon.
            NSString *parameterName = [[NSString alloc] initWithFormat:@":%@", dictionaryKey];
            
			...
            
            // Get the index for the parameter name.
            int namedIdx = sqlite3_bind_parameter_index(pStmt, [parameterName UTF8String]);
            
            FMDBRelease(parameterName);
            
            if (namedIdx > 0) {
                // Standard binding from here.
                [self bindObject:[dictionaryArgs objectForKey:dictionaryKey] toColumn:namedIdx inStatement:pStmt];
                // increment the binding count, so our check below works out
                idx++;
            }
            else {
                NSLog(@"Could not find index for %@", dictionaryKey);
            }
        }
    }
    else {
        
        while (idx < queryCount) {
            
            if (arrayArgs && idx < (int)[arrayArgs count]) {
                obj = [arrayArgs objectAtIndex:(NSUInteger)idx];
            }
            else if (args) {
                obj = va_arg(args, id);
            }
            else {
                //We ran out of arguments
                break;
            }
            
            if (_traceExecution) {
                if ([obj isKindOfClass:[NSData class]]) {
                    NSLog(@"data: %ld bytes", (unsigned long)[(NSData*)obj length]);
                }
                else {
                    NSLog(@"obj: %@", obj);
                }
            }
            
            idx++;
            
            [self bindObject:obj toColumn:idx inStatement:pStmt];
        }
    }
    
    if (idx != queryCount) {
        NSLog(@"Error: the bind count is not correct for the # of variables (executeQuery)");
        sqlite3_finalize(pStmt);
        _isExecutingStatement = NO;
        return nil;
    }
    
    FMDBRetain(statement); // to balance the release below
    
    if (!statement) {
        statement = [[FMStatement alloc] init];
        [statement setStatement:pStmt];
        
        if (_shouldCacheStatements && sql) {
            [self setCachedStatement:statement forQuery:sql];
        }
    }
    
  	...
}
```

首先通过 `sqlite3_bind_parameter_count()` 获得 sql 的参数个数。然后检查传入的是字典还是数组。如果是字典，遍历字典，通过 `sqlite3_bind_parameter_index()` 拿到键对应的参数索引，然后绑定；如果是数组就依次绑定到对应的列中。绑定也是使用的 sqlite 提供的针对不同类型的一系列绑定方法。

绑定完了后，将这个 `sqlite3_stmt` 暂存。由于是已经绑定了参数，所以可见前面 `sqlite3_reset()` 做的就是将参数清空。

##### 创建 FMResultSet 保存结果

现在 sql 已经创建完成，只欠执行了。FMDB 并没有立即执行，而是创建了一个 `FMResultSet` 对象，用来保存每次 sql 的结果。因为一个 db 可以执行多个 sql，所以就要创建多个 `FMResultSet` 。所以在创建 sql 的最后，还要创建一个 `FMResultSet`:

```objc
- (FMResultSet *)executeQuery:(NSString *)sql withArgumentsInArray:(NSArray*)arrayArgs orDictionary:(NSDictionary *)dictionaryArgs orVAList:(va_list)args {
  	...
      
    // the statement gets closed in rs's dealloc or [rs close];
    rs = [FMResultSet resultSetWithStatement:statement usingParentDatabase:self];
    [rs setQuery:sql];
  	
  	...
}
```

##### 执行 sql

执行 sql 在 `FMResultSet` 中进行：

```objc
- (BOOL)nextWithError:(NSError **)outErr {
    
    int rc = sqlite3_step([_statement statement]);
    
    if (SQLITE_BUSY == rc || SQLITE_LOCKED == rc) {
		...
    }
    else if (SQLITE_DONE == rc || SQLITE_ROW == rc) {
        // all is well, let's return.
    }
    else if (SQLITE_ERROR == rc) {
		...
    }
    else if (SQLITE_MISUSE == rc) {
		...
    }
    else {
		...
    }
    
    if (rc != SQLITE_ROW) {
        [self close];
    }
    
    return (rc == SQLITE_ROW);
}
```

其实上面的关键就是调用 sqlite 的 `sqlite3_step()` 方法。那么后面一大串是什么呢？就是执行 sql 的结果的一些状态：

- SQLITE_BUSY 数据库文件有锁
- SQLITE_LOCKED 数据库中的某张表有锁
- SQLITE_DONE sqlite3_step()执行完毕
- SQLITE_ROW sqlite3_step()获取到下一行数据
- SQLITE_ERROR 一般用于没有特别指定错误码的错误，就是说函数在执行过程中发生了错误，但无法知道错误发生的原因。
- SQLITE_MISUSE 没有正确使用SQLite接口，比如一条语句在sqlite3_step函数执行之后，没有被重置之前，再次给其绑定参数，这时bind函数就会返回SQLITE_MISUSE。

##### 获取数据

前面执行完 sql 之后，你就可以拿到数据了，`FMResultSet` 中提供了方法将当前行转化为一个字典：

```objc
- (NSDictionary*)resultDictionary {
    
    NSUInteger num_cols = (NSUInteger)sqlite3_data_count([_statement statement]);
    
    if (num_cols > 0) {
        NSMutableDictionary *dict = [NSMutableDictionary dictionaryWithCapacity:num_cols];
        
        int columnCount = sqlite3_column_count([_statement statement]);
        
        int columnIdx = 0;
        for (columnIdx = 0; columnIdx < columnCount; columnIdx++) {
            
            NSString *columnName = [NSString stringWithUTF8String:sqlite3_column_name([_statement statement], columnIdx)];
            id objectValue = [self objectForColumnIndex:columnIdx];
            [dict setObject:objectValue forKey:columnName];
        }
        
        return dict;
    }
    else {
        NSLog(@"Warning: There seem to be no columns in this set.");
    }
    
    return nil;
}
```

过程就是通过循环，拿出每一列的数据，加入到字典中。当然我们也可以自己获取列数据及列名。

#### 创建 update sql

select sql 需要配合 `FMResultSet`，更新数据库内容则比较简单。直接使用 `FMDatabase` 更新即可。update sql 的代码几乎和 select sql 一模一样，不同的是，更新操作在创建好 sql 后，直接执行 `sqlite3_step()`，执行完后根据是否要缓存选择性执行重置 `sqlite3_reset()` 或者关闭 `sqlite3_finalize()`。代码太长且重复，就不贴了。 

#### 加解密

FMDB 封装了为 db 加解密的方法。解密使用如下方法，在打开 db 前使用，否则报错：

```objc
- (BOOL)setKey:(NSString*)key {
    NSData *keyData = [NSData dataWithBytes:[key UTF8String] length:(NSUInteger)strlen([key UTF8String])];
    
    return [self setKeyWithData:keyData];
}

- (BOOL)setKeyWithData:(NSData *)keyData {
#ifdef SQLITE_HAS_CODEC
    if (!keyData) {
        return NO;
    }
    
    int rc = sqlite3_key(_db, [keyData bytes], (int)[keyData length]);
    
    return (rc == SQLITE_OK);
#else
#pragma unused(keyData)
    return NO;
#endif
}
```

其实是一个非常简单的封装，就是将 String 转化为 Data，然后使用 `sqlite3_key` 进行解密。

有解密必然是要现有加密的，使用 `sqlite3_rekey()` 方法，可以完成没有密码的时候创建密码，有密码的时候修改密码或者清除密码的操作。代码和解密类似，也不贴了。 

#### 关闭数据库

关闭数据库，需要做两方面处理，一方面是清除 fmdb 创建的缓存，一方面是释放 sqlite 资源：

```objc
- (BOOL)close {
    
    [self clearCachedStatements];
    [self closeOpenResultSets];
    
    if (!_db) {
        return YES;
    }
    
    int  rc;
    BOOL retry;
    BOOL triedFinalizingOpenStatements = NO;
    
    do {
        retry   = NO;
        rc      = sqlite3_close(_db);
        if (SQLITE_BUSY == rc || SQLITE_LOCKED == rc) {
            if (!triedFinalizingOpenStatements) {
                triedFinalizingOpenStatements = YES;
                sqlite3_stmt *pStmt;
                while ((pStmt = sqlite3_next_stmt(_db, nil)) !=0) {
                    NSLog(@"Closing leaked statement");
                    sqlite3_finalize(pStmt);
                    retry = YES;
                }
            }
        }
        else if (SQLITE_OK != rc) {
            NSLog(@"error closing!: %d", rc);
        }
    }
    while (retry);
    
    _db = nil;
    return YES;
}
```

这里先尝试用 `sqlite3_close()` 关闭，如果不行，那么再 `sqlite3_next_stmt()` 来获取每个 stmt，然后将他们 `sqlite3_finalize()`。整个过程在一个大的 while 循环中，直到数据库关闭为止。

#### 多线程

有些费时的更新操作我们不希望在主线程中进行。FMDB 提供了 `FMDatabaseQueue` 这个类帮助我们创建了后台线程。但是要注意，只是帮我们创建了后台线程，我们不能在多个线程中共用一个 FMDatabase 对象，这个类不是线程安全的，否则会引起数据混乱。

##### 初始化创建队列

创建队列的代码如下：

```objc
static const void * const kDispatchQueueSpecificKey = &kDispatchQueueSpecificKey;

- (instancetype)initWithPath:(NSString*)aPath flags:(int)openFlags vfs:(NSString *)vfsName {
    
    self = [super init];
    
    if (self != nil) {
        
        _db = [[[self class] databaseClass] databaseWithPath:aPath];
        FMDBRetain(_db);
        
		...
          
        _path = FMDBReturnRetained(aPath);
        
        _queue = dispatch_queue_create([[NSString stringWithFormat:@"fmdb.%@", self] UTF8String], NULL);
        dispatch_queue_set_specific(_queue, kDispatchQueueSpecificKey, (__bridge void *)self, NULL);
        _openFlags = openFlags;
    }
    
    return self;
}
```

主要分为两步，第一步是创建 db。第二步是创建队列。之后还用 `dispatch_queue_set_specific` 绑定了 `FMDatabaseQueue` 对象以及 `queue`，这个的用处下面再说。

##### 执行 sql

FMDB 为 `FMDatabaseQueue` 提供了一个方法批量处理某一个 db 的 sql。它接收一个 sql 的 block：

```objc
- (void)inDatabase:(void (^)(FMDatabase *db))block {
    /* Get the currently executing queue (which should probably be nil, but in theory could be another DB queue
     * and then check it against self to make sure we're not about to deadlock. */
    FMDatabaseQueue *currentSyncQueue = (__bridge id)dispatch_get_specific(kDispatchQueueSpecificKey);
    assert(currentSyncQueue != self && "inDatabase: was called reentrantly on the same queue, which would lead to a deadlock");
    
    FMDBRetain(self);
    
    dispatch_sync(_queue, ^() {
        
        FMDatabase *db = [self database];
        block(db);
        
		...
    });
    
    FMDBRelease(self);
}
```

可以看到，主要就是在之前创建的 queue 中同步执行 sql。那么 `dispatch_get_specific` 有何用意呢？简单的说就是如果当前队列为 `_queue`，下面的同步操作就会产生死锁。所以这里 `dispatch_get_specific` 就是为了验证一下，现在是不是在 `_queue` 队列中。如果是，那么 `currentSyncQueue` 就不为空，那么直接通过断言触发异常。

其实我觉得这个判断没什么用啦，因为 `inDatabase:` 方法是使用者调用的，根本不可能让它在 `_queue` 中执行啊。

#### 事务处理

事务处理的方法如下：

```objc
- (void)inTransaction:(void (^)(FMDatabase *db, BOOL *rollback))block {
    [self beginTransaction:NO withBlock:block];
}

- (void)beginTransaction:(BOOL)useDeferred withBlock:(void (^)(FMDatabase *db, BOOL *rollback))block {
    FMDBRetain(self);
    dispatch_sync(_queue, ^() { 
        
        BOOL shouldRollback = NO;
        
        if (useDeferred) {
            [[self database] beginDeferredTransaction];
        }
        else {
            [[self database] beginTransaction];
        }
      
        block([self database], &shouldRollback);
      
        if (shouldRollback) {
            [[self database] rollback];
        }else {
            [[self database] commit];
        }
    });
    
    FMDBRelease(self);
}

- (BOOL)rollback {
    BOOL b = [self executeUpdate:@"rollback transaction"];

    if (b) {
        _inTransaction = NO;
    }
    
    return b;
}

- (BOOL)commit {
    BOOL b =  [self executeUpdate:@"commit transaction"];

    if (b) {
        _inTransaction = NO;
    }
    
    return b;
}

- (BOOL)beginDeferredTransaction {
    
    BOOL b = [self executeUpdate:@"begin deferred transaction"];
    if (b) {
        _inTransaction = YES;
    }
    
    return b;
}

- (BOOL)beginTransaction {
    
    BOOL b = [self executeUpdate:@"begin exclusive transaction"];
    if (b) {
        _inTransaction = YES;
    }
    
    return b;
}
```

事务处理主要还是调用事务的相关 sql 语句，使用者可以通过 sql 是否执行成功，来决定是否需要回滚。

这里 block 的入参使用的是 `BOOL *roolback`，然后传入一个 BOOL 的地址 `&shouldRoolBack`，可以实现不用返回值传递数据。注意，这种取地址的写法在使用的时候要用 `*shouldRoolBack` 来取地址上的值，因为是基本类型的指针：

```objc
BOOL roolback = YES;
BOOL *r = &roolback;
*r = NO;
NSLog(@"%d，roolback 从1变为了0",roolback);   
```

非基本类型的指针的赋值直接就是 `p = Person()` 这种地址的赋值就行了，**不存在 `*p` 的情况**。基本类型的指针必须通过 `*r` 拿到堆上的值 `*r = No`

### 总结

FMDB 的整个过程相对简单，简单来说就是先初始化控制类 `FMDatabase`，然后通过这个类打开 db，执行 sql，关闭数据库等操作。执行的 sql 需要转化为 sqlite 使用的 `sqlite3_stmt` 类型，并缓存。对于有结果的 sql，或创建一个 `FMResultSet` 来保存 sql 已经其相应结果。多线程通过 `FMDatabaseQueue` 实现，它可以为 sql 开启后台线程执行，并且封装了 sqlite 的原子性操作的语句来实现事务。

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/FMDB_1.png?raw=true)

## JQFMDB

JQFMDB 是 FMDB 的一层简单封装。FMDB 只是对 sqlite 进行了封装。JQFMDB 在其基础上封装了一些常用的 sql 语句。也就是说，一般的数据库操作只要调用适当的方法即可，不需要我们自己写 sql 语句了。

另外，JQFMDB 还提供了字典模型转换的功能。即你在执行 sql 方法的时候，传入 model 的 class 类型。会自动进行属性和键的匹配。

代码比较简单，稍微想一下就知道是如何完成的。所以就不做具体解析了。

## BGFMDB



