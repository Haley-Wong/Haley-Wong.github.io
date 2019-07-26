---
layout:     post
title:      "iOS 知识小集（Status Bar变换）"
date:       2016-05-19
author:     "Haley_Wong"
catalog:    true
tags:
    - Tools
---

# 背景
iOS 中经常会有需要在某个界面改变状态栏颜色或者某个界面隐藏状态栏的需求。而改变状态栏颜色和控制状态栏显示和隐藏的API，在iOS 的不同版本中也发生了很多变化。

# iOS 7以前
在iOS 7之前，状态栏是不占视图位置的。每个控制器中的根view都是从屏幕的Y轴20px处开始显示的。所以那个时候整个app状态栏的风格，一般只在plist文件里设置【对应于General中的Status Bar Style】。印象里似乎只有黑白两种风格，已记不清了！😂 😂 😂

![iOS 7以前状态栏设置](/img/blogs/ios_statusbar/img_01.webp)

从API来看，那时候也是支持在代码里修改状态栏的样式以及显示和隐藏的。只是因为状态栏对整个APP的影响不大，所以一般在plist里设置好后，用不着再去修改了。

![API](/img/blogs/ios_statusbar/img_02.webp)

# iOS 7 ~iOS 9
从iOS 7开始系统风格大变样，图标扁平了，状态栏也不在闹独立了。因为状态栏的会受到导航栏或者View背景色的影响，所以状态栏的风格也需要实时调整了。

想要改变状态栏的样式，想要控制状态栏的显示与隐藏，该怎么做呢？

**1. 用UIApplication的API**

首先，需要在plist文件里将【View controller-based status bar appearance】设置为NO，因为它的默认值是YES，然后就可以利用UIApplication 来设置了。

![plist设置](/img/blogs/ios_statusbar/img_03.webp)

先上效果动画：
![状态栏变换.gif](/img/blogs/ios_statusbar/img_04.webp)
再上源码：

```
- (IBAction)changeStatus:(UISegmentedControl *)sender {
    if (sender.selectedSegmentIndex == 0) {
//        [[UIApplication sharedApplication] setStatusBarStyle:UIStatusBarStyleDefault];
        // 带动画效果,动画效果其实就是变换的时间变慢了
        [[UIApplication sharedApplication] setStatusBarStyle:UIStatusBarStyleDefault animated:YES];
    } else {
//        [[UIApplication sharedApplication] setStatusBarStyle:UIStatusBarStyleLightContent];
        // 带动画效果，动画效果其实就是变换的时间变慢了
        [[UIApplication sharedApplication] setStatusBarStyle:UIStatusBarStyleLightContent animated:YES];
    }  
}

- (IBAction)showOrHidden:(UISegmentedControl *)sender {
    if (sender.selectedSegmentIndex == 0) {
        // 第二个参数是个枚举类型
        [[UIApplication sharedApplication] setStatusBarHidden:NO withAnimation:UIStatusBarAnimationSlide];
        // 不带动画效果
//        [[UIApplication sharedApplication] setStatusBarHidden:NO];
    } else {
        [[UIApplication sharedApplication] setStatusBarHidden:YES withAnimation:UIStatusBarAnimationSlide];
        // 不带动画效果
//        [[UIApplication sharedApplication] setStatusBarHidden:YES];
    }
}
```

**2. 重写ViewController方法**

首先，要确保plist文件中【View controller-based status bar appearance】为YES，没有添加这个key的时候，默认是YES。

![plist设置](/img/blogs/ios_statusbar/img_05.webp)

然后在视图控制器中，重写如下三个方法即可：

![要重写的方法](/img/blogs/ios_statusbar/img_06.webp)

因为这三个方法都有默认值，如果我们要的状体栏样式什么的跟默认值效果一致，则不需要重写；如果不想要默认的效果，则直接在这三个方法里return 相应的值即可。你不必三个方法都重写，看实际情况。例如，我想要在这个界面时状态栏为白色，状态栏不隐藏，那么我只用重写`-preferredStatusBarStyle`，like this：
```
- (UIStatusBarStyle)preferredStatusBarStyle
{
    return UIStatusBarStyleLightContent;
}
```

因为我这里需要做一个切换所以，我首先定义了两个property：
```
@property (assign, nonatomic)   UIStatusBarStyle    statusBarStyle; /**< 状态栏样式 */
@property (assign, nonatomic)   BOOL    statusBarHidden;    /**< 状态栏隐藏 */
```
然后改变UISegmentedControl的值时，在响应的Action方法里改变上述property的值，再调用
`-setNeedsStatusBarAppearanceUpdate`即可。
示例代码：
```
#pragma mark - ViewController方式
- (IBAction)changeStyle:(UISegmentedControl *)sender {
    if (sender.selectedSegmentIndex == 0) {
        _statusBarStyle = UIStatusBarStyleDefault;
    } else {
        _statusBarStyle = UIStatusBarStyleLightContent;
    }
    [self setNeedsStatusBarAppearanceUpdate];
}

- (IBAction)statusShowOrHidden:(UISegmentedControl *)sender {
    if (sender.selectedSegmentIndex == 0) {
        _statusBarHidden = NO;
    } else {
        _statusBarHidden = YES;
    }
    [self setNeedsStatusBarAppearanceUpdate];
}

#pragma mark - 需要重写的几个状态栏方法
/**
 *  控制状态栏的样式
 *  要刷新状态栏，让其重新执行该方法需要调用{-setNeedsStatusBarAppearanceUpdate}
 *
 *  @return 将要显示的状态栏样式
 */
- (UIStatusBarStyle)preferredStatusBarStyle
{
    return _statusBarStyle;
}

/**
 *  状态栏显示还是隐藏
 *  要刷新状态栏，让其重新执行该方法需要调用{-setNeedsStatusBarAppearanceUpdate}
 *
 *  @return BOOL值
 */
- (BOOL)prefersStatusBarHidden
{
    return _statusBarHidden;
}

/**
 *  状态栏改变的动画，这个动画只影响状态栏的显示和隐藏
 *
 *  @return 动画效果
 */
- (UIStatusBarAnimation)preferredStatusBarUpdateAnimation
{
    return UIStatusBarAnimationSlide;
}
```

