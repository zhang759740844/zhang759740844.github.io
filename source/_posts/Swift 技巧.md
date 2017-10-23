title: 《Swift 技巧》笔记
date: 2017/10/20 14:07:39  
categories: iOS
tags: 

	- Swift


------

两天读完了这本书，干货满满

<!--more-->

## Swift 新元素
### 柯里化
原本一个函数要接收两个参数，但是通过科里化就不需要了，先传入一个参数，然后返回一个包含这个参数的闭包，然后就可以再传入任意的参数了。

### 将 protocol 方法声明为 mutating
struct 和 enum 的方法需要使用 `mutating` 标记，否则无法通过编译。并且这个标记对于 class 类型没有影响。所以尽量将协议方法声明为 `mutating`

### Sequence
所以能够 for…in 的都实现了 `Sequence` 协议。会调用这个协议的 `makeIterator()` 方法，该方法返回任意实现了 `IteratorProtocol` 协议的类。在枚举的时候，会调用该协议的 `next()` 方法。

### 元组
对于一个需要返回多个对象的函数，可以使用元组。

### @autoclosure
如果一个函数的入参是一个闭包，并且只有一句且闭包类型为 `() -> T`，那么可以在函数声明时，在函数类型前加上 `@autoclosure` 标记，就可以省略 `{}`

一个疑问：为什么要用 `@autoclosure` 而不直接传这个一句闭包的结果呢？因为这个闭包不一定会执行，如果直接传结果，会造成性能损耗。

### @escaping
如果一个函数的入参中包含一个闭包，且这个闭包将会被保存，那么需要将这个闭包标记为 `@escaping`。

其中所有的 `self` 的属性都需要写明，并且要注意需要打破引用循环。

### Optional Chaining
可选链上，即使左后访问的是一个非可选的值，但是我们拿到的仍然是一个可选值。如下，即使 name 是非可选，toyName 仍然是可选的：
```swift
let toyName = xiaoming.pet?.toy?.name
```

另外还有一个注意点，比如下面的例子：
```swift
let playClosure = {(child: Child) -> ()? in
	child.pet?.toy?.play()
}
```
由于只有一句，那么编译器默认省略了 return，由于 play() 返回 void，又因为是可选链，所以最终这个闭包返回的是可选的Void，即 Void？

### 操作符
可以对操作符进行重载：
```swift
func +(left: Vector2D, right: Vector2D) -> Vector2D {
	return Vector2D(x: left.x + right.x, y: left.y +right.y)
}
```

创建新的操作符要先对操作符进行声明，用的不多，暂且不表。

### func 的参数修饰
可以在入参类型前加上 `inout` 标记，相当于在函数内部创建了一个新的值，然后在函数返回时将这个值赋给 & 修饰的变量

### 下标
下标就像是计算型属性和函数的结合，需要一个入参和出参的声明，并且还要提供 get、set 方法。

### 方法嵌套
一个方法太长，为了明确职责，我们一般将其切分成多个小的方法，但是这些小的方法一般只会调用一次，平铺这些方法并不是最优办法。所以可以将这些方法嵌套在原方法内，这样既能明确职责，又不会出现很多方法。

### 命名空间
Swift 的命名空间是以 target 作为区分的。如果重名，要在类型名称前面加上 target 名。

如果是同一 target 内的同名，那么可以使用 struct 将类型包裹。

### typealias
为已有类型添加一个别名，增加代码的可读性。主要用在协议的关联类型上。在实现协议的类中，为协议的关联类型指定具体的类型。

如果类型存在泛型，别名同样需要引入：
```swift
typealias Worker<T> = Person<T>
```

可以为组合类型实现别名：
```swift
typealias Pat = Cat & Dog
```

### associatedtype
在协议中可以使用 `associatedtype` 关键字来定义一个占位符类型，需要实现协议的类通过 `typealias` 或者类型推断，将占位符类型转化为一个具体的类型：
```swift
protocol Animal {
	associatedtype F
	func eat(_ food: F)
}

struct Tiger: Animal {
	typealias F = Meat
	func eat(_ food: Meat) {
		print("eat \(food)
	}
}
```

