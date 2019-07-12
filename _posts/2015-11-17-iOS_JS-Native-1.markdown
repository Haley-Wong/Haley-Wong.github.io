---
layout:     post
title:      "iOS下JS与OC互相调用（一）--UIWebView 拦截URL"
date:       2015-11-17
author:     "Haley-Wong"
catalog:    true
tags:
    - JS与iOS Native交互
---

最近准备把之前用UIWebView实现的JS与原生相互调用功能，用WKWebView来替换。顺便搜索整理了一下JS 与OC 交互的方式，非常之多啊。目前我已知的JS 与 OC 交互的处理方式：
* 1.在JS 中做一次URL跳转，然后在OC中拦截跳转。（这里分为UIWebView 和 WKWebView两种，去年因为还要兼容iOS 6，所以没办法只能采用UIWebView来做。）
* 2.利用WKWebView 的MessageHandler。
* 3.利用系统库JavaScriptCore，来做相互调用。（iOS 7推出的）
* 4.利用第三方库WebViewJavascriptBridge。
* 5.利用第三方cordova库，以前叫PhoneGap。（这是一个库平台的库）
* 6.当下盛行的React Native。

今天就详细的介绍一下使用UIWebView拦截URL 的方式来实现JS与OC 的交互。
为什么不使用第三方库或者RAC呢？
因为就相互调用的接口使用的非常少啊，就那么三两个，完全没必要使用牛刀啊。

![](/img/blogs/js-native-1/img_01.gif)

## UIWebView 拦截URL
我之前就使用的是UIWebView + 拦截URL 的方式实现的JS与OC 交互。
原因是因为要兼容iOS 6。
### 1.创建UIWebView，并加载本地HTML。
加载本地HTML的目的是便于自己写JS调用做测试，最终肯定还是加载网络HTML。
```
    self.webView = [[UIWebView alloc] initWithFrame:self.view.frame];
    self.webView.delegate = self;
    NSURL *htmlURL = [[NSBundle mainBundle] URLForResource:@"index.html" withExtension:nil];
//    NSURL *htmlURL = [NSURL URLWithString:@"http://www.baidu.com"];
    NSURLRequest *request = [NSURLRequest requestWithURL:htmlURL];
   
    // 如果不想要webView 的回弹效果
    self.webView.scrollView.bounces = NO;
    // UIWebView 滚动的比较慢，这里设置为正常速度
    self.webView.scrollView.decelerationRate = UIScrollViewDecelerationRateNormal;
    [self.webView loadRequest:request];
    [self.view addSubview:self.webView];
```
本地的HTML里，我定义了几个按钮，来触发调用原生的方法，然后再将执行结果回调到js 里。

