---
layout:     post
title:      "iOS 知识小集（横竖屏切换）"
date:       2016-05-20
author:     "Haley_Wong"
catalog:    true
tags:
    - Tools
---

iOS 中横竖屏切换的功能，在开发iOS app中总能遇到。以前看过几次，感觉简单，但是没有敲过代码实现，最近又碰到了，demo尝试了几种情况，这里就做下总结。

**注意**

![横屏两种情况是反的你知道吗？](/img/blogs/orientationchange/img_01.webp)
`UIInterfaceOrientationLandscapeRight`与`UIInterfaceOrientationMaskLandscapeRight`都代表横屏，Home键在右侧的情况；`UIDeviceOrientationLandscapeLeft`则是Home键在左侧。

# 一般情形

**所有界面都支持横竖屏切换**

如果App的所有切面都要支持横竖屏的切换，那只需要勾选【General】 中的【Device Orientation】，选择希望支持的方向即可。

![图中支持竖屏和Home在右侧](/img/blogs/orientationchange/img_01.webp)

如上设置完之后，当设备竖屏的时候，所有的界面都是竖屏显示的；而当设备横屏Home在右侧时，所有的界面会横屏显示。其他方向不支持，界面不会改变。

> 这里有个坑：
在iOS 9 之后横屏时，状态栏会消失。
解决方法：确保plist 中的【View controller-based status bar appearance】为YES，然后重写ViewController的 `- (BOOL)prefersStatusBarHidden` ，返回值是NO。

```
- (BOOL)prefersStatusBarHidden
{
    return NO;
}
```

# 特殊情形

**个别界面固定方向，其他所有界面都支持横竖屏切换**

这种情况，在【General】-->【Device Orientation】中设置好支持的方向后，只需要在这些特殊的视图控制器中重写两个方法：

```
// 支持设备自动旋转
- (BOOL)shouldAutorotate
{
    return YES;
}

/** 
*  设置特殊的界面支持的方向,这里特殊界面只支持Home在右侧的情况
*/
- (UIInterfaceOrientationMask)supportedInterfaceOrientations 
{
    return UIInterfaceOrientationMaskLandscapeRight;
}
```

**个别界面支持横竖屏切换，其他所有界面都固定方向**

可能大多数App会是这种需求，某些特殊界面只能横屏，如视频播放类App。
这里有两种处理方式：

**方式一** 

在【General】-->【Device Orientation】中设置好需要支持的所有方向。然后使用一个基类控制器，在基类控制器中重写两个控制横竖屏的方法：

```
// 支持设备自动旋转
- (BOOL)shouldAutorotate
{
    return YES;
}

// 支持竖屏显示
- (UIInterfaceOrientationMask)supportedInterfaceOrientations
{
    return UIInterfaceOrientationMaskPortrait;
}
```

再然后，特殊的界面上再重写这俩方法，让其可以自动切换方向。

```
// 如果需要横屏的时候，一定要重写这个方法并返回NO
- (BOOL)prefersStatusBarHidden
{
    return NO;
}

// 支持设备自动旋转
- (BOOL)shouldAutorotate
{
    return YES;
}

// 支持横屏显示
- (UIInterfaceOrientationMask)supportedInterfaceOrientations
{
    // 如果该界面需要支持横竖屏切换
    return UIInterfaceOrientationMaskLandscapeRight | UIInterfaceOrientationMaskPortrait;
    // 如果该界面仅支持横屏
   // return UIInterfaceOrientationMaskLandscapeRight；
}
```

**方式二**

用方式一的方法，还需要借助一个基类，所有的控制器都要继承这个基类，太麻烦？

另一种方式，是借助通知来控制界面的横竖屏切换。还是整个App中大部分界面都是竖屏，某个界面可以横竖屏切换的情况。

首先，在【General】-->【Device Orientation】设置仅支持竖屏，like this:

![Device Orientation](/img/blogs/orientationchange/img_03.webp)

然后在特殊的视图控制器里的ViewDidLoad中注册通知：

```
    [[UIDevice currentDevice] beginGeneratingDeviceOrientationNotifications];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(deviceOrientationDidChange) name:UIDeviceOrientationDidChangeNotification object:nil];
```

