title: Swift 3.1 语法学习（五）
date: 2017/2/22 14:07:16  
categories: iOS
tags: 

	- Swift


------

接着上一篇

<!--more-->


## 类和结构体

### 定义语法

使用 `class` 和 `struct` 分别表示类和结构体。示例如下：

```swift
struct Resolution {
    var width = 0
    var height = 0
}
class VideoMode {
    var resolution = Resolution()
    var interlaced = false
    var frameRate = 0.0
    var name: String?
}
```

这里面可以注意一点。就是对于变量进行了初始化。这样有什么好处呢？可以使 swift 进行类型推断出当前变量的类型。在 **swift 中，每个变量的类型都必须是确定的。**

### 类和结构体实例

创建类和结构体实例的语法相似：

```swift
let someResolution = Resolution()
let someVideoMode = VideoMode()
```

通过这种方式所创建的类或者结构体实例，其属性均会被初始化为默认值。更多构造过程在后面将会更详细的讨论。

### 属性访问

通过 `.` 语法，可以访问实例的属性。与 oc 不同的是，Swift 允许直接设置结构体属性的子属性。反正通过 `.` 什么都能拿到就是了。

### 结构体类型的成员逐一构造器

所有**结构体（特指结构体）**都有一个自动生成的*成员逐一构造器*，用于初始化新结构体实例中成员的属性。新实例中各个属性的初始值可以通过属性的名称传递到成员逐一构造器之中：

```swift
let vga = Resolution(width:640, height: 480)
```

与结构体不同，类实例没有默认的成员逐一构造器。详细见后面。

### 结构体和枚举是值类型

什么是值类型？就是我们所说的值传递和引用传递中的值传递。指在被赋给一个变量常量或者传递给一个函数的时候，值会被拷贝。

所有的基本类型都是值拷贝这和以前毫无异议。但是在 Swift 中，**字符串、数组和字典也是值拷贝**，这和 oc 中很不相同。这意味着数组和字典中的所有元素都会被拷贝一份。其实是因为**它们底层都是以结构体的形式实现的**。

```swift
let hd = Resolution(width: 1920, height: 1080)
var cinema = hd
```

例如上面的例子，`cinema` 和 `hd` 相同，但其实在内存中是两个不同的对象。修改其中一个的值不会改变另一个实例相应属性的值。**枚举同样**。

### 类是引用类型

这点没啥不同的。就是不同常量或者变量指向内存上的相同地址。



### 恒等运算符

Swift 中内建了两个恒等运算符：

- 等价于（`===`）
- 不等价于 （`!==`）

注意这和“等于” `==` 有什么区别呢？

- "等价于"表示两个**类类型**(注意只能用在类类型中)的常量或者变量引用同一个类实例。
- “等于”表示两个实例的值“相等”或“相同”。

也就是说 `===` 为 `true` ，那么两个类实例必然指向同一块内存地址。`==` 为 `true` 则只要类内属性相同即可。（oc 中的 `==` 就是这里的 `===`，比较的是指针地址。）



### 指针

一个引用某个引用类型实例的 Swift 常量或者变量，与 C 语言中的指针类似，但是并不直接指向某个内存地址，也不要求你使用星号（`*`）来表明你在创建一个引用。Swift 中的这些引用与其它的常量或变量的定义方式相同。(反正就是简写了)



### 类和结构体的选择

结构体实例总是通过值传递，类实例总是通过引用传递。这意味两者适用不同的任务。

其实大部分数据构造都是用类，而非结构体。只有在数据结构非常简单的时候用结构体。



### 字符串、数组、字典的赋值和复制行为

上面也说过了 Swift 中的 `String`，`Array`和`Dictionary`类型均以结构体的形式实现。这意味着被赋值给新的常量或变量，或者被传入函数或方法中时，它们的值会被拷贝。

Objective-C 中`NSString`，`NSArray`和`NSDictionary`类型均以类的形式实现，而并非结构体。它们在被赋值或者被传入函数或方法时，不会发生值拷贝，而是传递现有实例的引用。

不过不用担心值拷贝会影响新能，Swift 中有优化。

## 属性

### 存储属性

类和结构体中用 `var` 或者 `let` 修饰的就是存储属性：

