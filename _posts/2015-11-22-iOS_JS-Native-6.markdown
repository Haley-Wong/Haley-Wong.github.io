---
layout:     post
title:      "iOS下JS与OC互相调用（六）--WKWebView + WebViewJavascriptBridge"
date:       2015-11-22
author:     "Haley-Wong"
catalog:    true
tags:
    - JS与iOS Native交互
---

上一篇文章介绍了UIWebView 如何通过WebViewJavascriptBridge 来实现JS 与OC 的互相调用，这一篇来介绍一下WKWebView 又是如何通过WebViewJavascriptBridge 来实现JS 与OC 的互相调用的。WKWebView 下使用WebViewJavascriptBridge与UIWebView 大同小异。主要是示例化的类不一样，一些与webView 相关的API调用不一样罢了。

![](/img/blogs/js-native-6/img_01.jpg)

WKWebView 下使用`WebViewJavascriptBridge`来实现JS 与OC 的互相调用,也是通过拦截URL来实现的。

下面开始介绍WKWebView 如何通过WebViewJavascriptBridge 来实现JS 与OC 的互相调用。
关于下载[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)，然后导入工程的部分就不再赘述了。
#### 第一步，创建WKWebView。
这一步，唯一需要注意的地方，就是不用再设置`WKWebView` 的`navigationDelegate`，下一步你就知道为什么了。
```
- (void)initWKWebView
{
    WKWebViewConfiguration *configuration = [[WKWebViewConfiguration alloc] init];
    configuration.userContentController = [WKUserContentController new];
    
    WKPreferences *preferences = [WKPreferences new];
    preferences.javaScriptCanOpenWindowsAutomatically = YES;
    preferences.minimumFontSize = 30.0;
    configuration.preferences = preferences;
    
    self.webView = [[WKWebView alloc] initWithFrame:self.view.frame configuration:configuration];
    
    NSString *urlStr = [[NSBundle mainBundle] pathForResource:@"index.html" ofType:nil];
    NSString *localHtml = [NSString stringWithContentsOfFile:urlStr encoding:NSUTF8StringEncoding error:nil];
    NSURL *fileURL = [NSURL fileURLWithPath:urlStr];
    [self.webView loadHTMLString:localHtml baseURL:fileURL];
    
    self.webView.UIDelegate = self;
    [self.view addSubview:self.webView];
}
```

#### 第二步，创建WebViewJavascriptBridge实例。
这里与上一篇文章有一些不同，WKWebView 使用的是`WKWebViewJavascriptBridge`，而UIWebView 使用的是`WebViewJavascriptBridge `。
```
_webViewBridge = [WKWebViewJavascriptBridge bridgeForWebView:self.webView];
// 如果控制器里需要监听WKWebView 的`navigationDelegate`方法，就需要添加下面这行。
[_webViewBridge setWebViewDelegate:self];
```
上一步说了不用再设置`WKWebView` 的`navigationDelegate`，那是因为在`{-bridgeForWebView:}`内已经将`WKWebView` 的`navigationDelegate`设置为`WKWebViewJavascriptBridge`的实例了。

```
+ (instancetype)bridgeForWebView:(WKWebView*)webView {
    WKWebViewJavascriptBridge* bridge = [[self alloc] init];
    [bridge _setupInstance:webView];
    [bridge reset];
    return bridge;
}

- (void) _setupInstance:(WKWebView*)webView {
    _webView = webView;
    _webView.navigationDelegate = self;
    _base = [[WebViewJavascriptBridgeBase alloc] init];
    _base.delegate = self;
}
```
#### 第三步，注册 js 要调用的Native 功能
为了便于维护，我将所有js 要调用Native 功能放在了一个方法里添加，然后每个功能再单独处理。
示例代码如下：
```
#pragma mark - private method
- (void)registerNativeFunctions
{
    [self registScanFunction];
    
    [self registShareFunction];
    
    [self registLocationFunction];
    
    [self regitstBGColorFunction];
    
    [self registPayFunction];
    
    [self registShakeFunction];
}

// 注册的获取位置信息的Native 功能
- (void)registLocationFunction
{
    [_webViewBridge registerHandler:@"locationClick" handler:^(id data, WVJBResponseCallback responseCallback) {
        // 获取位置信息
        
        NSString *location = @"广东省深圳市南山区学府路XXXX号";
        // 将结果返回给js
        responseCallback(location);
    }];
}
```

关于`- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler`，我们可以这样理解，后面的block 参数是js 要调用的Native 实现，前面的handlerName 是这个Native 实现的别名。然后js 里调用handlerName 这个别名，`WebViewJavascriptBridge`最终会执行block 里的Native 实现。

