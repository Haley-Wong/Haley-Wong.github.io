---
layout:     post
title:      "算法之路---归并排序"
date:       2019-05-09
author:     "Haley_Wong"
catalog:    true
tags:
    - 算法
---

[归并排序](https://baike.baidu.com/item/%E5%BD%92%E5%B9%B6%E6%8E%92%E5%BA%8F/1639015?fr=aladdin)是建立在归并操作上的一种有效的排序算法，该算法是采用分治法的一个非常典型的应用。分治法就是将一个大问题分解成小问题然后递归求解，然后再将小问题的结果合并，最终得到问题的解。

## 归并操作

归并操作，也叫归并算法，指的是将两个有序的序列合并成一个有序的序列的方法。

## 算法描述

归并操作的工作原理如下：

第一步：申请空间，使其大小为两个有序序列之和，该空间用来存放合并后的序列。

第二步：设定两个指针，初始位置分别为两个有序列表的起始位置。

第三步：比较两个指针所指向的元素，选择相对小的元素放入到合并空间（即之前申请的空间内），并移动指针到下一位置。

重复步骤3，知道某一指针超过序列尾。

最后，将另一序列剩下的元素直接复制到合并序列的尾部。

## 稳定性

归并排序是稳定的排序，即相等的元素的顺序不会改变。归并排序的比较次数小于快速排序的比较次数，但是移动次数一般多余快速排序的移动次数。一般用于总体无序，但是各子项相对有序的序列。

## 归并排序的OC实现
归并排序又分为自上而下和自下而上两种方式。

* 自上而下，就是利用递归（或者迭代）将目标序列先拆分成两个小序列，然后对小序列再分别执行归并排序，以此类推直到小序列中只有一个元素，然后开始合并小序列。
* 自下而上，就是直接从目标序列中，将两个相邻的元素两两归并，分别得到归并后的序列，再进行两两归并，直到最后只有一个序列，即为排序完成后的序列。

因为归并排序是比较好理解的一种排序算法，所以不做过多赘述，直接来看实现吧。

自上而下的归并排序：

```
/**
 归并排序(自上而下)
 
 @param randomNumbers 随机数组
 @return 排序后的数组
 */
+ (NSMutableArray *)mergeSort:(NSMutableArray *)randomNumbers
{
    if (randomNumbers.count <= 1) {
        return randomNumbers;
    }
    
    if (![randomNumbers isKindOfClass:[NSMutableArray class]]) {
        NSLog(@"参数类型错误，请使用NSMutableArray类型对象做入参");
        return nil;
    }
    
    // 创建一个临时数组，以防递归过程中频繁创建数组耗时
    NSMutableArray *tempArray = [NSMutableArray arrayWithCapacity:randomNumbers.count];
    
    [self sort:randomNumbers low:0 high:(int)randomNumbers.count-1 tempArray:tempArray];
    
    return randomNumbers;
}

+ (void)sort:(NSMutableArray *)randomNumbers low:(int)low high:(int)high tempArray:(NSMutableArray *)tempArray
{
    if (high <= low) {
        return ;
    }
    
    int mid = low + (high - low) / 2;
    
    [self sort:randomNumbers low:low high:mid tempArray:tempArray];
    [self sort:randomNumbers low:mid + 1 high:high tempArray:tempArray];
    [self merge:randomNumbers low:low mid:mid high:high tempArray:tempArray];
}

+ (void)merge:(NSMutableArray *)randomNumbers low:(int)low mid:(int)mid high:(int)high tempArray:(NSMutableArray *)aux
{
    int i = low;
    int j = mid + 1;
    // 先将数据存到临时数组中
    for (int z = low; z <= high; z++) {
        aux[z] = randomNumbers[z];
    }
    
    for (int k = low; k <= high; k++) {
        if (i > mid) {
            // 说明左子序列的元素都已用完
            randomNumbers[k] = aux[j++];
        } else if (j > high) {
            // 说明右子序列的元素都已用完
            randomNumbers[k] = aux[i++];
        } else if ([aux[i] intValue] < [aux[j] intValue]) {
            randomNumbers[k] = aux[i++];
        } else {
            randomNumbers[k] = aux[j++];
        }
    }
}

```

而自下而上的归并排序是这样的：

```
/**
 归并排序(自下而上)
 
 @param randomNumbers 随机数组
 @return 排序后的数组
 */
+ (NSMutableArray *)mergeSort2:(NSMutableArray *)randomNumbers
{
    if (randomNumbers.count <= 1) {
        return randomNumbers;
    }
    
    if (![randomNumbers isKindOfClass:[NSMutableArray class]]) {
        NSLog(@"参数类型错误，请使用NSMutableArray类型对象做入参");
        return nil;
    }
    
    int count = (int)randomNumbers.count;
    NSMutableArray *tempArray = [NSMutableArray arrayWithCapacity:randomNumbers.count];
    for (int sz = 1; sz < count; sz = sz + sz) {
        for (int lo = 0; lo < count - sz; lo += (sz + sz)) {
            // 让前两个序列归并，因为一个序列有sz个元素，所以中间元素是lo+sz-1，最大的元素是lo+sz+sz-1。
            // 为了防止越界，high 要去 lo + sz + sz -1 与 count -1中较小的元素
            int high = MIN(lo + sz + sz - 1, count - 1);
            [self merge:randomNumbers low:lo mid:lo + sz - 1 high:high tempArray:tempArray];
        }
    }
    
    return randomNumbers;
}

+ (void)merge:(NSMutableArray *)randomNumbers low:(int)low mid:(int)mid high:(int)high tempArray:(NSMutableArray *)aux
{
    int i = low;
    int j = mid + 1;
    // 先将数据存到临时数组中
    for (int z = low; z <= high; z++) {
        aux[z] = randomNumbers[z];
    }
    
    for (int k = low; k <= high; k++) {
        if (i > mid) {
            // 说明左子序列的元素都已用完
            randomNumbers[k] = aux[j++];
        } else if (j > high) {
            // 说明右子序列的元素都已用完
            randomNumbers[k] = aux[i++];
        } else if ([aux[i] intValue] < [aux[j] intValue]) {
            randomNumbers[k] = aux[i++];
        } else {
            randomNumbers[k] = aux[j++];
        }
    }
}
```

Have Fun!








 
  


