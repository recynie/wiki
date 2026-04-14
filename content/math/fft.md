---
title: 快速傅里叶变换
date: 2026-04-05
description: 快速傅里叶变换（FFT）用于多项式乘法的O(n log n)算法
tags:
  - fft
  - polynomial
  - fft
  - number-theory
draft: false
permalink:
---

## 定义

快速傅里叶变换（Fast Fourier Transform，FFT）是一种高效计算[[../math/number-theory|离散傅里叶变换]]（DFT）的算法。

### 基本思想

给定两个多项式：

$$
A(x) = a_0 + a_1x + a_2x^2 + \cdots + a_{n-1}x^{n-1}
$$

$$
B(x) = b_0 + b_1x + b_2x^2 + \cdots + b_{n-1}x^{n-1}
$$

朴素多项式乘法需要 $O(n^2)$ 的时间复杂度，而 FFT 可以将时间复杂度降低到 $O(n \log n)$。[^1]

### 应用场景

- **多项式乘法**：在竞赛中处理多项式乘积
- **大整数乘法**：将大整数视为多项式系数进行乘法
- **卷积运算**：信号处理中的卷积

---

## 核心原理

### n 次单位根

n 次单位根是满足 $w^n = 1$ 的复数 $w$。主 n 次单位根定义为：

$$
\omega_n = e^{2\pi i / n} = \cos\frac{2\pi}{n} + i\sin\frac{2\pi}{n}
$$

**重要性质**：

1. **消去引理**：$\omega_{dn}^{dk} = \omega_n^k$
2. **折半引理**：$\omega_n^{k+n/2} = -\omega_n^k$
3. **求和引理**：$\sum_{j=0}^{n-1} \omega_n^{kj} = 0$（当 $k$ 不能被 $n$ 整除时）

### 离散傅里叶变换（DFT）

DFT 将多项式在 $n$ 个 n 次单位根处求值：

$$
Y_k = \sum_{j=0}^{n-1} \omega_n^{kj} \cdot y_j, \quad k = 0, 1, \ldots, n-1
$$

### 逆变换（IDFT）

通过在单位根处再次求值并进行归一化，可以恢复原系数：

$$
y_j = \frac{1}{n} \sum_{k=0}^{n-1} \omega_n^{-kj} \cdot Y_k
$$

### 蝶形运算

FFT 的核心是蝶形运算（Butterfly Operation），通过将问题递归地分解为两个规模为 $n/2$ 的子问题：

$$
\begin{aligned}
y_{even} &= x_0 + \omega_n^k \cdot x_1 \\
y_{odd} &= x_0 - \omega_n^k \cdot x_1
\end{aligned}
$$

其中 $\omega_n^k$ 是旋转因子。

---

## 代码实现

### 递归版本

```cpp
#include <bits/stdc++.h>
using namespace std;
using cd = complex<double>;
const double PI = acos(-1);

void fft(vector<cd> & a, bool invert) {
    int n = a.size();
    if (n == 1) return;
    
    vector<cd> a0(n/2), a1(n/2);
    for (int i = 0, j = 0; i < n; i += 2, ++j) {
        a0[j] = a[i];
        a1[j] = a[i+1];
    }
    
    fft(a0, invert);
    fft(a1, invert);
    
    double ang = 2 * PI / n * (invert ? -1 : 1);
    cd w(1), wn(cos(ang), sin(ang));
    
    for (int i = 0; i < n/2; ++i) {
        a[i] = a0[i] + w * a1[i];
        a[i + n/2] = a0[i] - w * a1[i];
        w *= wn;
    }
}
```

### 迭代版本（位逆序）

```cpp
void fft_iterative(vector<cd> & a) {
    int n = a.size();
    
    // 位逆序置换
    for (int i = 1, j = 0; i < n; ++i) {
        int bit = n >> 1;
        for (; j & bit; bit >>= 1) j ^= bit;
        j ^= bit;
        if (i < j) swap(a[i], a[j]);
    }
    
    for (int len = 2; len <= n; len <<= 1) {
        double ang = 2 * PI / len * (invert ? -1 : 1);
        cd wlen(cos(ang), sin(ang));
        for (int i = 0; i < n; i += len) {
            cd w(1);
            for (int j = 0; j < len/2; ++j) {
                cd u = a[i+j], v = a[i+j+len/2] * w;
                a[i+j] = u + v;
                a[i+j+len/2] = u - v;
                w *= wlen;
            }
        }
    }
    
    if (invert) {
        for (cd & x : a) x /= n;
    }
}
```

