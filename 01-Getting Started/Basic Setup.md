# 基本设置

本页包含内容：

-   [搭建你的 Swift 环境](#setting_up_your_swift_environment)

-   [理解 Swift 导入过程](#understanding_the_swift_import_process)

Swift 被设计用来无缝兼容 Cocoa 和 Objective-C。在 Swift 中，你可以使用 Objective-C API（无论是系统框架还是你自己的代码），你也可以在 Objective-C 中使用 Swift API。这种兼容性让 Swift 以了一个简单、方便、强大的工具集成到你的 Cocoa 应用开发流程中。

这篇指南包括了三个有关兼容性的重要方面，让你可以更好地利用兼容性来开发 Cocoa 应用：

- *互用性* 你可以将 Swift 和 Objective-C 相接合，在 Objective-C 中使用 Swift 类，并利用熟悉的 Cocoa 类、设计模式以及实践经验。
- *混合搭配* 你可以创建结合了 Swift 和 Objective-C 文件的混合语言应用，并且它们能跟彼此进行通信。
- *迁移* 由于以上两点，从现有的 Objective-C 代码迁移到 Swift 会非常简单，这使运用最新的 Swift 特性取代你的 Objective-C 应用中的一部分内容成为了可能。

在你开始学习这些特性前，你需要对如何搭建 Swift 环境来访问 Cocoa 系统框架有个大体了解。

<a name="setting_up_your_swift_environment"></a>
## 搭建你的 Swift 环境

为了开始体验在 Swift 中访问 Cocoa 框架，使用 Xcode 的一个模板来创建一个基于 Swift 的应用。

##### 在 Xcode 中创建一个 Swift 项目

1.选择`File > New > Project > (iOS or OS X) > Application > your template of choice`。

2.点击 Language 弹出菜单并选择 Swift。

![image](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/Art/newproject_2x.png)

Swift 项目的结构几乎和 Objective-C 项目一模一样，只有一个重要的区别：Swift 没有头文件。在接口和实现之间没有显示的划分，所以一个类中的所有信息都在一个单独的`.swift`文件中。

现在开始，你可以开始体验在`AppDelegate`中编写 Swift 代码，或者你可以通过选择`File > New > File > (iOS or OS X) > Other > Swift`来创建一个新的 Swift 类。

> 注意  
> Executables built from the command line expect to find the Swift libraries in their @rpath. If you plan to ship a Swift executable built from the command line, you’ll need to ship the Swift dynamic libraries as well. Swift executables built from within Xcode have the runtime statically linked. 

<a name="understanding_the_swift_import_process"></a>
## 理解 Swift 导入过程

在你建立 Xcode 项目后，你可以在 Swift 里导入任何 Cocoa 平台的框架。

任何 Objective-C 框架（或 C 语言类库）都将将作为一个 *module* 直接导入到 Swift 中。这包括了所有 Objective-C 系统框架——比如 Foundation、UIKit 和 SpriteKit，就像系统支持公共的 C 语言类库一样。举个例子，想导入 Foundation，只要简单地添加 import 语句到你的 Swift 文件的顶部：

```swift 
import Foundation
```

这个 import 导入了所有 Foundation 的 API，包括`NSDate`，`NSURL`，`NSMutableData`，以及它们的所有方法、属性和分类，并且这些都可以在 Swift 中直接使用。

导入过程是非常简洁的。在 Objective-C 中，框架在头文件中公开声明 API。在 Swift 中，那些头文件被编译成 Objective-C 的模块，接着被导入到 Swift 作为 Swift 的 API。该导入决定了 Objective-C 代码中的函数，类，方法以及声明的类型如何在 Swift 中出现。对于函数和方法，这个过程将影响它们的参数和返回值的类型。对于类型来说，导入过程会做下面这些事情：

- 重映射 Objective-C 类型到 Swift 中的同等类型，就像`id`到`AnyObject`
- 重映射 Objective-C 核心类型到 Swift 中的替代类型，就像`NSString`到`String`
- 重映射 Objective-C 概念到 Swift 中相对应的概念，就像指针到可选

在[与 Objective-C API 交互](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/01Interacting%20with%20Objective-C%20APIs.md)章节，你将会了解到关于这些映射的更多信息，以及如何在你的 Swift 代码中利用它们。导入 Swift 的模块到 Objective-C 和将 Objective-C 的模块导入到 Swift 的过程是非常相似的。Swift 声明它的 API，比如来自系统框架的 API，作为 Swift 模块。伴随这些 Swift 模块的生成，还会生成 Objective-C 头文件。这些头文件声明了那些可以映射回 Objective-C 的 API。一些 Swift API 无法映射回 Objective-C，因为它们利用了 Objective-C 中不存在的语言特性。关于在 Objective-C 中使用 Swift 的更多信息，请参看[在同一项目中使用 Swift 和 Objective-C](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/03Mix%20and%20Match/Swift%20and%20Objective-C%20in%20the%20Same%20Project.md)。

> 注意  
> 你不能直接把 C++ 代码导入 Swift，必须为 C++ 代码创建一个 Objective-C 或者 C 的封装。
