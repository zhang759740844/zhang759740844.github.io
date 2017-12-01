title: FMDB 及其封装框架的实现过程
date: 2017/11/30 10:07:12  
categories: iOS
tags:
	- 学习笔记
---

iOS 上的数据库用的不多，看一看主流的 FMDB 以及其封装框架是怎么实现的。

<!--more-->

FMDB 是 sqlite 的简单封装，主要用来执行 sql 语句，并取出数据。主要有三个类：

- `FMDatabase` : FMDB 最重要的类，用来执行 sql 语句。
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

源码非常简单，就不分析了。可以参见[FMDB源码阅读](http://www.cnblogs.com/polobymulberry/p/5178770.html)。这次是想看看 FMDB 的封装类的实现方式。

## BGFMDB

