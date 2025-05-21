---
title: "275. H 指数 II"
description: 
date: 2025-04-03T12:14:32+08:00
image: https://www4.bing.com//th?id=OHR.MountHamilton_ZH-CN4280549129_UHD.jpg
license: 
hidden: false
comments: true
tags: ["二分查找"]
categories: ["LeetCode"]
slug: "H Index Ii"
---

## 题目描述

给你一个整数数组 citations ，其中 citations[i] 表示研究者的第 i 篇论文被引用的次数，citations 已经按照 升序排列 。计算并返回该研究者的 h 指数。

h 指数的定义：h 代表“高引用次数”（high citations），一名科研人员的 h 指数是指他（她）的 （n 篇论文中）至少 有 h 篇论文分别被引用了至少 h 次。

请你设计并实现对数时间复杂度的算法解决此问题。

## 解法

- 二分查找

### 二分查找

需要注意的是，二分查找的对象是 hindex，而不是 citations。

如示例 1：
> 输入：citations = [0,1,3,5,6]
> 
> 输出：3
> 
> 解释：给定数组表示研究者总共有 5 篇论文，每篇论文相应的被引用了 0, 1, 3, 5, 6 次。
>     由于研究者有3篇论文每篇 至少 被引用了 3 次，其余两篇论文每篇被引用 不多于 3 次，所以她的 h 指数是 3 。

hindex = [0,1,2,3,4] ,hindex 才是二分查找的对象。

n = citations.length = 5

1. 假设当 hindex[i] = 0 时，至少有 0 篇论文引用次数 >= 0，此时 citations[n - i] = 6 >=0, 所以 hindex = 0 是一个有效的答案。
2. 假设当 hindex[i] = 1 时，至少有 1 篇论文引用次数 >= 1，此时 citations[n - i] = 5 >=1, 所以 hindex = 1 是一个有效的答案。
3. 假设当 hindex[i] = 2 时，至少有 2 篇论文引用次数 >= 2，此时 citations[n - i] = 5 >=2, 所以 hindex = 2 是一个有效的答案。
4. 假设当 hindex[i] = 3 时，至少有 3 篇论文引用次数 >= 3，此时 citations[n - i] = 3 >=3, 所以 hindex = 3 是一个有效的答案。
5. 假设当 hindex[i] = 4 时，至少有 4 篇论文引用次数 >= 4，此时 citations[n - i] = 1 < 4, 所以 hindex = 4 不是一个有效的答案。

二分查找的目标是找到最大的 hindex，使得至少有 hindex 篇论文引用次数 >= hindex。


```java
public int hIndex(int[] citations) {
    // 在区间 (left, right) 内询问
    int n = citations.length;
    int left = 0;
    int right = n + 1;
    while (left + 1 < right) { // 区间不为空
        // 循环不变量：
        // left 的回答一定为「是」
        // right 的回答一定为「否」
        int mid = left + (right - left) / 2;
        // 引用次数最多的 mid 篇论文，引用次数均 >= mid
        if (citations[n - mid] >= mid) {
            left = mid; // 询问范围缩小到 (mid, right)
        } else {
            right = mid; // 询问范围缩小到 (left, mid)
        }
    }
    // 根据循环不变量，left 现在是最大的回答为「是」的数
    return left;
}
```