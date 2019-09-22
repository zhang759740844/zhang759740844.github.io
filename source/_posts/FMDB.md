title: FMDB 及其封装框架的实现过程
date: 2017/11/30 10:07:12  
categories: iOS
tags:

	- 学习笔记
---

iOS 上的数据库用的不多，看一看主流的 FMDB 以及其封装框架是怎么实现的。

<!--more-->

## 简单SQL

### 检索

#### 检索数据

##### 检索单个列

```sql
select prod_name from products;
```

##### 检索多个列

```sql
select prod_id, prod_name, prod_price from products;
```

##### 检索所有列

```sql
select * from produces
```

##### 检索的列去重

```sql
select distinct prod_name from products;
```

##### 限制展示个数

```sql
select prod_name from products limit 5;
```

#### 排序检索

##### 简单排序

```sql
select prod_name from products order by prod_name;
```

##### 多列排序

```sql
select prod_id, prod_name, prod_price from products order by prod_name, prod_id;
```

##### 指定排序方向

```sql
select prod_id, prod_name, prod_price from products order by prod_name desc, prod_id limit 6;
```

### 过滤

#### 过滤数据

##### where 过滤

```sql
select prod_name from products where prod_price = 123;
```

##### 范围值

```sql
select prod_name from products where prod_id between 1 and 100;
```

##### 空值

```sql
select price_id from products where prod_name is null;
// 非空值
select price_id from products where prod_name is not null;
```

##### and or 操作符

```sql
select prod_price from products where (vend_id = 1003 or vend_id = 1004) and prod_price >= 100;
```

##### 指定范围

```sql
select prod_name from products where vend_id in (1003, 1004);
```

#### 正则过滤

##### 字符包含匹配

```sql
select prod_name from products where prod_name regexp '1000';
```

`.`匹配任意一个字符：

```sql
select prod_name from products where prod_name regexp '.000';
```

##### OR 匹配任意一个条件

```sql
select prod_name from products where prod_name regexp '1000|2000|3000';
```

##### 匹配几个字符串之一

```sql
select prod_name from products where prod_name regexp '[123] Ton';
```

```sql
select prod_name from products where prod_name regexp '[1-5a-z] Ton';
```

`^`  表示否定：

```sql
select prod_name from products where prod_name regexp '[^123] Ton';
```



##### 匹配多个实例

| 元字符 |        说明        |
| :----: | :----------------: |
|   +    |     1个或多个      |
|   ?    |     0个或一个      |
|   *    |     0个或多个      |
|  {n}   |    指定数目匹配    |
|  {n,}  | 不少于指定数目匹配 |
| {n,m}  |    匹配数目范围    |

直接跟在要匹配的字符后面，如果要匹配的是字符串，需要给字符串加上括号

##### 定位符

| 元字符 |   说明   |
| :----: | :------: |
|   ^    | 文本开始 |
|   $    | 文本结束 |

### 函数

#### 数据聚合

对一个列的所有行进行操作，返回一个聚合的值

##### sum() 求和avg() 求平均值

```sql
select sum(prod_price) as sum_price from products;
select avg(prod_price) as avg_price from products;
```

##### count() 求数量

```sql
 // 排除 prod_price 为 null 的
select count(prod_price) as num_price from  products;
// count(*) 表示所有行数
select count(*) as all_row from products;
```

##### min() max()求最小最大

```sql
select min(prod_price) as min_price from products;
select max(prod_price) as max_price from products;
```

### 分组

分组的作用是对执行分组，并对结果进行聚合

#### 创建分组

获取不同值的 `vend_id` 并将每个值的数量以 `num_prods` 字段展示

```sql
select vend_id, count(*) as num_prods from products group by vend_id;
```

#### 过滤分组

**`having` 在数据分组后进行过滤，`where` 在数据分组前进行过滤**

```sql
select vend_id, count(*) as num_prods from products where prod_price >= 10 group by vend_id having count(*) >= 2;
```

#### 分组和排序

在上面的基础上再对某一列排序：

```sql
select vend_id, count(*) as num_prods from products where prod_price >= 10 group by vend_id having count(*) >= 2 order by num_prods;
```

### 联结表

#### 联结

##### 定义

检索数据的时候对多张表进行操作。

譬如一个商品表，要记录这个商品的供应商的信息。如果供应商的信息存在这个商品表中，就会存在诸多重复数据。因此，应该讲供应商信息也单独创建一个表。供应商的 ID 为供应商表的**主键**。商品表只保存这个供应商的 ID，这个供应商的 ID 叫做商品表的**外键**，供应商表和商品表通过这个外键进行的关联。

##### 创建联结

