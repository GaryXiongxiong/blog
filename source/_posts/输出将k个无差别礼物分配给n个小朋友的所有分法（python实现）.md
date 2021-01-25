---
title: LeetCode笔记 输出将k个无差别礼物分配给n个小朋友的所有分法（python实现）
date: 2020-08-01 08:38:56
tags:
- Algorithm
categories: 
- blog
---

我的算法思路为遍历所有可能性，感觉有点蠢

<!--more-->
```python
import sys
#为小朋友分配礼物
def distribute(giftNum,childNum):
        children = [[[0]*childNum]]
        # children[n]为小朋友们分n个礼物的所有可能分法
        # 如childNum为2时，children[1]=[[1,0],[0,1]]
        # 上一行代码初始化了小朋友们分0个礼物是的情况
        for times in range(1,giftNum+1):
                children.append([])
                # 增加一行用以记录增加一个礼物时的情况
                for prob in range(len(children[times-1])):
                # 遍历少一个礼物时的每一种情况
                # 对于每一种情况，增加一个礼物时会可能出现childNum种新情况
                        for tarChild in range(childNum):
                        # 遍历增加一个礼物后的每一种情况,即该礼物分到各个小朋友手中时产生的情况
                                child = children[times-1][prob].copy()
                                child[tarChild] += 1
                                # 优化速度注释掉如下{
                                # same = False
                                # for i in children[times]:
                                #         if child == i:
                                #                 same = True
                                # }
                                if children[times].count(child) == 0:
                                # 判断该情况是否与之前情况有重合
                                        children[times].append(child)
        return children[giftNum]

if __name__ == "__main__":
    inputLine = sys.stdin.readline().strip().split(" ")
    inputLine = [int(str) for str in inputLine]
    n = inputLine[0]
    k = inputLine[1]
    distribution = distribute(n,k)
    print(len(distribution))
    for prob in distribution:
                count = 0
                for i in prob:
                        print("*"*i,end='')
                        count = count+1
                        if count < k:
                                print("|",end='')
                print()
        
```
