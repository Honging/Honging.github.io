---
layout: post
title: Keyframes--为移动客户端提供可扩展的高质量动画
date: 2016-11-29
categories: blog
tags: [translate]
description: 首篇翻译，必定有误！勿信
---

![Keyframes](/docs/images/doc-logo.png)
# Keyframes

*仅翻译，由于知识所限肯定有错误。*
*原文地址：[https://github.com/facebookincubator/Keyframes](https://github.com/facebookincubator/Keyframes)*

Keyframes不仅具有如ExtendScript脚本从After Effects文件中提取图片动画数据一样的优点，而且在Android和iOS系统中均表现良好。Keyframes可以通过使用复杂的图形和路径   曲线实现基于矢量的高质量动画效果，而且文件也非常小。

## 用法

### 在After Effects下开发动画

关于使用Keyframes库开发动画效果的更多内容请戳：[Keyframes After Effects Guidelines](https://github.com/facebookincubator/Keyframes/blob/master/docs/AfterEffectsGuideline.md)。

### 提取图片数据

使用提取脚本需要安装**Adobe After Effects**和**Adobe ExtendScript Toolkit**。如果已经安装Keyframes JSON文件，那么再安装相应iOS和Android库即可。

了解在AE comp中运行ExtendScript脚本的详细步骤请移步其[相关说明](https://github.com/facebookincubator/Keyframes/blob/master/Keyframes%20After%20Effects%20Scripts)。

### iOS的实现

#### 建立表现

使用在**提取图片数据**步骤生成的JSON blob提供的deserializer（解串行器）创建一个`KFVector`模型对象。如果你的JSON blob在资料目录中存在，它应该是这样的：

    NSString *filePath = [[NSBundle bundleForClass:[self class]] pathForResource:@"asset_name" ofType:@"json" inDirectory:nil];
    NSData *data = [NSData dataWithContentsOfFile:filePath];
    NSDictionary *vectorDictionary = [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:nil];
    KFVector *vector = KFVectorFromDictionary(vectorDictionary);

然后用这个`KFVector`创建一个`KFVectorLayer`文件，`KFVectorLayer`用法与一般`CALayer`相同。

    KFVectorLayer *layer = [KFVectorLayer layer];
    // Set a non-zero layer frame before setting the KFVector is required.
    layer.frame = $AnyNonZeroCGRect$;
    layer.faceModel = vector;

如果你更喜欢使用`UI视图`，你也可以用`KFVector`创建一个`KFVectorView`。

    KFVectorView *view = [[KFVectorView alloc] initWithFrame:$AnyNonZeroCGRect$ faceVector:vector];

#### 运行！

使用`KFVectorLayer`中的`startAnimaiton`，`pauseAnimation`，`resumeAnimation`和`seekToProgress`来控制动画。

    // Animation will start from beginning.
    [layer startAnimation];
    
    // Pause the animation at current progress.
    [layer pauseAnimation];
    
    // Resume the animation from where we paused last time.
    [layer resumeAnimation];
    
    // Seek to a given progress in range [0, 1]
    [layer seekToProgress:0.5]; // seek to the mid point of the animation

### Android的实现

#### 建立表现

使用在**提取图片数据**步骤生成的JSON blob提供的deserializer（解串行器）创建一个`KFImage`模型对象。如果你的JSON blob在资源目录中存在，它应该是这样的：

    InputStream stream = getResources().getAssets().open("asset_name");
    KFImage kfImage = KFImageDeserializer.deserialize(stream);

现在用此`KFImage`创建一个KeyframesDrawable（可拖拽Keyframe）对象，该对象的使用同一般Drawable类。
在此强烈建议在任何视窗下都使用软件层展示Keyframes动画。

    Drawable kfDrawable = new KeyframesDrawableBuilder().withImage(kfImage).build();

    ImageView imageView = (ImageView) findViewById(R.id.some_image_view);
    imageView.setLayerType(View.LAYER_TYPE_SOFTWARE, null);
    imageView.setImageDrawable(kfDrawable);

#### 运行！

使用开始动画重复播放动画，停止动画使动画在当前动画结束后停止播放。

    // Starts a loop that progresses animation and invalidates the drawable.
    kfDrawable.startAnimation();

    // Stops the animation when the current animation ends.
    kfDrawable.stopAnimationAtLoopEnd();

## 理解Keyframes模型对象

### 图片

在Keyframes中一张可扩展的动态图片由一系列的重要的域共同描述。在高层级，一张图片包含如何扩展（`canvas—size`）图片和如何以准确速度重复播放动画（`frame_rate`，`frame_count`）等信息。动画本身不必绑定到离散的图片获取帧率`frame_rate`上，因为Keyframes渲染库支持少量框架。另外一张图片`Image`也包含一系列描述将被变成不同形状的属性`Feature`，而动画组`Animation Group`用于描述那些马上可以被应用于多个属性或其他动画中的转换。注意这些`Image`和`Animation Group`的全局变量。

接下来让我们分解这张简单的星星在圆中不断变化大小的图片。获取的动画为24帧，框架的数字显示在左上角，星星的大小显示在下面。

![Star, Real Time](/docs/images/doc-star-realtime.gif)

让我们降慢速度到一帧一帧动作。

![Star, Slowwwww](/docs/images/doc-star-slow.gif)

### 属性

属性指image的任何独立视觉对象。更重要的是它同能够描述任意给定时间下属性的形状的SVG类型命令一样，包含拖动命令。一个属性可能属于一个更大的动画组，同时也包含包括私有化的`STROKE_WIDTH`动作在内的其自身的私有化动作。使用相同Keyframe和分类模式下描述的属性的形状随着时间而改变。

#### 形状

形状指所有构造在一起描述线性的或闭合的可被填涂或勾画的形状的线性动作命令的表单。这些命令是一系列紧挨着的`Move`,`Line`,`Quadratic`和`Cubic`命令。

下面是上图的一些重要形状，并标有相关顶点（方点）和控制点（圆点）。

![Circle Shape](/docs/images/doc-circle-shape.png)
![Star Shape](/docs/images/doc-star-shape.png)

### 动画

#### 过渡

Keyframes渲染库支持通用基于矩阵的过渡动作，如`SCALE`，`ROTATE`和`TRANSLATE`。对于私有化的`Feature`，额外添加使用非矩阵属性`STROKE_WIDTH`。`Animation`可以属于私有`Feature`或作为更大的`Animation Group`的一部分。

#### 动画过程

动画过渡的属性值和重复播放动画时的衔接由两个关键域`Keyframe`和`Timing Curves`决定。通过两者结合使用，我们可以计算出动画中任意过渡时刻下的属性值。

**Keyframes**是拥有特定属性值的动画的指定帧。举个例子，一个10帧的变化图形大小的动画，我们想要在开始和结束图形大小都是100%，那么就要在第0帧和第10帧都是100%，而在第7帧达到最大150%。在这个例子中，图形`SCALE`过渡的关键帧将是`[0, 7, 10]`，其对应值为`[100%, 150%, 100%]`。

**Timing Curves**描述相邻帧过渡的速率。每一条计时曲线均有从点`[0, 0]`到点`[1, 1]`的3次方赛贝尔曲线方程约束。X轴为从原始帧到目标帧过程中的时间值，Y轴描述给定时刻下的值的变化量：`origValue + (destValue - origValue) * Y`。

比如前面的变化大小的星星动态图，其大小随时间变化而变化的曲线图如下图，附有顶点处值和计时曲线取值。

![Scale Animation Curve](/docs/images/doc-scale-curve.png)

### 尝试结合使用前面所有属性

结合一张`Image`对象的域和过程值，我们可以建立任意给定时刻下`Image`的图形，对图形使用任何过渡效果或者复原特定帧的`Image`对象。因为动画有具有弹性的`timing_curves`驱动，一个Keyframe `Image`对象并不局限于必须使用整帧播放，但是需要注意的是所有过程值都必须在相关帧数下赋予。这意味着一个10帧的动画必须在`[0..10]`范围内赋予所有值。

## 贡献

阅读[`CONTRIBUTING.md`](/CONTRIBUTING.md)查看如何帮助我们改进此库！
![](http://cnfeat.qiniudn.com/signitrue-2015-03-05.png)








