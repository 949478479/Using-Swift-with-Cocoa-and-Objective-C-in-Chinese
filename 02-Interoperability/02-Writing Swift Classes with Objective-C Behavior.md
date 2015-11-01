# 使用 Objective-C 特性编写 Swift 类

本页包含内容：

- [继承 Objective-C 类](#inheriting_from_objective-c_classes)
- [采纳协议](#adopting_protocols)
- [编写构造器和析构器](#writing_initializers_and_deinitializers)
- [兼容使用 Swift 类名的 Objective-C API](#using_swift_class_names_with_objective_c_apis)
- [与 Interface Builder 结合](#integrating_with_interface_builder)
- [指定属性特性](#specifying_property_attributes)
- [子类化 NSManagedObject](#implementing_core_data_managed_object_subclasses)

*互用性* 让你可以编写融合了 Objective-C 语言特性的 Swift 类。在编写 Swift 类时，不仅可以继承 Objective-C 类，采纳 Objective-C 协议，还可以使用 Objective-C 的一些其它功能。这意味着可以基于 Objective-C 中耳熟能详的既有特性来编写 Swift 类，还可以充分利用 Swift 提供的更为强大的现代化语言特性。

<a name="inheriting_from_objective-c_classes"></a>
## 继承 Objective-C 类

在 Swift，可以定义一个继承自 Objective-C 类的 Swift 子类。在 Swift 的类名后面加上一个冒号（:），冒号后面跟上 Objective-C 类的类名即可。

```swift
import UIKit
class MySwiftViewController: UIViewController {
	// 定义类
}
```

可以从 Objective-C 父类中继承所有的功能。如果要覆盖父类中的实现，不要忘记使用`override`关键字。

### NSCoding 协议

`NSCoding`协议要求采纳协议的类型实现其所要求的构造器`init(coder:)`。直接采纳`NSCoding`协议的类必须实现这个方法。对于采纳`NSCoding`协议的类的子类，如果有一个或者多个自定义的构造器或者有不具有默认初始值的属性，也必须实现这个方法。Xcode 提供了下面这个 fix-it 来提供一个占位实现：

```swift
required init(coder aDecoder: NSCoder) {
	fatalError("init(coder:) has not been implemented")
}
```

对于那些从 Storyboards 里加载的对象，或者用`NSUserDefaults`或`NSKeyedArchiver`类归档到硬盘的对象，则必须提供该构造器的完整实现。当然，当一个类型不会以此种方式实例化时，就不需要提供该构造器的完整实现。

<a name="adopting_protocols"></a>
## 采纳协议

Objective-C 协议会被导入为 Swift 协议。所有协议都写在一个用逗号分隔的列表中，跟在父类类名后面（如果该类有父类的话）。

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
> 在 Swift，类和协议的命名空间是统一的，因此 Objective-C 的`NSObject`协议会被重映射为 Swift 的`NSObjectProtocol`。

<a name="writing_initializers_and_deinitializers"></a>
## 编写构造器和析构器

Swift 编译器能确保构造器不会遗留任何未初始化的属性，从而增加代码的安全性和可预测性。另外，与 Objective-C 不同，Swift 不提供单独的内存分配方法。你会始终使用原生的 Swift 构造器，即使是和 Objective-C 类协作，Swift 也会将 Objective-C 构造器转换为 Swift 构造器。请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的[构造器](http://wiki.jikexueyuan.com/project/swift/chapter2/14_Initialization.html)章节了解关于如何实现构造器的更多信息。

如果希望在对象释放前进行额外的清理工作，可以实现一个析构器来代替`dealloc`方法。在对象被释放前，Swift 会自动调用析构器。当 Swift 调用完子类的析构器后，会自动调用父类的析构器。使用 Objective-C 类或者继承自 Objective-C 类的 Swift 类时，Swift 也会自动调用该类父类中的`dealloc`方法。请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的[析构器](http://wiki.jikexueyuan.com/project/swift/chapter2/15_Deinitialization.html)章节了解关于如何实现析构器的更多信息。

<a name="using_swift_class_names_with_objective_c_apis"></a>
## 兼容使用 Swift 类名的 Objective-C API

Swift 类的命名基于其所在模块，即使是使用来自 Objective-C 的代码。和 Objective-C 不同，所有的类都是全局命名空间的一部分，名字不能重复。Swift 类可以基于其所在模块来消除歧义。例如，`MyFramework`框架中的`DataManager`类在 Swift 中的全限定名是`MyFramework.DataManager`。一个 Swift 应用的 target 就是一个模块，因此，在一个叫`MyGreatApp`的应用里，一个叫做`Observer`的 Swift 类的全限定名是`MyGreatApp.Observer`。

Swift 类在 Objective-C 代码中使用时，为了保持命名空间，Swift 类会将其全限定名暴露给 Objective-C 运行时。因此，使用那些用到 Swift 类名字符串的 API 时，必须使用类的全限定名。例如，你创建了一个基于文档的 Mac 应用，需要在应用的`Info.plist`里提供`NSDocument`子类的类名。在 Swift 中，必须使用`NSDocument`子类的全限定名，即应用名或者框架名加上子类名。

下面的例子中，`NSClassFromString`函数用于从一个字符串表示的类名获取该类的类对象。为了检索 Swift 类，需要使用全限定名，即需要加上应用的名字。
	
```	swift
let myPersonClass: AnyClass？= NSClassFromString("MyGreatApp.Person")
```

<a name="integrating_with_interface_builder"></a>
## 与 Interface Builder 结合

Swift 编译器包含一些特性，能让 Swift 类使用 Interface Builder 的一些特色功能。和 Objective-C 一样，在 Swift 也可使用 outlets，actions 和实时渲染。

### 使用 Outlets 和 Actions

使用 Outlets 和 Actions 可以连接源代码和 Interface Builder 中的 UI 对象，只需在属性或方法声明前标记`@IBOutlet`或`@IBAction`特性。声明一个 outlet 集合同样是用`@IBOutlet`特性，只不过会为该类型指定一个数组。

在 Swift 中声明一个 outlet 时，应该将类型声明为隐式解包可选类型。通过这种方式，Swift 编译器会自动为它分配空值`nil`，就不需要在构造器中为其分配初始值了。在运行时，构造过程完成后，storyboard 会连接 outlets。如果从 storyboard 或者`xib`文件实例化对象，则可以假定 outlet 已经连接完毕。

例如，下面的 Swift 代码声明了一个拥有 outlet、outlet 集合和 action 的类：

```swift
class MyViewController: UIViewController {
    @IBOutlet weak var button: UIButton!
    @IBOutlet var textFields: [UITextField]! // outlet 集合必须用强引用
    @IBAction func buttonTapped(_: AnyObject) {
        print("button tapped!")
    }
}
```

在`buttonTapped`方法中，sender 参数没有用到，因此可以省略该参数名。

### 实时渲染

可以使用`@IBDesignable`和`@IBInspectable`特性开启实时渲染，在 Interface Builder 中对自定义视图进行交互式设计。继承`UIView`或者`NSView`来自定义一个视图时，可以在类声明前标记`@IBDesignable`特性。在 Interface Builder 里添加自定义的视图后（在 Identity Inspector 面板的 Class 输入框中进行设置），Interface Builder 将在画布上实时渲染自定义视图。

还可以将`@IBInspectable`特性添加到类型兼容“用户定义运行时属性”（可以在 Identity Inspector 面板的 User Defined Runtime Attributes 中查看）的属性前。将自定义的视图添加到 Interface Builder 后，就可以在 Attributes Inspector 面板中编辑这些属性。

```swift
@IBDesignable
class MyCustomView: UIView {
	@IBInspectable var textColor: UIColor
	@IBInspectable var iconHeight: CGFloat
	// ...
}
```

![](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02-Interoperability/Attributes%20Inspector%402x.png)
![](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02-Interoperability/Identity%20Inspector%402x.png)

<a name="specifying_property_attributes"></a>
## 指定属性特性

在 Objective-C，属性通常会有一系列用于指定该属性的一些附加信息的属性特性。在 Swift，将通过不同的方式指明这些属性特性。

### 强引用和弱引用

在 Swift，属性默认都是强引用，可以使用`weak`关键字指明某个属性持有其所指向对象的弱引用。该关键字仅能修饰可选的 class 类型。请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的[类和结构](http://wiki.jikexueyuan.com/project/swift/chapter2/09_Classes_and_Structures.html)章节了解更多信息。

### 可读写和只读

在 Swift，没有`readwrite`和`readonly`特性。声明一个存储型属性时，使用`let`使其只读；使用`var`使其可读写。声明一个计算型属性时，为其提供一个 getter 方法使其只读；同时提供 getter 方法和 setter 方法使其可读写。请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的[属性](http://wiki.jikexueyuan.com/project/swift/chapter2/10_Properties.html)章节了解更多信息。

### 拷贝语义

在 Swift，Objective-C 的`copy`属性特性被转化为`@NSCopying`。这种类型的属性必须符合`NSCopying`协议。请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的[类和结构](http://wiki.jikexueyuan.com/project/swift/chapter2/09_Classes_and_Structures.html)章节了解更多信息。

<a name="implementing_core_data_managed_object_subclasses"></a>
## 子类化 NSManagedObject

Core Data 提供了底层存储以及`NSManagedObject`子类的属性实现，还提供了在对多关系中添加和移除对象的实例方法实现。可以使用`@NSManaged`特性告知 Swift 编译器，一个声明的实现部分将在运行时由 Core Data 提供。

在`NSManagedObject`子类中，为每一个和 Core Data 实体模型文件中相对应的属性或者方法声明标记`@NSManaged`特性。例如，思考下面这个叫做“Person”的 Core Data 实体，它有个叫做“name”的 String 类型的属性，以及一个叫做“friends”的对多关系。

![](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02-Interoperability/coredataeditor_2x.png)

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

`name`和`friends`属性都标记了`@NSManaged`特性，以此表明 Core Data 会在运行时提供其实现和存储。由于“friends”是个对多关系，因此 Core Data 还提供了一些相关的存取方法。

为了在 Swift 中配置一个`NSManagedObject`的子类供 Core Data 模型实体使用，在 Xcode 中打开 Data Model Inspector，在 Class 输入框中输入类名，并在 Module 输入框的下拉列表中选择“Current Product Module”。

![](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02-Interoperability/coredatanamespace_2x.png)
