---
layout:     post
title:      "算法之路---希尔排序"
date:       2019-05-07
author:     "Haley_Wong"
catalog:    true
tags:
    - 算法
---

[希尔排序(Shell's Sort)](https://baike.baidu.com/item/%E5%B8%8C%E5%B0%94%E6%8E%92%E5%BA%8F/3229428?fr=aladdin)是插入排序的一种又称“缩小增量排序”（Diminishing Increment Sort），是直接插入排序算法的一种更高效的改进版本。希尔排序是非稳定排序算法。该方法因D.L.Shell于1959年提出而得名。

希尔排序是基于插入排序的以下两点性质而提出改进方法的：

* 1.插入排序在对几乎已经排好序的数据操作时，效率高，即可以达到线性排序的效率。
* 2.但插入排序一般来说是低效的，因为插入排序每次只能将数据移动一位。

在极端情况下数据一次只能移动一位，所以需要移动很多次，那我们可以设计一个合适的值，一次移动多位，就可以提高插入排序的效率。

## 基本思想
假设我们先取一个小于n的整数d1，作为第一次的增量，然后分别对[0,d1,2*d1,3*d1....]，[1,1+d1,1+2*d1,....]，......等数组做插入排序。

然后，再取一个小于d1的整数d2，作为第二次的增量，再次对[0,d2,2*d2,3*d2....],[1,1+d2,1+2*d2,....]，......等数组做插入排序。

......

直到增量d为1时，整个要排序的数被分成一组，排序完成。

>怎么理解呢？
>我们可以这样理解：
>1.将原数组按照增量d1拆分成多个子数组，然后在子数组内做增量排序。
>2.再将原数组按照增量d2拆分成多个子数组，然后再在子数组内做增量排序。
>3.......
>4.直到增量d为1时，整个要排序的数组被分成一组，排序完成。

只不过，真实执行时，没必要真的将原数组拆分成子数组，这样会导致空间复杂度增加，也没什么必要。

## 稳定性
我们知道一次插入排序是稳定的，不会改变相同元素的相对顺序。但是在希尔排序中，一个元素可能会被移动的很远，所以相同的元素可能在各自的插入排序中移动，最后其稳定性就会被打乱，所以shell排序是不稳定的。

## 希尔排序的OC实现

```
/**
 希尔排序
 
 @param randomNumbers 随机数数组
 @return 排序后的数组
 */
+ (NSMutableArray *)shellSort:(NSMutableArray *)randomNumbers
{
    if (randomNumbers.count == 0) {
        return randomNumbers;
    }
    
    if (![randomNumbers isKindOfClass:[NSMutableArray class]]) {
        NSLog(@"参数类型错误，请使用NSMutableArray类型对象做入参");
        return nil;
    }
    
    NSUInteger count = randomNumbers.count;
    int d = (int)randomNumbers.count / 2;
    
    while (d > 1) {
        d = d / 2;
        for (int x = 0; x < d; x++) {
            for (int i = x + d; i < count; i += d) {
                id temp = randomNumbers[i];
                int j;
                for (j = i - d; j >= 0 && [randomNumbers[j] intValue] > [temp intValue]; j -= d) {
                    randomNumbers[j + d] = randomNumbers[j];
                }
                randomNumbers[j + d] = temp;
            }
        }
    }
    
    return randomNumbers;
}
```

## 算法分析

需要大量的辅助空间，和归并排序一样容易实现。希尔排序是基于插入排序的一种算法， 在此算法基础之上增加了一个新的特性，提高了效率。希尔排序的时间复杂度与增量序列的选取有关，Hibbard增量的希尔排序的时间复杂度为O(n<sup>3/2</sup>)，希尔排序时间复杂度的下界是n*log2n。

希尔排序没有快速排序算法快 O(n(logn))，因此中等大小规模表现良好，对规模非常大的数据排序不是最优选择，但是比O(n<sup>2</sup>)复杂度的算法快得多。并且希尔排序非常容易实现，算法代码短而简单。

## 增量序列的选择
Shell排序的执行时间依赖于增量序列。

好的增量序列的共同特征：

* 1.最后一个增量必须为1；
* 2.应该尽量避免序列中的值(尤其是相邻的值)互为倍数的情况。

所以，如果我们对希尔排序的增量序列进行优化，排序算法的时间会稍微减少一点点：

```
/**
 希尔排序(对区间算法优化)
 
 @param randomNumbers 随机数数组
 @return 排序后的数组
 */
+ (NSMutableArray *)shellSort2:(NSMutableArray *)randomNumbers
{
    if (randomNumbers.count <= 1) {
        return randomNumbers;
    }
    
    if (![randomNumbers isKindOfClass:[NSMutableArray class]]) {
        NSLog(@"参数类型错误，请使用NSMutableArray类型对象做入参");
        return nil;
    }
    
    NSUInteger count = randomNumbers.count;
    int d = 1;
    while (d < count / 3) {
        d = 3 * d + 1;
    }
    
    while (d >= 1) {
        for (int i = d; i < count; i++) {
            for (int j = i; j >= d && [randomNumbers[j] intValue] < [randomNumbers[j - d] intValue]; j -= d) {
                [randomNumbers exchangeObjectAtIndex:j withObjectAtIndex:j-d];
            }
        }
        d = d / 3;
    }
    
    return randomNumbers;
}
```

Have Fun!








 
  


