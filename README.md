# TestDemo
# 一、Swift调用内部OC类

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/386fa47f76294f73a062592dea0f3d95~tplv-k3u1fbpfcp-zoom-1.image)

1.  红色实线：已经完成的、且无法变更的，我们需要这样来缩减外放文件，减少暴露。
1.  黑色实线：目前已经实现的OC侧与 Swift 侧的向下引用。

3.  蓝色虚线：OC 与 Swift 内部类的互相调用，这也是混编最重要的工作。


### 问题背景：在目前已经存在OC SDK中，存在着大量的基本工具类，我们需要使Swift内部类调用到这些OC基础类。

-   这样的工具类功能单一，耦合度低，可以有效减少 Swift 代码工程量。
-   目前Swift编写的内部类，OC 内部类不存在调用的情况，所以**我们只考虑 Swift 调用 OC 内部类的情况。**

## 所以我们目前的任务：`使TestSwift2调用TestOC2的 onlyForSwift2 方法`

### ｜Plan A：

1.  在 **Target** 中选择 **SWSDK** ，***Build Phases*** 将 **TestOC2** 从 **project** 拖入 **public**。
1.  在 ***SWSDK.h*** 中 加入 ***#import <SWSDK/TestOC2.h>** *。\


``` objc
#import "TestOC2.h"

@implementation TestOC2
+ (void)run {
    NSLog(@"TestOC2:内部OC类，非Public接口。");
}
+ (void)onlyForSwift2 {
    NSLog(@"SwiftTest2调用%s",__func__);
}
@end

class TestSwift2: NSObject {
    static func run() -> Void {
        print("TestSwift2: Swift内部类")
        TestOC2.onlyForSwift2()
    }
}
```

尽管目的是达到了，但是 ***TestOC2.h*** 也因此而公开了，要明白在实际项目中，我们这样的操作可不是一个两个，所以会暴露太多代码，不可取。（而且调用一次，就要进行这样的操作太蛋疼了 III#，#）

### ｜Plan B：

联系到我们在之前说的 modulemap 文件，进行库继承映射。

1.  新建一个Heade.h文件，在其添加要import的头文件：\


``` c
#ifndef Header_h
#define Header_h

#import "TestOC2.h"
#import "TestOC3.h"

#endif /* Header_h */
```
 


2.  新建一个文件，将其名字改成 ***module.modulemap*** ， 然后在其内编写如下。

``` c
module OCTest{
    umbrella header "Header.h"
    export *
}
```



3.  在 Building Setting 内搜索 Swift Compiler - Search Paths 后，在 Import Paths 中输入目录路径。如下图所示：\
    ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed56ff60e029494bbf8da45edf730bcf~tplv-k3u1fbpfcp-zoom-1.image)

3. ` ⚠️注意：PrefixHeader.pch文件可能存在失效的问题，可能需要删除文件以及Setting内路径。`

<!---->

5.  **以下是TestOC2、TestOC3内容:***

``` objc
@implementation TestOC2
+ (void)run {
    NSLog(@"%s",__func__);
}

+ (void)onlyForSwift2 {
    NSLog(@"SwiftTest2调用%s",__func__);
    [TestOC3 runFor_TestOC2];
}
@end
//--------------------------------------
@implementation TestOC3

+ (void)runFor_TestOC2 {
    NSLog(@"TestOC2调用了%s",__func__);
}

@end


//--------在Swift中调用TestOC2-----------
import UIKit
import OCTest

class TestSwift2: NSObject {
    static func run() -> Void {
        print("TestSwift2: Swift内部类")
        TestOC2.onlyForSwift2()
    }
}
//
2021-11-04 21:28:35.115221+0800 TestDemo[27264:7623092] +[TestOC run]
SWSDK/TestSwift.swift: run()
SWSDK/TestSwift2.swift: run()
2021-11-04 21:28:35.115688+0800 TestDemo[27264:7623092] SwiftTest2调用+[TestOC2 onlyForSwift2]
2021-11-04 21:28:35.115717+0800 TestDemo[27264:7623092] TestOC2调用了+[TestOC3 runFor_TestOC2]
2021-11-04 21:28:35.115734+0800 TestDemo[27264:7623092] +[TestOC2 run]
```
--------
***PLAN B明显能解决我们的问题，不暴露且可以被Swift调用，但是到现在还存在一些问题。***

