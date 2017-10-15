title: Swift 3.1 语法学习（七）
date: 2017/2/22 14:07:39  
categories: iOS
tags: 

	- Swift


------

接着上一篇

<!--more-->

## 访问控制

### 模块和源文件

模块指的是独立的代码单元，框架或应用程序会作为一个独立的模块来构建和发布。在 Swift 中，一个模块可以使用 `import` 关键字导入另外一个模块。在 Swift 中，Xcode 的每个 target（例如框架或应用程序）都被当作独立的模块处理。

源文件就是 Swift 中的源代码文件，它通常属于一个模块，即一个应用程序或者框架。

### 访问级别

- `open`:提供最高的权限，模块内外都能访问和重写。和 `public` 的区别下面说
- `public`:可以访问同一模块源文件中的任何实体，在模块外也可以通过导入该模块来访问源文件里的所有实体，但是不允许重写。
- `internal`:可以访问同一模块源文件中的任何实体，但是不能从模块外访问该模块源文件中的实体。
- `fileprivate`:限制实体只能在所在的源文件内部使用，即使不在同一个作用域内。
- `private`：提供最低的权限，只有在同一个作用域内可访问。

`open` 和 `public` 在作用上很像。`open` 则是为了弥补 `public` 语义上的不足。通过 `open` 和 `public` 标记区别一个元素在其他 module 中是只能被访问还是可以被 override。现在的 `public` 在 module 内可以被 override，在被 import 到其他地方后其他用户使用的时候不能被 override，而 `open` 则可以。下面的例子很好的解释了：

```swift
/// ModuleA:

// 这个类在ModuleA的范围外是不能被继承的，只能被访问
public class NonSubclassableParentClass {

    public func foo() {}

    // 这是错误的写法，因为class已经不能被继承，
    // 所以他的方法的访问权限不能大于类的访问权限
    open func bar() {}

    // final的含义保持不变
    public final func baz() {}
}

// 在ModuleA的范围外可以被继承
open class SubclassableParentClass {
    // 这个属性在ModuleA的范围外不能被override
    public var size : Int

    // 这个方法在ModuleA的范围外不能被override
    public func foo() {}

    // 这个方法在任何地方都可以被override
    open func bar() {}

    ///final的含义保持不变
    public final func baz() {}
}

/// final的含义保持不变
public final class FinalClass { }
/// ModuleB:

import ModuleA

// 这个写法是错误的，编译会失败
// 因为NonSubclassableParentClass类访问权限标记的是public，只能被访问不能被继承
class SubclassA : NonSubclassableParentClass { }

// 这样写法可以通过，因为SubclassableParentClass访问权限为 `open`.
class SubclassB : SubclassableParentClass {

    // 这样写也会编译失败
    // 因为这个方法在SubclassableParentClass 中的权限为public，不是`open'.
    override func foo() { }

    // 这个方法因为在SubclassableParentClass中标记为open，所以可以这样写
    // 这里不需要再声明为open，因为这个类是internal的
    override func bar() { }
}

open class SubclassC : SubclassableParentClass {
    // 这种写法会编译失败，因为这个类已经标记为open
    // 这个方法override是一个open的方法，则也需要表明访问权限
    override func bar() { } 
}

open class SubclassD : SubclassableParentClass {
    // 正确的写法，方法也需要标记为open
    open override func bar() { }    
}