```
<input type="button" value="扫一扫" onclick="scanClick()" />
<input type="button" value="获取定位" onclick="locationClick()" />
<input type="button" value="修改背景色" onclick="colorClick()" />
<input type="button" value="分享" onclick="shareClick()" />
<input type="button" value="支付" onclick="payClick()" />
<input type="button" value="摇一摇" onclick="shake()" />
<input type="button" value="返回" onclick="goBack()" />

// js 就列几个比较有代表性的functions:

function loadURL(url) {
    var iFrame;
    iFrame = document.createElement("iframe");
    iFrame.setAttribute("src", url);
    iFrame.setAttribute("style", "display:none;");
    iFrame.setAttribute("height", "0px");
    iFrame.setAttribute("width", "0px");
    iFrame.setAttribute("frameborder", "0");
    document.body.appendChild(iFrame);
    // 发起请求后这个iFrame就没用了，所以把它从dom上移除掉
    iFrame.parentNode.removeChild(iFrame);
    iFrame = null;
}

function asyncAlert(content) {
    setTimeout(function(){
               alert(content);
               },1);
}

function locationClick() {
    loadURL("haleyAction://getLocation");
}
            
function setLocation(location) {
    asyncAlert(location);
    document.getElementById("returnValue").value = location;
}
```
虽然HTML 内容很少，但是还有不少学问：
> 1.为什么自定义一个`loadURL` 方法，不直接使用`window.location.href`?
>
> 答：因为如果当前网页正使用`window.location.href`加载网页的同时，调用`window.location.href`去调用OC原生方法，会导致加载网页的操作被取消掉。
同样的，如果连续使用`window.location.href`执行两次OC原生调用，也有可能导致第一次的操作被取消掉。所以我们使用自定义的`loadURL`，来避免这个问题。
`loadURL`的实现来自[关于UIWebView和PhoneGap的总结](http://blog.devtang.com/blog/2012/03/24/talk-about-uiwebview-and-phonegap/)一文。

> 2.为什么loadURL 中的链接，使用统一的scheme?
>
> 答:便于在OC 中做拦截处理，减少在JS中调用一些OC 没有实现的方法时，webView 做跳转。因为我在OC 中拦截URL 时，根据scheme (即`haleyAction`)来区分是调用原生的方法还是正常的网页跳转。然后根据host（即//后的部分`getLocation`）来区分执行什么操作。

> 3.为什么自定义一个`asyncAlert`方法？
>
> 答：因为有的JS调用是需要OC 返回结果到JS的。`stringByEvaluatingJavaScriptFromString`是一个同步方法，会等待js 方法执行完成，而弹出的alert 也会阻塞界面等待用户响应，所以他们可能会造成死锁。导致alert 卡死界面。如果回调的JS 是一个耗时的操作，那么建议将耗时的操作也放入`setTimeout`的`function` 中。

### 2.拦截 URL
UIWebView 有一个代理方法，可以拦截到每一个链接的Request。return YES,webView 就会加载这个链接；return NO,webView 就不会加载这个连接。我们就在这个拦截的代理方法中处理自己的URL。
这是我的示例代码：
```
#pragma mark - UIWebViewDelegate
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
{
    NSURL *URL = request.URL;
    NSString *scheme = [URL scheme];
    if ([scheme isEqualToString:@"haleyaction"]) {
        [self handleCustomAction:URL];
        return NO;
    }
    return YES;
}
```
这里通过scheme，来拦截掉自定义的URL 就非常容易了，如果不同的方法使用不同的scheme，那么判断起来就非常的麻烦。
然后，看看我的处理连接的方法：
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
顺便看一下如何将结果回调到JS中：
```
- (void)getLocation
{
    // 获取位置信息
    
    // 将结果返回给js
    NSString *jsStr = [NSString stringWithFormat:@"setLocation('%@')",@"广东省深圳市南山区学府路XXXX号"];
    [self.webView stringByEvaluatingJavaScriptFromString:jsStr];
}
```
当然，有时候我们在JS中调用OC 方法的时候，也需要传参数到OC 中，怎么传呢？
就像一个get 请求一样，把参数放在后面：
```
function shareClick() {
    loadURL("haleyAction://shareClick?title=测试分享的标题&content=测试分享的内容&url=http://www.baidu.com");
}
```
那么如果获取到这些参数呢?
所有的参数都在URL的query中，先通过`&`将字符串拆分，在通过`=`把参数拆分成key 和实际的值。下面有示例代码：
```
- (void)share:(NSURL *)URL
{
    NSArray *params =[URL.query componentsSeparatedByString:@"&"];
    
    NSMutableDictionary *tempDic = [NSMutableDictionary dictionary];
    for (NSString *paramStr in params) {
        NSArray *dicArray = [paramStr componentsSeparatedByString:@"="];
        if (dicArray.count > 1) {
            NSString *decodeValue = [dicArray[1] stringByReplacingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
            [tempDic setObject:decodeValue forKey:dicArray[0]];
        }
    }
    
    NSString *title = [tempDic objectForKey:@"title"];
    NSString *content = [tempDic objectForKey:@"content"];
    NSString *url = [tempDic objectForKey:@"url"];
    // 在这里执行分享的操作
    
    // 将分享结果返回给js
    NSString *jsStr = [NSString stringWithFormat:@"shareResult('%@','%@','%@')",title,content,url];
    [self.webView stringByEvaluatingJavaScriptFromString:jsStr];
}
```
### 3. OC调用JS方法
关于将OC 执行结果返回给JS 需要注意的是：
> 如果回调执行的JS 方法带参数，而参数不是字符串时，不要加`单引号`,否则可能导致调用JS 方法失败。比如我这样的：
```
NSData *jsonData = [NSJSONSerialization dataWithJSONObject:userProfile options:NSJSONWritingPrettyPrinted error:nil];
NSString *jsonStr = [[NSString alloc] initWithData:jsonData encoding:NSUTF8StringEncoding];
NSString *jsStr = [NSString stringWithFormat:@"loginResult('%@',%@)",type, jsonStr];
[_webView stringByEvaluatingJavaScriptFromString:jsStr];
```
如果第二个参数用单引号包起来，就会导致JS端的loginResult不会调用。

另外，利用`[webView stringByEvaluatingJavaScriptFromString:@"var arr = [3, 4, 'abc'];"];`,可以往HMTL的JS环境中插入全局变量、JS方法等。

示例工程地址：[JS_OC_URL](https://github.com/Haley-Wong/JS_OC/tree/master/JS_OC_URL)
> 如果你下载不了，先回到工程的一级目录再下载。

