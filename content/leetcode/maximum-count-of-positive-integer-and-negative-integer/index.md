---
title: "2529. 正整数和负整数的最大计数"
description: 
date: 2025-04-02T15:44:03+08:00
image: 2025-04-02-15-48-13.png
math: 
license: 
hidden: false
comments: true
tags: ["遍历","二分查找"]
categories: ["LeetCode"]
slug: maximum-count-of-positive-integer-and-negative-integer
---

## 题目描述
给你一个按 非递减顺序 排列的数组 nums ，返回正整数数目和负整数数目中的最大值。

换句话讲，如果 nums 中正整数的数目是 pos ，而负整数的数目是 neg ，返回 pos 和 neg二者中的最大值。
注意：0 既不是正整数也不是负整数。

## 解法

- 遍历：时间复杂度 O(n)，空间复杂度 O(1)
- 二分查找：时间复杂度 O(logn)，空间复杂度 O(1)

### 遍历

遍历数组，统计负数数目 neg 和正数数目 pos。最后返回 max(neg,pos)。

```java
public int maximumCount(int[] nums) {
        int neg = 0;
        int pos = 0;
        for (int x : nums) {
            if (x < 0) {
                neg++;
            } else if (x > 0) {
                pos++;
            }
        }
        return Math.max(neg, pos);
    }
```

### 二分查找

由于数组是有序的，我们可以二分找到第一个 ≥0 的数的下标 i，那么下标在 [0,i−1] 中的数都小于 0，这恰好有 i 个。

同样地，二分找到第一个 >0 的数的下标 j，那么下标在 [j,n−1] 中的数都大于 0，这有 n−j 个。

所以通过二分查找第一个 ≥0 和第一个 >0 的位置，就可以用 O(logn) 的时间解决本题，

```java
public int maximumCount(int[] nums) {
    int neg = lowerBound(nums, 0);    // 找第一个大于等于0的元素
    int pos = nums.length - lowerBound(nums, 1);
    return Math.max(neg, pos);
}

private int lowerBound(int[] nums, int target) {
    int left = -1, right = nums.length;
    while (left + 1 < right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) {
            right = mid;    // 边界左缩 找到第一个等于target的元素
        } else if (nums[mid] < target) {
            left = mid;
        } else if (nums[mid] > target) {
            right = mid;
        }
    }
    return right;
}
```