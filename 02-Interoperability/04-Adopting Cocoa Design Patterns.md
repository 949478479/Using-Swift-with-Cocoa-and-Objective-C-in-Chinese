# 采用 Cocoa 设计模式

- [代理](#delegation)
- [惰性初始化](#lazy_initialization)
- [错误处理](#error_handling)
    - [捕获和处理错误](#catching_and_handling_an_error)
    - [将错误转换为可选值](#converting_errors_to_optional_values)
    - [抛出错误](#throwing_an_error)
    - [处理异常](#handling_exceptions)
    - [捕获和处理自定义错误](#catching_and_handling_custom_errors)
- [键值观察](#key_value_observing)
- [撤销](#undo)
- [目标-动作](#target_action)
- [单例](#singleton)
- [内省](#introspection)
- [序列化](#serializing)
- [本地化](#localization)
- [自动释放池](#autorelease_pools)
- [API 可用性](#API_Availability)
- [处理命令行参数](#Processing_Command-Line_Arguments)

使用 Cocoa 既有的设计模式，能帮助开发者开发出设计巧妙、扩展性强的应用程序。这些模式很多都依赖于在 Objective-C 中定义的类。由于 Swift 与 Objective-C 的互用性，你依然可以在 Swift 中使用这些设计模式。在许多情况下，你甚至可以使用 Swift 的语言特性扩展或简化这些 Cocoa 设计模式，使这些设计模式更加强大易用。

<a name="delegation"></a>
## 代理

在 Swift 和 Objective-C，代理通常表现为一个定义交互方法的协议和符合协议的代理属性。就像在 Objective-C，在向代理发送可能无法响应的消息之前，应询问代理能否响应消息。在 Swift，可以使用可选链语法在一个可能为`nil`的对象上调用可选的代理方法，并使用`if-let`语法解包可能存在的返回值。下面的代码阐明了这个过程：

1. 检查`myDelegate`不为`nil`。
2. 检查`myDelegate`是否实现了`window:willUseFullScreenContentSize:`方法。
3. 如果步骤1和步骤2的检查顺利通过，那么调用该方法，将返回值赋值给名为`fullScreenSize`的常量。
4. 在控制台打印方法的返回值。

```swift
class MyDelegate: NSObject, NSWindowDelegate {
    func window(_ window: NSWindow, willUseFullScreenContentSize proposedSize: NSSize) -> NSSize {
        return proposedSize
    }
}
myWindow.delegate = MyDelegate()
if let fullScreenSize = myWindow.delegate?.window(myWindow, willUseFullScreenContentSize: mySize) {
    print(NSStringFromSize(fullScreenSize))
}
```

<a name="lazy_initialization"></a>
## 惰性初始化

惰性属性的值只会在被初次访问时才被初始化。如果属性的初始化过程十分复杂或者代价昂贵，或者属性的初始值无法在实例的构造过程完成前确定时，那么就可以使用惰性属性。

在 Objective-C，一个属性可能会覆写其自动合成的读取方法，只在实例变量为`nil`时才初始化实例变量：

```objective-c
@property NSXMLDocument *XML;

- (NSXMLDocument *)XML {
    if (_XML == nil) {
        _XML = [[NSXMLDocument alloc] initWithContentsOfURL:[[Bundle mainBundle] URLForResource:@"/path/to/resource" withExtension:@"xml"] options:0 error:nil];
    }

    return _XML;
}
```

在 Swift，可以使用`lazy`修饰符声明一个存储属性，这将使计算初始值的表达式只在属性被初次访问时才进行求值：

```swift
lazy var XML: XMLDocument = try! XMLDocument(contentsOf: Bundle.main.url(forResource: "document", withExtension: "xml")!, options: 0)
```

由于惰性属性只在被初次访问时才进行初始化，此时实例本身已被完全初始化，因此在初始化表达式中可以使用`self`：

```swift
var pattern: String
lazy var regex: NSRegularExpression = try! NSRegularExpression(pattern: self.pattern, options: [])
```

如果还需在初始化的基础上进行额外的设置，可以通过返回属性初始值的自求值闭包给属性赋值：

```swift
lazy var currencyFormatter: NumberFormatter = {
    let formatter = NumberFormatter()
    formatter.numberStyle = .currency
    formatter.currencySymbol = "¤"
    return formatter
}()
```

> 注意  
> 如果一个惰性属性还未被初始化就被多个线程同时访问，那么此时无法保证此惰性属性只被初始化一次。

请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的 [延迟存储属性](http://wiki.jikexueyuan.com/project/swift/chapter2/10_Properties.html#stored_properties) 小节。

<a name="error_handling"></a>
## 错误处理

在 Cocoa 中，会产生错误的方法将 `NSError` 指针参数作为最后一个参数，当产生错误时，该参数会被 `NSError` 对象填充。Swift 会自动将 Objective-C 中会产生错误的方法转换为根据 Swift 原生错误处理机制抛出错误的方法。

> 注意  
> 某些接受错误的方法，例如委托方法，或者接受一个带有 `NSError` 参数的块作为参数的方法，不会被 Swift 导入为抛出方法。

例如，请考虑如下来自于 `NSFileManager` 的 Objective-C 方法：

```objective-c
- (BOOL)removeItemAtURL:(NSURL *)URL
                  error:(NSError **)error;
```

在 Swift，它会被这样导入：

```swift
func removeItem(at: URL) throws
```

注意 `removeItem(at:)` 方法被 Swift 导入时，返回值类型为 `Void`，没有 `error` 参数，并且还有一个 `throws` 声明。

如果 Objective-C 方法的最后一个非块类型的参数是 `NSError **` 类型，Swift 会将之替换为 `throws` 关键字，以表明该方法可以抛出一个错误。如果 Objective-C 方法的错误参数也是它的第一个参数，Swift 会尝试删除选择器的第一部分中的 “WithError” 或 “AndReturnError” 后缀（如果存在）来进一步简化方法名。如果简化后的方法名会和其他方法名冲突，则不会对方法名进行简化。

如果产生错误的 Objective-C 方法返回一个用来表示方法调用成功或失败的 `BOOL` 值，Swift 会把返回值转换为 `Void`。同样，如果产生错误的 Objective-C 方法返回一个`nil` 值来表明方法调用失败，Swift 会把返回值转换为非可选类型。

否则，如果不能推断任何约定，则该方法保持不变。

> 注意  
> 在一个产生错误的 Objective-C 方法声明上使用 `NS_SWIFT_NOTHROW` 宏可以防止该方法被 Swift 作为抛出方法导入。

<a name="catching_and_handling_an_error"></a>
### 捕获和处理错误

在 Objective-C，错误处理是可选的，这意味着除非提供了一个错误指针，否则方法产生的错误会被忽略。在 Swift，调用一个会抛出错误的方法时必须明确进行错误处理。

以下示例演示了在 Objective-C 中调用方法时如何处理错误：

```objective-c
NSFileManager *fileManager = [NSFileManager defaultManager];
NSURL *fromURL = [NSURL fileURLWithPath:@"/path/to/old"];
NSURL *toURL = [NSURL fileURLWithPath:@"/path/to/new"];
NSError *error = nil;
BOOL success = [fileManager moveItemAtURL:fromURL toURL:toURL error:&error];
if (!success) {
    NSLog(@"Error: %@", error.domain);
}
```

Swift 中等效的代码如下所示：

```swift
let fileManager = FileManager.default
let fromURL = URL(fileURLWithPath: "/path/to/old")
let toURL = URL(fileURLWithPath: "/path/to/new")
do {
    try fileManager.moveItem(at: fromURL, to: toURL)
} catch let error as NSError {
    print("Error: \(error.domain)")
}
```

此外，你可以使用 `catch` 子句来匹配特定的错误代码以便区分可能的失败情况：

```swift
do {
    try fileManager.moveItem(at: fromURL, to: toURL)
} catch CocoaError.fileNoSuchFile {
    print("Error: no such file exists")
} catch CocoaError.fileReadUnsupportedScheme {
    print("Error: unsupported scheme (should be 'file://')")
}
```

<a name="converting_errors_to_optional_values"></a>
### 将错误转换为可选值

在 Objective-C 中，当你只关心是否有错误，而不是发生什么特定错误时，你可以传递 `NULL` 作为错误参数。在 Swift，你可以使用 `try?` 关键字将抛出表达式转换为返回可选值的表达式，然后检查返回值是否为 `nil`。

例如，`NSFileManager` 的实例方法 `URLForDirectory(_:inDomain:appropriateForURL:create:)` 会返回指定搜索路径和域中的 URL，或者如果适当的 URL 不存在且不能创建，则会产生错误。在 Objective-C 中，此方法成功或是失败可以通过是否返回 URL 对象来判断。

```objective-c
NSFileManager *fileManager = [NSFileManager defaultManager];
     
NSURL *tmpURL = [fileManager URLForDirectory:NSCachesDirectory 
                                    inDomain:NSUserDomainMask 
                           appropriateForURL:nil 
                                      create:YES 
                                       error:NULL];
if (tmpURL != nil) {
    // ...
}
```

在 Swfit 中你可以像下面这样做：

```swift
let fileManager = FileManager.default
if let tmpURL = try? fileManager.url(for: .cachesDirectory, in: .userDomainMask, appropriateFor: nil, create: true) {
    // ...
}
```

<a name="throwing_an_error"></a>
### 抛出错误

如果在一个 Objective-C 方法中发生错误，则使用错误对象来填充该方法的错误指针参数：

```objective-c
// 发生一个错误
if (errorPtr) {
   *errorPtr = [NSError errorWithDomain:NSURLErrorDomain
                                   code:NSURLErrorCannotOpenFile
                               userInfo:nil];
}
```

如果在一个 Swift 方法中发生错误，则错误会被抛出，并且自动传播给调用者：

```swift
// 发生一个错误
throw NSError(domain: NSURLErrorDomain, code: NSURLErrorCannotOpenFile, userInfo: nil)
```

如果 Objective-C 代码调用会抛出错误的 Swift 方法，则该错误会自动填充到桥接的 Objective-C 方法的错误指针参数。

例如，考虑 `NSDocument` 中的 `read(from:ofType:)` 方法。在 Objective-C 中，此方法的最后一个参数是 `NSError **` 类型。在 Swift 的 `NSDocument` 子类中重写此方法时，该方法会以抛出错误的方式替代错误指针参数。

```swift
class SerializedDocument: NSDocument {
    static let ErrorDomain = "com.example.error.serialized-document"

    var representedObject: [String: Any] = [:]

    override func read(from fileWrapper: FileWrapper, ofType typeName: String) throws {
        guard let data = fileWrapper.regularFileContents else {
            throw NSError(domain: NSURLErrorDomain, code: NSURLErrorCannotOpenFile, userInfo: nil)
        }

        if case let JSON as [String: Any] = try JSONSerialization.jsonObject(with: data, options: []) {
            self.representedObject = JSON
        } else {
            throw NSError(domain: SerializedDocument.ErrorDomain, code: -1, userInfo: nil)
        }
    }
}
```

如果该方法无法使用文档的常规文件内容来创建对象，就会抛出一个 `NSError` 对象。如果该方法在 Swift 中调用，则错误会传播到它的调用域。如果该方法在 Objective-C 中调用，则错误会填充错误指针参数。

<a name="handling_exceptions"></a>
### 处理异常

在 Objective-C 中，异常与错误不同。Objective-C 异常处理使用 `@try`，`@catch` 和 `@throw` 语法来标明不可恢复的程序错误。这与司空见惯的 Cocoa 错误模式截然不同，后者使用一个尾随的 `NSError` 参数来标明你在开发过程中设计的可恢复错误。

在 Swift 中，你可以从使用 Cocoa 错误模式传递的错误中恢复，如前文[错误处理](#error_handling)中所述。然而，没有可靠的方法可以从 Swift 中的 Objective-C 异常中恢复。要处理 Objective-C 异常，则需编写 Objective-C 代码，以便在异常到达任何 Swift 代码之前捕获异常。

关于 Objective-C 异常的更多信息，请参阅 [*Exception Programming Topics*](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Exceptions/Exceptions.html#//apple_ref/doc/uid/10000012i)。

<a name="catching_and_handling_custom_errors"></a>
### 捕获和处理自定义错误

Objective-C 框架可以使用自定义错误域和枚举来组织相关的错误类别。

下面的例子展示了使用 Objective-C 中的 `NS_ERROR_ENUM` 宏定义的自定义错误类型：

```objective-c
extern NSErrorDomain const MyErrorDomain;
typedef NS_ERROR_ENUM(MyErrorDomain, MyError) {
    specificError1 = 0,
    specificError2 = 1
};
```

如下示例展示了如何在 Swift 中使用该自定义错误类型生成错误：

```swift
func customThrow() throws {
    throw NSError(
        domain: MyErrorDomain,
        code: MyError.specificError2.rawValue,
        userInfo: [
            NSLocalizedDescriptionKey: "A customized error from MyErrorDomain."
        ]
    )
}

do {
    try customThrow()
} catch MyError.specificError1 {
    print("Caught specific error #1")
} catch let error as MyError where error.code == .specificError2 {
    print("Caught specific error #2, ", error.localizedDescription)
    // Prints "Caught specific error #2. A customized error from MyErrorDomain."
} let error {
    fatalError("Some other error: \(error)")
}
```

<a name="key_value_observing"></a>
## 键值观察

键值观察是一种能让某个对象在其他对象指定属性变化时得到通知的机制。只要 Swift 类继承自 `NSObject` 类，就可以在 Swift 中通过如下两步实现键值观察。

1. 为想要观察的属性添加 `dynamic` 修改符和 `@objc` 属性。关于 `dynamic` 修饰符的更多信息，请参阅[强制动态派发](01-Interacting%20with%20Objective-C%20APIs.md#%E5%BC%BA%E5%88%B6%E5%8A%A8%E6%80%81%E6%B4%BE%E5%8F%91)小节。

	```swift
	class MyObjectToObserve: NSObject {
	    @objc dynamic var myDate = NSDate()
	    func updateDate() {
	        myDate = NSDate()
	    }
	}
	```

2. 为键路径创建一个对应的观察者并调用 `observe(_:options:changeHandler)` 方法。关于键路径的更多信息，请参阅[键和键路径](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C-in-Chinese/blob/master/02-Interoperability/01-Interacting%20with%20Objective-C%20APIs.md#%E9%94%AE%E5%92%8C%E9%94%AE%E8%B7%AF%E5%BE%84)小节。

	```swift
	class MyObjectToObserve: NSObject {
	    @objc dynamic var myDate = NSDate()
	    func updateDate() {
            myDate = NSDate()
	    }
    }

    class MyObserver: NSObject {
	    @objc var objectToObserve: MyObjectToObserve
	    var observation: NSKeyValueObservation?

        init(object: MyObjectToObserve) {
            objectToObserve = object
            super.init()

            observation = observe(\.objectToObserve.myDate) { object, change in
                print("Observed a change to \(object.objectToObserve).myDate, updated to: \(object.objectToObserve.myDate)")
            }
        }
    }

    let observed = MyObjectToObserve()
    let observer = MyObserver(object: observed)

    observed.updateDate()
	```

<a name="undo"></a>
## 撤销

在 Cocoa 中，可以使用`NSUndoManager`注册一个操作来允许用户撤销该操作。在 Swift，可以像在 Objective-C 一样使用 Cocoa 的撤销功能。

对于应用程序响应链上的对象，也就是`macOS`的`NSResponder`和`iOS`的`UIResponder`，包括它们的子类，都有一个只读的`undoManager`属性。该属性返回一个可选类型的`NSUndoManager`对象，该对象管理着应用程序的撤销栈。每当用户进行一个操作时，例如编辑文本内容，或是删除表视图的选中行，可以用`NSUndoManager`注册一个撤销操作，从而允许用户恢复到操作之前的状态。一个撤销操作应该记录必要的步骤来抵消之前的操作，例如将文本内容设置为修改前的原始内容，或是重新添加被删除的表视图单元格。

`NSUndoManager`支持两种方式注册撤销操作：一种是“简单撤销”，它会执行一个选择器，最多只能接收一个对象类型的参数；另一种则使用`NSInvocation`对象，从而可以支持多个参数以及多种参数类型。

例如，思考下面的`Task`模型，它使用`ToDoListController`来展示一个待办任务列表：

```swift
class Task {
    var text: String
    var completed: Bool = false

    init(text: String) {
        self.text = text
    }
}

class ToDoListController: NSViewController, NSTableViewDataSource, NSTableViewDelegate {
    @IBOutlet var tableView: NSTableView!
    var tasks: [Task] = []

    // ...
}
```

对于 Swift 中的属性，可以在属性观察器`willSet`中创建一个撤销操作，使用`self`作为`target`，相应的 Objective-C 写入方法作为`selector`，属性的当前值作为`object`：

```swift
@IBOutlet var notesLabel: NSTextView!
     
var notes: String? {
    willSet {
        undoManager?.registerUndoWithTarget(self, selector: "setNotes:", object: self.title)
        undoManager?.setActionName(NSLocalizedString("todo.notes.update", comment: "Update Notes"))
    }
     
    didSet {
        notesLabel.string = notes
    }
}
```

如果方法接受多个参数，可以使用`NSInvocation`创建撤销操作：

```swift
@IBOutlet var remainingLabel: NSTextView!

func mark(task: Task, asCompleted completed: Bool) {
    if let target = undoManager?.prepare(withInvocationTarget: self) as? ToDoListController {
        target.mark(task: task, asCompleted: !completed)
        undoManager?.setActionName(NSLocalizedString("todo.task.mark", comment: "Mark As Completed"))
    }

    task.completed = completed
    tableView.reloadData()

    let numberRemaining = tasks.filter{ $0.completed }.count
    remainingLabel.string = String(format: NSLocalizedString("todo.task.remaining", comment: "Tasks Remaining: %d"), numberRemaining)
}
```

`prepare(withInvocationTarget:)`方法会返回`target`的代理对象。通过转换为`ToDoListController`，就可以利用该代理对象直接调用`mark(task:asCompleted:)`。

更多信息请参阅 [Undo Architecture](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/UndoArchitecture/UndoArchitecture.html#//apple_ref/doc/uid/10000010)。

<a name="target_action"></a>
## 目标-动作

目标-动作是一种常见的 Cocoa 设计模式，可以在特定事件发生时，让某个对象向另一个对象发送消息。Swift 和 Objective-C 的目标-动作模式基本类似。在 Swift，可以使用`Selector`类型引用 Objective-C 的选择器。请参阅 [选择器](01-Interacting%20with%20Objective-C%20APIs.md#%E9%80%89%E6%8B%A9%E5%99%A8) 小节查看在 Swift 中使用目标-动作模式的示例。

<a name="singleton"></a>
## 单例

单例模式提供了一个可全局访问的共享对象。可以自己创建在应用程序内共享的单例对象，从而提供一个统一的资源或服务的访问入口，比如一个播放音效的音频通道或发起 HTTP 请求的网络管理者。

在 Objective-C，可以用`dispatch_once`函数包裹初始化代码，从而保证在应用程序的生命周期内，块内的代码只会执行一次，这样就确保了只有唯一的实例会被创建：

```objective-c
+ (instancetype)sharedInstance {
    static id _sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _sharedInstance = [[self alloc] init];
    });
    
    return _sharedInstance;
}
```

在 Swift，可以简单地使用静态类型属性来实现单例，它能保证延迟初始化只执行一次，即使在多个线程同时访问的情况下：

```swift
class Singleton {
    static let sharedInstance = Singleton()
}
```

如果在初始化的基础上还需要进行额外的设置，那么可以通过调用闭包返回初始值的方式来初始化：

```swift
class Singleton {
    static let sharedInstance: Singleton = {
        let instance = Singleton()
        // 设置代码
        return instance
    }()
}
```

请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的 [类型属性](http://wiki.jikexueyuan.com/project/swift/chapter2/10_Properties.html#type_properties) 小节获取更多信息。

<a name="introspection"></a>
## 内省

在 Objective-C，使用`isKindOfClass:`方法检查某个对象是否是特定类型的实例，使用`conformsToProtocol:`方法检查某个对象是否符合特定协议。在 Swift，可以使用`is`运算符来检查类型，可以使用`as?`运算符向下转换到指定类型。

可以使用`is`运算符检查一个实例是否是指定类或其子类的实例。若是，那么`is`返回结果为`true`，反之为`false`。

```swift
if object is UIButton {
    // object 是 UIButton 类型
} else {
    // object 不是 UIButton 类型
}
```

也可以使用`as?`运算符尝试向下转换到子类型。`as?`运算符返回可选类型的值，可结合`if-let`语句使用。

```swift
if let button = object as? UIButton {
    // object 成功转换为 UIButton 并绑定到 button
} else {
    // object 无法转换为 UIButton
}
```
请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的 [类型转换](http://wiki.jikexueyuan.com/project/swift/chapter2/19_Type_Casting.html) 章节获取更多信息。

检查协议符合性以及转换到符合协议类型的语法和上述类型检查和转换的语法是完全一样的。如下是使用`as?`检查协议符合性的示例：

```swift
if let dataSource = object as? UITableViewDataSource {
    // object 符合 UITableViewDataSource 协议并绑定到 dataSource
} else {
    // object 不符合 UITableViewDataSource 协议
}
```

注意，经过转换之后，`dataSource`常量的类型为`UITableViewDataSource`，所以只能访问和调用`UITableViewDataSource`协议定义的属性和方法。想进行其他操作时，必须将其转换为其他类型。

请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的 [协议](http://wiki.jikexueyuan.com/project/swift/chapter2/22_Protocols.html) 章节获取更多信息。

<a name="serializing"></a>
## 序列化

通过序列化，可以在应用程序中编码和解码对象，将其转换为独立于体系结构的数据形式，例如 JSON 或属性列表，并能从这类数据形式转换回对象。这类数据形式可以写入文件，传给其他本地进程，以及通过网络进行传递。

在 Objective-C 中，可以使用 Foundation 框架的 `NSJSONSerialiation` 类或 `NSPropertyListSerialization` 类从 JSON 或者属性列表来实例化对象，通常这种对象会是 `NSDictionary<NSString *, id>` 类型。

在 Swift 中，标准库定义了一套标准化方法来编码和解码数据。若要使用这套方法，你可以让自定义类型遵守 `Encodable` 或 `Decodable` 协议，若要同时遵守这两种协议，更便捷的方式是直接遵守 `Codable` 协议。你可以通过 Foundation 库中的 `JSONEncoder` 和 `PropertyListEncoder` 类将某个实例转化为 JSON 或属性列表数据。与此相似，你可以用 `JSONDecoder` 和 `PropertyListDecoder` 类从 JSON 或属性列表数据解码并初始化实例。

例如，一个应用从 web 服务器接收到一些表示食品杂货店商品的 JSON 数据，如下所示：

```swift
{
    "name": "Banana",
    "points": 200,
    "description": "A banana grown in Ecuador.",
    "varieties": [
    	"yellow",
    	"green",
    	"brown"
    ]
}
```

如下代码演示了如何编写一个表示食品杂货店商品的 Swift 类型，该类型可以使用任何提供了编码器和解码器的序列化格式：

```swift
struct GroceryProduct: Codable {
    let name: String
    let points: Int
    let description: String
    let varieties: [String]
}
```

你可以从 JSON 数据形式创建 `GroceryProduct` 实例，只需创建一个 `JSONDecoder` 实例，然后传入 `GroceryProduct.self` 类型以及相应的 JSON 数据：

```swift
let json = """
    {
         "name": "Banana",
         "points": 200,
         "description": "A banana grown in Ecuador.",
         "varieties": [
             "yellow",
             "green",
             "brown"
          ]
    }
""".data(using: .utf8)!

let decoder = JSONDecoder()
let banana = try decoder.decode(GroceryProduct.self, from: json)

print("\(banana.name) (\(banana.points) points): \(banana.description)")
// Prints "Banana (200 points): A banana grown in Ecuador."
```

关于如何编码和解码更复杂的自定义类型的相关信息，请参阅 [Encoding and Decoding Custom Types](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types)。关于如何编码和解码 JSON 的更多信息，请参阅 [Using JSON with Custom Types](https://developer.apple.com/documentation/foundation/archives_and_serialization/using_json_with_custom_types)。

<a name="localization"></a>
## 本地化

在 Objective-C，通常用`NSLocalizedString`系列宏来本地化字符串。这套宏包括`NSLocalizedString`，`NSLocalizedStringFromTable`，`NSLocalizedStringFromTableInBundle`，以及`NSLocalizedStringWithDefaultValue`。而在 Swift，`NSLocalizedString(_:tableName:bundle:value:comment:)`函数就可以实现`NSLocalizedString`系列宏的这些功能。

Swift 并没有为每个本地化宏单独定义函数，而是为`NSLocalizedString(_:tableName:bundle:value:comment:)`函数的`tableName`，`bundle`，`value`参数提供了默认值，以便可以在需要时重写它们。

例如，本地化字符串最常见的形式可能仅仅需要一个本地化键和一句注释：

```swift
let format = NSLocalizedString("Hello, %@!", comment: "Hello, {given name}!")
let name = "Mei"
let greeting = String(format: format, arguments: [name as CVarArg])
print(greeting)
// 打印 "Hello, Mei!"
```

或者，为了使用单独的包来提供本地化资源，应用可能需要使用更复杂的本地化形式：

```swift
if let path = Bundle.main.path(forResource: "Localization", ofType: "strings", inDirectory: nil, forLocalization: "ja"),
    let bundle = Bundle(path: path) {
    let translation = NSLocalizedString("Hello", bundle: bundle, comment: "")
    print(translation)
}
// 打印 "こんにちは"
```

更多信息请参阅 [*Internationalization and Localization Guide*](https://developer.apple.com/library/content/documentation/MacOSX/Conceptual/BPInternational/Introduction/Introduction.html#//apple_ref/doc/uid/10000171i)。

<a name="autorelease_pools"></a>
## 自动释放池

自动释放池块可以让对象放弃所有权而又不会被立即释放。通常，你不需要创建自己的自动释放池块，但是有的情况下则必须创建，例如生成次级线程时，还有一些时候则最好创建，例如通过循环创建大量临时的自动释放对象时。

在 Objective-C，自动释放池块使用`@autoreleasepool`标记。在 Swift，你可以使用`autoreleasepool(_:)`函数在自动释放池块中执行一个闭包。

```swift
import Foundation

autoreleasepool {
    // 创建自动释放对象的相关代码
}
```

更多信息请参阅 [*Advanced Memory Management Programming Guide.*](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011)。

<a name="API_Availability"></a>
## API 可用性

一些 API 并非在所有平台的所有版本中都可用。为了确保应用程序能够适应任何功能上的差异，就需要检查这些 API 的可用性。

在 Objective-C，使用`respondsToSelector:`和`instancesRespondToSelector:`方法检查一个类或者实例的方法是否可用。否则，调用方法可能会抛出`NSInvalidArgumentException`类型的`unrecognized selector sent to instance`异常。例如，`CLLocationManager`实例的`requestWhenInUseAuthorization`方法从`iOS 8.0`和`macOS10.10`开始才可用：

```objective-c
if ([CLLocationManager instancesRespondToSelector:@selector(requestWhenInUseAuthorization)]) {
    // 方法可用
} else {
    // 方法不可用
}
```

在 Swift，试图调用项目所支持的最低平台版本不支持的方法将会引发编译时错误。

如上例子在 Swift 中如下所示：

```swift
let locationManager = CLLocationManager()
locationManager.requestWhenInUseAuthorization()
// error: only available on iOS 8.0 or newer
```

如果应用程序在版本低于`iOS 8.0`或者`macOS 10.10`的平台上运行，那么`requestWhenInUseAuthorization()`方法将不可用，因此编译器会报告错误。

Swift 代码可以使用 API 可用性在运行时作为判断条件。可用性检查可以作为控制流语句的一个条件，例如`if`，`guard`，`while`语句。

针对之前的例子，可以在`if`语句中检查可用性，当`requestWhenInUseAuthorization()`方法在运行时可用时才去调用：

```swift
let locationManager = CLLocationManager()
if #available(iOS 8.0, macOS 10.10, *) {
    locationManager.requestWhenInUseAuthorization()
}
```

或者，可以在`guard`语句中检查可用性，除非当前的平台版本符合指定要求，否则将退出作用域。这种方式简化了处理不同平台功能时的逻辑。

```swift
let locationManager = CLLocationManager()
guard #available(iOS 8.0, macOS 10.10, *) else { return }
locationManager.requestWhenInUseAuthorization()
```

每个平台参数由下面列出的平台名称组成，后面跟着相应的版本号。最后一个参数是一个星号（`*`），用来处理未来可能的平台。

平台名称：

- iOS
- iOSApplicationExtension
- macOS
- macOSApplicationExtension
- watchOS
- watchOSApplicationExtension
- tvOS
- tvOSApplicationExtension

所有的 Cocoa API 都提供了可用性信息，因此能确信代码可以在应用所支持的任何平台上如期工作。

可以用`@available`特性标注自己的 API 声明来指明其可用性。`@available`特性的语法和`#available`一样，以逗号分隔的参数提供平台版本要求。

例如：

```swift
@available(iOS 8.0, macOS 10.10, *)
func useShinyNewFeature() {
    // ...
}
```

> 注意  
> 使用`@available`特性标记的方法可以安全地使用满足指定平台要求的 API 而不用再进行可用性检查。

<a name="Processing_Command-Line_Arguments"></a>
## 处理命令行参数

在`macOS`，通常通过点击 Dock 或者 Launchpad 上的应用程序图标启动应用程序，也可以双击 Finder 中的应用程序图标。然而，也可以使用终端通过编程的方式打开应用程序，并可以为其传递一些命令行参数。

可以访问类型属性`CommandLine.arguments`获取应用程序启动时指定的一系列命令行参数。

```
$ /path/to/app --argumentName value
```

```swift
for argument in CommandLine.arguments {
    print(argument)
}
// 打印 /path/to/app
// 打印 --argumentName
// 打印 value
```

`CommandLine.arguments`的第一个元素总是可执行文件的路径。从`Process.arguments[1]`开始才是启动时指定的命令行参数。

> 注意  
> 这等同于访问`ProcessInfo.processInfo`的`arguments`属性。
