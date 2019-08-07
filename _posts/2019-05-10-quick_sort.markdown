---
layout:     post
title:      "算法之路---快速排序"
date:       2019-05-10
author:     "Haley_Wong"
catalog:    true
tags:
    - 算法
---

[快速排序](https://baike.baidu.com/item/%E5%BF%AB%E9%80%9F%E6%8E%92%E5%BA%8F%E7%AE%97%E6%B3%95/369842?fromtitle=%E5%BF%AB%E9%80%9F%E6%8E%92%E5%BA%8F&fromid=2084344&fr=aladdin)是对冒泡排序的一种改进。

快速排序由C. A. R. Hoare在1960年提出。它的基本思想是：通过一趟排序将要排序的数据分割成独立的两部分，其中一部分的所有数据都比另外一部分的所有数据都要小，然后再按此方法对这两部分数据分别进行快速排序，整个排序过程可以递归进行，以此达到整个数据变成有序序列。

快速排序应该是应用最广泛的排序算法了。因为它实现简单，使用于各种不同的输入数据且在一般应用中比其他排序算法快得多。快速排序引人注目的特点包括它是原地排序，而且将长度为N的数组排序所需的时间和NlgN成正比。

## 快速排序的基本算法
快速排序也是一种使用分治策略的排序算法。它将一个数组分成两个子数组，将两部分独立地排序。快速排序和归并排序是互补的：归并排序将数组分成两个子数组分别排序，并将有序的子数组归并以将整个数组排序；而快速排序将数组排序的方式是当两个子数组都有序时整个数组也就自然有序了。在归并排序中递归发生在处理整个数组之前；而在快速排序中递归发生在处理整个数组之后。在归并排序中，一个数组被等分为两半；在快速排序中，切分的位置区决与数组的内容。

快速排序是一种不稳定的排序算法。

## 快速排序的OC实现

```
/**
 快速排序

 @param randomNumbers 随机数组
 @return 排序后的数组
 */
+ (NSMutableArray *)quickSort:(NSMutableArray *)randomNumbers
{
    if (![randomNumbers isKindOfClass:[NSMutableArray class]]) {
        NSLog(@"参数类型错误，请使用NSMutableArray类型对象做入参");
        return nil;
    }

    if (randomNumbers.count <= 1) {
        return randomNumbers;
    }

    [self quick_sort:randomNumbers low:0 high:(int)randomNumbers.count - 1];

    return randomNumbers;
}

+ (void)quick_sort:(NSMutableArray *)randomNumbers low:(int)low high:(int)high
{
    if (low >= high) {
        return;
    }

    int j = [self partition:randomNumbers low:low high:high];
    [self quick_sort:randomNumbers low:low high:j-1];
    [self quick_sort:randomNumbers low:j+1 high:high];
}

+ (int)partition:(NSMutableArray *)randomNumbers low:(int)low high:(int)high
{
    // 将数组切分为a[low...j-1], a[j], a[j+1...high]
    int i= low, j = high + 1;
    NSNumber *element = randomNumbers[low];

    while (YES) {
        //扫描左右，检查扫描是否结束并交换元素
        while ([randomNumbers[++i] intValue] < [element intValue]) {
            if (i == high) {
                break;
            }
        }

        while ([randomNumbers[--j] intValue] > [element intValue]) {
            if (j == low) {
                break;
            }
        }

        if (i >= j) {
            break;
        }

        [randomNumbers exchangeObjectAtIndex:i withObjectAtIndex:j];
    }

    [randomNumbers exchangeObjectAtIndex:low withObjectAtIndex:j];
    return j;
}
```

考虑到待排序的数组序列中可能会有很多与切分元素相同的元素，这样会做很多无用的交换操作，我们可以考虑用三向切分法，也就是将数组中的元素切分为三个数组：a[low...lt-1], a[lt...gt], a[gt+1...high]。a[low...lt-1] 中的元素全部都小于切分的元素temp，a[lt...gt] 中的元素全部都等于切分的元素temp，a[gt+1...high] 中的元素全部都大于切分的元素temp。

三向切分法的快速排序实现如下：

```
/**
 快速排序（三向切分快速排序）
 
 @param randomNumbers 随机数组
 @return 排序后的数组
 */
+ (NSMutableArray *)quickSort2:(NSMutableArray *)randomNumbers
{
    if (![randomNumbers isKindOfClass:[NSMutableArray class]]) {
        NSLog(@"参数类型错误，请使用NSMutableArray类型对象做入参");
        return nil;
    }
    
    if (randomNumbers.count <= 1) {
        return randomNumbers;
    }
    
    [self quick_sort2:randomNumbers low:0 high:(int)randomNumbers.count - 1];
    
    return randomNumbers;
}

+ (void)quick_sort2:(NSMutableArray *)randomNumbers low:(int)low high:(int)high
{
    if (low >= high) {
        return;
    }
    //  三向切分的目的是将数组切分为a[low...lt-1], a[lt...gt], a[gt+1...high]
    // a[low...lt-1] 中的元素全部都小于切分的元素temp;
    // a[lt...gt] 中的元素全部都等于切分的元素temp;
    // a[gt+1...high] 中的元素全部都大于切分的元素temp;
    
    // 先假设所有元素都等于temp，每有一个元素小于temp，则lt++;每有一个元素大于temp，则gt--;
    int lt = low, i = low + 1, gt = high;
    NSNumber *temp = randomNumbers[low];
    
    while (i <= gt) {
        if ([randomNumbers[i] intValue] < [temp intValue]) {
            [randomNumbers exchangeObjectAtIndex:lt withObjectAtIndex:i];
            lt++;
            i++;
        } else if ([randomNumbers[i] intValue] > [temp intValue]) {
            [randomNumbers exchangeObjectAtIndex:gt withObjectAtIndex:i];
            gt--;
        } else {
            i++;
        }
    }
    
    [self quick_sort2:randomNumbers low:low high:lt -1];
    [self quick_sort2:randomNumbers low:gt + 1 high:high];
}

```

Have Fun!








 
  


