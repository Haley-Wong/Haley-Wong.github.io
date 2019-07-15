---
layout:     post
title:      "Xcode 创建.a和framework静态库"
date:       2016-01-22
author:     "Haley_Wong"
catalog:    true
tags:
    - Tools
---

最近因为项目中的聊天SDK，需要封装成静态库，所以实践了一下创建静态库的步骤，做下记录。

# 库介绍
库从本质上来说是一种可执行代码的二进制格式，可以被载入内存中执行。库分静态库和动态库两种。
iOS中的静态库有 .a 和 .framework两种形式；动态库有.dylib 和 .framework 形式，后来.dylib动态库又被苹果替换成.tbd的形式。

# 静态库与动态库的区别

静态库和动态库是相对编译期和运行期的：静态库在程序编译时会被链接到目标代码中，程序运行时将不再需要改静态库；而动态库在程序编译时并不会被链接到目标代码中，只是在程序运行时才被载入，因为在程序运行期间还需要动态库的存在。

总结：同一个静态库在不同程序中使用时，每一个程序中都得导入一次，打包时也被打包进去，形成一个程序。而动态库在不同程序中，打包时并没有被打包进去，只在程序运行使用时，才链接载入（如系统的框架如UIKit、Foundation等），所以程序体积会小很多，但是苹果不让使用自己的动态库，否则审核就无法通过。

# 创建.a静态库
第一步，新建工程。一般使用工程名就使用库的名称，比如我这里用FMDB来创建静态库，我的工程名就取名为FMDB，创建的.a静态库就是libFMDB.a。

![使用静态库模板新建工程.png](/img/blogs/ios-framework/img_01.jpeg)

![创建的工程.png](/img/blogs/ios-framework/img_02.png)

第二步，删除系统默认创建的【FMDB.h】和【FMDB.m】文件，导入需要打包的源文件。

![导入源文件后.png](/img/blogs/ios-framework/img_03.jpeg)

第三步(方式一)，修改项目配置

![修改配置.png](/img/blogs/ios-framework/img_04.jpeg)
点击上图中的【3】，弹出的列表中选择【New Headers Phase】,打开【Headers (0 items)】，点击左下角的【+】，选择所有的.h文件。

![配置需要暴漏的文件的.h头.png](/img/blogs/ios-framework/img_05.jpeg)

第三步（方式二），修改项目配置
![修改项目配置.png](/img/blogs/ios-framework/img_06.jpeg)

第四步，修改导出product配置
![修改编译配置.png](/img/blogs/ios-framework/img_07.jpeg)

第五步，修改编译指令集
![设置Release为NO.png](/img/blogs/ios-framework/img_08.jpeg)

模拟器：iPhone4s~5 : i386 iPhone5s~6plus : x86_64

真机：iPhone3gs~4s : armv7  iPhone5~5c : armv7s  iPhone5s~6plus : arm64

如果第五步这里，设置为YES，那么编译出来的.a静态库就只包含当前设备的指令集。

举个例子：如果我们选择iPhone 5模拟器【Command+B】编译，则编译出来的.a静态库只能用iPhone4s~5模拟器跑程序，用iPhone5s~6plus，则会报找不到x86_64的libFMDB库。

设置为NO，则会把所有指令集的都打包合并。

第六步，编译（快捷键【Command+B】

编译时，需要用模拟器和真机各编译一次，这样Products目录下的libFMDB.a静态库才会变为黑色，右键show in Finder，可以进入Products目录下。

![编译结果.png](/img/blogs/ios-framework/img_09.jpeg)

为什么需要用模拟器和真机各编译一次呢？

可以看到Products目录下有【Release-iphoneos】和【Release-iphonesimulator】两个文件件。前者里面是真机使用的.a静态库，后者是模拟器使用的.a静态库。

> 注意：如果步骤四中，不将Build Configuration改为Release,则打包出来的静态库会存于【Debug-iphoneos】和【Debug-iphonesimulator】两个文件夹下。
我们一般都使用Release模式，因为程序最终发布之后是Release版的，所以静态库也是在Release模式下使用。

如果想要通用需要将模拟器使用的静态库与真机使用的静态库合并成一个静态库，可以使用终端命令来实现。命令格式： 

lipo -create 第一个.a文件的绝对路径  第二个.a文件的绝对路径 -output 最终的.a文件路径。

本文中使用的命令如下：
```
lipo -create /Users/harvey/Library/Developer/Xcode/DerivedData/FMDB-ctegiztcjikewoeprxxtmryzetfa/Build/Products/Release-iphoneos/libFMDB.a /Users/harvey/Library/Developer/Xcode/DerivedData/FMDB-ctegiztcjikewoeprxxtmryzetfa/Build/Products/Release-iphonesimulator/libFMDB.a -output /Users/harvey/Desktop/libFMDB.a
```
> 补充：经过多次实践，第三步的操作省略，依然可以导出可正常使用的包。
如果静态库中有category类，则在使用静态库的项目配置中【Other Linker Flags】需要添加参数【-ObjC]或者【-all_load】。

#创建framework静态库
第一步，新建项目

![新建项目.png](/img/blogs/ios-framework/img_10.jpeg)

