
# Swift Package Manager探讨

## 概述
Swift Package Manager（简称SPM），是Apple官方提供的软件包管理器，类比如Ruby的RubyGems, Python的pip。开源框架里知名度较高的[SwiftyJSON](https://github.com/SwiftyJSON/SwiftyJSON)、[SQLite.swift](https://github.com/stephencelis/SQLite.swift)等都已支持SPM集成。<br>
WWDC18专门开了一个[Session](https://developer.apple.com/videos/play/wwdc2018/411/)介绍了SPM的内部结构、基本使用及设计理念，并介绍了之后即将支持的一些新特性。本文最后笔者对Session里简单带过的libSwiftPM和libSyntax做了简单分析。



## 为什么要使用SPM：
- 最重要的一点：官方支持  
	官方支持，保证了所有Swift软件包的可信来源和通用标准。同时，SPM作为Swift项目的一部分，Apple在Swift工具链中提供了更强大的原生支持。
	
- 跨平台构建  
	SPM是一个跨平台方案，SPM提供了完整的构建环境，能在不同平台上来构建和分发Swift软件包。
	
- 和Swift更紧密的关系  
	SPM采用Swift编写，可以使用Swift自身的强大特性，并且与Swift的核心库紧密联系。

- SPM有更先进的设计  
	这个我们在后面章节展开。
	
	
- SPM的社区影响范围日益增长<br>
	作为Swift开源项目的一部分，社区的热度不断升温，CocoaPods的众多核心成员也参与架构设计，影响力越来越大。
	
	
## SPM基本用法
SPM的基本概念可参见[官网描述](https://swift.org/package-manager/)。  


### 常用命令
- **swift build**: 编译Package
- **swift run**: 编译Package里的可执行文件并运行
- **swift test**: 运行Package里的单元测试
- **swift package**: 这个命令的用法很多，Package的初始化、修改、更新、resolve检查、编译选项指定等 

### 内部结构
对于一个Package来说，主要分为三部分，Target、Products、Dependencies.  

![](img/package.png)
- **Products**  
Products可以理解为Package输出的形态。对比与Cocoapods只能以静态库/动态库形式，SPM还可以输出为可执行文件。  

- **Target**  
Target理解为包含多个文件的一个模块，target之间是可以有依赖关系，同时，输出的Products也会依赖一个或多个target。  

- **Dependencies**  
Dependencies描述了当前的package需要依赖其他package。  

一个典型的Package目录如下：
```swift
example-package-playingcard
├── Sources
│   └── DeckOfPlayingCards
│       ├── DeckOfPlayingCards.swift
│       ├── Rank.swift
│       └── Suit.swift
└── Package.swift
```
其中Package.swift的定义：

```swift
let package = Package(
    name: "DeckOfPlayingCards",
    products: [
        .library(name: "DeckOfPlayingCards", targets: ["DeckOfPlayingCards"]),
    ],
    dependencies: [
        .package(url: "https://github.com/apple/example-package-fisheryates.git", from: "2.0.0"),
        .package(url: "https://github.com/apple/example-package-playingcard.git", from: "3.0.0"),
    ],
    targets: [
        .target(
            name: "DeckOfPlayingCards",
            dependencies: ["FisherYates", "PlayingCard"]),
        .testTarget(
            name: "DeckOfPlayingCardsTests",
            dependencies: ["DeckOfPlayingCards"]),
    ])
```
Sources目录下的文件不需要显示声明，SPM会自动导入，Dependencies依赖了两个外部Package， 其中Target的依赖既有Package内部Target的依赖也有外部Target的依赖。

Package.resolved被放在顶层包的一个文件中，类似于许多软件管理器lock的实现，swift使用该文件来做依赖关系解析。在执行swift build swift test swift package generate-xcodeproj命令时，默认会隐式调用swift package resolve检查依赖版本是否正确。
Package.resolved的详细设计思路可以参考[提案](https://github.com/apple/swift-evolution/blob/master/proposals/0175-package-manager-revised-dependency-resolution.md)。
	

	
## SPM的设计思想
SPM的设计思想也遵循了Swift的一些设计思想：

- 安全: SPM的编译环境是在一个独立的沙盒里，无法任意执行命令和脚本。

- 快速: 得益于新的llbuild构建系统，SPM可以支持数百万节点的关系依赖图。

- 表现力: 采用Swift作为SPM的配置描述语言。

下图描述了Apple对SPM的大体思想：
![](img/1.png)
	
#### Configuration  
相对于Cocoapods等笨重啰嗦的DSL语法，SPM采用了Swift来作为SPM的配置描述语言，大大降低了开发人员的学习成本同时保留的Swift强大的语言特性，并可以得到 Swift相关工具的支持。  
#### Dependencies
SPM使用git tag来做版本依赖，格式上遵循Semantic Versioning。<br>
 >  **Semantic Versioning（Semver）**: 是为了解决软件包管理器中著名的[Dependency hell](https://en.wikipedia.org/wiki/Dependency_hell)问题而设计的一套简单的规则和条件来约束版本号配置的解决方案。<br> Semver规定格式里分为：主版本号.次版本号.修订号，版本号递增规则如下：
	<div>
	**1. 主版本号**：当你做了不兼容的 API 修改。<br>
	**2. 次版本号**：当你做了向下兼容的功能性新增。 <br>
	**3. 修订号**：当你做了向下兼容的问题修正。
	<div>
 	Semver同时支持语义化关键字来实现版本控制，需要对Semver更深入了解的读者可参考[官网介绍]([Semantic Versioning](https://semver.org/))。下图就是一个采用Semver版本控制的Package依赖树结构。<br> 	<div>
	
![](img/semfer.png)

#### Building<br>
	SPM采用了[swift-llbuild](https://github.com/apple/swift-llbuild) ,一个并行、高效、增量编译的构建系统作为其构建引擎。目前Xcode9也已集成，用作对LLVM、Clang及Swift的构建。同时，因为独立沙盒的构建环境，SPM提供了很强的安全性。
	
#### Workflow Features<br>
	如果你的某个软件包还处于开发模式下，tag的集成方式很不方便，类似于Cocoapods的做法，SPM也会提供[分支依赖](	https://github.com/apple/swift-evolution/blob/master/proposals/0150-package-manager-branch-support.md)和[本地依赖](https://github.com/apple/swift-evolution/blob/master/proposals/0201-package-manager-local-dependencies.md)做法。

	
		
#### Tools Evolution<br>
	和Cocoapods一样，当我们的软件包需要依赖特定的swift版本，需要在配置描述里显示声明。声明的枚举类型如下
	
```swift
/// Represents the version of the Swift language that should be used for
/// compiling Swift sources in the package.
public enum SwiftVersion {
    case v3
    case v4
    case v4_2

    /// User-defined value of Swift version.
    ///
    /// The value is passed as-is to Swift compiler's `-swift-version` flag.
    case version(String)
}
```
例如指定版本为4.0

```swift
// swift-tools-version:4.2

import PackageDescription

let package = Package(
    name: "HTTPClient",
    ...
    swiftLanguageVersions: [.v4, .v4_2]
)

```

## SPM的未来新特性
- **与其他工具的更好的集成<br>**
	使用**libSwiftPM**和**libSyntax**对SPM做更灵活的扩展实现。
- **发布和部署<br>**
	目前的发布流程依赖于手动建立git tag, git push origin [tag name] 的做法，之后会有更自动化的工具实现。
- **支持复杂的Packages<br>**
	 对资源文件的支持，更复杂的编译设置，和扩展的编译工具等
- **更好的管理Package生态<br>**
	跨平台的沙盒隔离、更安全的校验方案，支持fork操作和搜索索引等。
	
	
##### 预见到等到合适的时机，SPM成为Swift Package的标准管理工具是大势所趋。

参考：
<br>https://developer.apple.com/videos/play/wwdc2018/411/
<br>https://swift.org/blog/swift-package-manager-manifest-api-redesign/
<br>https://medium.com/xcblog/apple-swift-package-manager-a-deep-dive-ebe6909a5284

	


