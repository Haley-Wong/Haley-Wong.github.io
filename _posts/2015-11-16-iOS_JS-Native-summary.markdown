---
layout:     post
title:      "iOS下JS与原生OC互相调用(总结)"
date:       2015-11-16
author:     "Haley-Wong"
catalog:    true
tags:
    - JS与Native交互
---

iOS开发免不了要与UIWebView打交道，然后就要涉及到JS与原生OC交互，今天总结一下JS与原生OC交互的两种方式。

## JS调用原生OC篇
### 方式一 
第一种方式是用JS发起一个假的URL请求，然后利用UIWebView的代理方法拦截这次请求，然后再做相应的处理。

我写了一个简单的HTML网页和一个btn点击事件用来与原生OC交互，HTML代码如下：
``` 
<html>
    <header>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<script type="text/javascript">
            function showAlert(message){
                alert(message);
            }
        
            function loadURL(url) {
                var iFrame;
                iFrame = document.createElement("iframe");
                iFrame.setAttribute("src", url);
                iFrame.setAttribute("style", "display:none;");
                iFrame.setAttribute("height", "0px");
                iFrame.setAttribute("width", "0px");
                iFrame.setAttribute("frameborder", "0");
                document.body.appendChild(iFrame);
                // 发起请求后这个 iFrame 就没用了，所以把它从 dom 上移除掉
                iFrame.parentNode.removeChild(iFrame);
                iFrame = null;
            }
            function firstClick() {
                loadURL("firstClick://shareClick?title=分享的标题&content=分享的内容&url=链接地址&imagePath=图片地址");
            }
        </script>
    </header>
    
    <body>
        <h2> 这里是第一种方式 </h2>
        <br/>
        <br/>
        <button type="button" onclick="firstClick()">Click Me!</button>
        
    </body>
</html>
```
然后在项目的控制器中实现UIWebView的代理方法：

```
#pragma mark - UIWebViewDelegate
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
{
    NSURL * url = [request URL];
   if ([[url scheme] isEqualToString:@"firstclick"]) {
        NSArray *params =[url.query componentsSeparatedByString:@"&"];
        
        NSMutableDictionary *tempDic = [NSMutableDictionary dictionary];
        for (NSString *paramStr in params) {
            NSArray *dicArray = [paramStr componentsSeparatedByString:@"="];
            if (dicArray.count > 1) {
                NSString *decodeValue = [dicArray[1] stringByReplacingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
                [tempDic setObject:decodeValue forKey:dicArray[0]];
            }
        }
       UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"方式一" message:@"这是OC原生的弹出窗" delegate:self cancelButtonTitle:@"收到" otherButtonTitles:nil];
       [alertView show];
       NSLog(@"tempDic:%@",tempDic);
        return NO;
    }
    
    return YES;
}
```

> 注意：
> 1. JS中的firstClick,在拦截到的url scheme全都被转化为小写。
> 
> 2.html中需要设置编码，否则中文参数可能会出现编码问题。
> 
> 3.JS用打开一个iFrame的方式替代直接用document.location的方式，以避免多次请求，被替换覆盖的问题。


早期的JS与原生交互的开源库很多都是用得这种方式来实现的，例如：
* PhoneGap(Cordova的前身)
* [WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)。