```sql
select vend_name, prod_name, prod_price from vendors, products where vendors.vend_id = products.vend_id order by vend_name, prod_name;
```

上面从供应商(`vendors`) 和商品(`products`)两张表中取 `vend_name`,`prod_name`,`prod_price` 这三列的数据。通过 `vend_id` 进行关联。

相当于在运行时把两张表通过主键和外键关联，结合成了一张表。

#### 外连接

要查询表A中所有满足条件的行，并顺便把表B中的相应行取出来：

```sql
SELECT * FROM player LEFT JOIN team on player.team_id = team.team_id where player.age < 30
```

从 player 表中把年纪小于 30 的都取出来，并且把 player 对应的 team 的相关信息从 team 表中取出。

on 关键字用于配合 left join，找到 player 相关的 team

### 增删改

#### 插入

##### 插入一行

```sql
insert into customers(
	cust_name,
	cust_city,
	cust_address
) values (
	'zachary',
  'shanghai',
  'minhang'
);
```

##### 插入多个行

```sql
insert into customers(
	cust_name,
	cust_city,
	cust_address
) values (
	'zachary',
  'shanghai',
  'minhang'
), (
  'zachary2',
  'shanghai2',
  'minhang2'
);
```

##### 插入检索的数据

```sql
insert into customers(
 	cust_name,
	cust_city,
	cust_address
) select
		cust_name,
		cust_city,
		cust_address
	from custnew; 
```

把从 `custnew` 表中检索出的这几列插入到 `customers` 中

#### 更新数据

```sql
update customers
set cust_name = 'zhang',
		cust_address = NULL
where cust_id = 10004;
```

#### 删除数据

```sql
delete from customers
where cust_id = 1004;
```

### 操作表

#### 创建表

```sql
create table customer (
	cust_id				int 			not null auto_increment,
  cust_name 		char(50) 	not null,
  cust_address 	char(50) 	default 'shanghai',
  cust_email 		char(50),
  primary key (cust_id)
) engine=innodb;
```

创建表的时候要指定字段，类型；

非空的字段要手动通过 `not null` 标识；

可以通过 `default` 指定默认值；

表的主键通过创建表的时候用 `primary key` 指定，并且主键是必须唯一的，所以可以通过 `auto_increment` 标识其自增。

最后 `engine=innodb` 指定引擎。

#### 更新表结构

添加列和删除列分别使用 `add` 和 `drop`

```sql
alter table vendors
add vend_phone char(20),
drop column vend_num;
```

#### 删除表

```sql
drop table customer2;
```

## 数据库基础

### 事务

什么是事务？事务就有四大特性：ACID

- A：原子性
- C：一致性
- I：隔离性
- D：持久性

### 并发存在异常

- **脏读**：读到了其他事务还没有提交的数据。
- 不可重复读：对某数据进行读取，发现两次读取的结果不同，也就是说没有读到相同的内容。这是因为有其他事务对这个数据同时进行了修改或删除。
- 幻读：事务A根据条件查询得到了N条数据，但此时事务B更改或者增加了M条符合事务A查询条件的数据，这样当事务A再次进行查询的时候发现会有N+M条数据，产生了幻读。

### 数据库设计范式

设计数据库模型的时候，需要对内部属性之间的联系的合理化程度进行定义。这种规范叫做范式(NF)。

1NF 指的是数据库表中的任何属性都是原子性的，不可再分。

2NF 指的数据表里的非主属性都要和这个数据表的候选键有完全依赖关系。比如:

```
一张表中的字段如下
(球员编号, 比赛编号) → (姓名, 年龄, 比赛时间, 比赛场地，得分)

需要拆分为两行表=>
(球员编号) → (姓名，年龄)
(比赛编号) → (比赛时间, 比赛场地)
```

3NF 在满足 2NF 的同时，对任何非主属性都不传递依赖于候选键。比如：

```
(球员编号) → (姓名，年龄，球队名称，球队教练)
=>
(球员编号) → (姓名，年龄)
(球队名称) → (球队教练)
```

> 超键：能唯一标识元组的属性集叫做超键。
>
> 候选键：如果超键不包括多余的属性，那么这个超键就是候选键。

## FMDB

### 基本结构

FMDB 是 sqlite 的简单封装，主要用来执行 sql 语句，并取出数据。主要有三个类：

- `FMDatabase` : FMDB 最重要的类，用来保存 database 的实例，并对这个 db 执行 sql 语句。
- `FMResultSet`：保存了 sql 语句的执行结果。
- `FMDatabaseQueue`：在多线程环境下执行 sql。

### 基本使用

#### 建立开启数据库和关闭

