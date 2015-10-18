> 翻译：[shockinglee](https://github.com/shockinglee)

> 校对：[shanyimin](https://github.com/shanyimin) [ChildhoodAndy](https://github.com/dabing1022) [Phenmod](https://github.com/Phenmod)

# 与 C 的 API 交互

本节包含内容：

- [基本数据类型（Primitive Types）](#primitive_types)
  
- [枚举（Enumerations）](#enumerations)

- [选项集（Option Sets）](#Option Sets)

- [联合体（Unions）](#Unions)
 
- [指针（Pointer）](#pointer)

- [全局常量（Global Constants）](#global_constants)

- [预处理指令（Preprocessor Directives）](#preprocessor_directives)

作为与 Objective-C 语言的互用性的一部分，Swift 对一些 C 语言的类型和特性保持了兼容性。如果你的代码有需要，Swift 也提供了一些方式让你和常见的 C 语言设计以及模式协同工作。

<a name="primitive_types"></a>
## 基本数据类型（Primitive Types）

Swift 提供了一些与 C 语言基本类型，例如`char`,`int`,`float`和`double`等类型的对应类型。然而，这些类型和 Swift 核心基本类型之间不能进行隐式转换，例如`Int`。因此，你的代码只有在必要时才应使用这些类型，其它任何可能的情况下都应该使用`Int`。

| C 类型 | Swift 类型 |
| ------ | ------ |
| bool | CBool |
| char, signed char | CChar |
| unsigned char | CUnsignedChar |
| short | CShort |
| unsigned short | CUnsignedShort |
| int | CInt |
| unsigned int | CUnsignedInt |
| long | CLong |
| unsigned long | CUnsignedLong |
| long long | CLongLong |
| unsigned long long | CUnsignedLongLong |
| wchar_t | CWideChar |
| char16_t | CChar16 |
| char32_t | CChar32 |
| float | CFloat |
| double | CDouble |

<a name="enumerations"></a>
## 枚举（Enumerations）

任何用宏`NS_ENUM`来声明的 C 风格枚举，都会被 Swift 导入为一个 Swift 枚举。无论枚举是在系统框架还是在自己的代码中定义的，当它们导入到 Swift 时，它们的前缀名将被截去。

例如，看这个 Objective-C 枚举的声明：

```Objective-C
typedef NS_ENUM(NSInteger, UITableViewCellStyle) {
	UITableViewCellStyleDefault,
    UITableViewCellStyleValue1,
    UITableViewCellStyleValue2,
    UITableViewCellStyleSubtitle
};
```

在 Swift 中，它被导入后会像这样：

```Swift
enum UITableViewCellStyle: Int {
	case Default
    case Value1
	case Value2
    case Subtitle
}
```

当你需要使用一个枚举值时，使用以点（.）开头的枚举变量名：

```Swift
let cellStyle: UITableViewCellStyle = .Default
```

<a name="Option Sets"></a>
## 选项集（Option Sets）

对于使用宏`NS_OPTIONS`声明的 C 风格枚举，Swift 会把它导入为一个 Swfit 选项集。选项集像枚举一样，会把前缀截掉，只剩下选项值名称。

例如，看这个 Objective-C 选项集的声明：

```Objective-C
typedef NS_OPTIONS(NSUInteger, NSJSONReadingOptions) {
	NSJSONReadingMutableContainers = (1UL << 0),
	NSJSONReadingMutableLeaves     = (1UL << 1),
	NSJSONReadingAllowFragments    = (1UL << 2)
};
```

在 Swift 中，它被导入后会像这样：

```Swift
struct NSJSONReadingOptions : OptionSetType {
	init(rawValue: UInt)
    
    static var MutableContainers: NSJSONReadingOptions { get }
    static var MutableLeaves: NSJSONReadingOptions { get }
    static var AllowFragments: NSJSONReadingOptions { get }
}
```

在 Objective-C 中，一个选项集是一些整数值的位掩码。你可以使用按位或操作符`|`来组合选项值，使用按位与操作符`&`来检测选项值。创建一个选项集，可以使用常量值或者表达式。一个空的选项集使用常数`0`来表示。

在 Swift 中，选项集使用一个遵守`OptionSetType`协议的结构体来表示，其中每个选项值都是一个静态变量。选项集类似于 Swift 的集合类型`Set`，你可以用`insert(_:)`或者`unionInPlace(_:)`方法来添加选项值，用`remove(_:)`或者`subtractInPlace(_:)`方法来删除选项值，用`contains(_:)`方法来检测选项值。创建一个选项集的值可以使用一个数组字面量，访问选项值像枚举一样也用点（`.`）开头。创建一个空的选项集可以使用一个空的数组字面量，也可以调用默认初始化函数。

```Swift
let options: NSDataBase64EncodingOptions = [
	.Encoding76CharacterLineLength,
    .EncodingEndLineWithLineFeed
]
let string = data.base64EncodedStringWithOptions(options)
```

<a name="Unions"></a>
## 联合体（Unions）

Swift 仅部分支持 C 的`union`类型。在导入混有 C 的联合体或者位段（bitfields）的类型时，例如 Foundation 的`NSDecimal`类型，Swift 不能使用不支持的字段。但是，参数和/或返回值为这些类型的 C 和 Objective-C 的 API 是能够在 Swift 中使用的。

<a name="pointer"></a>
## 指针（pointer）

Swift 尽可能避免让你直接使用指针。然而，当你需要直接操作内存的时候，Swift 也为你提供了多种指针类型。下面的表使用`Type`作为类型名称的占位符。

对于返回类型，变量和参数，遵循如下映射：

| C 句法 | Swift 句法 |
| ------ | ------ |
| const Type * | UnsafePointer\<Type\> |
| Type * | UnsafeMutablePointer\<Type\> |

对于类类型，遵循如下映射：

| C 句法 | Swift 句法 |
| ------ | ------ |
| Type * const * | UnsafePointer\<Type\> |
| Type * __strong * | UnsafeMutablePointer\<Type\> |
| Type ** | AutoreleasingUnsafeMutablePointer\<Type\> |

### 常量指针（Constant Pointers）

当一个函数声明为接受`UnsafePointer<Type>`参数时，该函数可以接受下列任意一种类型作为参数：

* `nil`，作为空指针传入；
* 一个`UnsafePointer<Type>`，`UnsafeMutablePointer<Type>`，或者`AutoreleasingUnsafeMutablePointer<Type>`的值，在必要情况下会转换成`UnsafePointer<Type>`的值；
* 一个`String`类型的值，如果`Type`是`Int8`或者`UInt8`的话。该字符串会自动在一个缓冲区内被转换为 UTF8，该缓冲区在本次调用期间有效；
* 一个左操作数为`Type`类型的`inout`表达式，传入的是这个左值的内存地址；
* 一个`[Type]`值，将作为该数组的起始指针传入，并且它的生命周期将在本次调用期间被延长。

如果你定义了一个这样的函数：

```Swift
func takesAPointer(x: UnsafePointer<Float>) {
	// ...
}
```

那么你可以使用以下任意一种方式来调用该函数：

```Swift
var x: Float = 0.0
var p: UnsafePointer<Float> = nil
takesAPointer(nil)
takesAPointer(p)
takesAPointer(&x)
takesAPointer([1.0, 2.0, 3.0])
```

如果函数声明为接受一个`UnsafePointer<Void>`参数，那么该函数可以接受任意`UnsafePointer<Type>`类型的操作数。

如果你定义了一个这样的函数：

```Swift
func takesAVoidPointer(x: UnsafePointer<Void>) {
	// ... 
}
```

那么你可以使用以下任意一种方式来调用这个函数：

```Swift
var x: Float = 0.0, y: Int = 0
var p: UnsafePointer<Float> = nil, q: UnsafePointer<Int> = nil
takesAVoidPointer(nil)
takesAVoidPointer(p)
takesAVoidPointer(q)
takesAVoidPointer(&x)
takesAVoidPointer(&y)
takesAVoidPointer([1.0, 2.0, 3.0] as [Float])
let intArray = [1, 2, 3]
takesAVoidPointer(intArray)
```

### 可变指针（Mutable Pointers）

当一个函数声明为接受`UnsafeMutablePointer<Type>`参数时，该函数可以接受下列任意一种类型作为参数：

* `nil`，作为空指针传入；
* 一个`UnsafeMutablePointer<Type>`类型的值；
* 一个`inout`表达式，其左操作数是`Type`类型的，且被存储起来了,传入的是这个左值的内存地址；
* 一个`inout`的`[Type]`类型的值，将作为该数组的起始指针传入，并且它的生命周期将在本次调用期间被延长。

如果你定义了一个这样的函数：

```Swift
func takesAMutablePointer(x: UnsafeMutablePointer<Float>) {
	// ...
}
```

那么你可以使用以下任意一种方式来调用这个函数：

```Swift
var x: Float = 0.0
var p: UnsafeMutablePointer<Float> = nil
var a: [Float] = [1.0, 2.0, 3.0]
takesAMutablePointer(nil)
takesAMutablePointer(p)
takesAMutablePointer(&x)
takesAMutablePointer(&a)
```

当一个函数声明为接受`UnsafeMutablePointer<Void>`参数，该函数可以接受任意`UnsafeMutablePointer<Type>`类型的操作数。

如果你定义了一个这样的函数：

```Swift
func takesAMutableVoidPointer(x: UnsafeMutablePointer<Void>) {
	// ...
}
```

那么你可以使用以下任意一种方式来调用这个函数：

```Swift
var x: Float = 0.0, y: Int = 0
var p: UnsafeMutablePointer<Float> = nil, q: UnsafeMutablePointer<Int> = nil
var a: [Float] = [1.0, 2.0, 3.0], b: [Int] = [1, 2, 3]
takesAMutableVoidPointer(nil)
takesAMutableVoidPointer(p)
takesAMutableVoidPointer(q)
takesAMutableVoidPointer(&x)
takesAMutableVoidPointer(&y)
takesAMutableVoidPointer(&a)
takesAMutableVoidPointer(&b)
```

### 自动释放指针（Autoreleasing Pointers）

当一个函数声明为接受`AutoreleasingUnsafeMutablePointer<Type>`参数时，该函数可以接受下列任意一种类型作为参数：

* `nil`，作为空指针传入；
* 一个`AutoreleasingUnsafeMutablePointer<Type>`类型的值；
* 一个`inout`表达式，其操作数首先被拷贝到一个无拥有者的缓冲区，传递给被调用函数的就是这个缓冲区的地址。在调用返回时，缓冲区中的值被加载、retain、并重新分配给操作数。

注意，这个列表中没有包含数组。

如果你定义了一个这样的函数：

```Swift
func takesAnAutoreleasingPointer(x: AutoreleasingUnsafeMutablePointer<NSDate?>) {
	// ...
}
```

那么你可以使用以下任意一种方式来调用这个函数：

```Swift
var x: NSDate? = nil
var p: AutoreleasingUnsafeMutablePointer<NSDate?> = nil
takesAnAutoreleasingPointer(nil)
takesAnAutoreleasingPointer(p)
takesAnAutoreleasingPointer(&x)
```

指针指向的类型并不会被桥接。例如，`NSString **`转换到 Swift 后，是`AutoreleasingUnsafeMutablePointer<NSString?>`，而不是`AutoreleasingUnsafeMutablePointer<String?>`。

### 函数指针（Function Pointers）

C 语言的函数指针以符合其调用约定的闭包的形式被导入 Swift 中，用`@convention(c)`属性表示。例如，一个类型为`int (*)(void)`的 C 语言函数指针，会导入为 Swift 的`@convention(c) () -> Int32`。

在调用一个以函数指针为参数的函数时，你可以传递一个顶级的 Swift 函数作为其参数，也可以传递闭包字面量，或者`nil`。只有符合 C 语言函数指针调用约定的 Swift 函数，才能作为参数传递。例如，Core Foundation 的`CFArrayCreateMutable(_:_:_:)`函数，它有个类型为`CFArrayCallBacks`结构体的参数。这个`CFArrayCallBacks`结构体就是用一些函数指针进行初始化的：

```swift
func customCopyDescription(p: UnsafePointer<Void>) -> Unmanaged<CFString>! {
    // return an Unmanaged<CFString>! value
}
 
let callbacks = CFArrayCallBacks(
   	version: 0,
    retain: nil,
    release: nil,
   	copyDescription: customCopyDescription,
   	equal: { (p1, p2) -> DarwinBoolean in
       	// return Bool value
   	}
)
 
var mutableArray = CFArrayCreateMutable(nil, 0, &callbacks)
```

在上面的例子中，在`CFArrayCallBacks`初始化时，传给`retain`和`release`作参数的是`nil`，传给`copyDescription`作参数的是函数`customCopyDescription`，传给`equal`作参数的是一个闭包字面量。

<a name="global_constants"></a>
## 全局常量（Global Constants）

在 C 和 Objective-C 语言源文件中定义的全局常量会自动地被 Swift 编译器导入为 Swift 全局常量。

<a name="preprocessor_directives"></a>
## 预处理指令（Preprocessor Directives）

Swift 编译器不包含预处理器。取而代之的是，它利用编译时属性，编译配置，以及语言特性来完成相同的功能。因此，Swift 没有引进预处理指令。

### 简单宏（Simple Macros）

在 C 和 Objective-C 中，通常使用`#define`指令来定义一个简单的常数，在 Swift，你可以使用全局常量来代替。例如，常量定义`#define FADE_ANIMATION_DURATION 0.35`，在 Swift 可以用`let FADE_ANIMATION_DURATION = 0.35`来更好地表述。由于用于定义常量的简单宏会被直接映射成 Swift 全局常量，Swift 编译器会自动导入在 C 或 Objective-C 源文件中定义的简单宏。

### 复杂宏（Complex Macros）

在 C 和 Objective-C 中使用的复杂宏在 Swift 中没有相对应的东西。复杂宏是那些不用来定义常量的宏，包括加括号的函数式宏。你在 C 和 Objective-C 使用复杂的宏以避免类型检查的限制或避免重复键入大量的样板代码。然而，宏也会让调试和重构更加困难。在 Swift 中你可以使用函数和泛型来达到同样的效果，这没有任何妥协。因此，在 C 和 Objective-C 源文件中定义的复杂宏在 Swift 是不能使用的。

### 编译配置（Build Configurations）

Swift 代码使用和 C 以及 Objective-C 代码不同的方式进行条件编译。Swift 代码可以根据编译配置的组合进行条件编译。编译配置包括`true`和`false`字面值，命令行标志，和下表中的平台测试函数。你可以使用`-D <＃Flag＃>`指定命令行标志。

| 函数 | 有效参数 |
| --- | --- |
| os() | OSX，iOS，watchOS |
| arch() | x86_64，arm，arm64，i386 |

> 注意

> 编译配置`arch(arm)`在`ARM 64`设备上不会返回`true`。编译配置`arch(i386)`在`32–bit`模拟器上会返回`true`。

一个简单的条件编译语句如同下面这段代码：

```swift
#if build configuration
	statements
#else
	statements 
#endif
```

*statements* 由零个或多个有效的 Swift 语句组成，可以包括表达式，普通语句和控制流语句。可以使用`&&`和`||`操作符往一个条件编译语句上添加新的编译条件，使用`!`操作符来否定某条件，使用`#elseif`来添加编译块：

```swift
#if build configuration && !build configuration
	statements
#elseif build configuration
	statements
#else
	statements
#endif
```

与 C 语言预处理器的条件编译不同的是，Swift 条件编译的语句必须是独立完整、语法有效的代码块。这是因为所有的 Swift 代码都会做语法检查，即使有的代码不会被编译。
