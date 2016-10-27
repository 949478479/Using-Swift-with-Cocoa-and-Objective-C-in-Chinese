# 迁移 Objective-C 代码到 Swift

- [为 Objective-C 代码做好迁移准备](#preparing_your_objective-c_code_for_migration)
- [迁移过程](#the_migration_process)
	- [准备迁移](#before_you_start)
	- [开始迁移](#as_you_work)
	- [完成迁移](#after_you_finish)
- [故障排除贴士](#troubleshooting_tips_and_reminders)

迁移工作提供了一个重新审视现有的 Objective-C 应用程序的机会，通过用 Swift 代码替换 Objective-C 代码来改善程序的架构，逻辑以及性能。简而言之，通过之前学习的工具，即混搭和互用来对应用程序进行增量迁移。在决定哪些特性和功能用 Swift 实现，哪些依然用 Objective-C 实现时，混搭和互用会让这一切变得简单可行。通过这些工具，可以一步步探索 Swift 广泛的功能并整合到现有的 Objective-C 应用程序中，而不必立刻使用 Swift 重写整个应用程序。

<a name="preparing_your_objective-c_code_for_migration"></a>
## 为 Objective-C 代码做好迁移准备

在开始迁移代码之前，请确保 Objective-C 和 Swift 代码间有着最佳兼容性。这意味着你可能需要整理现有项目，并将 Objective-C 现代化特性应用其中。为了更好地与 Swift 无缝交互，现有的 Objective-C 代码需要遵循现代编码实践。在开始前，这有个简短的实践列表，请参阅 [*Adopting Mordern Objective-C*](https://developer.apple.com/library/prerelease/ios/releasenotes/ObjectiveC/ModernizationObjC/AdoptingModernObjective-C/AdoptingModernObjective-C.html#//apple_ref/doc/uid/TP40014150)。

<a name="the_migration_process"></a>
## 迁移过程

迁移代码到 Swift 的最有效的方式是逐个文件迁移，即一次迁移一个类。由于无法在 Objective-C 中继承 Swift 类，因此最好选择一个没有子类的类开始。使用单个`.swift`文件替换对应的`.m`和`.h`文件，实现和接口将直接放进这个 Swift 文件。你也不用创建头文件，Xcode 会在你需要引用头文件的时候自动生成头文件。

<a name="before_you_start"></a>
### 准备迁移

* 在 Xcode 中，选择`File > New > File > (iOS，watchOS，tvOS，macOS) > Source > Swift File`为对应的 Objective-C `.m`和`.h`文件创建一个 Swift 类。可以使用相同或者不同的类名。类前缀在 Swift 中不是必须的。

* 导入相关的系统框架。

* 如果需要在 Swift 文件中使用同一 target 中的 Objective-C 代码，可以填写一个 Objective-C 桥接头文件。具体的操作步骤，请参阅[在应用程序的 target 中导入代码](../03-Mix%20and%20Match/Swift%20and%20Objective-C%20in%20the%20Same%20Project.md#importing_code_from_within_the_same_app_target)小节。

* 为了让 Swift 类能在 Objective-C 中使用，Swift 类必须继承自 Objective-C 类。如果想为 Swift 类指定在 Objective-C 中的类名，可以使用`@objc(name)`特性，`name`就是 Swift 类在 Objective-C 中的类名。关于`@objc`的更多信息，请参阅[Swift 类型兼容性](../02-Interoperability/01-Interacting%20with%20Objective-C%20APIs.md#swift_type_compatibility)小节。

<a name="as_you_work"></a>
### 开始迁移

* 可以通过继承 Objective-C 类，采用 Objective-C 协议，或者其他方式，来让 Swift 类集成 Objective-C 特性。更多信息请参阅[使用 Objective-C 特性编写 Swift 类和协议](../02-Interoperability/02-Writing%20Swift%20Classes%20and%20Protocols%20with%20Objective-C%20Behavior.md)章节。

* 使用 Objective-C API 的时候，你需要知道 Swift 是怎样转化某些 Objective-C 语言特性的。更多信息请参阅[与 Objective-C API 交互](../02-Interoperability/01-Interacting%20with%20Objective-C%20APIs.md)章节。

* 使用 Cocoa 框架的代码时，记住某些类型已经被桥接，这意味着可以使用 Swift 类型去替代 Objective-C 类型。更多信息请参阅[使用 Cocoa 框架](../02-Interoperability/03-Working%20with%20Cocoa%20Frameworks.md)章节。

* 使用 Cocoa 设计模式的时候，请参阅[采用 Cocoa 设计模式](../02-Interoperability/04-Adopting%20Cocoa%20Design%20Patterns.md)章节。

* 对于如何将属性从 Objective-C 转换到 Swift，请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的[属性](http://wiki.jikexueyuan.com/project/swift/chapter2/10_Properties.html)章节。

* 在需要的时候，使用`@objc(name)`特性为 Swift 中的属性或方法提供 Objective-C 命名。例如，可以像下面这样将`enabled`属性的取值方法在 Objective-C 中的命名更改为`isEnabled`：

	```swift
	var enabled: Bool {
		@objc(isEnabled) get {
			// ...
		}
	}
	```

* 分别用`func`和`class func`来声明实例方法（`-`）和类方法（`+`）。

* 将简单宏声明为全局常量，将复杂宏声明为函数。

<a name="after_you_finish"></a>
### 完成迁移

* 将 Objective-C 代码中对应的导入语句更改为`#import "ProductModuleName-Swift.h"`，更多信息请参阅[在应用程序的 target 中导入代码](../03-Mix%20and%20Match/Swift%20and%20Objective-C%20in%20the%20Same%20Project.md#importing_code_from_within_the_same_app_target)小节。

* 将原始的 Objective-C `.m`文件在`Target Membership`选择框中的勾选取消，从而将其从 target 中移除。不要立刻删除`.m`和`.h`文件，以备解决问题时使用。

* 如果为 Swift 类起了一个新的类名，在相关代码中请使用新的 Swift 类名代替原来的 Objective-C 类名。

<a name="troubleshooting_tips_and_reminders"></a>
## 故障排除贴士

尽管对于不同的项目，迁移过程是不尽相同的，但仍有一些通用的办法能解决代码迁移时遇到的问题：

* 无法在 Objective-C 中继承 Swift 类。因此，被迁移的类不能有任何 Objective-C 子类。

* 迁移一个类到 Swift 时，必须从 target 中移除相关的`.m`文件，避免编译时出现符号重复的错误。

* 为了能在 Objective-C 中使用，Swift 类必须是一个 Objective-C 类的子类。

* 在将 Swift 代码导入 Objective-C 代码时，切记 Objective-C 不能转化某些 Swift 的独有特性。详细列表请参阅[在 Objective-C 中使用 Swift](../03-Mix%20and%20Match/Swift%20and%20Objective-C%20in%20the%20Same%20Project.md#using_swift_from_objective-c)小节。

* 可以在 Objective-C 代码中通过`Commond + 单击`一个 Swift 类名的方式来查看 Xcode 为它自动生成的头文件。

* 可以`Option + 单击`一个符号来查看它的详细信息，比如它的类型，特性以及文档注释等。
