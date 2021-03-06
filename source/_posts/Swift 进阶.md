title: 《Swift 进阶》笔记
date: 2017/10/18 14:07:39  
categories: iOS
tags: 

	- Swift


------

这书读的累觉不爱

<!--more-->

## Swift 风格
- 使用类型推断。省略显而易见的类型有助于提高可读性
- 优先使用结构体。只有确定要使用类的特性或者引用语义的时候才使用类
- 避免使用强制解包或隐式强制解包，除非能确定非 nil
- 试着使用 map 和 reduce 代替 for 循环
- 除非需要改变某个值，否则尽量用 let
- 尽量不要使用 self。使用 self 是一个清晰的信号，表明闭包会捕获 self

## 內建集合
### 数组
#### 数组和可变性
- 值类型并不是赋个值就创建一个内存空间。值类型在不变的情况下还是共享内存的。只有其中一个变化的时候才会分配新的内存。这样能减少复制带来的性能问题。
#### 数组和可选值
- 应该尽量不使用强制解包。养成习惯后会导致你强制解包原本不该解包的东西。
#### 数组变形
- map 对每个元素进行操作，闭包返回处理过的值，最后得到一个处理完成的数组
- filter 对每个元素进行判断，闭包返回 Bool 值，最后得到一个所有符合要求元素的数组
- reduce 对所有元素进行运算，闭包返回的值和下一个元素作为下一次运算的输入，最后返回一个所有元素的处理结果值。
- flatMap 针对于每一次 map 闭包处理后返回的是一个数组，即最终返回一个元素是数组的数组的情况。可以使用 flatMap 将内部数组展开，最后得到一个包含所有元素的数组。
- forEach 的 return 并不能返回给外部。所以一般不要使用 forEach，而使用 for…in
#### 数组类型
- 当我们通过数组下标获得数组中的一段时，获得的不是一个新的 Array，而是 ArraySlice。ArraySlice 是 Array 的一种表现方式，没有真正的 alloc。相当于在原数组的基础上，多加了保存两个 index 的变量，也就是说，index 不是从 0 开始的。

### Range
- range 表示一个区间，但是并不是所有范围都是序列或者集合类型。只有那些步长为整数的才是集合类型。

## 集合类型协议
### 序列
- 一个集合 Collection 就是一个序列 Sequence，这要满足 `Sequence` 协议。满足 `Sequence` 协议要求实现一个返回**迭代器** 实例的方法 `makeIterator()`
#### 迭代器
- 迭代器要满足 `IteratorProtocol` 协议，要求实现 `next()` 方法。
- for 循环的基本过程是，集合调用 `makeIterator()` 返回一个迭代器实例，迭代器在创建的时候将集合传入。不停调用迭代器的 `next()` 方法，返回集合中的元素。
- 绝大多数迭代器都是值语义，复制的时候两个迭代器分别工作。AnyIterator 是一个用来封装其他迭代器的迭代器，它是引用语义的。AnyIterator 相当于一个盒子，盒子被引用复制，盒子里的迭代器将被共享。
- 创建一个序列的方式:
	1. AnySequence 提供了一个初始化方法，接收一个返回值为迭代器的函数。
	2. 通过 `sequence(first:next:)` 或者 `sequence(state:next:)` 两个方法。除了 next 要求传入一个生成后续值的闭包外，前者传入一个初始值，后者传入一个元组。 
#### 无限序列	
- 序列可以是无限的，而集合则是有限的。这是一个重要区别
- 可以通过 `let value = someSequence.prefix(10)` 的方式获取序列的前十个元素。然后 `Array(value)` 生成一个数组。
#### 不稳定序列
- 不只是数组这样的才叫序列。网络流，文件流也可以当做序列。序列并不保证可以被多次遍历都一样。
- 如果序列遵守 `Colleciton` 协议，那么就是集合。集合是稳定的。这是两者的另一个差别。
#### 序列和迭代器的关系
- 需要在序列中提供一个迭代器的意义在于，每次遍历的时候都生成一个新的迭代器，这样内部状态独立。否则内部状态随着 for 循环改变。
#### 子序列
- Sequence 提供了一个关联类型 `associatedtype SubSequence`。在返回原序列切片的操作中，SubSequence 被用作返回值的类型，比如 ArraySlice。