![效果gif](/img/blogs/ios_statusbar/img_07.webp)
# iOS 9 之后

如上面第二张图所示,UIApplication的控制状态栏的方法，在iOS 9之后被弃用了。
所以iOS 9之后尽量使用重写ViewController方法的方式吧。

# 注意点

**情形一**

如果我们使用UINavigationController，会发现在原来的ViewController里修改状态栏的style不起作用了，但是控制状态栏的显示和隐藏依然OK。但是使用UITabBarController依然正常，状态栏不受UITabBarController影响。

重写UINavigationController的三个方法：
```
- (UIStatusBarStyle)preferredStatusBarStyle
{
    NSLog(@"导航栏-%s",__func__);
    return [self.topViewController preferredStatusBarStyle];
}

- (UIStatusBarAnimation)preferredStatusBarUpdateAnimation
{
    NSLog(@"导航栏-%s",__func__);
    return UIStatusBarAnimationNone;
}

- (BOOL)prefersStatusBarHidden
{
    NSLog(@"导航栏-%s",__func__);
    return NO;
}
```
从打印结果：
```
2016-05-18 13:18:10.248 PractiseProject[3296:112707] 导航栏--[BaseNavigationController preferredStatusBarStyle]
2016-05-18 13:18:10.249 PractiseProject[3296:112707] -[ViewController prefersStatusBarHidden]
2016-05-18 13:18:10.275 PractiseProject[3296:112707] 导航栏--[BaseNavigationController preferredStatusBarStyle]
2016-05-18 13:18:10.275 PractiseProject[3296:112707] -[ViewController prefersStatusBarHidden]
2016-05-18 13:18:10.275 PractiseProject[3296:112707] 导航栏--[BaseNavigationController preferredStatusBarStyle]
2016-05-18 13:18:10.276 PractiseProject[3296:112707] -[ViewController prefersStatusBarHidden]
```
可以看出，只调用了第一个方法。所以我们只需要重写UINavigaitonController的`- preferredStatusBarStyle`即可。

**情形一**

状态栏的样式、是否显示实际上是由顶层window的当前视图控制器决定的。
比如我们在程序入口处创建一个新的window：
```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
    
    self.statusWindow = [[UIWindow alloc] initWithFrame:application.statusBarFrame];
    // 这里设置windowLevel 为UIWindowLevelStatusBar或者UIWindowLevelAlert都可以
    self.statusWindow.windowLevel = UIWindowLevelStatusBar;
    // 颜色必须为clearColor，否则会盖住状态栏的区域
    self.statusWindow.backgroundColor = [UIColor clearColor];
    self.statusWindow.rootViewController = [[StatusViewContrller alloc] init];
    self.statusWindow.hidden = NO;
    
    return YES;
}
```
然后在StatusViewContrller中重写如下方法：

```
- (UIStatusBarStyle)preferredStatusBarStyle
{
    return UIStatusBarStyleLightContent;
}

// 如果想要显示状态栏，必须重写这个方法，并return NO
- (BOOL)prefersStatusBarHidden
{
    return NO;
}
```
这样最终状态栏的样式就由StatusViewContrller决定了，而不是由原来的ViewController决定了。

创建顶层window之后，修改状态栏的样式就不方便了。

为了解决这个问题，我们可以将StatusViewContrller弄成单例，然后定义两个property来控制样式和是否隐藏即可。

```
#import <UIKit/UIKit.h>

@interface StatusViewContrller : UIViewController

@property (assign, nonatomic)   UIStatusBarStyle    statusBarStyle; /**< 状态栏样式 */
@property (assign, nonatomic)   BOOL    statusBarHidden; /**< 状态栏隐藏 */

+ (instancetype)sharedInstance;

@end
```

重写两个property的set方法，设置完属性后调用状态栏刷新方法：

```
// 创建单例的关键代码
static id instance = nil;

+ (instancetype)sharedInstance
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    
    return instance;
}

// 这个方法是关键
+ (instancetype)allocWithZone:(struct _NSZone *)zone
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [super allocWithZone:zone];
    });
    
    return instance;
}

// 重写的方法
- (UIStatusBarStyle)preferredStatusBarStyle
{
    return _statusBarStyle;
}

- (BOOL)prefersStatusBarHidden
{
    return _statusBarHidden;
}

// setter 
- (void)setStatusBarStyle:(UIStatusBarStyle)statusBarStyle
{
    _statusBarStyle = statusBarStyle;
    
    [self setNeedsStatusBarAppearanceUpdate];
}

- (void)setStatusBarHidden:(BOOL)statusBarHidden
{
    _statusBarHidden = statusBarHidden;
    
    [self setNeedsStatusBarAppearanceUpdate];
}
```

> 创建了顶层window后，唯一需要注意的是顶层window和其根视图控制器的背景色必须为clearColor。

That's all。Have Fun!

