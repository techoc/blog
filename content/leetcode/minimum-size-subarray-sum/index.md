---
title: "209. 长度最小的子数组"
description: 
date: 2025-03-27T23:00:32+08:00
image: 2025-03-29-13-44-20.png
math: 
license: 
hidden: false
comments: true
tag: ["滑动窗口","二分查找"]
categories: ["LeetCode"]
---

## 题目描述
给定一个含有 n 个正整数的数组和一个正整数 target 。
找出该数组中满足其和 ≥ target 的长度最小的 连续子数组 [numsl, numsl+1, ..., numsr-1, numsr] ，并返回其长度。如果不存在符合条件的子数组，返回 0 。

> [209. 长度最小的子数组](https://leetcode-cn.com/problems/minimum-size-subarray-sum/)
> 
> [209. Minimum Size Subarray Sum](https://leetcode.com/problems/minimum-size-subarray-sum/)

## 解法

- 滑动窗口：时间复杂度 O(n)，空间复杂度 O(1)
- 二分查找：时间复杂度 O(nlogn)，空间复杂度 O(n)

### 滑动窗口

滑动窗口需要两个指针，一个指向窗口的左边界，一个指向窗口的右边界。窗口的左边界和右边界都是从 0 开始，然后右边界不断右移，直到找到满足条件的子数组，然后左边界右移，直到不满足条件。

```java
public int minSubArrayLen(int target, int[] nums) {
    int left = 0, right = 0;
    int sum = 0;
    int res = Integer.MAX_VALUE;
    while (right < nums.length) {
        int num = nums[right];
        right++;
        sum += num;

        // 收缩左边界
        while (sum >= target) {
            res = Math.min(res, right - left);  // 更新最小长度 要在左边界收缩之前更新最小长度
            int num2 = nums[left];  // 要移除的元素
            left++;  // 左边界右移
            sum -= num2;    // 减去要移除的元素
        }
    }
    return res == Integer.MAX_VALUE ? 0 : res;
}
```

### 二分查找

```java
public int minSubArrayLen(int target, int[] nums) {
    // Step 1: 初始化前缀和数组
    int n = nums.length;
    long[] prefixSum = new long[n + 1]; // 使用 long 防止溢出
    for (int i = 1; i <= n; i++) {
        prefixSum[i] = prefixSum[i - 1] + nums[i - 1];
    }

    // Step 2: 遍历每个位置 i，用二分查找寻找满足条件的最小 j
    int minLength = Integer.MAX_VALUE;
    for (int i = 1; i <= n; i++) {
        // 我们需要找到 prefixSum[j] <= prefixSum[i] - target 的最大 j
        long requiredSum = prefixSum[i] - target;
        int j = binarySearch(prefixSum, requiredSum);

        // 如果找到了有效的 j，则更新最小长度
        if (j >= 0) {
            minLength = Math.min(minLength, i - j);
        }
    }

    // Step 3: 返回结果
    return minLength == Integer.MAX_VALUE ? 0 : minLength;
}

// 辅助函数：二分查找，找到最后一个小于等于 target 的索引
private int binarySearch(long[] prefixSum, long target) {
    int left = 0, right = prefixSum.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (prefixSum[mid] <= target) {
            left = mid + 1; // 继续向右找更大的值
        } else {
            right = mid - 1; // 向左缩小范围
        }
    }
    return right; // right 是最后一个小于等于 target 的索引
}
```