---
layout:     post
title:      "iOS下JS与OC互相调用（七）--Cordova 基础"
date:       2015-11-23
author:     "Haley-Wong"
catalog:    true
tags:
    - JS与iOS Native交互
---

### Cordova 简介
在介绍Cordova之前，必须先提一下PhoneGap。PhoneGap 是Nitobi软件公司2008年推出的一个框架，旨在弥补web 和iOS 之间的不足，使得web 和 iPhone SDK 之间的交互更容易。后来又加入了Android SDK 和BlackBerry SDK，再然后又陆续加入了更多的平台。

但是在2011年，Nitobi公司被Adobe收购，PhoneGap也被提交到Apache Incubator。由于Adobe现在拥有PhoneGap商标，PhoneGap v2.0版产品就更名为Apache Cordova。
据说Cordova是Nitobi团队当时坐落的街道名称，用此名来纪念Nitobi团队的贡献。Apache Cordova是从PhoneGap中抽出的核心代码，是驱动PhoneGap的核心引擎。

![](/img/blogs/js-native-7/img_01.jpg)

上图是目前Cordova支持的平台,摘自[Cordova官网](http://cordova.apache.org/)，它们分别是Android、iOS、Windows Phone、BlackBerry、ubuntu、火狐、LGwebOS、FireOS。

### Cordova工程结构
从下面这幅图可以看出Cordova工程的结构，以及与Native API 之前的关系。
![](/img/blogs/js-native-7/img_02.jpg)

### Get Started Fast
官网中也把Cordova 的使用划分了一些步骤。按照这五个步骤，可以很容易的创建一个Cordova Demo 工程起来，但是实际的使用要比这个Demo 工程复杂的多。

#### 1. 安装Cordova 
Cordova 命令行需要运行在 [Node.js](http://nodejs.org/)  上，在 [NPM](https://npmjs.org/package/cordova) 也可用。我们可以按照 [platform specific guides](http://cordova.apache.org/docs/en/latest/index.html#develop-for-platforms) 去安装别的依赖平台。因此，在安装Cordova之前，要先安装Node.js 和 NPM（可以参考文章[Mac 下安装Node.js](http://www.jianshu.com/p/f21fdbdf47df)）。打开命令行提示符(Windows 下) 或者 终端 (Mac 下)，然后输入 `npm install -g cordova` 即可安装Cordova。

如果安装失败，看到下面的错误提示信息，说明我们要用管理员身份安装。

![](/img/blogs/js-native-7/img_03.jpg)

以管理员身份安装Cordova的命令：
```
sudo npm install -g cordova
```
安装过程可能比较慢，安装成功后，可以看到类似如下的目录结构，并且没有错误信息：

![](/img/blogs/js-native-7/img_04.jpg)

#### 2.创建一个工程 
用命令行工具创建一个空的Cordova工程。首先跳转到 你希望保存新工程的文件夹（命令是 `cd 文件夹路径`），然后输入命令 `cordova create 工程名`。

当然，我们也可以直接 输入命令 `cordova create 文件夹路径/工程名`，在某个文件夹下直接创建工程。

查看更多的创建工程命令，可以输入命令 `cordova help create`。

我在终端中输入如下命令：
```
cordova create /Users/harvey/Desktop/Other/MyApp 

```
然后在Other 文件夹中就创建了一个叫MyApp的文件夹，目录结构如下：

![](/img/blogs/js-native-7/img_05.jpg)

#### 3.添加平台 
创建完Cordova 工程之后，跳转到工程文件夹（命令是 `cd 文件夹路径`）。

我这里使用的命令是:
```
cd /Users/harvey/Desktop/Other/MyApp
```
然后在这个文件夹中，我们需要添加一个 App 需要支持的平台。 添加一个平台，需要输入命令:
```
cordova platform add <platform name>
```
例如我们需要支持浏览器，那么就输入:
```
cordova platform add browser
```
如果我们需要支持iOS，那么就输入:
```
cordova platform add ios
```
> 注意ios 要小写。

查看Cordova可以支持的平台，可以输入 :
```
cordova platform
```
我输入`cordova platform`之后，终端显示的结果：
```
HarveydeMac-mini:MyApp harvey$ cordova platform
Installed platforms:
  browser 4.1.0
  ios 4.2.1
Available platforms: 
  amazon-fireos ~3.6.3 (deprecated)
  android ~5.2.0
  blackberry10 ~3.8.0
  firefoxos ~3.6.3
  osx ~4.0.1
  webos ~3.7.0
```
`Installed platforms` 是我已经安装过的平台，`Available platforms` 是还可以安装的平台。

#### 4.运行 App 
使用命令行工具，运行App的命令是：
```
cordova run <platform name>
```
例如，我想在浏览器中运行 App，我就在终端里输入：
```
cordova run browser

```
然后，就会打开浏览器，就会运行App。下面是我的命令和运行效果图：
![](/img/blogs/js-native-7/img_06.jpg)

当然，如果我们想要在iOS 上运行 App,我们也可以输入：
```
cordova run ios
```
也可以到指定目录下打开iOS 工程文件

![](/img/blogs/js-native-7/img_07.jpg)

查看更多的关于运行App 的命令，可以输入 `cordova help run`。

