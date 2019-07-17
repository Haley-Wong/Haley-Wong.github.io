---
layout:     post
title:      "iOS动画（补充）--特殊Layer动画"
date:       2016-04-14
author:     "Haley_Wong"
catalog:    true
tags:
    - 动画
---

iOS中有一些特殊的layer，也可以做一些动画效果，本文就补充两个可以做动画效果的layer: CAEmitterLayer 和 CAReplicatorLayer。

### CAEmitterLayer
Emitter 发射器，因为可以用它来做爆炸、发射、下雪等效果。
比如，这个下雪效果：
![下雪.gif](/img/blogs/special-animate/img_01.webp)
```
- (void)setEmitter
{
    CAEmitterLayer *snowEmitter = [CAEmitterLayer layer];
    //发射点的位置
    snowEmitter.emitterPosition = CGPointMake(self.view.bounds.size.width * 0.5, -30);
    //
    snowEmitter.emitterSize = CGSizeMake(self.view.bounds.size.width * 2.0, 0.0);
    snowEmitter.emitterShape = kCAEmitterLayerLine;
    snowEmitter.emitterMode = kCAEmitterLayerOutline;
    
    snowEmitter.shadowColor = [UIColor whiteColor].CGColor;
    snowEmitter.shadowOffset = CGSizeMake(0.0, 1.0);
    snowEmitter.shadowRadius = 0.0;
    snowEmitter.shadowOpacity = 1.0;
    
    CAEmitterCell *snowCell = [CAEmitterCell emitterCell];
    
    snowCell.birthRate = 1.0; //每秒出现多少个粒子
    snowCell.lifetime = 120.0; // 粒子的存活时间
    snowCell.velocity = -10; //速度
    snowCell.velocityRange = 10; // 平均速度
    snowCell.yAcceleration = 2;//粒子在y方向上的加速度
    snowCell.emissionRange = 0.5 * M_PI; //发射的弧度
    snowCell.spinRange = 0.25 * M_PI; // 粒子的平均旋转速度
    snowCell.contents = (id)[UIImage imageNamed:@"snow"].CGImage;
    snowCell.color = [UIColor colorWithRed:0.6 green:0.658 blue:0.743 alpha:1.0].CGColor;
 
    snowEmitter.emitterCells = @[snowCell];
    
    [self.view.layer insertSublayer:snowEmitter atIndex:0];
}
```
喷射效果：
![喷射.gif](/img/blogs/special-animate/img_02.webp)
主要代码：
```
- (void)setEmitter
{
    CAEmitterLayer *snowEmitter = [CAEmitterLayer layer];
    //发射点的位置
    snowEmitter.emitterPosition = self.view.center;
    //
    snowEmitter.emitterSize = CGSizeMake(10.0, 0.0);
    snowEmitter.emitterShape = kCAEmitterLayerLine;
    snowEmitter.emitterMode = kCAEmitterLayerOutline;
    
    CAEmitterCell *snowCell = [CAEmitterCell emitterCell];
    
    snowCell.birthRate = 50.0;
    snowCell.lifetime = 10.0;
    snowCell.velocity = 40;
    snowCell.velocityRange = 10;
    snowCell.yAcceleration = 2;
    snowCell.emissionRange =  M_PI / 9;
    snowCell.scale = 0.1; //缩小比例
    snowCell.scaleRange = 0.08;// 平均缩小比例
    snowCell.contents = (id)[UIImage imageNamed:@"Sparkle"].CGImage;
    snowCell.color = [UIColor colorWithRed:0.6 green:0.658 blue:0.743 alpha:1.0].CGColor;
    
    snowEmitter.emitterCells = @[snowCell];
    
    [self.view.layer insertSublayer:snowEmitter atIndex:0];
}
```
烟花爆炸效果：
![烟花.gif](/img/blogs/special-animate/img_03.webp)
```
- (void)fireworks
{
    CAEmitterLayer *emitter = [CAEmitterLayer layer];
    emitter.frame = self.view.bounds;
    [self.view.layer addSublayer:emitter];
    
    //configure emitter
    emitter.renderMode = kCAEmitterLayerAdditive;
    emitter.emitterPosition = self.view.center;
    emitter.emitterSize = CGSizeMake(25, 0);
    
    //create a particle template
    CAEmitterCell *cell = [[CAEmitterCell alloc] init];
    cell.contents = (id)[UIImage imageNamed:@"Sparkle"].CGImage;
    cell.birthRate = 150;
    cell.lifetime = 5.0;
    cell.alphaSpeed = -0.4;
    cell.velocity = 50;
    cell.velocityRange = 50;
    cell.scale = 0.1;
    cell.scaleRange = 0.08;
    cell.emissionRange = M_PI * 2.0;
    
    emitter.emitterCells = @[cell];
}
```
按钮点击效果：

