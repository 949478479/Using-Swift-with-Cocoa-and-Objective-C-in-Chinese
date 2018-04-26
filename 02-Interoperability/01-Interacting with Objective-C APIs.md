# 与 Objective-C API 交互

- [初始化](#initialization)
    - [类工厂方法和便利构造器](#class_factory_methods_and_convenience_initializers)
    - [可失败初始化](#failable_initialization)
- [访问属性](#accessing_properties)
- [方法](#working_with_methods)
- [id 兼容性](#id_compatibility)
    - [将 Any 向下转换](#downcasting_any)
    - [动态方法查找](#dynamic_method_lookup)
    - [无法识别的选择器和可选链语法](#unrecognized_selectors_and_optional_chaining)
- [为空性和可选类型](#nullability_and_optionals)
	- [将可选值桥接为 NSNull 实例](#bridging_optionals_to_nonnullable_objects)
- [协议限定类](#protocol_qualified_classes)
- [轻量泛型](#lightweight_generics)
- [扩展](#extensions)
- [闭包](#closures)
    - [在捕获 self 时避免强引用循环](#avoiding_strong_reference_cycles_when_capturing_self)
- [对象比较](#object_comparison)
    - [哈希](#hashing)
- [Swift 类型兼容性](#swift_type_compatibility)
    - [配置 Swift 在 Objective-C 中的接口](#configuring_swift_interfaces_in_objective_c)
    - [强制动态派发](#requiring_dynamic_dispatch)
- [选择器](#selectors)
    - [Objective-C 方法的不安全调用](#unsafe_invocation_of_objective_c_methods)
- [键和键路径](#keys_and_key_paths)

**互用性**是能让 Swift 和 Objective-C 相接合的特性，这使你能够在一种语言编写的文件中使用另一种语言。当你准备开始把 Swift 融入到你的开发流程中时，学会如何利用互用性来重新定义、改善并提高你编写 Cocoa 应用的方法，不失为一个好主意。

互用性很重要的一点就是允许你在编写 Swift 代码时使用 Objective-C API。导入一个 Objective-C 框架后，可以使用原生的 Swift 语法实例化这些类并与之交互。

<a name="initialization"></a>
## 初始化

要在 Swift 中实例化一个 Objective-C 类，你应该使用 Swift 语法调用它的一个构造器。

Objective-C 构造器以`init`开头，如果它接收参数，则会以`initWith`开头。Objective-C 构造器导入到 Swift 后，`init`前缀变为`init`关键字，以此表明该方法是 Swift 构造方法。如果构造器接收参数，`With`单词会被移除，方法名的其余部分则会被分配为相应的命名参数。

例如，思考如下 Objective-C 构造器声明：

```objective-c
- (instancetype)init;
- (instancetype)initWithFrame:(CGRect)frame
                        style:(UITableViewStyle)style;
```

Swift 中等价的构造器声明如下所示：

```swift
init() { /* ... */ }
init(frame: CGRect, style: UITableViewStyle) { /* ... */ }
```

Objective-C 和 Swift 构造器语法的区别在实例化对象时将更为明显：

在 Objective-C 这样写：

```objective-c
UITableView *myTableView = [[UITableView alloc] initWithFrame:.zero 
                                                        style:UITableViewStyleGrouped];
```

在 Swift 则应该这样写：

```swift
let myTableView: UITableView = UITableView(frame: .zero, style: .grouped)
```

注意，你不需要调用`alloc`，Swift 会为你处理。另外，调用任何 Swift 构造方法时都不会出现`init`单词。

你可以在赋值时显式地声明类型，也可以省略类型，Swift 能从构造器自动推断出类型。

```swift
let myTextField = UITextField(frame: CGRect(x: 0.0, y: 0.0, width: 200.0, height: 40.0))
```

此处的`UITableView`和`UITextField`对象和你在 Objective-C 中实例化的一样。你可以用同样的方式使用它们，例如访问属性或者调用各自类中的方法。

<a name="class_factory_methods_and_convenience_initializers"></a>
### 类工厂方法和便利构造器

为了统一和简洁，Objective-C 中的类工厂方法被导入为 Swift 中的便利构造器，从而能使用同样简洁的构造器语法。

例如，在 Objective-C 像这样调用一个工厂方法：

```objective-c
UIColor *color = [UIColor colorWithRed:0.5 green:0.0 blue:0.5 alpha:1.0];
```

在 Swift 应该这样写：

```swift
let color = UIColor(red: 0.5, green: 0.0, blue: 0.5, alpha: 1.0)
```

<a name="failable_initialization"></a>
### 可失败初始化

在 Objective-C，构造器会直接返回它们初始化的对象。初始化失败时，为了告知调用者，Objective-C 构造器会返回`nil`。在 Swift，这种模式被内置到语言特性中，被称为`可失败初始化`。

许多系统框架中的 Objective-C 构造器在导入到 Swift 时会被检查初始化是否会失败。你可以在你的 Objective-C 类中，使用`为空性标注`来指明初始化是否会失败，请参阅 [为空性和可选类型](#nullability_and_optionals) 小节。如果初始化不会失败，这些 Objective-C 构造器会作为`init(...)`导入，而如果初始化会失败，则会作为`init?(...)`导入。在没有任何为空性标注的情况下，Objective-C 构造器会作为`init!(...)`导入。

例如，当指定路径的图片文件不存在时，使用`UIImage(contentsOfFile:)`构造器初始化`UIImage`对象便会失败。你可以用可选绑定语法对这种构造器的可选类型的返回值进行解包，从而判断初始化是否成功。

```swift
if let image = UIImage(contentsOfFile: "MyImage.png") {
    // 加载图片成功
} else {
    // 无法加载图片
}
```

<a name="accessing_properties"></a>
## 访问属性

在 Objective-C 中使用`@property`语法声明的属性会以如下方式导入到 Swift：

- 具有为空性属性特性的属性（`nonnull`，`nullable`，`null_resettable`），作为可选类型或者非可选类型的属性导入。请参阅 [为空性和可选类型](#nullability_and_optionals) 小节。
- 具有`readonly`属性特性的属性，作为只读计算型属性（`{ get }`）导入。
- 具有`weak`属性特性的属性，作为标记`weak`关键字（`weak var`）的属性导入。
- 具有`assign`，`copy`，`strong`，`unsafe_unretained`属性特性的属性，导入后会具有相应的内存管理策略。
- 具有`class`属性特性的属性，作为类型属性导入。
- 原子属性特性（`atomic`，`nonatomic`）在 Swift 属性声明中并无相应体现，不过 Objective-C 中具有原子性的属性在 Swift 中将保持其原子性。
- 存取器属性特性（`getter=`，`setter=`）会被 Swift 忽略。

在 Swift 中使用点语法对属性进行存取，直接使用属性名即可，不需要附加圆括号。

例如，如下代码设置了`UITextField`对象的`textColor`和`text`属性：

```swift
myTextField.textColor = UIColor.darkGray
myTextField.text = "Hello world"
```

在 Objective-C，一个有返回值的无参数方法可以像属性那样使用点语法调用。然而，它们会被导入为 Swift 中的实例方法，只有在 Objective-C 中使用`@property`声明的属性才会导入为 Swift 中的属性。方法的导入和调用请参阅 [方法](#working_with_methods) 小节。

<a name="working_with_methods"></a>
## 方法

在 Swift 中使用点语法调用方法。

当 Objective-C 方法导入到 Swift 时，Objective-C 选择器的第一部分将会成为基本方法名并出现在圆括号的前面。第一个参数将直接在圆括号中出现，并且没有参数名。选择器的其余部分作为相应的参数名，出现在方法的圆括号内。选择器的所有部分在调用时都必须写上。

例如，在 Objective-C 这样写：

```objective-c
[myTableView insertSubview:mySubview atIndex:2];
```

在 Swift 则应该这样写：

```swift
myTableView.insertSubview(mySubview, atIndex: 2)
```

如果调用一个无参数的方法，仍须在方法名后面跟上一对圆括号：

```swift
myTableView.layoutIfNeeded()
```

<a name="id_compatibility"></a>
## id 兼容性

Objective-C 中的`id`类型会作为 Swift 中的`Any`类型导入。无论在编译期还是运行期，当 Swift 值或者对象作为`id`参数传入 Objective-C 时，编译器都会引入一种通用的桥接转换操作。当`id`类型作为`Any`类型导入到 Swift 时，运行时会自动处理桥接回相应的 Swift 类类型或值类型的工作。

```swift
var x: Any = "hello" as String
x as? String   // 值为 "hello" 的 String 
x as? NSString // 值为 "hello" 的 NSString

x = "goodbye" as NSString
x as? String   // 值为 "goodbye" 的 String
x as? NSString // 值为 "goodbye" 的 NSString
```

<a name="downcasting_any"></a>
### 将 Any 向下转换

使用一个`Any`类型的对象时，如果其具体类型已知或者可以断定，将其向下转换到具体类型往往会更有用处。然而，由于`Any`类型可以引用任何类型，因此无法保证向下转换到具体类型一定成功。

可以使用条件转换操作符（`as?`），这将返回目标类型的可选值：

```swift
let userDefaults = UserDefaults.standard
let lastRefreshDate = userDefaults.object(forKey: "LastRefreshDate") // lastRefreshDate 类型为 Any?
if let date = lastRefreshDate as? Date {
    print("\(date.timeIntervalSinceReferenceDate)")
}
```

如果确信对象的类型，可以使用强制下转操作符（`as!`）：

```swift
let myDate = lastRefreshDate as! Date
let timeInterval = myDate.timeIntervalSinceReferenceDate
```

然而，一旦强制下转失败，将触发一个运行时错误：

```swift
let myDate = lastRefreshDate as! String // Error
```

<a name="dynamic_method_lookup"></a>
### 动态方法查找

Swift 还引入了`AnyObject`类型来表示一种对象类型，这种对象具有能够动态查找`@objc`方法的能力。这使你在访问返回值为`id`类型的 Objective-C 方法时能保持无类型的灵活性。

例如，你可以将任意类型的对象赋值给一个`AnyObject`类型的常量或者变量，如果是变量，你还可以再用其他类型的对象为其赋值。你还可以在`AnyObject`类型的值上调用任意 Objective-C 方法以及访问属性，而无需将其转换为具体类型。

```swift
var myObject: AnyObject = UITableViewCell()
myObject = NSDate()
let futureDate = myObject.addingTimeInterval(10)
let timeSinceNow = myObject.timeIntervalSinceNow
```

<a name="unrecognized_selectors_and_optional_chaining"></a>
### 无法识别的选择器和可选链语法

因为`AnyObject`类型的值在运行时才能确定其真实类型，所以有可能在不经意间写出不安全代码。在 Swift，和 Objective-C 一样，试图调用不存在的方法将触发未识别的选择器错误。

例如，如下代码在编译时没有任何编译警告，但是会在运行时触发一个错误：

```swift
myObject.character(at: 5)
// 崩溃，myObject 无法响应该方法
```

Swift 使用可选来防止这种不安全的行为。当用`AnyObject`类型的值调用一个 Objective-C 方法时，方法调用在行为上类似于隐式解包可选。可以像调用协议中的可选方法那样，使用可选链语法在`AnyObject`上调用方法。

例如，在下面的代码中，第一和第二行代码不会被执行，因为`count`属性和`character(at:)`方法不存在于`NSDate`对象中。`myCount`常量会被推断为`Int?`，并被赋值为`nil`。你也可以使用`if-let`语句来有条件地解包这种可能无法被响应的方法的返回值，就像第三行所做的一样。

```swift
// myObject 类型为 AnyObject，保存着 NSDate 类型的值
let myCount = myObject.count // myCount 类型为 Int?，其值为 nil
let myChar = myObject.character?(at: 5) // myChar 类型为 unichar?，其值为 nil
if let fifthCharacter = myObject.character?(at: 5) { 
    print("Found \(fifthCharacter) at index 5") // 条件分支不会被执行
}
```

> 注意  
> 尽管在使用`AnyObject`类型的值调用方法时不需要强制解包，但建议使用强制解包的方式来防止一些意想不到的行为。

<a name="nullability_and_optionals"></a>
## 为空性和可选类型

在 Objective-C，用于操作对象的原始指针的值可能会是`NULL`（在 Objective-C 中称为`nil`）。而在 Swift，所有的值，包括结构体与对象的引用，都能确保是非空值。作为替代，可以将可能会值缺失的值包装为该类型的`可选类型`。当你需要表示值缺失的情况时，你可以将其赋值为`nil`。请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的 [可选类型](http://wiki.jikexueyuan.com/project/swift/chapter2/01_The_Basics.html#optionals) 小节获取更多关于可选类型的信息。

在 Objective-C 可以使用`为空性标注`来指明一个参数类型、属性类型或者返回值类型是否可以为`NULL`或`nil`值。单个类型声明可以使用`_Nullable`和`_Nonnull`标注，单个属性声明可以使用`nullable`，`nonnull`，`null_resettable`属性特性，批量标注可以使用`NS_ASSUME_NONNULL_BEGIN`和`NS_ASSUME_NONNULL_END`宏。如果一个类型没有任何为空性标注，Swift 就无法辨别它是可选类型还是非可选类型，因此将其作为隐式解包可选类型导入。

- 以`_Nonnull`或者范围宏标注的类型，会作为`非可选类型`导入到 Swift。
- 以`_Nullable`标注的类型，会作为`可选类型`导入到 Swift。
- 没有为空性标注的类型，会作为`隐式解包可选类型`导入到 Swift。

举个例子，思考如下 Objective-C 声明：

```objective-c
@property (nullable) id nullableProperty;
@property (nonnull) id nonNullProperty;
@property id unannotatedProperty;
 
NS_ASSUME_NONNULL_BEGIN
- (id)returnsNonNullValue;
- (void)takesNonNullParameter:(id)value;
NS_ASSUME_NONNULL_END
 
- (nullable id)returnsNullableValue;
- (void)takesNullableParameter:(nullable id)value;
 
- (id)returnsUnannotatedValue;
- (void)takesUnannotatedParameter:(id)value;
```

导入到 Swift 后将如下所示：

```swift
var nullableProperty: Any?
var nonNullProperty: Any
var unannotatedProperty: Any!

func returnsNonNullValue() -> Any
func takesNonNullParameter(value: Any)

func returnsNullableValue() -> Any?
func takesNullableParameter(value: Any?)

func returnsUnannotatedValue() -> Any!
func takesUnannotatedParameter(value: Any!)
```

大多数 Objective-C 系统框架，包括 Foundation，都已经提供了为空性标注，这使你能以更加类型安全的方式去操作各种值。

<a name="bridging_optionals_to_nonnullable_objects"></a>
### 将可选值桥接为 NSNull 实例

Swift 会根据可选值是否有值来决定是否将其桥接为 Objective-C 的`NSNull`实例。如果可选值的值为`nil`，Swift 会将`nil`值桥接为`NSNull`实例。否则，Swift 会将可选值桥接为它所包装的值。例如，当一个可选值传给一个接受非空`id`类型参数的 Objective-C API 时，或者将一个元素为可选类型的数组（`[T?]`）桥接为`NSArray`时。

如下代码展示了`String?`实例如何根据它是否有值来桥接到 Objective-C。

```objective-c
@implementation OptionalBridging
+ (void)logSomeValue:(nonnull id)valueFromSwift {
    if ([valueFromSwift isKindOfClass: [NSNull class]]) {
        os_log(OS_LOG_DEFAULT, "Received an NSNull value.");
    } else {
        os_log(OS_LOG_DEFAULT, "%s", [valueFromSwift UTF8String]);
    }
}
@end
```

因为`valueFromSwift`参数的类型是`id`，它会作为`Any`类型导入到 Swift。由于很少将可选值传递给`Any`类型，因此在将可选值传递给类方法`logSomeValue(_:)`时需要将参数显式转换为`Any`类型，从而消除编译警告。

```swift
let someValue: String? = "Bridge me, please."
let nilValue: String? = nil

OptionalBridging.logSomeValue(someValue as Any)  // Bridge me, please.
OptionalBridging.logSomeValue(nilValue as Any)   // Received an NSNull value.
```

<a name="protocol_qualified_classes"></a>
## 协议限定类

Objective-C 中的协议限定类会被导入为 Swift 中的协议类型值。例如，如下 Objective-C 方法会在特定类上执行一个操作：

```swift
- (void)doSomethingForClass:(Class<NSCoding>)codingClass;
```

该方法将以如下形式导入到 Swift：

```swift
func doSomething(for codingClass: NSCoding.Type)
```

<a name="lightweight_generics"></a>
## 轻量泛型

Objective-C 的`NSArray`，`NSSet`以及`NSDictionary`类型的声明在使用轻量泛型参数化后，被导入到 Swift 时会附带容器中元素的类型信息。

例如，思考如下 Objective-C 属性声明：

```objective-c
@property NSArray<NSDate *> *dates;
@property NSCache<NSObject *, id<NSDiscardableContent>> *cachedData;
@property NSDictionary <NSString *, NSArray<NSLocale *>> *supportedLocales;
```

导入到 Swift 后将如下所示：

```swift
var dates: [Date]
var cachedData: NSCache<AnyObject, NSDiscardableContent>
var supportedLocales: [String: [Locale]]
```

使用了轻量泛型的 Objective-C 类会作为泛型类导入到 Swift。所有 Objective-C 泛型类型参数导入到 Swift 后，都会带有类型约束来要求该类型必须是一个类类型。如果 Objective-C 泛型参数化带有类限定，导入到 Swift 后，它会带有一个类型约束来要求该类必须是指定类的子类。与之类似，如果带有协议限定，导入到 Swift 后，它会带有一个类型约束来要求该类必须符合指定协议。如果未特别指定类型限定，Swift 会将根据泛型类的类型约束来推断泛型参数化。例如，思考如下 Objective-C 类声明和分类声明：

```objective-c
@interface List<T: id<NSCopying>> : NSObject
- (List<T> *)listByAppendingItemsInList:(List<T> *)otherList;
@end

@interface ListContainer : NSObject
- (List<NSValue *> *)listOfValues;
@end

@interface ListContainer (ObjectList)
- (List *)listOfObjects;
@end
```

上述声明将以如下形式导入到 swift：

```swift
class List<T: NSCopying> : NSObject {
    func listByAppendingItemsInList(otherList: List<T>) -> List<T>
}

class ListContainer : NSObject {
    func listOfValues() -> List<NSValue>
}

extension ListContainer {
    func listOfObjects() -> List<NSCopying>
}
```

<a name="extensions"></a>
## 扩展

Swift 扩展和 Objective-C 分类相似。扩展可以为既有类，结构和枚举扩充功能，即使它们是在 Objective-C 中定义的。通过导入相应的模块，还可以为系统框架中的类型或者自定义类型定义扩展。

例如，可以扩展`UIBezierPath`类，基于三角形的边长与起点来创建一个简单的三角形路径。

```swift
extension UIBezierPath {
    convenience init(triangleSideLength: CGFloat, origin: CGPoint) {
        self.init()
        let squareRoot = CGFloat(sqrt(3.0))
        let altitude = (squareRoot * triangleSideLength) / 2
        move(to: origin)
        addLine(to: CGPoint(x: origin.x + triangleSideLength, y: origin.y))
        addLine(to: CGPoint(x: origin.x + triangleSideLength / 2, y: origin.y + altitude))
        close()
    }
}
```

还可以利用扩展来增加属性（包括类型属性和静态属性）。然而，这些属性必须是计算型属性，因为扩展不能为类，结构体，枚举添加存储型属性。

下面这个例子为`CGRect`结构增加了一个叫`area`的计算型属性：

```swift
extension CGRect {
    var area: CGFloat {
        return width * height
    }
}
let rect = CGRect(x: 0.0, y: 0.0, width: 10.0, height: 50.0)
let area = rect.area
```

甚至可以利用扩展为类添加协议一致性而无需子类化。如果协议是在 Swift 中定义的，还可以为结构或是枚举添加协议一致性，即使它们是在 Objective-C 中定义的。

无法通过扩展覆盖类型中既有的方法与属性。

<a name="closures"></a>
## 闭包

Objective-C 的块会作为符合其调用约定的 Swift 闭包导入，通过`@convention(block)`特性表示。例如，下面是一个 Objective-C 的块变量：

```objective-c
void (^completionBlock)(NSData *) = ^(NSData *data) {
    // ...
}
```

它在 Swift 中如下所示：

```swift
let completionBlock: (Data) -> Void = { data in
    // ...
}
```

Swift 闭包与 Objective-C 块兼容，因此可以把一个 Swift 闭包传递给一个接受块作为参数的 Objective-C 方法。而且因为 Swift 闭包与函数具有相同类型，因此甚至可以直接传递一个 Swift 函数的函数名。

闭包与块有相似的捕获语义，但有个关键的不同：被捕获的变量可以直接修改，而不是一份值拷贝。换言之，Objective-C `__block`变量的行为是 Swift 变量的默认行为。

<a name="avoiding_strong_reference_cycles_when_capturing_self"></a>
### 在捕获 self 时避免强引用循环

在 Objective-C，如果块捕获了`self`，一定要慎重考虑内存管理问题。

块会保持对被捕获对象的强引用，包括`self`。一旦`self`也持有对块的强引用，例如通过一个拷贝语义的块属性，就将导致强引用循环。可以通过让块捕获`self`的弱引用来避免此问题：

```objective-c
__weak typeof(self) weakSelf = self;
self.block = ^{
    __strong typeof(self) strongSelf = weakSelf;
    [strongSelf doSomething];
};
```

和 Objective-C 块类似，Swift 闭包也会保持对被捕获对象的强引用，包括`self`。为了避免强引用循环，可以在闭包的捕获列表中指定`self`为`unowned`：

```swift
self.closure = { [unowned self] in
    self.doSomething()
}
```

请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的 [解决闭包引起的循环强引用](http://wiki.jikexueyuan.com/project/swift/chapter2/16_Automatic_Reference_Counting.html#resolving_strong_reference_cycles_for_closures) 小节获取更多信息。

<a name="object_comparison"></a>
## 对象比较

在 Swift，比较两个对象可以使用两种方式。第一种，使用`==`运算符，比较两个对象内容是否相同。第二种，使用`===`运算符，判断常量或者变量是否引用同一对象。

Swift 为源自`NSObject`的类实现了`Equatable`协议，并提供了`==`运算符和`===`运算符的默认实现。`==`运算符的默认实现会调用`isEqual:`方法，`===`运算符的默认实现则会检查指针是否相同。你不应该为 Objective-C 类型重写这两个运算符。

`NSObject`类为`isEqual:`方法提供的基本实现仅仅是比较两个指针是否相同。可以根据需要在子类中重写`isEqual:`方法，基于对象内容进行比较，而不是比较指针。关于如何实现对象比较逻辑的更多信息，请参阅 [Object comparison](https://developer.apple.com/library/prerelease/ios/documentation/General/Conceptual/DevPedia-CocoaCore/ObjectComparison.html#//apple_ref/doc/uid/TP40008195-CH37)。

> 注意  
> Swift 会自动为`!=`和`!==`运算符提供实现，无需再进行重写。

<a name="hashing"></a>
### 哈希

如果`NSDictionary`的键类型不带有类限定，Swift 会将其导入为键类型为`AnyHashable`的`Dictionary`。与之类似，如果`NSSet`中的元素没有类型限定，Swift 会将其导入为元素类型为`AnyHashable`的`Set`。如果`NSDictionary`或`NSSet`对其键或元素的类型做了参数化，那么 Swift 导入它们时则会使用相应的类型作为键或元素的类型。例如，考虑如下 Objective-C 声明：

```objective-c
@property NSDictionary *unqualifiedDictionary;
@property NSDictionary<NSString *, NSDate *> *qualifiedDictionary;

@property NSSet *unqualifiedSet;
@property NSSet<NSString *> *qualifiedSet;
```

Swift 会以如下形式导入它们： 

```swift
var unqualifiedDictionary: [AnyHashable: Any]
var qualifiedDictionary: [String: Date]

var unqualifiedSet: Set<AnyHashable>
var qualifiedSet: Set<String>
```

Swift 在导入 Objective-C 中未指定的类型或`id`类型时，如果这种类型必须符合`Hashable`协议，就会将之导入为`AnyHashable`类型。`AnyHashable`由任意`Hashable`类型隐式转换而来，并可以使用`as?`和`as!`操作符将其转换为更具体的类型。

<!-- 3.0 文档已移除这段
Swift 为源自`NSObject`的类实现了`Hashable`协议，并提供了`hashValue`属性的默认实现，即调用`NSObject`的`hash`属性。

`NSObject`子类在重写`isEquals:`方法时，还应该重写`hash`属性。
-->

<a name="swift_type_compatibility"></a>
## Swift 类型兼容性

在 Swift 中继承 Objective-C 类时，该 Swift 子类及其成员，即属性、方法、下标和构造器，只要兼容于 Objective-C，就可在 Objective-C 中直接使用。但这不包括一些 Swift 独有特性，如下列所示：

- 泛型
- 元组
- 原始值类型不是`Int`类型的枚举
- 结构体
- 顶级函数
- 全局变量
- 类型别名
- 可变参数
- 嵌套类型
- 柯里化函数

Swift API 转换到 Objective-C 时：

- Swift 可选类型会被标注`__nullable`。
- Swift 非可选类型会被标注`__nonnull`。
- Swift 常量存储属性和计算属性会成为 Objective-C 只读属性。
- Swift 变量存储属性会成为 Objective-C 读写属性。
- Swift 类型属性会成为 Objective-C 中具有`class`特性的属性。
- Swift 类型方法会成为 Objective-C 类方法。
- Swift 构造器和实例方法会成为 Objective-C 实例方法。
- Swift 中抛出错误的方法会成为接受`NSError **`参数的 Objective-C 方法。如果 Swift 方法没有参数，`AndReturnError:`会被拼接到 Objective-C 方法名上，否则，`error:`会被拼接在方法名上。如果 Swift 方法没有指定返回类型，那么相应的 Objective-C 方法会拥有 `BOOL` 类型的返回值。如果 Swift 方法返回非可选值，那么对应的 Objective-C 方法会返回可选值。

例如，思考如下 Swift 声明：

```swift
class Jukebox: NSObject {
    var library: Set<String>

    var nowPlaying: String?

    var isCurrentlyPlaying: Bool {
        return nowPlaying != nil
    }

    class var favoritesPlaylist: [String] {
        // return an array of song names
    }

    init(songs: String...) {
        self.library = Set<String>(songs)
    }

    func playSong(named name: String) throws {
        // play song or throw an error if unavailable
    }
}
```

上述声明导入到 Objective-C 后如下所示：

```objective-c
@interface Jukebox : NSObject
@property (nonatomic, strong, nonnull) NSSet<NSString *> *library;
@property (nonatomic, copy, nullable) NSString *nowPlaying;
@property (nonatomic, readonly, getter=isCurrentlyPlaying) BOOL currentlyPlaying;
@property (nonatomic, class, readonly, nonnull) NSArray<NSString *> * favoritesPlaylist;
- (nonnull instancetype)initWithSongs:(NSArray<NSString *> * __nonnull)songs OBJC_DESIGNATED_INITIALIZER;
- (BOOL)playSong:(NSString * __nonnull)name error:(NSError * __nullable * __null_unspecified)error;
@end
```

> 注意  
> 无法在 Objective-C 中继承一个 Swift 类。

<a name="configuring_swift_interfaces_in_objective_c"></a>
### 配置 Swift 在 Objective-C 中的接口

在某些情况下，需要更精确地控制如何将 Swift API 暴露给 Objective-C。你可以使用`@objc(name)`特性来改变类、属性、方法、枚举类型以及枚举用例暴露给 Objective-C 代码的声明。

例如，如果 Swift 类的类名包含 Objective-C 中不支持的字符，就可以为其提供一个在 Objective-C 中的别名。如果要为 Swift 函数提供 Objective-C 别名，使用 Objective-C 选择器语法，并记得在选择器参数片段后添加冒号（`:`）。

```swift
@objc(Color)
enum Цвет: Int {

    @objc(Red)
    case Красный

    @objc(Black)
    case Черный
}

@objc(Squirrel)
class Белка: NSObject {

    @objc(color)
    var цвет: Цвет = .Красный

    @objc(initWithName:)
    init (имя: String) {
        // ...
    }

    @objc(hideNuts:inTree:)
    func прячьОрехи(количество: Int, вДереве дерево: Дерево) {
        // ...
    }
}
```

对 Swift 类使用`@objc(name)`特性时，该类将在 Objective-C 中可用并且没有任何命名空间。因此，这个特性在迁徙被归档的 Objecive-C 类到 Swift 时会非常有用。由于归档文件中存储了被归档对象的类名，因此应该使用`@objc(name)`特性来指定被归档的 Objective-C 对象的类名，这样归档文件才能通过新的 Swift 类解档。

> 注意  
> 相反，Swift 还提供了`@nonobjc`特性，可以让一个 Swift 声明在 Objective-C 中不可用。可以利用它来解决桥接方法循环，以及重载由 Objective-C 中导入的类中的方法。另外，如果一个 Objective-C 方法在 Swift 中被重写后，无法再以 Objective-C 的语言特性呈现，例如将参数变为了 Swift 中的可变参数，那么这个方法必须标记为`@nonobjc`。

<a name="requiring_dynamic_dispatch"></a>
### 强制动态派发

即使将 Swift API 暴露给 Objective-C 运行时，也无法保证调用属性、方法、下标或构造器时进行动态派发。Swift 编译器依旧可能绕过 Objective-C 运行时，通过消虚拟化或内联成员访问来优化代码的性能。

可以使用`dynamic`修饰符来强制通过运行时系统进行动态派发。一般很少需要强制动态派发，但是，如果要使用键值观察技术或者`method_exchangeImplementations`这种需要在运行时替换方法实现的函数，就必须使用动态派发。如果 Swift 编译器内联了方法的实现或者消虚拟化，新的方法实现就不会被调用了。

> 注意  
> 标记`dynamic`修饰符的声明无法再标记`@nonobjc`特性。

<a name="selectors"></a>
## 选择器

Objective-C 选择器是一种用于引用 Objective-C 方法名的类型。在 Swift，Objective-C 选择器用`Selector`结构体表示。可以使用`#selector`表达式创建一个选择器，将方法名传入即可，例如`#selector(MyViewController.tappedButton(sender:))`。创建 Objective-C 存取方法的选择器时，为了区分，将属性名加上`getter:`或`setter:`标签，例如`#selector(getter: MyViewController.myButton)`。

```swift
import UIKit
class MyViewController: UIViewController {
    let myButton = UIButton(frame: CGRect(x: 0, y: 0, width: 100, height: 50))

    override init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?) {
        super.init(nibName: nibNameOrNil, bundle: nibBundleOrNil)
        let action = #selector(MyViewController.tappedButton)
        myButton.addTarget(self, action: action, for: .touchUpInside)
    }

    func tappedButton(sender: UIButton?) {
        print("tapped button")
    }

    required init?(coder: NSCoder) {
        super.init(coder: coder)
    }
}
```

> 注意  
> Objective-C 方法引用还可以加上圆括号，并使用`as`运算符来消除重载方法之间的歧义，例如`#selector(((UIView.insertSubview(_:at:)) as (UIView) -> (UIView, Int) -> Void))`。

<a name="unsafe_invocation_of_objective_c_methods"></a>
### Objective-C 方法的不安全调用

可以使用`perform(_:)`方法以及它的变体在兼容 Objective-C 的对象上调用方法。使用选择器调用方法并非内在安全的，因为编译器无法保证结果，甚至无法保证对象能否响应选择器。因此，坚决不提倡使用这些 API，除非代码确实依赖于 Objective-C 运行时提供的动态方法决议。例如，实现一个类似`NSResponder`这种在接口中使用了目标-行为设计模式的类。绝大多数情况下，将对象转换为`AnyObject`类型，再使用可选链语法调用方法会更为安全和方便，请参阅 [id 兼容性](#id_compatibility) 小节获取更多信息。

`perform(_:)`方法同步执行，并返回隐式解包可选类型的非托管对象（`Unmanaged<AnyObject>!`），这是因为通过执行选择器返回的值的类型和所有权无法在编译期决定。请参阅 [非托管对象](03-Working%20with%20Cocoa%20Data%20Types.md#%E9%9D%9E%E6%89%98%E7%AE%A1%E5%AF%B9%E8%B1%A1) 小节获取更多信息。与此相反，该方法的一些变体在指定线程执行选择器，或者延迟执行，例如`perform(_:on:with:waitUntilDone:modes:)`和`perform(_:with:afterDelay:)`，它们没有返回值。

```swift
let string: NSString = "Hello, Cocoa!"
let selector = #selector(NSString.lowercased(with:))
let locale = Locale.current
if let result = string.perform(selector, with: locale) {
    print(result.takeUnretainedValue())
}
// 打印输出 "hello, cocoa!"
```

向一个对象发送无法识别的选择器将造成接收者调用`doesNotRecognizeSelector(_:)`方法，其默认实现是抛出`NSInvalidArgumentException`异常。

```swift
let array: NSArray = ["delta", "alpha", "zulu"]
// 下面这句代码不会导致编译错误，因为该选择器存在于 NSDictionary 中
let selector = #selector(NSDictionary.allKeysForObject)
// 下面这句代码将抛出异常，因为 NSArray 无法响应该选择器
array.performSelector(selector)
```

<a name="keys_and_key_paths"></a>
## 键和键路径

在 Objective-C，键是一个表示某个对象特定属性的字符串。键路径是一个由多个键构成的字符串，键之间用点分隔，表示一条能访问到某个对象特定属性的路径。键和键路径经常用于*键值编码*（KVC），这是一种通过字符串来访问某个对象的属性的机制。键和键路径也常用于*键值监听*（KVO），这是一种能让某个对象在另一个对象的特定属性改变时获得通知的机制。

在 Swift，可以用 key-path 表达式创建键路径来访问属性。例如，可以用 `\Animal.name` 这样的 key-path 表达式来访问 `Animal` 类中的 `name` 属性。键路径通过 key-path 表达式创建，其中包含了属性所在类型的类型信息。对一个实例使用键路径所得到的结果就如同直接访问该实例的对应属性一样。key-path 表达式接受属性引用以及链式属性引用，例如 `\Animal.name.count`。

```swift
let llama = Animal(name: "Llama")
let nameAccessor = \Animal.name
let nameCountAccessor = \Animal.name.count

llama[keyPath: nameAccessor]
// "Llama"
llama[keyPath: nameCountAccessor]
// "5"
```

在 Swift，还可以用 `#keyPath` 字符串表达式生成可被编译器检查的键和键路径，并可将之用于 KVC 方法 `value(forKey:)` 和  `value(forKeyPath:)`，以及 KVO 方法 `addObserver(_:forKeyPath:options:context:)`。`#keyPath` 字符串表达式接受链式的方法或属性引用。它还支持可选值链式引用，例如 `#keyPath(Person.bestFriend.name)`。与使用 key-path 表达式这种创建方式不同的是，`#keyPath` 字符串表达式不会将属性或方法所属类型的类型信息传递给接受键路径的 API。

> 注意  
> `#keyPath` 表达式语法类似 `#selector` 表达式语法，请参阅 [选择器](#selectors)。

```swift
class Person: NSObject {
    @objc var name: String
    @objc var friends: [Person] = []
    @objc var bestFriend: Person? = nil

    init(name: String) {
        self.name = name
    }
}

let gabrielle = Person(name: "Gabrielle")
let jim = Person(name: "Jim")
let yuanyuan = Person(name: "Yuanyuan")
gabrielle.friends = [jim, yuanyuan]
gabrielle.bestFriend = yuanyuan

#keyPath(Person.name)
// "name"
gabrielle.value(forKey: #keyPath(Person.name))
// "Gabrielle"
#keyPath(Person.bestFriend.name)
// "bestFriend.name"
gabrielle.value(forKeyPath: #keyPath(Person.bestFriend.name))
// "Yuanyuan"
#keyPath(Person.friends.name)
// "friends.name"
gabrielle.value(forKeyPath: #keyPath(Person.friends.name))
// ["Yuanyuan", "Jim"]
```