通知方法的实现过程：
```
- (void)deviceOrientationDidChange
{
    NSLog(@"deviceOrientationDidChange:%ld",(long)[UIDevice currentDevice].orientation);
    if([UIDevice currentDevice].orientation == UIDeviceOrientationPortrait) {
        [[UIApplication sharedApplication] setStatusBarOrientation:UIInterfaceOrientationPortrait];
        [self orientationChange:NO];
        //注意： UIDeviceOrientationLandscapeLeft 与 UIInterfaceOrientationLandscapeRight
    } else if ([UIDevice currentDevice].orientation == UIDeviceOrientationLandscapeLeft) {
        [[UIApplication sharedApplication] setStatusBarOrientation:UIInterfaceOrientationLandscapeRight];
        [self orientationChange:YES];
    }
}

- (void)orientationChange:(BOOL)landscapeRight
{
    if (landscapeRight) {
        [UIView animateWithDuration:0.2f animations:^{
            self.view.transform = CGAffineTransformMakeRotation(M_PI_2);
            self.view.bounds = CGRectMake(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT);
        }];
    } else {
        [UIView animateWithDuration:0.2f animations:^{
            self.view.transform = CGAffineTransformMakeRotation(0);
            self.view.bounds = CGRectMake(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT);
        }];
    }
}
// 用到的两个宏：
    #define SCREEN_WIDTH ([UIScreen mainScreen].bounds.size.width)
    #define SCREEN_HEIGHT ([UIScreen mainScreen].bounds.size.height)
```

> 最重要的一点: 需要重写如下方法，并且返回NO。

```
- (BOOL)shouldAutorotate
{
    return NO;
}
```

这样，在设备出于横屏时，界面就会变成横屏，设备处于竖屏时，界面就会变成竖屏。

# 填坑

* 上面方式二，因为【General】-->【Device Orientation】因为只设置了竖屏，所以当横屏时，如果有键盘弹出，键盘是竖屏时的样式。
解决办法：在【General】-->【Device Orientation】中加上横屏时的方向。
* 如果VieController 是放在UINavigationController或者UITabBarController中，需要重写它们的方向控制方法。

```
// UINavigationController：
- (BOOL)shouldAutorotate
{
    return [self.topViewController shouldAutorotate];
}

- (UIInterfaceOrientationMask)supportedInterfaceOrientations
{
    return [self.topViewController supportedInterfaceOrientations];
}

// UITabBarController:
- (BOOL)shouldAutorotate
{
    return [self.selectedViewController shouldAutorotate];
}

- (UIInterfaceOrientationMask)supportedInterfaceOrientations
{
    return [self.selectedViewController supportedInterfaceOrientations];
}

```
如果想要点击某个按钮之后，强制将竖屏显示的界面变成横屏呢？

有人可能会想到这样写:

```
// 横屏
- (IBAction)landscapAction:(id)sender {
    [[UIApplication sharedApplication] setStatusBarOrientation:UIInterfaceOrientationLandscapeRight];
    [self orientationChange:YES];
}
```

但是按照上面的写法，会导致返回到之前的界面时，视图方向错误，即使返回前执行如下代码：

```
[[UIApplication sharedApplication] setStatusBarOrientation:UIInterfaceOrientationPortrait];
[self orientationChange:NO];
```
也没有作用，下面是在开源工程中无意看到的写法：

```
// 横屏
- (IBAction)landscapAction:(id)sender {
    [self interfaceOrientation:UIInterfaceOrientationLandscapeRight];
}

// 竖屏
- (IBAction)portraitAction:(id)sender {
    [self interfaceOrientation:UIInterfaceOrientationPortrait];
}

- (void)interfaceOrientation:(UIInterfaceOrientation)orientation
{
    if ([[UIDevice currentDevice] respondsToSelector:@selector(setOrientation:)]) {
        SEL selector             = NSSelectorFromString(@"setOrientation:");
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:[UIDevice instanceMethodSignatureForSelector:selector]];
        [invocation setSelector:selector];
        [invocation setTarget:[UIDevice currentDevice]];
        int val                  = orientation;
        [invocation setArgument:&val atIndex:2];
        [invocation invoke];
    }
}
```

上面的方法会将设备的方向强制设置为某个方向，然后再监控设备方向改变的通知，即可实现横竖屏切换。
这里有一个用JS 和原生item 控制横竖屏切换的Demo。[地址](https://github.com/Haley-Wong/ChangeOrientation)
这是效果图：

![横竖屏切换.gif](/img/blogs/orientationchange/img_04.webp)

横竖屏切换总结就到这了，Have Fun!