![按钮点击.gif](/img/blogs/special-animate/img_04.webp)
初始设置：
```
CAEmitterCell *emitterCell = [CAEmitterCell emitterCell];
    emitterCell.name = @"explosion";
    emitterCell.alphaRange = 0.2;
    emitterCell.alphaSpeed = -1.0;
    
    emitterCell.lifetime = 0.7;
    emitterCell.lifetimeRange = 0.2;
    emitterCell.birthRate = 0;
    emitterCell.velocity = 40;
    emitterCell.velocityRange = 10.0;
    emitterCell.scale = 0.05;
    emitterCell.scaleRange = 0.02;
    emitterCell.contents = (id)[UIImage imageNamed:@"Sparkle"].CGImage;
    
    _explosionLayer = [CAEmitterLayer layer];
    _explosionLayer.name = @"emitterLayer";
    _explosionLayer.emitterShape = kCAEmitterLayerCircle;
    _explosionLayer.emitterMode = kCAEmitterLayerOutline;
    _explosionLayer.emitterPosition = CGPointMake(CGRectGetMidX(self.bounds), CGRectGetMidY(self.bounds));
    _explosionLayer.emitterSize = CGSizeMake(25, 0);
    _explosionLayer.emitterCells = @[emitterCell];
    _explosionLayer.renderMode = kCAEmitterLayerOldestFirst;
    [self.layer addSublayer:_explosionLayer];
    
    self.emitterCells = @[emitterCell];
```
点击执行动画
```
- (void)startAnimate
{
    if (_animating) {
        return;
    }
    _animating = YES;
    [self performSelector:@selector(explode) withObject:nil afterDelay:0.2];
}

- (void)explode
{
    _explosionLayer.beginTime = CACurrentMediaTime();
    [_explosionLayer setValue:@500 forKeyPath:@"emitterCells.explosion.birthRate"];
    [self performSelector:@selector(stop) withObject:nil afterDelay:0.1];
}

- (void)stop
{
    _animating = NO;
    [_explosionLayer setValue:@0 forKeyPath:@"emitterCells.explosion.birthRate"];
}
```
### CAReplicatorLayer
CAReplicatorLayer 可以多次拷贝某个layer，然后重新布局，实现动画效果。
多个视图绕Y轴旋转，可以做loading效果：

![旋转.gif](/img/blogs/special-animate/img_05.webp)

主要代码：
```
- (void)setup
{
    CGFloat margin = 5.0;
    CGFloat width = self.layer.bounds.size.width;
    CGFloat dotW = (width - 2 * margin) / 3;
    
    CAShapeLayer *shapeLayer = [CAShapeLayer layer];
    shapeLayer.frame = CGRectMake(0, (width - dotW) * 0.5, dotW, dotW);
    shapeLayer.path = [UIBezierPath bezierPathWithRect:CGRectMake(0, 0, dotW, dotW)].CGPath;
    shapeLayer.fillColor = [UIColor redColor].CGColor;
    
    CAReplicatorLayer *replicatorLayer = [CAReplicatorLayer layer];
    replicatorLayer.frame = CGRectMake(0, 0, width, width);
    replicatorLayer.instanceDelay = 0.1;
    replicatorLayer.instanceCount = 3;
    CATransform3D transform = CATransform3DMakeTranslation(margin + dotW, 0, 0);

    replicatorLayer.instanceTransform = transform;
    [replicatorLayer addSublayer:shapeLayer];
    [self.layer addSublayer:replicatorLayer];
    
    CABasicAnimation *basicAnima = [CABasicAnimation animationWithKeyPath:@"transform"];
    basicAnima.fromValue = [NSValue valueWithCATransform3D:CATransform3DRotate(CATransform3DIdentity, 0, 0, 1.0, 0)];
    basicAnima.toValue = [NSValue valueWithCATransform3D:CATransform3DRotate(CATransform3DIdentity, M_PI, 0, 1.0, 0)];
    basicAnima.repeatCount = HUGE;
    basicAnima.duration = 0.6;
    
    [shapeLayer addAnimation:basicAnima forKey:@"scaleAnimation"];
    
}
```
水波纹效果，类似QQ语音通话和支付宝的咻咻咻