注意，`associatedtype` 后，就不能将协议当做独立的类型使用了。这是因为 Swift 需要在编译时确定所有类型，所以不能容忍使用了类型中的 F 没有确定。如果一定要使用，需要使用泛型。比如下面方法中我们只能在方法中使用泛型 T，来代表所有实现 Animal 协议的类型，而不能直接使用 Animal 了：
```swift
func isDangerous<T: Animal>(animal: T) -> Bool {
}
```

### 可变参数函数
写一个可变参数的函数只需要在声明参数时再类型后面加上 `...` 即可。输入的参数将在函数体内部被当做数组来使用。

### 初始化方法顺序
Swift 的初始化方法需要保证类型的所有属性都被初始化。需要先初始化当前类型的所有属性后，才能调用 `super.init()`，然后修改父类属性。如果我们不需要修改父类属性，那么可以省略 super 的调用，编译时会自动加上。

### Designated，Convenience 和 Required
初始化方法遵循两个原则：
1. 初始化路径必须保证对象完全初始化
2. 子类的 designated 初始化方法必须调用父类的 designated 方法

对于某些希望子类中一定实现的 designated 初始化方法，我们可以添加  `required` 关键字就行限制，强制重写。

### 初始化返回 nil
使用 `init?` 定义可失败方法

### static 和 class
描述类型方法或者属性的时候，可以使用关键字 `static` 或者 `class`。但是 class 有一些限制，不能再枚举和机构体中使用，且不能用在类类型的存储属性上。

所以其实无论怎么样，使用 static 就不会错，不要没事用 class

### 多类型容器
容器存放多类型，可以使用 Any 类型，但是这样会失去属性的特性。我们可以将容器声明为都满足某一种协议。

### default 默认参数
可以为某一个入参添加默认值，只需要在在参数类型后加上值，形如`str1: String =  "Default"`。调用的时候，如果想要使用默认值，只要在调用的时候省略掉那个参数即可。

### … 和 ..<
这两个操作符可以接受 Int 返回一个 `Range`。除此之外，还可以接受任何实现了 `Comparable` 协议的输入，比如 `String`。下例表示一个 a-z 的字母范围。
```swift
let interval = "a"..."z"
```

### AnyClass，元类型和 .self
`A.Type` 表示的是 A 这个类型的类型。我们可以通过 `.self` 取出：
```swift
let typeA: A.Type = A.self
```

Swift 中定义了一个别名 `AnyClass` 表示任意类型的类型。
```swift
typealias AnyClass = AnyObject.Type
```

我们可以通过这个类型的类型来调用类型方法：
```swift
let anyClass: AnyClass = A.self
(anyClass as! A.Type).method()
```
上面的例子和直接使用 `A.method()` 是一个效果。但是使用元类型的好处就是可以动态的传递某个类型，而不是如这里只能用 A 调用。

`Any` 表示所有类型，包括 class，struct，enum
`AnyObject` 值表示 class
`AnyClass` 表示 `AnyObject.Type`

### 协议和类方法中的 Self
在协议和类方法中，我们可以使用 `Self` 表示自己的类型：
```swift
protocol IntervalType {
	func clamp(intervalToClamp: Self) -> Self
}
```
声明协议的时候，我们希望协议中使用的类型就是实现这个协议本生的类型，所以使用 `Self`。

这个 Self 表示自身，或者子类。因此，我们不能使用明确的类型，比如下面这样就是错的：
```swift
// 错误示例
func copy() -> Self {
	let result = MyClass()
	return result
}
```
我们创建的也需要是 Self 类型的：
```swift
// 正确示例
func copy() -> Self {
	let result = type(of: Self).init()
	return result
}

required init() {
}
```
注意，这里一定要为 `init()` 方法加上 `required` 关键词。因为前面说了 `Self` 是当前类或者子类，所以我们必须保证子类也实现了 `init()` 方法。另外，我们可以把当前类定义为 `final` ，即不可能有子类，那么就不需要 `required` 了。

### 属性观察
我们可以为一个存储型属性添加 `willSet` 和 `didSet` 对属性进行观察。我们可以使用 `newValue` 和 `oldValue` 获取设定和已经设定的值。

属性观察的一个重要的作用是作为设置值的验证。

