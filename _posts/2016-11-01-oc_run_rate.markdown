---
layout:     post
title:      "Objective-C 中如何测量代码的效率"
date:       2016-11-01
author:     "Haley_Wong"
catalog:    true
tags:
    - Tools
---

# 背景
在我们编程的时候，可能经常会有一些疑问：

* 我们写的某个方法的执行效率是多少？
* 方法 A 和 方法 B 哪个更快？

因此，我们不可避免的要用到一些方法来计算代码的执行效率。计算代码的执行效率可以使用的API有：

* NSDate
* CFAbsoluteTimeGetCurrent
* CACurrentMediaTime
* dispatch_benchmark

## NSDate
看到NSDate，大家应该都能想到怎么使用吧。为了更直观一点，我还是使用代码片段来演示好了：

```
NSTimeInterval startTime = [[NSDate new] timeIntervalSinceReferenceDate];
NSLog(@"斐波那契数：%d",fibonacci(10)) ;
NSTimeInterval endTime = [[NSDate new] timeIntervalSinceReferenceDate];
NSLog(@"耗时:%f", endTime - startTime);
```
上面是一段 C与OC混合的代码片段，计算斐波那契数列计算第10个数的值需要消耗的时间。
> 利用NSDate 来计算运行效率：代码段运行前记录一次时间，运行后记录一次，然后比较时间差。
> 时间的单位是 `秒`。

## CFAbsoluteTimeGetCurrent
利用CFAbsoluteTimeGetCurrent 主要是利用`CFAbsoluteTimeGetCurrent()`函数来获取当前的绝对时间。同样的我也是用代码片段来演示：
```
CFAbsoluteTime startTime = CFAbsoluteTimeGetCurrent();
NSLog(@"斐波那契数：%d",fibonacci(10)) ;
CFAbsoluteTime endTime = CFAbsoluteTimeGetCurrent();
NSLog(@"耗时:%f",endTime - startTime);
```
> 利用CFAbsoluteTimeGetCurrent 来计算运行效率：代码段运行前记录一次时间，运行后记录一次，然后比较时间差。
> 
>时间的单位是 `秒`。

看到这里可能会有疑问`CFAbsoluteTimeGetCurrent()`是如何获取时间的呢？

我们追踪进去查看代码，就会有答案了，这是源码：
```
typedef double CFTimeInterval;
typedef CFTimeInterval CFAbsoluteTime;
/* absolute time is the time interval since the reference date */
/* the reference date (epoch) is 00:00:00 1 January 2001. */

CF_EXPORT
CFAbsoluteTime CFAbsoluteTimeGetCurrent(void);
```
从`CFTimeInterval`的定义和注释可以看出，`CFAbsoluteTimeGetCurrent(void)`返回的时间就是当前时间相对与`reference date`的时间。

`CFTimeInterval` 是对double 的重命名，而`NSTimeInterval` 也是对double 的重命名。

它们之间的关系就可想而知了！

`CFAbsoluteTimeGetCurrent()` 其实等价于 `[[NSDate new] timeIntervalSinceReferenceDate]` 。

## CACurrentMediaTime
利用`CACurrentMediaTime`主要是利用`CACurrentMediaTime()`函数来计算时间。

还是先用示例来演示用法：
```
CFTimeInterval startTime = CACurrentMediaTime();
NSLog(@"斐波那契数：%d",fibonacci(10)) ;
CFTimeInterval endTime = CACurrentMediaTime();
NSLog(@"耗时:%f",endTime - startTime);
```
> 计算执行效率时间上依然是：代码段运行前记录一次时间，运行后记录一次，然后比较时间差。
时间的单位是 `秒`。

跟踪查看源码中对`CACurrentMediaTime()`的定义
```
/* Returns the current CoreAnimation absolute time. This is the result of
 * calling mach_absolute_time () and converting the units to seconds. */

CA_EXTERN CFTimeInterval CACurrentMediaTime (void)
    __OSX_AVAILABLE_STARTING (__MAC_10_5, __IPHONE_2_0);
```
可以看出CACurrentMediaTime() 是对mach_absolute_time()的封装。返回的是CoreAnimation 中的当前时间。

## dispatch_benchmark
dispatch_benchmark 是 [libdispatch(Grand Central Dispatch)](http://libdispatch.macosforge.org/) 的一部分。但严肃地说，这个方法并没有被公开声明，所以我们必须要自己声明：
```
extern uint64_t dispatch_benchmark(size_t count, void (^block)(void));
```
> 第二个参数是执行的代码片段block。
> 第一个参数是执行的次数（即运行block 的次数）。

```
uint64_t t = dispatch_benchmark(1000000, ^{
    NSLog(@"斐波那契数：%d",fibonacci(10)) ;
  });
NSLog(@"耗时: %llu ns", t);
```
> 警告：
> 这里写的是我自己的理解，但是不一定正确，如果你有比较确切的资料或者不同的理解，麻烦告知我，万分感谢！
>
>dispatch_benchmark 应该是通过计算多次执行某代码片段的总时间，通过多次运行的总时间除以迭代运行的次数来计算一次运行的时间，以减小单次运行的误差。
>
>`dispatch_benchmark(size_t count, void (^block)(void)) `返回的就是单次运行代码段的时间。

关于dispatch_benchmark[^fn]的更多的文章，我们可以去文章末的资料中查看。

## 区别
* 1、它们所属的框架不同。
```
NSDate 来自Foundation框架，只需要#import <Foundation/Foundation.h>，就可以使用了。
CFAbsoluteTimeGetCurrent 来自CoreFoundation框架，而Foundation框架是包含CoreFoundation框架的。
CACurrentMediaTime 来自QuartzCore框架，而UIKit框架是包含了QuartzCore框架的。
dispatch_benchmark 来自 libdispatch(G C D)库，而Foundation框架已包含了libdispatch库。
```
* 2、参考时间不同。
NSDate 和 CFAbsoluteTimeGetCurrent 是通过ReferenceDate来计算相差的秒值。与服务器的时间有关系。

而CACurrentMediaTime() 是封装的mach_absolute_time()，mach_absolute_time() 是基于内建时钟的，能够更精确更原子化地测量，并且不会因为外部时间变化而变化（例如时区变化、夏时制、秒突变等）。
dispatch_benchmark的时间计算方式未知（推测不是根据参考时间计算）。

* 3、时间的精度不同
NSDate、CFAbsoluteTimeGetCurrent、CACurrentMediaTime计算出来的时间精度都是`秒`。
而dispatch_benchmark 的时间精度是`纳秒`。


## 最后提醒
因为从操作系统本身的一切基本因素都是可变性非常强的，性能应该通过大量的试验来测量。对于大多数应用来说，样本数量在 10<sup>5</sup> 到 10<sup>8</sup> 之间是合理的。

所以我们应该运行要执行的代码段 10<sup>5</sup> 到 10<sup>8</sup>次，再来求平均值。

[^fn]:[更多关于dispatch_benchmark的介绍](http://nshipster.cn/benchmarking/)。


Have Fun!