open class SubclassE : SubclassableParentClass {
    // 也可以显式的指出这个方法不能在被override
    public final override func bar() { }    
}
```

在 Swift3 中 `private` 修饰的变量 extension 中是不能访问的，因为不在一个作用域中。但是 Swift4 中修改了这一个限制。现在在 extension 中也是能够访问 private 的属性的。

### 访问级别基本原则

Swift 中的访问级别遵循一个基本原则：**不可以在某个实体中定义访问级别更高的实体**。

> 看起来很简单的一个原则，但是要很注意的。

### 默认访问级别

对于非 `private` 的类，你不为代码中的实体显式指定访问级别时，它们默认为 `internal` 级别。对于 `private` 的类，没必要显式指定访问级别，默认为 `private`。

### 元组类型

**元组的访问级别将由元组中访问级别最严格的类型来决定**。例如，如果你构建了一个包含两种不同类型的元组，其中一个类型为 `internal` 级别，另一个类型为 `private` 级别，那么这个元组的访问级别为 `private`。

> 元组不同于类、结构体、枚举、函数那样有单独的定义。元组的访问级别是在它被使用时自动推断出的，而无法明确指定。

### 函数类型

**函数的访问级别根据访问级别最严格的参数类型或返回类型的访问级别来决定**。但是，**如果这种访问级别不符合函数定义所在环境的默认访问级别，那么就需要明确地指定该函数的访问级别**。

下面的例子定义了一个名为 `someFunction` 的全局函数，并且没有明确地指定其访问级别。也许你会认为该函数应该拥有默认的访问级别 `internal`，但事实并非如此。事实上，如果按下面这种写法，代码将无法通过编译：

```swift
func someFunction() -> (SomeInternalClass, SomePrivateClass) {
       // 此处是函数实现部分
}
```

根据元组访问级别的原则，该元组的访问级别是 `private`（元组的访问级别与元组中访问级别最低的类型一致）。因为该函数返回类型的访问级别是 `private`，所以你必须使用 `private` 修饰符，明确指定该函数的访问级别：

```swift
private func someFunction() -> (SomeInternalClass, SomePrivateClass) {
       // 此处是函数实现部分
}
```

将该函数指定为 `public` 或 `internal`，或者使用默认的访问级别 `internal` 都是错误的，因为如果把该函数当做 `public` 或 `internal` 级别来使用的话，可能会无法访问 `private` 级别的返回值。

### 枚举类型

枚举成员的访问级别和该枚举类型相同，你不能为枚举成员单独指定不同的访问级别。

```swift
public enum CompassPoint {
       case North
       case South
       case East
       case West
}
```

#### 原始值和关联值

枚举定义中的任何原始值或关联值的类型的访问级别至少不能低于枚举类型的访问级别。例如，你不能在一个 `internal` 访问级别的枚举中定义 `private` 级别的原始值类型。

### 嵌套类型

如果在 `private` 级别的类型中定义嵌套类型，那么该嵌套类型就自动拥有 `private` 访问级别。如果在 `public` 或者 `internal` 级别的类型中定义嵌套类型，那么该嵌套类型自动拥有 `internal` 访问级别。如果想让嵌套类型拥有 `public` 访问级别，那么需要明确指定该嵌套类型的访问级别。

### 子类

子类的访问级别不得高于父类的访问级别。例如，父类的访问级别是 `internal`，子类的访问级别就不能是 `public`。

此外，你可以**在符合当前访问级别的条件下**重写任意类成员（方法、属性、构造器、下标等）。可以通过重写为继承来的类成员提供更高的访问级别：

```swift
public class A {
       fileprivate func someMethod() {}
}

internal class B: A {
       override internal func someMethod() {}
}
```

注意提供更高访问级别需要注意符合当前访问级别，上面 `someMethod` 是 `fileprivate` 的，所以可以重写，但是下面就是错的：

```swift
public class A {
		//不能这么写
       private func someMethod() {}
}

