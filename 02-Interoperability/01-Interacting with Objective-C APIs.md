# 与 Objective-C API 交互

本页包含内容：

- [初始化](#initialization)
- [访问属性](#accessing_properties)
- [方法](#working_with_methods)
- [id 兼容性](#id_compatibility)
- [为空性和可选类型](#Nullability_and_Optionals)
- [轻量泛型](#Lightweight_Generics)
- [扩展](#extensions)
- [闭包](#closures)
- [对象比较](#object_comparison)
- [Swift 类型兼容性](#swift_type_compatibility)
- [Objective-C 选择器](#objective_c_selectors)

*互用性* 是能让 Swift 和 Objective-C 相接合的特性，这使你能够在一种语言编写的文件中使用另一种语言。当你准备开始把 Swift 融入到你的开发流程中时，学会如何利用互用性来重新定义，改善并增强你编写 Cocoa 应用的方式真是极好的。

互用性很重要的一点就是允许你在编写 Swift 代码时使用 Objective-C API。当你导入一个 Objective-C 框架后，你可以使用原生的 Swift 语法实例化这些类并与之交互。

<a name="initialization"></a>
## 初始化

要在 Swift 中实例化一个 Objective-C 类，你应该使用 Swift 语法调用它的一个构造器。当 Objective-C 的`init`方法导入到 Swift 后，它们将用原生的 Swift 构造语法呈现。构造方法的“init”前缀会被截断，并作为一个关键字，用来表明该方法是构造方法。那些以“initWith”开头的`init`方法，“With”也会被去除。然后再将首字母变成小写，并作为 Swift 构造方法第一个参数的参数名。余下的每一部分方法名依次作为剩余的各个参数的参数名。所有这些方法名片段都位于构造方法的圆括号中，并且在调用构造方法时必须写上。

举个例子，你在 Objective-C 中会这样写：

```objective-c
UITableView *myTableView = [[UITableView alloc] initWithFrame:CGRectZero 
                                                        style:UITableViewStyleGrouped];
```
在 Swift 中，你应该这样写：

```swift
let myTableView: UITableView = UITableView(frame: CGRectZero, style: .Grouped)
```

你不需要调用`alloc`，Swift 会为你处理。注意，调用任何 Swift 构造方法时都不会出现“init”单词。

你可以在初始化时显式地声明对象的类型，也可以忽略它，Swift 能正确推断对象的类型。

```swift
let myTextField = UITextField(frame: CGRect(x: 0.0, y: 0.0, width: 200.0, height: 40.0))
```

这里的`UITableView`和`UITextField`对象和你在 Objective-C 中使用的一样。你可以用同样的方式使用它们，例如访问属性或者调用各自类中的方法。

为了统一和简洁，Objective-C 的工厂方法在 Swift 中映射为便利构造器，从而能使用同样简洁明了的构造语法。例如，在 Objective-C 中你可能会像下面这样调用一个工厂方法：

```objective-c
UIColor *color = [UIColor colorWithRed:0.5 green:0.0 blue:0.5 alpha:1.0];
```

在 Swift 中，你应该这样写：

```swift
let color = UIColor(red: 0.5, green: 0.0, blue: 0.5, alpha: 1.0)
```

### 可失败初始化

在 Objective-C 中，构造器会直接返回它们初始化的对象。初始化失败时，为了告知调用者，Objective-C 中的构造器会返回`nil`。在 Swift 中，这种模式被内置到语言特性中，被称为*可失败初始化*。

许多在 iOS 和 OS X 系统框架中的 Objective-C 构造器在导入到 Swift 时会被检查初始化是否可能会失败。你可以在你的 Objective-C 代码中，使用 *为空性注释* 来指明初始化是否可能会失败，正如[为空性和可选类型](#Nullability_and_Optionals)中所描述的那样。如果初始化不会失败，这些 Objective-C 构造器便会作为`init(...)`导入，而如果初始化可能会失败，则会作为`init?(...)`导入。在没有任何为空性注释的情况下，构造器会作为`init!(...)`导入。

例如，当指定路径的图片文件不存在时，`UIImage(contentsOfFile:)`构造器初始化`UIImage`对象便会失败。而如果初始化成功，则可以用可选绑定对可选类型的返回值进行解包。

```swift
if let image = UIImage(contentsOfFile: "MyImage.png") {
    // loaded the image successfully
} else {
    // could not load the image
}
```

<a name="accessing_properties"></a>
## 访问属性

在 Objective-C 中使用`@property`关键字声明的属性，导入到 Swift 时，会遵循以下规则：

- 拥有为空性属性特性的属性（`nonnull`，`nullable`，`null_resettable`），作为可选类型或者非可选类型的属性导入。参见[为空性和可选类型](#Nullability_and_Optionals)中的描述。
- 拥有`readonly`属性特性的属性，作为只读计算属性（`{ get }`）导入。
- 拥有`weak`属性特性的属性，作为标记`weak`关键字（`weak var`）的属性导入。
- 拥有`assign`，`copy`，`strong`，`unsafe_unretained`属性特性的属性，导入后会具有相应的内存管理策略。
- 原子属性特性（`atomic`，`nonatomic`）会被忽略，在 Swift 中所有属性都是`nonatomic`的。
- 存取器属性特性（`getter=`，`setter=`）也会被忽略。

在 Swift 中使用点语法对属性进行存取，直接使用属性名即可，不需要附加圆括号：

```swift
myTextField.textColor = UIColor.darkGrayColor()
myTextField.text = "Hello world"
```

> 注意  
> `darkGrayColor()`后跟一对圆括号，因为它是一个类方法，而不是一个属性。

在 Objective-C 中，一个有返回值的无参数方法可以像属性那样使用点语法调用。但在 Swift 中不能再这样了，因为它们会被导入为实例方法，只有在 Objective-C 中使用`@property`关键字声明的属性才会被作为属性导入。方法的导入和调用请参见[方法](#working_with_methods)中的描述。

<a name="working_with_methods"></a>
## 方法

在 Swift 中使用点语法调用方法。

当 Objective-C 方法导入到 Swift 时，Objective-C 选择器的第一部分将会成为方法名并出现在圆括号的前面，而第一个参数将直接在圆括号中出现，并且没有参数名，剩下的参数名与参数一一对应，并且都在方法的圆括号中。方法名的所有部分在调用时都必须写上。

举个例子，你在 Objective-C 中会这样写：

```objective-c
[myTableView insertSubview:mySubview atIndex:2];
```

在 Swift 中，你应该这样写：

```swift
myTableView.insertSubview(mySubview, atIndex: 2)
```

如果你调用一个无参数的方法，依旧必须在方法名后面跟上一对圆括号：

```swift
myTableView.layoutIfNeeded()
```

<a name="id_compatibility"></a>
## id 兼容性

Swift 中有一种叫做`AnyObject`的协议类型，用于表示任意类型对象，就像 Objective-C 中的`id`一样。`AnyObject`协议使你能编写类型安全的 Swift 代码，同时还维持了无类型对象的灵活性。因为`AnyObject`协议提供了安全保证，Swift 将`id`类型导入为`AnyObject`类型。

举个例子，和`id`类型一样，你可以为`AnyObject`类型的常量或变量分配任意类型对象。如果是变量，你还可以再次为其重新分配任意类型对象。

```swift
var myObject: AnyObject = UITableViewCell()
myObject = NSDate()
```

你也可以在调用 Objective-C 方法或者访问属性时不将它转换为具体的类型，包括标记`@objc`的 Objective-C 兼容方法。

```swift
let futureDate = myObject.dateByAddingTimeInterval(10)
let timeSinceNow = myObject.timeIntervalSinceNow
```

然而，因为直到运行时才能确定`AnyObject`类型对象的真实类型，所以有可能在不经意间写出不安全代码。和 Objective-C 一样，如果用`AnyObject`类型对象调用不存在的方法，将引发运行时错误。比如下面的代码在运行时将会引发一个选择器未识别的的错误：

```swift
myObject.characterAtIndex(5)
// crash, myObject does't respond to that method
```

不过，你可以利用 Swift 中可选类型的特性来避免这个 Objective-C 中的常见错误，当你用`AnyObject`类型对象调用一个 Objective-C 方法时，你可以像调用协议中的可选方法那样，使用可选链语法在`AnyObject`类型对象上调用方法。

> 注意  
> 通过`AnyObject`类型对象访问属性将总是返回一个可选类型的值,而不会引发运行时错误。

举个例子，在下面的代码中，第一和第二行代码不会被执行，因为`count`属性和`characterAtIndex:`方法不存在于`NSDate`对象中。`myCount`常量会被推断成可选类型的`Int`并被赋值为`nil`。你也可以使用`if-let`语句来解包这种未必存在的方法的返回值，就像第三行所做的一样。

```swift
let myCount = myObject.count
let myChar = myObject.characterAtIndex?(5)
if let fifthCharacter = myObject.characterAtIndex?(5) {
    print("Found \(fifthCharacter) at index 5")
}
```

在使用`AnyObject`类型对象时，如果知道其真实类型，那么将其向下转换到真实类型当然是极好的。然而，因为`AnyObject`可以表示任何类型，所以转换到具体类型时难以保证必定成功。你可以使用条件下转操作符（`as?`）来转换，这将返回下转操作目标类型的可选类型：

```swift
let userDefaults = NSUserDefaults.standardUserDefaults()
let lastRefreshDate: AnyObject? = userDefaults.objectForKey("LastRefreshDate")
if let date = lastRefreshDate as? NSDate {
    println("\(date.timeIntervalSinceReferenceDate)")
}
```

当然，如果你能完全确定这个对象的类型，你可以使用强制下转操作符（`as!`）来进行强制转换，这将返回下转操作目标类型的非可选类型：

```swift
let myDate = lastRefreshDate as! NSDate
let timeInterval = myDate.timeIntervalSinceReferenceDate
```

然而，如果强制下转失败，将导致一个运行时错误：

```swift
let myDate = lastRefreshDate as! NSString // Error
```

因此，你应该只在完全确定`AnyObject`类型对象的真实类型时才使用强制下转。

<a name="Nullability_and_Optionals"></a>
## 为空性和可选类型

在 Objective-C 中，用于操作对象的原始指针的值可能会是`NULL`（在 Objective-C 中称为`nil`）。而在 Swift 中，所有的值，包括结构体与对象的引用，都能确保是非空值。作为替代，你可以将可能会值缺失的值包装为该类型的 *可选类型*。当你需要表示值缺失的情况时，你可以将其赋值为`nil`。更多关于可选类型的信息，可以参看 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的[可选类型](http://wiki.jikexueyuan.com/project/swift/chapter2/01_The_Basics.html#optionals)部分。

Objective-C 可以使用 *为空性注释* 来指明一个参数类型，属性类型或者返回值类型是否可以为`NULL`或者`nil`值。单个类型声明可以使用`_Nullable`和`_Nonnull`注释，单个属性声明可以使用`nullable`，`nonnull`，`null_resettable`属性特性，大范围注释可以使用`NS_ASSUME_NONNULL_BEGIN`和`NS_ASSUME_NONNULL_END`这对宏。如果一个类型没有任何为空性注释，Swift 将无法分辨出它是可选类型还是非可选类型，并将其作为隐式解包可选类型导入。

- 以`_Nonnull`或者范围宏注释的类型，会作为 *非可选类型* 导入到 Swift。
- 以`_Nullable`注释的类型，会作为 *可选类型* 导入到 Swift。
- 没有为空性注释的类型，会作为 *隐式解包可选类型* 导入到 Swift。

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

当它们导入到 Swift 后：

```Swift
var nullableProperty: AnyObject?
var nonNullProperty: AnyObject
var unannotatedProperty: AnyObject!
 
func returnsNonNullValue() -> AnyObject
func takesNonNullParameter(value: AnyObject)
 
func returnsNullableValue() -> AnyObject?
func takesNullableParameter(value: AnyObject?)
 
func returnsUnannotatedValue() -> AnyObject!
func takesUnannotatedParameter(value: AnyObject!)
```

大多数 Objective-C 系统框架，包括 Foundation，都已经提供了为空性注释，使你能以符合原有习惯且更加类型安全的方式去使用它们。

<a name="Lightweight_Generics"></a>
## 轻量泛型

Objective-C 中的`NSArray`，`NSSet`以及`NSDictionary`类型的声明在使用轻量泛型参数化后，被导入到 Swift 时会附带容器中元素的类型信息。

举个例子，思考如下 Objective-C 属性声明：

```objective-c
@property NSArray<NSDate *>* dates;
@property NSSet<NSString *>* words;
@property NSDictionary<NSURL *, NSData *>* cachedData;
```

导入到 Swift 后：

```swift
var dates: [NSDate]
var words: Set<String>
var cachedData: [NSURL: NSData]
```

> 注意  
> 除了 Foundation 中的集合类型，其他 Objective-C 类型使用轻量泛型会被 Swift 忽略。

<a name="extensions"></a>
## 扩展

Swift 的扩展和 Objective-C 的分类相似。扩展可以为既有类，结构和枚举扩充功能，即使它们是在 Objective-C 中定义的。通过导入相应的模块，你还可以为系统框架中的类型或者你自己的类型定义扩展。

例如，你可以扩展`UIBezierPath`类来创建一个简单的三角形路径，这个方法只需提供三角形的边长与起点。

```swift
extension UIBezierPath {
    convenience init(triangleSideLength: CGFloat, origin: CGPoint) {
        self.init()
        let squareRoot = CGFloat(sqrt(3.0))
        let altitude = (squareRoot * triangleSideLength) / 2
        moveToPoint(origin)
        addLineToPoint(CGPoint(x: origin.x + triangleSideLength, y: origin.y))
        addLineToPoint(CGPoint(x: origin.x + triangleSideLength / 2, y: origin.y + altitude))
        closePath()
    }
}
```

你还可以利用扩展来增加属性（包括类型属性和静态属性）。然而，这些属性必须是计算型属性，因为扩展不能为类，结构体，枚举添加存储型属性。

下面这个例子为`CGRect`类增加了一个叫`area`的计算型属性：

```swift
extension CGRect {
    var area: CGFloat {
        return width * height
    }
}
let rect = CGRect(x: 0.0, y: 0.0, width: 10.0, height: 50.0)
let area = rect.area
```

你甚至可以利用扩展为类添加协议符合性而无需子类化。如果协议是在 Swift 中定义的，你还可以为结构或是枚举添加协议符合性，即使它们是在 Objective-C 中定义的。

注意，你无法通过扩展来覆盖类型中既有的方法与属性。

<a name="closures"></a>
## 闭包

Objective-C 中的 block 会作为符合其调用约定的 Swift 闭包导入，通过`@convention(block)`属性表示。例如，下面是一个 Objective-C 中的 block 变量：

```objective-c
void (^completionBlock)(NSData *, NSError *) = ^(NSData *data, NSError *error) {
    // ...
}
```

它在 Swift 中像下面这样：

```swift
let completionBlock: (NSData, NSError) -> Void = { (data, error) in 
    // ...
}
```

Swift 闭包与 Objective-C block 兼容，因此你可以把一个 Swift 闭包传递给一个接收 block 作为参数的 Objective-C 方法。而且因为 Swift 闭包与函数类型相同，你甚至可以直接传递一个 Swift 函数的函数名过去。

闭包与 block 有相似的捕获语义，但有个关键的不同：被捕获的变量是可以直接修改的，而 block 在默认情况下捕获的只是变量的值拷贝。换句话说，Swift 闭包在默认情况下捕获的变量就等效于 Objective-C 中的`__block`变量。

### 避免强引用循环

在 Objective-C 中，如果 block 捕获了`self`，一定要慎重考虑内存管理问题。

block 会维持对被捕获对象的强引用，包括`self`。一旦`self`也持有对 block 的强引用，例如一个 copy 语义的 block 属性，就将导致强引用循环。你可以让 block 捕获`self`的弱引用来避免此问题：

```objective-c
__weak typeof(self) weakSelf = self;
self.block = ^{
    __strong typeof(self) strongSelf = weakSelf;
    [strongSelf doSomething];
};
```

和 Objective-C block 类似，Swift 闭包也会维持对被捕获对象的强引用，包括`self`。为了避免强引用循环，你可以在闭包的捕获列表中指定`self`为`unowned`:

```swift
self.closure = { [unowned self] in
    self.doSomething()
}
```

可以参看 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中[解决闭包引起的循环强引用](http://wiki.jikexueyuan.com/project/swift/chapter2/16_Automatic_Reference_Counting.html#resolving_strong_reference_cycles_for_closures)小节来获取更多信息。

<a name="object_comparison"></a>
## 对象比较

在 Swift 中，比较两个对象可以使用两种方式。第一种，使用 ==，比较两个对象内容是否相同。第二种，使用 === ，判断常量或者变量是否引用同一个对象。

Swift 的 == 操作符为继承自`NSObject`类的对象提供了默认实现。在 == 操作符的实现中，Swift 调用`NSObject`类中定义的`isEqual:`方法。`NSObject`类的实现仅仅比较两个对象地址是否相同，所以你需要在你自己的类中重新实现`isEqual:`方法。因为你可以直接传递 Swift 对象（包括那些不继承自`NSObject`的）给 Objective-C API，所以如果你希望比较两个对象的内容而不仅仅是比较其地址的话，你应该为这些对象实现`isEqual:`方法。

作为实现比较的一部分，确保根据 [Object comparison](https://developer.apple.com/library/prerelease/ios/documentation/General/Conceptual/DevPedia-CocoaCore/ObjectComparison.html#//apple_ref/doc/uid/TP40008195-CH37) 中的规则实现对象的`hash`属性。如果你希望你的类的实例能够作为字典中的键，还需要符合`Hashable`协议并实现`hashValues`属性。

<a name="swift_type_compatibility"></a>
## Swift 类型兼容性

当你在 Swift 中创建了一个继承自 Objective-C 类的子类时，该类以及该类的成员：属性，方法，下标和构造器，便会在 Objective-C 中自动可用。在某些情况下，你需要更细粒度地控制如何将 Swift API 暴露给 Objective-C。如果你的 Swift 类没有继承自 Objective-C 的类，又或者你想更改暴露给 Objective-C 代码的接口中的符号名称，你便可以使用`@objc`属性。如果你正在使用诸如键值观察这种需要动态替换方法实现的 API，还可以通过使用`dynamic`修饰符来要求访问成员变量时通过 Objective-C 运行时的动态派发。

### 在 Objective-C 中暴露 Swift 接口（Exposing Swift Interfaces in Objective-C）

当你定义一个继承自`NSObject`类或者其他 Objective-C 类的 Swift 子类时，该类便会自动兼容 Objective-C。Swift 编译器已经为你做好了这部分所需要的工作。所有的步骤都由 Swift 编译器自动完成，如果你不会在 Objective-C 代码中导入 Swift 类，你也不需要担心类型适配问题。否则，如果你的 Swift 类并不继承于 Objective-C 类而你希望能在 Objective-C 的代码中使用它，你可以使用下面描述的`@objc`属性。

`@objc`可以让你的 Swift API 在 Objective-C 以及 Objective-C runtime 中使用。换句话说，你可以通过在任何 Swift 方法，属性，下标，构造器，类，枚举前添加`@objc`，从而让它们可以在 Objective-C 代码中使用。

> 注意  

> 嵌套类型不能标记`objc`属性。  
> 另外只有原始值为整形的 Swift 枚举，例如`Int`，才能使用`objc`属性。

如果你的类继承自 Objective-C，编译器会自动帮助你完成这一步。而如果一个类前面加上了`@objc`关键字，编译器就会在类中所有成员前加`@objc`。当你使用`@IBOutlet`，`@IBAction`，或者是`@NSManaged`属性时，`@objc`属性也会自动添加。当一个 Objetive-C 类使用选择器来实现 target-action 设计模式时，也要用到此属性，例如，`NSTimer`或者`UIButton`。

> 注意

> 如果标记了`private`访问级别修饰符，编译器将不会自动插入`@objc`属性。

当你在 Objective-C 中使用 Swift API 时，编译器通常会对语句做直接翻译。

例如，Swift API `func playSong(name: String)`在 Objective-C 中会被解释为`- (void)playSong:(NSString *)name`。然而，有一个例外：当在 Objective-C 中使用 Swift 的初始化函数时，编译器会在方法前添加“initWith”并且将原初始化函数的第一个参数首字母大写。

例如，这个 Swift 初始化函数`init(songName: String, artist: String)`在 Objective-C 中将被翻译为`- (instancetype)initWithSongName:(NSString *)songName artist:(NSString *)artist`。

Swift 同时也提供了一个`@objc`属性的变体，通过它你可以指定转换到 Objective-C 后的符号名。例如，如果你的 Swift 类的名字包含 Objective-C 中不支持的字符，你就可以为 Objective-C 提供一个可供替代的名字。如果你要为 Swift 函数提供一个 Objective-C 选择器风格的名字，记得在参数名后添加冒号：

```swift
@objc(Squirrel)
class Белка: NSObject {

    @objc(initWithName:)
    init (имя: String) { 
        // ...
    }
    
    @objc(hideNuts:inTree:)
    func прячьОрехи(Int, вДереве: Дерево) {
       // ...
    }
}
```

当你在 Swift 类中使用`@objc(name)`关键字时，该类可在 Objective-C 没有任何命名空间。因此，这个关键字在你迁徙被归档的 Objecive-C 类到 Swift 时非常有用。由于归档过的对象存储了类的名字，你应该使用`@objc(name)`来声明与旧的归档过的类相同的名字，这样旧的类才能被新的 Swift 类解档。

### 要求动态派发（Requiring Dynamic Dispatch）

`@objc`属性将你的 Swift API 暴露给了 Objective-C runtime，但是它并不能保证属性，方法，下标，或构造器的动态派发。Swift 编译器可能仍然通过绕过 Objective-C runtime 来消虚拟化（devirtualize）或内联成员访问来优化代码的性能。当一个成员声明用`dynamic`修饰符标记时，对该成员的访问将始终是动态派发的。由于标记`dynamic`修饰符后需要用 Objective-C runtime 来派发，因此会被隐式地标记`@objc`属性。

一般很少需要动态派发。但是，当你要在运行时替换一个 API 的实现时你必须使用`dynamic`修饰符。例如，你可以使用 Objective-C runtime 的`method_exchangeImplementations`函数在应用程序运行中替换某个方法的实现。如果 Swift 编译器内联了方法的实现或者消虚拟化对它的访问，新的实现将不会被使用。

<a name="objective_c_selectors"></a>
## Objective-C 选择器

Objective-C 选择器是一种用于引用 Objective-C 方法名的类型。在 Swift 中，Objective-C 选择器用`Selector`结构体表示。你可以通过字符串字面量创建一个选择器，比如`let mySelector: Selector = "tappedButton:"`。因为字符串字面量能够自动被转换为选择器，所以你可以把字符串字面量直接传递给任何能够接受选择器的方法。

```swift
import UIKit
class MyViewController: UIViewController {

    let myButton = UIButton(frame: CGRect(x: 0, y: 0, width: 100, height: 50))
    
    override init?(nibName nibNameOrNil: String?, bundle nibBundleOrNil: NSBundle?) {
        super.init(nibName: nibNameOrNil, bundle: nibBundleOrNil)
        myButton.addTarget(self, action: "tappedButton:", forControlEvents: .TouchUpInside)
    }
    
    func tappedButton(sender: UIButton!) {
        print("tapped button")
    }
    
    required init?(coder: NSCoder) {
        super.init(coder: coder)
    }
}
```

如果你的 Swift 类继承自 Objective-C 类，那么所有方法和属性都可以用于 Objective-C 选择器。反之，如果你的 Swift 类不是继承自 Objective-C 类，你需要在要用于选择器的方法前面加上`@objc`属性前缀，详情请看 [Swift 类型兼容性](#swift_type_compatibility)。

### 使用 performSelector 发送消息

你可以使用`performSelector(_:)`方法以及它的变体向 Objective-C 兼容的对象发送消息。

`performSelector`系列 API 可以向指定线程发送消息，或者延迟发送没有返回值的消息。该系列 API 同步执行，并返回隐式解包可选的非托管对象（`Unmanaged<AnyObject>!`），因为返回值的类型和所有权无法在编译期决定。可以查看[非托管对象](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/03Working%20with%20Cocoa%20Data%20Types.md#%E9%9D%9E%E6%89%98%E7%AE%A1%E5%AF%B9%E8%B1%A1)获取更多信息。

```swift
let string: NSString = "Hello, Cocoa!"
let selector: Selector = "lowercaseString"
if let result = string.performSelector(selector) {
    print(result.takeUnretainedValue())
}
// prints "hello, cocoa!"
```

向一个对象发送无法识别的选择器将造成接收者调用`doesNotRecognizeSelector(_:)`方法，其默认实现是引发`NSInvalidArgumentException`异常。

```swift
let array: NSArray = ["delta", "alpha", "zulu"]
let invalidSelector: Selector = "invalid"
array.performSelector(invalidSelector) // raises an exception
```

在 Objective-C 运行时中直接向对象发送消息是内在不安全的，因为编译器无法保证消息发送的结果，或是消息是否能在第一时间被处理。同样的，不鼓励使用`performSelector`系列 API，除非你的代码确实依赖于 Objective-C 运行时提供的动态方法决议。否则，正如 [id 兼容性](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/01Interacting%20with%20Objective-C%20APIs.md#id-%E5%85%BC%E5%AE%B9%E6%80%A7)中所描述的那样，使用可选链向`AnyObject`类型发送消息更为安全方便。
