---
layout: post
title: "在iOS中创建静态库"
date: 2013-11-30 22:45
comments: true
categories: iOS raywenderlich 翻译
---

本文由 [**Migrant**](http://migrant.github.io/) 翻译自 [Creating a Static Library in iOS Tutorial](http://www.raywenderlich.com/41377/creating-a-static-library-in-ios-tutorial)，转载请注明出处。

{% img right /images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-00.jpg Create a static library in iOS! %}

如果你作为iOS开发者已经有一段时间，可能会有一套属于自己的类和工具函数，它们在你的大多数项目中被重用。

重用代码的最简单方法是简单的 _拷贝/粘贴_ 源文件。然而，这种方法很快就会成为维护时的噩梦。因为每个app都有自己的一份代码副本,你很难在修复bug或者升级时保证所有副本的同步。

这就是静态库要拯救你的。一个静态库是若干个类,函数,定义和资源的包装，你可以将其打包并很容易的在项目之间共享。

在本教程中，你将用两种方法亲手创建你自己的通用静态库。

<!--more-->

为了获得最佳效果，你应该熟悉Objective-C和iOS编程。Core Image的相关知识并不是必须的，但是如果你对示例工程和滤镜代码如何工作感兴趣，了解它会有所帮助。

准备好以效率的名义减少，重用并再生你的代码!

## 为什么使用静态库

创建静态库可能出于以下几个理由:

* 你想将一些你和你团队中的同事们经常使用的类打包并轻松的分享给周围其他人。
* 你想让一些通用代码处于自己的掌控之下，以便于修复和升级。
* 你想将库共享给其他人，但不想让他们看到你的源代码。
* 你想创建一个还在不断开发的库的快照版本。

本教程假设你已经完成学习[Core Image Tutorial](http://www.raywenderlich.com/22167/beginning-core-image-in-ios-6)，并对其中展示如何应用图片特效的代码得心应手。

将上述代码添加到一个静态库中，然后在一个app的修改版本中使用这个静态库。我们会得到一个带有上面列表中全部好处的完全相同的应用。

## 开始

运行Xcode，选择`File\New\Project`，在`Choose a template ` 对话框中选择`iOS\Framework & Library\Cocoa Touch Static Library`,如下图:

![New Lib](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-01.png "New Lib")

点击`Next`。在工程选项对话框中，输入`ImageFilters`作为产品名。再输入一个唯一的公司标识，确保`Use Automatic Reference Counting`被选中且`Include Unit Tests`未选中。如下图:

![Lib Name](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-02.png "Lib Name")

点击`Next`。最后，选择你想保存工程的位置并点击`Create`。

Xcode已经准备好静态库工程，甚至已经为你添加了一个`ImageFilters`类。这就是你的滤镜代码将要存放的地方。

>注意: 你可以添加任意数量的类到静态库中或者从中删除原有的类。本教程中的代码都会写在开始就被创建好的`ImageFilters`类中。

你的Xcode工程还是一片空白，现在我们添加一些代码进去!

## 图片滤镜

该库使用UIKit，为iOS设计，所以你要做的第一件事就是在头文件中导入UIKit。打开`ImageFilters.h`，在文件顶部添加以下代码:

```objc
#import <UIKit/UIKit.h>
```

接下来将以下声明部分的代码粘贴到`@interface ImageFilters : NSObject`下面

```objc
@property (nonatomic,readonly) UIImage *originalImage;
 
- (id)initWithImage:(UIImage *)image;
- (UIImage *)grayScaleImage;
- (UIImage *)oldImageWithIntensity:(CGFloat)level;
```

这些头文件中的声明定义了类的公开接口。其他开发者(包括你自己)使用该库时，只需通过阅读该头文件就可以知道类名和暴露的方法。

现在增加实现。打开`ImageFilters.m`文件，粘贴以下代码到`#import "ImageFilters.h"`下面:

```objc
@interface ImageFilters()
 
@property (nonatomic,strong) CIContext  *context;
@property (nonatomic,strong) CIImage    *beginImage;
 
@end
```

上面的代码声明了一些内部使用的属性。它们不是公开的，所以使用该库的引用没有使用它们的入口。

最后，你需要实现方法。粘贴以下代码到`@implementation ImageFilters:`下面:

```objc
- (id)initWithImage:(UIImage *)image
{
    self = [super init];
    if (self) {
        _originalImage  = image;
        _context        = [CIContext contextWithOptions:nil];
        _beginImage     = [[CIImage alloc] initWithImage:_originalImage];
    }
    return self;
}
 
- (UIImage*)imageWithCIImage:(CIImage *)ciImage
{
    CGImageRef cgiImage = [self.context createCGImage:ciImage fromRect:ciImage.extent];
    UIImage *image = [UIImage imageWithCGImage:cgiImage];
    CGImageRelease(cgiImage);
 
    return image;
}
 
- (UIImage *)grayScaleImage
{
    if( !self.originalImage)
        return nil;
 
    CIImage *grayScaleFilter = [CIFilter filterWithName:@"CIColorControls" keysAndValues:kCIInputImageKey, self.beginImage, @"inputBrightness", [NSNumber numberWithFloat:0.0], @"inputContrast", [NSNumber numberWithFloat:1.1], @"inputSaturation", [NSNumber numberWithFloat:0.0], nil].outputImage;
 
    CIImage *output = [CIFilter filterWithName:@"CIExposureAdjust" keysAndValues:kCIInputImageKey, grayScaleFilter, @"inputEV", [NSNumber numberWithFloat:0.7], nil].outputImage;
 
    UIImage *filteredImage = [self imageWithCIImage:output];
    return filteredImage;
}
 
- (UIImage *)oldImageWithIntensity:(CGFloat)intensity
{
    if( !self.originalImage )
        return nil;
 
    CIFilter *sepia = [CIFilter filterWithName:@"CISepiaTone"];
    [sepia setValue:self.beginImage forKey:kCIInputImageKey];
    [sepia setValue:@(intensity) forKey:@"inputIntensity"];
 
    CIFilter *random = [CIFilter filterWithName:@"CIRandomGenerator"];
 
    CIFilter *lighten = [CIFilter filterWithName:@"CIColorControls"];
    [lighten setValue:random.outputImage forKey:kCIInputImageKey];
    [lighten setValue:@(1 - intensity) forKey:@"inputBrightness"];
    [lighten setValue:@0.0 forKey:@"inputSaturation"];
 
    CIImage *croppedImage = [lighten.outputImage imageByCroppingToRect:[self.beginImage extent]];
 
    CIFilter *composite = [CIFilter filterWithName:@"CIHardLightBlendMode"];
    [composite setValue:sepia.outputImage forKey:kCIInputImageKey];
    [composite setValue:croppedImage forKey:kCIInputBackgroundImageKey];
 
    CIFilter *vignette = [CIFilter filterWithName:@"CIVignette"];
    [vignette setValue:composite.outputImage forKey:kCIInputImageKey];
    [vignette setValue:@(intensity * 2) forKey:@"inputIntensity"];
    [vignette setValue:@(intensity * 30) forKey:@"inputRadius"];
 
    UIImage *filteredImage = [self imageWithCIImage:vignette.outputImage];
 
    return filteredImage;
}
```

这段代码实现了初始化和图片滤镜功能。详细解释上述代码的功能已经超出了本教程的范围，你可以从[Core Image Tutorial](http://www.raywenderlich.com/22167/beginning-core-image-in-ios-6)中了解到更多的关于 Core Image 和滤镜的知识。

到这里，你已经有了一个静态库，它有一个暴露了以下3个方法的公开类`ImageFilters`:

* `initWithImage` : 初始化滤镜类
* `grayScaleImage` : 创建灰阶图片
* `oldImageWithIntensity` : 创建怀旧效果的图片

现在构建并运行你的库。你会注意到Xcode的"Run"按钮只是执行了一次构建，而并不能真正的运行库去查看效果，因为并没有真正的app。

静态库的后缀名是`.a`而并不是一个`.app`或者`.ipa`文件。可以在工程导航栏中的`Products`文件夹下找到生成的静态库。右键点击`libImageFilters.a`并在弹出菜单中选择`Show in Finder`。

![Show in Finder](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-03.png "Show in Finder")

Xcode会在Finder中打开文件夹，你可以看到以下类似的结构:

![Lib Structure](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-04.png "Lib Structure")

离完成一个库产品还剩两件事:

* **Header files** : 在`include`文件夹中可以找到库的所有公开头文件。在该示例中，只有一个公开类所以文件夹中只有一个`ImageFilters.h`文件。稍后你会在你的app工程中用到这个头文件以便于Xcode在编译期识别暴露的类。
* **Binary Libraty** : Xcode生成的静态库是`ImageFilters.a`。想在应用中使用该库，你需要用该文件链接。

这两个部分和你想在应用里包含一些新的框架时所需要做的事很相似，简单的导入框架头文件并建立链接。

库已经准备就绪，需要附加说明的是，默认情况下，库文件只会为当前的架构构建。如果你在模拟器下构建，那么库会包含对应i386架构的结果代码；如果在真机设备下构建，你将会得到对应ARM架构的代码。你可能需要构建两个版本的库，并且当从模拟器切换到设备的时候选择其中一个使用。

怎么办？

幸运的是，有一个更好的办法可以不建立多个配置或在工程中构建产品就可以支持多个平台。你可以创建一个对应 _2个_ 架构的包含结果代码的`universal binary`。

## 通用二进制

通用二进制是一种特殊的二进制文件，它包含对应多个架构的结果代码。你可能在从PowerPC(PPC)到Inter(i386)的Mac电脑产品线的过渡中对其有所熟悉。在这个过程中，Mac应用程序通常迁移为包含 _2个_ 可执行包的一个二进制文件，这样应用程序即能在Inter也能在PowerPC的Mac电脑上运行。

同时支持ARM和i386的概念并没有太大不同。在这里静态库要包含支持iOS设备(ARM)和模拟器(i386)的结果代码。Xcode可以识别通用库，每次你构建应用的时候，它会根据目标选择适当的架构。

为了创建通用二进制库，需要使用一个名为[lipo](https://developer.apple.com/library/mac/documentation/darwin/reference/manpages/man1/lipo.1.html)的系统工具。

![Lipo Cat](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-05.jpg "Lipo Cat")

别担心宝贝，不是那种lipo! :] (lipo有脂肪的意思 -- 译者注)

**lipo**是一个命令行工具，它允许在通用文件上执行操作(类似于创建通用二进制, 列出通用文件内容等等)。本教程中使用**lipo**的目的是联合不同架构的二进制文件到单个输出文件中。你可以直接在命令行中使用**lipo**命令，但在本教程中你可以让Xcode执行一段创建通用库的命令行脚本来为你做这件事。

Xcode中一个集合目标可以一次构建多个目标，包括命令行脚本。在Xcode菜单中选择`File/New/Target`，选择`iOS/Other`并点击`Aggregate`，如图:

![Aggregate Target](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-06.png "Aggregate Target")

将目标命名为`UniversalLib`，确保选中`ImageFilters`工程，如图:

![Aggregate Universal](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-07.png "Aggregate Universal")

在工程导航视图中选中`ImageFilters`，然后选择`UniversalLib`目标。切换到`Build Phases`标签；在这里设置构建目标时将要执行的动作。

点击`Add Build Phase`按钮，在弹出的菜单中选择`Add Run Script`，如下图:

![Aggregate Phase](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-08.png "Aggregate Phase")

现在你需要设置脚本项。展开`Run Script`模块，在`Shell`行下粘贴如下代码:

```sh
# define output folder environment variable
UNIVERSAL_OUTPUTFOLDER=${BUILD_DIR}/${CONFIGURATION}-universal
 
# Step 1. Build Device and Simulator versions
xcodebuild -target ImageFilters ONLY_ACTIVE_ARCH=NO -configuration ${CONFIGURATION} -sdk iphoneos  BUILD_DIR="${BUILD_DIR}" BUILD_ROOT="${BUILD_ROOT}"
xcodebuild -target ImageFilters -configuration ${CONFIGURATION} -sdk iphonesimulator -arch i386 BUILD_DIR="${BUILD_DIR}" BUILD_ROOT="${BUILD_ROOT}"
 
# make sure the output directory exists
mkdir -p "${UNIVERSAL_OUTPUTFOLDER}"
 
# Step 2. Create universal binary file using lipo
lipo -create -output "${UNIVERSAL_OUTPUTFOLDER}/lib${PROJECT_NAME}.a" "${BUILD_DIR}/${CONFIGURATION}-iphoneos/lib${PROJECT_NAME}.a" "${BUILD_DIR}/${CONFIGURATION}-iphonesimulator/lib${PROJECT_NAME}.a"
 
# Last touch. copy the header files. Just for convenience
cp -R "${BUILD_DIR}/${CONFIGURATION}-iphoneos/include" "${UNIVERSAL_OUTPUTFOLDER}/"
```

代码并不十分复杂，它是这样工作的:

* **UNIVERSAL_OUTPUTFOLDER** 包括了通用二进制包将要被存放的文件夹:"Debug-universal"
* **Step 1.** 第2行执行了`xcodebuild`并命令它构建ARM架构的二进制文件。(你可以看到这行中的`-sdk iphoneos`参数)
* 下一行再次执行了`xcodebuild`命令并在另一个文件夹中构建了一个针对Inter架构的iPhone模拟器的二进制文件，在这里关键参数是`-sdk iphonesimulator -arch i386`。(如果感兴趣，你可以在[man page](http://developer.apple.com/documentation/Darwin/Reference/ManPages/man1/xcodebuild.1.html)了解更多关于xcodebuild的资料)
* **Step 2.** 现在已经有了2个.a文件分别对应两个架构。执行`lipo -create`，用它们创建出一个通用二进制。
* 最后一行的作用是复制头文件到通用构建文件夹的外层。(用`cp`命令)

你的Run Script窗口应该看起来如下:

![Aggregate Script](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-09.png "Aggregate Script")

现在你已经准备好构建一个静态库的通用版本。在方案下啦菜单中选择集合目标`UniversalLib`，如下(不像截图上的"iOS Device"，你看到的可能是自己的设备名字):

![Aggregate Scheme](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-10.png "Aggregate Scheme")

点击`Play`按钮来为集合方案构建目标。

在`libImageFilters.a`上再次选择`Show in Finder`查看结果。将Finder切换到列视图查看文件夹层次，可以看到一个包含库的通用版本的叫做`Debug-Universal`的新文件夹(或`Release-Universal`如果你构建了发布版本)，如下图:

![St Finder](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-11.png "St Finder")

除了这个链接到模拟器和真实设备的二进制文件，你还可以找到普通的头文件和静态库文件。

这是你创建自己的通用静态库所需要学习的所有知识。

概括起来，一个静态库工程和一个应用工程非常相似。可以拥有一个或多个类，最后的产品是头文件和一个.a文件。这个.a文件就是可以链接到多个应用程序中的静态库。

## 在应用中使用静态库

在应用中使用`ImageFilters`类和直接使用源代码并没有太大区别:导入头文件然后开始使用类。问题是Xcode并不知道头文件和库文件的位置。

有两种办法可以将静态库引入到工程中:

* **方法 1:** 直接引用头文件和库二进制文件(.a)
* **方法 2:** 将库工程作为子项目

选择哪一种方法完全取决于你的喜好或者是否有静态库的源代码和工程配置文件任由你支配。

本教程将分别介绍两种方法。你可以自由尝试第一个或第二个，但推荐按照文中介绍的顺序分别尝试两个。在两个部分的开头，需要一个zip文件，该文件是在[Core Image Tutorial](http://www.raywenderlich.com/22167/beginning-core-image-in-ios-6)中创建的应用的修改版本，修改后的版本使用了库中新的`ImageFilters`类。

本教程的主要目的是教你如何使用静态库，所以修改后的工程包括了所有应用需要的源代码。这样你就可以将注意力集中在使用库所需要的工程设置上。

## 方法 1: 头文件和库二进制文件

在本节中，你需要下载[starter project for this section](http://cdn2.raywenderlich.com/wp-content/uploads/2013/07/CoreImageFun.staticlib.zip)。复制压缩文件到硬盘上的任意文件夹并解压。可以看到如下的文件夹结构:

![Lib File Tree](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-12.png "Lib File Tree")

为了方便起见，.a通用库文件和头文件已经复制了一份在其中，但工程并未设置使用它们。你将从这里开始。

>注意: 标准的Unix引入惯例是一个`include`文件夹，用来存放头文件，一个`lib`文件夹用来存放库文件(.a)。这种文件夹结构这是一种惯例，并不强制。你并不需要一定遵从这种结构或者复制文件到工程文件夹中。在你自己的应用中，你可以任意选择头文件和库文件的位置，只要随后在Xcode工程中设置了适当的路径。

打开工程，构建并运行你的应用，将会看到以下错误:

![Lib Error Include](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-13.png "Lib Error Include")

正如所期望的那样，应用并不知道去哪里寻找头文件。为了解决这个问题，你需要在工程中添加一个`Header Search Path`，指明头文件存放的位置。设置头文件搜索路径始终是使用静态库的第一步。

按照下图示范，在导航栏中点击工程根节点(1)，选择`CoreImageFun`目标(2)。选择`Build Settings`(3)，在列表中找出`Header Search Paths`设置项。如果必要，可以在搜索框中输入"header search"来过滤庞大的设置列表(4)。

![Lib Header Search](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-14.png "Lib Header Search")

双击`Header Search Paths`项，弹出一个浮动窗口，点击`+`按钮，输入:

```
$SOURCE_ROOT/include
```

弹出窗口应该如下所示:

![Lib Header Search](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-15.png "Lib Header Search")

**$SOURCE_ROOT**是一个Xcode环境变量，指向工程根文件夹。Xcode会使用包含你工程的实际文件夹代替此变量，这意味着即使你把工程移动到其它文件夹或驱动器，它仍然可以指向最新的位置。

在弹出窗口范围外点击鼠标使其消失，你会看到Xcode已经自动将变量转换为工程的实际位置，如图所示:

![Lib Header Search](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-16.png "Lib Header Search")

构建并运行应用，看看结果是什么。呃……一些链接错误出现了:

![Lib Header Search](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-17.png "Lib Header Search")

这看起来并不是很好，但是给了你另一个你所需要的信息。仔细看，会发现所有的编译错误全都消失了，全部被链接错误所代替。这表示Xcode找到了头文件并且用它去编译应用，但在链接阶段，Xcode无法找到`ImageFilter`类的结果代码。为什么？

很简单 -- 你还没有告诉Xcode去哪里寻找包含类实现的库文件。(看，没什么大不了)

如下面的屏幕截图所示，回到`CoreImageFun`目标(2)的构建设置(1)。选择`Build Phases`标签(3)，展开`Link Binary With Libraries`部分(4)。最后，点击`+`按钮(5)。

![Lib Link](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-18.png "Lib Link")

在出现的窗口中，点击`Add Other…`按钮，在工程根文件夹下的`lib`子目录中找到`libImageFilters.a`库文件，如图:

![Lib Link](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-19.png "Lib Link")

完成这些以后，你的Build Phase标签看起来如下:

![Lib Link](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-20.png "Lib Link")

最后一步是增加`-ObjC`链接标识。该链接尝试更高效的只包含需要的代码，而有时会排除静态库代码。使用该标识，库中的所有Objective-C类和类别都将被适当的加载。你可以从苹果的[Technical Q&A QA1490](http://developer.apple.com/library/mac/#qa/qa1490/_index.html)了解详细信息。

点击`Build Settings`标签，找到`Other linker Flags`设置，如图:

![OBJC Flag](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-21.png "OBJC Flag")

在弹出窗口中，点击`+`按钮并输入`-ObjC`，如图:

![OBJC Flag](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-22.png "OBJC Flag")

最后构建并运行应用，此时应该不会得到任何构建错误信息，应用顺利展示它的光彩之处:

![App](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-23.png "App")

拖动滑块改变滤镜级别，或者点击GrayScale按钮。对图片应用滤镜的代码来自于静态库，而不是应用。

恭喜 -- 你已经构建了你的第一个静态库并在一个真正的应用里使用它!你会发现这种包含头文件和库的方法在很多第三方库中使用，如AdMob，TestFlight或一些不提供源代码的商业库。

## 方法 2: 子项目

在这部分，请在[这里](http://cdn1.raywenderlich.com/wp-content/uploads/2013/07/CoreImageFun.subproject.zip)下载所需工程。

复制下载的文件到任意位置，解压。可以看到以下文件夹结构:

![Subproject Structure](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-24.png "Subproject Structure")

如果学习了方法一，你可能注意到了工程的差异。这个工程里没有任何的头文件和静态库文件 -- 因为根本不需要。作为替代方案，你要将你在本教程开始创建的`ImageFilters`库工程添作为依赖加到本工程中。

在做这些之前，构建并运行应用。会看到以下错误:

![Lib Error Include](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-25.png "Lib Error Include")

如果学习过上一个方法，你已经知道如何修复这个问题。在示例工程中，你在`ViewController`类中使用了`ImageFilters`类，但并未告诉Xcode去哪里寻找头文件。Xcode会尝试寻找`ImageFilters.h`文件，但是失败了。

将`ImageFilters`库工程作为子项目所需的所有操作就是拖拽库工程文件到库文件树中。如果该工程已经在另一个Xcode窗口中被打开，那么Xcode无法正确将其添加为子工程。所以在继续本教程之前，确保`ImageFilters`库工程已经被关闭。

在Finder中找到名为`ImageFilters.xcodeproj`库工程文件。拖拽它到`CoreImageFun`工程左侧的导航栏中，如图:

![Subproject Drag](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-26.png "Subproject Drag")

完成拖放后，你的工程浏览视图应该如下图所示:

![Subproject Drag](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-27.png "Subproject Drag")

现在Xcode已经识别了子工程，你可以将库添加为工程依赖。这样Xcode就可以在构建主应用之前确保库为最新版本。

点击工程文件(1)，选择`CoreImageFun`目标(2)。点击`Build Phases`标签(3)并展开`Target Dependencies`(4)，如图:

![Dependencies](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-28.png "Dependencies")

点击`+`按钮增加一个新依赖。如下图所示，确保你从弹出窗口中选择了`ImageFilters`目标(不是`universalLib`):

![Dependency](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-29.png "Dependency")

添加完成之后，依赖窗口应该如图所示:

![Dependency](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-30.png "Dependency")

最后，设置静态库工程链接到应用。展开`Link Binary with libraries`，点击`+`按钮，如图:

![Subproject Link](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-31.png "Subproject Link")

选择`libImageFilters.a`，点击`Add`:

![Subproject Link](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-32.png "Subproject Link")

添加库之后，`Link Binary with Libraries`部分应该如图所示:

![Subproject Link](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-33.png "Subproject Link")

像方法一那样，最后一步是增加`-ObjC`链接标识。点击`Build Settings`标签，找到`Other linker Flags`设置，如图:

![OBJC Flag](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-34.png "OBJC Flag")

在弹出窗口中，点击`+`按钮并输入`-ObjC`，如图:

![OBJC Flag](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-35.png "OBJC Flag")

构建并运行应用，应该没有任何错误，应用会再一次被打开:

![App](/images/posts/2013-11-30-creating-a-static-library-in-ios-tutorial-36.png "App")

拖动滑块或者点击GrayScale按钮查看图片滤镜结果。滤镜逻辑的代码完全包含在库中。

如果按照第一种方法在应用中添加库(使用头文件和库文件)，你可能注意到和第二种方法的区别。在方法二中，你没有在工程设置中添加任何头文件搜索路径。另一个区别是你没有使用通用库。

为什么会有这样的区别？当添加一个库作为一个子工程，Xcode会为你考虑几乎所有的事情。添加子工程和依赖后，Xcode知道去哪里寻找头文件和二进制文件，也知道根据你的设置去选择哪需要构建哪一个架构的库。这非常方便。

如果你使用你自己的库或者拥有源代码和工程文件，将库作为子工程不失为一个引入静态库的简便的方法。让你更容易作为工程依赖构建整合，并担心更少的事情。

## 未来

你可以从[这里](http://cdn5.raywenderlich.com/wp-content/uploads/2013/07/Staticlibrary.fullsource.zip)下载到包括了本教程所有代码的工程。

希望本教程能够让你对静态库的基本概念和怎样在应用中使用它们有一个更深入的了解。

下一步就是用所学到的知识构建你自己的库了。你肯定有一些添加到工程中通用类。它们是你加入你自己的可复用库的优秀候选人。你也可以考虑根据功能创建多个库:网络部分的代码作为一个，UI部分作为另一个，等等。你可以只向工程中添加所需要的代码。

为了强化和深入探讨你在本教程中所学到的概念，我推荐苹果的文档[Introduction to Using Static Libraries in iOS](http://developer.apple.com/library/ios/#technotes/iOSStaticLibraries/Introduction.html)。

希望你喜欢本教程，如果你有任何问题或评论，请在下面的讨论区中讨论。