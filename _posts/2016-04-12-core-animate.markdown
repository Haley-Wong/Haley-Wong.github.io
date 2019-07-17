---
layout:     post
title:      "iOS动画三板斧（二）--CoreAnimation动画"
date:       2016-04-12
author:     "Haley_Wong"
catalog:    true
tags:
    - 动画
---

# 介绍
第二板斧就是用的最多的`CoreAnimation`动画库，简称是CA，所以动画类都是CA开头。所有的动画类都在 `QuartzCore` 库中，在iOS7之前使用需要`#import <QuartzCore/QuartzCore.h>`,iOS7之后系统已经将其自动导入了。`CoreAnimation`动画都是作用在`layer`上。

先来看下动画类的层级关系：
![动画层级结构.png](/img/blogs/core-animate/img_01.webp)

关于上图中的层级结构只需要了解一下，用的多了，自然就记住了。本篇只讲述CABasicAnimation、CAKeyframeAnimation、CAAnimationGroup的使用。

# 使用
上面讲了CA动画都是作用在Layer上，而CA动画中修改的也是Layer的动画属性，可以产生动画的layer属性也有`Animatable`标识。

### 1.CABasicAnimation
CABasicAnimation动画主要是设置某个动画属性的初始值fromValue和结束值toValue，来产生动画效果。

先上个示例代码，将一个视图往上移动一段距离：
![animation.gif](/img/blogs/core-animate/img_02.webp)

```
    CABasicAnimation *postionAnimation = [CABasicAnimation animationWithKeyPath:@"position.y"];
    postionAnimation.duration = 1.0f;
    postionAnimation.fromValue = @(self.squareView.center.y);
    postionAnimation.toValue = @(self.squareView.center.y - 300);
    postionAnimation.removedOnCompletion = NO;
    postionAnimation.delegate = self;
    postionAnimation.autoreverses = YES;
    postionAnimation.fillMode = kCAFillModeForwards;
    postionAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    [self.squareView.layer addAnimation:postionAnimation forKey:@"posstionAnimation"];
```

> * 动画的创建使用`animationWithKeyPath:`,因为使用的keyPath所以动画属性或者其结构体中元素都可以产生动画。
* `duration` 动画的时长。
* `fromValue`和`toValue` 是CABasicAnimation的属性，都是id类型的，所以要将基本类型包装成对象。
* `removedOnCompletion` 决定动画执行完之后是否将该动画的影响移除，默认是YES,则layer回到动画前的状态。
* `fillMode` 是个枚举值（四种），当removedOnCompletion设置为NO之后才会起作用。可以设置layer是保持动画开始前的状态还是动画结束后的状态，或是其他的。
* `autoreverses` 表示动画结束后是否 backwards(回退) 到动画开始前的状态。可与上面两个属性组合出不同效果。
* `timingFunction` 动画的运动是匀速线性的还是先快后慢等，类似UIView动画的opitions。另外，CAMediaTimingFunction 方法可以自定义。
* `delegate` 代理，两个动画代理方法：`- (void)animationDidStart:(CAAnimation *)anim;` 和`- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag;`
* `- (void)addAnimation:(CAAnimation *)anim forKey:(nullable NSString *)key;` 给某个layer添加动画，与之对应的移除某个动画是`- (void)removeAnimationForKey:(NSString *)key;`
* 还有一些其他的属性，就不一一介绍了，可以在使用的使用去.h文件中查看。

通过与代理方法结合使用，可以用多段动画组合成一个完整动画。
### 2.CAKeyframeAnimation
CAKeyframeAnimation我们一般称为关键帧动画，主要是利用其`values`属性，设置多个关键帧属性值，来产生动画。

