---
layout:     post
title:      "算法之路---汉诺塔（又称河内之塔）"
date:       2017-03-13
author:     "Haley_Wong"
catalog:    true
tags:
    - 算法
---

[汉诺塔](https://zh.wikipedia.org/wiki/%E6%B1%89%E8%AF%BA%E5%A1%94)是很简单也很经典的算法之一。

**汉诺塔**是根据一个传说形成的数学问题：

有三根杆子A，B，C 。A杆上有N个(N>1)穿孔圆盘，盘的尺寸由下到上依次变小。要求按下列规则将所有圆盘移至C杆：

* 1 每次只能移动一个圆盘；
* 2 大盘不能叠在小盘上面。

提示：可将圆盘临时置于B杆，也可以将A杆移除的圆盘重新移动回A杆，但都必须遵循上述两条规则。

问：如何移？最少要移动多少次？


![](/img/blogs/towerofHanoi/img_01.webp)

![3个圆盘的汉诺塔移动](/img/blogs/towerofHanoi/img_02.webp)


![4个圆盘的汉诺塔移动](/img/blogs/towerofHanoi/img_03.webp)


##传说
最早发明这个问题的人是[法国数学家爱德华*卢卡斯](https://zh.wikipedia.org/wiki/%E7%88%B1%E5%BE%B7%E5%8D%8E%C2%B7%E5%8D%A2%E5%8D%A1%E6%96%AF)。

传说印度某间寺院有三根柱子，上串64个金盘。寺院里的僧侣依照一个古老的预言，以上述规则移动这些盘子；预言说当这些盘子移动完毕，世界就会灭亡。这个传说叫做[梵天](https://zh.wikipedia.org/wiki/%E6%A2%B5%E5%A4%A9)寺之塔问题(Tower of Brahma puzzle)。但不知道是卢卡斯自创的这个传说，还是他受他人启发。

若传说属实，僧侣们需要2<sup>64</sup>− 1步才能完成这个任务；若他们每秒可完成一个盘子的移动，就需要5849亿年才能完成。整个宇宙现在也不过137亿年。

这个传说有若干变体：寺院换成修道院、僧侣换成修士等等。寺院的地点众说纷纭，其中一说是位于[越南](https://zh.wikipedia.org/wiki/%E8%B6%8A%E5%8D%97)的[河内](https://zh.wikipedia.org/wiki/%E6%B2%B3%E5%85%A7)，所以被命名为“河内塔”。另外亦有“金盘是[创世](https://zh.wikipedia.org/wiki/%E5%88%9B%E4%B8%96)时所造”、“僧侣们每天移动一盘”之类的背景设定。

佛教中确实有“浮屠”（[塔](https://zh.wikipedia.org/wiki/%E5%A1%94)）这种建筑；有些浮屠亦遵守上述规则而建。“河内塔”一名可能是由[中南半岛](https://zh.wikipedia.org/wiki/%E4%B8%AD%E5%8D%97%E5%8D%8A%E5%B3%B6)在殖民时期传入欧洲的。

## 解答
如取N=64，最少需移动2<sup>64</sup>− 1次。即如果一秒钟能移动一块圆盘，仍将需5849.42亿年。目前按照[宇宙大爆炸理论](https://zh.wikipedia.org/wiki/%E5%AE%87%E5%AE%99%E5%A4%A7%E7%88%86%E7%82%B8%E7%90%86%E8%AE%BA)的推测，宇宙的年龄仅为137亿年。

在真实玩具中，一般N=8；最少需移动255次。如果N=10，最少需移动1023次。如果N=15，最少需移动32767次；这就是说，如果一个人从3岁到99岁，每天移动一块圆盘，他最多仅能移动15块。如果N=20，最少需移动1048575次，即超过了一百万次。

## 解法
解法的基本思想是递归。假设有A、B、C 三个塔，A塔有N块盘，目标是把这些盘全部移动到C塔。那么先把塔顶部的N-1块盘移动到B塔，再把A塔剩下的大盘移动到C，最后把B塔的N-1块盘移动到C。

每次移动多于一块盘时，则再次使用上述算法来移动。

怎么来理解呢？

我们可以倒着理解，要将A塔上的所有圆盘移动到C塔，且所有圆盘是下大上小。那么必定有一个过程是最大的圆盘（也就是第N个圆盘）从A移动到C。那么必须保证上面的N-1个圆盘全在B塔，且圆盘满足上面小，下面大。当第N个圆盘从A移动到C之后，又得把N-1个圆盘从B塔移动到C塔，这样工作就完成了。

但是怎么把A塔上的N-1个圆盘移动到B塔呢？

这里需要一点想象力，可以想象成只有N-1个圆盘，从A塔移动到B塔（此时的B塔其实就相当于上面的C塔），我们称A塔为A1塔，B塔为C1塔，C塔为B1塔，那么问题就变成了如何将N-1个盘从A1塔移动到C1塔？

同样的需要将上面的N-2个圆盘从A1塔移动到B1塔，然后将第N-1个圆盘从A1塔移动到C1塔，然后再将B1塔上的N-2个圆盘移动到C1塔。

同理，递推第N-2个塔.....。

将 B塔上的 N-1个盘，移动到C塔的过程与上面原理一样。

## 递归解
以Objective-C语言实现：
```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view from its nib.
    int n = 9;
    [self hannoi:n from:@"A" buffer:@"B" to:@"C"];
    NSLog(@"一共 %d 步",count);
}

- (void)hannoi:(int)n from:(NSString *)from buffer:(NSString *)buffer to:(NSString *)to
{
    if (n == 1) {
        NSLog(@"Move disk %d from %@ to %@", n, from, to);
    } else {
        [self hannoi:n-1 from:from buffer:to to:buffer];
        NSLog(@"Move disk %d from %@ to %@", n, from, to);
        [self hannoi:n-1 from:buffer buffer:from to:to];
    }
}

console : 一共 511 步
```
以C++语言实现：
```
using namespace std;
#include <iostream>
#include <cstdio>

void hannoi (int n, char from, char buffer, char to)
{
    if (n == 1) {
        cout << "Move disk " << n << " from " << from << " to " << to << endl;
    } else {
        hannoi (n-1, from, to, buffer);
        cout << "Move disk " << n << " from " << from << " to " << to << endl;
        hannoi (n-1, buffer, from, to);
    }
}

int main()
{
    int n;
    cin >> n;
    hannoi (n, 'A', 'B', 'C');
    return 0;
}
```
通过以上递归过程可知hannoi(n) = 2 * hannoi(n-1) + 1 ，hannoi(n) = 2<sup>n-1</sup> + 2<sup>n-2</sup> + 2<sup>n-3</sup>+ ...... + 2<sup>2</sup> + 2 +1。
对等比数列求和得出hannoi(n) = 2<sup>n</sup> -1。

Have Fun!








 
  


