# 采用 Cocoa 设计模式

- [代理](#delegation)
- [惰性初始化](#lazy_initialization)
- [错误处理](#error_handling)
    - [捕获和处理错误](#catching_and_handling_an_error)
    - [转换错误为可选值](#converting_errors_to_optional_values)
    - [抛出错误](#throwing_an_error)
- [键值观察](#key_value_observing)
- [撤销](#undo)
- [目标-动作](#target_action)
- [单例](#singleton)
- [内省](#introspection)
- [序列化](#serializing)
    - [验证序列化数据](#validating_serialized_representations)
- [本地化](#localization)
- [自动释放池](#autorelease_pools)
- [API 可用性](#API_Availability)
- [处理命令行参数](#Processing Command-Line Arguments)

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

在 Cocoa 中，产生错误的方法将`NSError`指针参数作为最后一个参数，当错误产生时，该参数会被`NSError`对象填充。Swift 会自动将 Objective-C 中产生错误的方法转换为根据 Swift 原生错误处理机制抛出错误的方法。

> 注意  
> 某些产生错误的方法，例如代理方法，或者接受一个带有`NSError`参数的块作为参数的方法，不会被 Swift 导入为`throws`方法。

例如，思考如下来自于`NSFileManager`的 Objective-C 方法：

```objective-c
- (BOOL)removeItemAtURL:(NSURL *)URL
                  error:(NSError **)error;
```

在 Swift，它会被这样导入：

```swift
func removeItem(at: URL) throws
```

注意`removeItem(at:)`方法被 Swift 导入时，返回值类型为`Void`，没有`error`参数，并且还有一个`throws`声明。

如果 Objective-C 方法的最后一个非块类型的参数是`NSError **`类型，Swift 会将之替换为`throws`关键字，以表明该方法可以抛出一个错误。如果 Objective-C 方法的错误参数是它的第一个参数，Swift 会尝试删除选择器第一部分中的`WithError`或`AndReturnError`后缀来进一步简化方法名。如果简化后的方法名会和其他方法名冲突，则不会对方法名进行简化。

如果产生错误的 Objective-C 方法返回一个用来表示方法调用成功或失败的`BOOL`值，Swift 会把返回值转换为`Void`。同样的，如果产生错误的 Objective-C 方法返回一个`nil`值来表明方法调用失败，Swift 会把返回值转换为非可选类型。

否则，如果没有约定可被推断，则该方法保持不变。

> 注意  
> 使用`NS_SWIFT_NOTHROW`宏声明一个产生错误的 Objective-C 方法可以防止该方法作为`throws`方法导入到 Swift。

<a name="catching_and_handling_an_error"></a>
### 捕获和处理错误

在 Objective-C，错误处理是可选的，这意味着方法产生的错误会被忽略，除非提供了一个错误指针。在 Swift，调用一个会抛出错误的方法时必须明确进行错误处理。

下面演示了如何在 Objective-C 中处理调用方法时产生的错误：

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

Swift 中等价的代码如下所示：

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

此外，还可以使用`catch`子句来匹配特定的错误代码从而方便区分不同的失败条件：

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
### 转换错误为可选值

在 Objective-C，可以向错误参数传递`NULL`来忽略错误。在 Swift，可以使用`try?`关键字将`throws`方法转换为返回可选类型的方法，然后检查返回值是否为`nil`。

例如，`NSFileManager`的实例方法`URLForDirectory(_:inDomain:appropriateForURL:create:)`会根据指定搜索路径和域返回一个 URL，如果 URL 不存在也无法被创建，将会产生一个错误。在 Objective-C，可以检查返回的 URL 是否有值来判断此方法成功或是失败。

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

在 Swfit 则应该这样做：

```swift
let fileManager = FileManager.default
if let tmpURL = try? fileManager.url(for: .cachesDirectory, in: .userDomainMask, appropriateFor: nil, create: true) {
    // ...
}
```

<a name="throwing_an_error"></a>
### 抛出错误

如果一个错误发生在 Objective-C 方法中，那么错误对象会填充方法的错误指针参数：

```objective-c
// 发生一个错误
if (errorPtr) {
   *errorPtr = [NSError errorWithDomain:NSURLErrorDomain
                                   code:NSURLErrorCannotOpenFile
                               userInfo:nil];
}
```

如果一个错误发生在 Swift 方法中，那么错误会被抛出，并且自动传递给调用者：

```swift
// 发生一个错误
throw NSError(domain: NSURLErrorDomain, code: NSURLErrorCannotOpenFile, userInfo: nil)
```

如果 Objective-C 代码调用会抛出错误的 Swift 方法，那么发生错误时，该错误会自动填充到桥接而来的 Objective-C 方法的错误指针参数。

例如，思考`NSDocument`中的`read(from:ofType:)`方法。在 Objective-C，这个方法的最后一个参数是`NSError **`。在 Swift 的`NSDocument`子类中重写该方法时，该方法会以抛出错误的方式替代错误指针参数。

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

如果方法无法使用正规的文件内容来创建一个对象，就会抛出一个`NSError`对象。如果该方法在 Swift 中调用，那么该错误会被传递到它的调用域。如果该方法在 Objective-C 中调用，错误会填充到错误指针参数。

> 注意  
> 尽管 Swift 的错误处理类似 Objective-C 的异常处理，但它是完全不同的功能。如果一个 Objective-C 方法在运行时抛出了一个异常，对于 Swift 来说，则会触发一个运行时错误。无法直接在 Swift 中重新获得来自 Objective-C 的异常。任何异常处理行为必须在 Objective-C 代码中实现。

<a name="key_value_observing"></a>
## 键值观察

键值观察是一种能让某个对象在其他对象特定属性变化时得到通知的机制。只要 Swift 类源自 NSObject 类，就可以在 Swift 中通过如下三步来实现键值观察：

1. 为想要观察的属性添加`dynamic`修改符。关于`dynamic`的更多信息，请参阅 [强制动态派发](01-Interacting%20with%20Objective-C%20APIs.md#%E5%BC%BA%E5%88%B6%E5%8A%A8%E6%80%81%E6%B4%BE%E5%8F%91) 小节。

	```swift
	class MyObjectToObserve: NSObject {
	
	    dynamic var myDate = NSDate()
	    
	    func updateDate() {
	        myDate = NSDate()
	    }
	}
	```

2. 创建一个全局的上下文变量。

	```swift
	private var myContext = 0
	```

3. 为键路径添加一个观察者，重写`observeValue(for:of:change:context:)`方法，并在`deinit`中移除观察者。

	```swift
	class MyObserver: NSObject {
	    var objectToObserve = MyObjectToObserve()
	    override init() {
	        super.init()
	        objectToObserve.addObserver(self, forKeyPath: "myDate", options: .new, context: &myContext)
	    }
	
	    override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey : Any]?, context: UnsafeMutableRawPointer?) {
	        if context == &myContext {
	            if let newValue = change?[.newKey] {
	                print("Date changed: \(newValue)")
	            }
	        } else {
	            super.observeValue(forKeyPath: keyPath, of: object, change: change, context: context)
	        }
	    }
	
	    deinit {
	        objectToObserve.removeObserver(self, forKeyPath: "myDate", context: &myContext)
	    }
	}
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

在 Objective-C，可以使用 Foundation 框架的`NSJSONSerialiation`类和`NSPropertyListSerialization`类，利用 JSON 或者属性列表来实例化对象，通常这种对象会是`NSDictionary<NSString *, id>`类型。Swift 也支持此功能，但由于 Swift 是类型安全的，因此需要一些额外的类型转换。

例如，思考下面的`Venue`结构，它有一个名为`name`的`String`类型的属性，一个名为`coordinates`的
`CLLocationCoordinate2D`类型的属性，以及一个名为`category`的`Category`枚举类型的属性，并且枚举类型是嵌套类型：

```swift
import Foundation
import CoreLocation

struct Venue {
    enum Category: String {
        case entertainment
        case food
        case nightlife
        case shopping
    }

    var name: String
    var coordinates: CLLocationCoordinate2D
    var category: Category
}
```

使用`Venue`结构的应用程序可能会从一个网络服务器接收到如下这些 JSON 数据：

```
{
    "name": "Caffe Macs",
    "coordinates": {
        "lat": 37.330576,
        "lng": -122.029739
    },
    "category": "Food"
}
```

可以提供一个可失败的`Venue`构造器，接受一个`[String: Any]`类型的`attributes`参数，也就是`NSJSONSerialiation`或`NSPropertyListSerialization`的返回值类型：

```swift
init?(attributes: [String: Any]) {
    guard let name = attributes["name"] as? String,
        let coordinates = attributes["coordinates"] as? [String: Double],
        let latitude = coordinates["lat"],
        let longitude = coordinates["lng"],
        let category = Category(rawValue: attributes["category"] as? String ?? "Invalid")
        else {
            return nil
    }

    self.name = name
    self.coordinates = CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    self.category = category
}
```

`guard`语句包含多个可选绑定表达式，从而确保`attributes`参数提供的所有值都符合预期的类型。如果任意一个可选绑定表达式在赋值给常量时失败，`guard`语句会立即停止评估剩余的条件，并执行`else`分支返回`nil`。

可以使用`NSJSONSerialization`利用 JSON 数据创建一个字典，然后将该字典传给`Venue`的构造器，从而创建一个
`Venue`实例：

```swift
let JSON = "{\"name\": \"Caffè Macs\",\"coordinates\": {\"lat\": 37.330576,\"lng\": -122.029739},\"category\": \"food\"}"
let data = JSON.data(using: String.Encoding.utf8)!
let attributes = try! JSONSerialization.jsonObject(with: data, options: []) as! [String: Any]

let venue = Venue(attributes: attributes)!
print(venue.name)
// 打印 Caffè Macs
```

<a name="validating_serialized_representations"></a>
### 验证序列化数据

在先前的例子中，`Venue`的构造器只会在所有必要信息齐备的情况下返回一个`Venue`实例，否则只会简单地返回`nil`。

像上述例子这样根据一系列值构造实例时，如果能在构造失败时判断出失败原因并向外传递会很有帮助。为了实现这一点，可以将可失败构造器重构为一个抛出错误的构造器：

```swift
enum ValidationError: Error {
    case missing(String)
    case invalid(String)
}

init(attributes: [String: Any]) throws {
    guard let name = attributes["name"] as? String else {
        throw ValidationError.missing("name")
    }

    guard let coordinates = attributes["coordinates"] as? [String: Double] else {
        throw ValidationError.missing("coordinates")
    }

    guard let latitude = coordinates["lat"],
        let longitude = coordinates["lng"]
        else {
            throw ValidationError.invalid("coordinates")
    }

    guard let categoryName = attributes["category"] as? String else {
        throw ValidationError.missing("category")
    }

    guard let category = Category(rawValue: categoryName) else {
        throw ValidationError.invalid("category")
    }

    self.name = name
    self.coordinates = CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    self.category = category
}
```

之前的构造器使用一个`guard`语句一次性检查`attributes`中的所有值，而新的构造器单独检查每个值，如果某个值不存在或者无效，则抛出相应错误。

例如，如果 JSON 数据未提供`name`字段及其对应的值，构造器会抛出枚举值`ValidationError.Missing`，并将`name`字段作为其关联值：

```
{
    "coordinates": {
        "lat": 37.77492,
        "lng": -122.419
    },
    "category": "Shopping"
}
```

```swift
let JSON = "{\"coordinates\": {\"lat\": 37.7842, \"lng\": -122.4016}, \"category\": \"Convention Center\"}"
let data = JSON.data(using: String.Encoding.utf8)!
let attributes = try! JSONSerialization.jsonObject(with: data, options: []) as! [String: Any]

do {
    let venue = try Venue(attributes: attributes)
} catch ValidationError.missing(let field) {
    print("Missing Field: \(field)")
}
// 打印 Missing Field: name
```

或者，如果 JSON 数据提供了所有字段，但是`category`字段对应的值不匹配`Category`枚举中的任何原始值，构造器会抛出枚举值`ValidationError.Invalid`，并将`category`字段作为其关联值：

```
{
    "name": "Moscone West",
    "coordinates": {
        "lat": 37.7842,
        "lng": -122.4016
    },
    "category": "Convention Center"
}
```

```swift
let JSON = "{\"name\": \"Moscone West\", \"coordinates\": {\"lat\": 37.7842, \"lng\": -122.4016}, \"category\": \"Convention Center\"}"
let data = JSON.data(using: String.Encoding.utf8)!
let attributes = try! JSONSerialization.jsonObject(with: data, options: []) as! [String: Any]

do {
    let venue = try Venue(attributes: attributes)
} catch ValidationError.invalid(let field) {
    print("Invalid Field: \(field)")
}
// 打印 Invalid Field: category
```

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

<a name="Processing Command-Line Arguments"></a>
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
