# 采用 Cocoa 设计模式

本页包含内容：

- [代理](#Delegation)
- [错误处理](#error_handling)
- [键值观察](#Key-Value_Observing)
- [目标——动作](#Target_Action)
- [单例](#Singleton)
- [内省](#Introspection)
- [API 可用性](#API_Availability)

使用 Cocoa 现有的一些设计模式，是帮助开发者开发一款拥有合理设计思路、稳定的性能、良好的可扩展性应用的有效方法之一。这些模式都依赖于在 Objective-C 中定义的类。因为 Swift 与 Objective-C 的互用性，所以你依然可以在 Swift 代码中使用这些设计模式。在一些情况下，你甚至可以使用 Swift 语言的特性扩展或简化这些 Cocoa 设计模式，使这些设计模式更强大、更易于使用。

<a name="Delegation"></a>
## 委托

在 Swift 和 Objective-C 中，委托通常由一个定义交互方法的协议和遵循协议的委托属性表示。就像在 Objective-C 中，在你向委托发送可能无法响应的消息之前，你会去询问委托是否能响应消息。在 Swift 中，你可以使用可选链接在一个可能为`nil`的对象上调用可选的代理方法，并使用`if-let`语法展开可能存在的返回值。

下面列出的代码可以说明这个过程：

1. 检查`myDelegate`不为`nil`。
2. 检查`myDelegate`是否实现了`window:willUseFullScreenContentSize:`方法。
3. 如果步骤1和2检查结果成立，那么调用该方法，将该方法的返回值分配给名为`fullScreenSize`的常量。
4. 将该方法的返回值输出在控制台。

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

<a name="error_handling"></a>
## 错误处理

在 Cocoa 中，产生错误的方法将`NSError`指针参数作为最后一个参数，当错误产生时，该参数会被`NSError`对象填充。Swift 会自动将 Objective-C 中产生错误的方法转换为根据 Swift 原生错误处理功能抛出错误的方法。

> 注意

> 某些产生错误的方法，例如代理方法或者参数是拥有`NSError`对象参数的 block 的方法，不会被 Swift 处理为`throw`方法。

例如，考虑下面的来自于`NSFileManager`的 Objective-C 方法：

```objective-c
- (BOOL)removeItemAtURL:(NSURL *)URL
                  error:(NSError **)error;
```

在 Swift 中，它会被这样导入：

```swift
func removeItemAtURL(URL: NSURL) throws
```

注意到`removeItemAtURL(_:)`方法被 Swift 导入时，返回值类型为`Void`，没有`error`参数，而有一个`throws`声明。

如果 Objective-C 方法的最后一个非闭包参数是`NSError **`类型，Swift 则会将之替换为`throws`关键字，以表明该方法可以抛出一个错误。如果 Objective-C 方法的错误参数也是它的第一个参数，Swift 则会尝试通过删除选择器的第一部分中的"WithError"或"AndReturnError" 后缀来进一步简化方法名。如果简化后的方法名会和另一方法的方法名冲突，则不会简化该方法名。

如果产生错误的 Objective-C 的方法返回一个用来表示方法调用成功或失败的`BOOL`值，Swift 会把函数的返回值转换为`Void`。同样的，如果产生错误的 Objective-C 方法返回一个`nil`值来表明方法调用的失败，Swift 会把函数的返回值转换为非可选类型。

否则，如果没有约定可供推断，则该方法保持不变。

> 注意

> 使用`NS_SWIFT_NOTHROW `宏声明一个产生错误的 Objective-C 方法可以防止该方法作为`throw`方法导入 Swift 中。

### 捕获和处理错误

在 Objective-C 中，错误处理是可选的，意味着方法产生的错误会被忽略除非你提供了一个错误指针。在 Swift 中，调用一个会抛出错误的方法要求显式地进行错误处理。

下面是如何在 Objective-C 中处理调用方法时产生的错误：

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

此外，你可以使用`catch`从句来匹配特定的错误代码来区分不同的失败条件：

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

在 Objective-C 中，你可以向错误参数传递`NULL`来忽略潜在的错误。在 Swift 中，你使用`try?`关键字将一个抛出异常的方法转换为一个返回可选值的方法，然后检查返回值是否为`nil`。

例如，`NSFileManager`的实例方法`URLForDirectory(_:inDomain:appropriateForURL:create:)`返回一个表示搜索路径和域的 URL，如果对应的 URL 不存在并且无法创建，将会产生一个错误。在 Objective-C 中，可以通过检查返回的 URL 是否有值来判断此方法成功或是失败。

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

在 Swfit 中你可以这样做：

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

如果一个错误发生在 Objective-C 方法中，那么该错误被用来填充方法的错误指针参数：

```objective-c
// 发生一个错误
if (errorPtr) {
   *errorPtr = [NSError errorWithDomain:NSURLErrorDomain
                                   code:NSURLErrorCannotOpenFile
                               userInfo:nil];
}
```

如果一个错误发生在 Swift 方法中，那么该错误便会被抛出，并且会自动传递给调用者：

```swift
// 发生一个错误
throw NSError(domain: NSURLErrorDomain, code: NSURLErrorCannotOpenFile, userInfo: nil)
```

如果 Objective-C 代码调用抛出错误的 Swift 方法，那么发生错误时该错误会被自动传递给桥接过来的 Objective-C 方法的错误指针参数。

例如，思考`NSDocument`中的`readFromFileWrapper(_:ofType:)`方法。在 Objective-C 中，这个方法的最后一个参数是`NSError **`。当在 Swift 的`NSDocument`的子类中重写该方法时，该方法会以抛出异常的方式替代错误指针参数。

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

如果方法不能够使用正规的文件的内容来创建一个对象，则会抛出一个`NSError`对象。如果方法是从 Swift 代码中调用的，那么该错误会被传递到它的调用域。如果该方法是在 Objective-C 代码中被调用，错误将会填充到错误指针参数里。

在 Objective-C 中，错误处理是可选的，意味着方法产生的错误会被忽略除非你提供了一个错误指针。在 Swift 中，调用一个会抛出错误的方法要求显式地进行错误处理。

> 注意

> 尽管 Swift 的错误处理类似 Objective-C 的异常处理，但它是完全不同的功能。如果一个 Objective-C 方法运行时抛出了一个异常，Swift 则会触发一个运行时错误。没有办法直接在 Swift 中重新获得来自 Objective-C 的异常。任何异常处理行为必须在 Objective-C 代码中实现。

<a name="Key-Value_Observing"></a>
## 键值观察

键值观察是一种机制，该机制允许对象获得其他对象的特定属性的变化的通知。只要你的 Swift 类继承自 NSObject 类，你就可以在 Swift 中通过下面三步来实现键值观察：

1. 为你想要观察的属性添加`dynamic`修改符。关于`dynamic`的更多信息，请见 [要求动态派发（Requiring Dynamic Dispatch）](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/01Interacting%20with%20Objective-C%20APIs.md#%E8%A6%81%E6%B1%82%E5%8A%A8%E6%80%81%E6%B4%BE%E5%8F%91requiring-dynamic-dispatch)。

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
3. 为 key-path 增加一个观察者，重写`observeValueForKeyPath:ofObject:change:context:`方法，并且在`deinit`中移除观察者。

```swift
class MyObserver: NSObject {

    var objectToObserve = MyObjectToObserve()
    
    override init() {
        super.init()
        objectToObserve.addObserver(self, forKeyPath: "myDate", options: .New, context: &myContext)
    }    
    
    override func observeValueForKeyPath(keyPath: String?, ofObject object: AnyObject?, change: [NSObject : AnyObject]?, context: UnsafeMutablePointer<Void>) {
    
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

<a name="Target_Action"></a>
## 目标-行为模式（Target-Action）

当有特定事件发生，需要一个对象向另一个对象发送消息时，通常采用 Cocoa 的 Target-Action 设计模式。Swift 和 Objective-C 中的 Target-Action 模式基本类似。在 Swift 中，你可以使用`Selector`类型引用 Objective-C 中的选择器。请在 [Objective-C 选择器（Objective-C Selectors）](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/01Interacting%20with%20Objective-C%20APIs.md#objective-c-%E9%80%89%E6%8B%A9%E5%99%A8objective-c-selectors)中查看在 Swift 中使用 Target-Action 设计模式的示例。

<a name="Singleton"></a>
## 单例（Singleton）

单例对象提供了一个共享的可全局访问的对象。你可以创建自己的单例对象在应用程序内共享，从而提供一个统一的资源或服务的访问入口,比如一个播放音效的音频通道或发起 HTTP 请求的网络管理器。

在 Objective-C 中，你可以用在 app 生命周期内保证 block 内代码只执行一次的`dispatch_once`函数包裹初始化代码来确保只有唯一的实例被创建。

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

在 Swift 中，你可以简单地使用保证延迟初始化只执行一次的静态类型属性来实现单例，甚至在多个线程同时访问的情况下也没有问题：

```swift
class Singleton {
    static let sharedInstance = Singleton()
}
```

如果你需要进行额外的设置，则可以使用闭包来初始化：

```swift
class Singleton {
    static let sharedInstance: Singleton = {
        let instance = Singleton()
        // 设置代码
        return instance
    }()
}
```

可以在[《The Swift Programming Language 中文版》](http://wiki.jikexueyuan.com/project/swift/)中的 [类型属性（Type Properties）](http://wiki.jikexueyuan.com/project/swift/chapter2/10_Properties.html#type_properties) 一节查看更多信息。

<a name="Introspection"></a>
## 内省（Introspection）

在 Objective-C 中，你使用`isKindOfClass:`方法检查某个对象是否是特定类型的实例，以及使用`conformsToProtocol:`方法检查某个对象是否符合特定协议。在 Swift 中，你可以使用`is`操作符实现上述功能，也可以使用`as?`操作符向下转换到指定类型。

你可以使用`is`操作符检查一个实例是否是指定类的子类。如果该实例是指定类的子类，那么`is`返回结果为`true`，反之为`false`。

```swift
if object is UIButton {
    // object 是 UIButton 类型
} else {
    // object 不是 UIButton 类型
}
```

你也可以使用`as?`操作符尝试向下转换到子类型，`as?`操作符返回可选值，可结合`if-let`语句使用。

```swift
if let button = object as? UIButton {
    // object 成功转换为 UIButton 并绑定到 button
} else {
    // object 无法转换为 UIButton
}
```
可以在[《The Swift Programming Language 中文版》](http://wiki.jikexueyuan.com/project/swift/)中的 [类型转换（Type Casting）](http://wiki.jikexueyuan.com/project/swift/chapter2/19_Type_Casting.html) 一节查看更多信息。

检查是否符合协议以及转换到符合协议类型的语法是完全一样的，下面是使用`as?`检查符合协议的示例：

```swift
if let dataSource = object as? UITableViewDataSource {
    // object 符合 UITableViewDataSource 协议并绑定到 dataSource
} else {
    // object 不符合 UITableViewDataSource 协议
}
```

注意，当转换之后，`dataSource`常量即是`UITableViewDataSource`类型，所以你只能访问和调用`UITableViewDataSource`协议定义的属性和方法。当你想进行其他操作时，必须将其转换为其他的类型。

可以在[《The Swift Programming Language 中文版》](http://wiki.jikexueyuan.com/project/swift/) 中的 [协议（Protocols）](http://wiki.jikexueyuan.com/project/swift/chapter2/22_Protocols.html) 一节查看更多相关信息。

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
