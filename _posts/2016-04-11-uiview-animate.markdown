---
layout:     post
title:      "iOS动画三板斧（一）--UIView动画"
date:       2016-04-11
author:     "Haley_Wong"
catalog:    true
tags:
    - 动画
---

# 前言
iOS 精致的app，离不开酷炫合宜的动画。而iOS中的动画实现也有多种不同的方式。今天就来介绍一下iOS中的动画。本篇是第一篇，就讲一下最简单的动画实现方式，初学动画，简单的动画一般都是用这种方式来实现的。

# UIView 动画

UIView动画就是利用UIView的API来实现动画效果。而利用UIView API也可以分为两种，一种block形式，一种多API组合。

### 一、block形式的UIView 动画
常用的block UIView 动画方法有如下几个：
![API.png](/img/blogs/uiview-animate/img_01.jpeg)
按顺序分别编号①、②、③、④、⑤。

① 关键帧动画，先上示例代码，将一个按钮从原来尺寸放大到1.5倍，在缩小到0.8，再恢复到原始大小：
![animation.gif](/img/blogs/uiview-animate/img_02.gif)

```
[UIView animateKeyframesWithDuration:0.5 delay:0 options:UIViewKeyframeAnimationOptionLayoutSubviews animations:^{
        [UIView addKeyframeWithRelativeStartTime:0 relativeDuration:1/3.0 animations:^{
            customBtn.transform = CGAffineTransformMakeScale(1.5, 1.5);
        }];
        [UIView addKeyframeWithRelativeStartTime:1/3.0 relativeDuration:1/3.0 animations:^{
            customBtn.transform = CGAffineTransformMakeScale(0.8, 0.8);
        }];
        [UIView addKeyframeWithRelativeStartTime:2/3.0 relativeDuration:1/3.0 animations:^{
            customBtn.transform = CGAffineTransformIdentity;
        }];
    } completion:nil];
```

其中第一个参数是动画执行的时长（单位：秒）；第二个参数是多久后执行这个动画（单位：秒）；第三个参数是个枚举类型，动画的类型；第四个参数就是动画的block，设置关键帧动画的几个关键帧，属性变化信息，第五个参数是动画执行完毕后的回调block。

而内部的方法是为关键帧动画添加关键帧，属性信息。第一个参数，是这一关键帧开始的时间（0-1.0之间，是相对外面方法duration的比例值，即0.5）；第二个参数是该关键帧变化占用的时间（也是duration的比例）；第三个参数，是到达该关键帧时的属性值。

② 最简单的UIView动画API

将试图移动并放大至某个frame示例代码：

![animation2.gif](/img/blogs/uiview-animate/img_03.gif)
```
[UIView animateWithDuration:3.0 animations:^{
        squareView.frame = CGRectMake(50, 50, 100, 100);
    }];
```
第一个参数是动画执行时长（单位：秒）；第二个参数就是动画的block，属性变化信息的最终值。

③ 最常用的UIView动画API

先上示例代码，将试图移出屏幕外之后，将其删除：

```
 [UIView animateWithDuration:3.0 animations:^{
        squareView.frame = CGRectMake(-100, -100, 100, 100);
    } completion:^(BOOL finished) {
        [squareView removeFromSuperview];
    }];
```
③ 比 ② 多了一个动画执行完毕后的回调。可以在执行完动画后，移除某个试图或者再次调用动画API, 执行一个新的动画。

比如这样：
```
[UIView animateWithDuration:3.0 animations:^{
        squareView.frame = CGRectMake(50, 50, 100, 100);
    } completion:^(BOOL finished) {
        [UIView animateWithDuration:1.0 animations:^{
            squareView.frame = CGRectMake(50, 50, 200, 200);
        }];
    }];
```

④ 与 ③ 类似，多了一个延迟时间（单位：秒）和动画类型；延迟时间是表示多久之后执行该动画，动画执行类型有样式，比如速度先快后慢，开始快中间慢结束时再快，匀速等等。大家可以移除设置不同的枚举值来比较。

示例代码：
```
    [UIView animateWithDuration:3.0 delay:0 options:UIViewAnimationOptionCurveLinear animations:^{
        squareView.frame = CGRectMake(-50, -50, 100, 100);
    } completion:^(BOOL finished) {
        [squareView removeFromSuperview];
    }];
```

⑤ 弹性动画，像橡皮筋一样，将试图改变至属性所设置的值后，会有一个回弹效果。根据设置的初速度和阻尼系数慢慢停止，最终停留在属性所设置的值的状态。

![animation.gif](/img/blogs/uiview-animate/img_04.gif)
示例代码：
```
[UIView animateWithDuration:0.7 delay:0.0 usingSpringWithDamping:0.5 initialSpringVelocity:0.9 options:UIViewAnimationOptionBeginFromCurrentState|UIViewAnimationOptionAllowUserInteraction animations:^{
        _someView.frame = CGRectMake(50, 50, 100, 100);
    } completion:^(BOOL finished) {
        
    }];
```
相比于方法④，多了一个damping阻尼系数和初速度initialSpringVelocity，当阻尼系统大于等于1时，会平稳减速，不会有震荡效果，如果小于1，则会来回震荡，直到停止。

> * 在animations block中只能修改UIView的部分属性，产生动画效果。而可以产生动画效果的属性，苹果在其注释中都有标记 `animatable`。
* 在animations block中可以修改多个视图的动画属性，或者修改某个视图的多个动画属性。

![animatable property.png](/img/blogs/uiview-animate/img_05.png)

### 二、一般形式的UIView动画
先介绍常用的API：

![API介绍.png](/img/blogs/uiview-animate/img_06.jpeg)
上图中的API没有截完整，还有几个设置动画参数的方法，大家可以去探索一下。

示例代码：
```
    [UIView beginAnimations:@"centerAnimation" context:nil];
    [UIView setAnimationCurve:UIViewAnimationCurveLinear];
    [UIView setAnimationDuration:2.0];
    [UIView setAnimationDelegate:self];
    [UIView setAnimationWillStartSelector:@selector(animationWillStart)];
    
    self.squareView.center = CGPointMake(50, 50);
    [UIView commitAnimations];
```

>  需要注意的有几点：
* 要先设置完动画参数，然后在修改视图的动画属性，否则动画参数可能不起作用。因此，修改视图动画属性的语句写在 commitAnimatons上一行。
* 如果要在动画开始前，动画结束后做一些事情要先设置delegate，然后在设置好动画开始前，动画开始后要调用的selector。

动画第一板斧就到这里，have fun!


