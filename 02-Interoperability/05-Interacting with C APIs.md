# 与 C 语言 API 交互

本页包含内容：

- [基本类型](#primitive_types)
- [枚举](#enumerations)
- [选项集合](#Option Sets)
- [联合体](#Unions)
- [位字段](#Bit Fields)
- [匿名结构体和联合体字段](#Unnamed_Structure_and_Union_Fields)
- [指针](#pointer)
	- [常量指针](#Constant_Pointers)
	- [可变指针](#Mutable_Pointers)
	- [自动释放指针](#Autoreleasing_Pointers)
	- [函数指针](#Function_Pointers)
- [计算数据类型大小](#Data Type Size Calculation)
- [可变参数函数](#Variadic_Functions)
- [全局常量](#global_constants)
- [一次性初始化](#One_Time_Initialization)
- [预处理指令](#preprocessor_directives)
	- [简单宏](#Simple_Macros)
	- [复杂宏](#Complex_Macros)
	- [编译配置](#Build_Configurations)

为了更好地支持与 Objective-C 语言的互用性，Swift 对一些 C 语言的类型和特性保持了兼容性，提供了一些方式来配合常见的 C 语言设计和模式。

<a name="primitive_types"></a>
## 基本类型

Swift 提供了一些与 C 语言基本类型（例如`char`，`int`，`float`和`double`等）对应的类型。然而，这些类型和 Swift 基本类型（例如`Int`）之间不能进行隐式转换。因此，只有在必要时才应使用这些类型，否则尽可能地使用`Int`这种原生 Swift 类型。

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
## 枚举

用`NS_ENUM`宏声明的 C 语言枚举，会被导入为原始值类型为`Int`的 Swift 枚举。无论枚举是在系统框架还是在自己的代码中定义的，导入到 Swift 后，它们的前缀名将被移除。

例如，下面是一个 Objective-C 枚举的声明：

```Objective-C
typedef NS_ENUM(NSInteger, UITableViewCellStyle) {
	UITableViewCellStyleDefault,
    UITableViewCellStyleValue1,
    UITableViewCellStyleValue2,
    UITableViewCellStyleSubtitle
};
```

在 Swift，它被导入后会像这样：

```Swift
enum UITableViewCellStyle: Int {
	case Default
    case Value1
	case Value2
    case Subtitle
}
```

需要使用一个枚举值时，使用以点（`.`）开头的枚举变量名：

```Swift
let cellStyle: UITableViewCellStyle = .Default
```

Swift 会将未使用`ENUM`或`NS_OPTIONS`宏声明的 C 语言枚举导入为结构体。每个枚举成员会被导入为该结构体类型的全局只读计算型属性，而并非结构体的属性。

例如，下面是个未使用`ENUM`宏声明的 C 语言枚举：

```objective-c
typedef enum Error {
    ErrorNone = 0,
    ErrorFileNotFound = -1,
    ErrorInvalidFormat = -2,
};
```

在 Swift，它被导入后会像这样：

```swift
struct Error: RawRepresentable, Equatable { }

var ErrorNone: Error { get }
var ErrorFileNotFound: Error { get }
var ErrorInvalidFormat: Error { get }
```

被导入到 Swift 的 C 语言枚举会自动符合`Equatable`协议。

<a name="Option Sets"></a>
## 选项集合

使用`NS_OPTIONS`宏声明的 C 语言枚举，会被导入为 Swfit 选项集合。选项集合会像枚举一样把前缀移除，只剩下选项值名称。

例如，下面是一个 Objective-C 选项集合的声明：

```Objective-C
typedef NS_OPTIONS(NSUInteger, NSJSONReadingOptions) {
	NSJSONReadingMutableContainers = (1UL << 0),
	NSJSONReadingMutableLeaves     = (1UL << 1),
	NSJSONReadingAllowFragments    = (1UL << 2)
};
```

在 Swift，它被导入后会像这样：

```Swift
struct NSJSONReadingOptions: OptionSetType {
	init(rawValue: UInt)
    
    static var MutableContainers: NSJSONReadingOptions { get }
    static var MutableLeaves: NSJSONReadingOptions { get }
    static var AllowFragments: NSJSONReadingOptions { get }
}
```

在 Objective-C，一个选项集合是一些整数值的位掩码。可以使用按位或操作符（`|`）组合选项值，使用按位与操作符（`&`）检测选项值。可以使用常量值或者表达式创建一个选项集合。一个空的选项集合使用常数`0`表示。

在 Swift，选项集合用一个符合`OptionSetType`协议的结构体表示，每个选项值都是结构体的一个静态变量。选项集合类似于 Swift 的集合类型`Set`，可以用`insert(_:)`或者`unionInPlace(_:)`方法添加选项值，用`remove(_:)`或者`subtractInPlace(_:)`方法删除选项值，用`contains(_:)`方法检查选项值。可以使用一个数组字面量创建一个选项集合，访问选项值像枚举一样也用点（`.`）开头。可以使用一个空数组字面量创建一个空选项集合，也可以调用默认构造器。

```Swift
let options: NSDataBase64EncodingOptions = [
	.Encoding76CharacterLineLength,
    .EncodingEndLineWithLineFeed
]
let string = data.base64EncodedStringWithOptions(options)
```

<a name="Unions"></a>
## 联合体

Swift 仅部分支持 C 语言的`union`类型。在导入混有 C 语言的联合体或者位字段的类型时，Swift 无法访问不支持的字段。但是，参数或者返回值包含这些类型的 C 和 Objective-C 的 API 是可以在 Swift 中使用的。

<a name="Bit Fields"></a>
## 位字段

Swift 会将结构体中的位字段导入为结构体的计算型属性，例如 Foundation 中的`NSDecimal`类型：

```objective-c
// Objective-C 声明
typedef struct {
    signed   int _exponent:8;
    unsigned int _length:4;     
    unsigned int _isNegative:1;
    unsigned int _isCompact:1;
    unsigned int _reserved:18;
    unsigned short _mantissa[NSDecimalMaxSize];
} NSDecimal;
```

```swift
// Swift 声明
public struct NSDecimal {
    public var _exponent: Int32
    public var _length: UInt32 
    public var _isNegative: UInt32
    public var _isCompact: UInt32
    public var _reserved: UInt32
    public var _mantissa: (UInt16, UInt16, UInt16, UInt16, UInt16, UInt16, UInt16, UInt16)
    public init()
    public init(
    _exponent: Int32,
    _length: UInt32, 
    _isNegative: UInt32, 
    _isCompact: UInt32, 
    _reserved: UInt32,
    _mantissa: (UInt16, UInt16, UInt16, UInt16, UInt16, UInt16, UInt16, UInt16))
}
```

<a name="Unnamed_Structure_and_Union_Fields"></a>
## 匿名结构体和联合体字段

C 结构体和联合体类型可能会被定义为匿名的结构体和联合体字段。Swift 不支持匿名结构体，因此这些字段会被导入为嵌套类型，并以 `__Unnamed_fieldName` 的形式命名。

例如，如下这个名为 `Pie` 的 C 结构体包含一个名为 `crust` 的匿名结构体类型字段，还包含一个名为 `filling` 的匿名联合体类型字段：

```swift
struct Pie {
	struct { bool flakey; } crust;
	union { int fruit; int meat; } filling;
}
```

导入到 Swift 中时，`crust` 属性的类型为 `Pie.__Unnamed_crust`，`filling` 属性的类型为 `Pie.__Unnamed_filling`。

<a name="pointer"></a>
## 指针

Swift 尽可能地避免直接使用指针。然而，需要直接操作内存的时候，Swift 也提供了多种指针类型。下面的表使用 `Type` 作为类型名称的占位符来表示相应的映射语法。

对于返回类型，变量和参数，遵循如下映射：

| C 语法 | Swift 语法 |
| ------ | ------ |
| const Type * | UnsafePointer\<Type\> |
| Type * | UnsafeMutablePointer\<Type\> |

对于类类型，遵循如下映射：

| C 语法 | Swift 语法 |
| ------ | ------ |
| Type * const * | UnsafePointer\<Type\> |
| Type * __strong * | UnsafeMutablePointer\<Type\> |
| Type ** | AutoreleasingUnsafeMutablePointer\<Type\> |

<a name="Constant_Pointers"></a>
### 常量指针

如果函数接受 `UnsafePointer<Type>` 参数，那么该函数可以接受下列任意一种类型作为参数：

* `nil`，作为空指针传入。
* 一个 `UnsafePointer<Type>`，`UnsafeMutablePointer<Type>`，或者 `AutoreleasingUnsafeMutablePointer<Type>` 类型的值，后面两种类型在必要时会转换成 `UnsafePointer<Type>`。
* 一个 `String` 类型的值，如果 `Type` 指代 `Int8` 或者 `UInt8`。`String` 类型的值会被自动转换为 UTF8 形式到一个缓冲区内，该缓冲区的指针会被传递给函数。
* 一个左操作数为 `Type` 类型的 `inout` 表达式，左操作数的内存地址作为函数参数传入。
* 一个 `[Type]` 类型的值，将作为该数组的起始指针传入。

传递给函数的指针仅保证在函数调用期间内有效，不要试图保留指针并在函数返回之后继续使用。

如果定义了一个类似下面这样的函数：

```Swift
func takesAPointer(x: UnsafePointer<Float>) {
	// ...
}
```

那么可以使用以下任意一种方式来调用该函数：

```Swift
var x: Float = 0.0
var p: UnsafePointer<Float> = nil
takesAPointer(nil)
takesAPointer(p)
takesAPointer(&x)
takesAPointer([1.0, 2.0, 3.0])
```

如果函数接受 `UnsafePointer<Void>` 参数，那么该函数可以接受任意 `UnsafePointer<Type>` 类型的参数。

如果定义了一个类似下面这样的函数：

```Swift
func takesAVoidPointer(x: UnsafePointer<Void>) {
	// ... 
}
```

那么可以使用以下任意一种方式来调用该函数：

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

<a name="Mutable_Pointers"></a>
### 可变指针

如果函数接受 `UnsafeMutablePointer<Type>` 参数，那么该函数可以接受下列任意一种类型作为参数：

* `nil`，作为空指针传入。
* 一个 `UnsafeMutablePointer<Type>` 类型的值。
* 一个左操作数为 `Type` 类型的 `inout` 表达式，左操作数的内存地址作为函数参数传入。
* 一个 `inout [Type]` 类型的值，将作为该数组的起始指针传入，其生命周期会延续到本次调用结束。

如果定义了一个类似下面这样的函数：

```Swift
func takesAMutablePointer(x: UnsafeMutablePointer<Float>) {
	// ...
}
```

那么可以使用以下任意一种方式来调用该函数：

```Swift
var x: Float = 0.0
var p: UnsafeMutablePointer<Float> = nil
var a: [Float] = [1.0, 2.0, 3.0]
takesAMutablePointer(nil)
takesAMutablePointer(p)
takesAMutablePointer(&x)
takesAMutablePointer(&a)
```

如果函数接受 `UnsafeMutablePointer<Void>` 参数，那么该函数可以接受任意 `UnsafeMutablePointer<Type>` 类型的参数。

如果定义了一个类似下面这样的函数：

```Swift
func takesAMutableVoidPointer(x: UnsafeMutablePointer<Void>) {
	// ...
}
```

那么可以使用以下任意一种方式来调用该函数：

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

<a name="Autoreleasing_Pointers"></a>
### 自动释放指针

如果函数接受 `AutoreleasingUnsafeMutablePointer<Type>` 参数，那么该函数可以接受下列任意一种类型作为参数：

* `nil`，作为空指针传入。
* 一个 `AutoreleasingUnsafeMutablePointer<Type>` 类型的值。
* 一个 `inout` 表达式，其操作数首先被拷贝到一个无主临时缓冲区，缓冲区的地址会作为函数参数传入。函数调用结束时，缓冲区中的值被加载并被 retain，然后重新分配给操作数。

注意，上述列表中没有包含数组。

如果定义了一个类似下面这样的函数：

```Swift
func takesAnAutoreleasingPointer(x: AutoreleasingUnsafeMutablePointer<NSDate?>) {
	// ...
}
```

那么可以使用以下任意一种方式来调用该函数：

```Swift
var x: NSDate? = nil
var p: AutoreleasingUnsafeMutablePointer<NSDate?> = nil
takesAnAutoreleasingPointer(nil)
takesAnAutoreleasingPointer(p)
takesAnAutoreleasingPointer(&x)
```

多级指针的类型不会被桥接。例如，`NSString **` 转换到 Swift 后，是 `AutoreleasingUnsafeMutablePointer<NSString?>`，而不是 `AutoreleasingUnsafeMutablePointer<String?>`。

<a name="Function_Pointers"></a>
### 函数指针

Swift 将 C 语言的函数指针导入为符合其调用约定的闭包，使用 `@convention(c)` 特性来表示。例如，一个类型为 `int (*)(void)` 的 C 语言函数指针，会以 `@convention(c) () -> Int32` 的形式导入到 Swift。

调用一个以函数指针为参数的函数时，可以传递一个顶级的 Swift 函数作为其参数，也可以传递闭包字面量，或者 `nil`。You can also pass a closure property of a generic type or a generic method as long as no generic type parameters are referenced in the closure’s argument list or body.例如，Core Foundation 的 `CFArrayCreateMutable(_:_:_:)` 函数接受一个 `CFArrayCallBacks` 结构体作为参数，这个 `CFArrayCallBacks` 结构体使用一些函数指针进行初始化：

```swift
func customCopyDescription(p: UnsafePointer<Void>) -> Unmanaged<CFString>! {
    // 返回一个 Unmanaged<CFString>! 类型的值
}
 
let callbacks = CFArrayCallBacks(
   	version: 0,
    retain: nil,
    release: nil,
   	copyDescription: customCopyDescription,
   	equal: { (p1, p2) -> DarwinBoolean in
       	// 返回 Bool 类型的值
   	}
)
 
var mutableArray = CFArrayCreateMutable(nil, 0, &callbacks)
```

上面的例子中，结构体 `CFArrayCallBacks` 的构造器使用 `nil` 作为参数 `retain` 和 `release` 的值，使用函数 `customCopyDescription` 作为参数 `copyDescription` 的值，以及使用一个闭包字面量作为参数 `equal` 的值。

<a name="Data Type Size Calculation"></a>
## 计算数据类型大小

在 C 语言，`sizeof`操作符会返回任意变量或数据类型的大小。在 Swift，可以使用`sizeof`函数获取某个类型的大小，或者使用`sizeofValue`函数获取某个类型的值的大小。然而，和 C 语言的`sizeof`操作符不同，Swift 的`sizeof`函数和`sizeofValue`函数不包含任何内存对齐的填充部分。例如，Darwin 中的`timeval`结构体在 C 语言中占 16 字节，而在 Swift 中占 12 字节：

```swift
print(sizeof(timeval))
// 打印 "12"
```

如果希望计算结果和 C 语言的`sizeof`操作符一致，可以使用`strideof`函数或`strideofValue`函数来计算经过内存对齐的类型的大小：

```swift
print(strideof(timeval))
// 打印 "16"
```

例如，`setsockopt`函数为 socket 指定一个`timeval`结构体作为接收数据时的超时选项（`SO_RCVTIMEO`）时，需要传递指向`timeval`结构体的指针，以及该结构体的大小，此时可以使用`strideof`来计算结构体大小：

```swift
let sockfd = socket(AF_INET, SOCK_STREAM, 0)
var optval = timeval(tv_sec: 30, tv_usec: 0)
let optlen = socklen_t(strideof(timeval))
if setsockopt(sockfd, SOL_SOCKET, SO_RCVTIMEO, &optval, optlen) == 0 {
	// ...
}
```

> 注意  
> 只有符合 C 语言函数指针调用约定的 Swift 函数才能作为对应的函数指针参数。另外，如同 C 语言的函数指针，标记`@convention(c)`特性的 Swift 函数不具备捕获能力。   
> 更多信息请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的 [类型特性](http://wiki.jikexueyuan.com/project/swift/chapter3/06_Attributes.html#type_attributes) 小节。

<a name="Variadic_Functions"></a>
## 可变参数函数

在 Swift，可以调用 C 中的可变参数函数，例如 `vasprintf` 函数，调用这种函数需使用 `getVaList(_:)` 或 `withVaList(_:_:)` 函数。`getVaList(_:)` 函数接受一个 `CVarArgType` 类型的数组，返回一个 `CVaListPointer` 类型的值。相反，`withVaList(_:_:)` 函数在闭包体中通过闭包参数来提供该值，而不是直接返回它。最终，`CVaListPointer` 类型的值会传递给接受可变参数的 C 函数的 `va_list` 参数。

例如，如下示例代码演示了如何在 Swift 中调用 `vasprintf` 函数：

```swift
func sprintf(format: String, _ args: CVarArgType...) -> String? {
    return withVaList(args) { va_list in
        var buffer: UnsafeMutablePointer<Int8> = nil
        return format.withCString { CString in
            guard vasprintf(&buffer, CString, va_list) != 0 else {
                return nil
            }
            return String.fromCString(buffer)
        }
    }
}
print(sprintf("√2 ≅ %g", sqrt(2.0))!)
// 打印输出 "√2 ≅ 1.41421"
```

<a name="global_constants"></a>
## 全局常量

C 和 Objective-C 的源文件中定义的全局常量会自动被 Swift 编译器导入为 Swift 全局常量。

<a name="One_Time_Initialization"></a>
## 一次性初始化

在 C 语言中，POSIX 的 `pthread_once()` 函数，以及 GCD 的 `dispatch_once()` 和 `dispatch_once_f()` 函数，提供了一种机制来保证初始化代码只会执行一次。在 Swift，全局常量和存储型类型属性可以确保相应的初始化过程只执行一次，即使从多个线程同时进行访问。Swift 从语言特性层面提供了这一功能，因此并未暴露相应的 POSIX 以及 GCD 函数的调用过程。

<a name="preprocessor_directives"></a>
## 预处理指令

Swift 编译器不包含预处理器，它能利用编译时特性，编译配置，以及语言特性来实现相同的功能。因此，预处理指令不会导入到 Swift。

<a name="Simple_Macros"></a>
### 简单宏

在 C 和 Objective-C，通常使用`#define`指令来定义一个简单的常数，在 Swift，可以使用全局常量来代替。例如，对于常量定义`#define FADE_ANIMATION_DURATION 0.35`，在 Swift 可以表示为`let FADE_ANIMATION_DURATION = 0.35`。由于用于定义常量的简单宏会被直接映射成 Swift 全局常量，因此 Swift 编译器会自动导入 C 或 Objective-C 源文件中定义的简单宏。

<a name="Complex_Macros"></a>
### 复杂宏

C 和 Objective-C 的复杂宏在 Swift 中没有相对应的东西。复杂宏是那些不是用来定义常量的宏，包括加括号的函数式宏。在 C 和 Objective-C 使用复杂的宏来避免类型检查的限制或避免重复键入大量样板代码。然而，宏也会让调试和重构变得更加困难。在 Swift，可以使用函数和泛型达到同样效果。因此，C 和 Objective-C 源文件中定义的复杂宏无法在 Swift 使用。

<a name="Build_Configurations"></a>
### 编译配置

Swift 的条件编译和 C 以及 Objective-C 不同，Swift 的条件编译基于对编译配置的评估。编译配置包括 `true` 和 `false` 字面值，命令行标志，以及下表中的平台和语言版本测试函数。还可以使用 `-D <＃flag＃>` 自定义命令行标志。

| 函数 | 有效参数 |
| --- | --- |
| os() | OSX，iOS，watchOS，tvOS |
| arch() | x86_64，arm，arm64，i386 |
| swift() | >= 指定的版本号 |

> 注意  
> 编译配置 `arch(arm)` 在 ARM 64 位的设备上不会返回 `true`。编译配置 `arch(i386)` 在 32 位的 iOS 模拟器上会返回 `true`。

一个简单的条件编译语句形式如下：

```swift
#if build configuration
	statements
#else
	statements 
#endif
```

*statements* 由零个或多个有效的 Swift 语句组成，可以包括表达式，普通语句和控制流语句。可以使用 `&&` 和 `||` 运算符添加新的编译条件，使用 `!` 运算符对编译条件取反，以及使用 `#elseif` 添加新的条件编译块：

```swift
#if build configuration && !build configuration
	statements
#elseif build configuration
	statements
#else
	statements
#endif
```

与 C 语言预处理器的条件编译不同的是，Swift 的条件编译块中的语句必须是独立且语法有效的代码，因为所有的 Swift 代码都会进行语法检查，即使某些代码不会被编译。
