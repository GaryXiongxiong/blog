---
title: 递归算法简单实现列表全排列（Python为例）
date: 2020-06-01 08:24:34
tags:
- Algorithm
categories: 
- blog
---

## 算法说明

#### 目的：

输出给定列表array的全部排列

<!--more-->
#### 算法步骤：

1. 利用递归思想将n阶问题化简为n-1阶问题
2. 面对列表[1,2,3]，我们通常写出全排列的方法为：写出以1为开头的排列[1,2,3],[1,3,2]，再写出以2为开头的排列[2,1,3],[2,3,1]依次类推
3. 故我们在面对3元素的列表时，我们是轮流取出元素作为头，之后将其余两个元素全排列后放在第一个元素后
4. 所以我们可以知道全排列[x1,x2,x3,...,xn]时，我们会以x1作为头部，在其后加上其余元素即[x2,x3,...,xn]的全排列；再以x2作为头部，在其后加上[x1,x3,...,xn]的全排列；直到以xn作为头部，在其后加上[x1,x2,...,xn-1]的全排列
5. 同时设定基线条件：当列表仅有一个元素时，仅有一种排列

#### 算法实现

```python
def arrange(array):#全排列数组-通过递归
    output = []
    if len(array) == 1:
        return [array]#当数组只有一个元素时直接返回该数组
    else:#对于多个元素的数组，全排列相当于以不同的元素首，并将其余元素分全排列
        # 例如全排列[1,2,3]，相当于以1为头部排列[2,3]，以2为头部排列[1,3]，以3为头部排列[1,2]
        # 所以全排列为[1]+arrange([2,3]),[2]+arrange([1,3]),[3]+arrange([1,2])
        # 推广至n个元素的数列
        # arrange([x1,x2,...,xn])为[x1]+arrange([x2,x3,...,xn]),[x2]+arrange([x1,x3,...,xn]),...,[xn]+arrange([x1,x2,...,xn-1])
        for i in array:
            subArr = array.copy()
            subArr.remove(i)
            for j in arrange(subArr):
                newArr = [i]+j
                output.append(newArr)

    return output


```