先上一个示例代码，将一个视图先放大，再缩小，再放大的动画：
![animation.gif](/img/blogs/core-animate/img_03.webp)
```
    CAKeyframeAnimation *keyAnimation = [CAKeyframeAnimation animationWithKeyPath:@"transform"];
    keyAnimation.duration = 1.0f;
    keyAnimation.beginTime = CACurrentMediaTime() + 1.0;
    
    CATransform3D transform1 = CATransform3DMakeScale(1.5, 1.5, 0);
    CATransform3D transform2 = CATransform3DMakeScale(0.8, 0.8, 0);
    CATransform3D transform3 = CATransform3DMakeScale(3, 3, 0);
    
    keyAnimation.values = @[[NSValue valueWithCATransform3D:transform1],[NSValue valueWithCATransform3D:transform2],[NSValue valueWithCATransform3D:transform3]];
    keyAnimation.keyTimes = @[@0,@0.5,@1];
    keyAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    keyAnimation.removedOnCompletion = NO;
    keyAnimation.fillMode = kCAFillModeForwards;
    [_someView.layer addAnimation:keyAnimation forKey:nil];
```
> * `beginTime` 也是CAAnimation类的属性，可以设置动画延迟多久执行，示例代码是延迟1秒执行。
* `values` 是`CAKeyframeAnimation`的属性，设置keyPath属性在几个关键帧的值，也是id类型的。
* `keyTimes` 也是`CAKeyframeAnimation`的属性，每个值对应相应关键帧的时间比例值。
* `timingFunctions` 也是`CAKeyframeAnimation`的属性，对应每个动画段的动画过渡情况；而timingFunction是CAAnimation的属性。

### 3.CAAnimationGroup
CAAnimationGroup的用法与其他动画类一样，都是添加到layer上，比CAAnimation多了一个`animations`属性。

先看示例代码，动画效果是视图一边向上移动，一边绕Y轴旋转：

![animation.gif](/img/blogs/core-animate/img_04.webp)
```
    CABasicAnimation *rotationYAnimation = [CABasicAnimation animationWithKeyPath:@"transform.rotation.y"];
    rotationYAnimation.fromValue = @0;
    rotationYAnimation.toValue = @(M_PI);
    rotationYAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    CABasicAnimation *postionAnimation = [CABasicAnimation animationWithKeyPath:@"position.y"];
    postionAnimation.fromValue = @(_markView.center.y);
    postionAnimation.toValue = @(_markView.center.y - 100);
    postionAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    
    CAAnimationGroup *animationGroup = [CAAnimationGroup animation];
    animationGroup.duration = kUpDuration;
    animationGroup.removedOnCompletion = NO;
    animationGroup.fillMode = kCAFillModeForwards;
    animationGroup.delegate = self;
    animationGroup.animations = @[rotationYAnimation, postionAnimation];
    
    [_markView.layer addAnimation:animationGroup forKey:kJumpAnimation];
```
> CAAnimationGroup的`animations`中可以放其他任何动画类（包括CAAnimationGroup），需要注意的是animations里的动画设置了duration之后动画可能会有不同，一般里面不设置，在最外层设置group的duration即可。

接上面示例之后的动画，实现视图继续绕Y轴旋转90°，下落回原处：
```
- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag
{
    if ([anim isEqual:[_someView.layer animationForKey:@"jumpUp"]]) {
        CABasicAnimation *rotationYAnimation = [CABasicAnimation animationWithKeyPath:@"transform.rotation.y"];
        rotationYAnimation.fromValue = @(M_PI_2);
        rotationYAnimation.toValue = @(M_PI);
        rotationYAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
        
        CABasicAnimation *postionAnimation = [CABasicAnimation animationWithKeyPath:@"position.y"];
        postionAnimation.fromValue = @(_someView.center.y - 14);
        postionAnimation.toValue = @(_someView.center.y);
        postionAnimation.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
        
        CAAnimationGroup *animationGroup = [CAAnimationGroup animation];
        animationGroup.duration = 0.256;
        animationGroup.removedOnCompletion = NO;
        animationGroup.fillMode = kCAFillModeForwards;
        animationGroup.delegate = self;
        animationGroup.animations = @[rotationYAnimation, postionAnimation];
        
        [_someView.layer addAnimation:animationGroup forKey:@"jumpDown"];
    }
}
```
该动画可以用来实现跳起来旋转时，修改某个图片视图或者按钮的图片等。

