---
layout:     post
title:      "iOS下JS与OC互相调用（二）--WKWebView 拦截URL"
date:       2015-11-18
author:     "Haley-Wong"
catalog:    true
tags:
    - JS与iOS Native交互
---

在上篇文章中讲述了使用UIWebView拦截URL的方式来处理JS与OC交互，由于UIWebView比较耗内存，性能上不太好，而苹果在iOS 8中推出了WKWebView。同样的用WKWebView也可以拦截URL，做JS 与OC交互。关于WKWebView与UIWebView的对比，大家请自动百度或者google。


| 打开百度网页前 | 打开百度网页后
--------- | --------- | -------------
UIWebView | 内存47M | 内存75.6M，最高峰83M
WKWebView | 内存47M | 内存51M

### WKWebView 拦截URL
WKWebView 与 UIWebView 拦截URL 的处理方式基本一样。除了代理方法和WKWebView的使用不太一样，关于WKWebView更详尽的讲解和用法，还是自行搜索学习，本文重点还是讲解如何实现JS 与OC 互相调用。
> 提醒：WKWebView 是iOS 8 推出的WebKit.framework中的控件，只有app 不需要兼容iOS 7及以下的时候才可以使用。

先看动态效果图：

![](/img/blogs/js-native-2/img_01.gif)

#### 1.创建WKWebView，加载本地HTML。
WKWebView的创建有几点不同：
* 1.初始化多了个`configuration`参数，当然这个参数我们也可以不传，直接使用默认的设置就好。
* 2.WKWebView的代理有两个`navigationDelegate`和`UIDelegate`。我们要拦截URL，就要通过`navigationDelegate`的一个代理方法来实现。如果在HTML中要使用alert等弹窗，就必须得实现`UIDelegate`的相应代理方法。
* 3.在iOS 9之前，WKWebView加载本地HTML会有一些问题。（不能加载本地HTML，或者部分CSS/本地图片加载不了等）

我这里创建WKWebView的示例代码是这样的:
```
    WKWebViewConfiguration *configuration = [[WKWebViewConfiguration alloc] init];
    configuration.userContentController = [WKUserContentController new];
    
    WKPreferences *preferences = [WKPreferences new];
    preferences.javaScriptCanOpenWindowsAutomatically = YES;
    preferences.minimumFontSize = 30.0;
    configuration.preferences = preferences;

    self.webView = [[WKWebView alloc] initWithFrame:self.view.frame configuration:configuration];
 
    NSString *urlStr = [[NSBundle mainBundle] pathForResource:@"index.html" ofType:nil];
    NSURL *fileURL = [NSURL fileURLWithPath:urlStr];
    [self.webView loadFileURL:fileURL allowingReadAccessToURL:fileURL];
    
    self.webView.navigationDelegate = self;
    [self.view addSubview:self.webView];
```
因为加载的本地HTML内容，跟上一篇UIWebView中介绍的HTML内容一样，所以关于HTML中的内容就不再讲解了。
#### 2.拦截URL
使用WKNavigationDelegate中的代理方法，拦截自定义的URL来实现JS调用OC方法。
```
#pragma mark - WKNavigationDelegate
- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler
{
    NSURL *URL = navigationAction.request.URL;
    NSString *scheme = [URL scheme];
    if ([scheme isEqualToString:@"haleyaction"]) {
        [self handleCustomAction:URL];
        decisionHandler(WKNavigationActionPolicyCancel);
        return;
    }
    decisionHandler(WKNavigationActionPolicyAllow);
}
```
需要注意的是：
> 1.如果实现了这个代理方法，就必须得调用`decisionHandler`这个block，否则会导致app 崩溃。block参数是个枚举类型，`WKNavigationActionPolicyCancel`代表取消加载，相当于UIWebView的代理方法return NO的情况；`WKNavigationActionPolicyAllow`代表允许加载，相当于UIWebView的代理方法中 return YES的情况。
>
> 2.其他的关于为什么要统一设置scheme，在上一篇中讲过。

关于如何区分执行不同的OC 方法，也与UIWebView的处理方式一样,通过URL 的host 来区分执行不同的方法：
```
#pragma mark - private method
- (void)handleCustomAction:(NSURL *)URL
{
    NSString *host = [URL host];
    if ([host isEqualToString:@"scanClick"]) {
        NSLog(@"扫一扫");
    } else if ([host isEqualToString:@"shareClick"]) {
        [self share:URL];
    } else if ([host isEqualToString:@"getLocation"]) {
        [self getLocation];
    } else if ([host isEqualToString:@"setColor"]) {
        [self changeBGColor:URL];
    } else if ([host isEqualToString:@"payAction"]) {
        [self payAction:URL];
    } else if ([host isEqualToString:@"shake"]) {
        [self shakeAction];
    } else if ([host isEqualToString:@"goBack"]) {
        [self goBack];
    }
}
```

#### 3.OC 调用 JS 方法
JS 调用OC 方法后，有的操作可能需要将结果返回给JS。这时候就是OC 调用JS 方法的场景。
WKWebView 提供了一个新的方法`evaluateJavaScript:completionHandler:`，实现OC 调用JS 等场景。
```
- (void)getLocation
{
    // 获取位置信息
    
    // 将结果返回给js
    NSString *jsStr = [NSString stringWithFormat:@"setLocation('%@')",@"广东省深圳市南山区学府路XXXX号"];
    [self.webView evaluateJavaScript:jsStr completionHandler:^(id _Nullable result, NSError * _Nullable error) {
        NSLog(@"%@----%@",result, error);
    }];
}
```
`evaluateJavaScript:completionHandler:`没有返回值，JS 执行成功还是失败会在completionHandler 中返回。所以使用这个API 就可以避免执行耗时的JS，或者alert 导致界面卡住的问题。

#### 4.WKWebView中使用弹窗
在上面提到，如果在WKWebView中使用alert、confirm 等弹窗，就得实现WKWebView的`WKUIDelegate`中相应的代理方法。

例如，我在JS中要显示alert 弹窗，就必须实现如下代理方法，否则alert 并不会弹出。
```
#pragma mark - WKUIDelegate
- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(void))completionHandler
{
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:@"提醒" message:message preferredStyle:UIAlertControllerStyleAlert];
    [alert addAction:[UIAlertAction actionWithTitle:@"知道了" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
        completionHandler();
    }]];
    
    [self presentViewController:alert animated:YES completion:nil];
}
```
其中`completionHandler`这个block 一定得调用，至于在哪里调用，倒是无所谓，我们也可以写在方法实现的第一行，或者最后一行。

示例工程地址：[JS_OC_URL](https://github.com/Haley-Wong/JS_OC/tree/master/JS_OC_URL)

