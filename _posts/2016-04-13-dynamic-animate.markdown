---
layout:     post
title:      "iOS动画三板斧(三)--UIDynamic动画"
date:       2016-04-13
author:     "Haley_Wong"
catalog:    true
tags:
    - 动画
---

终于到了动画三板斧第三篇了，这里用UIDynamic来实现动画。UIDynamic是iOS 7之后新添加的一些物理仿真动画库，包含在`UIKit`框架中。

# 介绍
使用UIDynamic，需要理解几个概念：1、UIDynamicAnimator，2、UIDynamicBehavior，3、UIDynamicItem。
> 
* `UIDynamicAnimator` 相当于动画引擎。它初始化时，需要一个ReferenceView，用它的坐标系统作为参考坐标系。
* `UIDynamicBehavior` 相当于仿真动画体。创建时，需要附带动画将要作用的视图（即UIDynamicItem）,可以传一个包含多个视图的数组。
* `UIDynamicItem` 就是仿真动画将要作用的视图。

常用的UIDynamicBehavior有：
> 
* UIGravityBehavior    重力行为
* UICollisionBehavior  碰撞行为
* UIAttachmentBehavior 附着行为
* UIPushBehavior  推动行为
* UIDynamicItemBehavior 动力行为
* UISnapBehavior 捕获行为

以上每种行为都可以单独使用，也可以组合使用来实现复杂的动画效果。

# 实战

### 创建动画引擎

```
_animator = [[UIDynamicAnimator alloc] initWithReferenceView:self.view];
```
### 给视图添加仿真行为
#### 1.UIGravityBehavior （重力行为）

```
- (void)animateTest
{
    UIGravityBehavior *gravityBehavior = [[UIGravityBehavior alloc] initWithItems:@[_someView]];
    gravityBehavior.gravityDirection = CGVectorMake(0, 1);
    gravityBehavior.magnitude = 2.5;
    [_animator addBehavior:gravityBehavior];
}
```

`gravityDirection`是表示重力方向，这是二维坐标系中的方向，默认是(0.0,1.0)，表示垂直向下，数值越大；数值可以为负，如（0.0，-1.0）就表示重力方向是垂直向上。也可以利用x和y来表示二维坐标系中的任意方向。例如(1.0,1.0)沿右下角45度方向，（1.0,100000)极度接近竖直向下方向。

`magnitude`表示力的系数，正数时，沿`gravityDirection`方向，数值越大，加速度越大；负数时，`gravityDirection`的反方向，数值越小，加速度越大。

#### 2.UICollisionBehavior （碰撞行为） 
在上述代码中，_someView视图会因为重力作用，直接掉出屏幕外。而添加碰撞行为，并设置好碰撞的边界时，_someView会在碰撞边界上回弹直至静止。

```
- (void)animateTest
{
    // 重力行为
    UIGravityBehavior *gravityBehavior = [[UIGravityBehavior alloc] initWithItems:@[_someView]];
    gravityBehavior.gravityDirection = CGVectorMake(0, 1);
    gravityBehavior.magnitude = 2.5;
    [_animator addBehavior:gravityBehavior];
    
    // 碰撞行为
    UICollisionBehavior *collisionBehavior = [[UICollisionBehavior alloc] initWithItems:@[_someView]];
    //设置碰撞边界有如下几张方式：
    //1.设置碰撞边界为referenceView的边界。
//    collisionBehavior.translatesReferenceBoundsIntoBoundary = YES;
    // 2.设置碰撞边界以referenceView作为参考，设置insets作为边界。
    [collisionBehavior setTranslatesReferenceBoundsIntoBoundaryWithInsets:UIEdgeInsetsMake(0, 0, 20, 0)];
    // 3.用两个点的连线作为碰撞边界
//    [collisionBehavior addBoundaryWithIdentifier:@"pointBoundary" fromPoint:CGPointMake(0, 300) toPoint:CGPointMake(320, 600)];
    // 4.以某个贝塞尔曲线作为碰撞边界
//    [collisionBehavior addBoundaryWithIdentifier:@"pathBoundary" forPath:_bezierPath];
    [_animator addBehavior:collisionBehavior];
}
```
![添加碰撞行为后.gif](/img/blogs/dynamic-animate/img_01.webp)

#### 3.UIAttachmentBehavior （附着行为）
附着行为一般都是添加手势，让视图跟着手势移动，因为一般都是与手势搭配使用。

```
- (void)panAction:(UIPanGestureRecognizer *)panGesture
{
    CGPoint location = [panGesture locationInView:self.view];
    if (panGesture.state == UIGestureRecognizerStateBegan) {
        _attachmentBehavior = [[UIAttachmentBehavior alloc] initWithItem:_someView attachedToAnchor:location];
        [_animator addBehavior:_attachmentBehavior];
    } else if (panGesture.state == UIGestureRecognizerStateChanged) {
        _attachmentBehavior.anchorPoint = location;
    } else if (panGesture.state == UIGestureRecognizerStateEnded) {
        [_animator removeBehavior:_attachmentBehavior];
    }
}
```

