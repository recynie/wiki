---
title: 前缀和
date: 2026-03-31
description: 前缀和（Prefix Sum）与差分
tags:
  - prefix-sum
  - algorithm
draft: false
permalink:
---

## 定义

前缀和（Prefix Sum）是一种重要的预处理技术，用于快速计算数组某个区间内元素的和。给定数组 $A[0..n-1]$，前缀和数组 $P$ 定义为：

$$
P[i] = \sum_{k=0}^{i} A[k] \quad (0 \leq i < n)
$$

即 $P[i]$ 表示原数组 $A$ 从下标 0 到下标 $i$（含）的所有元素之和。

有了前缀和数组，任意区间 $[l, r]$（含）的和可以 $O(1)$ 计算：

$$
\sum_{k=l}^{r} A[k] = P[r] - P[l-1] \quad (l > 0)
$$

如果 $l = 0$，则 $\sum_{k=0}^{r} A[k] = P[r]$。[^1]

## 一维前缀和

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

// 构建前缀和数组
vector<long long> buildPrefixSum(const vector<int>& arr) {
    int n = arr.size();
    vector<long long> prefix(n);
    prefix[0] = arr[0];
    for (int i = 1; i < n; i++) {
        prefix[i] = prefix[i - 1] + arr[i];
    }
    return prefix;
}

// 查询区间 [l, r] 的和（包含端点）
long long rangeSum(const vector<long long>& prefix, int l, int r) {
    if (l == 0) return prefix[r];
    return prefix[r] - prefix[l - 1];
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<int> arr = {1, 3, 5, 7, 9, 11};
    auto prefix = buildPrefixSum(arr);
    
    cout << rangeSum(prefix, 1, 3) << endl;  // 3 + 5 + 7 = 15
    cout << rangeSum(prefix, 0, 5) << endl;  // 1 + 3 + 5 + 7 + 9 + 11 = 36
    cout << rangeSum(prefix, 2, 2) << endl;  // 5
    
    return 0;
}
```

### 模板

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n;
    cin >> n;
    vector<long long> a(n), prefix(n);
    
    for (int i = 0; i < n; i++) {
        cin >> a[i];
        prefix[i] = a[i] + (i ? prefix[i-1] : 0);
    }
    
    // 查询区间 [l, r]
    int l, r;
    cin >> l >> r;
    long long ans = prefix[r] - (l ? prefix[l-1] : 0);
    cout << ans << endl;
    
    return 0;
}
```

## 二维前缀和

二维前缀和用于快速计算矩阵中某个子矩阵的元素之和。给定矩阵 $A[m][n]$，二维前缀和 $P[i][j]$ 表示以 $(0, 0)$ 为左上角、$(i, j)$ 为右下角的子矩阵元素之和：

$$
P[i][j] = \sum_{x=0}^{i} \sum_{y=0}^{j} A[x][y]
$$

利用二维前缀和，可以 $O(1)$ 计算任意子矩阵 $(x_1, y_1)$ 到 $(x_2, y_2)$ 的和：

$$
\sum_{x=x_1}^{x_2} \sum_{y=y_1}^{y_2} A[x][y] = P[x_2][y_2] - P[x_1-1][y_2] - P[x_2][y_1-1] + P[x_1-1][y_1-1]
$$

### 代码实现

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int m = 4, n = 5;
    vector<vector<int>> a(m, vector<int>(n));
    vector<vector<long long>> prefix(m, vector<long long>(n));
    
    // 读取矩阵（示例）
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            a[i][j] = i * n + j + 1;  // 示例值
        }
    }
    
    // 构建二维前缀和
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            prefix[i][j] = a[i][j];
            if (i > 0) prefix[i][j] += prefix[i-1][j];
            if (j > 0) prefix[i][j] += prefix[i][j-1];
            if (i > 0 && j > 0) prefix[i][j] -= prefix[i-1][j-1];
        }
    }
    
    // 查询子矩阵和
    auto query = [&](int x1, int y1, int x2, int y2) -> long long {
        long long res = prefix[x2][y2];
        if (x1 > 0) res -= prefix[x1-1][y2];
        if (y1 > 0) res -= prefix[x2][y1-1];
        if (x1 > 0 && y1 > 0) res += prefix[x1-1][y1-1];
        return res;
    };
    
    cout << query(1, 1, 2, 3) << endl;  // 示例
    
    return 0;
}
```

## 差分数组

差分是前缀和的逆运算。当我们需要对数组的某个区间进行**批量加/减操作**时，差分数组可以让我们 $O(1)$ 完成更新，最后再用前缀和还原。

给定原数组 $A$，差分数组 $D$ 定义为：

$$
D[i] = A[i] - A[i-1] \quad (A[-1] = 0)
$$

对区间 $[l, r]$ 所有元素加 $v$：

```cpp
// 差分数组的更新：O(1)
void rangeAdd(vector<long long>& diff, int l, int r, int v) {
    diff[l] += v;
    if (r + 1 < diff.size()) {
        diff[r + 1] -= v;
    }
}

