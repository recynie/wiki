---
title: 高斯消元
date: 2026-04-05
description: 高斯消元法（Gaussian Elimination）详解：解线性方程组、求矩阵秩、计算行列式
tags:
  - gaussian-elimination
  - linear-algebra
  - math
draft: false
permalink:
---

## 定义

**高斯消元法**是线性代数中最基础的算法之一，用于求解线性方程组 $Ax = b$。[^1]

通过初等行变换，将增广矩阵化为行阶梯形，再通过回代求出解。

## 解线性方程组

### 原理

设有 $n$ 个方程、$n$ 个未知数的方程组：

$$
\begin{cases}
a_{11}x_1 + a_{12}x_2 + \cdots + a_{1n}x_n = b_1 \\
a_{21}x_1 + a_{22}x_2 + \cdots + a_{2n}x_n = b_2 \\
\cdots \\
a_{n1}x_1 + a_{n2}x_2 + \cdots + a_{nn}x_n = b_n
\end{cases}
$$

高斯消元分为**前向消元**和**回代**两步。

### 代码实现

```cpp
const double EPS = 1e-8;

// 高斯消元解 Ax = b
// 返回值：0 表示唯一解，1 表示无解，2 表示无穷多解
int gauss(vector<vector<double>>& a, vector<double>& ans) {
    int n = a.size();
    int m = a[0].size() - 1;  // 未知数个数
    
    int col = 0;  // 当前列
    for (int row = 0; row < n && col < m; row++, col++) {
        // 找到当前列绝对值最大的行
        int sel = row;
        for (int i = row; i < n; i++) {
            if (fabs(a[i][col]) > fabs(a[sel][col])) {
                sel = i;
            }
        }
        
        // 该列全为0，继续下一列
        if (fabs(a[sel][col]) < EPS) {
            row--;
            continue;
        }
        
        // 交换行
        swap(a[sel], a[row]);
        
        // 归一化当前行
        double div = a[row][col];
        for (int j = col; j <= m; j++) {
            a[row][j] /= div;
        }
        
        // 消元：将当前列的其他行变为0
        for (int i = 0; i < n; i++) {
            if (i != row) {
                double mult = a[i][col];
                for (int j = col; j <= m; j++) {
                    a[i][j] -= mult * a[row][j];
                }
            }
        }
    }
    
    // 检验：若有剩余方程系数全为0但b不为0，则无解
    for (int i = 0; i < n; i++) {
        bool allZero = true;
        for (int j = 0; j < m; j++) {
            if (fabs(a[i][j]) > EPS) {
                allZero = false;
                break;
            }
        }
        if (allZero && fabs(a[i][m]) > EPS) {
            return 1;  // 无解
        }
    }
    
    // 提取解
    for (int i = 0; i < m; i++) {
        ans[i] = 0;
    }
    for (int i = 0; i < n; i++) {
        int pivot = -1;
        for (int j = 0; j < m; j++) {
            if (fabs(a[i][j]) > EPS) {
                pivot = j;
                break;
            }
        }
        if (pivot != -1) {
            ans[pivot] = a[i][m];
        }
    }
    
    return 2;  // 唯一解或无穷多解
}
```

## 整数版本（模域）

```cpp
const long long MOD = 1000000007;

long long mod_pow(long long a, long long e) {
    long long r = 1;
    while (e) {
        if (e & 1) r = r * a % MOD;
        a = a * a % MOD;
        e >>= 1;
    }
    return r;
}

// 整数高斯消元（求逆矩阵或解方程）
int gauss_mod(vector<vector<long long>>& a, int n, int m) {
    int col = 0;
    for (int row = 0; row < n && col < m; row++, col++) {
        // 找主元
        int sel = row;
        for (int i = row; i < n; i++) {
            if (a[i][col] > a[sel][col]) {
                sel = i;
            }
        }
        
        if (a[sel][col] == 0) {
            row--;
            continue;
        }
        
        swap(a[sel], a[row]);
        
        // 归一化（乘以逆元）
        long long inv = mod_pow(a[row][col], MOD - 2);
        for (int j = col; j <= m; j++) {
            a[row][j] = a[row][j] * inv % MOD;
        }
        
        // 消元
        for (int i = 0; i < n; i++) {
            if (i != row && a[i][col]) {
                long long mult = a[i][col];
                for (int j = col; j <= m; j++) {
                    a[i][j] = (a[i][j] - mult * a[row][j]) % MOD;
                    if (a[i][j] < 0) a[i][j] += MOD;
                }
            }
        }
    }
    return 0;
}
```

## 求矩阵的秩

```cpp
int rank(vector<vector<double>>& a) {
    int n = a.size(), m = a[0].size();
    int r = 0;
    for (int col = 0; col < m && r < n; col++) {
        int sel = r;
        for (int i = r; i < n; i++) {
            if (fabs(a[i][col]) > fabs(a[sel][col])) {
                sel = i;
            }
        }
        if (fabs(a[sel][col]) < EPS) continue;
        swap(a[sel], a[r]);
        for (int i = 0; i < n; i++) {
            if (i != r && fabs(a[i][col]) > EPS) {
                double div = a[i][col] / a[r][col];
                for (int j = col; j < m; j++) {
                    a[i][j] -= div * a[r][j];
                }
            }
        }
        r++;
    }
    return r;
}
```

## 求行列式

```cpp
double det(vector<vector<double>> a) {
    int n = a.size();
    double ans = 1;
    for (int i = 0; i < n; i++) {
        int sel = i;
        for (int j = i + 1; j < n; j++) {
            if (fabs(a[j][i]) > fabs(a[sel][i])) {
                sel = j;
            }
        }
        if (fabs(a[sel][i]) < EPS) return 0;
        if (sel != i) {
            swap(a[sel], a[i]);
            ans = -ans;
        }
        ans *= a[i][i];
        for (int j = i + 1; j < n; j++) {
            a[i][j] /= a[i][i];
        }
        for (int j = i + 1; j < n; j++) {
            for (int k = i + 1; k < n; k++) {
                a[j][k] -= a[j][i] * a[i][k];
            }
        }
    }
    return ans;
}
```

## 应用场景

| 应用 | 说明 |
|------|------|
| 线性方程组求解 | $Ax = b$ |
| 矩阵求逆 | $[A|I] \to [I|A^{-1}]$ |
| 矩阵秩的计算 | 行阶梯形非零行数 |
| 行列式计算 | 上三角化 |
| 线性规划 | 单纯形法的核心 |
| 最小二乘法 | 拟合曲线 |

## 复杂度分析

| 操作 | 时间复杂度 | 空间复杂度 |
|------|------------|------------|
| 高斯消元 | $O(n^3)$ | $O(n^2)$ |

## 参考资料

[^1]: [高斯消元 - OI Wiki](https://oi-wiki.org/math/numerical/gauss/)
