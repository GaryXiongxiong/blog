---
title: LeetCode笔记-用前缀和与哈希表解决【560-和为k的子数组】
tags:
- Algorithm
categories: 
- blog
date: 2020-10-22 16:59:32
---
给定一个整数数组和一个整数 **k，**你需要找到该数组中和为 **k** 的连续的子数组的个数。

- 数组的长度为 [1, 20,000]
- 数组中元素的范围是 [-1000, 1000] ，且整数 **k** 的范围是 [-1e7, 1e7]

<!--more-->

## 示例

```
输入:nums = [1,1,1], k = 2
输出: 2 , [1,1] 与 [1,1] 为两种不同的情况。
```

## 思路

看到这种子集合，我一开始想到了动态规划，题解如下：

我们做一张表对应数组`[a,b,c]`:

|      | a     | b    | c    |
| ---- | ----- | ---- | ---- |
| a    | a     | 0    | 0    |
| b    | a+b   | b    | 0    |
| c    | a+b+c | b+c  | c    |

列为起始元素，行为结束元素

可以推出我们每行的值等于上一行的值加上本行对应元素。每行只填到自身。对应代码如下：

```python
def subarraySum(self, nums: List[int], k: int) -> int:
    n:int = len(nums)
    count:int = 0;
    # 构建n*n的dp表格
    dp_tensor = [[[0 for row in range(n)] for col in range(n)]]
    for index in range(n):
        for j in range(index+1):
            # 计算并填写表格
            prev:int = dp_tensor[index-1][j] if index>0 else 0
            dp_tensor[index][j] = prev+nums[index]
            if dp_tensor[index][j] == k:
                count+=1
    return count
```

时间复杂度为`O(N^2)`感觉并没有发挥DP的优势，跟暴力穷举的时间复杂度类似了。提交后果然超时。

观看LeetCode相关讨论后学习到本题需要使用前缀和与哈希表解决：

## 题解

#### 使用前缀和

设我们有一个数组`A[]`，其中对应`n`位置的前缀和为`A[0]`累加至`A[n-1]`,依此我们可以构建前缀和数组`prev_sums[]`，当我们需要知道从`A[i]`到`A[j]`的子数组和时，我们只需计算`prev_sums[j+1]-prev_sums[i]`，对应的，我们修改代码至如下：

```python
def subarraySum(self, nums: List[int], k: int) -> int:
    n: int = len(nums)
    prev_sums: List[int] = [0]
    count: int = 0
    # 构建前缀和序列
    for i in range(1, n+1):
        prev_sums.append(prev_sums[i-1]+nums[i-1])
    # 双指针遍历前缀和序列寻找前缀和差为k的两个元素，并累加数量
    for i in range(n):
        for j in range(i+1):
            if k == prev_sums[i+1]-prev_sums[j]:
                count += 1
    return count
```

#### 使用哈希表（字典）

这时，时间复杂度依然为`O(N^2)`，为了进一步优化，需要想到，我们只需计算和为k的子数组数量，也就是`prev_sums[i+1]-prev_sums[j]=k`的数量，就能得出最后的答案。我们可以利用字典中查找`key`复杂度只为`O(1)`的特性，把每个`prev_sum`存储到一个字典的`key`中，同时把该`prev_sum`的个数存储到值中。同时在循环中我们只考虑`j<=i`的情况，所以我们只需要维护一个存储key为`prev_sum`，值为`prev_sum的个数`的字典，遍历一次，在遍历过程中，计算出`当前prev_sum-k`的值，并在字典中查找这个key，如果有对应的值则把值加到`count`上，最后得出答案。

## 最终代码

```python
def subarraySum(self, nums: List[int], k: int) -> int:
    n: int = len(nums)
    prev_sum = 0
    prev_sum_map = {0 : 1}
    count: int = 0
    for i in range(1, n+1):
        # 计算i-1元素的前缀和
        prev_sum+=nums[i-1]
        # 在字典中查找 prev_sum-k，如果有，则说明有以当前元素为结尾的和为k的子数组，有prev_sum_map[prev_sum-k]个
        if prev_sum-k in prev_sum_map:
            # 把以当前元素为结尾的和为k的子数组的数量加到map上
            count+=prev_sum_map[prev_sum-k]
        # 更新当前元素前缀和在字典中的个数
        prev_sum_map[prev_sum] = prev_sum_map.get(prev_sum,0)+1
    return count
```

此算法时间复杂度为`O(N)`。