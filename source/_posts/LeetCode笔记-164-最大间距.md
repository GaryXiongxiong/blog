---
title: LeetCode笔记-用桶排序与鸽笼原则解决【164-最大间距】
tags:
- Algorithm
categories: 
- blog
date: 2020-10-21 11:10:39
---
给定一个无序的数组，找出数组在排序之后，相邻元素之间最大的差值。

如果数组元素个数小于 2，则返回 0。

- 你可以假设数组中所有元素都是非负整数，且数值在 32 位有符号整数范围内。
- 请尝试在线性时间复杂度和空间复杂度的条件下解决此问题。

<!--more-->

## 示例

#### 示例一

```
输入: [3,6,9,1]
输出: 3
解释: 排序后的数组是 [1,3,6,9], 其中相邻元素 (3,6) 和 (6,9) 之间都存在最大差值 3。
```

#### 示例二

```
输入: [10]
输出: 0
解释: 数组元素个数小于 2，因此返回 0。
```

## 思路

看到数**组元素全部为非负整数**以及**线性时间复杂度的排序**就想到了桶排序。桶排序简单来说就是把元素根据大小在一次遍历中放到对应的桶中。之后将每个桶中的元素依次取出组成排序后的数组。可参考[桶排序（箱排序）原理及其时间复杂度详解](http://data.biancheng.net/view/115.html)中的图解：

> 在某个期末考试中，老师要把大家的分数排序，比如有 5 个学生，分别考 5、9、5、1、6 分（满分 10 分），从大到小排序应该是 9、6、5、5、1，大家有没有办法写一段程序随机读取 5 个数，然后对它们排序呢？
>
> 看到这个问题，我们用 5 分钟想一下该怎么办。办法当然很多，这里使用桶排序的思想来处理。
>
> 我们找到 11 个桶，分别编号为 0-10，对应 0-10 分，如
>
> ![img](/images/buckets-1.jpg)
>
>
> 接着我们把这些分数按照桶的编号放入桶中，如
>
> ![img](/images/buckets-2.jpg)
>
>
> 接着我们从最大编号的桶到最小编号的桶依次输出每个桶中的分数，分别是 9、6、5、5、1 了。是不是很轻松地完成排序了呢？这就是桶排序的思想。

但桶排序的问题也较为明显：在资源分布不均匀时会占用大量的空间，他的空间复杂度是O(m)，其中m位桶的个数。

所以在此题中，我们能不能直接通过桶排序并计算最大的连续空桶数呢？代码如下：

```python
def maximumGap(self, nums) -> int:
    if len(nums)<2:
        return 0
    # 初始化桶，桶的个数为数组中最大值
    buckets = [0]*(max(nums)+1)
    # 把数字放入对应的桶中
    for num in nums:
        buckets[num]+=1
    gap:int = 0
    max_gap:int = 0
    init = False
    # 遍历桶并数出最大连续空桶的数量
    for bucket in buckets:
        gap+=1
        if bucket>0:
            if not init:
                init = True
            elif gap > max_gap:
                max_gap = gap
            gap = 0
    return max_gap
```

这个很莽的算法时间复杂度为O(m+n)，空间复杂度为O(m)。（你就说是不是线性复杂度吧）

提交后果不其然在`[2,9999999]`的用例上超时了。

要优化这个算法，我们就需要考虑在一个桶中放多个元素，可是如果只是单纯的扩大桶的容量，我们还是需要在每个桶中进行排序，无法降低时间复杂度到线性。看了看LeetCode的讨论贴后，发现引入鸽笼原则（或者叫抽屉原则）可以解决这个问题，这个原则很简单也很符合常识：

> 在N个鸽笼中放N-m个鸽子，则至少有m个鸽笼是空的。
>
> 在N个鸽笼中放大于N个鸽子，则至少有一个鸽笼中有多个鸽子。

具体到本例中，我们把N个数在桶排序时放入N+1个桶，则会产生至少1个空桶。而由于所有桶的大小是一致的，所以最大间隔一定是在一个或多个连续的空桶左右产生的。例如，我们在处理数组`[1,2,10,9]`时，我们需要将`[1,10]`这个区间均匀的分割为N+1，也就是5个区间，区间大小为`(10-1)//5+1=2`,如下：

```
[1,3) [3,5) [5,7) [7,9) [9,11)
```

这里为了包含最大值，需要为整除后的值+1。之后我们把这四个数放入如上五个桶中：

```
[[1,2],[],[],[],[9,10]
```

可以看到中间产生了3个空桶，我们只需计算每段空桶前后两个桶中前桶的最大值与后桶的最小值便可知道一个可能为最大间距的值。得到所有空桶左右的间距后取最大间距，就是我们最终需要的答案。

## 代码（Python为例）

```python
def maximumGap(self, nums) -> int:
    if len(nums)<2:
        return 0
    n:int = len(nums)
    max_num:int = max(nums)
    min_num:int = min(nums)
    if max_num==min_num:
        return 0
    # 计算每个桶的大小
    bucket_size:int = ((max_num-min_num)//(n+1))+1
    # 初始化桶列表
    buckets = [[] for row in range(n+1)]
    
    for num in nums:
        # 找到数字对应的桶
        index:int = (num-min_num)//bucket_size
        # 把数字放入桶中
        buckets[index].append(num)
    gap:int = 0
    max_gap:int = 0
    prev_bucket = buckets[0]
    gapped_with_empty_bucket = False 
    # 遍历桶列表判断空桶前后的间距
    for bucket in buckets:
        # 若当前桶为空桶
        if len(bucket) == 0:
            # 修改状态为中间有空桶
            gapped_with_empty_bucket = True
        # 若当前桶为非空桶
        else:
            # 若中间有空桶
            if gapped_with_empty_bucket:
                # 与上一个非空桶计算间距
                gap = min(bucket)-max(prev_bucket)
                # 更新最大间距
                max_gap = gap if gap>max_gap else max_gap
                # 修改状态为中间无空桶
                gapped_with_empty_bucket = False
            # 更新上一个非空桶为当前桶
            prev_bucket = bucket
    return max_gap
```

