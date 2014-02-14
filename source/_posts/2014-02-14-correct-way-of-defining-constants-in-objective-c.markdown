---
layout: post
title: "正确定义Objective-C常量"
date: 2014-02-14 11:39
comments: true
categories: iOS 翻译
---

本文由 [**Migrant**](http://migrant.github.io/) 翻译自 [Correct Way of Defining Constants in Objective-C](http://walkingsmarts.com/correct-way-of-defining-constants-in-objective-c/)，转载请注明出处。

本文只是一个关于如何在Cocoa代码中定义常量的书签贴，答案来自于[stackoverflow.com的这个问题](http://stackoverflow.com/questions/538996/constants-in-objective-c)。这里为那些懒人提供了一些简短的介绍和帖子本身。你可能读遍了苹果开发者文档，知道一些特定的方法参数只能接受定义为常量的枚举值列表。比如事件类型标记(`NSKeyUpMask`，`NSKeyDownMask`，等等)，persistent store coordinator的存储类型(`NSSQLiteStoreType`，`NSBinaryStoreType`和`NSInMemoryStoreType`)，当然还有很多其他的。所有的这些归结为几行代码。实际上定义常量的时候代码行数是你想要的常量的两倍。步骤为:首先，创建`Constants.h`和`Constants.m`文件用来存放我们的常量。在`Constants.h`中，指定常量名字，将常量声明为一个指向`NSString`对象的指针:

```objc
// Constants.h
extern NSString * const MyOwnConstant;
extern NSString * const YetAnotherConstant;
```

最后，在`Constants.m`中通过赋值定义常量:

```objc
// Constants.m
NSString * const MyOwnConstant = @"myOwnConstant";
NSString * const  YetAnotherConstant = @"yetAnotherConstant";
```

现在你所需要做的只是引入`Constants.h`文件到你工程的预编译头文件。如果你有点小聪明，可能脑中会有两个问题。第一个问题或许是:在能够使用`#define`的情况下为什么要使用这种方法?这是个非常有意义的问题。答案很简单(但是在读到[这个答案](http://stackoverflow.com/questions/538996/constants-in-objective-c/539191#539191)之前还不是很明显) -- 使用这种方法你可以进行指针比较(`@"myString" == MyConstant`)而不是字符串比较(`[@"myString" isEqualToString:MyConstant]`)。前者非常非常快。第二个问题应该是为什么应该完全使用常量。又一个有意义的问题。你可以在每个使用常量的地方使用常量对应的值。但是有两个"但是"。第一，始终有人的因素。你很容易输入错字符串，而编译器并不会抱怨你的语法。但如果使用常量，它就会在你输入错常量名称的时候给予你警告。还有(第二个"但是")，XCode会尽最大努力的帮助我们自动完成代码，这些常量也不例外，因此方法会变得非常方便。Happy coding!