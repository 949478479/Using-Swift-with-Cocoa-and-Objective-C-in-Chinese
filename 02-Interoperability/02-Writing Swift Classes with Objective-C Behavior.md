# 使用 Objective-C 特性编写 Swift 类

本节包括内容：

- [继承 Objective-C 类](#inheriting_from_objective-c_classes)
- [采纳协议](#adopting_protocols)
- [编写构造器和析构器](#writing_initializers_and_deinitializers)
- [兼容使用 Swift 类名的 Objective-C API](#using_swift_class_names_with_objective_c_apis)
- [与 Interface Builder 结合](#integrating_with_interface_builder)
- [指定属性特性](#specifying_property_attributes)
- [子类化 NSManagedObject](#implementing_core_data_managed_object_subclasses)

*互用性* 让你可以编写融合了 Objective-C 语言特性的 Swift 类。在编写 Swift 类时，你不仅可以继承 Objective-C 类，采纳 Objective-C 协议，还可以使用 Objective-C 的一些其它功能。这意味着你可以基于 Objective-C 中耳熟能详的既有特性来编写 Swift 类，还可以结合 Swift 提供的更为强大的现代化语言特性对其进行改进。

<a name="inheriting_from_objective-c_classes"></a>
## 继承 Objective-C 类

在 Swift 中，你可以定义一个继承自 Objective-C 类的 Swift 子类。在 Swift 的类名后面加上一个冒号（:），冒号后面跟上 Objective-C 类的类名即可。

```swift
import UIKit
class MySwiftViewController: UIViewController {
	// 定义类
}
```

你能从 Objective-C 父类中继承所有的功能。如果你要覆盖父类中的实现，不要忘记使用`override`关键字。

### NSCoding 协议

`NSCoding`协议要求采纳协议的类型实现其所要求的构造器`init(coder:)`。直接采纳`NSCoding`协议的类必须实现这个方法。对于采纳`NSCoding`协议的类的子类，如果有一个或者多个自定义的构造器或者有不带初始值的属性，也必须实现这个方法。Xcode 提供了下面这个 fix-it 来提供一个占位实现：

```swift
required init(coder aDecoder: NSCoder) {
	fatalError("init(coder:) has not been implemented")
}
```

对那些从 Storyboards 里加载的对象，或者用`NSUserDefaults`或`NSKeyedArchiver`类归档到磁盘的对象，你必须提供该构造器的完整实现。当然，当一个类型不会以此种方式实例化的时候，你并不需要提供该构造器的完整实现。

<a name="adopting_protocols"></a>
## 采纳协议

Objective-C 协议会被导入为 Swift 协议。所有协议都写在一个用逗号隔开的列表中，跟在父类类名后面（如果该类有父类的话）。

```swift
class MySwiftViewController: UIViewController, UITableViewDelegate, UITableViewDataSource {
    // 定义类
}
```

声明符合单个协议的类型时，直接使用协议名作为其类型，类似于 Objective-C 中`id<SomeProtocol>`这种形式。声明符合多个协议的类型时，使用`protocol<SomeProtocol, AnotherProtocol>`这种协议组合的形式，类似于 Objective-C 中`id<SomeProtocol, AnotherProtocol>`这种形式。

```swift
var textFieldDelegate: UITextFieldDelegate
var tableViewController: protocol<UITableViewDataSource, UITableViewDelegate>
```

> 注意  
> 在 Swift 中，类和协议的命名空间是统一的，因此 Objective-C 的`NSObject`协议会被重映射为 Swift 的`NSObjectProtocol`。

<a name="writing_initializers_and_deinitializers"></a>
## 编写构造器和析构器

Swift 的编译器确保在初始化后类里不会有任何未初始化的属性，这样做能够增加代码的安全性和可预测性。另外，与 Objective-C 不同，Swift 不提供单独的内存分配方法供开发者调用。当你使用原生的 Swift 初始化方法时（即使是和 Objective-C 类协作），Swift 会将 Objective-C 的初始化方法转换为 Swift 的初始化方法。关于如何实现自定义构造器的更多信息，请查看[《The Swift Programming Language 中文版》](http://wiki.jikexueyuan.com/project/swift/)中的 [构造器（Initializers）](http://wiki.jikexueyuan.com/project/swift/chapter2/14_Initialization.html)部分。

当你希望在实例被释放前，执行额外的清理工作时，你可以实现一个析构器来代替`dealloc`方法。在实例被释放前，Swift 会自动调用析构器来执行析构过程。Swift 调用完子类的析构器后，会自动调用父类的析构器。当你使用 Objective-C 类或者是继承自 Objective-C 类的 Swift 类时，Swift 也会自动为开发者调用这个类的父类里的`dealloc`方法。关于如何实现自定义析构器的更多信息，请查看[《The Swift Programming Language 中文版》](http://wiki.jikexueyuan.com/project/swift/)中的 [析构器（Deinitializers）](http://wiki.jikexueyuan.com/project/swift/chapter2/15_Deinitialization.html)部分。

<a name="integrating_with_interface_builder"></a>
## 集成 Interface Builder（Integrating with Interface Builder）

Swift 编译器包含一些特性，使你的 Swift 类集成了 Interface Builder 里的一些特色功能。和 Objective-C 里一样，你能在 Swift 里面使用 outlets，actions 和实时渲染（live rendering）。

### 使用 Outlet 和 Action（Working with Outlets and Actions）

使用 Outlet 和 Action 可以连接源代码和 Interface Builder 中的 UI 对象。在 Swift 中使用 Outlet 和 Action，需要在属性或方法声明前插入`@IBOutlet`或`@IBAction`关键字。声明一个 Outlet 集合同样是用`@IBOutlet`属性，只不过为该类型指定了一个数组。

当你在 Swift 中声明一个 Outlet 时，你应该将类型声明为隐式解包可选。通过这种方式，Swift 编译器会自动为它分配一个空值`nil`，因此你就不需要在构造器中为其分配一个初始值了。初始化过程完成后，storyboard 会将 Outlet 进行连接。如果你从 storyboard 或者`xib`文件里面初始化对象，你可以认为 Outlet 已经连接好了。

例如，下面的 Swift 代码声明了一个拥有 Outlet、Outlet 集合和 Action 的类：

```swift
class MyViewController: UIViewController {

    @IBOutlet weak var button: UIButton!
    
    @IBOutlet var textFields: [UITextField]!
    
    @IBAction func buttonTapped(_: AnyObject) {
        print("button tapped!")
    }
}
```

在`buttonTapped`方法中，消息发送者的信息没有被使用，因此可以省略该方法的参数名。

### 实时渲染（live rendering）

你可以在 Interface Builder 中用`@IBDesignable`和`@IBInspectable`来创建生动的交互式自定义视图。你继承`UIView`或者`NSView`来自定义一个视图时，可以在类声明前添加`@IBDesignable`属性。当你在 Interface Builder 里添加了自定义的视图后（在监视器面板的自定义视图类中进行设置），Interface Builder 将在画布上渲染你自定义的视图。

你还可以将`@IBInspectable`属性添加到和用户定义的运行时属性（user defined runtime attributes）类型兼容的属性前。这样，当你将自定义的视图添加到 Interface Builder 中后，就可以在监视器面板中编辑这些属性。

```swift
@IBDesignable
class MyCustomView: UIView {
	@IBInspectable var textColor: UIColor
	@IBInspectable var iconHeight: CGFloat
	// ...
}
```

<a name="specifying_property_attributes"></a>
## 指明属性特性（Specifying Property Attributes）

在 Objective-C 中，属性通常都有一系列属性特性来指明该属性的一些附加信息。在 Swift 中，你可以通过不同的方法来指明属性的这些特性。

### 强类型和弱类型（Strong and Weak）

Swift 里属性默认都是强类型的。使用`weak`关键字修饰一个属性，指明该属性持有其存储对象的弱引用。该关键字仅能修饰可选类型（optional）。更多的信息，请查阅[《The Swift Programming Language 中文版》](http://wiki.jikexueyuan.com/project/swift/)中的 [特性（Attributes）](http://wiki.jikexueyuan.com/project/swift/chapter2/09_Classes_and_Structures.html)。

### 读／写和只读（Read/Write and Read-Only）

在 Swift 中，没有`readwrite`和`readonly`特性。当声明一个存储型属性时，使用`let`修饰其为只读；使用`var`修饰其为可读／写。当声明一个计算型属性时，为其提供一个 getter 方法，使其成为只读的；提供 getter 方法和 setter 方法，使其成为可读／写的。更多信息，请查阅[《The Swift Programming Language 中文版》](http://wiki.jikexueyuan.com/project/swift/)中的 [属性（Properties）](http://wiki.jikexueyuan.com/project/swift/chapter2/10_Properties.html)。

### 拷贝语义（Copy Semantics）

在 Swift 中，Objective-C 的`copy`属性特性被转换为`@NSCopying`。这一类的属性必须遵守 `NSCopying`协议。更多信息，请查阅[《The Swift Programming Language 中文版》](http://wiki.jikexueyuan.com/project/swift/)中的 [特性（Attributes）](http://wiki.jikexueyuan.com/project/swift/chapter2/09_Classes_and_Structures.html)。

<a name="implementing_core_data_managed_object_subclasses"></a>
## 实现 Core Data Managed Object 子类（Implementing Core Data Managed Object Subclasses）

Core Data 提供了底层存储实现以及`NSManagedObject`子类的属性的实现，并且也提供了在一对多关系中添加和移除对象的实例方法的实现。你可以使用`@NSManaged`属性告知 Swift 编译器，一个声明的实现部分将在运行时由 Core Data 提供。

在你的`NSManagedObject`子类中，为每一个和 Core Data 模型文件中相对应的属性或者方法声明添加`@NSManaged`属性。例如，思考下面这个叫做“Person”的 Core Data 实体，它有个叫做“name”的 String 类型的属性，以及一个叫做“friends”的一对多关系。

![](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/Art/coredataeditor_2x.png)

相对应的`NSManagedObject`子类`Person`中的代码如下：

```swift
import CoreData
class Person: NSManagedObject {
	@NSManaged var name: String
    @NSManaged var friends: NSSet
        
	@NSManaged func addFriendsObject(friend: Person)
	@NSManaged func removeFriendsObject(friend: Person)
	@NSManaged func addFriends(friends: NSSet)
	@NSManaged func removeFriends(friends: NSSet)
}
```

`name`和`friends`属性都使用`@NSManaged`属性声明，以此表明 Core Data 会在运行时提供其存储和实现。因为“friends”是个一对多关系，因此 Core Data 还提供了一些相关的存取方法。

为了在 Swift 中配置一个`NSManagedObject`的子类供 Core Data 模型实体使用，在 Xcode 中打开模型实体检查器，在 Class 输入框中输入类名，并在 Module 输入框的下拉列表中选择“Current Product Module”。

![](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/Art/coredatanamespace_2x.png)

<a name="using_swift_class_names_with_objective_c_apis"></a>
## 兼容使用 Swift 类名的 Objective-C API（Using Swift Class Names with Objective-C APIs）

Swift 类的命名基于他们被编译的模块，即使是使用来自 Objective-C 的代码。和 Objective-C 不同的是，所有的类都是全局命名空间的一部分，必须没有相同的名字，Swift 类可以基于它们存在的模块来消除歧义。比如，MyFramework 框架中的 DataManager 类在 Swift 中的全限定名就是 MyFramework.DataManager。一个 Swift 应用的 target 就是一个模块，所以，在一个叫 MyGreatApp 的应用里，叫 Observer 的 Swift 类的全限定名是 MyGreatApp.Observer。

当一个 Swift 类在 Objective-C 代码中使用时，为了保持命名空间，Swift 类用他们的全限定名暴漏给 Objective-C runtime。因此，当你使用那些用到 Swift 类的类名字符串的 API，必须包含类的全限定名。比如，当你创建一个基于文档的 Mac 应用，要在应用的 Info.plist 里提供`NSDocument`子类的名字。在 Swift 中，你必须使用文档子类的全名，即包含你的应用名或者框架名。

下面的例子中，`NSClassFromString`函数用于从一个字符串表示的类名获取对该类的引用。为了检索 Swift 类，需要使用全限定名，包括应用的名字。
	
```	swift
let myPersonClass: AnyClass = NSClassFromString("MyGreatApp.Person")
```
