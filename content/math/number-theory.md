---
title: 数论基础
date: 2026-04-03
description: 数论基础：最大公约数、扩展欧几里得、快速幂、模运算
tags:
  - number-theory
  - gcd
  - modular-arithmetic
draft: false
permalink:
---

## 概述

数论是研究整数性质的学科，在算法竞赛中有着广泛的应用。本节介绍竞赛中最常用的数论基础知识。

## 最大公约数（GCD）

### 欧几里得算法

欧几里得算法（辗转相除法）可以在 $O(\log \min(a, b))$ 时间内求两个整数的最大公约数。

```cpp
long long gcd(long long a, long long b) {
    return b == 0 ? a : gcd(b, a % b);
}
```

### 性质

- $\gcd(a, b) = \gcd(b, a \bmod b)$
- $\gcd(a, 0) = |a|$
- $\gcd(a, b) = \gcd(a, b - a)$（更相减损术）

### 最小公倍数（LCM）

$$
\operatorname{lcm}(a, b) = \frac{a \times b}{\gcd(a, b)}
$$

```cpp
long long lcm(long long a, long long b) {
    return a / gcd(a, b) * b;  // 先除后乘避免溢出
}
```

## 扩展欧几里得算法

扩展欧几里得算法不仅求 $\gcd(a, b)$，还能求出贝祖系数 $x, y$，使得：

$$
ax + by = \gcd(a, b)
$$

### 代码实现

```cpp
// 返回 gcd(a, b)，同时求出 x, y 满足 ax + by = gcd(a, b)
long long exgcd(long long a, long long b, long long& x, long long& y) {
    if (b == 0) {
        x = 1;
        y = 0;
        return a;
    }
    long long x1, y1;
    long long g = exgcd(b, a % b, x1, y1);
    x = y1;
    y = x1 - (a / b) * y1;
    return g;
}
```

### 迭代版本

```cpp
long long exgcd(long long a, long long b, long long& x, long long& y) {
    long long x0 = 1, y0 = 0;
    long long x1 = 0, y1 = 1;
    
    while (b != 0) {
        long long q = a / b;
        tie(x0, x1) = make_tuple(x1, x0 - q * x1);
        tie(y0, y1) = make_tuple(y1, y0 - q * y1);
        tie(a, b) = make_tuple(b, a - q * b);
    }
    x = x0;
    y = y0;
    return a;
}
```

### 应用

- **求模逆元**：当 $\gcd(a, m) = 1$ 时，$a^{-1} \equiv x \pmod{m}$
- **解线性同余方程**：$ax \equiv b \pmod{m}$

## 快速幂

快速幂（Binary Exponentiation）可以在 $O(\log k)$ 时间内计算 $a^k$。

### 代码实现

```cpp
long long power(long long a, long long k) {
    long long res = 1;
    while (k > 0) {
        if (k & 1) res = res * a;
        a = a * a;
        k >>= 1;
    }
    return res;
}

// 模幂版本
long long mod_pow(long long a, long long k, long long mod) {
    long long res = 1;
    a %= mod;
    while (k > 0) {
        if (k & 1) res = res * a % mod;
        a = a * a % mod;
        k >>= 1;
    }
    return res;
}
```

## 模运算基础

### 基本性质

| 性质 | 表达式 |
|------|--------|
| 加法 | $(a + b) \bmod m = ((a \bmod m) + (b \bmod m)) \bmod m$ |
| 减法 | $(a - b) \bmod m = ((a \bmod m) - (b \bmod m) + m) \bmod m$ |
| 乘法 | $(a \times b) \bmod m = ((a \bmod m) \times (b \bmod m)) \bmod m$ |
| 幂 | $a^k \bmod m = \text{mod\_pow}(a, k, m)$ |

### 模逆元

模逆元 $a^{-1}$ 满足 $a \times a^{-1} \equiv 1 \pmod{m}$。

**存在条件**：$\gcd(a, m) = 1$

**求法**：使用扩展欧几里得算法

```cpp
long long mod_inverse(long long a, long long mod) {
    long long x, y;
    long long g = exgcd(a, mod, x, y);
    if (g != 1) return -1;  // 不存在逆元
    return (x % mod + mod) % mod;
}
```

### 线性同余方程

求解 $ax \equiv b \pmod{m}$：

1. 令 $g = \gcd(a, m)$
2. 若 $b \bmod g \neq 0$，无解
3. 否则，除以 $g$ 后用扩展欧几里得求逆元