#### 第四步，在HTML添加关键的js
HMTL 里在调用Native 功能之前，要先添加一个js 方法，然后主动调用一次该方法。
要添加的方法是：
```
function setupWebViewJavascriptBridge(callback) {
    if (window.WebViewJavascriptBridge) { return callback(WebViewJavascriptBridge); }
    if (window.WVJBCallbacks) { return window.WVJBCallbacks.push(callback); }
    window.WVJBCallbacks = [callback];
    var WVJBIframe = document.createElement('iframe');
    WVJBIframe.style.display = 'none';
    WVJBIframe.src = 'wvjbscheme://__BRIDGE_LOADED__';
    document.documentElement.appendChild(WVJBIframe);
    setTimeout(function() { document.documentElement.removeChild(WVJBIframe) }, 0)
}
```
如果你看过[iOS下JS与OC互相调用（一）--UIWebView 拦截URL](http://www.jianshu.com/p/7151987f012d),你就会发现这个方法与`loadURL`很像。

然后在js 中要主动调用一次上述的`setupWebViewJavascriptBridge`。
```
setupWebViewJavascriptBridge(function(bridge) {

      // 这里注册Native 要调用的js 功能。
     bridge.registerHandler('testJSFunction', function(data, responseCallback) {
        alert('JS方法被调用:'+data);
        responseCallback('js执行过了');
     })
     // 如果要有其他Native 调用的js 功能，在这里按照上面的格式添加。
})
```

主动调用`setupWebViewJavascriptBridge`有两个目的：
1. 执行一次`wvjbscheme://__BRIDGE_LOADED__`请求。
2. 注册Native 要调用的js 功能。

执行`wvjbscheme://__BRIDGE_LOADED__`，然后在WKWebView 的`navigationDelegate`方法中拦截该URL ，然后往HMTL中注入js。以下源码都摘自`WebViewJavascriptBridge`：
```
- (void)webView:(WKWebView *)webView
decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction
decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
    if (webView != _webView) { return; }
    NSURL *url = navigationAction.request.URL;
    __strong typeof(_webViewDelegate) strongDelegate = _webViewDelegate;

    if ([_base isCorrectProcotocolScheme:url]) {
        // 在这里拦截wvjbscheme://__BRIDGE_LOADED__
        if ([_base isBridgeLoadedURL:url]) {
            // 这里会注入js 
            [_base injectJavascriptFile];
        } else if ([_base isQueueMessageURL:url]) {
            [self WKFlushMessageQueue];
        } else {
            [_base logUnkownMessage:url];
        }
        decisionHandler(WKNavigationActionPolicyCancel);
    }
    
    if (strongDelegate && [strongDelegate respondsToSelector:@selector(webView:decidePolicyForNavigationAction:decisionHandler:)]) {
        [_webViewDelegate webView:webView decidePolicyForNavigationAction:navigationAction decisionHandler:decisionHandler];
    } else {
        decisionHandler(WKNavigationActionPolicyAllow);
    }
}

- (void)injectJavascriptFile {
    //读取js 内容
    NSString *js = WebViewJavascriptBridge_js();
    // 执行Native 的API，实现将js 注入 到HMTL中。
    [self _evaluateJavascript:js];
    if (self.startupMessageQueue) {
        NSArray* queue = self.startupMessageQueue;
        self.startupMessageQueue = nil;
        for (id queuedMessage in queue) {
            [self _dispatchMessage:queuedMessage];
        }
    }
}
```
WKWebView 执行js 的API 与 UIWebView 有些不同，WKWebView 用的是`{-evaluateJavaScript: completionHandler:}`，这个API 不会立刻返回执行结果，js 的执行结果会在block 中返回。

#### 第五步，在js 中调用 Native 功能。
讲完过程，终于到了 js 调用Native 的用法了。其实非常的简单，例如我想要利用Native 获取定位信息，那么在HTML中添加一个按钮，onclick事件是locationClick()，按照如下实现即可。
```
function locationClick() {
    WebViewJavascriptBridge.callHandler('locationClick',null,function(response) {
        alert(response);
        document.getElementById("returnValue").value = response;
    });
}
```
Native 执行完代码，将获取到的定位信息，通过callHandler 的第三方参数，回调返回到js 中。
response 可以是单个值，也可以是数组、键值对等。

当然如果我们调用Native 的时候，没有参数或者不需要Native 返回信息到js 中。我们还可以这样写：
```
// 没有参数，有回调可以这样写
function locationClick() {
    WebViewJavascriptBridge.callHandler('locationClick',function(response) {
        alert(response);
        document.getElementById("returnValue").value = response;
    });
}

// 没有参数，又不需要回调可以这样写
function shake() {
    WebViewJavascriptBridge.callHandler('shakeClick');
}
```
至此，JS 通过`WebViewJavascriptBridge`调用Native 的功能就完成了。

#### 第六步，Native 调用 JS 功能。
Native 调用js 功能与 js 调用Native 的原理和流程一样:
1. 现在js 中注册，Native 要调用的功能。
2. Native 调用注册时，该功能的别名，就可以完成调用。

在js 中注册 Native 要调用的功能，同样需要为该功能设置一个别名HandlerName。
其实这个步骤在前面介绍过，代码如下：
```
setupWebViewJavascriptBridge(function(bridge) {

      // 这里注册Native 要调用的js 功能。
     bridge.registerHandler('testJSFunction', function(data, responseCallback) {
        alert('JS方法被调用:'+data);
        responseCallback('js执行过了');
     })
     // 如果要有其他Native 调用的js 功能，在这里按照上面的格式添加。
})
```
上述代码，是在JS 中注册了一个别名叫`testJSFunction`的js功能，第二个参数是一个function。function里的data ，就是Native 调用该功能时传过来的参数，responseCallback是执行完js 代码后，通过responseCallback将必要的信息返回到Native中。

Native 调用js 里注册的功能，示例代码：
```
- (void)rightClick
{
    //    // 如果不需要参数，不需要回调，使用这个
    //    [_webViewBridge callHandler:@"testJSFunction"];
    //    // 如果需要参数，不需要回调，使用这个
    //    [_webViewBridge callHandler:@"testJSFunction" data:@"一个字符串"];
    // 如果既需要参数，又需要回调，使用这个
    [_webViewBridge callHandler:@"testJSFunction" data:@"一个字符串" responseCallback:^(id responseData) {
        NSLog(@"调用完JS后的回调：%@",responseData);
    }];
}
```
WKWebView 通过`WebViewJavascriptBridge`实现js 与Native 的交互，到这里就已经完成了。
这是工程示例中的效果图：

![](/img/blogs/js-native-6/img_02.gif)

示例工程地址：[JS_OC_WebViewJavascriptBridge](https://github.com/Haley-Wong/JS_OC/tree/master/JS_OC_WebViewJavascriptBridge)

Have Fun！