**| Q&A:**

1.  TestSwift2引用的只是TestOC2,为什么需要导入TestOC3进Header.h?

<!---->

    因为TestOC2引用了TestOC3, 所以不导入TestOC3，编译报错无法继续执行。



2.  为什么PCH无效了?

<!---->

     因为Module是PCH的衍生物，是由于PCH无法解决预编译文件越来越多的情况下产生的，这个在<https://www.jianshu.com/p/4969b1c47bc8>中有解释。
    概括为：
        代码无法直接复制粘贴在别的工程中，因为没有相同的PCH文件,可能引发找不到依赖的文件。
        依赖的头文件被隐藏了或者传递依赖很深，不直观。
        如果没有有效的管理，依赖会渐渐的膨胀，最终越来越大。

3.  如果以后ModuleMap中导入的头文件越来越多，如何避免像PCH那样的问题呢？

<!---->
        ModuleMap中引用了SubModule，我们可以区分每个module的作用，来显式或者隐式的导入需要的Module。
        

-----
## 二、ModuleMap的解决方案

### ｜先讲一讲module.modulemap

### ｜上英语课：

> ***The crucial link between modules and headers is described by a module map, which describes how a collection of existing headers maps on to the (logical) structure of a module. For example, one could imagine a module std covering the C standard library. Each of the C standard library headers (<stdio.h>, <stdlib.h>, <math.h>, etc.) would contribute to the std module, by placing their respective APIs into the corresponding submodule (std.io, std.lib, std.math, etc.). Having a list of the headers that are part of the std module allows the compiler to build the std module as a standalone entity, and having the mapping from header names to (sub)modules allows the automatic translation of #include directives to module imports.***

> **略）中国人不骗中国人：模块和头文件之间的关键链接由模块映射描述，它描述了现有头文件的集合如何映射到模块的（逻辑）结构。例如，可以想象一个模块 std 涵盖了 C 标准库。每个 C 标准库头文件（<stdio.h>、<stdlib.h>、<math.h> 等）都会通过将它们各自的 API 放入相应的子模块（std.io、 std.lib、std.math 等）。拥有作为 std 模块一部分的头文件列表允许编译器将 std 模块构建为独立实体，并且拥有从头文件名称到（子）模块的映射允许将 #include 指令自动转换为模块导入。**

> **鲁迅说过：“好记性不如烂笔头，看万遍不如淦一遍” （逃**

1.  **其实可以这么理解：SDK那么庞大，那么多文件、类，始终以一个 YourProductName.h 向外对接，离不开 Modulemap 的功劳。**

``` c
framework module SWSDK {
  umbrella header "XXX.h"
 
  export *
  module * { export * }
}
```

2.  **上面是纯 OCFramework，所以内部就一个** **SWSDK.h** **文件。**
2.  **下面是 OC 中混编了 Swift ,因为内部 OC 类引用 Swift 需要 #import <XXX/XXX-Swift.h>,这时候 Modulemap 也需要继承所有 Swift 开放类。**

``` c
framework module SWSDK {
  umbrella header "XXX.h"
 
  export *
  module * { export * }
}
 
module SWSDK.Swift {
    header "XXX-Swift.h"
    requires objc
}
```
----
### ｜介绍Module的使用

#### 目前我们的modulemap长这样：

``` c
module OCTest{
    umbrella header "Header.h"
    export *
}
```

如果以后文件增多，目前无法满足以下情况:

-   文件众多，不利于管理。

-   无法区分代码功能、可能存在网络类、工具类、系统扩展类等功能不一的类。

-   有些类也不需要频繁导入，只是偶尔使用，不需要占用编译空间。

#### module的更多用法