### 集合类型
- 集合类型指的是可以多次遍历的稳定的序列
- `Collection` 协议是建立在 `Sequence` 协议上的
- 本节后部分加起来实现了一个 FIFO 队列
#### 如何实现一个 FIFO 队列
- 创建一个 FIFO 队列的结构体，然后提供  FIFO 队列的特有 enqueue、dequeue 方法。
- 若你想自己创建一个 Collection 类型，由于 Collection 提供了很多默认实现，比如迭代器就有默认实现，你只要自己实现 startIndex、endIndex、根据索引访问特定位置，以及下标方法。
- 实现 `ExpressibleByArrayLiteral` 协议，可以用数组字面量( 形如[1,2,3] ) 的形式创建队列，数组也实现了这个协议，实现了这个协议的不一定是数组。

### 切片
- 切片和原来的集合共享存储缓冲区。也就是说，切片不会单独取出内存，它只是标记了集合的开始和结束位置。

## 可选值
### 通过枚举解决魔法数的问题
- 可选是一个枚举类型
```swift
enum Optional<Wrapped> {
	case none
	case some(wrapped)
}
```

### 可选值概览
#### if let 和 while let
- 通过可选绑定 `if let` 获得一个非空值。`while let` 代表一个当遇到 nil 时终止的循环。
- for…in 其实就是一个 `while let` 循环。
#### 双重可选值
- 可选值的值仍然是一个可选值 `optional<Optional<Int>>` 即 `Int??`
#### if var 和 while var
- 获取一个能在作用域内使用的副本的简介，并不会改变原来的值。
#### 解包后的可选值的作用域
- 如果使用 guard 语句要求必须有 return ，在循环中的话则是 break 和 continue
- 一个函数的返回值如果是 `Never`，那么表示什么也不返回。一般我们定义的方法不会返回 `Never`
#### 可选链
- 如果某个方法的返回值可能为空，那么可选链的调用上需要加上 `?`；反之，则不需要。
#### nil 合并运算符
- `a ?? b` 表示 a 不为空位 a，为空则为 b。类似于三元运算符
- `??` 可以结合 `if let`：
```swift
// “，”表示与，需要同时满足不为空
if let n = i,let m =j {}
// “??”表示或，只要其中一个不为空
if let n = i ?? j {}
```

## 结构体和类
### 结构体
- 使用 `let` 定义的的结构体，其中 `var` 定义的属性也是不能被改变的。需要注意的是，结构体中包含引用属性，只要引用属性没有指向新的地址，那么改变引用属性中的属性是没有问题的。
- 如果我们自定义了一个结构体初始化方法，Swift 就不会自动生成默认初始化方法了
- 初始化方法可以放在 extension 中
- 当我们改变结构体中的某个属性的时候，结构体的 `didSet` 方法会被触发。因为结构体的属性改变导致新建了一个结构体。如果是类，则不会触发
- 数组是结构体，因此当我们改变数组中的某个元素时，数组的 `didSet` 方法也会触发
- 要改变结构体其中属性的值，需要在方法前加上 `mutating` 标识。加了标识的方法不能被 `let` 定义的结构体对象调用。
- Swift 会自动将属性的 setter 标记为 `mutating`。因此，你无法调用 `let` 的 setter。
- 结构体的有一些方法有两个版本，比如数组的 `sort()` 和 `sorted()`，前者一类使用祈使动词短语的是 `mutating` 方法，在原对象上修改；后者返回一个新对象
- `mutating` 关键字如何工作？它将隐式的 self 参数变为 `inout` 可变。

