title: Swift 3.1 语法学习（六）
date: 2017/2/22 14:07:17  
categories: iOS
tags: 

	- Swift


------

接着上一篇

<!--more-->
## 方法

类、结构体、枚举都可以定义实例、类型方法（这和 oc 中不同，oc 只能在类中定义方法）。

### 实例方法

实例方法的语法和函数完全一致：

```swift
class Counter {
    var count = 0
    func increment() {
        count += 1
    }
    func incrementBy(amount: Int) {
        count += amount
    }
    func reset() {
        count = 0
    }
}
```

和调用属性一样，用点语法调用实例方法：

```swift
let counter = Counter()
// the initial counter value is 0
counter.increment()
// the counter's value is now 1
counter.increment(by: 5)
// the counter's value is now 6
counter.reset()
// the counter's value is now 0
```

#### self 属性

每个实例都有一个隐式的属性叫做 `self`。我们可以在实例方法中，通过 `self` 获取当前实例。

可以改写上面的 `incremnet()` 方法：

```swift
func increment() {
    self.count += 1
}
```

实际使用中，并不太需要 `self`，因为没必要。那么什么时候是必须的呢？

```swift
struct Point {
    var x = 0.0, y = 0.0
    func isToTheRightOf(x: Double) -> Bool {
        return self.x > x
    }
}
let somePoint = Point(x: 4.0, y: 5.0)
if somePoint.isToTheRightOf(x: 1.0) {
    print("This point is to the right of the line where x == 1.0")
}
// Prints "This point is to the right of the line where x == 1.0"
```

比如上面的代码，实例方法的入参名和实例属性相同，这个时候必须要用 `self` 以示区分。




