# 使用 Cocoa 框架

- [Foundation](#foundation)
    - [被桥接的类型](#bridged_types)
    - [被重命名的类型](#renamed_types)
    - [字符串](#strings)
    - [数值](#numbers)
    - [数组](#arrays) 
    - [集合](#sets)
    - [字典](#dictionaries)
- [Core Foundation](#core_foundation)
    - [重映射类型](#remapped_types)
    - [内存受管理的对象](#memory_managed_objects)
    - [非托管对象](#unmanaged_objects)
- [统一日志](#unified_logging)

为了便于与 Objective-C 进行交互，Swift 提供了高效便捷的方式来使用 Cocoa 框架。

Swift 会自动将一些 Objective-C 类型转换为 Swift 类型，也会将一些 Swift 类型转换为 Objective-C 类型。可以在 Objective-C 和 Swift 间相互转换的数据类型被称为**桥接**类型。例如，在 Swift，可以将一个`String`类型的值传递给一个接收`NSString`对象的 Objective-C 方法。除此之外，很多 Cocoa 框架，包括 Foundation，AppKit，以及 UIKit，都改善了它们的 API，从而在 Swift 中使用起来更加自然。例如，`NSCoder`的方法`decodeObjectOfClass(_:forKey:)`使用 Swift 泛型来提供强类型的方法签名。

<a name = "foundation"></a>
## Foundation

Foundation 框架为应用和框架提供了基层功能，包括数据存储，文本处理，日期时间，排序过滤，持久化和网络等。

<a name = "bridged_types"></a>
### 被桥接的类型

Swift 中的 Foundation 提供了如下桥接值类型用于替代如下 Objective-C 引用类型：

Objective-C 引用类型 | Swift 值类型
:-----------------:|:-------------:
NSAffineTransform | AffineTransform
NSArray | Array
NSCalendar | Calendar
NSCharacterSet | CharacterSet
NSData | Data
NSDateComponents | DateComponents
NSDateInterval | DateInterval
NSDate | Date
NSDecimalNumber | Decimal
NSDictionary | Dictionary
NSIndexPath | IndexPath
NSIndexSet | IndexSet
NSLocale | Locale
NSMeasurement | Measurement
NSNotification | Notification
NSNumber | Bool, Double, Float, Int, UInt
NSPersonNameComponents | PersonNameComponents
NSSet | Set
NSString | String
NSTimeZone | TimeZone
NSURLComponents | URLComponents
NSURLQueryItem | URLQueryItem
NSURL | URL
NSURLComponents | URLComponents
NSURLQueryItem | URLQueryItem
NSURLRequest | URLRequest
NSUUID | UUID 

这些值类型的功能与其对应的引用类型相同。包含不可变和可变子类的类簇被统一桥接为单个值类型。Swift 代码使用`var`和`let`来控制可变性，因此它们不再需要两个类了。原先的引用类型依然可以通过加上`NS`前缀来访问。

可以使用 Objective-C 桥接引用类型的地方就可以使用 Swift 桥接值类型，从而可以在 Swift 代码中以更加自然的方式使用这些原有的功能。基于这个原因，你应该尽量避免在 Swift 代码中使用 Objective-C 桥接引用类型。事实上，Swift 代码导入 Objective-C API 时，导入器会将引用类型替换为相对应的值类型。与之类似，Objective-C 代码导入 Swift API 时，导入器也会将值类型替换为相对应的引用类型。

如果你需要使用原本的 Foundation 对象，你可以使用`as`操作符在这些桥接类型间进行转换。

<a name = "renamed_types"></a>
### 被重命名的类型

Swift 中的 Foundation 重名了一些类，协议，枚举以及常量。

在导入 Foundation 类时，Swift 丢弃了类名的`NS`前缀，但有如下例外情况：

- Objective-C 和 Objective-C 运行时中的一些特定类，例如`NSObject`，`NSAutoreleasePool`，`NSException`，`NSProxy`。
- 平台特定类，例如`NSBackgroundActivity`，`NSUserNotification`，`NSXPCConnection`。
- 拥有对应的桥接值类型的一些类，例如`NSString`，`NSDictionary`，`NSURL`。
- 目前尚无对应的桥接值类型，但是未来会有的一些类，例如`NSAttributedString`，`NSRegularExpression`，`NSPredicate`。

Foundation 中的类经常会声明一些枚举和常量类型。导入这些类型时，Swift 会将它们移到相关类型的内部作为嵌套类型。例如，`NSJSONReadingOptions`的选项集会被导入为`JSONSerialization.ReadingOptions`。

<a name = "strings"></a>
### 字符串

字符串可以在`String`类型和`NSString`类型之间桥接。你可以使用`as`运算符将`String`值转换为`NSString`对象。你也可以通过提供类型标注的方式利用字符串字面量创建`NSString`对象。

```swift
import Foundation
let string: String = "abc"
let bridgedString: NSString = string as NSString

let stringLiteral: NSString = "123"
if let integerValue = Int(stringLiteral as String) {
    print("\(stringLiteral) is the integer \(integerValue)")
}
// 打印 "123 is the integer 123"
```

> 注意  
> Swift 的`String`类型由独立编码的 Unicode 字符组成，并提供了在各种 Unicode 表现形式下访问这些字符的支持。`NSString`类以 UTF-16 码元序列的形式编码兼容 Unicode 的文本字符串。表示字符串长度，字符索引，区间的这些依据 16 位平台字节序值的`NSString`方法，均有相对应的 Swift 方法，这些方法使用`String.Index`和`Range<String.Index>`值而不是`Int`和`NSRange`值。

<a name = "numbers"></a>
### 数值

Swift 会在`NSNumber`类和 Swift 算术类型之间桥接，包括`Int`，`Double`，`Bool`。

你可以使用`as`运算符将 Swift 数值转换为`NSNumber`对象。因为`NSNumber`可以包含多种不同类型，将其转换为 Swift 数值时必须使用`as?`运算符。你也可以通过提供类型标注的方式利用浮点数，整数，布尔字面量创建`NSNumber`对象。

```swift
import Foundation
let number = 42
let bridgedNumber: NSNumber = number as NSNumber

let integerLiteral: NSNumber = 5
let floatLiteral: NSNumber = 3.14159
let booleanLiteral: NSNumber = true
```

> 注意  
> Objective-C 中具有平台适应性的整数类型，例如`NSInteger`和`NSUInteger`，会被桥接为`Int`。

<a name="arrays"></a>
### 数组

Swift 会在`Array`类型和`NSArray`类之间桥接。使用轻量泛型的`NSArray`对象被桥接时，其元素类型也会被桥接过去。如果`NSArray`对象没有使用轻量泛型，那么它会被桥接为`[Any]`类型的 Swift 数组。

例如，思考以下 Objective-C 声明：

```objective-c
@property NSArray *objects;
@property NSArray<NSDate *> *dates;
- (NSArray<NSDate *> *)datesBeforeDate:(NSDate *)date;
- (void)addDatesParsedFromTimestamps:(NSArray<NSString *> *)timestamps;
```

Swift 以如下所示导入它们：

```swift
var objects: [Any]
var dates: [Date]
func datesBeforeDate(date: Date) -> [Date]
func addDatesParsedFromTimestamps(timestamps: [String])
```

还可以直接利用 Swift 数组字面量创建`NSArray`对象，这同样遵循上面提到的桥接规则。将常量或变量声明为`NSArray`类型并为其赋值数组字面量时，Swift 将会创建`NSArray`对象，而不是 Swift 数组。

```swift
let schoolSupplies: NSArray = ["Pencil", "Eraser", "Notebkko"]
// schoolSupplies 是一个包含三个元素的 NSArray 对象
```

<a name="sets"></a>
### 集合

Swift 也会在`Set`类型和`NSSet`类之间桥接。使用轻量泛型的`NSSet`对象会被桥接为`Set<ObjectType>`类型的 Swift 集合。如果`NSSet`对象没有使用轻量泛型，那么它会被桥接为`Set<AnyHashable>`类型的 Swift 集合。

例如，思考以下 Objective-C 声明：

```objective-c
@property NSSet *objects;
@property NSSet<NSString *> *words;
- (NSSet<NSString *> *)wordsMatchingPredicate:(NSPredicate *)predicate;
- (void)removeWords:(NSSet<NSString *> *)words;
```

Swift 以如下所示导入它们：

```swift
var objects: Set<AnyHashable>
var words: Set<String>
func wordsMatchingPredicate(predicate: NSPredicate) -> Set<String>
func removeWords(words: Set<String>)
```

还可以直接利用 Swift 数组字面量创建`NSSet`对象，这同样遵循上面提到的桥接规则。将常量或者变量声明为`NSSet`类型，并为其赋值数组字面量时，Swift 将会创建`NSSet`对象，而不是 Swift 集合。

```swift
let amenities: NSSet = ["Sauna", "Steam Room", "Jacuzzi"]
// amenities 是一个包含三个元素的 NSSet 对象
```

<a name="dictionaries"></a>
### 字典

Swift 同样会在`Dictionary`类型和`NSDictionary`类之间桥接。使用轻量泛型的`NSDictionary`对象会被桥接为`[Key: Value]`类型的 Swift 字典。如果`NSDictionary`对象没有使用轻量泛型，那么它会被桥接为`[AnyHashable: Any]`类型的 Swift 字典。

例如，思考以下 Objective-C 声明：

```objective-c
@property NSDictionary *keyedObjects;
@property NSDictionary<NSURL *, NSData *> *cachedData;
- (NSDictionary<NSURL *, NSNumber *> *)fileSizesForURLsWithSuffix:(NSString *)suffix;
- (void)setCacheExpirations:(NSDictionary<NSURL *, NSDate *> *)expirations;
```

Swift 以如下所示导入它们：

```swift
var keyedObjects: [AnyHashable: Any]
var cachedData: [URL: Data]
func fileSizesForURLsWithSuffix(suffix: String) -> [URL: NSNumber]
func setCacheExpirations(expirations: [URL: NSDate])
```

还可以直接利用 Swift 字典字面量创建`NSDictionary`对象，这同样遵循上面提到的桥接规则。将常量或者变量声明为`NSDictionary`类型，并为其赋值字典字面量时，Swift 将会创建`NSDictionary`对象，而不是 Swift 字典。

```swift
let medalRankings: NSDictionary = ["Gold": "1st Place", "Silver": "2nd Place", "Bronze": "3rd Place"]
// medalRankings 是一个包含三个键值对的 NSDictionary 对象
```

<a name = "core_foundation"></a>
## Core Foundation

Core Foundation 类型会被导入为 Swift 类。无论是否提供了内存管理标注，Swift 都会自动管理 Core Foundation 对象的内存，包括你自己实例化的 Core Foundation 对象。在 Swift，可以将每一对可以桥接的 Fundation 和 Core Foundation 类型互换使用。如果先将 Core Foundation 类型桥接为 Foundation 类型，就可以进一步桥接为 Swift 标准库类型。

<a name="remapped_types"></a>
### 重映射类型

Swift 导入 Core Foundation 类型时，会将这些类型的名字重映射，从类型名字的末端移除 *Ref*，所有的 Swift 类都是引用类型，因此该后缀是多余的。

Core Foundation 的`CFTypeRef`类型会重映射为`Anyobject`类型。以前使用`CFTypeRef`的地方，现在该换成`AnyObject`了。

<a name="memory_managed_objects"></a>
### 内存受管理的对象

对于从带内存管理标注的 API 返回的 Core Foundation 对象，Swift 会自动对其进行内存管理，你不需要再调用`CFRetain`，`CFRelease`，或者`CFAutorelease`函数。

如果自定义的 C 函数或 Objective-C 方法返回 Core Foundation 对象，需要用`CF_RETURNS_RETAINED`或者`CF_RETURNS_NOT_RETAINED`宏标注这些函数或方法，从而帮助编译器自动插入内存管理函数调用。还可以使用`CF_IMPLICIT_BRIDGING_ENABLED`和`CF_IMPLICIT_BRIDGING_DISABLED`宏围住那些遵循 Core Foundation 内存管理命名规定的 C 函数声明，从而能够根据命名推导出内存管理策略。

如果只调用那些带有内存管理标注并且不会间接返回 Core Foundation 对象的 API，那么可以略过本节的剩余部分了。否则，需要进一步学习有关 Core Foundation 非托管对象的知识。

<a name="unmanaged_objects"></a>
### 非托管对象

对于那些没有内存管理标注的 API，编译器无法自动对返回的 Core Foundation 对象进行内存管理。Swift 将这些返回的 Core Foundation 对象包装在一个`Unmanaged<Instance>`结构中。所有被间接返回的 Core Foundation 对象也都是非托管对象。例如，这有个不带内存管理标注的 C 函数：

```objective-c
CFStringRef StringByAddingTwoStrings(CFStringRef string1, CFStringRef string2)
```

Swift 以如下所示导入它们：

```swift
func StringByAddingTwoStrings(_: CFString!, _: CFString!) -> Unmanaged<CFString>! {
    // ...
}
```

从没有内存管理标注的 API 接收到非托管对象后，在使用它之前，必须将它转换为能够接受内存管理的对象。通过这种方式，Swift 就可以为你对其进行内存管理。`Unmanaged<Instance>`结构提供了两个方法，用于将一个非托管对象转换为一个可接受内存管理的对象，即`takeUnretainedValue()`方法和`takeRetainedValue()`方法。这两个方法均会返回解包后的原始对象，可以根据 API 返回的是非保留对象还是被保留对象来选择对应的方法。

例如，假设上面的 C 函数不会保留`CFString`对象。在使用这个对象前，应该使用`takeUnretainedValue()`函数。

```swift
let memoryManagedResult = StringByAddingTwoStrings(str1, str2).takeUnretainedValue()
// memoryManagedResult 是一个接受内存管理的 CFString
```

当然，也可以对一个非托管对象使用`retain()`，`release()`和`autorelease()`方法，但是这种做法并不推荐。

想要了解更多信息，请参阅 [*Memory Management Programming Guide for Core Foundation*](https://developer.apple.com/library/prerelease/ios/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/CFMemoryMgmt.html#//apple_ref/doc/uid/10000127i) 中的 [*Core Foundation Object Lifecycle Management*](https://developer.apple.com/library/prerelease/ios/documentation/CoreFoundation/Conceptual/CFMemoryMgmt/Articles/lifecycle.html#//apple_ref/doc/uid/TP40002439-SW1) 小节。

<a name="unified_logging"></a>
## 统一日志

统一日志系统提供了一个 API 来捕获系统各个层级传递的消息，它是 Foundation 框架中`NSLog`函数的替代者。统一日志只在 `iOS 10.0`，`macOS 10.12`，`tvOS 10.0`，`watchOS 3.0`，以及更高的系统平台上可用。

在 Swift，你可以通过顶级函数`os_log(_:dso:log:type:_:)`来使用统一日志系统，该函数声明在`os`模块的子模块`log`中。

```swift
import os.log

os_log("This is a log message.")
```

你可以通过使用`NSString`或`printf`格式化字符串连同单个或多个尾参数来格式化日志信息。

```swift
let fileSize = 1234567890
os_log("Finished downloading file. Size: %{iec-bytes}d", fileSize)
```

你还可以指定一个日志系统中定义的日志级别，例如`Info`，`Debug`，`Error`，从而根据日志事件的重要性来控制日志信息如何被处理。例如，某条信息可能很有帮助，但是该信息并不是排查错误所必需的，那么这条信息就应该记录在信息级别。

```swift
os_log("This is additional info that may be helpful for troubleshooting.", type: .info)
```

为了将某条信息记录到指定子系统，你可以创建一个新的`OSLog`对象，指定子系统和类别，并将其作为参数传入`os_log`函数。

```swift
et customLog = OSLog("com.your_company.your_subsystem_name.plist", "your_category_name")
os_log("This is info that may be helpful during development or debugging.", log: customLog, type: .debug)
```

想要了解更多关于统一日志的信息，请参阅 [Logging](https://developer.apple.com/reference/os/1891852-logging)。