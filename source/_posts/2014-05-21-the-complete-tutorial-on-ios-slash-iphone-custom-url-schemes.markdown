---
layout: post
title: "自定义 URL Scheme 完全指南"
date: 2014-05-21 11:25
comments: true
categories: iOS 翻译
---

本文由 [**Migrant**](http://objcio.com) 翻译自 [The Complete Tutorial on iOS/iPhone Custom URL Schemes](http://iosdevelopertips.com/cocoa/launching-your-own-application-via-a-custom-url-scheme.html)，转载请注明出处。
 
**注意**: *自从自定义 URL 的引入，本文始终是我博客中阅读量最大的文章。虽然大多数都相同，但仍然有一些细微差别的变化。本文是原帖的重写版，更新为最新的 iOS 和 Xcode 版本。*

iPhone / iOS SDK 最酷的特性之一就是应用将其自身"绑定"到一个自定义 URL scheme 上，该 scheme 用于从浏览器或其他应用中启动本应用。

## 注册自定义 URL Scheme

注册自定义 URL Scheme 的第一步是创建 URL Scheme -- 在 Xcode Project Navigator 中找到并点击工程 info.plist 文件。当该文件显示在右边窗口，在列表上点击鼠标右键，选择 *Add Row*:

向下滚动弹出的列表并选择 *URL types*。

![iOS Custom URL Scheme](/images/posts/2014-05-21-the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes-01.gif)

点击左边剪头打开列表，可以看到 *Item 0*，一个字典实体。展开 *Item 0*，可以看到 *URL Identifier*，一个字符串对象。该字符串是你自定义的 URL scheme 的名字。建议采用反转域名的方法保证该名字的唯一性，比如 *com.yourCompany.yourApp*。

![urlScheme2a](/images/posts/2014-05-21-the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes-02.gif)

点击 *Item 0* 新增一行，从下拉列表中选择 *URL Schemes*，敲击键盘回车键完成插入。

![iOS Custom URL Scheme](/images/posts/2014-05-21-the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes-03.gif)

注意 URL Schemes 是一个数组，允许应用定义多个 URL schemes。

![iOS Custom URL Scheme](/images/posts/2014-05-21-the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes-04.gif)

展开该数据并点击 *Item 0*。你将在这里定义自定义 URL scheme 的名字。只需要名字，不要在后面追加 :// -- 比如，如果你输入 iOSDevApp，你的自定义 url 就是 iOSDevApp://

![iOS Custom URL Scheme](/images/posts/2014-05-21-the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes-05.gif)

此时，整个定义如下图:

![iOS Custom URL Scheme](/images/posts/2014-05-21-the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes-06.gif)

虽然我赞同 Xcode 使用描述性的名字的目的，不过看到创建的实际的 key 也是非常有用的。这里有一个方便的技巧，右键点击 plist 并选择 *Show Raw Keys/Values*，就能看到以下效果:

![iOS Custom URL Scheme](/images/posts/2014-05-21-the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes-07.png)

还有另一种有用的输出格式，XML，因为可以非常容易的看到字典和原始数组及其包括的实体的结构。点击 plist 并选择 *Open As - Source Code*:

![iPhone Custom URL Scheme](/images/posts/2014-05-21-the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes-08.gif)

<!--more-->

## 从 Safari 中调用自定义 URL Scheme

定义了 URL scheme，我们可以运行一个快速测试来验证应用是否如我们所期望的被调用。在这之前，我创建了一个准 UI 以辨别带有自定义 URL 的应用。该应用只有一个 UILabel，带有文本 "App With Custom URL"。[下载源代码](http://iosdevelopertips.com/downloads/#customURLScheme)

![iOS App with Custom URL](/images/posts/2014-05-21-the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes-09.png)

使用模拟器调用应用的步骤:

* 在 Xcode 中运行应用
* 一旦应用被安装，自定义 URL scheme 就会被注册
* 通过模拟器的硬件菜单中选择 Home 来关闭应用
* 启动 Safari
* 在浏览器地址栏输入之前定义的 URL scheme(如下)

![Call Custom URL Scheme from Safari](/images/posts/2014-05-21-the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes-10.png)

此时 Safari 将会关闭，应用会被带回到前台。祝贺你刚刚使用自定义 URL scheme 调用了一个 iPhone 应用。

## 从另一个 iPhone 应用中调用自定义 URL Scheme

让我们看看如何从另一个应用中调用自定义 URL scheme。我又创建了一个非常简单的 iPhone 应用，它只有一个 UILabel 和一个 UIButton -- 前者显示了一段信息，告诉你这个应用将要通过自定义 URL scheme 来调用另一个应用，按钮则开始这个行为。[下载源代码](http://iosdevelopertips.com/downloads/#customURLScheme)

![iPhone app that call Custom URL Scheme](/images/posts/2014-05-21-the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes-11.png)

buttonPressed 方法中的代码处理 URL 调用:

```objc
- (void)buttonPressed:(UIButton *)button
{
  NSString *customURL = @"iOSDevTips://";
 
  if ([[UIApplication sharedApplication] 
    canOpenURL:[NSURL URLWithString:customURL]])
  {
    [[UIApplication sharedApplication] openURL:[NSURL URLWithString:customURL]];
  }
  else
  {
    UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"URL error"
                          message:[NSString stringWithFormat:
                            @"No custom URL defined for %@", customURL]
                          delegate:self cancelButtonTitle:@"Ok" 
                          otherButtonTitles:nil];
    [alert show];
  }    
}
```

第 5 行代码检查自定义 URL 是否被定义，如果定义了，则使用 shared application 实例来打开 URL (第 8 行)。openURL: 方法启动应用并将 URL 传入应用。在此过程中，当前的应用被退出。

## 通过自定义 URL Scheme 向应用传递参数

有时你需要通过自定义 URL 向应用中传递参数。让我们看看该如何完成这个工作。

NSURL 作为从一个应用调用另一个的基础，遵循 [RFC 1808](https://tools.ietf.org/html/rfc1808) (Relative Uniform Resource Locators) 标准。 因此你所熟悉的基于网页内容的 URL 格式在这里也适用。

在自定义了 URL scheme 的应用中，app delegate 必须实现以下方法:

```objc
- (BOOL)application:(UIApplication *)application 
  openURL:(NSURL *)url 
  sourceApplication:(NSString *)sourceApplication 
  annotation:(id)annotation
```

从一个应用传递参数到另一个的诀窍是通过 URL。例如，假设我们使用以下的 URL scheme，想传递一个名为 "token"的参数和一个标识注册状态的标志，我们可以像这样创建一个 URL:

```objc
NSString *customURL = @"iOSDevTips://?token=123abct&registered=1";
```

在 web 开发中，字符串 *?token=123abct&registered=1* 被称作查询询串(query string).

在被调用(设置了自定义 URL)的应用的 app delegate 中，获取参数的代码如下:

```objc
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url
        sourceApplication:(NSString *)sourceApplication annotation:(id)annotation
{
  NSLog(@"Calling Application Bundle ID: %@", sourceApplication);
  NSLog(@"URL scheme:%@", [url scheme]);
  NSLog(@"URL query: %@", [url query]);
 
  return YES;
}
```

以上代码在应用被调用时的输出为:


```
Calling Application Bundle ID: com.3Sixty.CallCustomURL
URL scheme:iOSDevTips
URL query: token=123abct&registered=1
```

注意 "Calling Application Bundle ID"，你可以用这个来确保只有你定义的应用可以与你的应用直接交互。

让我们改变一下代码，来验证发起调用的应用的 Bundle ID 是否合法:

```objc
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url
        sourceApplication:(NSString *)sourceApplication annotation:(id)annotation
{
  // Check the calling application Bundle ID
  if ([sourceApplication isEqualToString:@"com.3Sixty.CallCustomURL"])
  {
    NSLog(@"Calling Application Bundle ID: %@", sourceApplication);
    NSLog(@"URL scheme:%@", [url scheme]);
    NSLog(@"URL query: %@", [url query]);
 
    return YES;
  }
  else
    return NO;
}
```

有一点要特别注意，你不能阻止其他应用通过自定义 URL scheme 调用你的应用，然而你可以跳过后续的操作并返回 NO，就像上面的代码那样。也就是说，如果你想阻止其它应用调用你的应用，创建一个与众不同的 URL scheme。尽管这不能保证你的应用不会被调用，但至少大大降低了这种可能性。

## 自定义 URL Scheme 示例工程

我意识到按照本文的每一步做下来还是有一点复杂的。我做好了两个非常基础的 iOS 应用，一个自定义了 URL scheme，另一个则去调用它，并传递了一个比较短的参数列表(query string)。这些是体验自定义 URL 的很好的入门点。

* [Download Xcode project for app with Custom URL scheme](http://iosdevelopertips.com/downloads/#customURLScheme)
* [Download Xcode project for app to call custom URL scheme](http://iosdevelopertips.com/downloads/#customURLScheme)

## 其它资源

[How to Properly Validate URL Parameters](https://developer.apple.com/library/ios/documentation/Security/Conceptual/SecureCodingGuide/Articles/ValidatingInput.html#//apple_ref/doc/uid/TP40007246)
[URL Scheme Reference Docs](https://developer.apple.com/library/ios/featuredarticles/iPhoneURLScheme_Reference/Introduction/Introduction.html#//apple_ref/doc/uid/TP40007899)