```cpp
// 解线性同余方程 ax ≡ b (mod m)
vector<long long> solve_linear_congruence(long long a, long long b, long long m) {
    vector<long long> solutions;
    long long g = gcd(a, m);
    
    if (b % g != 0) return solutions;  // 无解
    
    long long a1 = a / g, b1 = b / g, m1 = m / g;
    long long x, y;
    exgcd(a1, m1, x, y);
    x = (x % m1 + m1) % m1;
    x = x * b1 % m1;
    
    // 通解：x + k * m1
    for (long long k = 0; k < g; k++) {
        solutions.push_back((x + k * m1) % m);
    }
    return solutions;
}
```

## 中国剩余定理（CRT）

求解同余方程组：

$$
\begin{cases}
x \equiv a_1 \pmod{m_1} \\
x \equiv a_2 \pmod{m_2} \\
\vdots \\
x \equiv a_n \pmod{m_n}
\end{cases}
$$

其中 $m_1, m_2, \ldots, m_n$ 两两互质。

### 代码实现

```cpp
long long crt(const vector<long long>& a, const vector<long long>& m) {
    long long x = 0, M = 1;
    int n = a.size();
    
    for (int i = 0; i < n; i++) {
        long long Mi = M;
        long long ai = a[i] % m[i];
        long long xi, yi;
        long long gi = exgcd(Mi, m[i], xi, yi);
        // 合并：x = x + M * ((ai - x) * inv(Mi) mod m[i])
        long long t = ((ai - x % m[i] + m[i]) % m[i]) * mod_inverse(Mi / gi, m[i] / gi) % (m[i] / gi);
        x = x + M * t;
        M = M / gi * m[i];
        x = (x % M + M) % M;
    }
    
    return x;
}
```

## 素数判定

### 试除法

```cpp
bool is_prime(long long n) {
    if (n < 2) return false;
    for (long long i = 2; i * i <= n; i++) {
        if (n % i == 0) return false;
    }
    return true;
}
```

### 埃拉托斯特尼筛法（Sieve of Eratosthenes）

```cpp
vector<int> sieve(int n) {
    vector<bool> is_prime(n + 1, true);
    is_prime[0] = is_prime[1] = false;
    vector<int> primes;
    
    for (int i = 2; i <= n; i++) {
        if (is_prime[i]) {
            primes.push_back(i);
            if ((long long)i * i <= n) {
                for (int j = i * i; j <= n; j += i) {
                    is_prime[j] = false;
                }
            }
        }
    }
    return primes;
}
```

### 线性筛（欧拉筛）

```cpp
vector<int> linear_sieve(int n) {
    vector<int> primes;
    vector<bool> is_composite(n + 1, false);
    
    for (int i = 2; i <= n; i++) {
        if (!is_composite[i]) {
            primes.push_back(i);
        }
        for (int p : primes) {
            if (i * p > n) break;
            is_composite[i * p] = true;
            if (i % p == 0) break;
        }
    }
    return primes;
}
```

## 组合数学基础

### 排列组合

- 排列 $P(n, k) = \frac{n!}{(n-k)!}$
- 组合 $C(n, k) = \binom{n}{k} = \frac{n!}{k!(n-k)!}$

```cpp
long long C(int n, int k) {
    if (k < 0 || k > n) return 0;
    long long res = 1;
    for (int i = 1; i <= k; i++) {
        res = res * (n - k + i) / i;
    }
    return res;
}
```

### 模意义下的组合数

使用预处理阶乘和逆元：

```cpp
const int MOD = 1e9 + 7;
vector<long long> fac(MaxN), ifac(MaxN);

long long init_combination(int n) {
    fac[0] = 1;
    for (int i = 1; i <= n; i++) fac[i] = fac[i-1] * i % MOD;
    ifac[n] = mod_pow(fac[n], MOD - 2, MOD);
    for (int i = n; i > 0; i--) ifac[i-1] = ifac[i] * i % MOD;
}

long long C(int n, int k) {
    if (k < 0 || k > n) return 0;
    return fac[n] * ifac[k] % MOD * ifac[n-k] % MOD;
}
```

## 相关主题

- [[../algorithms/linear-sieve|线性筛]]：素数筛法的优化
- [[../data-structure/fenwick|树状数组]]：在某些数论问题中的应用
- 裴蜀定理、费马小定理等进阶数论内容

## 参考资料

[^1]: [数论基础 - OI Wiki](https://oi-wiki.org/math/number-theory/basic/)
[^2]: [最大公约数 - OI Wiki](https://oi-wiki.org/math/number-theory/gcd/)
[^3]: [模算术简介 - OI Wiki](https://oi-wiki.org/math/number-theory/mod-arithmetic/)
