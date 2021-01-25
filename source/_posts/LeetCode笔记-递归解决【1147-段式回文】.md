---
title: LeetCode笔记-递归解决【1147-段式回文】
tags:
- Algorithm
categories: 
- blog
date: 2020-10-26 09:56:12
---
段式回文 其实与 一般回文 类似，只不过是最小的单位是 一段字符 而不是 单个字母。

举个例子，对于一般回文 "`abcba`" 是回文，而 "`volvo`" 不是，但如果我们把 "`volvo`" 分为 "`vo`"、"`l`"、"`vo`" 三段，则可以认为 “`(vo)(l)(vo)`” 是段式回文（分为 3 段）。

<!--more-->

给你一个字符串 `text`，在确保它满足段式回文的前提下，请你返回 **段** 的 **最大数量** `k`。

如果段的最大数量为 `k`，那么存在满足以下条件的 `a_1, a_2, ..., a_k`：

- 每个 `a_i` 都是一个非空字符串；
- 将这些字符串首位相连的结果 `a_1 + a_2 + ... + a_k` 和原始字符串 `text` 相同；
- 对于所有`1 <= i <= k`，都有 `a_i = a_{k+1 - i}`。

## 示例

**示例 1：**

```
输入：text = "ghiabcdefhelloadamhelloabcdefghi"
输出：7
解释：我们可以把字符串拆分成 "(ghi)(abcdef)(hello)(adam)(hello)(abcdef)(ghi)"。
```

**示例 2：**

```
输入：text = "merchant"
输出：1
解释：我们可以把字符串拆分成 "(merchant)"。
```

**示例 3：**

```
输入：text = "antaprezatepzapreanta"
输出：11
解释：我们可以把字符串拆分成 "(a)(nt)(a)(pre)(za)(tpe)(za)(pre)(a)(nt)(a)"。
```

**示例 4：**

```
输入：text = "aaa"
输出：3
解释：我们可以把字符串拆分成 "(a)(a)(a)"。
```

 ## 思路

第一反应是通过双指针，从字符串两端逐渐向中间推进寻找相同字符串，发现后即可取出一个回文段：

```
	[a,b,c,d,a,b,c]
i=0  ^           ^
i=1  ^-^       ^-^
i=2  ^-^-^   ^-^-^ 发现回文段（a,b,c）
```

取出一个回文段后，我们可以继续推进，寻找下一个回文段，直到两个指针间距<=1，如果这时中间部分未找到回文段，则把中间部分整个提取出作为一个回文段：

```
	[a,b,c,d,e,a,b,c]
i=2  ^-^-^     ^-^-^ 
这时，中间的(d,e)为一个回文段，k=3
```

为简化算法，我们可以使用递归，即找到一对回文段后，可去掉找到的回文段，并再次调用该函数本身，`k=k(去除已找到的前后两个回文段)+2`。递归基线条件有如下3个：

1. 输入字符串长度=0，则k=0
2. 输入字符串长度=1，则k=1
3. 若字符串中找不到回文段，则k=1

至此，我们已经可以写出代码：

```python
def longestDecomposition(self, text: str) -> int:
    n:int = len(text)
    if(n==0):
        return 0
    elif(n==1):
        return 1
    mid:int = math.floor(n/2.0)
    for i in range(mid):
        if text[0:i+1]==text[n-i-1:n]:
            return self.longestDecomposition(text[i+1:n-i-1])+2
    return 1
```

提交结果如下：

> 执行用时: **52 ms**
>
> 内存消耗: **13.8 MB**

用时仅击败15%，看来还有优化空间，经过考虑后，我们仅需要在当前字符`text[i]`与最后一个字符相等时，才需要对比整个字符串，所以可以对循环中的判断条件做出以下调整来节省时间：

```python
if text[i] == text[n-1] and text[0:i+1]==text[n-i-1:n]:
```

修改后结果如下：

> 执行用时: **36 ms**
>
> 内存消耗: **13.7 MB**

用时击败89%提交记录。

## 代码

```python
def longestDecomposition(self, text: str) -> int:
    n:int = len(text)
    if(n==0):
        return 0
    elif(n==1):
        return 1
    mid:int = math.floor(n/2.0)
    for i in range(mid):
        if text[i] == text[n-1] and text[0:i+1]==text[n-i-1:n]:
            return self.longestDecomposition(text[i+1:n-i-1])+2
    return 1
```

