---
title: 位运算 (Bit Manipulation)
date: 2026-04-03
description: 位运算基础、常用技巧、状态压缩与状压DP
tags:
  - bit-manipulation
  - algorithm
draft: false
permalink:
---

## 基础位运算 (Basic Bit Operations)

位运算是对整数二进制形式进行操作的运算，是算法竞赛中的基础技能。

| 运算 | 符号 | 说明 |
|------|------|------|
| AND | `&` | 按位与，均为1时结果为1 |
| OR | `|` | 按位或，存在1时结果为1 |
| XOR | `^` | 按位异或，相同为0，不同为1 |
| NOT | `~` | 按位取反 |
| 左移 | `<<` | 右侧补0，相当于乘2 |
| 右移 | `>>` | 左侧补符号位（算术右移） |

### 基础示例

```cpp
int a = 5;  // 二进制: 0101
int b = 3;  // 二进制: 0011

cout << (a & b) << endl;  // 1  (0001)
cout << (a | b) << endl;  // 7  (0111)
cout << (a ^ b) << endl;  // 6  (0110)
cout << (~a) << endl;     // -6 (补码)
cout << (a << 1) << endl; // 10 (1010)
cout << (a >> 1) << endl; // 2  (0010)
```

---

## 常用技巧 (Common Tricks)

### 判断奇偶

```cpp
// n 为奇数返回 true，偶数返回 false
bool isOdd = (n & 1);
```

原理：奇数二进制末位为1，偶数为0。

### 获取第 k 位

```cpp
// 获取 n 的第 k 位 (从0开始计数)
int bit = (n >> k) & 1;
```

### 设置第 k 位为 1

```cpp
// 将 n 的第 k 位设为1
n |= (1 << k);
```

### 清除第 k 位（即将该位设为0）

```cpp
// 将 n 的第 k 位设为0
n &= ~(1 << k);
```

### 取反第 k 位

```cpp
// 将 n 的第 k 位翻转
n ^= (1 << k);
```

### 获取最低位的 1 (lowbit)

```cpp
// 返回 n 二进制表示中最低位的1及其后面的0组成的数
int lowbit = n & (-n);
```

**应用**：可用于统计 n 中1的个数：

```cpp
int count = 0;
while (n) {
    n -= n & -n;  // 消除最低位的1
    ++count;
}
```

### 判断是否为 2 的幂

```cpp
// n 是2的幂返回 true
bool isPowerOf2 = (n & (n - 1)) == 0;
```

**原理**：2的幂（如 1000、10000）减1后会变成 0111、01111，与原数按位与结果为0。

### 统计 1 的个数

```cpp
// GCC/Clang 内置函数
int cnt = __builtin_popcount(n);      // 32位整数
int cntll = __builtin_popcountll(n);   // 64位整数
```

---

## 状态压缩 (State Compression)

状态压缩是利用 bitset 或整数的二进制位来表示一个集合，适用于集合规模较小（通常 ≤ 20~30）的情况。

### 基本表示

- 第 `i` 位为 1 表示元素 `i` 在集合中
- 第 `i` 位为 0 表示元素 `i` 不在集合中

```cpp
// 集合 {0, 2, 5} 的状态压缩表示
int state = (1 << 0) | (1 << 2) | (1 << 5);  // = 37
```

### 集合运算

```cpp
// 交集: A ∩ B
int intersection = A & B;

// 并集: A ∪ B
int union_set = A | B;

// 对称差: A Δ B (在A或B中，但不在两者交集中)
int symmetric_diff = A ^ B;

// 补集 (在全集中)
int complement = full ^ A;

// 判断元素 i 是否在集合 S 中
bool contains = (S >> i) & 1;

// 向集合中添加元素 i
S |= (1 << i);

// 从集合中移除元素 i
S &= ~(1 << i);
```

### 枚举子集

```cpp
// 枚举 state 的所有子集
for (int sub = state; sub; sub = (sub - 1) & state) {
    // sub 是 state 的一个非空子集
}

// 包含空集的版本
for (int sub = state; ; sub = (sub - 1) & state) {
    // 处理 sub
    if (sub == 0) break;
}
```

**原理**：`(sub - 1) & state` 跳过了一些状态，但能保证枚举所有子集且不重复。

---

## 实战应用 (Practical Applications)

### 1. 枚举子集问题

**问题**：给定集合 `S`，输出所有子集。

```cpp
vector<int> elements = {1, 2, 3};
int n = elements.size();
int state = (1 << n) - 1;  // 111 表示 {1,2,3}

for (int mask = 0; mask <= state; ++mask) {
    cout << "{ ";
    for (int i = 0; i < n; ++i) {
        if (mask & (1 << i))
            cout << elements[i] << " ";
    }
    cout << "}" << endl;
}
```

### 2. 状压 DP (State Compression DP)

**经典问题**：旅行商问题（TSP）的状压DP解法。

```cpp
const int INF = 1e9;
int n;  // 城市数量
int dist[20][20];
int dp[1 << 20][20];  // dp[mask][i] = 到达状态mask且终点为i的最小距离

// 初始化
memset(dp, INF, sizeof(dp));
dp[1 << 0][0] = 0;  // 从城市0出发

// DP转移
for (int mask = 0; mask < (1 << n); ++mask) {
    for (int u = 0; u < n; ++u) {
        if (!(mask & (1 << u)) || dp[mask][u] == INF) continue;
        for (int v = 0; v < n; ++v) {
            if (mask & (1 << v)) continue;
            int next = mask | (1 << v);
            dp[next][v] = min(dp[next][v], dp[mask][u] + dist[u][v]);
        }
    }
}

// 答案: 从任意城市返回起点的最小距离
int ans = INF;
int full = (1 << n) - 1;
for (int i = 1; i < n; ++i) {
    ans = min(ans, dp[full][i] + dist[i][0]);
}
```

### 3. 集合划分问题

将 n 个元素划分为两个集合，使得两集合元素和的差最小：

```cpp
int n;
vector<int> a(n);
int sum = 0;
for (int x : a) sum += x;

int ans = sum;
int state = 1 << n;
for (int mask = 0; mask < state; ++mask) {
    int s1 = 0;
    for (int i = 0; i < n; ++i) {
        if (mask & (1 << i)) s1 += a[i];
    }
    int s2 = sum - s1;
    ans = min(ans, abs(s1 - s2));
}
```

---

## Code Template

### 常用位操作模板

```cpp
struct BitUtils {
    // 获取最低位的1
    static int lowbit(int x) { return x & -x; }
    
    // 设置第k位为1
    static int setBit(int x, int k) { return x | (1 << k); }
    
    // 清除第k位
    static int clearBit(int x, int k) { return x & ~(1 << k); }
    
    // 翻转第k位
    static int flipBit(int x, int k) { return x ^ (1 << k); }
    
    // 获取第k位
    static int getBit(int x, int k) { return (x >> k) & 1; }
    
    // 判断是否为2的幂
    static bool isPowerOf2(int x) { return x && !(x & (x - 1)); }
    
    // 统计1的个数
    static int countBits(int x) { return __builtin_popcount(x); }
    
    // 枚举子集
    static void forEachSubset(int state, function<void(int)> f) {
        for (int sub = state; sub; sub = (sub - 1) & state) {
            f(sub);
        }
    }
};
```

---

## 参考资料

[^1]: [位运算 - OI Wiki](https://oi-wiki.org/math/bit/)

[^2]: [Bit Operations - CP-Algorithms](https://cp-algorithms.com/algebra/bit-operations.html)