### 多项式乘法函数

```cpp
vector<long long> multiply(const vector<long long> & a, const vector<long long> & b) {
    vector<cd> fa(a.begin(), a.end()), fb(b.begin(), b.end());
    int n = 1;
    while (n < (int)a.size() + (int)b.size()) n <<= 1;
    fa.resize(n);
    fb.resize(n);
    
    fft(fa, false);
    fft(fb, false);
    for (int i = 0; i < n; ++i) fa[i] *= fb[i];
    fft(fa, true);
    
    vector<long long> result(n);
    for (int i = 0; i < n; ++i) result[i] = long long(round(fa[i].real()));
    return result;
}
```

---

## 数论变换（NTT）

在模运算下，可以使用数论变换（Number Theoretic Transform）替代 FFT。

### 原理

NTT 使用原根（Primitive Root）代替复数单位根。对于素数 $p = q \cdot n + 1$ 和原根 $g$，则：

$$
\omega_n = g^q \pmod{p}
$$

### 常用模数

| 模数 | 原根 |
|------|------|
| 7340033 | 5 |
| 998244353 | 3 |
| 1004535809 | 479 |

### NTT 代码实现

```cpp
const int MOD = 998244353;
const int G = 3;

int mod_pow(int a, int e) {
    int r = 1;
    while (e) {
        if (e & 1) r = 1LL * r * a % MOD;
        a = 1LL * a * a % MOD;
        e >>= 1;
    }
    return r;
}

void ntt(vector<int> & a, bool invert) {
    int n = a.size();
    for (int i = 1, j = 0; i < n; ++i) {
        int bit = n >> 1;
        for (; j & bit; bit >>= 1) j ^= bit;
        j ^= bit;
        if (i < j) swap(a[i], a[j]);
    }
    
    for (int len = 2; len <= n; len <<= 1) {
        int wlen = mod_pow(G, (MOD-1)/len);
        if (invert) wlen = mod_pow(wlen, MOD-2);
        for (int i = 0; i < n; i += len) {
            int w = 1;
            for (int j = 0; j < len/2; ++j) {
                int u = a[i+j], v = 1LL * a[i+j+len/2] * w % MOD;
                a[i+j] = u + v;
                if (a[i+j] >= MOD) a[i+j] -= MOD;
                a[i+j+len/2] = u - v;
                if (a[i+j+len/2] < 0) a[i+j+len/2] += MOD;
                w = 1LL * w * wlen % MOD;
            }
        }
    }
    
    if (invert) {
        int n_inv = mod_pow(n, MOD-2);
        for (int & x : a) x = 1LL * x * n_inv % MOD;
    }
}
```

---

## 典型应用

### 1. 多项式乘法

给定两个多项式，求其乘积的系数。这是 FFT 最基本的应用。

### 2. 大整数乘法

将大整数的每一位视为多项式系数，即可使用 FFT 加速乘法。[^2]

### 3. 字符串匹配

将字符串匹配问题转化为多项式卷积问题。

### 4. 卷积运算

在[[../math/combinatorics|组合数学]]中，卷积用于计算排列组合问题。

---

## 例题

### 多项式乘法

**题目**：给定两个多项式 $A(x)$ 和 $B(x)$，次数均不超过 1000，求其乘积。

**思路**：

1. 将系数向量扩展到 2 的幂次长度
2. 使用 FFT 分别对两个多项式进行 DFT
3. 对应位置相乘
4. 使用逆 FFT 恢复结果

**代码**：

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n, m;
    cin >> n >> m;
    vector<long long> a(n+1), b(m+1);
    for (int i = 0; i <= n; ++i) cin >> a[i];
    for (int i = 0; i <= m; ++i) cin >> b[i];
    
    vector<long long> res = multiply(a, b);
    
    for (int i = 0; i <= n + m; ++i) {
        cout << res[i] << ' ';
    }
    return 0;
}
```

---

## 参考资料

[^1]: 本内容参考 [CP-Algorithms: FFT](https://cp-algorithms.com/algebra/fft.html)
[^2]: 大整数乘法的具体实现可参考任意一本竞赛教材