### 4.CATransition（2016-04-22补充）
CATransition一般来做转场动画。先上gif动画效果
![transition动画.gif](/img/blogs/core-animate/img_05.webp)
```
  //修改视图的背景色
    _someView.backgroundColor = [UIColor greenColor];
    CATransition *animation = [CATransition animation];
    animation.duration = 0.5;
    /* 这里可设置的参数有：kCATransitionFade、kCATransitionPush、kCATransitionReveal、kCATransitionMoveIn、
    "cube"、"suckEffect"、"oglFlip"、"rippleEffect"、"pageCurl"、"pageUnCurl"、"cameraIrisHollowOpen"、
     "cameraIrisHollowClose"，这些都是动画类型
    */
    animation.type = @"cube";
    // 动画执行的方向，kCATransitionFromRight、kCATransitionFromLeft、kCATransitionFromTop、kCATransitionFromBottom
    animation.subtype = kCATransitionFromRight;
    animation.timingFunction = UIViewAnimationOptionCurveEaseInOut;
    [_someView.layer addAnimation:animation forKey:nil];
    //也可以写这里
//    _someView.backgroundColor = [UIColor greenColor];
```
只需要在动画开始前或者动画开始后替换掉视图上显示的内容即可。
示例代码可能与gif图不太一致，因为gif图是从其他demo中录制下来的。