### lazy 修饰符
我们可以对一个变量使用 `lazy` 关键字，表示延迟加载。我们可以对这个变量直接赋值，或者使用执行一个闭包。这会在该变量第一次被使用时执行。

### 隐式解包 Optional
隐式解包 Optional 就是在变量后加上 `!`，达到形式上的非可选的效果，但是实质上还是可选类型。

最好少用隐式解包，现在的一种常用的场景是 IBOutlet：
```swift
@IBOutlet weak var button: UIButton!
```

### 多重 Optional
多重可选就是可选变量的可选，普通的可选的两个状态是有或者没有，多重可选的状态是没有或者另一个可选变量：
```swift
var string: String? = “String”
var another: String?? = string
```
如上所示，another 的值为一个可选类型，

因为可选是一个枚举类型，分为有值和没值，所以可以想象多重可选产生的树状结构。

### Optional Map
对于可选类型的集合也可以使用 Map，如果元素不为 nil，则对元素执行 map，如果元素为 nil，则直接返回 nil

### Protocol Extension
协议的拓展可以给协议提供一个默认的实现。我们可以大胆的用动态派发使用最终的实现，不论是类中的具体实现，还是协议中的默认实现。这其实变相的，将 protocol 中的方法设定为了 optional

但是还存在一种情况，协议拓展中实现了协议中没有定义的方法。这个协议没有规定该方法必须要实现。当你协议的实现类也实现了这个方法的时候，那就要看调用该方法的是以协议的类型还是以实现类的类型了。如果以协议类型调用，那么调用协议拓展中的实现；如果以类类型调用，那么调用类中的实现。

### where 和模式匹配
where 的使用场景还是很多的，总结了四种场景。前两种可以代替 if，后两种用来限制泛型：
- case 条件中的限定
- for…in 循环中限制循环的掉入
- 限制泛型，当泛型中存在关联类型时，还可以限制关联类型
- 限制 extension 中的泛型

```swift
// case
switch A
	case let x where x == 5:
	
// for...in
for score in n where score > 60 {
}

// class
class A<T,K>: a where T: t, K: k, T.tType = K.kType: {
}

// func
func method<T>(param: T) where T: s {
}

// extension
extension A where T: t, K: k, T.tType = K.kType {
}
```

### indirect 和嵌套 enum
如果要在枚举类型中嵌套使用当前类型的话，需要在枚举类型前加上关键字 `indirect`。

## 从 OC 到 Swift
### Selector
OC 中使用 `@selector` 获取关键字，Swift 中使用 `#selector` 获取一个方法暴露给 OC。在要暴露给 OC 的方法前需要添加 `@objc` 关键字：
```swift
@objc private func call(to: String){
}

let someMethod = #selector(call(to:))
NSTimer.scheduledTimerWithTimeInterval(1,target: self. selector: someMethod, userInfo: nil, repeats: true)
```

### 实例方法的动态调用
Swift 提供了一种方法替代直接调用实例方法。通过类型先取出某个实例方法的签名，然后再传递实例拿到实例需要调用的方法：
```swift
let f = MyClass.method
let object = MyClass()
let result = f(object)(1)
```
其实第一句相当于做了以下的字面量转换，也就是将这个方法放到了一个闭包中去：
```swift
let f = { (obj: MyClass) in obj.method }
```

这种写法只适用于实例方法，不适用于属性的 getter 和 setter 方法。另外，如果有多个同名但是函数签名不同的方法，或者与类方法同名的话，可以在获得方法签名的时候，显示的加上类型声明：
```swift
// 有同名类方法，那么获取的就是类方法
let f1 = MyClass.method
// 同f1，因为是类方法，所以不需要传入类对象
let f2: (Int) ->Int = MyClass.method
// 实例方法，需要传入类对象
let f3: (MyClass) -> (Int) = MyCLass.method
```

### 单例
Swift 可以无缝使用 GCD，但是 Swift3 中移除了 dispatch_once。可以使用 Swift 中有更简单的保证线程安全的方式 let：
```swift
class MyClass {
	static let shared = MyClass()
	private init() {}
}
```
最主要的就是将 init 方法私有化，这样外部无法调用

