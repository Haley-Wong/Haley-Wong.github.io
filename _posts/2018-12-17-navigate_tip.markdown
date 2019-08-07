---
layout:     post
title:      "iOS导航栏控制Tips"
date:       2018-12-17
author:     "Haley_Wong"
catalog:    true
tags:
    - Tools
---

许久不写UI，对UI的很多东西都生疏了，最近使用导航栏的各种场景做一些总结。

## 1.导航栏的显示与隐藏

导航栏的显示与隐藏，分两种情况：

* 1.从不显示导航栏的页面push到显示导航栏的页面。
* 2.从显示导航栏的页面Push到不显示导航栏的页面。

> 注意：
> 1.如果导航栏不显示时，系统的侧滑返回功能无效。
> 2.虽然侧滑返回功能无效，但是导航栏的 `.interactivePopGestureRecognizer.delegate`还是存在的。

针对以上两种情况分别处理，整个Push过程都假设是从A页面跳转到B页面

### 1.1 从不显示导航栏的页面Push到显示导航栏的页面。
关于导航栏的显示，是否顺滑，是通过如下两个方法来控制。

```
// 不显示动画，导航栏显示就比较突兀
[self.navigationController setNavigationBarHidden:YES];

// 显示动画，在侧滑时，导航栏显示就比较顺滑
[self.navigationController setNavigationBarHidden:YES animated:YES];
```
所以，做法是

A页面：

```
- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    
    [self.navigationController setNavigationBarHidden:YES animated:YES];
}
```

B页面：

```
- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    
    [self.navigationController setNavigationBarHidden:NO animated:YES];
}
```

### 1.2 从显示导航栏的页面跳转到不显示导航栏的页面

这种情况的做法如下：
A页面：

```
- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    [self.navigationController setNavigationBarHidden:NO animated:YES];
}
```

B页面：

```
// 在页面将要出现时，记录原始侧滑手势代理对象，并将手势代理设置为当前页面
- (void)viewWillAppear:(BOOL)animated
{
    [super viewWillAppear:animated];
    
    self.interactivePopDelegate = self.navigationController.interactivePopGestureRecognizer.delegate;
    self.navigationController.interactivePopGestureRecognizer.delegate = self;
    
    [self.navigationController setNavigationBarHidden:YES animated:YES];
}

// 在页面消失时，还原侧滑手势代理对象
- (void)viewDidDisappear:(BOOL)animated
{
    [super viewDidDisappear:animated];
    
    self.navigationController.interactivePopGestureRecognizer.delegate = self.interactivePopDelegate;
    self.interactivePopDelegate = nil;
}

// 实现手势代理，为了防止影响其他手势，可以判断一下手势类型
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer
{
    if ([gestureRecognizer isKindOfClass:[UIScreenEdgePanGestureRecognizer class]]) {
        return YES;
    }
    ...... 其他手势的处理
    
    
    return NO;
}

```

## 2.统一重写导航栏返回按钮

有时候，我们可能需要统一工程中的返回按钮样式，比如都是 `箭头+返回` 或者都是 `箭头`。
方案有两种：

* 1.创建一个BaseViewController，然后统一设置navigationItem.leftBarButtonItem。
* 2.重写导航控制器的Push方法，在push之前，设置`navigationItem.backBarButtonItem`。

> 注意：
> 如果重写了导航栏的`leftBarButtonItem`，那么侧滑返回功能也就失效了，需要侧滑返回功能需要自己处理。

第一种方案比较简单就不做赘述了，第二种方案是这样的：

自定义导航控制器，然后重写如下方法：

```
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated
{
    UIBarButtonItem *backItem = [[UIBarButtonItem alloc] initWithTitle:@"返回" style:UIBarButtonItemStyleDone target:nil action:nil];
    viewController.navigationItem.backBarButtonItem = backItem;
    
    [super pushViewController:viewController animated:animated];
}
```

如果不需要`返回`这两个字，只需要这样写就好。

```
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated
{
    UIBarButtonItem *backItem = [[UIBarButtonItem alloc] initWithTitle:nil style:UIBarButtonItemStyleDone target:nil action:nil];
    viewController.navigationItem.backBarButtonItem = backItem;
    
    [super pushViewController:viewController animated:animated];
}
```

## 3.监听返回按钮的点击事件

在有些场景，我们需要监听返回按钮的事件。比如，当页面用户输入了一些内容后，用户要点击返回，想要回到上一个页面时，提醒用户是否要缓存已经输入的内容。

如果我们重写了导航栏的返回按钮，那么处理这种情况就很Easy，不做赘述了。

但是，如果我们没有重写过系统的返回按钮，想要处理这种情况就比较麻烦，但是也是可以处理的。

处理步骤如下：
1.首先创建一个`UIViewController`的类别，头文件(.h)的内容如下：

```
@protocol BackItemProtocol <NSObject>

- (BOOL)navigationShouldPopWhenBackButtonClick;

@end

@interface UIViewController (BackItem)<BackItemProtocol>

@end

@interface UINavigationController (BackItem)

@end
```
包含一个协议、UIViewController的类别、UINavigationController的类别。

然后，实现文件（.m）如下：

