---
layout:     post
title:      "Runtime系列（一）-- 基础知识"
date:       2016-07-26
author:     "Haley_Wong"
catalog:    true
tags:
    - Runtime
---

众所周知，Objective-C 是一种运行时语言。运行时怎么来体现的呢？比如一个对象的类型确定，或者对象的方法实现的绑定都是推迟到软件的运行时才能确定的。而运行时的诸多特性都是由Runtime 来实现的。

Runtime 其实就是一套C语言API库，因此它的实现也还是C语言。如果你想看Runtime的实现源码，可以去官网下载：[objc4-646.tar.gz](http://opensource.apple.com/tarballs/objc4/objc4-646.tar.gz)（我看的是这个）。

本篇不打算介绍objc_msgSend，但是关于OC中的消息最终怎么被转化为objc_msgSend这个过程，还是有必要找一篇文章好好的看一下。

以下内容部分摘录自:

王巍 (@onevcat) 的 [深入Objective-C的动态特性](https://onevcat.com/2012/04/objective-c-runtime/)

Bang 的[如何动态调用 C 函数](http://blog.cnbang.net/tech/3219/)

如果你觉得看的不尽兴，可以去看下这两篇文章。

### 动态特性
在开始介绍runtime 之前，先讲讲动态特性。经常被提到和用到的有三种：

* **动态类型（Dynamic typing）**
* **动态绑定（Dynamic binding）**
* **动态加载（Dynamic loading）**

##### 动态加载
先来说说** 动态加载 ** ，动态加载就是根据需求加载所需要的资源。有一个典型的例子，就是iPhone 会根据机型的不同加载不同的图片。iOS 下一般会有xxx.png、xxx@2x.png、xxx@3x.png。iOS 应用会在非retina设备上加载1倍图，在retina小尺寸设备（如4、4s、5、5c、5s、6、6s)上加载@2x图片，然后在大屏retina 设备(如6+、6s+)上加载@3x的图片。

##### 动态类型
然后再来说说** 动态类型 **，即运行时再决定对象的类型。动态类型这个特性在日常开发中非常的常见，最简单的就是id类型。稍微常用的就是某个类和其子类的类型确定。id类型即通用的对象类，任何对象都可以被id指针所指，而在实际使用中，往往使用introspection来确定该对象的实际所属类：
```
id obj = someInstance;
if ([obj isKindOfClass:someClass])
{
    someClass *classSpecifiedInstance = (someClass *)obj;
    // Do Something to classSpecifiedInstance which now is an instance of someClass
    //...
}
```
`-isMemberOfClass:`是 `NSObject`的方法，用以确定某个 `NSObject`对象是否是某个类的成员。与之相似的为 `-isKindOfClass:`，可以用以确定某个对象是否是某个类或其子类的成员。这两个方法为典型的introspection方法。在确定对象为某类成员后，可以安全地进行强制转换，继续之后的工作。

动态类型有利有弊，有了动态类型，我们可以在运行时根据对象的类型不同执行不同的逻辑代码；但是也导致一些错误不能及时的发现。

比如，我们经常会遇到的这类错误：

```
*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[_NSZeroData count]: unrecognized selector sent to instance 0x7f8632ed7ab0'
```

这是错误的示例代码：

![Paste_Image.png](/img/blogs/runtime_base/img_01.webp)

这就是我们错误的将一个NSData 对象赋值给了NSArray 实例，然后又调用数组的count 方法。在2014 年以前，并不会出现这样的警告信息，所以那时候很容易出现类似这样的错误。随着Swift 的推出，OC 中也加入了类型检查。现在我们就可以很及时的减少这类错误的产生。

##### 动态绑定
基于动态类型，在某个实例对象被确定后，其类型便被确定了。该对象对应的属性和响应的消息也被完全确定，这就是动态绑定。在继续之前，需要明确Objective-C中消息的概念。由于OC的动态特性，在OC中其实很少提及“函数”的概念，传统的函数一般在编译时就已经把参数信息和函数实现打包到编译后的源码中了，而在OC中最常使用的是消息机制。调用一个实例的方法，所做的是向该实例的指针发送消息，实例在收到消息后，从自身的实现中寻找响应这条消息的方法。

关于传统的函数编译时，把参数信息和函数打包进编译后的源码，以及调用过程，可以参看：Bang的[如何动态调用 C 函数](http://blog.cnbang.net/tech/3219/)

OC 的编译过程之所以不一样，是因为在汇编过程，被苹果自己写的汇编接管了。

动态绑定所做的，即是在实例所属类确定后，将某些属性和相应的方法绑定到实例上。这里所指的属性和方法当然包括了原来没有在类中实现的，而是在运行时才需要的新加入的实现。在Cocoa层，我们一般向一个NSObject对象发送-respondsToSelector:或者-instancesRespondToSelector:等来确定对象是否可以对某个SEL做出响应，而在OC消息转发机制被触发之前，对应的类的+resolveClassMethod:和+resolveInstanceMethod:将会被调用，在此时有机会动态地向类或者实例添加新的方法，也即类的实现是可以动态绑定的。

一个例子：
```
void dynamicMethodIMP(id self, SEL _cmd)
{
    // implementation ....
}

//该方法在OC消息转发生效前被调用
+ (BOOL) resolveInstanceMethod:(SEL)aSEL
{ 
    if (aSEL == @selector(resolveThisMethodDynamically)) {
        //向[self class]中新加入返回为void的实现，SEL名字为aSEL，实现的具体内容为dynamicMethodIMP class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, “v@:”);
        return YES;
    }
    return [super resolveInstanceMethod:aSel];
}
```
当然也可以在任意需要的地方调用`class_addMethod`或`method_setImplementation`（前者添加实现，后者替换实现），来完成动态绑定的需求。

关于动态绑定，我的理解例子是：

假设程序里有Person这么一个类，它有name、age、height 以及方法-eat（假设除name、age、height 和-eat 外，其他的属性和方法忽略），然后我们在某处创建了Person这个实例对象。

![](/img/blogs/runtime_base/img_02.webp)

如果这里理解的有误，欢迎指正。

刚开始这个实例对象就像白纸一样干净，不知道它的具体类型，也没有属性和方法。然后在动态类型阶段，确定它的实际类型。再经过动态绑定，才会为其绑定相应的属性和方法，这时候这个对象才算完整了。

关于runtime 的一些基础知识就先到这里了。


