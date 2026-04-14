---
title: 双指针
date: 2026-04-05
description: 双指针（Two Pointers）算法技巧详解：利用两个指针协同遍历解决数组、链表等问题
tags:
  - two-pointers
  - algorithm
draft: false
permalink:
---

## 定义

**双指针**是一种常用算法技巧，通过使用两个指针（通常初始化为不同位置）协同遍历数据结构，在 $O(n)$ 或 $O(n \log n)$ 时间内解决问题。[^1]

核心思想：通过控制指针的移动，避免不必要的遍历，将朴素 $O(n^2)$ 的双层循环优化为 $O(n)$。

## 基本类型

### 1. 对撞指针（左右指针）

两个指针分别从序列两端向中间移动，常用于有序数组查找。

**例：两数之和 II**

给定升序数组 $a$，找到两数之和等于目标值的下标。

```cpp
vector<int> twoSum(vector<int>& a, int target) {
    int l = 0, r = a.size() - 1;
    while (l < r) {
        int sum = a[l] + a[r];
        if (sum == target) return {l + 1, r + 1};
        if (sum < target) l++;
        else r--;
    }
    return {};
}
```

### 2. 滑动窗口（快慢指针）

维护一个窗口，通过移动左右边界解决问题。

**例：最大子序和（长度受限）**

给定长度为 $n$ 的数组，找到长度不超过 $k$ 的最大子序和。

```cpp
long long maxSubarraySum(vector<int>& a, int k) {
    int n = a.size();
    long long sum = 0, ans = LLONG_MIN;
    for (int l = 0, r = 0; r < n; r++) {
        sum += a[r];
        while (r - l + 1 > k) {
            sum -= a[l++];
        }
        ans = max(ans, sum);
    }
    return ans;
}
```

### 3. 分离链表

快指针每次走两步，慢指针每次走一步，用于检测环、找中点等。

**例：链表中点**

```cpp
ListNode* middleNode(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
    }
    return slow;
}
```

**例：环形链表检测**

```cpp
bool hasCycle(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true;
    }
    return false;
}
```

## 典型应用

### 有序数组去重

```cpp
int removeDuplicates(vector<int>& a) {
    int n = a.size();
    if (n <= 1) return n;
    int l = 0, r = 1;
    while (r < n) {
        if (a[l] != a[r]) {
            a[++l] = a[r];
        }
        r++;
    }
    return l + 1;
}
```

### 合并两个有序数组

```cpp
void merge(vector<int>& a, vector<int>& b) {
    int n = a.size(), m = b.size();
    vector<int> res;
    int i = 0, j = 0;
    while (i < n && j < m) {
        if (a[i] <= b[j]) res.push_back(a[i++]);
        else res.push_back(b[j++]);
    }
    while (i < n) res.push_back(a[i++]);
    while (j < m) res.push_back(b[j++]);
}
```

### 子序列判定

判断字符串 $t$ 是否为 $s$ 的子序列：

```cpp
bool isSubsequence(string s, string t) {
    int n = s.size(), m = t.size();
    int i = 0, j = 0;
    while (i < n && j < m) {
        if (s[i] == t[j]) {
            i++;
            j++;
        } else {
            i++;
        }
    }
    return j == m;
}
```

### 反转链表

```cpp
ListNode* reverseList(ListNode* head) {
    ListNode *pre = nullptr, *cur = head;
    while (cur) {
        ListNode* nxt = cur->next;
        cur->next = pre;
        pre = cur;
        cur = nxt;
    }
    return pre;
}
```

## 复杂度分析

| 类型 | 时间复杂度 | 空间复杂度 |
|------|------------|------------|
| 对撞指针 | $O(n)$ | $O(1)$ |
| 滑动窗口 | $O(n)$ | $O(1)$ 或 $O(k)$ |
| 快慢指针 | $O(n)$ | $O(1)$ |

## 技巧总结

1. **有序数组**：优先考虑对撞指针
2. **区间问题**：考虑滑动窗口
3. **链表问题**：快慢指针是常用技巧
4. **去重/合并**：利用指针跳过重复元素
5. **循环不变式**：始终保持对「窗口内/外」状态的清晰定义

## 与其他算法的结合

- **双指针 + 二分**：在有序数组中查找满足条件的区间
- **双指针 + 哈希**：处理两数之和、三数之和等问题
- **双指针 + 莫队**：莫队算法本质上是「指针移动优化」的高级应用

## 参考资料

[^1]: [双指针 - OI Wiki](https://oi-wiki.org/misc/two-pointer/)
