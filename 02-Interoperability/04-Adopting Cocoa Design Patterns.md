# 采用 Cocoa 设计模式

本页包含内容：

- [代理](#Delegation)
- [惰性初始化](#Lazy Initialization)
- [错误处理](#error_handling)
- [键值观察](#Key-Value_Observing)
- [撤销](#Undo)
- [目标-动作](#Target_Action)
- [单例](#Singleton)
- [内省](#Introspection)
- [序列化](#Serializing)
- [API 可用性](#API_Availability)
- [处理命令行参数](#Processing Command-Line Arguments)

使用 Cocoa 既有的设计模式，能帮助开发者开发出设计巧妙、扩展性强的应用程序。这些模式很多都依赖于在 Objective-C 中定义的类。由于 Swift 与 Objective-C 的互用性，依然可以在 Swift 中使用这些设计模式。在某些情况下，甚至可以使用 Swift 的语言特性扩展或简化这些 Cocoa 设计模式，使这些设计模式更加强大易用。

<a name="Delegation"></a>
## 代理

在 Swift 和 Objective-C，代理通常由一个定义交互方法的协议和遵循协议的代理属性表示。就像在 Objective-C，在向代理发送可能无法响应的消息之前，应询问代理能否响应消息。在 Swift，可以使用可选链语法在一个可能为`nil`的对象上调用可选的代理方法，并使用`if-let`语法解包未必有值的返回值。

下面的代码阐明了这个过程：

1. 检查`myDelegate`不为`nil`。
2. 检查`myDelegate`是否实现了`window:willUseFullScreenContentSize:`方法。
3. 如果步骤1和步骤2顺利通过，那么调用该方法，将返回值赋值给名为`fullScreenSize`的常量。
4. 在控制台打印方法的返回值。

```swift
class MyDelegate: NSObject, NSWindowDelegate {
    func window(window: NSWindow, willUseFullScreenContentSize proposedSize: NSSize) -> NSSize {
        return proposedSize
    }
}

var myDelegate: NSWindowDelegate? = MyDelegate()
if let fullScreenSize = myDelegate?.window?(myWindow, willUseFullScreenContentSize: mySize) {
    print(NSStringFromSize(fullScreenSize))
}
```

<a name="Lazy Initialization"></a>
## 惰性初始化

惰性属性的值只会在被初次访问时才被初始化。当属性的初始化过程十分复杂或者代价昂贵，或者属性的初始值无法在实例的构造过程完成前确定时，惰性属性会十分有用。

在 Objective-C，一个属性可能会覆写其自动合成的 getter 方法，只在实例变量为`nil`时才初始化实例变量：

```objective-c
@property NSXMLDocument *XMLDocument;
     
- (NSXMLDocument *)XMLDocument {
    if (_XMLDocument == nil) {
        _XMLDocument = [[NSXMLDocument alloc] initWithContentsOfURL:[[NSBundle mainBundle]     
            URLForResource:@"/path/to/resource" withExtension:@"xml"] options:0 error:nil];
    }
    return _XMLDocument;
}
```

在 Swift，可以使用`lazy`修饰符声明一个存储属性，这将使计算初始值的表达式只在属性被初次访问时才进行求值：

```swift
lazy var XMLDocument: NSXMLDocument = try! NSXMLDocument(contentsOfURL: 
    NSBundle.mainBundle().URLForResource("document", withExtension: "xml")!, options: 0)
```

由于惰性属性只在被初次访问时才进行初始化，此时实例本身已被完全初始化，因此在初始化表达式中可以使用`self`：

```swift
var pattern: String
lazy var regex: NSRegularExpression = try! NSRegularExpression(pattern: self.pattern, options: [])
```

如果还需在初始化的基础上进行额外的设置，则可以将一个自求值的闭包赋值给属性：

```swift
lazy var ISO8601DateFormatter: NSDateFormatter = {
    let formatter = NSDateFormatter()
    formatter.locale = NSLocale(localeIdentifier: "en_US_POSIX")
    formatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ssZZZZZ"
    return formatter
}()
```

> 注意  
> 如果一个惰性属性还未被初始化就被多个线程同时访问，此时无法保证此惰性属性只被初始化一次。

请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 的 [延迟存储属性](http://wiki.jikexueyuan.com/project/swift/chapter2/10_Properties.html#stored_properties) 小节。

<a name="error_handling"></a>
## 错误处理

在 Cocoa 中，产生错误的方法将`NSError`指针参数作为最后一个参数，当错误产生时，该参数会被`NSError`对象填充。Swift 会自动将 Objective-C 中产生错误的方法转换为根据 Swift 原生错误处理功能抛出错误的方法。

> 注意  
> 某些产生错误的方法，例如代理方法，或者方法接受 block 参数，而 block 接受`NSError`对象参数，不会被 Swift 导入为`throws`方法。

例如，思考下面的来自于`NSFileManager`的 Objective-C 方法：

```objective-c
- (BOOL)removeItemAtURL:(NSURL *)URL
                  error:(NSError **)error;
```

在 Swift，它会被这样导入：

```swift
func removeItemAtURL(URL: NSURL) throws
```

注意`removeItemAtURL(_:)`方法被 Swift 导入时，返回值类型为`Void`，没有`error`参数，而有一个`throws`声明。

如果 Objective-C 方法的最后一个非 block 参数是`NSError **`类型，Swift 会将之替换为`throws`关键字，以表明该方法可以抛出一个错误。如果 Objective-C 方法的错误参数是它的第一个参数，Swift 会尝试删除选择器第一部分中的“WithError”或“AndReturnError”后缀来进一步简化方法名。如果简化后的方法名会和其他方法名冲突，则不会进行简化。

如果产生错误的 Objective-C 方法返回一个用来表示方法调用成功或失败的`BOOL`值，Swift 会把返回值转换为`Void`。同样的，如果产生错误的 Objective-C 方法返回一个`nil`值来表明方法调用失败，Swift 会把返回值转换为非可选类型。

否则，如果没有约定可供推断，则该方法保持不变。

> 注意  
> 使用`NS_SWIFT_NOTHROW`宏声明一个产生错误的 Objective-C 方法可以防止该方法作为`throws`方法导入到 Swift。

### 捕获和处理错误

在 Objective-C，错误处理是可选的，这意味着方法产生的错误会被忽略，除非提供了一个错误指针。在 Swift，调用一个会抛出错误的方法时必须显式地进行错误处理。

下面演示了如何在 Objective-C 中处理调用方法时产生的错误：

```objective-c
NSFileManager *fileManager = [NSFileManager defaultManager];
NSURL *fromURL = [NSURL fileURLWithPath:@"/path/to/old"];
NSURL *toURL = [NSURL fileURLWithPath:@"/path/to/new"];
NSError *error = nil;
BOOL success = [fileManager moveItemAtURL:URL toURL:toURL error:&error];
if (!success) {
    NSLog(@"Error: %@", error.domain);
}
```

下面是 Swift 中等价的代码：

```swift
let fileManager = NSFileManager.defaultManager()
let fromURL = NSURL(fileURLWithPath: "/path/to/old")
let toURL = NSURL(fileURLWithPath: "/path/to/new")
do {
    try fileManager.moveItemAtURL(fromURL, toURL: toURL)
} catch let error as NSError {
    print("Error: \(error.domain)")
}
```

此外，还可以使用`catch`子句来匹配特定的错误代码从而区分不同的失败条件：

```swift
do {
    try fileManager.moveItemAtURL(fromURL, toURL: toURL)
} catch NSCocoaError.FileNoSuchFileError {
    print("Error: no such file exists")
} catch NSCocoaError.FileReadUnsupportedSchemeError {
    print("Error: unsupported scheme (should be 'file://')")
}
```

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

在 Swfit 应该这样：

```swift
let fileManager = NSFileManager.defaultManager()
if let tmpURL = try? fileManager.URLForDirectory(.CachesDirectory, 
                                       inDomain: .UserDomainMask, 
                              appropriateForURL: nil,
                                         create: true) {
    // ...
}
```

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

如果 Objective-C 代码调用会抛出错误的 Swift 方法，那么发生错误时，该错误会自动传递给桥接而来的 Objective-C 方法的错误指针参数。

例如，思考`NSDocument`中的`readFromFileWrapper(_:ofType:)`方法。在 Objective-C，这个方法的最后一个参数是`NSError **`。在 Swift 的`NSDocument`子类中重写该方法时，该方法会以抛出错误的方式替代错误指针参数。

```swift
class SerializedDocument: NSDocument {

    static let ErrorDomain = "com.example.error.serialized-document"
    
    var representedObject: [String: AnyObject] = [:]
    
    override func readFromFileWrapper(fileWrapper: NSFileWrapper, ofType typeName: String) throws {
        guard let data = fileWrapper.regularFileContents else {
            throw NSError(domain: NSURLErrorDomain, code: NSURLErrorCannotOpenFile, userInfo: nil)
        }
        
        if case let JSON as [String: AnyObject] = try NSJSONSerialization.JSONObjectWithData(data, options: []) {
            self.representedObject = JSON
        } else {
            throw NSError(domain: SerializedDocument.ErrorDomain, code: -1, userInfo: nil)
        }
    }
}
```

如果方法无法使用正规的文件内容来创建一个对象，则会抛出一个`NSError`对象。如果该方法在 Swift 中调用，那么该错误会被传递到它的调用域。如果该方法在 Objective-C 中调用，错误会填充到错误指针参数。

> 注意  
> 尽管 Swift 的错误处理类似 Objective-C 的异常处理，但它是完全不同的功能。如果一个 Objective-C 方法在运行时抛出了一个异常，对于 Swift 来说，则会触发一个运行时错误。无法直接在 Swift 中重新获得来自 Objective-C 的异常。任何异常处理行为必须在 Objective-C 代码中实现。

<a name="Key-Value_Observing"></a>
## 键值观察

键值观察能让对象获得其他对象特定属性的变化通知。只要 Swift 类继承自 NSObject 类，就可以在 Swift 中通过下面三步来实现键值观察：

1. 为想要观察的属性添加`dynamic`修改符。关于`dynamic`的更多信息，请参阅 [要求动态派发](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/01Interacting%20with%20Objective-C%20APIs.md#%E8%A6%81%E6%B1%82%E5%8A%A8%E6%80%81%E6%B4%BE%E5%8F%91requiring-dynamic-dispatch)。

```swift
class MyObjectToObserve: NSObject {

    dynamic var myDate = NSDate()
    
    func updateDate() {
        myDate = NSDate()
    }
}
```

2. 创建一个全局上下文变量。

```swift
private var myContext = 0
```

3. 为 key-path 添加一个观察者，重写`observeValueForKeyPath:ofObject:change:context:`方法，并在`deinit`中移除观察者。

```swift
class MyObserver: NSObject {

    var objectToObserve = MyObjectToObserve()
    
    override init() {
        super.init()
        objectToObserve.addObserver(self, forKeyPath: "myDate", options: .New, context: &myContext)
    }    
    
    override func observeValueForKeyPath(keyPath: String?, 
        ofObject object: AnyObject?, change: [NSObject : AnyObject]?, context: UnsafeMutablePointer<Void>) {
    
        if context == &myContext {
            if let newValue = change?[NSKeyValueChangeNewKey] {
                print("Date changed: \(newValue)")
            }
        } else {
            super.observeValueForKeyPath(keyPath, ofObject: object, change: change, context: context)
        }
    }
    
    deinit {
        objectToObserve.removeObserver(self, forKeyPath: "myDate", context: &myContext)
    }
}
```

<a name="Undo"></a>
## 撤销

在 Cocoa 中，可以使用`NSUndoManager`注册一个操作，从而允许用户撤销该操作。在 Swift，可以像在 Objective-C 一样利用 Cocoa 的撤销功能。

对于应用程序响应链上的对象，也就是 OS X 的`NSResponder`和 iOS 的`UIResponder`，以及它们的子类，都有一个只读的`undoManager`属性。该属性返回一个可选类型的`NSUndoManager`对象，该对象管理着应用程序的撤销栈。每当用户进行一个操作时，例如编辑文本内容，或是删除表视图的选中行，可以用`NSUndoManager`注册一个撤销操作，从而允许用户恢复到操作之前的状态。一个撤销操作会记录必要的步骤来恢复到操作之前的状态，例如将文本内容设置为修改前的原始内容，或是重新添加被删除的选中行。

`NSUndoManager`支持两种方式注册撤销操作：一种是“简单撤销”，它会执行一个选择器，最多只能接收一个对象类型的参数；另一种则使用`NSInvocation`对象，从而可以支持多个参数以及多种参数类型。

例如，思考下面的`Task`模型，它使用`ToDoListController`来展示一个完成的任务列表：

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

对于 Swift 中的属性，可以在属性观察器`willSet`中创建一个撤销操作，使用`self`作为`target`，相应的 Objective-C setter 方法作为`selector`，属性的当前值作为`object`：

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
     
func markTask(task: Task, asCompleted completed: Bool) {
    if let target = undoManager?.prepareWithInvocationTarget(self) as? ToDoListController {
        target.markTask(task, asCompleted: !completed)
        undoManager?.setActionName(NSLocalizedString("todo.task.mark", comment: "Mark As Completed"))
    }
        
    task.completed = completed
    tableView.reloadData()
        
    let numberRemaining = tasks.filter{ $0.completed }.count
    remainingLabel.string = String(format: NSLocalizedString("todo.task.remaining", 
        comment: "Tasks Remaining: %d"), numberRemaining)
}
```

`prepareWithInvocationTarget(_:)`方法会返回`target`的代理。通过转换为`ToDoListController`，就可以直接对`markTask(_:asCompleted:)`发起调用。

更多信息请参阅 [The Undo Architecture Programming GuideTarget-Action](https://developer.apple.com/library/prerelease/ios/documentation/Cocoa/Conceptual/UndoArchitecture/UndoArchitecture.html#//apple_ref/doc/uid/10000010-SW1)。

<a name="Target_Action"></a>
## 目标-动作

目标-动作是一种常见的 Cocoa 设计模式，用于当特定事件发生，需要一个对象向另一个对象发送消息时。Swift 和 Objective-C 的目标-动作模式基本类似。在 Swift，可以使用`Selector`类型引用 Objective-C 的选择器。请参阅 [Objective-C 选择器](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/01Interacting%20with%20Objective-C%20APIs.md#objective-c-%E9%80%89%E6%8B%A9%E5%99%A8objective-c-selectors) 查看在 Swift 使用目标-动作模式的示例。

<a name="Singleton"></a>
## 单例

单例对象提供了一个共享的可全局访问的对象。可以创建自己的单例对象在应用程序内共享，从而提供一个统一的资源或服务的访问入口,比如一个播放音效的音频通道或发起 HTTP 请求的网络管理器。

在 Objective-C，可以用`dispatch_once`函数包裹初始化代码，从而保证在应用程序的生命周期内，block 内的代码只会执行一次，这样就确保了只有唯一的实例会被创建。

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

如果需要进行额外的设置，则可以使用闭包来初始化：

```swift
class Singleton {
    static let sharedInstance: Singleton = {
        let instance = Singleton()
        // 设置代码
        return instance
    }()
}
```

请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 的 [类型属性](http://wiki.jikexueyuan.com/project/swift/chapter2/10_Properties.html#type_properties) 小节获取更多信息。

<a name="Introspection"></a>
## 内省

在 Objective-C，使用`isKindOfClass:`方法检查某个对象是否是特定类型的实例，以及使用`conformsToProtocol:`方法检查某个对象是否符合特定协议。在 Swift，可以使用`is`操作符实现上述功能，也可以使用`as?`操作符向下转换到指定类型。

可以使用`is`操作符检查一个实例是否是指定类或其子类的实例。若是，那么`is`返回结果为`true`，反之为`false`。

```swift
if object is UIButton {
    // object 是 UIButton 类型
} else {
    // object 不是 UIButton 类型
}
```

也可以使用`as?`操作符尝试向下转换到子类型。`as?`操作符返回可选类型的值，可结合`if-let`语句使用。

```swift
if let button = object as? UIButton {
    // object 成功转换为 UIButton 并绑定到 button
} else {
    // object 无法转换为 UIButton
}
```
请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 的 [类型转换](http://wiki.jikexueyuan.com/project/swift/chapter2/19_Type_Casting.html) 章节获取更多信息。

检查协议符合性以及转换到符合协议类型的语法和上述语法是完全一样的。下面是使用`as?`检查协议符合性的示例：

```swift
if let dataSource = object as? UITableViewDataSource {
    // object 符合 UITableViewDataSource 协议并绑定到 dataSource
} else {
    // object 不符合 UITableViewDataSource 协议
}
```

注意，经过转换之后，`dataSource`常量即是`UITableViewDataSource`类型，所以只能访问和调用`UITableViewDataSource`协议定义的属性和方法。想进行其他操作时，必须将其转换为其他类型。

请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 的 [协议](http://wiki.jikexueyuan.com/project/swift/chapter2/22_Protocols.html) 章节获取更多信息。

<a name="Serializing"></a>
## 序列化

通过序列化，可以在应用程序中编码和解码对象，将其转换为独立于体系结构的表示形式，例如 JSON 或属性列表，或者从这种形式转换回对象。

<a name="API_Availability"></a>
## API 可用性

一些类和方法并不是在你的应用的所有平台的所有版本中都可用。为了确保你的应用能够适应任何功能上的差异，你需要检查这些 API 的可用性。

在 Objective-C 中，你使用`respondsToSelector:`和`instancesRespondToSelector:`方法来检查一个类或者实例的方法是否可用。如果没有检查，调用方法则可能会抛出`NSInvalidArgumentException`"unrecognized selector sent to instance"异常。例如，`requestWhenInUseAuthorization`方法只从 iOS 8.0 和 OS X 10.10 开始对`CLLocationManager`实例可用。

```objective-c
if ([CLLocationManager instancesRespondToSelector:@selector(requestWhenInUseAuthorization)]) {
    // 方法可用
} else {
    // 方法不可用
}
```

在 Swift 中，尝试着调用一个目标平台版本不支持的方法将会报出编译时错误。

下面是上一个例子，采用 Swift 编写：

```swift
let locationManager = CLLocationManager()
locationManager.requestWhenInUseAuthorization()
// error: only available on iOS 8.0 or newer
```

如果应用的目标版本低于 iOS 8.0 或者 OS X 10.10，`requestWhenInUseAuthorization()`方法则不可用，所以编译器会报告错误。

Swift 代码可以使用 API 可用性来作为运行时的条件判断。可用性检查可以使用在一个控制流语句的条件中，例如`if`,`guard`或者`while`语句。

拿前面的例子举例，你可以使用`if`语句来检查可用性，只有当方法在运行时可用时方可调用`requestWhenInUseAuthorization()`。

```swift
let locationManager = CLLocationManager()
if #available(iOS 8.0, OSX 10.10, *) {
    locationManager.requestWhenInUseAuthorization()
}
```

或者，你可以使用`guard`语句来检查可用性，除非当前的目标符合规定要求，否则将会退出作用域。这种方法简化了处理不同平台功能的逻辑。

```swift
let locationManager = CLLocationManager()
guard #available(iOS 8.0, OSX 10.10, *) else { return }
locationManager.requestWhenInUseAuthorization()
```

每个平台参数由下面列出的平台名称组成，后面跟着相应的版本号。最后一个参数是一个星号（*），是用来处理未来潜在的平台。

平台名称：

- iOS
- iOSApplicationExtension
- OSX
- OSXApplicationExtension
- watchOS

所有的 Cocoa API 都提供有可用性信息，所以你可以很有把握地编写应用所针对的平台的代码。

你可以用`@available`属性标注你的 API 声明来表明其可用性。`@available`属性使用和`#available`同样的语法来做运行时检查，平台版本要求参数都以逗号隔开。

例如：

```swift
@available(iOS 8.0, OSX 10.10, *)
func useShinyNewFeature() {
    // ...
}
```

> 注意  
> 使用`@available`属性标记的方法可以安全地使用满足特定平台需求的可用 API 而不用显式地做可用性检查。

<a name="Processing Command-Line Arguments"></a>
## 处理命令行参数
