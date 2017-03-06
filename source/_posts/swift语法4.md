title: Swift 3.1 语法学习（四）
date: 2017/2/22 14:07:17  
categories: iOS
tags: 

	- Swift


------

接着上一篇

<!--more-->

## 继承

### 定义一个基类

不继承于其它类的类，称之为*基类*。

```swift
class Vehicle {
    var currentSpeed = 0.0
    var description: String {
        return "traveling at \(currentSpeed) miles per hour"
    }
    func makeNoise() {
        // 什么也不做-因为车辆不一定会有噪音
    }
}
```

### 子类生成

为了指明某个类的超类，将超类名写在子类名的后面，用冒号分隔：

```swift
class SomeClass: SomeSuperclass {
    // 这里是子类的定义
}
```

继承上面的基类：

```swift
class Bicycle: Vehicle {
    var hasBasket = false
}
```

除了继承了 `Vehicle` 的属性外，还定义了一个 `hasBasket` 属性。

基本和其他语言的继承无异。

### 重写

如果要重写某个特性，你需要在重写定义的前面加上`override`关键字。这么做，你就表明了你是想提供一个重写版本，而非错误地提供了一个相同的定义。任何缺少`override`关键字的重写都会在编译时被诊断为错误。

`override`关键字会提醒 Swift 编译器去检查该类的超类（或其中一个父类）是否有匹配重写版本的声明。

#### 访问超类的方法属性和下标

使用 `super` 关键字。

#### 重写方法

没什么特别的。

#### 重写属性

你可以重写继承来的实例属性或类型属性，提供自己定制的 getter 和 setter，或添加属性观察器使重写的属性可以观察属性值什么时候发生改变。

##### 重写属性的 Getters 和 Setters

你可以提供定制的 getter（或 setter）来重写任意继承来的属性，无论继承来的属性是存储型的还是计算型的属性。子类并不知道继承来的属性是存储型的还是计算型的，它只知道继承来的属性会有一个名字和类型。**你在重写一个属性时，必需将它的名字和类型都写出来。**这样才能使编译器去检查你重写的属性是与超类中同名同类型的属性相匹配的。

你可以将一个继承来的只读属性重写为一个读写属性，只需要在重写版本的属性里提供 getter 和 setter 即可。但是，你不可以将一个继承来的读写属性重写为一个只读属性。

如果你不想在重写版本中的 getter 里修改继承来的属性值，你可以直接通过`super.someProperty`来返回继承来的值，其中`someProperty`是你要重写的属性的名字。

##### 重写属性观察器

你可以通过重写属性为一个继承来的属性添加属性观察器。这样一来，当继承来的属性值发生改变时，你就会被通知到，无论那个属性原本是如何实现的。

```swift
class AutomaticCar: Car {
    override var currentSpeed: Double {
        didSet {
            gear = Int(currentSpeed / 10.0) + 1
        }
    }
}

let automatic = AutomaticCar()
automatic.currentSpeed = 35.0
print("AutomaticCar: \(automatic.description)")
// AutomaticCar: traveling at 35.0 miles per hour in gear 4
```

### 防止重写

可以通过把方法，属性或下标标记为*final*来防止它们被重写，只需要在声明关键字前加上`final`修饰符即可（例如：`final var`，`final func`，`final class func`，以及`final subscript`）。

你可以通过在关键字`class`前添加`final`修饰符（`final class`）来将整个类标记为 final 的。这样的类是不可被继承的，试图继承这样的类会导致编译报错。

## 构造过程

构造过程是使用类、结构体或枚举类型的实例之前的准备过程。在新实例可用前必须执行这个过程，具体操作包括设置实例中每个存储型属性的初始值和执行其他必须的设置或初始化工作。

通过定义构造器（`Initializers`）来实现构造过程，这些构造器可以看做是用来创建特定类型新实例的特殊方法。与 Objective-C 中的构造器不同，Swift 的构造器无需返回值。

类的实例也可以通过定义析构器（`deinitializer`）在实例释放之前执行特定的清除工作。

### 存储属性初始赋值

类和结构体在创建实例时，必须为所有存储型属性设置合适的初始值。存储型属性的值不能处于一个未知的状态。你可以在构造器中为存储型属性赋初值，也可以在定义属性时为其设置默认值。

注意，当你为存储型属性设置默认值或者在构造器中为其赋值时，它们的值是被直接设置的，不会触发任何属性观察者（`property observers`）。

#### 构造器

构造器在创建某个特定类型的新实例时被调用。它的最简形式类似于一个不带任何参数的实例方法，以关键字`init`命名：

```swift
init() {
    // 在此处执行构造过程
}
```

例子：

```swift
struct Fahrenheit {
    var temperature: Double
    init() {
        temperature = 32.0
    }
}
var f = Fahrenheit()
print("The default temperature is \(f.temperature)° Fahrenheit")
// 输出 "The default temperature is 32.0° Fahrenheit”
```

#### 默认属性值

你可以使用更简单的方式在定义结构体`Fahrenheit`时为属性`temperature`设置默认值：

```swift
struct Fahrenheit {
    var temperature = 32.0
}
```

### 自定义构造过程

你可以通过输入参数和可选类型的属性来自定义构造过程，也可以在构造过程中修改常量属性。

#### 构造参数

自定义`构造过程`时，可以在定义中提供构造参数，指定所需值的类型和名字。构造参数的功能和语法跟函数和方法的参数相同。

```swift
struct Celsius {
    var temperatureInCelsius: Double
    init(fromFahrenheit fahrenheit: Double) {
        temperatureInCelsius = (fahrenheit - 32.0) / 1.8
    }
    init(fromKelvin kelvin: Double) {
        temperatureInCelsius = kelvin - 273.15
    }
}
let boilingPointOfWater = Celsius(fromFahrenheit: 212.0)
// boilingPointOfWater.temperatureInCelsius 是 100.0
let freezingPointOfWater = Celsius(fromKelvin: 273.15)
// freezingPointOfWater.temperatureInCelsius 是 0.0”
```

第一个构造器拥有一个构造参数，其外部名字为`fromFahrenheit`，内部名字为`fahrenheit`；第二个构造器也拥有一个构造参数，其外部名字为`fromKelvin`，内部名字为`kelvin`。这两个构造器都将唯一的参数值转换成摄氏温度值，并保存在属性`temperatureInCelsius`中。