第二步，删除系统默认创建的【FMDB.h】和【FMDB.m】文件，导入需要打包的源文件。

![导入源码后的工程.png](/img/blogs/ios-framework/img_11.png)

第三步，修改项目配置

首先，设置需要暴漏的头文件
![header文件设置.png](/img/blogs/ios-framework/img_12.jpeg)

> 这里需要注意的是暴露出来的头文件中import的其他类也得添加到public中暴露出来。
   如果不想将import的类暴露出来，那么在头文件中用@class 然后在对应的.m文件中再import。

然后，设置编译模式，在Xcode菜单【Product】--->【Scheme】--->【Edit Scheme...】中
![设置编译模式.png](/img/blogs/ios-framework/img_13.png)

再然后，设置编译出的静态库包含的指令集
![设置编译出的静态库包含的指令集.png](/img/blogs/ios-framework/img_14.png)

最后，修改生成的Mach-O格式

![修改Mach-O 格式.png](/img/blogs/ios-framework/img_15.png)

第四步，编译生成静态库

编译时，需要用模拟器和真机各编译一次，这样Products目录下的libFMDB.a静态库才会变为黑色，右键show in Finder，可以进入Products目录下。

![编译生成的framework静态库.png](/img/blogs/ios-framework/img_16.jpeg)

第五步，合并模拟器版framework和真机版framework

合并的命令同上面相似，不同之处是：framework静态库合并的不是framework,而是framework下的一个二进制文件，即上一步图中标记的文件。

lipo -create 第一个framework下二进制文件的绝对路径  第二个framework下二进制文件的绝对路径 -output 最终的二进制文件路径。

本文中使用的命令如下：
```
lipo -create /Users/harvey/Library/Developer/Xcode/DerivedData/FMDB-clvayfrjgytqrbdkyqrtcjkxfeuz/Build/Products/Release-iphonesimulator/FMDB.framework/FMDB /Users/harvey/Library/Developer/Xcode/DerivedData/FMDB-clvayfrjgytqrbdkyqrtcjkxfeuz/Build/Products/Release-iphoneos/Release-iphoneos.framework/FMDB -output /Users/harvey/Desktop/FMDB
```
最后将任何一个framework中的二进制文件替换成合并后的二进制文件即可，把framework添加到要使用的项目中即可使用。

> 注意：如果创建的framework中使用了category类，则在使用framework的项目配置中【Other Linker Flags】需要添加参数【-ObjC]或者【-all_load】。
> 
> 如果使用framework的使用出现【Umbrella header for module 'XXXX' does not include header 'XXXXX.h'】,是因为错把xxxxx.h拖到了public中。
>
>如果出现【dyld: Library not loaded:XXXXXX】，是因为打包的framework版本太高。比如打包framework时，选择的是iOS 9.0，而实际的工程环境是iOS 8开始的。

如果创建的framework类中使用了.dylib或者.tbd，首先需要在实际项目中导入.dylib或者.tbd动态库，然后需要设置【Allow Non-modular Includes ....】为YES，否则会报错"Include of non-modular header inside framework module"。

![设置【Allow Non-modular Includes ....】.png](/img/blogs/ios-framework/img_17.jpeg)

> 补充：打包成的静态库肯定是比源码类要大很多的，因为是由不同指令集不同设备的版本合并成的。所以如果你很在意你的app大小，并且也不是很需要打包成静态库的话，还是用原始类吧。
framework静态库中是可以包含图片资源的；而.a静态库中不能包含图片资源，只能另外创建一个目录存放。

## 填坑记录
上面的注意里提到了一些坑，以及解决办法。这里再记录一些：

**1.**framework中用到了`NSClassFromString`，但是转换出来的class 一直为nil。
先来看一下这个API的官方描述

![官方描述.png](/img/blogs/ios-framework/img_18.jpeg)

什么意思呢？

如果转换出来的class为nil，有两种情况：一种情况是这个类不存在；第二种情况是这个类还没有被load。所以一般出现问题，都是第二种情况。

怎么解决这个问题呢？

在主工程的【Other Linker Flags】需要添加参数【-ObjC]即可。

**2.**framework中把图片、音频打包进bundle中，但是一直加载不到。

打包的framework中有一个bundle,bundle里有一些图片、音频等资源。但是用如下方式：

```
NSString *bundlePath = [[NSBundle mainBundle] pathForResource:@"XXXX" ofType:@"bundle"];
NSString *mp3Path = [[NSBundle bundleWithPath:bundlePath] pathForResource:@"Message_system" ofType:@"mp3"];
```

获取的mp3Path一直为nil。framework，只暴露了一些.h头文件，bundle没有暴露出来，无法获取。那么我们只需要将bundle与framework一起放入目标工程中即可。其实bundle根本不用打包进framework中。

例如：
我们创建了一个叫ABC.framework的静态库。库中使用了`Message_system.mp3`,那么我们创建一个bundle，命名为ABC.bundle，然后将Message_system.mp3放入bundle中。打包的时候，framework并不包含ABC.bundle。最后在要使用ABC.framework的工程中，新建一个文件夹or group,然后把ABC.framework和ABC.bundle一起拖进去，就可以啦。

Have Fun!