```objc
// 创建数据库
NSString* dir = [NSSearchPathForDirectoriesInDomains( NSLibraryDirectory, NSUserDomainMask, YES) lastObject];

NSString* dbPath = [dir stringByAppendingPathComponent:@"student.sqlite"];

FMDatabase* db = [FMDatabase databaseWithPath:dbPath];

//2.获取数据库
if ([db open]) {
   NSLog(@"打开数据库成功");
} else {
   NSLog(@"打开数据库失败");
}
```

当文件不存在时，fmdb 会自己创建一个。

```objc
// 关闭数据库
[db close];
```

#### 创建删除表

```objc
//3.创建表
BOOL result = [db executeUpdate:@"CREATE TABLE IF NOT EXISTS t_student (
	id integer PRIMARY KEY AUTOINCREMENT,
	name text NOT NULL,
	age integer NOT NULL,
	sex text NOT NULL);"
];
if (result) {
    NSLog(@"创建表成功");
} else {
    NSLog(@"创建表失败");
}
```

```objc
// 如果表格存在 则销毁
BOOL result = [_db executeUpdate:@"drop table if exists t_student"];
if (result) {
    NSLog(@"删除表成功");
} else {
    NSLog(@"删除表失败");
}
```

#### 增删改查

##### 增

```objc
for (int i = 0; i < 4; i++) {
    //插入数据
    NSString * name = [NSString stringWithFormat: @ "测试名字%@", @(mark_student)];
    int age = mark_student;
    NSString * sex = @ "男";
    mark_student++;
    //1.executeUpdate:不确定的参数用？来占位（后面参数必须是oc对象，；代表语句结束）
    BOOL result = [db executeUpdate: @ "INSERT INTO t_student (name, age, sex) VALUES (?,?,?)", name, @(age), sex];
    //2.executeUpdateWithForamat：不确定的参数用%@，%d等来占位 （参数为原始数据类型，执行语句不区分大小写）
    //    BOOL result = [_db executeUpdateWithFormat:@"insert into t_student (name,age, sex) values (%@,%i,%@)",name,age,sex];
    //3.参数是数组的使用方式
    //    BOOL result = [_db executeUpdate:@"INSERT INTO t_student(name,age,sex) VALUES  (?,?,?);" withArgumentsInArray:@[name,@(age),sex]];
    if (result) {
        NSLog(@ "插入成功");
    } else {
        NSLog(@ "插入失败");
    }
}
```

通过 `executeUpdate` 来执行非 select 的增删改语句。另外，有三种方式给 sql 传参：

1. 使用 `?` 做占位符，那么传参就必须是 oc 对象。
2. 使用 `%@` 等做占位符，传参可以是任意相依类型。
3. 使用数组传参，占位符还是使用 `?`，数组中保存的也是 oc 对象。

##### 删改

这两个和增加数据没什么不同，因此就放在一起了：

```objc
//1.不确定的参数用？来占位 （后面参数必须是oc对象,需要将int包装成OC对象）
int idNum = 11;
BOOL result1 = [_db executeUpdate: @ "delete from t_student where id = ?", @(idNum)];
//2.不确定的参数用%@，%d等来占位
//BOOL result = [_db executeUpdateWithFormat:@"delete from t_student where name = %@",@"王子涵"];
if (result1) {
    NSLog(@ "删除成功");
} else {
    NSLog(@ "删除失败");
}
```

```objc
//修改学生的名字
NSString * newName = @ "新名字";
NSString * oldName = @ "测试名字2";
BOOL result2 = [_db executeUpdate: @ "update t_student set name = ? where name = ?", newName, oldName];
if (result2) {
    NSLog(@ "修改成功");
} else {
    NSLog(@ "修改失败");
}
```

##### 查

表的查询要通过 `executeQuery` 执行：

```objc
//查询整个表
FMResultSet * resultSet = [_db executeQuery: @ "select * from t_student"];
//根据条件查询
//FMResultSet * resultSet = [_db executeQuery:@"select * from t_student where id < ?", @(4)];
//遍历结果集合
while ([resultSet next]) {
    int idNum = [resultSet intForColumn: @ "id"];
    NSString * name = [resultSet objectForColumn: @ "name"];
    int age = [resultSet intForColumn: @ "age"];
    NSString * sex = [resultSet objectForColumn: @ "sex"];
    NSLog(@ "学号：%@ 姓名：%@ 年龄：%@ 性别：%@", @(idNum), name, @(age), sex);
}
```

查询结果会保存在 `FMResultSet` 类的实例中。即使操作结果只有一行，也需要先调用 `FMResultSet` 的`next` 方法。

FMDB 提供如下多个方法来获取不同类型的数据：

