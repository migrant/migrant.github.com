---
layout: post
title: "理解Frame"
date: 2013-12-04 13:36
comments: true
categories: iOS 翻译
---

本文由 [**Migrant**](http://migrant.github.io/) 翻译自 [Understanding Frame](http://macoscope.com/blog/understanding-frame/)，转载请注明出处。

Frame是布局的核心。每个开发者都使用frame定位和改变`UIView`和`CALayer`的大小。在本文中我将把焦点集中在`CALayer`上，因为它是`UIView`的底层实现，`view.frame`简单的返回了`view.layer.frame`。此外，我不会讨论`setFrame:`方法。虽然看起来范围十分有限，但实际上有许多有趣的事情在平凡又古老的`frame` getter方法中发生。

<!--more-->

## Frame依赖于什么

众所周知，`frame`是一个派生属性，实际上它基于一些其他的属性。实际上在计算frame值的时候会参考4个(!)属性:`bounds`，`anchorPoint`，`transform`，和`position`。

我们从`bounds`开始。bounds很棘手，它混合了层的内部和外部。`bounds.size`定义了层本身的面积，声明了它所存在的区域。设置`masksToBounds`为`YES`会把所有子层超出bounds范围的部分裁掉。另一方面，`bounds`的`origin`属性并不影响层本身的布局；然而它会影响它内部的子层的布局方式。`bounds.origin`定义了层内部坐标系的原点。

这里有一个例子展示了`bounds.origin`如何工作。例如我们定义`bounds.origin`为`CGPointMake (20.0f, 30.0f)`

![bounds.origin](/images/posts/2013-12-04-understanding-frame-01.png "bounds.origin")

如何定义本地坐标系？只要把层的左上角放到`bounds.origin`上就行了。

![bounds.origin](/images/posts/2013-12-04-understanding-frame-02.png "bounds.origin")

`anchorPoint`是一个稍微有点不同的讨厌鬼。首先，它的值标准化为0.0-1.0的范围内。获得以"点"为单位的值需要用`bounds.size`乘以标准化的值。更重要的是，`anchorPoint`定义了应用变换的坐标系的原点。

![anchorPoint](/images/posts/2013-12-04-understanding-frame-03.png "anchorPoint")

变换具有相同`bounds`但有不同`anchorPoint`的层(蓝色)会有很大区别(灰色)。

`position`是最简单的一个概念。它定义了经过`bounds.size`，`anchorPoint`和`transform`的混合后，添加到层中的最终位置。

## 精度的快速讨论

在写这篇博客的时候，我留意到有时我的计算结果和CoreAnimation返回的计算结果相比有所出入。有可能是我计算错误或者有精度问题。我理所当然的首先检查了精度问题。幸运的是我的直觉是正确的。`CGFloat`在32位架构上是一个`float`的类型定义(在64位架构上是`double`)，而似乎CoreAnimation并没有理会`CGFloat`的实际类型而在内部直接使用了`double`。

要证实这个猜测并不困难。使用[Hooper](http://www.hopperapp.com/)工具检查`CALayer`的`frame`getter方法的执行内容，我发现了一个叫做`mat4_apply_to_rect`的函数。然后我在这里设置了一个符号断点，实际上也就是在`CA::Mat4Impl::mat4_apply_to_rect(double const*, double*)`和`CA::Mat4Impl::mat4_apply_to_rect(float const*, float*)`上分别设置了一个断点，以确定哪一个函数被执行。当在设备上运行代码的时候，断点停在了参数是`double`的函数中，即使使用的是32位ARM架构的iPhone。

在一些极端情况下，使用`float`和`double`的差异是显而易见的。然而因为我们的目标是对CoreAnimation进行逆向工程并得到完全相同的结果，所以我们也使用`double`。我们定义一些和CoreGraphics中相同的非常简单的结构体。

```objc
typedef struct MCSDoublePoint {
  double x, y;
} MCSDoublePoint;

typedef struct MCSDoubleSize {
  double width, height;
} MCSDoubleSize;

typedef struct MCSDoubleRect {
  MCSDoublePoint origin;
  MCSDoubleSize size;
} MCSDoubleRect;
```

值得注意的是在64位iOS设备上，我们精心构建的`struct`会变得多余，因为在该架构上，`CGPoint`，`CGSize`和`CGRect`本来就是用`doubles`的。

## 变换

在深入分析frame之前，我们先了解一下变换。虽然`CALayer`使用的是一个完整的4×4的矩阵模拟`CATransform3D`，但它对计算`frame`的目的真的没有影响。所以，我们把焦点集中在`CGAffineTransform`上，它可以用每个人都喜欢的`CATransform3DGetAffineTransform`方法从`CATransform3D`中简单获得。

让我们从点开始，使用仿射变换来变换点是入门级的袋鼠:

```objc
MCSDoublePoint MCSDoublePointApplyTransform(MCSDoublePoint point, CGAffineTransform t)
{
  MCSDoublePoint p;
  p.x = (double)t.a * point.x + (double)t.c * point.y + t.tx;
  p.y = (double)t.b * point.x + (double)t.d * point.y + t.ty;
  return p;
}
```

上面的代码实现基于`CGPointApplyAffineTransform`，从根本上来讲是一个3x3的变换矩阵乘一个三维向量。

![equation](/images/posts/2013-12-04-understanding-frame-04.gif "equation")

这个矩阵被`CGAffineTransform`的值填充，被乘的向量由点的x坐标，y坐标和`1.0`组成，让结果向量从矩阵中也得到转换过的元素。

通过点变换，我们很容易变换矩形。通过变换矩形的顶点并用直线连接它们创建一个平行四边形(通常可以是任意四边形)。
但这并不是`CGRectApplyAffineTransform`的如何工作的。这个函数接收一个`CGRect`参数并返回一个`CGRect`。正如头文件`CGAffineTransform.h`中的注释声明的:

> 通常来说因为仿射变换并不保护矩形，这个函数返回一个最小的包括经过变换的`rect`的四个顶点的矩形。

读过这个以后，使用double再现`CGRectApplyAffineTransform`变得相对直接:

```objc
MCSDoubleRect MCSDoubleRectApplyTransform(MCSDoubleRect rect, CGAffineTransform transform)
{
  double xMin = rect.origin.x;
  double xMax = rect.origin.x + rect.size.width;
  double yMin = rect.origin.y;
  double yMax = rect.origin.y + rect.size.height;

  MCSDoublePoint points[4] = {
    [0] = MCSDoublePointApplyTransform((MCSDoublePoint){xMin, yMin}, transform),
    [1] = MCSDoublePointApplyTransform((MCSDoublePoint){xMin, yMax}, transform),
    [2] = MCSDoublePointApplyTransform((MCSDoublePoint){xMax, yMin}, transform),
    [3] = MCSDoublePointApplyTransform((MCSDoublePoint){xMax, yMax}, transform),
  };

  double newXMin =  INFINITY;
  double newXMax = -INFINITY;
  double newYMin =  INFINITY;
  double newYMax = -INFINITY;

  for (int i = 0; i < 4; i++) {
    newXMax = MAX(newXMax, points[i].x);
    newYMax = MAX(newYMax, points[i].y);
    newXMin = MIN(newXMin, points[i].x);
    newYMin = MIN(newYMin, points[i].y);
  }

  MCSDoubleRect result = {newXMin, newYMin, newXMax - newXMin, newYMax - newYMin};

  return result;
}
```

我们计算了四个顶点的坐标，变换它们并且得到`x`和`y`的极值。

## 计算Frame

我们通过努力了解了每一个影响frame的因素，现在，获得frame将会变得很有趣:

* 定义一个面积为`bounds.size`的矩形

![](/images/posts/2013-12-04-understanding-frame-05.png)

* 计算该矩形内的`anchorPoint`位置

![](/images/posts/2013-12-04-understanding-frame-06.png)

* 将矩形放入坐标系内，`anchorPoint`作为坐标系的原点

![](/images/posts/2013-12-04-understanding-frame-07.png)

* 应用任何你实施的变换，保持一个"包含了经过转换的顶点的最小矩形"

![](/images/posts/2013-12-04-understanding-frame-08.png)

* 根据`position`移动`anchorPoint`

![](/images/posts/2013-12-04-understanding-frame-09.png)

* 灰色的就是结果矩形

实现这些操作的代码如下:

```objc
- (CGRect)frameWithBounds:(CGRect)bounds anchorPoint:(CGPoint)anchorPoint transform:(CATransform3D)transform position:(CGPoint)position
{
  MCSDoubleRect rect;

  rect.size.width = bounds.size.width;
  rect.size.height = bounds.size.height;
  rect.origin.x = (double)-bounds.size.width * anchorPoint.x;
  rect.origin.y = (double)-bounds.size.height * anchorPoint.y;

  rect = MCSDoubleRectApplyTransform(rect, CATransform3DGetAffineTransform(transform));

  rect.origin.x += position.x;
  rect.origin.y += position.y;

  return CGRectMake(rect.origin.x, rect.origin.y, rect.size.width, rect.size.height);
}
```

虽然代码不多，但利用了我们讨论过的所有概念。

## 这些如何映射到`UIView`

关于`frame`getter方法，`bounds`和`center`，`UIView`并没有做什么工作；它只是简单的各自调用它底层的CALayer的`frame`，`bounds`和`position`方法。

注意`center`到`position`的映射 -- 改变底层`layer`的`anchorPoint`会使`center`不能正确的对应到层的"中心"或者层的边界矩形的"中点"。