### 写时复制
- Swift 中集合类型实现了写时复制，即赋值时两个集合指向内存中的同一个位置，只有在改变其中一个集合时，才会复制数组，再进行修改。这样能减少结构体频繁的复制操作带来的性能损耗
- 写时复制不是编译器的优化，而是要自己实现的。
#### 写时复制(昂贵方式)
- 对于自己实现的结构体，在方法中复制自己，然后对复制的对象进行修改返回
- 对于引用类型，需要将其包装在一个结构体内，作为一个私有属性。创建一个计算属性，get 方法创建这个引用类型的复制并覆盖原属性，然后返回。在需要写时复制的方法中操作计算属性即可。
#### 写时复制(高效方式)
- 上面存在的问题是，如果你对象只有唯一引用，还是会在修改前复制对象。我们需要先使用 `isKnownUniquelyReferenced` 函数来检查引用的唯一性。

### 闭包和可变性
- **Swift 的结构体一般被存储在栈上**，而非堆上，这是 Swift 的一种优化。默认情况下结构体存储在堆上，绝大多数情况下，优化会生效，将结构体存储到栈上。但是当结构体变量被一个闭包引用的时候，优化失效，结构体将存储在堆上。因为如果存在栈上，当前作用域结束就会销毁，但是被闭包引用了，需要在未来的某个时候用到，所以必须要放在堆上。

## 函数
### inout 函数和可变方法
- Swift 中 `inout` 参数前面使用的 `&` 符号不是传递引用。事实上，`inout` 做的事是将值传回去，替代原来的值。
- 可以在嵌套函数的外部函数中使用 `inout` 参数，但是不能让 `inout` 参数逃逸。因为逃逸了就不知道应该什么时候复制回去了。

### 计算属性
- 延迟存储属性在第一次被访问时，闭包被执行。
- 延迟存储属性闭包中访问外部成员的时候要明确使用 self
- 我们可以将属性的 `didSet` 和 `IBOutlet` 配合使用，在 IBOutlet 被链接的时候运行代码，做一些视图配置。

### 自动闭包和逃逸闭包
- 使用 `@autoclosure` 标注自动为一个参数创建闭包。(我觉得这样代码可读性很差)
- 逃逸闭包需要加上 `@escaping` 标注，否则编译器不允许保存这个闭包。

## 字符串
### 字符串和集合
- String 符合 Collection 协议的所有要求，但是 String 不是一个集合。String 提供了 characters 这个集合访问字符串
- 数组上的切片不是 Array 而是 ArraySlice。而 String 的切片类型则是自身。

###  String 的内部结构
- 结构体 String 内部包含了一个 `StringCore` 类型的结构体，`StringCore` 包含了三个属性：`_baseAddress` 栈上或者堆上的位置；`_countAndFlags` 长度；`_owner`负责写时赋值和内存回收的对象。
- 字符串 split 不会做大量的拷贝，而是持有字符串某个切片。这会导致即使切片只有几个字符大小，还是会导致整个字符串内存无法释放。

### CustomStringConvertible
- 如果想要自定义 print 的输出，可以在一个 extension 中实现 `CustomStringConvertible` 协议，重写 `description` 属性的 get 方法。

## 错误处理
### Result 类型
- Result 类型和可选值非常类似。Result 类型也是两个成员组成的枚举，一个代表失败情况，并关联了具体的错误值；一个代表成功情况，它关联了一个泛型参数。
- 这样，对于可能抛出错误的方法，我们使其返回一个 Result 类型。然后判断到底是成功枚举还是失败枚举，然后取出关联值。