关于这种方式调用OC方法，唐巧早期有篇文章有过介绍：
[关于UIWebView和PhoneGap的总结](http://blog.devtang.com/blog/2012/03/24/talk-about-uiwebview-and-phonegap/)

### 方式二
在iOS 7之后，apple添加了一个新的库JavaScriptCore，用来做JS交互，因此JS与原生OC交互也变得简单了许多。

首先导入JavaScriptCore库, 然后在OC中获取JS的上下文

```
JSContext *context = [self.webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
```
再然后定义好JS需要调用的方法，例如JS要调用share方法，则可以在UIWebView加载url完成后，在其代理方法中添加要调用的share方法：
```
- (void)webViewDidFinishLoad:(UIWebView *)webView
{
    JSContext *context = [self.webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
    //定义好JS要调用的方法, share就是调用的share方法名
    context[@"share"] = ^() {
        NSLog(@"+++++++Begin Log+++++++");
        NSArray *args = [JSContext currentArguments];
      
        dispatch_async(dispatch_get_main_queue(), ^{
            UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"方式二" message:@"这是OC原生的弹出窗" delegate:self cancelButtonTitle:@"收到" otherButtonTitles:nil];
            [alertView show];
        });
        
        for (JSValue *jsVal in args) {
            NSLog(@"%@", jsVal.toString);
        }
        
        NSLog(@"-------End Log-------");
    };
}
```
> 注意：
> 可能最新版本的iOS系统做了改动，现在（iOS9，Xcode 7.3，去年使用Xcode 6 和iOS 8没有线程问题）中测试,block中是在子线程，因此执行UI操作，控制台有警告，需要回到主线程再操作UI。

其中相对应的html部分如下：
```
<html>
    <header>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
        <script type="text/javascript">
        
            function secondClick() {
                share('分享的标题','分享的内容','图片地址');
            }
        
        function showAlert(message){
            alert(message);
        }
        
        </script>
    </header>
    
    <body>
        <h2> 这里是第二种方式 </h2>
        <br/>
        <br/>
        <button type="button" onclick="secondClick()">Click Me!</button>
        
    </body>
</html>
```
JS部分确实要简单的多了。

## OC调用JS篇
### 方式一

```
NSString *jsStr = [NSString stringWithFormat:@"showAlert('%@')",@"这里是JS中alert弹出的message"];
[_webView stringByEvaluatingJavaScriptFromString:jsStr];
```
> 注意：该方法会同步返回一个字符串，因此是一个同步方法，可能会阻塞UI。

### 方式二
继续使用JavaScriptCore库来做JS交互。
```
JSContext *context = [self.webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
NSString *textJS = @"showAlert('这里是JS中alert弹出的message')";
[context evaluateScript:textJS];
```

**重点：**

* `stringByEvaluatingJavaScriptFromString`是一个同步的方法，使用它执行JS方法时，如果JS 方法比较耗的时候，会造成界面卡顿，尤其是js 弹出alert 的时候。alert 也会阻塞界面，等待用户响应，而`stringByEvaluatingJavaScriptFromString`又会等待js执行完毕返回。这就造成了死锁。

* 官方推荐使用`WKWebView`的`evaluateJavaScript:completionHandler:`代替这个方法。
其实我们也有另外一种方式，自定义一个延迟执行alert 的方法来防止阻塞，然后我们调用自定义的alert 方法。同理，耗时较长的js 方法也可以放到setTimeout 中。

```
function asyncAlert(content) {
    setTimeout(function(){
         alert(content);
         },1);
}
```

我也写了一个demo,包括JS与OC交互的两种方式：
[JS_OC_summary](https://github.com/Haley-Wong/JS_OC/tree/master/JS_OC_summary)

如果你看的还不尽兴，后面还有几篇JS相互调用的文章。

[iOS下JS与OC互相调用（一）--UIWebView 拦截URL](https://juejin.im/post/5a952a345188257a7f1dd5ae)

[iOS下JS与OC互相调用（二）--WKWebView 拦截URL](https://juejin.im/post/5a952bf05188257a7f1dd5b9)

[iOS下JS与OC互相调用（三）--MessageHandler](https://juejin.im/post/5a952cd85188257a6e405b9d)

[iOS下JS与OC互相调用（四）--JavaScriptCore](https://juejin.im/post/5a952d4d5188257a585113e9)

[iOS下JS与OC互相调用（五）--UIWebView + WebViewJavascriptBridge](https://juejin.im/post/5a9530195188257a67179bd1)

[iOS下JS与OC互相调用（六）--WKWebView + WebViewJavascriptBridge](https://juejin.im/post/5a9531a6f265da4e8f04d257)

[iOS下JS与OC互相调用（七）--Cordova 基础](https://juejin.im/post/5a95321d6fb9a06356314692)

[iOS下JS与OC互相调用（八）--Cordova详解+实战](https://juejin.im/post/5a95326a5188257a7e3f5410)