```swift
struct FixedLengthRange {
    var firstValue: Int
    let length: Int
}
var rangeOfThreeItems = FixedLengthRange(firstValue: 0, length: 3)
// 该区间表示整数0，1，2
rangeOfThreeItems.firstValue = 6
// 该区间现在表示整数6，7，8
```

#### 常量结构体的存储属性

**如果创建了一个结构体的实例并将其赋值给一个常量，则无法修改该实例的任何属性，即使有属性被声明为变量也不行**：

```swift
let rangeOfFourItems = FixedLengthRange(firstValue: 0, length: 4)
// 该区间表示整数0，1，2，3
rangeOfFourItems.firstValue = 6
// 尽管 firstValue 是个变量属性，这里还是会报错
```

因为 `rangeOfFourItems` 被声明成了常量（用 `let` 关键字），即使 `firstValue` 是一个变量属性，也无法再修改它了。

**这种行为是由于结构体（struct）属于*值类型*。当值类型的实例被声明为常量的时候，它的所有属性也就成了常量。**属于*引用类型*的类（class）则不一样。把一个引用类型的实例赋给一个常量后，仍然可以修改该实例的变量属性。

**因此注意，数组字典等如果设置为 `let` ，那么里面的内容也是不能改的。**

#### 延迟存储属性

指当第一次被调用的时候才会计算其初始值的属性。在属性声明前使用 `lazy` 来标识。注意，延迟存储属性必须被声明为 `var`。

```swift
class DataImporter {
    /*
    DataImporter 是一个负责将外部文件中的数据导入的类。
    这个类的初始化会消耗不少时间。
    */
    var fileName = "data.txt"
    // 这里会提供数据导入功能
}

class DataManager {
    lazy var importer = DataImporter()
    var data = [String]()
    // 这里会提供数据管理功能
}

let manager = DataManager()
manager.data.append("Some data")
manager.data.append("Some more data")
// DataImporter 实例的 importer 属性还没有被创建
```

上面这个类中，`DataImporter` 是一个很费时的操作。所以设置为 `lazy`，只有在第一次访问到的时候才会被创建。



### 计算属性

计算属性不直接存储值，而是提供一个 getter 和一个可选的 setter，来间接获取和设置其他属性或变量的值。**其实就是每次点到这个属性的时候都会再计算一遍，适用于会根据其他值变化的属性，这样就不用每次用到的时候专门调用一个处理方法了。而是由系统直接调用了 getter 方法**

```swift
struct Point {
    var x = 0.0, y = 0.0
}
struct Size {
    var width = 0.0, height = 0.0
}
struct Rect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            let centerX = origin.x + (size.width / 2)
            let centerY = origin.y + (size.height / 2)
            return Point(x: centerX, y: centerY)
        }
        set(newCenter) {
            origin.x = newCenter.x - (size.width / 2)
            origin.y = newCenter.y - (size.height / 2)
        }
    }
}
var square = Rect(origin: Point(x: 0.0, y: 0.0),
    size: Size(width: 10.0, height: 10.0))
let initialSquareCenter = square.center
square.center = Point(x: 15.0, y: 15.0)
print("square.origin is now at (\(square.origin.x), \(square.origin.y))")
// 输出 "square.origin is now at (10.0, 10.0)”
```

上面这个矩形的类，通过 `origin` 和 `size` 来计算出 `center`。其中 setter 方法的 `newCenter` 由于类型推断，默认为 `Point` 类型，就不用再写明类型了。

#### 便捷 setter 声明

由于 `setter` 函数必然要传入一个新值，所以 Swift 定义了一个默认名称 `newValue`。所以可以采取简略的形式：

```swift
struct AlternativeRect {
    var origin = Point()
    var size = Size()
    var center: Point {
        get {
            let centerX = origin.x + (size.width / 2)
            let centerY = origin.y + (size.height / 2)
            return Point(x: centerX, y: centerY)
        }
        set {
            origin.x = newValue.x - (size.width / 2)
            origin.y = newValue.y - (size.height / 2)
        }
    }
}
```

#### 只读计算属性

只有 getter 没有 setter 的计算属性就是*只读计算属性*。只读计算属性总是返回一个值，可以通过点运算符访问，但不能设置新的值。