```objc
intForColumn:
longForColumn:
longLongIntForColumn:
boolForColumn:
doubleForColumn:
stringForColumn:
dateForColumn:
dataForColumn:
```

当然，一个一个自己获取列名也太麻烦了，可以使用 `FMResultSet` 提供的 `resultDictionary` 方法，获取整个字典：

```objc
while ([resultSet next]) {
	NSDictionary *result = [resultSet resultDictionary];
}
```

在使用 `while` 循环的时候，不需要手动关闭 FMResultSet，因为 `[FMResultSet next]` 遍历到最后会调用 `[FMResultSet close]`

#### 线程安全

`FMDatabase` 本身不是线程安全的，所以不要在多线程中使用。需要使用 `FMDatabaseQueue` 来帮助保证线程安全：

```objc
// 创建，最好放在一个单例的类中
FMDatabaseQueue *queue = [FMDatabaseQueue databaseQueueWithPath:aPath];
// 使用
[queue inDatabase:^(FMDatabase *db) {
    [db executeUpdate:@"INSERT INTO myTable VALUES (?)", [NSNumber numberWithInt:1]];
    [db executeUpdate:@"INSERT INTO myTable VALUES (?)", [NSNumber numberWithInt:2]];
    [db executeUpdate:@"INSERT INTO myTable VALUES (?)", [NSNumber numberWithInt:3]];
    FMResultSet *rs = [db executeQuery:@"select * from foo"];
    while ([rs next]) {
        // …
    }
}];
```

#### 事务

在数据库中，事务可以保证数据操作的完整性：

```objc
[_dataBaseQueue inDatabase:^(FMDatabase * _Nonnull db) {
    [db open];
    // 开启事物
    [db beginTransaction];
    BOOL isDeleteGroupSuccess = [db executeUpdate:@"DELETE FROM grouptable WHERE gcid = ?", groupID];
    BOOL isDeleteMembershipSuccess = [db executeUpdate:@"DELETE FROM groupshiptable WHERE gcid = ?", groupID];
    if (!isDeleteGroupSuccess || !isDeleteMembershipSuccess) {
        // 当对两个表的操作中，其中一个失败，数据回滚
        [db rollback];
        return;
    }
    // 提交事物
    [db commit];
    [db close];
}];
```

通过`beginTransaction` 开启一个事务。任意情况下发生错误的时候可以通过 `rollback` 回退，否则通过 `commit` 提交事务。

#### 本地调试

本地调试可以使用免费的数据库查看工具 ，比如：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/fmdb_2.png?raw=true)

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
    ...
    return YES;
}
```

如果已经打开了就直接返回。否则调用 sqlite 提供的的 `sqlite3_open()` 方法，打开的数据库。

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
  	...
}
```

`cachedStatementForQuery:` 就是在字典中查找 sql 对应的 `sqlite3_stmt` 的方法，它返回的 `FMStatement` 是 `sqlite3_stmt` 的封装。

##### 绑定参数

参数随着方法一起传了进来，一般参数有两种，一种是字典类型的，根据 sql 中的参数名插入，还有一种是数组型的，依次替换 sql 中的占位符。(代码很长，就不贴了)

首先通过 `sqlite3_bind_parameter_count()` 获得 sql 的参数个数。然后检查传入的是字典还是数组。如果是字典，遍历字典，通过 `sqlite3_bind_parameter_index()` 拿到键对应的参数索引，然后绑定；如果是数组就依次绑定到对应的列中。绑定也是使用的 sqlite 提供的针对不同类型的一系列绑定方法。

绑定完了后，将这个 `sqlite3_stmt` 暂存。由于是已经绑定了参数，所以可见前面 `sqlite3_reset()` 做的就是将参数清空。

> SQLite 支持使用占位符 `?`，并且在必要的时候绑定参数。所以你不需要把实际的值放入字符串中去。这是一个安全上的考量，它可以守护程序避免 SQL 注入。它也可以帮助你减少必须 escape 值（sql 提供的转义用的命令）这样的不必要的麻烦。

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
		...
    return (rc == SQLITE_ROW);
}
```

其实上面的关键就是调用 sqlite 的 `sqlite3_step()` 方法。

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

有些费时的更新操作我们不希望在主线程中进行。FMDB 提供了 `FMDatabaseQueue` 这个类帮助我们创建了后台线程。**其实就是封装了子线程的操作，其实你也可以自己创建子线程，然后进行 sql，两者没什么区别。**

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

**其实这个判断，就是要求使用者不要用在刚刚创建的 `_queue` 中调用执行 sql 的方法，而是直接在主队列中调用，方法执行的时候会自动将 sql 执行在 `_queue` 中。**

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