### 条件编译
Swift 中没有了宏的概念，不能使用 `#ifdef` 来检查某个符号是否通过宏编译。但是我们仍可以通过 `#if`，`#else`，`#endif` 来控制编译流程和内容。

为了自定义编译符号，我们需要在编译选项中进行设置，在项目的 Build Setting 中，找到 Swift Compiler - Custom Flags，然后再 Other Swift Flags 上加上 -D DEBUG 即可定义一个 DEBUG 宏。 

### 编译标记
OC 中一般使用 `#param mark - xxx` 的方式来标记代码区间。Swift 中可以使用 `// MARK: xxx` 进行标记。在冒号后面加上横杠 - ，这样导航栏中这个位置会再多显示一个横条。

还有包括 `// TODO:`，`// FIXME:` 的标记。

### @UIApplicationMain
OC 中会有一个 main 文件，调用 UIApplicationMain 的方法。Swift 中只有在 AppDelegate 上有一个 `@UIApplicationMain` 标记。这个标签将被标注的类作为委托，去创建一个 UIApplication 并启动整个程序。编译的时候，编译器将寻找这个标记的类，然后插入像 main 函数这样的模板代码。

### @objc 和 dynamic
可以对需要暴露给 OC 使用的任意地方(包括类，属性和方法等)的声明前加上 `@objc` 修饰符。这个步骤只需要对那些不是继承于 NSObject 的类型进行，如果使用 Swift 写的 class 是继承于 NSObject 的话，Swift 会自动为所有 **非 private** 类和成员加上 `@objc`。

 添加了 `@objc` 也并不意味着方法或者属性会变成动态派发，Swift 会将其优化为静态调用。如果需要动态调用，需要使用修饰符 `dynamic`

### 可选协议和协议拓展
Swift 的协议中没有可选项，所有方法默认必须实现。可以使用协议拓展，在可选的方法定义默认实现。

### 内存管理，weak 和 unowned
闭包中的弱引用，在函数类型前添加中括号：
```swift
{ [unowned self, weak someObject] (number: Int) -> Bool in
	return true
}
```

### @autoreleasepool
Swift 中也可以自建自动释放池：
```swift
autoreleasepool {
}
```

### 值类型和引用类型
Swift 中的数组是值类型，在频繁增减其中元素时，复制产生的消耗非常大。因此，在使用数组和字典时，如果数据增减频繁，使用 `NSMutableArray` 和 `NSMutableDictionary` 会比较好，容器内条目小并且数目稳定的时候，使用 Swift 的 `Array` 和 `Dictonary`。

### String 还是 NSString
String 和 NSString 可以无缝替换。但是最好使用原生的 String

### UnsafePointer
C 语言中的指针，将会在 Swift 中被转为 `UnsafePointer<Type>`类型：
```swift
// C 中
void method(const int *num) {
	printf("%d", *num);
}

// Swift 中
func method(_ num: UnsafePointer<CInt>) {
	print(num.pointee)
}
```
C 中的基础类型，Swift 遵循统一的命名规则，在基本类型前加上 C。

传入指针地址是都是在前面加上 `&`

### GCD 和延时调用
Swift 中无缝使用 GCD:
```swift
// 创建⽬标队列
let workingQueue = DispatchQueue(label: "my_queue")
// 派发到刚创建的队列中，GCD 会负责进⾏线程调度
workingQueue.async {
	// 在 workingQueue 中异步进⾏
	print("努⼒⼯作")
	Thread.sleep(forTimeInterval: 2)	// 模拟两秒的执⾏时间
	DispatchQueue.main.async {
		// 返回到主线程更新 UI
		print("结束⼯作，更新 UI")
	}
}
```
延迟调用：
```swift
let time: TimeInterval = 2
DispatchQueue.main.asuncAfter(deadline: DisaptchTime.now() + time) {
	print("2秒后输出")
}
```

### 获取对象类型
调用 `type(of: xxx)` 来获取对象类型

### 自省
向一个对象发出询问，确定它是不是属于某个类，这种操作叫做自省。Swift 中使用 `is` 相当于 OC 中的 `isKindOfClass:`
```swift
let obj: AnyObject = ClassB()
if (obj is ClassB) {
	print("属于 ClassB")
}
```

