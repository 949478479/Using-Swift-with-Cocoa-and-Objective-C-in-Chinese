# 在同一工程中使用 Swift 和 Objective-C

本页包含内容：

- [混合搭配概述](#mix_and_match_overview)
- [在 App 的 target 内部导入代码](#importing_code_from_within_the_same_app_target)
- [在 Framework 的 target 内部导入代码](#importing_code_from_within_the_same_framework_target)
- [导入外部 Framework](#importing_external_frameworks)
- [在 Objective-C 中使用 Swift](#using_swift_from_objective-c)
- [为 Objective-C 接口重写 Swift 名称](#overriding_swift_names_for_Objective-C_interfaces)
- [令 Objective-C 接口在 Swift 中不可用](#Making Objective-C Interfaces Unavailable in Swift)
- [精炼 Objective-C 声明](#Refining Objective-C Declarations)
- [命名产品模块](#naming_your_product_module)
- [故障排除贴士](#troubleshooting_tips_and_reminders)

由于 Swift 与 Objective-C 的兼容性，可以在同一工程中同时使用两种语言，开发基于混合语言的 App。利用这种特性，可以用 Swift 的最新语言特性实现 App 的部分功能，并无缝并入现有的 Objective-C 代码中。

<a name="mix_and_match_overview"></a>
## 混合搭配概述

Objective-C 和 Swift 文件可以在同一工程中并存，无论这个工程原本是基于 Objective-C 还是 Swift。还可以直接往现有工程中添加另一种语言的源文件。这种自然的工作流使得创建混合语言的 App 或 Framework 和单独使用一种语言时一样简单。

基于混合语言编写 App 或 Framework 时，二者稍微有些区别。下图展示了同时使用两种语言时，在 target 内部导入代码的基本原理，后续小节会介绍更多细节。

![](DAG_2x.png)

<a name="importing_code_from_within_the_same_app_target"></a>
## 在 App 的 target 内部导入代码

在编写基于混合语言的 App 时，可能需要用 Swift 代码访问 Objective-C 代码，或者反过来。下面描述的流程适用于非 Framework 的 target 。

### 将 Objective-C 代码导入到 Swift

在 App 的 target 内部导入一系列 Objective-C 文件供 Swift 代码使用时，需要依靠 Objective-C 桥接头文件将这些文件暴露给 Swift。添加 Swift 文件到现有的 Objective-C App 时（或者反过来），Xcode 会自动创建 Objective-C 桥接头文件。

![](bridgingheader_2x.png)

如果接受，Xcode 会随着源文件的创建生成 Objective-C 桥接头文件，并用产品模块名跟上`“-Bridging-Header.h”`作为 Objective-C 桥接头文件的文件名。关于产品模块名的具体介绍，请参阅 [命名产品模块](#naming_your_product_module) 小节。

或者，可以选择`File > New > File > （iOS，watchOS，tvOS，OS X） > Source > Header File`来手动创建 Objective-C 桥接头文件。

可以通过编辑这个 Objective-C 桥接头文件将 Objective-C API 暴露给 Swift。

##### 在 target 内部将 Objective-C 代码导入到 Swift

1. 在 Objective-C 桥接头文件中，导入希望暴露给 Swift 的 Objective-C 头文件。例如：

    ```objective-c
    #import "XYZCustomCell.h"
    #import "XYZCustomView.h"
    #import "XYZCustomViewController.h"
    ```

2. 确保在`Build Settings > Swfit Compiler - Code Generation > Objective-C Bridging Header`中设置了 Objective-C 桥接头文件的路径。该路径形式类似`Info.plist`在`Build Settings`中指定的路径，相对于工程，而不是相对于其所在目录。Xcode 自动生成 Objective-C 桥接头文件时会自动设置该路径，不需要额外修改。

在 Objective-C 桥接头文件中导入的所有 Objective-C 头文件都会暴露给 Swift。target 内部的所有 Swift 文件都可以使用这些 Objective-C 头文件中的 API，不需要任何导入语句，而且还能用 Swift 语法使用这些自定义的 Objective-C API，就像使用系统的 Swift 类一样。

```swift
let myCell = XYZCustomCell()
myCell.subtitle = "A custom cell"
```

### 将 Swift 代码导入到 Objective-C

将 Swift 代码导入到 Objective-C 时，需要依靠 Xcode 生成的头文件将这些文件暴漏给 Objective-C（此头文件不是上小节描述的 Objective-C 桥接头文件，该头文件在工程目录下不可见，但是可以跳转进去查看）。这个自动生成的头文件是一个 Objective-C 头文件，声明了 target 内部的一些 Swift API。可以将这个 Objective-C 头文件看作 Swift 代码的保护伞头文件。该头文件以产品模块名跟上`“-Swift.h”`来命名。关于产品模块名的具体介绍，请参阅 [命名产品模块](#naming_your_product_module) 小节。

默认情况下，这个头文件包含标记`public`修饰符的 Swift API。如果 target 内有一个 Objective-C 桥接头文件的话，它还会包含标记`internal`修饰符的 Swift API。标记`private`修饰符的 Swift API 不会出现在这个头文件中，因为私有声明不会暴露给 Objective-C，除非它们被显式标记`@IBAction`，`@IBOutlet`，或`@objc`。如果 App 的 target 启用了单元测试，单元测试的 target 在导入 App 的 target 时，编译器会在导入语句前加上`@testable`特性，从而可以在单元测试的 target 内访问 App 的 target 内任何标记`internal`修饰符的 API，犹如它们标记了`public`修饰符一般。

关于访问级别修饰符的更多信息，请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的 [访问控制](http://wiki.jikexueyuan.com/project/swift/chapter2/24_Access_Control.html) 章节。

这个头文件会由 Xcode 自动生成，可直接导入到 Objective-C 代码中使用。注意，如果这个头文件中的 Swift API 使用了自定义的 Objective-C 类型，确保在导入这个自动生成的头文件前，先将相关的自定义的 Objective-C 类型对应的 Objective-C 头文件导入。

##### 在 target 内部将 Swift 代码导入到 Objective-C

- 在 target 内部的 Objective-C `.m`文件中，用如下语法导入 Swift 代码：

```objective-c
#import "ProductModuleName-Swift.h" // 将 ProductModuleName 替换为具体的产品模块名
```

target 内部的一些 Swift API 会暴露给包含这个导入语句的 Objective-C `.m`文件。关于如何在 Objective-C 中使用 Swift，请参阅 [在 Objective-C 中使用 Swift](#using_swift_from_objective-c) 小节。

|              | 导入到 Swift | 导入到 Objective-C  |
| :-------------:|:-----------:|:------------:|
| Swift 代码    | 不需要导入语句  | #import "ProductModuleName-Swift.h”  |
| Objective-C 代码     | 不需要导入语句；需要 Objective-C 桥接头文件| #import "Header.h"     |

<a name="importing_code_from_within_the_same_framework_target"></a>
## 在 Framework 的 target 内部导入代码

在编写基于混合语言的 Framework 时，可能需要在 Swift 代码中访问 Objective-C 代码，或者反过来。

### 将 Objective-C 代码导入到 Swift

若要将 Objective-C 文件导入到 target 内部的 Swift 代码中，需要将这些 Objective-C 文件导入到 Objective-C 的保护伞头文件中。

##### 在 target 内部将 Objective-C 代码导入到 Swift

1. 确保将 target 的`Build Settings > Packaging > Defines Module`设置为`Yes`。

2. 在保护伞头文件中导入希望暴露给 Swift 的 Objective-C 头文件。例如：

```objective-c
#import <XYZ/XYZCustomCell.h>
#import <XYZ/XYZCustomView.h>
#import <XYZ/XYZCustomViewController.h>
```

保护伞头文件中导入的 Objective-C 头文件都会暴露给 Swift。target 内部的所有 Swift 文件都可以使用这些 Objective-C 头文件中的 API，不需要任何导入语句，而且还能用 Swift 语法使用这些自定义的 Objective-C API，就像使用系统的 Swift 类一样。

```swift
let myOtherCell = XYZCustomCell()
myOtherCell.subtitle = "Another custom cell"
```

### 将 Swift 代码导入到 Objective-C

若要将 Swift 文件导入到 target 内部的 Objective-C 代码中，不需要导入任何东西到保护伞头文件，而是将 Xcode 为 Swift 代码自动生成的头文件导入到要访问 Swift 代码的 Objective-C `.m`文件。

由于这个自动生成的头文件是 Framework 公共接口的一部分，因此只有标记`public`修饰符的 API 才会出现在这个自动生成的头文件中。不过，在 Framework 内部的 Objective-C 代码中，依旧可以使用标记`internal`的 Swift API，只要它们所在的类继承自 Objective-C 类。然而，使用这些标记`internal`的 Swift API 时，却发现编译器会报错提示符号未定义，研究了一下发现可以采取如下办法解决：

```swift
// Swift 代码部分
@objc(Foo) 
class Foo: NSObject {
    func foo() { print("我是 foo") }
}
```

```objective-c
// Objective-c 代码部分
// 为了解决符号未定义的错误，手动写个接口声明，Swift 代码中需要写 @objc(Foo)，否则类名不匹配
@interface Foo : NSObject
- (void)foo;
@end

[[Foo new] foo]; // 然后就可以正常调用了
```

实际上，在 Xcode 为 Swift 代码自动生成的头文件中，一个 public 访问级别的 API 会类似下面这样：

```objective-c
SWIFT_CLASS("_TtC15ProductModuleName3Foo")
@interface Foo : NSObject
- (void)foo;
- (nonnull instancetype)init OBJC_DESIGNATED_INITIALIZER;
@end
```

因此导入这个头文件也就获得了对应的 API 声明，而 internal 访问级别的 API 不会出现在此头文件中，因此需要手动声明一下，否则就会编译报错。

关于访问级别修饰符的更多信息，请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的 [访问控制](http://wiki.jikexueyuan.com/project/swift/chapter2/24_Access_Control.html) 章节。

##### 在 target 内部将 Swift 代码导入到 Objective-C

1. 确保将 target 的`Build Settings > Packaging > Defines Module`设置为`Yes`。

2. 使用如下语法将 Swift 代码导入到 target 内部的 Objective-C `.m`文件：

```objective-c
#import <ProductName/ProductModuleName-Swift.h>
```

target 内部的一些 Swift API 会暴露给包含这个导入语句的 Objective-C `.m`文件。关于如何在 Objective-C 中使用 Swift，请参阅 [在 Objective-C 中使用 Swift](#using_swift_from_objective-c) 小节。

|              | 导入到 Swift | 导入到 Objective-C  |
| :-------------:|:-----------:|:------------:| 
| Swift 代码    | 不需要导入语句  | #import "ProductName/ProductModuleName-Swift.h"  |
| Objective-C 代码     | 不需要导入语句；需要 Objective-C umbrella header  | #import "Header.h"     |


<a name="importing_external_frameworks"></a>
## 导入外部 Framework

可以导入外部的 Framework，无论它是基于 Objective-C，Swift，还是混合语言的，而且导入流程都是一样的，只需确保`Build Setting > Pakaging > Defines Module`设置为`Yes`。

用如下语法将外部 Framework 导入到 Swift 文件：

```swift
import FrameworkName
```

用如下语法将外部 Framework 导入到 Objective-C `.m`文件：

```objective-c
@import FrameworkName;
```

|           | 导入到 Swift | 导入到 Objective-C  |
| :-------------:|:-----------:|:------------:| 
|任意语言框架 | import FrameworkName | @import FrameworkName; |

<a name="using_swift_from_objective-c"></a>
## 在 Objective-C 中使用 Swift

将 Swift 代码导入到 Objective-C 后，便可用正规的 Objective-C 语法使用 Swift 类。

```objective-c
MySwiftClass *swiftObject = [[MySwiftClass alloc] init];
[swiftObject swiftMethod];
```

Swift 的类或协议必须标记`@objc`特性，才能在 Objective-C 中使用。这个特性向编译器表明对应的 Swift 代码可以在 Objective-C 中使用。如果 Swift 类是 Objective-C 类的子类，编译器会自动添加`@objc`特性。详情请参阅 [Swift 类型兼容性](../02-Interoperability/01-Interacting%20with%20Objective-C%20APIs.md#swift_type_compatibility)。

可以访问在 Swift 类或协议中用`@objc`特性标记的任何东西，只要它兼容 Objective-C，这意味着以下的 Swift 独有特性不包括在内：

- 泛型
- 元组
- 原始值类型不是`Int`的 Swift 枚举
- Swift 结构体
- Swift 全局变量
- Swift 类型别名
- Swift 可变参数
- 嵌套类型
- 柯里化函数

例如，使用泛型类型作为参数，或者返回元组的方法将不能在 Objective-C 中使用。

> 注意  
> 不能在 Objective-C 中继承一个 Swift 类。

### 在 Objective-C 头文件中引用 Swift 类或协议

为避免循环引用，不要将 Swift 代码导入到 Objective-C 头文件（`.h`文件），而应该使用前向声明来引用一个 Swift 类或者协议。

```objective-c
// MyObjcClass.h
@class MySwiftClass;
@protocol MySwiftProtocol;
     
@interface MyObjcClass : NSObject
- (MySwiftClass *)returnSwiftClassInstance;
- (id <MySwiftProtocol>)returnInstanceAdoptingSwiftProtocol;
// ...
@end
```

Swift 类和协议的前向声明只能用于声明方法和属性。

### 在 Objective-C 的实现文件中采纳 Swift 协议

Objective-C 类可以在实现文件中导入 Xcode 自动生成的头文件，然后使用类扩展来采纳 Swift 协议。

```objective-c
// MyObjcClass.m
#import "ProductModuleName-Swift.h"
     
@interface MyObjcClass () <MySwiftProtocol>
// ...
@end
     
@implementation MyObjcClass
// ...
@end
```

<a name="overriding_swift_names_for_Objective-C_interfaces"></a>
## 为 Objective-C 接口重写 Swift 名称

Swift 编译器自动将 Objective-C 代码作为常规 Swift 代码导入，它将 Objective-C 类工厂方法导入为 Swift 构造器，还会缩短 Objective-C 枚举类型的名称。

代码中也许会存在无法自动处理的边界情况。对于 Objective-C 方法，枚举值，或者选项集合的值，如果需要更改它们导入到 Swift 后的名称，可以使用`NS_SWIFT_NAME`宏来自定义名称。

### 类工厂方法

如果 Swift 编译器无法识别类工厂方法，可以使用`NS_SWIFT_NAME`宏来指定构造器在 Swift 中的方法名，从而正确导入。例如：

```objective-c
+ (instancetype)recordWithRPM:(NSUInteger)RPM NS_SWIFT_NAME(init(RPM:));
```

如果 Swift 编译器错误地将一个方法识别为类工厂方法，可以使用`NS_SWIFT_NAME`宏来指定类方法在 Swift 中的方法名，从而正确导入。例如：

```objective-c
+ (id)recordWithQuality:(double)quality NS_SWIFT_NAME(record(quality:));
```

### 枚举

默认情况下，Swift 导入枚举时，会将枚举值的名称前缀截断。如果要自定义枚举值的名称，可以使用`NS_SWIFT_NAME`宏来指定枚举值在 Swift 中的名称。例如：

```objective-c
typedef NS_ENUM(NSInteger, ABCRecordSide) {
    ABCRecordSideA,
    ABCRecordSideB NS_SWIFT_NAME("FlipSide"),
};
```

<a name="Making Objective-C Interfaces Unavailable in Swift"></a>
## 令 Objective-C 接口在 Swift 中不可用

一些 Objective-C 接口可能不适合或者没必要暴露给 Swift，为了防止 Objective-C 接口导入到 Swift ，可以使用`NS_SWIFT_UNAVAILABLE`宏来传达一个提示信息，从而指引 API 使用者使用其他替代方式。

例如，一个 Objective-C 类提供了一个接收一些键值对作为可变参数的便利构造器，此时可以建议 Swift 用户使用字典字面量作为替代：

```objective-c
+ (instancetype)collectionWithValues:(NSArray *)values 
                             forKeys:(NSArray<NSCopying> *)keys NS_SWIFT_UNAVAILABLE("Use a dictionary literal instead");
```

试图在 Swift 中调用`++collectionWithValues:forKeys:`方法将导致一个编译错误。

<a name="Refining Objective-C Declarations"></a>
## 精炼 Objective-C 声明

可以使用`NS_REFINED_FOR_SWIFT`宏标记 Objective-C 方法的声明，然后在 Swift 中通过扩展提供一个精炼的 Swift 接口，并通过该接口去调用方法的原始实现。例如，接收一个或者多个指针参数的 Objective-C 方法可以精炼为一个返回元组值的 Swift 方法。使用该宏的方法导入到 Swift 后会做如下处理：

- 对于初始化方法，在第一个外部参数名前加双下划线（`__`）。
- 对于对象下标方法，在方法名前加双下划线（`__`），作为普通方法，而不是 Swift 下标方法。
- 对于其他方法，在方法名前加双下划线（`__`）。

思考如下 Objective-C 声明：

```objective-c
@interface Color : NSObject

- (void)getRed:(nullable CGFloat *)red
         green:(nullable CGFloat *)green
          blue:(nullable CGFloat *)blue
         alpha:(nullable CGFloat *)alpha NS_REFINED_FOR_SWIFT;
         
@end
```

可以通过 Swift 扩展来提供一个精炼后的接口：

```swift
extension Color {
    var RGBA: (red: CGFloat, green: CGFloat, blue: CGFloat, alpha: CGFloat) {
        var r: CGFloat = 0.0
        var g: CGFloat = 0.0
        var b: CGFloat = 0.0
        var a: CGFloat = 0.0
        __getRed(&r, green: &g, blue: &b, alpha: &a)
        return (red: r, green: g, blue: b, alpha: a)
    }
}
```

<a name="naming_your_product_module"></a>
## 命名产品模块

无论是 Xcode 为 Swift 代码自动生成的头文件，还是 Objective-C 桥接头文件，都会根据产品模块名来命名。默认情况下，产品模块名和产品名一样。然而，如果产品名包含非字母、非数字的特殊字符，例如点号（`.`），作为产品模块名时，这些特殊字符会被下划线（`_`）替换。如果产品名以数字开头，作为产品模块名时，第一个数字也会被下划线替换。

也可以为产品模块名提供一个自定义的名称，Xcode 会根据这个名称来命名桥接头文件和自动生成的头文件，只需修改`Build setting > Packaging > Product Module Name`中的名称即可。

> 注意  
> 无法改变 Framework 的产品模块名。

<a name="troubleshooting_tips_and_reminders"></a>
## 故障排除贴士

- 将 Swift 和 Objective-C 文件看作同一代码集合，注意命名冲突。

- 如果使用 Framework，确保`Build setting > Packaging > Defines Module`设置为`Yes`。

- 如果使用 Objective-C 桥接头文件，确保`Build setting > Swift Compiler > Code Generation`中的头文件路径是头文件自身相对于工程的路径。

- Xcode 根据产品模块名，而不是 target 的名称来命名 Objective-C 桥接头文件以及为 Swift 代码自动生成的头文件。详情请参阅 [命名产品模块](#naming_your_product_module)。

- 只有继承自 Objective-C 类的 Swift 类，以及标记了`@objc`的 Swift API，才能在 Objective-C 中使用。

- 将 Swift 代码导入到 Objective-C 时，注意 Objective-C 无法转化 Swift 的独有特性。详细列表请参阅 [在 Objective-C 中使用 Swift](#using_swift_from_objective-c)。

- 如果在 Swift 代码中使用了自定义的 Objective-C 类型，在 Objective-C 中使用这部分 Swift 代码时，确保先导入相关的 Objective-C 类型的头文件，然后再将 Xcode 为 Swift 代码自动生成的头文件导入。

- 标记`private`修饰符的 Swift 声明不会出现在自动生成的头文件中，因为私有声明不会暴露给 Objective-C，除非它们被显式标记`@IBAction`，`@IBOutlet`或者`@objc`。

- 对于 App 的 target，当存在 Objective-C 桥接头文件时，标记`internal`修饰符的声明也会出现在自动生成的头文件中。

- 对于 Framework 的 target，只有标记`public`修饰符的声明才会出现在自动生成的头文件中，不过依然可以在 Framework 内部的 Objective-C 代码中使用标记`internal`修饰符的 Swift API，只要它们所在的类继承自 Objective-C 类。关于访问级别修饰符的更多信息，请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的 [访问控制](http://wiki.jikexueyuan.com/project/swift/chapter2/24_Access_Control.html) 章节。