### 抛出和捕获
- Swift 并没有使用返回 Result 的方式处理失败，而是将方法标记为 `throws`。Result 是作用在类型上的，而 throws 是作用于函数的。对于每个可以抛出的函数，编译器会验证调用者有没有捕获错误，或者把错误向上传递。
- 包含 throws 的函数是这样的：
```swift
func contents(ofFIle filename: String) throws -> String
```
- 处理的时候我们要将会抛出异常的函数标记为 `try`。try 的目的是为了强调调用者知道函数会抛出错误。
- 我们需要通过 do/catch 来处理错误，或者把当前函数也标记为 throws。如果使用 catch，我们用模式匹配的方式获取某个特定的错误或者所有错误，然后在最后 catch-all 中处理所有错误。在 catch-all 中，变量 error 可以直接使用：
```swift
do {
	let result = try contents(ofFile:"input.txt")
	pint(result)
} catch FileError.fileDoesNotExist {
	print("File not found")
} catch {
	print(error)
}
```
- 除了系统提供的错误，我们还可以自己定义错误类型。创建一个继承于 `Error` 的枚举：
```swift
enum ParseError: Error {
	case wrongEncoding
	case warning(line: Int, message: String)
}
```
- Swift 的错误是无类型的，我们只将函数标记为 throws，但是没有指出应该抛出那个类型的错误。所以我们最好在函数定义的时候规范的加好注释。

### 错误和函数参数
- 在函数封装的时候可能会遇到一种情况：一个函数的函数参数可能会抛出错误也可能不会抛出错误。如果这个函数使用 try 调用这个函数参数，那么这个函数参数不会抛出错误的版本也需要加上 throws。
- 可以通过 `rethrows` 标记代替 throws。使用这个标记的函数还是需要对于函数参数使用 try，但是如果函数参数不会抛出错误，那么也不强制函数参数用 throws 标记了。

### 使用 defer 进行清理
- Swift 中的 `defer` ，围绕的代码块一定会在函数返回时被执行（就是相当于在 return 之后再执行一些操作）
- defer 类似于其他语言的 finally，但是 defer 不只是用于错误处理，可以将 defer 放在代码块的任意地方。
- 如果同一个作用域里使用多个 defer，那么它们会被逆序执行，可以把它想象成一个栈。用意在于，比如我们开启数据库然后连接，用 defer 就能自然的先关闭连接，再关闭数据库。

### 错误和可选值
- 可以使用 `try?` 来忽略函数返回的错误，也就是说不用再 do-catch 了。

### 高阶函数和错误
- 在异步操作，比如回调函数中并不能使用 throws，因为这并不表示函数里执行的内容可能失败，而是指函数本生会失败。这个时候，我们就要使用 Result 替代了。

## 泛型
### 泛型的工作方式
- Swift 为泛型引入一个中间层，当遇到泛型类型的值时，它会将其包装到一个容器中。
- 对于泛型类型的参数，编译器维护了一系列一个或多个目击表，其中包含一个值目击表，以及类型上每个协议约束一个协议目击表。目击表将被用来将运行时的函数调用动态派发到正确的实现
- 任意的泛型类型，总会存在值目击表。它包含了指向内存申请，复制和释放这些类型的基本操作的指针(就是创建泛型类型实例的指针)
- 关于协议目击表，对于这个协议声明的每个方法或者属性，协议目击表中都含有一个指针，指向该满足协议的类型中的对应实现。在泛型函数中对这些方法的每次调用，都会在运行时通过目击表转换为方法派发。(每个方法属性都有对应指针)
- 协议目击表提供了一组映射关系，可以知道协议和具体的协议功能的映射关系。
- 泛型参数的容器只包含值存储，目击表被单独存储。这样泛型函数中同样类型的其他变量就可以共享这个目击表了

### 泛型特化
- 因为泛型是动态派发，所以会有一些性能损耗。不过 Swift 可以通过泛型特化的方式避免额外开销。
- 泛型特化指的是编译器按照具体的参数类型，将泛型类型或者函数进行复制。(就是在编译的时候，用会用到的特定的类型代替泛型)

## 协议
- 一个由关联类型的协议不能作为类型，只能作为泛型约束
- 当我们通过协议类型创建一个变量的时候，这个变量会被包装到一个叫做存在容器的盒子中。
- 对于普通协议，会使用**不透明存在容器**，不透明存在容器中含有一个存储值的缓冲区，一些元数据，以及若干目击表。
- 对于只适用于类的协议，会有一个叫做**类存在容器**的特殊存在容器。





























