---
layout:     post
title:      "iOS工程修改名称"
date:       2016-10-09
author:     "Haley_Wong"
catalog:    true
tags:
    - Tools
---

我们在iOS开发中，难免会遇到项目做到一半要改名字的情况。如果项目名差的太大，工程名看起来总是不舒服的，有良心的开发者可能就会想着为工程改个贴切的名字，那么你就为用到本文记录的内容。

如果我们开发的两个项目相差不大，只有部分主题、布局有更改，那么我们就可以拷贝之前已经完成的项目，改改名字，再对部分界面和代码稍稍修改就可以啦。


## 如何修改工程名呢？
下面我就拿一个中等大小的项目来实际操作一下，并记录整个要修改的地方。
该项目的结构如下：

![项目结构](/img/blogs/ios_change_name/img_01.webp)

项目中还用到了几个第三方框架：

![第三方框架](/img/blogs/ios_change_name/img_02.webp)

接下来，就要开始修改项目名称了。假设我要把`doutu`改为`shopping`。

> 提醒：
> 
* 在改工程名之前，要注意三件事：一定要备份，一定要备份，一定要备份。
* 在开始第一步之前，请先执行第八步。

### 1.修改project名称
选中project 单击project名字   或者   选中project+回车。

![](/img/blogs/ios_change_name/img_03.webp)
修改project的名称之后，回车会有提示：
![](/img/blogs/ios_change_name/img_04.webp)

这里点击Rename，将project中部分`doutu`改为`shopping`。

修改之后，哪些地方有明显变化呢？

![Rename后](/img/blogs/ios_change_name/img_05.webp)

### 2.修改文件夹名称
`选中文件夹 单击文件夹名字`    或者 `选中文件夹  回车`。
![修改文件夹](/img/blogs/ios_change_name/img_06.webp)

修改之后，回车是这样的：

![修改文件夹名字后](/img/blogs/ios_change_name/img_07.webp)

虽然在Xcode 里文件夹的名字修改了，但是实际上文件夹里的名字还是没有修改，我们需要去真实文件夹里再修改一次。

![修改真实目录名称](/img/blogs/ios_change_name/img_08.webp)

### 3.修改工程中文件夹的路径
在上一步修改玩真实文件夹的名字后，工程中所有的类都变成了红色（文件找不到）。如下图所示：
![](/img/blogs/ios_change_name/img_09.webp)

主要是因为工程中的文件夹指引的路径不对。

![](/img/blogs/ios_change_name/img_10.webp)

按照如上步骤所示，找到我们刚才修改的真实`shopping`文件夹，点击Chose 即可。
这时候，因为文件夹关联的真实文件夹路径正确了，所有红色的文件都正常了。

这是修改后的样子：

![修改后](/img/blogs/ios_change_name/img_11.webp)

### 4.全局搜索
全局搜索`doutu`，搜索结果如下：

![7266902F-751B-42BE-BF13-FF84EB5E96BB.png](/img/blogs/ios_change_name/img_12.webp)

接下来是将`doutu`替换为`shopping`。

![DB9337FB-35A2-4509-84EB-BDF17C9BEA8C.png](/img/blogs/ios_change_name/img_13.webp)

点击Replace All之后，大部分`doutu`都会被替换为`shopping`，但是还是有一些顽固的残留。

![替换后](/img/blogs/ios_change_name/img_14.webp)

可以看出，这个是project 文件中，我的第三方框架的framework Search Paths 和Library Search Paths 的路径错误。即：

![](/img/blogs/ios_change_name/img_15.webp)

这里只需要将`doutu` 修改为 `shopping`即可。
怎么修改呢？
有两种方式，第一种双击`framework Search Paths` 和`Library Search Paths ` 后面的值，然后单独修改每个值。

![双击修改](/img/blogs/ios_change_name/img_16.webp)

第二种方案，先将`framework Search Paths` 和`Library Search Paths `中的值都删掉，然后把第三方删除，再重新添加。

![Paste_Image.png](/img/blogs/ios_change_name/img_17.webp)

这里点击Remove References删除，然后再把Vendor文件夹添加进工程即可。

### 5.修改pch文件路径
如果你的工程里添加了pch文件，因为修改了文件夹，project名字，所以pch文件夹路径也要修改。修改前编译运行，会报如下错误：

![pch文件找不到](/img/blogs/ios_change_name/img_18.webp)

在Build Settings 中搜索Prefix，修改Prefix Header 的值。

![](/img/blogs/ios_change_name/img_19.webp)

上面把`doutu/shopping-Prefix.pch`修改为`shopping/shopping-Prefix.pch`即可。

### 6.修改info.plist文件路径
此时再次编译运行，依然会有一个错误，错误如下：

![](/img/blogs/ios_change_name/img_20.webp)

然后依然去 `Build Setting` 中搜索info.plist。

![](/img/blogs/ios_change_name/img_21.webp)

上面将`doutu/Info.plist`修改为`shopping/Info.plist`即可。

到这里，工程应该已经可以正常运行了。

![Buid Succeeded](/img/blogs/ios_change_name/img_22.webp)

但是，如果你想追求完美，依然还有两个地方需要修改。
### 7.修改scheme 值
要修改的其实是这个地方的显示名称：

![](/img/blogs/ios_change_name/img_23.webp)

怎么修改呢？
点击scheme值，然后选择 `Manage Schemes...`

![](/img/blogs/ios_change_name/img_24.webp)

接下来会进入到一个弹出窗口，选中一行，`点击scheme值`或者 `回车`：

![](/img/blogs/ios_change_name/img_25.webp)

这里把`doutu` 修改为 `shopping`就会看到 scheme 变成了shopping，如下图所示：

![Paste_Image.png](/img/blogs/ios_change_name/img_26.webp)

### 8.修改大文件夹的名称

>其实这一步，应该在拷贝完工程后，直接修改的。所以这一步更应该放在第一步做。

![修改大文件夹的名称](/img/blogs/ios_change_name/img_27.webp)

### 9.修改推送文件的配置（补充）

从iOS 10 开始，工程里多了一个`entitlements`文件，所以修改完其他之后，还需要修改一下 `entitlements`文件的路径。 可以在 `Build Settings` -> `Signing` -> `Code Signing Entitlements` 中找到这个路径，修改为正确的文件路径即可。

当然，你也可以在5、6步的时候，顺便一起修改了。

到这里，就真的大功告成啦。Have Fun!

