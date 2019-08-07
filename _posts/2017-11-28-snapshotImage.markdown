---
layout:     post
title:      "iOS 中获取某个视图的截图"
date:       2017-11-28
author:     "Haley_Wong"
catalog:    true
tags:
    - Tools
---

最近在做SDK的截图，想触发类似系统的截屏功能，找了一圈，总结一下靠谱的几种方式。
我写了个UIView 的category，将这几种方式封装和简化了一下。

## 第一种情形截图
这种是最最普通的截图，针对一般的视图上添加视图的情况，基本都可以使用。
源码：
```
/**
 普通的截图
 该API仅可以在未使用layer和OpenGL渲染的视图上使用
 
 @return 截取的图片
 */
- (UIImage *)nomalSnapshotImage
{
    UIGraphicsBeginImageContextWithOptions(self.frame.size, NO, [UIScreen mainScreen].scale);
    [self.layer renderInContext:UIGraphicsGetCurrentContext()];
    UIImage *snapshotImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return snapshotImage;
}
```

## 第二种情形截图
如果一些视图是用OpenGL渲染出来的，那么使用上面的方式就无法截图到OpenGL渲染的部分，这时候就要用到改进后的截图方案：
```
/**
 针对有用过OpenGL渲染过的视图截图
 
 @return 截取的图片
 */
- (UIImage *)openglSnapshotImage
{
    CGSize size = self.bounds.size;
    UIGraphicsBeginImageContextWithOptions(size, NO, [UIScreen mainScreen].scale);
    CGRect rect = self.frame;
    [self drawViewHierarchyInRect:rect afterScreenUpdates:YES];
    UIImage *snapshotImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return snapshotImage;
}
```

## 第三种情形截图
有一些特殊的Layer（比如：`AVCaptureVideoPreviewLayer` 和 `AVSampleBufferDisplayLayer`） 添加到某个View 上后，使用上面的几种方式都无法截取到Layer上的内容，这个时候可以使用系统的一个API，但是该API只能返回一个UIView，返回的UIView 可以修改frame 等参数。

```
/**
 截图
 以UIView 的形式返回(_UIReplicantView)
 
 @return 截取出来的图片转换的视图
 */
- (UIView *)snapshotView
{
    UIView *snapView = [self snapshotViewAfterScreenUpdates:YES];
    return snapView;
}
```

> 遗留问题：
通过方式三截取的UIView，无法转换为UIImage，我试过将返回的截图View写入位图再转换成UIImage，但是返回的UIImage 要么为空，要么没有内容。如果有人知道解决方案请告知我。


## UIWebView的截图
去年在做蓝牙打印的时候，尝试过将UIWebView 的内容转换为UIImage，写过一个UIWebView的category，也算是对UIWebView 的截图，顺便也贴出来吧
```
- (UIImage *)imageForWebView
{
    // 1.获取WebView的宽高
    CGSize boundsSize = self.bounds.size;
    CGFloat boundsWidth = boundsSize.width;
    CGFloat boundsHeight = boundsSize.height;

    // 2.获取contentSize
    CGSize contentSize = self.scrollView.contentSize;
    CGFloat contentHeight = contentSize.height;
    // 3.保存原始偏移量，便于截图后复位
    CGPoint offset = self.scrollView.contentOffset;
    // 4.设置最初的偏移量为(0,0);
    [self.scrollView setContentOffset:CGPointMake(0, 0)];

    NSMutableArray *images = [NSMutableArray array];
    while (contentHeight > 0) {
        // 5.获取CGContext 5.获取CGContext
        UIGraphicsBeginImageContextWithOptions(boundsSize, NO, 0.0);
        CGContextRef ctx = UIGraphicsGetCurrentContext();
        // 6.渲染要截取的区域
        [self.layer renderInContext:ctx];
        UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        // 7.截取的图片保存起来
        [images addObject:image];

        CGFloat offsetY = self.scrollView.contentOffset.y;
        [self.scrollView setContentOffset:CGPointMake(0, offsetY + boundsHeight)];
        contentHeight -= boundsHeight;
    }
    // 8 webView 恢复到之前的显示区域
    [self.scrollView setContentOffset:offset];
    CGFloat scale = [UIScreen mainScreen].scale;
    CGSize imageSize = CGSizeMake(contentSize.width * scale,
                                  contentSize.height * scale);
   // 9.根据设备的分辨率重新绘制、拼接成完整清晰图片
    UIGraphicsBeginImageContext(imageSize);
    [images enumerateObjectsUsingBlock:^(UIImage *image, NSUInteger idx, BOOL *stop) {
        [image drawInRect:CGRectMake(0,scale * boundsHeight * idx,scale * boundsWidth,scale * boundsHeight)];
    }];
    UIImage *fullImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    
    return fullImage;
}
```

Have Fun!








 
  