gif图来自青玉伏案的demo:[他的文章有更详细的demo讲解，地址在这里](http://www.cnblogs.com/ludashi/p/4160208.html)

# 附加
附加的内容是关于CALayer和UIBezierPath。个人觉得理解了UIBezierPath和CALayer，才能更好的理解CoreAnimation动画。

### 1.UIBezierPath
UIBezierPath主要是用来绘制路径的，分为一阶、二阶.....n阶。一阶是直线，二阶以上才是曲线。而最终路径的显示还是得依靠CALayer。用CoreGraphics将路径绘制出来，最终也是绘制到CALayer上。

![贝塞尔曲线.png](/img/blogs/core-animate/img_06.webp)

> 
* 方法一：构造bezierPath对象，一般用于自定义路径。
* 方法二：绘制圆弧路径，参数1是中心点位置，参数2是半径，参数3是开始的弧度值，参数4是结束的弧度值，参数5是是否顺时针(YES是顺时针方向，NO逆时针)。
* 方法三：根据某个路径绘制路径。
* 方法四：根据某个CGRect绘制内切圆或椭圆（CGRect是正方形即为圆，为长方形则为椭圆）。
* 方法五：根据某个CGRect绘制路径。
* 方法六：绘制带圆角的矩形路径，参数2哪个角，参数3，横、纵向半径。
* 方法七：绘制每个角都是圆角的矩形，参数2是半径。

自定义路径时常用的API:
```
- (void)moveToPoint:(CGPoint)point; // 移到某个点
- (void)addLineToPoint:(CGPoint)point; // 绘制直线
- (void)addCurveToPoint:(CGPoint)endPoint controlPoint1:(CGPoint)controlPoint1 controlPoint2:(CGPoint)controlPoint2; //绘制贝塞尔曲线
- (void)addQuadCurveToPoint:(CGPoint)endPoint controlPoint:(CGPoint)controlPoint; // 绘制规则的贝塞尔曲线
- (void)addArcWithCenter:(CGPoint)center radius:(CGFloat)radius startAngle:(CGFloat)startAngle endAngle:(CGFloat)endAngle clockwise:(BOOL)clockwise
// 绘制圆形曲线
- (void)appendPath:(UIBezierPath *)bezierPath; // 拼接曲线
```
### 如果将路径显示的图案显示到视图上呢？

有三种方式：1、直接使用UIBezierPath的方法；2、使用CoreGraphics绘制；3、利用CAShapeLayer绘制。

示例代码如下，绘制一个右侧为弧型的视图：
![animation.gif](/img/blogs/core-animate/img_07.webp)
```
- (void)drawRect:(CGRect)rect
{
    UIColor *fillColor = [UIColor colorWithRed:0.0 green:0.722 blue:1.0 alpha:1.0];
    
    UIBezierPath *bezierPath = [UIBezierPath bezierPath];
    [bezierPath moveToPoint:CGPointMake(0, 0)];
    [bezierPath addLineToPoint:CGPointMake(rect.size.width - spaceWidth, 0)];
    [bezierPath addQuadCurveToPoint:CGPointMake(rect.size.width - spaceWidth, rect.size.height) controlPoint:CGPointMake(rect.size.width - spaceWidth + _deltaWith, rect.size.height * 0.5)];
    [bezierPath addLineToPoint:CGPointMake(0, rect.size.height)];
    [bezierPath addLineToPoint:CGPointMake(0, 0)];
    [bezierPath closePath];
    
    // 1、bezierPath方法
//    [fillColor setFill];
//    [bezierPath fill];
    
    // 2、使用CoreGraphics
//    CGContextRef ctx = UIGraphicsGetCurrentContext();
//    CGContextAddPath(ctx, bezierPath.CGPath);
//    CGContextSetFillColorWithColor(ctx, fillColor.CGColor);
//    CGContextFillPath(ctx);
    
    // 3.CAShaperLayer
    [self.layer.sublayers makeObjectsPerformSelector:@selector(removeFromSuperlayer)];
    CAShapeLayer *shapeLayer = [CAShapeLayer layer];
    shapeLayer.path = bezierPath.CGPath;
    shapeLayer.fillColor = fillColor.CGColor;
    [self.layer addSublayer:shapeLayer];
}
```

![进度条.gif](/img/blogs/core-animate/img_08.webp)
上图这样的视图是用UIBezierPath用多个CAShapeLayer制作出来的，而动画效果只需要改变进度的layer的strokeEnd和修改下面代表水面进度的视图位置即可。

动画的组合也可以有多种方式，组合动画的示例代码：
```
- (void)setProgress:(CGFloat)progress animated:(BOOL)animated duration:(NSTimeInterval)duration
{
    CGFloat tempPro = progress;
    if (tempPro > 1.0) {
        tempPro = 1.0;
    } else if (progress < 0.0){
        tempPro = 0.0;
    }
    _progress = tempPro;
    
    CABasicAnimation *pathAniamtion = [CABasicAnimation animationWithKeyPath:@"strokeEnd"];
    pathAniamtion.duration = duration;
    pathAniamtion.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut];
    pathAniamtion.fromValue = [NSNumber numberWithFloat:0.0f];
    pathAniamtion.toValue = [NSNumber numberWithFloat:_progress];
    pathAniamtion.autoreverses = NO;
    [_progressLayer addAnimation:pathAniamtion forKey:nil];
    
    // 水位上升的动画
    if (!_showSolidAnimation) {
        return;
    }
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        _imageView.transform = CGAffineTransformIdentity;
        [UIView animateWithDuration:duration animations:^{
            CGRect rect = _imageView.frame;
            CGFloat dy = rect.size.height * progress;
            _imageView.transform = CGAffineTransformMakeTranslation(0, -dy);
        }];
    });
}
```

在用自定义的CAShapeLayer做动画时，建议在动画开始前先将动画属性与最终的属性值一致，再开始动画，不要使用removedOnCompletion控制最终的状态，这在WWDC苹果这么建议。

另外一个酷炫动画的实现都是多种简单动画的组合！

关于CoreAnimation动画就先介绍这么多吧，Have fun!