internal class B: A {
       override internal func someMethod() {}
}
```

### 常量、变量、属性、下标

常量、变量、属性不能拥有比它们的类型更高的访问级别。例如，你不能定义一个 `public` 级别的属性，但是它的类型却是 `private` 级别的。同样，下标也不能拥有比索引类型或返回类型更高的访问级别。

如果常量、变量、属性、下标的类型是 `private` 级别的，那么它们必须明确指定访问级别为 `private`：

```swift
private var privateInstance = SomePrivateClass()
```

### 协议

如果想为一个协议类型明确地指定访问级别，在定义协议时指定即可。这将限制该协议只能在适当的访问级别范围内被采纳。

**协议中的每一个要求都具有和该协议相同的访问级别**。你不能将协议中的要求设置为其他访问级别。这样才能确保该协议的所有要求对于任意采纳者都将可用。

> 如果你定义了一个 `public` 访问级别的协议，那么该协议的所有实现也会是 `public` 访问级别。这一点不同于其他类型，例如，当类型是 `public` 访问级别时，其成员的访问级别却只是 `internal`。

#### 协议继承

如果定义了一个继承自其他协议的新协议，那么新协议拥有的访问级别最高也只能和被继承协议的访问级别相同。例如，你不能将继承自 `internal` 协议的新协议定义为 `public`协议。

#### 协议一致性

一个类型可以采纳比自身访问级别低的协议。例如，你可以定义一个 `public` 级别的类型，它可以在其他模块中使用，同时它也可以采纳一个 `internal` 级别的协议，但是只能在该协议所在的模块中作为符合该协议的类型使用。

采纳了协议的类型的访问级别取它本身和所采纳协议两者间最低的访问级别。也就是说如果一个类型是 `public` 级别，采纳的协议是 `internal` 级别，那么采纳了这个协议后，该类型作为符合协议的类型时，其访问级别也是 `internal`。

如果你采纳了协议，那么实现了协议的所有要求后，你必须确保这些实现的访问级别不能低于协议的访问级别。例如，一个 `public` 级别的类型，采纳了 `internal` 级别的协议，那么协议的实现至少也得是 `internal` 级别



## 补充

###  断言与先决条件

断言和先决条件用来检查某些条件是否满足。断言和先决条件不同之处在于什么时候做检查：断言只在 debug 时候检查，先决条件则在 debug 和生产构建中都生效。生产构建中，断言的条件不会被计算，因此无需担心新能。

####  断言的使用方式

使用 `assert(_:_:)` 来进行断言：

```swift
let age = -3
assert(age >= 0, "A person's age cannot be less than zero")
```

如果不需要检查条件直接抛出异常，那么可以使用 `assertionFailure(_:)`

```swift
if age > 10 {
    print("You can ride the roller-coaster or the ferris wheel.")
} else if age > 0 {
    print("You can ride the ferris wheel.")
} else {
    assertionFailure("A person's age can't be less than zero.")
}
```

#### 强制先决条件

使用上和断言类似，使用 `precondition(_:_:)` 也有 `preconditionFailure(_:)` 来强制抛出：

```swift
precondition(index > 0, "Index must be greater than zero.")
```



### 泛型中的 Where

泛型中的 Where 主要用来约束泛型或者协议中的关联类型。在类型、函数、拓展、协议中都可以使用。

#### 类型和函数中

类型或函数中本来是可以直接定义泛型的，可以直接对泛型做约束：

```swift
protocol a {
    associatedtype itemType
}

protocol b {
    associatedtype itemType
}

class s {
}

class S<T: a, K: b> : s {
}
```

但是这样，我们无法约束协议 a 和 b 中的关联类型。所以我们可以用 where 分句：

```swift
class S<T: a, K: b> : s where T.itemType == K.itemitemType {
}
或者

class S<T,K> : s where T: a, K: b, T.itemType == K.itemType {
}
```

泛型实现的协议以及协议中的关联类型都可以用 where 限制

#### extension 中

由于 extension 中不能定义泛型，只能使用泛型，所以不能直接在 `<>` 中直接约束泛型了，但是仍然可以使用 Where：

```swift
extension Stack where Element: Equatable {
    func isTop(_ item: Element) -> Bool {
        guard let topItem = items.last else {
            return false
        }
        return topItem == item
    }
}
```

#### 协议中

协议中不适用 `<>` 定义泛型，而是使用关联类型。如果要约束关联类型，那么也可以使用 where：

```swift
protocol pro {
}
protocol Container where Item: pro { 
    associatedtype Item
    mutating func append(_ item: Item)
    var count: Int { get }
    subscript(i: Int) -> Item { get }
    
    associatedtype Iterator: IteratorProtocol where Iterator.Element == Item
    func makeIterator() -> Iterator
}
```

 我们可以看到：**where 都是将泛型或者关联类型约束为实现某个协议而不是继承某个类，原因很明显，因为明确了某个类了就不需要泛型以及关联类型了**。



### 条件判断的 where

where 还可以用在条件判断上，比如之前在学习 Swith 的使用用到的，where 接受一个返回 Bool 类型的分句，用来做条件判断：

```swift
case let(x, y) where x == y:
```

在 for…in 循环中，同样可以使用 where：

```swift
for element in 1...5 where element >3 {
    print(element)
}
// 4,5
```

相当于：

```swift
for element in 1...5 {
    if element > 3 {
        print element
    }
}
```

