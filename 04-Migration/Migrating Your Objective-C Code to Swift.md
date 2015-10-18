> 翻译：[xudeheng](https://github.com/xudeheng)

> 校对：[ChildhoodAndy](http://github.com/dabing1022)

# 将 Objective-C 代码迁移到 Swift

本节内容包括：

- [为你的 Objective-c 代码做好迁移准备（Preparing Your Objective-C Code for Migration）](#preparing_your_objective-c_code_for_migration)

- [迁移步骤（The Migration Process）](#the_migration_process)

- [故障排除贴士（Troubleshooting Tips and Reminders）](#troubleshooting_tips_and_reminders)

迁移工作提供了一个重新审视现有 Objective-C 应用程序的机会，并通过替换部分 Swift 代码来改善程序的架构，逻辑以及性能。简单讲，迁移就是让你使用之前学习的工具--混合搭配（mix and match）以及互用性（interoperability）。当要选择哪些特性和功能用 Swift 实现，哪些依然用 Objective-C 实现时，混合搭配会让这一切变得简单。由于互用性，将 Swift 的特性整合到 Objective-C 也并不困难。通过这些工具可以一步步探索 Swift 的大量功能并整合到现有的 Objective-C 项目中而不必立刻使用 Swift 重写整个项目。

<a name="preparing_your_objective-c_code_for_migration"></a>
## 为你的 Objective-c 代码做好迁移准备（Preparing Your Objective-C Code for Migration）

在开始迁移你的代码之前，请确保你的 Objective-C 和 Swift 代码间有着最佳兼容性。这意味着整理并使用 Objective-C 的现代化特性来优化你的现有项目。为了更易于和 Swift 无缝交互，你的现有代码需要遵循现代编码实践。在开始前，这有个简短的实践列表，参看 [Adopting Mordern Objective-C](https://developer.apple.com/library/prerelease/ios/releasenotes/ObjectiveC/ModernizationObjC/AdoptingModernObjective-C/AdoptingModernObjective-C.html#//apple_ref/doc/uid/TP40014150)。

<a name="the_migration_process"></a>
## 迁移步骤（The Migration Process）

最有效的迁移代码的方式是基于逐个文件，即一次完成一个类。由于你不能在 Objective-C 中继承 Swift 类，最好选择一个没有子类的。你将用单个`.swift`文件替换对应的`.m`和`.h`文件，实现和接口将直接放进这个 Swift 文件。你也不用再创建头文件了，Xcode 会在你需要引用的时候自动生成头文件。

### 准备工作（Before You Start）

* 在 Xcode 中，选择`File > New > File > (iOS 或者 OS X) > Source > Swift File`为对应的 Objective-C `.m`和`.h`文件创建一个 Swift 类。你可以使用相同或者不同的类名，类前缀在 Swift 中不是必须的。

* 导入相关系统框架。

* 如果你需要在 Swift 文件中访问 Objective-C 代码的话，可以填写一个 Objective-C 桥接头文件。具体的操作步骤，请看 [在同一应用的 target 中导入代码（Importing Code from Within the Same App Target）](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/03Mix%20and%20Match/Swift%20and%20Objective-C%20in%20the%20Same%20Project.md#%E5%9C%A8%E5%90%8C%E4%B8%80%E5%BA%94%E7%94%A8%E7%9A%84-target-%E4%B8%AD%E5%AF%BC%E5%85%A5%E4%BB%A3%E7%A0%81importing-code-from-within-the-same-app-target)。

* 为了使你的 Swift 类能在 Objective-C 中使用，可以继承 Objective-C 类，或者标记上`@objc`属性。如果想为类指定在 Objective-C 中使用的特殊名称，则使用`@objc(name)`标记，`name`就是在 Objective-C 中引用的 Swift 类名。 关于`@objc`的更多信息，请看 [Swift 类型兼容性（Swift Type Compatibility）](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/01Interacting%20with%20Objective-C%20APIs.md#swift_type_compatibility)。

### 开始工作（As You Work）

* 你可以通过继承 Objective-C 类，采用 Objective-C 协议，或者更多的方式，来让 Swift 类集成  Objective-C 行为。更多信息，请看 [使用 Objective-C 特性编写 Swift 类（Writing Swift Classes with Objective-C Behavior）](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/02Writing%20Swift%20Classes%20with%20Objective-C%20Behavior.md)。

* 你使用 Objective-C API 的时候，你需要知道 Swift 是怎样来翻译某些 Objective-C 特性的。更多信息，请看 [与 Objective-C 的 API 交互（Interacting with Objective-C APIs）](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/01Interacting%20with%20Objective-C%20APIs.md)。

* 使用 Cocoa 框架的代码时，记住某些类型是可以被桥接的，这意味着你可以使用某些 Swift 类型来替代某些 Objective-C 类型。更多信息，请看 [与 Cocoa 数据类型共舞（Working with Cocoa Data Types）](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/03Working%20with%20Cocoa%20Data%20Types.md)。

* 使用 Cocoa 设计模式的时候，请看 [采用 Cocoa 设计模式（Adopting Cocoa Design Patterns）](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/02Interoperability/04Adopting%20Cocoa%20Design%20Patterns.md)来获取更多信息。

* 对于将属性从 Objective-C 转换到 Swift，请看[《The Swift Programming Language 中文版》](http://wiki.jikexueyuan.com/project/swift/)中 [属性（Properties）](http://wiki.jikexueyuan.com/project/swift/chapter2/10_Properties.html)部分。

* 在必要的时候，通过`@objc(name)`属性为 Swift 中的属性或方法提供 Objective-C 名称，就像这样：

```swift
var enabled: Bool {
	@objc(isEnabled) get {
		// ...
	}
}
```

* 分别用`func`和`class func`来表示实例方法（-）和类（+）方法。
* 将简单宏转换为常量，将复杂宏转换为函数。

### 大功告成（After You Finish）

* 在你的 Objective-C 代码中更新导入语句为`#import "ProductModuleName-Swift.h"`，可参阅 [在同一应用的 target 中导入代码（Importing Code from Within the Same App Target）](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/03Mix%20and%20Match/Swift%20and%20Objective-C%20in%20the%20Same%20Project.md#importing_code_from_within_the_same_app_target)。

* 取消原始的 Objective-C `.m`文件在`Target Membership`选择框中的勾选从而将其从 target 中移除。不要立刻删除`.m`和`.h`文件，以备解决问题用。

* 如果你给 Swift 类起了一个新名字，请使用新的 Swift 类名代替原来的 Objective-C 类名。

<a name="troubleshooting_tips_and_reminders"></a>
## 故障排除贴士（Troubleshooting Tips and Reminders）

尽管对于不同的项目，迁移的经历是不尽相同的，无论怎样，都有一些通用的步骤和工具能帮你解决代码迁移时碰到的问题：

* 你不能在 Objective-C 中继承 Swift 类。因此，被你迁移的类不能有任何的 Objective-C 子类存在于你的应用中。

* 当你迁移一个类到 Swift 的时候，你必须从 target 中移除相关的`.m`文件，以避免编译时出现符号重复的错误。

* 为了在 Objective-C 中可用，Swift 类必须是一个 Objective-C 类的子类，或者被标记为`@objc`。

* 当你在 Objective-C 中使用 Swift 代码的时候，记住 Objective-C 不能转化 Swift 的某些特性，这有个列表 [在 Objective-C 中使用 Swift（Using Swift from Objective-C）](https://github.com/949478479/Using-Swift-with-Cocoa-and-Objective-C/blob/master/03Mix%20and%20Match/Swift%20and%20Objective-C%20in%20the%20Same%20Project.md#using_swift_from_objective-c)。

* 可以通过`Commond + 单击`一个 Swift 类名来查看它生成的头文件。

* 可以通过`Option + 单击`一个符号来查看更详细的信息，比如它的类型，属性以及文档注释等。