![附着行为.gif](/img/blogs/dynamic-animate/img_02.webp)

#### 4.UIPushBehavior（推动行为）

推动行为的mode有连个值，一个是持续的推力，一个是初始推力。`pushDirection`与重力的参数类似，表示二维坐标系中推力的方向。`magnitude`系数，影响加速度。

下面的动画，是给视图一个向上的推力，然后在重力的作用下运动到最高点后下落，最后在设置好的碰撞边界处慢慢趋于静止。

```
- (void)animateTest
{
    // 推动行为
    UIPushBehavior *pushBehavior = [[UIPushBehavior alloc] initWithItems:@[_someView] mode:UIPushBehaviorModeInstantaneous];
    pushBehavior.pushDirection = CGVectorMake(0, - 80.0);
    pushBehavior.magnitude = 2.0;
    [_animator addBehavior:pushBehavior];
    
    // 重力行为
    UIGravityBehavior *gravityBehavior = [[UIGravityBehavior alloc] initWithItems:@[_someView]];
    gravityBehavior.gravityDirection = CGVectorMake(0, 1);
    gravityBehavior.magnitude = 2.5;
    [_animator addBehavior:gravityBehavior];

    // 碰撞行为
    UICollisionBehavior *collisionBehavior = [[UICollisionBehavior alloc] initWithItems:@[_someView]];
    //设置碰撞边界为referenceView的边界。
    collisionBehavior.translatesReferenceBoundsIntoBoundary = YES;
    [_animator addBehavior:collisionBehavior];
}
```
![推动行为.gif](/img/blogs/dynamic-animate/img_03.webp)

#### 5.UIDynamicItemBehavior （动力行为）

因为可以设置摩擦力、弹力、密度、阻力等参数，在模拟视图运动的能量损失。

```
- (void)animateTest
{
    //动力行为
    UIDynamicItemBehavior *itemBehavior = [[UIDynamicItemBehavior alloc] initWithItems:@[_someView]];
    itemBehavior.elasticity = 0.6; //弹力
    itemBehavior.friction = 1;     //摩擦力
    itemBehavior.density = 10;    //密度
    itemBehavior.resistance = 10; // 阻力
    itemBehavior.allowsRotation = YES; //允许旋转
    [_animator addBehavior:itemBehavior];
    
    // 推动行为
    UIPushBehavior *pushBehavior = [[UIPushBehavior alloc] initWithItems:@[_someView] mode:UIPushBehaviorModeInstantaneous];
    pushBehavior.pushDirection = CGVectorMake(0, - 80.0);
    pushBehavior.magnitude = 2.0;
    [_animator addBehavior:pushBehavior];

    // 碰撞行为
    UICollisionBehavior *collisionBehavior = [[UICollisionBehavior alloc] initWithItems:@[_someView]];
    //设置碰撞边界为referenceView的边界。
    collisionBehavior.translatesReferenceBoundsIntoBoundary = YES;
    [_animator addBehavior:collisionBehavior];
}
```

给视图一个初始向上的推力，然后在摩擦力，阻力等参数下慢慢减速至静止。遇到边界碰撞时会有能量损失。效果图如下:
![动力行为.gif](/img/blogs/dynamic-animate/img_04.webp)

#### 6.UISnapBehavior （捕获行为）
捕获行为，是移动视图到某个位置，然后到达后，有一个摆动效果。

```
- (void)animateTest
{
    // 捕获行为
    UISnapBehavior *snapBehavior = [[UISnapBehavior alloc] initWithItem:_someView snapToPoint:self.view.center];
    snapBehavior.damping = 0.1; // 0.0~~1.0，阻尼系数，影响能量损失。
    [_animator addBehavior:snapBehavior];
}
```
![捕获行为.gif](/img/blogs/dynamic-animate/img_05.webp)

动画的触发，我这里是给self.view添加了一个点击手势和pan手势。

```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    UITapGestureRecognizer *gesture = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(animateTest)];
    [self.view addGestureRecognizer:gesture];
    
    UIPanGestureRecognizer *panGesture = [[UIPanGestureRecognizer alloc] initWithTarget:self action:@selector(panAction:)];
    [self.view addGestureRecognizer:panGesture];
    
    _someView = [[UIView alloc] initWithFrame:CGRectMake(100, 200, 50, 50)];
    _someView.backgroundColor = [UIColor redColor];
    [self.view addSubview:_someView];

    _animator = [[UIDynamicAnimator alloc] initWithReferenceView:self.view];
}
```
看一个斯坦福公开课中，显示的动画，也是用动态仿真动画实现的。

![示例动画.gif](/img/blogs/dynamic-animate/img_06.webp)

多种仿真效果组合，可以组合出酷炫的动画效果。大家可以多尝试组合以及参数变化来做酷炫的动画，Have fun!