``` c
module OCCommon {
    umbrella header "Header.h"
    export *
}


/* Header.h内容
	#import "TestOC2.h"
	#import "TestOC3.h"
	#import "SWSDK.h"
	#import "TestOC.h"
*/


module OCTest{
    explicit module submodule {
        umbrella "Other" // 文件夹
        module * { export * }
    }
    
    module B {
        header "TestB.h"
        header "Other/TestNetTool.h"
        export *
    }
    
}
```

``` c
├── Header.h
├── Other //新增文件夹
│   ├── OtherOne.h
│   ├── OtherOne.m
│   ├── TestNetTool.h
│   └── TestNetTool.m
├── SWSDK.h
├── Swift
│   ├── TestSwift.swift
│   └── TestSwift2.swift
├── TestB.h //新增文件
├── TestB.m
├── ...// 原来文件
└── module.modulemap
```

新增定义 **OCTest Module**,其中包括一个 ***submodule*** 和一个 ***B module***;

-   Other 是一个目录，其内涵 OtherOne , TestNetTool 两个头文件。
-   `module 的搜索路径是以 modulemap 为根路径，然后填写相对路径，如果 Other 下还有一个目录，则应该这样写：umbrella "Other/目录名称"` 。

-   **submodule** 多了一个修饰 **explicit**. 表示这个 **moduleA** 需要显示的调用才能使用。
-   在同一个Module下，我们无需 import 头文件。比如 **Module B** 的 TestB 和 **TestNetTool** ，`PS: 路径内的 **TestNetTool** 无法访问 TestB`。
-   一个头文件可以存在不同的**Module**内，比如 ***TestNetTool*** 存在 ***submodule*** 与 ***B*** 中。

``` objc
#import "TestB.h"

@implementation TestB
+ (void)run {
    NSLog(@"%s",__func__);
    [TestNetTool request];
}

@end
```

进入 Swift 调用 OC Module

``` swift
import UIKit
import OCCommon
import OCTest
import OCTest.submodule

class TestSwift2: NSObject {
    static func run() -> Void {
        print("(#fileID): (#function)")
        //OCCommon
        TestOC2.onlyForSwift2()
        
        //Module B
        TestNetTool.request()
        TestB.run()
        
        //submodule
        Otherone.run() 
    }
}
//输出
2021-11-05 10:53:23.614885+0800 TestDemo[28007:7899400] +[TestOC run]
SWSDK/TestSwift.swift: run()
SWSDK/TestSwift2.swift: run()
2021-11-05 10:53:23.615341+0800 TestDemo[28007:7899400] 连接网络中...
2021-11-05 10:53:23.615365+0800 TestDemo[28007:7899400] +[TestB run]
2021-11-05 10:56:00.676746+0800 TestDemo[28029:7902689] +[OtherOne run]
```

1.  如果不显式调用 **OCTest.submodule**，则 **Otherone.run()** 会报警：`Cannot find 'OtherOne' in scope`
1.  想要使用子**module submodule**，我们必须要显示的指明 **import OCFile.submodule** 才能使用。

3.  只**import OCTest.submodule**，也可以使用 Module B.


### 其他关键字

> 除此之外还有一些别的关键字可能会用到。\
> config_macros, export_as, private conflict, framework, requiresexclude, header, textual explicit,link,umbrella, extern, module, use export\
> **1.link framework "MyFramework"**\
> 指明依赖的别的framework. 但需要注意的是，这个即便你写了，目前也不会自动帮你引入，类似于注释的效果。 具体有点像 Microsoft Visaul Studio 的 #pragma comment(lib ...)\
> **2.framework module COFile**\
> 表明这个module符合于 Darwin-style(达尔文风格) framework,目前达尔文风格framework只用于 macOS 和 iOS. 有兴趣的同学可以查阅下。\
> **3.module OCFile [system]**\
> 指明这个是一个系统module，当clang编译的时候，会考虑到它使用了系统的header，因此而忽略到警告。 方法类似的 #pragma GCC system_header. 因为系统header往往不能完全遵循C语法，所有头文件中警告信息往往不显示，除非#warning显示.\
> **4.module OCFile [system] [extern_c]**\
> 表明module中的C代码可以被C++使用


