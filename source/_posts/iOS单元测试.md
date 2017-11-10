title: iOS 单元测试
date: 2017/9/9 10:07:12  
categories: iOS
tags:
	- 学习笔记
---

对于单元测试一直都是轻视的，但是好的单元测试对一些低级错误的检测还是非常高效的。这里收集一下单元测试的使用方式。

<!--more-->

单元测试的好处主要在于，如果一个项目比较大，或者某个功能改动比较多，那么你可能顺手改掉了某些零碎的东西，比如某个配置，但是这个错误又不容易被发现。有了单元测试，当你改掉某些东西的时候，立马就会显示出来。

### 结构

可以通过 new 的方式创建一个 `iOS Unit Testing Bundle`  target。可以看到默认创建了一个单元测试文件：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/unitTest1.png?raw=true)

其中，。这个测试文件继承于 `XCTestCase`，并且提供了几个默认的方法：

- `setUp` 表示在进行测试方法前的准备工作。
- `tearDown` 表示测试方法后的善后工作。
- `testExample` 是一个具体的测试方法，所有的测试方法都必须以 `test` 为开头。Xcode 会识别到，在其前面的代码行数区域就会有一个小方块，可以执行，执行成功就是如图所示。
- `testPerformanceExample` 表示测试性能，在这个闭包中写入要测试的代码。

### 引入目标

我们可以自己创建单元测试文件。对于你要测试的类，你需要引入目标文件，你需要在 import 前加上 `@testable` 标注：

```swift
@testable import TargetClass
```

### 开始测试

在左面板的测试导航栏，可以看到所有的测试方法，你可以邮件运行所有方法，也可以单独运行其中一个方法。当然，你也可以在每个测试文件的测试方法前的代码行数处看到一个小方块，也可以点击这个小方块开始测试。

### 测试断言

测试肯定需要断言的，不然怎么知道测试是否成功呢。常用断言有以下：

```swift
XCTAssert(expression, format...)
XCTFail(format...)
XCTAssertTrue(expression, format...)
XCTAssertFalse(expression, format...)
XCTAssertEqual(expression1, expression2, format...)
XCTAssertNotEqual(expression1, expression2, format...)
XCTAssertEqualWithAccuracy(expression1, expression2, accuracy, format...)
XCTAssertNotEqualWithAccuracy(expression1, expression2, accuracy, format...)
XCTAssertNil(expression, format...)
XCTAssertNotNil(expression, format...)
```

这些断言的意思还是比较好理解的。

### 性能测试

性能测试可以设置基准时间，最大允许的标准差，超过这个标准差，就表示性能测试不通过。具体设置如图所示：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/unitTest3.png?raw=true)

点击性能测试前的灰色小方块，就会出现一个弹框，然后在其中设置即可，包括 baseline 和 Max STDDEV。

### 测试断点

我们知道异常有异常断点。测试失败也有断点。还是老地方设置：

![](https://github.com/zhang759740844/MyImgs/blob/master/MyBlog/unitTest4.png?raw=true)

### 异步测试

测试代码一般执行完就结束了。像网络请求等异步操作需要特殊的方式。

```swift
// 异步测试: 成功块，失败慢
func testValidCallToiTunesGetsHTTPStatusCode200() {
  let url = URL(string: "https://itunes.apple.com/search?media=music&entity=song&term=abba")
  let promise = expectation(description: "Status code: 200")
  let dataTask = sessionUnderTest.dataTask(with: url!) { data, response, error in
    if let error = error {
      XCTFail("Error: \(error.localizedDescription)")
      return
    } else if let statusCode = (response as? HTTPURLResponse)?.statusCode {
      if statusCode == 200 {
        promise.fulfill()
      } else {
        XCTFail("Status code: \(statusCode)")
      }
    }
  }
  dataTask.resume()
  waitForExpectations(timeout: 5, handler: nil)
}
```

这里的 `expectation()` 方法返回一个预期对象。然后通过 `waitForExpectations(timeout:handler:)` 方法设置一个超时时间，执行到这句的时候就会等待预期对象的到来。所以在异步的成功回调中调用 `fulfill()` 方法，表示预期对象到来了。这样代码就执行完了。只要不是通过断言抛出的错误，或者超时，都表示测试成功。

注意，`waitForExpectations` 是暂停，也就是说，这个方法后面的方法会先不执行，等到 `fulfill()` 后再执行。所以，**一般我们可以把判断的断言放在这个方法之后**。

### 方法命名

测试方法必须要以 `test` 为头，方法命名要把这个测试方法的功能讲清楚，不需要使用驼峰式命名，用下划线隔开即可。如：

```swift
func test_weatherDataAt_handle_statuscode_not_equal_to_200() {}
```



### 测试私有方法和属性

我们不能直接获取到私有的方法或者属性，但是我们可以通过 category 暴露私有方法和属性：

```objc
@interface JHSTestDataSource (UnitTest)
- (NSInteger)getSellGroupCount;
- (BOOL)needShowHeader:(NSInteger)section;
@end
```

### 最后  

关于如何 mock 数据，OC 中提供了 OCMock 这个库，可以通过 runtime 将方法返回的结果替换。但是 Swift 由于是静态语言，并不能提供支持。

