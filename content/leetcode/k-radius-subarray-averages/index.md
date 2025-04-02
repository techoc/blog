---
title: "2090. 半径为 k 的子数组平均值"
description: 
date: 2025-03-29T21:28:43+08:00
image: 2025-03-29-21-34-15.png
math: 
license: 
hidden: false
comments: true
tags: ["滑动窗口"]
categories: ["LeetCode"]
slug: k-radius-subarray-averages
---

## 题目描述

给你一个下标从 0 开始的数组 nums ，数组中有 n 个整数，另给你一个整数 k 。

半径为 k 的子数组平均值 是指：nums 中一个以下标 i 为 中心 且 半径 为 k 的子数组中所有元素的平均值，即下标在 i - k 和 i + k 范围（含 i - k 和 i + k）内所有元素的平均值。如果在下标 i 前或后不足 k 个元素，那么 半径为 k 的子数组平均值 是 -1 。

构建并返回一个长度为 n 的数组 avgs ，其中 avgs[i] 是以下标 i 为中心的子数组的 半径为 k 的子数组平均值 。

x 个元素的 平均值 是 x 个元素相加之和除以 x ，此时使用截断式 整数除法 ，即需要去掉结果的小数部分。

例如，四个元素 2、3、1 和 5 的平均值是 (2 + 3 + 1 + 5) / 4 = 11 / 4 = 2.75，截断后得到 2 。

> [2090. 半径为 k 的子数组平均值](https://leetcode.cn/problems/k-radius-subarray-averages/)
>
>[2090. K Radius Subarray Averages](https://leetcode.com/problems/k-radius-subarray-averages/)

## 解法
- 滑动窗口：时间复杂度 O(n)，空间复杂度 O(1)

### 滑动窗口

需要注意的是，在计算平均值时，需要使用 long 类型，否则会溢出。
```java
public int[] getAverages(int[] nums, int k) {
    int left = 0, right = 0;
    long sum = 0L;  // 子数组和  可能超出 int 范围 所以用 long 类型
    int[] result = new int[nums.length];
    Arrays.fill(result, -1);
    while (right < nums.length) {
        sum += nums[right];
        right++;    // 右指针右移
        while (right - left >= 2 * k + 1) {  // 子数组长度为 2k+1 即右指针 - 左指针 = 2k 进入子数组计算
            result[right - k - 1] = (int) (sum / (2L * k + 1));  // 计算子数组最大平均数 即子数组和 / 子数组长度
            sum -= nums[left];
            left++;    // 左指针右移
        }
    }
    return result;
}
```