### KVO
暂时还没有比较好的 KVO 的纯 Swift 方案，可以使用属性观察实现类似替代。现有库 Observable-Swift 对这一思路进行了封装。

### 判等
Swift 中，`==` 是你一个操作符的声明，在 `Equatable` 中声明了这个操作符的协议方法：
```swift
protocol Equatable {
	func ==(lhs: Self, rhs: Self) -> Bool
}
```

对于自建类型，如果想要能够使用 `==` 判等，需要实现 `Equatable` 协议。Swift 中的操作符都是全局的，我们不需要在拓展中实现这个重载方法，要在全局的 scope 里定义这个方法。使用的时候通过类型推断调用正确的 `==` 重载：
```swift
class TodoItem {
	let uuid: String
	
	init(uuid: String) {
		self.uuid = uuid
	}
}

extension TodoItem: Equatable {
}

func ==(lhs: TodoItem, rhs: TodoItem) -> Bool {
	return lhs.uuid == rhs.uuid
}
```

Swift 中使用  `===` 判断两个类型是否是同一个地址。

### Options
OC 中的 `NS_ENUM` 与 Swift 中的 `enum` 对应。那么 OC 中的 `NS_OPTIONS` 呢？对应于满足 `OptionSetType` 协议的 `struct` 类型，以及一组静态的 `get` 属性。

以动画选项为例：
```swift
public struct UIViewAnimationOptions : OptionSetType {
	public init(rawValue: UInt)	 
	static var layoutSubviews: UIViewAnimationOptions = UIViewAnimationOptions(rawValue: 0)
	static var allowUserInteraction: UIViewAnimationOptions UIViewAnimationOptions(rawValue: 1)
//...
	static var transitionFlipFromBottom: UIViewAnimationOptions UIViewAnimationOptions(rawValue: 1 << 1)
}
```
使用的时候直接使用类属性，options 接受一个 `OptionSetType` 的数组：
```swift
UIView.animate(withDuration: 0.3,
	delay: 0.0,
	options: [.curveEaseIn, .allowUserInteraction],
	animations: {},
	completion: nil)
```

### 数组 enumerate
OC 中需要获得数组的下标索引一般使用方法 `enumerateObjectsUsingBlock:`。Swift 中有一个替代方法：
```swift
var result = 0
for (idx, num) in [1,2,3,4,5,6].enumerated() {
	result += num
	if idx ==2 {
		break
	}
}
```

### delegate
Swift 中一般的 protocol 不能作为 delegate 的类型，因为 struct 和 enum 也能遵循 protocol，但是它们不是通过引用计数来管理内存的，所以这样就会导致 delegate 不能使用 weak 修饰。

所以一个能作为 delegate 类型的 protocol 必须是只能由 class 实现的。需要在声明的名字后面加上 class:
```swift
protocol MyClassDelegate: class{
	func method()
}
```

### Associated Object
OC 中可以通过 Associated Object 在 Category 中给已有类添加成员变量。Swift 中也可以通过 Associated Object 在 extension 中给已有类添加成员变量：
```swift
private var key: Void?

extension MyClass {
	var title: String? {
		get {
			return objc_getAssociatedObject(self, &key) as? String
		}

		set {
			objc_setAssociatedObject(self, &key, newValue, .OBJC_ASSOCIATION_RETAIN_NONATOMIC)
		}
	}
}

let a = MyClass()
a.title = "123"
```
key 的乐行为 `Void?`，通过 & 操作符取地址并作为 `UnsafePointer<Void>` 类型被传入。

### Lock
OC 中常用 `@sychronized` 作为锁，但是 Swift 中不存在了。如果想要 lock 一个变量的话需要如下：
```swift
func myMethod(anObj: AnyObject!) {
	objc_sync_enter(anObj)
	// 在 enter 和 exit 之间 anObj 不会被其他线程改变
	objc_sync_exit(anObj)
}
```

可以写一个全局方法做适当封装，使其类似于 OC：
```swift
func synchronized(_ lock: AnyObject, closure: () -> ()) {
	objc_sync_enter(lock)
	closure()
	objc_sync_exit(lock)
}

func myMethodLocked(anObj: AnyObject!) {
	synchronized(anObj) {
		// 在括号内 anObj 不会被其他线程改变
	}
}
```






















