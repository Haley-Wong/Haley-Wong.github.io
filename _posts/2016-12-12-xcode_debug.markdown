---
layout:     post
title:      "Xcode 调试技巧 --常用命令和断点"
date:       2016-12-12
author:     "Haley_Wong"
catalog:    true
tags:
    - Tools
---

Xcode 中的调试技巧与我们的日常开发息息相关，而这些调试技巧在我们解决Bug时，常常有事半功倍的作用，经常会用到的有各种断点 和 命令。而这些调试技巧也经常会在面试中问到，所以不知道的就来看看吧。

![调试主要观看区](/img/blogs/xcode_debug/img_01.webp)

## 调试命令
 在上图中，右侧绿色区域就是Log 输出区，在 Log 输出区可以使用一些命令，来辅助调试。
 
那有哪些调试命令呢？

想要看所有的调试命令，可以在上图的右侧区域输入`help`，就会列出所有的调试命令。
本文就介绍几个使用频率比较高的，其他就查看后，自行了解吧。

### 1. p 命令
```
-- ('expression --')  Evaluate an expression on the current thread.
                      Displays any returned value with LLDB's default formatting.
```
 p 命令是 print 命令的简写，使用p 命令可以查看基本数据类型的值，但是如果 使用 p 命令 查看的是对象，那么只会返回对象的指针地址。
 
p 命令后面除了可以接 变量、常量，还可以接 表达式。（❌但是不可以使用宏❌）

### 2. po 命令
po 命令可以理解为打印对象。功能与 p 命令类似，所以也是可以打印 常量、变量，打印表达式返回的对象等。（❌也不可以打印宏❌）

![p 和 po 使用范例](/img/blogs/xcode_debug/img_02.webp)

当然，这些打印功能，除了使用命令外，我们也可以使用左侧区域，点击变量右键---> print Description of “xxx”：

![Paste_Image.png](/img/blogs/xcode_debug/img_03.webp)

当然还有其他的打印方法：

![](/img/blogs/xcode_debug/img_04.webp)

### 3.expr 命令

expr 是 expression 的简写， 使用expr 命令，能够在调试时，动态的执行赋值表达式，同时打印出结果。我们可以在调试时，动态的修改变量的值，这在调试想要让应用执行异常路径（如执行某个else 情况）很有用。

```
(lldb) p i 
(NSInteger) $16 = 1
(lldb) expression i = 5
(NSInteger) $17 = 5
(lldb) po i 
5
```

### 4.call 命令
上面是动态修改变量的值， Xcode 还支持动态调用函数。在控制台执行该命令，可以在不修改代码，不重新编译的情况下，修改界面上的视图。

这里有一个动态将cell 的某个子视图移除的范例：
```
(lldb) po cell.contentView.subviews
<__NSArrayM 0x60800005f5f0>(
<UILabel: 0x7f91f4f18c90; frame = (5 5; 300 25); text = '2 - Drawing index is top ...'; userInteractionEnabled = NO; tag = 1; layer = <_UILabelLayer: 0x60800009ff40>>,
<UIImageView: 0x7f91f4d20050; frame = (105 20; 85 85); opaque = NO; userInteractionEnabled = NO; tag = 2; layer = <CALayer: 0x60000003ff60>>,
<UIImageView: 0x7f91f4f18f10; frame = (200 20; 85 85); opaque = NO; userInteractionEnabled = NO; tag = 3; layer = <CALayer: 0x608000039860>>
)

(lldb) call [label removeFromSuperview]
(lldb) po cell.contentView.subviews
<__NSArrayM 0x600000246de0>(
<UIImageView: 0x7f91f4d20050; frame = (105 20; 85 85); opaque = NO; userInteractionEnabled = NO; tag = 2; layer = <CALayer: 0x60000003ff60>>,
<UIImageView: 0x7f91f4f18f10; frame = (200 20; 85 85); opaque = NO; userInteractionEnabled = NO; tag = 3; layer = <CALayer: 0x608000039860>>
)
```

### 5.bt命令
`bt` 命令 可以打印出线程的堆栈信息，该信息比左侧的Debug Navigator 看到的还要详细一些。