如果是只读计算属性，那么连 `get` 关键字都可以扔掉了：

```swift
struct Cuboid {
    var width = 0.0, height = 0.0, depth = 0.0
    var volume: Double {
        return width * height * depth
    }
}
let fourByFiveByTwo = Cuboid(width: 4.0, height: 5.0, depth: 2.0)
print("the volume of fourByFiveByTwo is \(fourByFiveByTwo.volume)")
// 输出 "the volume of fourByFiveByTwo is 40.0"
```

### 属性观察器

可以监听属性值的变化（可以为除了延迟存储属性之外的其他存储属性添加属性观察器，因为可以通过 setter 方法直接监控）。

提供了两个属性观察器：

- `willSet` 新值被设置前调用
- `didSet` 新值被设置后调用

`willSet` 接受新的属性值作为常量传入，可以自己指定这个参数的名称。如果不指定，默认名称为 `newValue`。

`didSet` 将旧的属性值传入，不接受自定义参数名，默认参数名为 `oldValue`。如果在 `didSet` 方法中再次对该属性赋值，那么新值会覆盖旧的值。

```swift
class StepCounter {
    var totalSteps: Int = 0 {
        willSet(newTotalSteps) {
            print("About to set totalSteps to \(newTotalSteps)")
        }
        didSet {
            if totalSteps > oldValue  {
                print("Added \(totalSteps - oldValue) steps")
            }
        }
    }
}
let stepCounter = StepCounter()
stepCounter.totalSteps = 200
// About to set totalSteps to 200
// Added 200 steps
stepCounter.totalSteps = 360
// About to set totalSteps to 360
// Added 160 steps
stepCounter.totalSteps = 896
// About to set totalSteps to 896
// Added 536 steps
```

如果将属性通过 in-out 方式传入函数，`willSet` 和 `didSet` 也会调用。这是因为 in-out 参数采用了拷入拷出模式：即在函数内部使用的是参数的 copy，函数结束后，又对参数重新赋值。

### 全局变量和局部变量

全局变量是在函数、方法、闭包或任何类型之外定义的变量。局部变量是在函数、方法或闭包内部定义的变量。

全局的常量或变量都是延迟计算的，跟延迟存储属性相似，不同的地方在于，全局的常量或变量不需要标记`lazy`修饰符。
局部范围的常量或变量从不延迟计算。

### 类型属性

实例属性属于一个特定类型的实例，每创建一个实例，实例都拥有属于自己的一套属性值，实例之间的属性相互独立。类似于静态变量或常量。

跟实例的存储型属性不同，必须给存储型类型属性指定默认值，因为类型本身没有构造器，也就无法在初始化过程中使用构造器给类型属性赋值。
存储型类型属性是延迟初始化的，它们只有在第一次被访问的时候才会被初始化。即使它们被多个线程同时访问，系统也保证只会对其进行一次初始化，并且不需要对其使用 `lazy` 修饰符。

####  类型属性语法

在 C 或 Objective-C 中，与某个类型关联的静态常量和静态变量，是作为全局（*global*）静态变量定义的。但是在 Swift 中，类型属性是作为类型定义的一部分写在类型最外层的花括号内，因此它的作用范围也就在类型支持的范围内。

使用关键字 `static` 来定义类型属性。在为类定义计算型类型属性时，可以改用关键字 `class` 来支持子类对父类的实现进行重写。下面的例子演示了存储型和计算型类型属性的语法：

```swift
struct SomeStructure {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 1
    }
}
enum SomeEnumeration {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 6
    }
}
class SomeClass {
    static var storedTypeProperty = "Some value."
    static var computedTypeProperty: Int {
        return 27
    }
    class var overrideableComputedTypeProperty: Int {
        return 107
    }
}
```

(为什么要用 class 来标识呢？计算型存储属性又如何重写父类呢？)

#### 获取和设置类型属性的值

类型属性通过类型本身访问：
```swift
print(SomeStructure.storedTypeProperty)
// 输出 "Some value."
SomeStructure.storedTypeProperty = "Another value."
print(SomeStructure.storedTypeProperty)
// 输出 "Another value.”
print(SomeEnumeration.computedTypeProperty)
// 输出 "6"
print(SomeClass.computedTypeProperty)
// 输出 "27"
```