```
#import "UIViewController+BackItem.h"

@implementation UIViewController (BackItem)

- (BOOL)navigationShouldPopWhenBackButtonClick
{
    return YES;
}

@end


@implementation UINavigationController (BackItem)

// 这个其实是导航栏的协议方法，在这里重写了
- (BOOL)navigationBar:(UINavigationBar *)navigationBar shouldPopItem:(UINavigationItem *)item
{
    if([self.viewControllers count] < [navigationBar.items count]) {
        return YES;
    }
    
    BOOL shouldPop = YES;
    UIViewController *vc = [self topViewController];
    if([vc respondsToSelector:@selector(navigationShouldPopWhenBackButtonClick)]) {
        shouldPop = [vc navigationShouldPopWhenBackButtonClick];
    }
    
    if (shouldPop) {
        dispatch_async(dispatch_get_main_queue(), ^{
            [self popViewControllerAnimated:YES];
        });
    } else {
        for(UIView *subview in [navigationBar subviews]) {
            if(subview.alpha < 1) {
                [UIView animateWithDuration:.25 animations:^{
                    subview.alpha = 1;
                }];
            }
        }
    }
    return NO;
}

@end
```

默认是，不需要处理返回按钮的事件，直接使用系统的pop方法。

但是，如果我们需要在用户点击返回按钮时，弹窗提示，那就需要导入这个类别。

然后，重写一个方法：

```
- (BOOL)navigationShouldPopWhenBackButtonClick
{
    BOOL isFlag = 输入框不为空等等条件
    if (isFlag) {
        UIAlertController *alertVC = [UIAlertController alertControllerWithTitle:nil message:@"是否保存修改" preferredStyle:UIAlertControllerStyleAlert];
        UIAlertAction *cancelAction = [UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
            // 这里延时执行是因为UIAlertController阻塞UI，可能会导致动画的不流畅
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
                [self.navigationController popViewControllerAnimated:YES];
            });
        }];
        UIAlertAction *saveAction = [UIAlertAction actionWithTitle:@"保存" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        // 这里延时执行是因为UIAlertController阻塞UI，可能会导致动画的不流畅
            dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
                [self rightClick];
            });
        }];
        [alertVC addAction:cancelAction];
        [alertVC addAction:saveAction];
        [self presentViewController:alertVC animated:YES completion:nil];
        return NO;
    }
    
    return YES;
}
```

## 4.导航控制器的页面跳转方式

安卓中的页面跳转有四种方式: standard、singleTop、singleTask、singleInstance。

例如`singleTask`，在做IM类App，跳转到聊天室的场景，就非常有用，可以保证控制器栈中只有一个聊天室，避免返回时层级太深。

iOS端如果要仿这个效果的话，可以利用导航控制器的API：

```
- (void)setViewControllers:(NSArray<UIViewController *> *)viewControllers animated:(BOOL)animated
```

首先，为UINavigationController 创建一个类别。
比如：

```
UINavigationController+HLPushAndPop.h
UINavigationController+HLPushAndPop.m
```
然后，新增几个方法：

拿两个方法来举例
```
- (void)hl_pushSingleViewController:(UIViewController *)viewController
                           animated:(BOOL)animated;

- (void)hl_pushSingleViewController:(UIViewController *)viewController
                         parentClass:(Class)parentClass
                            animated:(BOOL)animated;
```

再然后，实现方法：

实现步骤：

* 1. 创建新的数组复制导航控制器原来的堆栈中的控制器。
* 2. 在原始堆栈数组中判断是否存在该类型的控制器，如果存在记录其索引。
* 3. 在复制的数组中将索引及上方所有控制器移除。
* 4. 把将要push出来的控制器添加到复制的数组中。
* 5. 将新的控制器数组设置为导航控制器的栈数组，根据参数判断是否要显示动画。

我这边做了一些发散，因为一些类可能会有很多子类，那么想要保证父类以及子类的实例都只有一个，所以将方法做了改进。
```
- (void)hl_pushSingleViewController:(UIViewController *)viewController
                           animated:(BOOL)animated
{
    [self hl_pushSingleViewController:viewController parentClass:viewController.class animated:animated];
}

- (void)hl_pushSingleViewController:(UIViewController *)viewController
                        parentClass:(Class)parentClass
                           animated:(BOOL)animated
{
    if (!viewController) {
        return;
    }
    // 如果要push的界面不是 parentClass以及其子类的实例，则按照方法1处理
    if (![viewController isKindOfClass:parentClass]) {
        [self hl_pushSingleViewController:viewController animated:animated];
        return;
    }
    
    // 判断 导航控制器堆栈中是否有parentClass以及其子类的实例
    NSArray *childViewControllers = self.childViewControllers;
    NSMutableArray *newChildVCs = [[NSMutableArray alloc] initWithArray:childViewControllers];
    BOOL isExit = NO;
    NSInteger index = 0;
    for (int i = 0; i < childViewControllers.count; i++) {
        UIViewController *vc = childViewControllers[i];
        if ([vc isKindOfClass:parentClass]) {
            isExit = YES;
            index = i;
            break;
        }
    }
    
    // 如果不存在，则直接push
    if (!isExit) {
        [self pushViewController:viewController animated:animated];
        return;
    }
    
    // 如果存在，则将该实例及上面的所有界面全部弹出栈，然后将要push的界面放到栈顶。
    for (NSInteger i = childViewControllers.count - 1; i >= index; i--) {
        [newChildVCs removeObjectAtIndex:i];
    }
    
    [newChildVCs addObject:viewController];
    viewController.hidesBottomBarWhenPushed = (newChildVCs.count > 1);
    [self setViewControllers:newChildVCs animated:animated];
}

```

当然了，除了上面这些场景，还可以扩展出一些其他的场景，比如我们期望将要push出来的控制器再某个栈中控制器的后面或者前面，这样当点击返回或者侧滑时，就直接回到了指定页面了。

或者我们知道将要返回的页面的类型，直接pop回指定页面。

扩展出来的其他方法都在Demo中了，有兴趣的可以看一下。

地址是：[HLProject](https://github.com/Haley-Wong/HLProject)

Have Fun!








 
  