`bt` 命令是打印当前线程的堆栈信息 
```
(lldb) bt 
* thread #1: tid = 0x27363, 0x000000010d204125 TestDemo`-[FifthViewController tableView:cellForRowAtIndexPath:](self=0x00007f91f4e153c0, _cmd="tableView:cellForRowAtIndexPath:", tableView=0x00007f91f5889600, indexPath=0xc000000000400016) + 2757 at FifthViewController.m:91, queue = 'com.apple.main-thread', stop reason = breakpoint 6.1
  * frame #0: 0x000000010d204125 TestDemo`-[FifthViewController tableView:cellForRowAtIndexPath:](self=0x00007f91f4e153c0, _cmd="tableView:cellForRowAtIndexPath:", tableView=0x00007f91f5889600, indexPath=0xc000000000400016) + 2757 at FifthViewController.m:91
    frame #1: 0x0000000111d0a7b5 UIKit`-[UITableView _createPreparedCellForGlobalRow:withIndexPath:willDisplay:] + 757
    frame #2: 0x0000000111d0aa13 UIKit`-[UITableView _createPreparedCellForGlobalRow:willDisplay:] + 74
    frame #3: 0x0000000111cde47d UIKit`-[UITableView _updateVisibleCellsNow:isRecursive:] + 3295
    frame #4: 0x0000000111d13d95 UIKit`-[UITableView _performWithCachedTraitCollection:] + 110
    frame #5: 0x0000000111cfa5ef UIKit`-[UITableView layoutSubviews] + 222
    frame #6: 0x0000000111c61f50 UIKit`-[UIView(CALayerDelegate) layoutSublayersOfLayer:] + 1237
    frame #7: 0x00000001117a5cc4 QuartzCore`-[CALayer layoutSublayers] + 146
    frame #8: 0x0000000111799788 QuartzCore`CA::Layer::layout_if_needed(CA::Transaction*) + 366
    frame #9: 0x0000000111799606 QuartzCore`CA::Layer::layout_and_display_if_needed(CA::Transaction*) + 24
    frame #10: 0x0000000111727680 QuartzCore`CA::Context::commit_transaction(CA::Transaction*) + 280
    frame #11: 0x0000000111754767 QuartzCore`CA::Transaction::commit() + 475
    frame #12: 0x00000001117550d7 QuartzCore`CA::Transaction::observer_callback(__CFRunLoopObserver*, unsigned long, void*) + 113
    frame #13: 0x0000000110743e17 CoreFoundation`__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__ + 23
    frame #14: 0x0000000110743d87 CoreFoundation`__CFRunLoopDoObservers + 391
    frame #15: 0x0000000110728b9e CoreFoundation`__CFRunLoopRun + 1198
    frame #16: 0x0000000110728494 CoreFoundation`CFRunLoopRunSpecific + 420
    frame #17: 0x0000000114390a6f GraphicsServices`GSEventRunModal + 161
    frame #18: 0x0000000111b9d964 UIKit`UIApplicationMain + 159
    frame #19: 0x000000010d21294f TestDemo`main(argc=1, argv=0x00007fff529fe620) + 111 at main.m:14
    frame #20: 0x000000011458a68d libdyld.dylib`start + 1
(lldb) 
```

`bt  all` 命令是打印所有线程的堆栈信息。打印出来的信息太多，就不展示了！

### 6.image 命令
`image  list` 命令可以列出当前App中的所有module（这个module 在后面符号断点时有用到），可以查看某一个地址对应的代码位置。

除了  `image  list` 还有 `image  add`、`image  lookup`等命令，可以自行查看。

当遇到crash 时，查看线程栈，只能看到栈帧的地址，使用 `image lookup –address 地址` 可以方便的定位到这个地址对应的代码行。

## 断点
Xcode 中的断点也是很有学问的，有普通断点、条件断点、符号断点、异常断点等很多种。

### 1.普通断点
打一个普通断点，只需要找到对应的行，在代码左侧（行号上）点击一下即可。

### 2.条件断点
条件断点是一种很有用的断点，特别是在for 循环中。如果我们需要在i = 5 时添加断点，其他时候不加，那么就可以使用条件断点。条件断点是在普通断点上 右键，选择 `Edit Breakpoint...`，再设置一个条件即可

![编辑普通断点](/img/blogs/xcode_debug/img_05.webp)

![添加条件](/img/blogs/xcode_debug/img_06.webp)

### 3.符号断点 
符号断点就是 `Symbolic Breakpoint`，其实是针对某一个特定函数的断点，可以是一个 OC函数，也可以是 C++函数。 添加的地方如下：

![符号断点](/img/blogs/xcode_debug/img_07.webp)


![符号断点条件](/img/blogs/xcode_debug/img_08.webp)

Symbol 栏 可以填  [类名 方法名]或者 方法名 ，module 也是选填项，它就是上面 `image`
 命令中列出来的module。
 
例如 ，我们如果只填一个viewDidLoad，那么就会在所有类（包括第三方库）的viewDidLoad 处打断点。

符号断点在调试一些没有源码的模块时比较有用，比如调试一个第三方提供的Lib库，或者系统的模块，可以在相应函数处下断点，可以大概调试清楚程序的运行流程，也可以在断点的时候查看到参数信息。

### 4.异常断点 
如果程序运行就崩溃，我们可以打一个异常断点，这样崩溃时就会触发断点，很容易定位到问题所在，也能看到更多的崩溃相关信息，如Log，函数调用栈。

![异常断点](/img/blogs/xcode_debug/img_09.webp)

![可以修改异常断点的条件](/img/blogs/xcode_debug/img_10.webp)

> 注意： 有的程序或者有的功能可能会使用异常来组织程序逻辑，比如调用AVAudioPlayer ，运行到 AVAudioPlayer 时，就会导致断点被触发。我们可以修改 Exception 参数，或者取消掉异常断点来解决。

### 5.Watch 断点
当某个变量发生变化的时候会触发。

创建一个Watch断点：

![Watch 断点](/img/blogs/xcode_debug/img_11.webp)

关于 Xcode 调试技巧中的 断点和命令就先这么多了，其他有用到的以后再补充。

Have Fun!







 
  