// 还原前缀和得到原数组
void restore(vector<long long>& diff) {
    for (int i = 1; i < diff.size(); i++) {
        diff[i] += diff[i - 1];
    }
}
```

### 完整示例

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n = 10;
    vector<long long> diff(n + 1, 0);  // 差分数组，多开一个位置
    
    // 模拟对数组执行以下操作：
    // [1, 4] 范围加 3
    // [3, 7] 范围加 2
    // [5, 9] 范围减 1
    
    auto rangeAdd = [&](int l, int r, int v) {
        diff[l] += v;
        diff[r + 1] -= v;
    };
    
    rangeAdd(1, 4, 3);  // 位置 1-4 加 3
    rangeAdd(3, 7, 2);  // 位置 3-7 加 2
    rangeAdd(5, 9, -1); // 位置 5-9 减 1
    
    // 还原原数组
    vector<long long> arr(n);
    arr[0] = diff[0];
    for (int i = 1; i < n; i++) {
        arr[i] = arr[i-1] + diff[i];
    }
    
    for (int i = 0; i < n; i++) {
        cout << arr[i] << " ";
    }
    cout << endl;
    
    return 0;
}
```

### 二维差分

二维差分用于对子矩阵进行批量加法操作：

```cpp
void matrixAdd(vector<vector<long long>>& diff, 
                int x1, int y1, int x2, int y2, long long v) {
    diff[x1][y1] += v;
    diff[x1][y2 + 1] -= v;
    diff[x2 + 1][y1] -= v;
    diff[x2 + 1][y2 + 1] += v;
}
```

## 典型应用场景

### 子数组和为 K 的个数

```cpp
#include <bits/stdc++.h>
using namespace std;

int subarraySum(vector<int>& nums, int k) {
    unordered_map<long long, int> mp;
    mp[0] = 1;
    
    long long prefix = 0;
    int count = 0;
    
    for (int num : nums) {
        prefix += num;
        if (mp.find(prefix - k) != mp.end()) {
            count += mp[prefix - k];
        }
        mp[prefix]++;
    }
    
    return count;
}
```

### 连续子数组最大和（Kadane 算法）

```cpp
#include <bits/stdc++.h>
using namespace std;

long long maxSubArray(vector<int>& nums) {
    long long maxSum = nums[0];
    long long curSum = nums[0];
    
    for (int i = 1; i < nums.size(); i++) {
        curSum = max((long long)nums[i], curSum + nums[i]);
        maxSum = max(maxSum, curSum);
    }
    
    return maxSum;
}
```

### 矩阵区域和检索

```cpp
class NumMatrix {
private:
    vector<vector<int>> prefix;
public:
    NumMatrix(vector<vector<int>>& matrix) {
        int m = matrix.size(), n = matrix[0].size();
        prefix.resize(m + 1, vector<int>(n + 1, 0));
        
        for (int i = 1; i <= m; i++) {
            for (int j = 1; j <= n; j++) {
                prefix[i][j] = matrix[i-1][j-1] + prefix[i-1][j] + prefix[i][j-1] - prefix[i-1][j-1];
            }
        }
    }
    
    int sumRegion(int row1, int col1, int row2, int col2) {
        return prefix[row2+1][col2+1] - prefix[row1][col2+1] - prefix[row2+1][col1] + prefix[row1][col1];
    }
};
```

## 参考资料

[^1]: 前缀和是一种空间换时间的策略，通过 $O(n)$ 的预处理实现 $O(1)$ 的区间查询。

[^2]: 差分数组适用于需要频繁对区间进行加减操作的场景，如航班票价更新、多次区间修改后查询等。
