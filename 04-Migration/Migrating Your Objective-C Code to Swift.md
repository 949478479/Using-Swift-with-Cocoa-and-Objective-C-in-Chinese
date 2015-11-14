# 将 Objective-C 代码迁移到 Swift

本页内容包括：

- [为 Objective-C 代码做好迁移准备](#preparing_your_objective-c_code_for_migration)
- [迁移过程](#the_migration_process)
- [故障排除贴士](#troubleshooting_tips_and_reminders)

迁移工作提供了一个重新审视现有的 Objective-C App 的机会，并能用 Swift 代码替换 Objective-C 代码，以此来改善程序的架构，逻辑以及性能。简而言之，将通过之前学习的工具，即混合搭配以及互用性来完成迁移工作。在决定哪些特性和功能用 Swift 实现，哪些依然用 Objective-C 实现时，混合搭配会让这一切变得简单。而由于互用性，将 Swift 的特性整合到 Objective-C 也并不困难。通过这些工具，可以一步步探索 Swift 广泛的功能并整合到现有的 Objective-C 项目中，而不必立即使用 Swift 重写整个项目。

<a name="preparing_your_objective-c_code_for_migration"></a>
## 为 Objective-C 代码做好迁移准备

在开始迁移代码之前，请确保 Objective-C 和 Swift 代码间有着最佳兼容性。这意味着整理现有项目，并将 Objective-C 现代化特性应用其中。为了更好地与 Swift 无缝交互，现有代码需要遵循现代编码实践。在开始前，这有个简短的实践列表，请参阅 [*Adopting Mordern Objective-C*](https://developer.apple.com/library/prerelease/ios/releasenotes/ObjectiveC/ModernizationObjC/AdoptingModernObjective-C/AdoptingModernObjective-C.html#//apple_ref/doc/uid/TP40014150)。

<a name="the_migration_process"></a>
## 迁移过程

迁移代码到 Swift 的最有效的方式是逐个文件迁移，即一次迁移一个类。由于无法在 Objective-C 中继承 Swift 类，因此最好选择一个没有子类的类。使用单个`.swift`文件替换对应的`.m`和`.h`文件，实现和接口将直接放进这个 Swift 文件。也不用再创建头文件了，Xcode 会在需要的时候自动生成头文件。

### 准备工作

* 在 Xcode 中，选择`File > New > File > (iOS，watchOS，tvOS，OS X) > Source > Swift File`为对应的 Objective-C `.m`和`.h`文件创建一个 Swift 类。可以使用相同或者不同的类名，另外，类前缀在 Swift 中是可选的。

* 导入相关的系统框架。

* 如果需要在 Swift 文件中访问 Objective-C 代码，可以填写一个 Objective-C 桥接头文件。具体的操作步骤，请参阅 [在 App 的 target 内部导入代码](../03-Mix%20and%20Match/Swift%20and%20Objective-C%20in%20the%20Same%20Project.md#importing_code_from_within_the_same_app_target) 小节。

* 为了让 Swift 类能在 Objective-C 中使用，可以继承自 Objective-C 类，或者标记`@objc`特性。如果想为类指定在 Objective-C 中的类名，可以使用`@objc(name)`特性，`name`就是 Swift 类在 Objective-C 中的类名。关于`@objc`的更多信息，请参阅 [Swift 类型兼容性](../02-Interoperability/01-Interacting%20with%20Objective-C%20APIs.md#swift_type_compatibility) 小节。

### 开始工作

* 可以通过继承 Objective-C 类，采用 Objective-C 协议，或者其他方式，来让 Swift 类集成 Objective-C 行为。更多信息请参阅 [使用 Objective-C 特性编写 Swift 类](../02-Interoperability/02-Writing%20Swift%20Classes%20with%20Objective-C%20Behavior.md) 章节。

* 使用 Objective-C API 的时候，需要知道 Swift 是怎样转化某些 Objective-C 特性的。更多信息请参阅 [与 Objective-C 的 API 交互](../02-Interoperability/01-Interacting%20with%20Objective-C%20APIs.md) 章节。

* 使用 Cocoa 框架的代码时，记住某些类型是可以被桥接的，这意味着可以使用某些 Swift 类型去替代某些 Objective-C 类型。更多信息请参阅 [与 Cocoa 数据类型协作](../02-Interoperability/03-Working%20with%20Cocoa%20Data%20Types.md) 章节。

* 使用 Cocoa 设计模式的时候，请参阅 [采用 Cocoa 设计模式](../02-Interoperability/04-Adopting%20Cocoa%20Design%20Patterns.md) 章节。

* 对于如何将属性从 Objective-C 转换到 Swift，请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的 [属性](http://wiki.jikexueyuan.com/project/swift/chapter2/10_Properties.html) 章节。

* 在必要的时候，通过`@objc(name)`特性为 Swift 中的属性或方法提供 Objective-C 名称。例如，可以像下面这样将`enabled`属性的 getter 在 Objective-C 中的名称更改为`isEnabled`：

	```swift
	var enabled: Bool {
		@objc(isEnabled) get {
			// ...
		}
	}
	```

* 分别用`func`和`class func`来表示实例方法（`-`）和类方法（`+`）。

* 将简单宏转换为全局常量，将复杂宏转换为函数。

### 大功告成

* 将 Objective-C 代码中对应的导入语句更改为`#import "ProductModuleName-Swift.h"`，请参阅 [在 App 的 target 内部导入代码](../03-Mix%20and%20Match/Swift%20and%20Objective-C%20in%20the%20Same%20Project.md#importing_code_from_within_the_same_app_target)。

* 取消原始的 Objective-C `.m`文件在`Target Membership`选择框中的勾选，从而将其从 target 中移除。不要立刻删除`.m`和`.h`文件，以备解决问题时使用。

* 如果为 Swift 类起了一个新的类名，请使用新的 Swift 类名代替原来的 Objective-C 类名。

<a name="troubleshooting_tips_and_reminders"></a>
## 故障排除贴士

尽管对于不同的项目，迁移过程是不尽相同的，但无论怎样，都有一些通用的方法能解决代码迁移时碰到的问题：

* 无法在 Objective-C 中继承 Swift 类。因此，被迁移的类不能有任何 Objective-C 子类。

* 迁移一个类到 Swift 时，必须从 target 中移除相关的`.m`文件，避免编译时出现符号重复的错误。

* 为了在 Objective-C 中可用，Swift 类必须是一个 Objective-C 类的子类，或者标记`@objc`。

* 在 Objective-C 中使用 Swift 代码时，记住 Objective-C 不能转化 Swift 的独有特性。详细列表请参阅 [在 Objective-C 中使用 Swift](../03-Mix%20and%20Match/Swift%20and%20Objective-C%20in%20the%20Same%20Project.md#using_swift_from_objective-c) 小节。

* 可以在 Objective-C 代码中`Commond + 单击`一个 Swift 类名来查看 Xcode 为它自动生成的头文件。

* 可以`Option + 单击`一个符号来查看更详细的信息，比如它的类型，属性以及文档注释等。