![水波纹.gif](/img/blogs/special-animate/img_06.webp)
```
- (void)setup
{
    self.layer.backgroundColor = [UIColor clearColor].CGColor;
    CAShapeLayer *pulseLayer = [CAShapeLayer layer];
    pulseLayer.frame = self.layer.bounds;
    pulseLayer.path = [UIBezierPath bezierPathWithOvalInRect:pulseLayer.bounds].CGPath;
    pulseLayer.fillColor = [UIColor redColor].CGColor;
    pulseLayer.opacity = 0.0;
    
    CAReplicatorLayer *replicatorLayer = [CAReplicatorLayer layer];
    replicatorLayer.frame = self.bounds;
    replicatorLayer.instanceCount = 8;
    replicatorLayer.instanceDelay = 0.5;
    [replicatorLayer addSublayer:pulseLayer];
    [self.layer addSublayer:replicatorLayer];

    CABasicAnimation *opacityAnima = [CABasicAnimation animationWithKeyPath:@"opacity"];
    opacityAnima.fromValue = @(0.3);
    opacityAnima.toValue = @(0.0);
    
    CABasicAnimation *scaleAnima = [CABasicAnimation animationWithKeyPath:@"transform"];
    scaleAnima.fromValue = [NSValue valueWithCATransform3D:CATransform3DScale(CATransform3DIdentity, 0.0, 0.0, 0.0)];
    scaleAnima.toValue = [NSValue valueWithCATransform3D:CATransform3DScale(CATransform3DIdentity, 1.0, 1.0, 0.0)];
    
    CAAnimationGroup *groupAnima = [CAAnimationGroup animation];
    groupAnima.animations = @[opacityAnima, scaleAnima];
    groupAnima.duration = 4.0;
    groupAnima.autoreverses = NO;
    groupAnima.repeatCount = HUGE;
    [pulseLayer addAnimation:groupAnima forKey:@"groupAnimation"];
}
```

![animation.gif](/img/blogs/special-animate/img_07.webp)

```
- (void)setup
{
    CGFloat margin = 5.0;
    CGFloat cols = 3;
    CGFloat width = self.layer.bounds.size.width;
    CGFloat dotW = (width - margin * (cols - 1)) / cols;
    
    CAShapeLayer *dotLayer = [CAShapeLayer layer];
    dotLayer.frame = CGRectMake(0, 0, dotW, dotW);
    dotLayer.path = [UIBezierPath bezierPathWithOvalInRect:CGRectMake(0, 0, dotW, dotW)].CGPath;
    dotLayer.fillColor = [UIColor redColor].CGColor;
    
    CAReplicatorLayer *replicatorLayerX = [CAReplicatorLayer layer];
    replicatorLayerX.frame = CGRectMake(0, 0, width, width);
    replicatorLayerX.instanceDelay = 0.3;
    replicatorLayerX.instanceCount = cols;
    [replicatorLayerX addSublayer:dotLayer];
    
    CATransform3D transform = CATransform3DTranslate(CATransform3DIdentity, dotW + margin, 0, 0);
    replicatorLayerX.instanceTransform = transform;
    transform = CATransform3DScale(transform, 1, -1, 0);
    
    CAReplicatorLayer *replicatorLayerY = [CAReplicatorLayer layer];
    replicatorLayerY.frame = CGRectMake(0, 0, width, width);
    replicatorLayerY.instanceDelay = 0.3;
    replicatorLayerY.instanceCount = 3;
    [replicatorLayerY addSublayer:replicatorLayerX];
    
    CATransform3D transforY = CATransform3DTranslate(CATransform3DIdentity, 0, dotW + margin, 0);
    replicatorLayerY.instanceTransform = transforY;
    
    
    [self.layer addSublayer:replicatorLayerY];
//    [self.layer addSublayer:replicatorLayerX];
    
    //添加动画
    CABasicAnimation *opacityAnima = [CABasicAnimation animationWithKeyPath:@"opacity"];
    opacityAnima.fromValue = @(0.3);
    opacityAnima.toValue = @(0.3);
    
    CABasicAnimation *scaleAnima = [CABasicAnimation animationWithKeyPath:@"transform"];
    scaleAnima.fromValue = [NSValue valueWithCATransform3D:CATransform3DScale(CATransform3DIdentity, 1.0, 1.0, 0.0)];
    scaleAnima.toValue = [NSValue valueWithCATransform3D:CATransform3DScale(CATransform3DIdentity, 0.2, 0.2, 0.0)];
    
    CAAnimationGroup *groupAnima = [CAAnimationGroup animation];
    groupAnima.animations = @[opacityAnima, scaleAnima];
    groupAnima.duration = 1.0;
    groupAnima.autoreverses = YES;
    groupAnima.repeatCount = HUGE;
    [dotLayer addAnimation:groupAnima forKey:@"groupAnimation"];
}
```

