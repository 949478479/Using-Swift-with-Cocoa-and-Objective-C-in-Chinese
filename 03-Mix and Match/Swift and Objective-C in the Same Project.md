# 在项目中同时使用 Swift 和 Objective-C

- [混搭概述](#mix_and_match_overview)
- [在应用程序 target 中导入代码](#importing_code_from_within_the_same_app_target)
    - [将 Objective-C 代码导入到 Swift](#importing_objective_c_into_swift)
    - [将 Swift 代码导入到 Objective-C](#importing_swift_into_objective_c)
- [在框架 target 中导入代码](#importing_code_from_within_the_same_framework_target)
    - [将 Objective-C 代码导入到 Swift](#importing_objective_c_into_swift_2)
    - [将 Swift 代码导入到 Objective-C](#importing_swift_into_objective_c_2)
- [导入外部框架](#importing_external_frameworks)
- [在 Objective-C 中使用 Swift](#using_swift_from_objective-c)
    - [在 Objective-C 头文件中引用 Swift 类或协议](#referencing_a_swift_class_or_protocol_in_an_objective_c_header)
    - [声明可被 Objective-C 类遵守的 Swift 协议](#declaring_a_swift_protocol_that_can_be_adopted_by_an_objective_c_class)
    - [在 Objective-C 的实现文件中采用 Swift 协议](#adopting_a_swift_protocol_in_an_objective_c_implementation)
    - [声明可在 Objective-C 中使用的 Swift 错误类型](#declaring_a_swift_error_type_that_can_be_used_from_objective_c)
- [为 Objective-C 接口提供 Swift 命名](#overriding_swift_names_for_objective_c_interfaces)
    - [类工厂方法](#class_factory_methods)
    - [枚举](#enumerations) 
- [让 Objective-C 接口在 Swift 中不可用](#making_objective_c_interfaces_unavailable_in_swift)
- [优化 Objective-C 声明](#refining_objective_c_declarations)
- [为产品模块命名](#naming_your_product_module)
- [故障排除贴士](#troubleshooting_tips_and_reminders)

由于 Swift 与 Objective-C 的兼容性，你可以在项目中同时使用两种语言，开发基于混合语言的应用程序。利用这种特性，你可以用 Swift 的最新语言特性实现应用程序的部分功能，并无缝并入现有的 Objective-C 代码中。

<a name="mix_and_match_overview"></a>
## 混搭概述

Objective-C 和 Swift 文件可以在项目中并存，无论这个项目原本是基于 Objective-C 还是 Swift。还可以直接往现有项目中添加另一种语言的源文件。这种自然的工作流程使得创建基于混合语言的应用程序或框架变得和单独使用一种语言一样简单。

基于混合语言编写应用程序或框架的过程还是有些区别的。下图展示了同时使用两种语言时，在同一 target 中导入代码的基本原理，后续小节会介绍更多细节。

![](DAG_2x.png)

<a name="importing_code_from_within_the_same_app_target"></a>
## 在应用程序 target 中导入代码

在编写基于混合语言的应用程序时，可能需要在 Swift 代码中使用 Objective-C 代码，或者反过来。下面描述的流程适用于非框架类型的 target 。

<a name="importing_objective_c_into_swift"></a>
### 将 Objective-C 代码导入到 Swift

在应用程序 target 中导入一系列 Objective-C 文件供 Swift 代码使用时，需要依靠 Objective-C 桥接头文件将这些文件暴露给 Swift。添加 Swift 文件到现有的 Objective-C 项目时（或者反过来），Xcode 会自动创建 Objective-C 桥接头文件。

![](bridgingheader_2x.png)

如果选择创建，Xcode 会随着源文件的创建生成 Objective-C 桥接头文件，并用产品模块名拼接上`"-Bridging-Header.h"`作为 Objective-C 桥接头文件的文件名。关于产品模块名的具体介绍，请参阅[为产品模块命名](#naming_your_product_module)小节。

或者，可以选择`File > New > File > （iOS，watchOS，tvOS，macOS） > Source > Header File`来手动创建 Objective-C 桥接头文件。

你可以编辑这个 Objective-C 桥接头文件将 Objective-C API 暴露给 Swift。

##### 在 target 中将 Objective-C 代码导入到 Swift

1. 在 Objective-C 桥接头文件中，导入希望暴露给 Swift 的 Objective-C 头文件。例如：

    ```objective-c
    #import "XYZCustomCell.h"
    #import "XYZCustomView.h"
    #import "XYZCustomViewController.h"
    ```

2. 确保在`Build Settings > Swfit Compiler - Code Generation > Objective-C Bridging Header`中设置了 Objective-C 桥接头文件的路径。该路径相对于项目，类似`Info.plist`在`Build Settings`中指定的路径。Xcode 自动生成 Objective-C 桥接头文件时会自动设置该路径，因此大多数情况下你不需要专门去设置它。

在 Objective-C 桥接头文件中导入的所有 Objective-C 头文件都会暴露给 Swift。target 中所有 Swift 文件都可以使用这些 Objective-C 头文件中的内容，不需要任何导入语句。不但如此，你还能用 Swift 语法使用这些自定义的 Objective-C 代码，就像使用系统的 Swift 类一样。

```swift
let myCell = XYZCustomCell()
myCell.subtitle = "A custom cell"
```

<a name="importing_swift_into_objective_c"></a>
### 将 Swift 代码导入到 Objective-C

将 Swift 代码导入到 Objective-C 时，需要依靠 Xcode 自动生成的头文件将这些 Swift 文件暴露给 Objective-C（此头文件不是上小节描述的 Objective-C 桥接头文件，该头文件在项目目录下不可见，但是可以跳转进去查看）。这个自动生成的头文件是一个 Objective-C 头文件，声明了 target 中的一些 Swift 接口。可以将这个 Objective-C 头文件看作 Swift 代码的保护伞头文件。该头文件以产品模块名拼接上`"-Swift.h"`来命名。关于产品模块名的具体介绍，请参阅[为产品模块命名](#naming_your_product_module)小节。

默认情况下，这个头文件包含所有标记`public`或`open`修饰符的 Swift 声明。如果 target 中有 Objective-C 桥接头文件的话，它还会包含标记`internal`修饰符的 Swift 声明。标记`private`或`fileprivate`修饰符的 Swift 声明不会出现在这个头文件中。私有声明不会暴露给 Objective-C，除非它们被标记`@IBAction`，`@IBOutlet`，`@objc`。如果应用程序的 target 启用了单元测试，在单元测试的 target 中导入应用程序的 target 时，在`import`语句前加上`@testable`特性，就可以在单元测试的 target 中访问应用程序的 target 中任何标记`internal`修饰符的声明，犹如它们标记了`public`修饰符一般。

关于访问级别修饰符的更多信息，请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的[访问控制](http://wiki.jikexueyuan.com/project/swift/chapter2/24_Access_Control.html)章节。

这个头文件会由 Xcode 自动生成，可直接导入到 Objective-C 代码中使用。注意，如果这个头文件中的 Swift 接口使用了自定义的 Objective-C 类型，在导入这个自动生成的头文件前，必须先将相关的自定义 Objective-C 类型对应的 Objective-C 头文件导入。

##### 在 target 中将 Swift 代码导入到 Objective-C

- 在 target 中的 Objective-C `.m`文件中，用如下语法导入 Swift 代码：

```objective-c
#import "ProductModuleName-Swift.h" // 将 ProductModuleName 替换为产品模块名
```

target 中的一些 Swift 声明会暴露给包含这个导入语句的 Objective-C `.m`文件。关于如何在 Objective-C 中使用 Swift，请参阅[在 Objective-C 中使用 Swift](#using_swift_from_objective-c)小节。

|              | 导入到 Swift | 导入到 Objective-C  |
| :-------------:|:-----------:|:------------:|
| Swift 代码 | 不需要导入语句 | #import "ProductModuleName-Swift.h" |
| Objective-C 代码 | 不需要导入语句；需要 Objective-C 桥接头文件 | #import "Header.h" |

<a name="importing_code_from_within_the_same_framework_target"></a>
## 在框架 target 中导入代码

在编写基于混合语言的框架时，往往需要在 Swift 代码中访问 Objective-C 代码，或者反过来。

<a name="importing_objective_c_into_swift_2"></a>
### 将 Objective-C 代码导入到 Swift

若要将 Objective-C 文件导入到 target 中的 Swift 代码中，需要将这些 Objective-C 文件导入到 Objective-C 的保护伞头文件中。

> 译者注  
> 此处的“保护伞头文件”指的是带有框架版本号和版本字符串声明的那个头文件。

##### 在 target 中将 Objective-C 代码导入到 Swift

1. 确保将 target 的`Build Settings > Packaging > Defines Module`设置为`Yes`。

2. 在保护伞头文件中导入希望暴露给 Swift 的 Objective-C 头文件。例如：

```objective-c
#import <XYZ/XYZCustomCell.h>
#import <XYZ/XYZCustomView.h>
#import <XYZ/XYZCustomViewController.h>
```

保护伞头文件中导入的 Objective-C 头文件都会暴露给 Swift。target 中所有 Swift 文件都可以使用这些 Objective-C 头文件中的内容，不需要任何导入语句。不但如此，你还能用 Swift 语法使用这些自定义的 Objective-C 代码，就像使用系统的 Swift 类一样。

```swift
let myOtherCell = XYZCustomCell()
myOtherCell.subtitle = "Another custom cell"
```

<a name="importing_swift_into_objective_c_2"></a>
### 将 Swift 代码导入到 Objective-C

若要将 Swift 文件导入到 target 中的 Objective-C 代码中，不需要导入任何东西到保护伞头文件，只需将 Xcode 为 Swift 代码自动生成的头文件导入到要使用 Swift 代码的 Objective-C `.m`文件。

由于这个自动生成的头文件是框架公共接口的一部分，因此只有标记`public`或`open`修饰符的 Swift 声明才会出现在这个自动生成的头文件中。

对于继承自 Objective-C 类的 Swift 子类中标记`internal`修饰符的方法和属性，它们只会暴露给 Objective-C 运行时系统，这意味着它们不会出现在 Swift 代码的头文件中，也无法在编译期访问它们。

> 译者注  
> 使用这些标记`internal`的 Swift 声明时，编译器会报错提示符号未定义，研究了一下发现可以采取如下方式解决：
>
```swift
// Swift 代码部分
@objc(Foo) 
class Foo: NSObject {
    func hello() { print("Hello!") }
}
```
>  
```objective-c
// Objective-C 代码部分
// 为了解决符号未定义的错误，手动写个接口声明，注意 Swift 代码中的 @objc(Foo)，否则会导致类名不匹配
@interface Foo : NSObject
- (void)hello;
@end
```

关于访问级别修饰符的更多信息，请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的[访问控制](http://wiki.jikexueyuan.com/project/swift/chapter2/24_Access_Control.html)章节。

##### 在 target 中将 Swift 代码导入到 Objective-C

1. 确保将 target 的`Build Settings > Packaging > Defines Module`设置为`Yes`。

2. 使用如下语法将 Swift 代码导入到 target 中的 Objective-C `.m`文件：

```objective-c
// 分别用产品名和产品模块名替换 ProductName 和 ProductModuleName
#import <ProductName/ProductModuleName-Swift.h> 
```

target 中的一些 Swift 声明会暴露给包含这个导入语句的 Objective-C `.m`文件。关于如何在 Objective-C 中使用 Swift，请参阅[在 Objective-C 中使用 Swift](#using_swift_from_objective-c)小节。

|              | 导入到 Swift | 导入到 Objective-C |
| :-------------:|:-----------:|:------------:| 
| Swift 代码 | 不需要导入语句 | #import <ProductName/ProductModuleName-Swift.h> |
| Objective-C 代码 | 不需要导入语句；需要 Objective-C 保护伞头文件 | #import "Header.h" |


<a name="importing_external_frameworks"></a>
## 导入外部框架

你可以导入位于其他 target 中的外部框架，无论它是基于 Objective-C，Swift，还是基于混合语言的。导入流程都是一样的，只需确保被导入框架的`Build Setting > Pakaging > Defines Module`设置为`Yes`。

使用如下语法将外部框架导入到 Swift 文件：

```swift
import FrameworkName
```

使用如下语法将外部框架导入到 Objective-C `.m`文件：

```objective-c
@import FrameworkName;
```

|           | 导入到 Swift | 导入到 Objective-C  |
| :-------------:|:-----------:|:------------:| 
| 任意语言框架 | import FrameworkName | @import FrameworkName; |

<a name="using_swift_from_objective-c"></a>
## 在 Objective-C 中使用 Swift

将 Swift 代码导入到 Objective-C 后，就可用标准 Objective-C 语法使用 Swift 类。

```objective-c
MySwiftClass *swiftObject = [[MySwiftClass alloc] init];
[swiftObject swiftMethod];
```

只有继承自 Objective-C 类的 Swift 类才能在 Objective-C 中使用。想要了解 Swift 如何将接口导入到 Objective-C 以及在 Objective-C 中能使用哪些 Swift 特性，请参阅[Swift 类型兼容性](../02-Interoperability/01-Interacting%20with%20Objective-C%20APIs.md#swift_type_compatibility)小节。

<a name="referencing_a_swift_class_or_protocol_in_an_objective_c_header"></a>
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

<a name="declaring_a_swift_protocol_that_can_be_adopted_by_an_objective_c_class"></a>
### 声明可被 Objective-C 类遵守的 Swift 协议

将 Swift 协议标记`@objc`特性，从而让 Objective-C 类可以遵守该协议。

```swift
@objc public protocol MySwiftProtocol {
    func requiredMethod()

    @objc optional func optionalMethod()
}
```

为了遵守协议，Objective-C 类必须实现协议中声明的所有构造器，属性，下标，方法。可选的协议要求必须标记`@objc`特性和`optional`修饰符。

<a name="adopting_a_swift_protocol_in_an_objective_c_implementation"></a>
### 在 Objective-C 的实现文件中采用 Swift 协议

Objective-C 类可以在实现文件（`.m`文件）中导入 Xcode 自动生成的 Swift 头文件，然后使用类扩展来采用 Swift 协议。

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

<a name="declaring_a_swift_error_type_that_can_be_used_from_objective_c"></a>
### 声明可在 Objective-C 中使用的 Swift 错误类型

遵守`Error`协议并标记`@objc`特性的 Swift 枚举会在 Swift 头文件中生成一个`NS_ENUM`枚举声明，并会为错误域生成相应的`NSString`字符串常量。例如，有如下 Swift 枚举声明：

```swift
@objc public enum CustomError: Int, Error {
    case a, b, c
}
```

Swift 头文件中相应的 Objective-C 声明如下所示：

```objective-c
// Project-Swift.h
typedef SWIFT_ENUM(NSInteger, CustomError) {
    CustomErrorA = 0,
    CustomErrorB = 1,
    CustomErrorC = 2,
};
static NSString * const CustomErrorDomain = @"Project.CustomError";
```

<a name="overriding_swift_names_for_objective_c_interfaces"></a>
## 为 Objective-C 接口提供 Swift 命名

Swift 编译器自动将 Objective-C 代码作为常规 Swift 代码导入，它将 Objective-C 类工厂方法导入为 Swift 构造器，还会缩短 Objective-C 枚举值的命名。

代码中也许会存在无法自动处理的特殊情况。对于 Objective-C 方法，枚举值，或者选项集的值，可以使用`NS_SWIFT_NAME`宏来自定义它们导入到 Swift 后的命名。

<a name="class_factory_methods"></a>
### 类工厂方法

如果 Swift 编译器无法识别类工厂方法，可以使用`NS_SWIFT_NAME`宏来指定类工厂方法导入为 Swift 构造器后的方法名，从而能够将其正确导入。例如：

```objective-c
+ (instancetype)recordWithRPM:(NSUInteger)RPM NS_SWIFT_NAME(init(RPM:));
```

如果 Swift 编译器错误地将一个普通的类方法识别为类工厂方法，可以使用`NS_SWIFT_NAME`宏来指定类方法导入为 Swift 类方法后的方法名，从而能够将其正确导入。例如：

```objective-c
+ (id)recordWithQuality:(double)quality NS_SWIFT_NAME(record(quality:));
```

<a name="enumerations"></a>
### 枚举

默认情况下，Swift 导入枚举时，会将枚举值的名称前缀截断。如果需要自定义枚举值的名称，可以使用`NS_SWIFT_NAME`宏来指定枚举值导入到 Swift 后的命名。例如：

```objective-c
typedef NS_ENUM(NSInteger, ABCRecordSide) {
    ABCRecordSideA,
    ABCRecordSideB NS_SWIFT_NAME(FlipSide),
};
```

<a name="making_objective_c_interfaces_unavailable_in_swift"></a>
## 让 Objective-C 接口在 Swift 中不可用

一些 Objective-C 接口可能不适合或者没必要暴露给 Swift，为了防止 Objective-C 接口导入到 Swift，可以使用`NS_SWIFT_UNAVAILABLE`宏来传达一个提示信息，从而指引 API 使用者使用其他替代方式。

例如，一个 Objective-C 类提供了一个接收一些键值对作为可变参数的便利构造器，可以建议 Swift 用户使用字典字面量作为替代：

```objective-c
+ (instancetype)collectionWithValues:(NSArray *)values 
                             forKeys:(NSArray<NSCopying> *)keys NS_SWIFT_UNAVAILABLE("使用字典字面量替代");
```

试图在 Swift 中调用`+collectionWithValues:forKeys:`方法将导致编译错误。

<a name="refining_objective_c_declarations"></a>
## 优化 Objective-C 声明

可以使用`NS_REFINED_FOR_SWIFT`宏标记 Objective-C 方法的声明，然后在 Swift 中通过扩展提供一个优雅的 Swift 接口，通过该接口去调用方法的原始实现。例如，接收一个或者多个指针参数的 Objective-C 方法可以优化为一个返回元组值的 Swift 方法。使用该宏标记的方法导入到 Swift 后会做如下处理：

- 对于初始化方法，在第一个外部参数名前加双下划线（`__`）。
- 对于对象下标方法，只要设值或取值方法被标记`NS_REFINED_FOR_SWIFT`宏，这对下标方法就会变为 Swift 中的普通方法。被标记宏的下标方法会在方法名前加双下划线（`__`）。
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

可以通过 Swift 扩展来提供一个优化后的接口：

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
## 为产品模块命名

无论是 Xcode 为 Swift 代码自动生成的头文件，还是 Objective-C 桥接头文件，都会根据产品模块名来命名。默认情况下，产品模块名和产品名一样。然而，如果产品名包含特殊字符（只能是字母、数字、下划线），例如点号（`.`），作为产品模块名时，这些特殊字符会被下划线（`_`）替换。如果产品名以数字开头，作为产品模块名时，第一个数字也会被下划线替换。

也可以为产品模块名提供一个自定义的名称，Xcode 会根据这个名称来命名桥接头文件和自动生成的头文件，只需修改`Build setting > Packaging > Product Module Name`中的名称即可。

> 注意  
> 无法改变框架的产品模块名。

<a name="troubleshooting_tips_and_reminders"></a>
## 故障排除贴士

- 将 Swift 和 Objective-C 文件看作同一代码集，注意命名冲突。

- 如果使用框架，确保框架的`Build setting > Packaging > Defines Module`（`DEFINES_MODULE`）设置为`Yes`。

- 如果使用 Objective-C 桥接头文件，确保`Build setting > Swift Compiler > Code Generation`（`SWIFT_OBJC_BRIDGING_HEADER`）中的头文件路径是头文件自身相对于项目的路径，例如`MyApp/MyApp-Bridging-Header.h`。

- Xcode 会根据产品模块名（`PRODUCT_MODULE_NAME`），而不是 target 的名称（`TARGET_NAME`）来命名 Objective-C 桥接头文件以及为 Swift 代码自动生成的头文件。详情请参阅[为产品模块命名](#naming_your_product_module)小节。

- 只有继承自 Objective-C 类的 Swift 类，以及标记了`@objc`（包括各种被隐式标记的情况）的 Swift 声明，才能在 Objective-C 中使用。

- 将 Swift 代码导入到 Objective-C 时，注意 Objective-C 无法转化 Swift 的独有特性。详情请参阅[在 Objective-C 中使用 Swift](#using_swift_from_objective-c)小节。

- 如果在 Swift 代码中使用了自定义的 Objective-C 类型，在 Objective-C 中使用这部分 Swift 代码时，确保先导入相关的 Objective-C 类型的头文件，然后再将 Xcode 为 Swift 代码自动生成的头文件导入。

- 标记`private`或`fileprivate`修饰符的 Swift 声明不会出现在自动生成的头文件中，因为私有声明不会暴露给 Objective-C，除非它们被标记`@IBAction`，`@IBOutlet`或者`@objc`。

- 对于应用程序的 target，当存在 Objective-C 桥接头文件时，标记`internal`修饰符的 Swift 声明也会出现在自动生成的头文件中。

- 对于框架的 target，只有标记`public`或`open`修饰符的声明才会出现在自动生成的头文件中，不过依然可以在框架内部的 Objective-C 代码中使用标记`internal`修饰符的 Swift API，只要它们所属的类继承自 Objective-C 类。关于访问级别修饰符的更多信息，请参阅 [*The Swift Programming Language 中文版*](http://wiki.jikexueyuan.com/project/swift/) 中的[访问控制](http://wiki.jikexueyuan.com/project/swift/chapter2/24_Access_Control.html)章节。
