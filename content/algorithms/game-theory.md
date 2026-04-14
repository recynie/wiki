---
title: 博弈论
date: 2026-04-03
description: 博弈论（Game Theory）与SG函数
tags:
  - game-theory
  - sg-function
  - combinatorial-game
draft: false
permalink:
---

## 1. 公平组合游戏 (Impartial Combinatorial Games)

### 定义

公平组合游戏满足以下条件：

- 两名玩家交替进行游戏
- 两名玩家的可行操作完全相同（公平性）
- 无法进行任何操作的患者输（通常为无子可取）
- 游戏不含随机因素（无掷骰子等）
- 游戏必然在有限步内结束

### 普通模式与异 SG模式 (Normal Play vs Misère Play)

| 模式 | 定义 |
|------|------|
| **普通模式 (Normal Play)** | 无法操作的玩家输 |
| **异 SG模式 (Misère Play)** | 无法操作的玩家赢 |

大多数竟赛题目采用普通模式，本文默认讨论普通模式。

---

## 2. Nim游戏 (Nim Game)

### 2.1 一堆 Nim

只有一堆石子，堆大小为 $n$。每次可以取 $1 \sim k$ 个石子（有时不设上限，即取任意正数）。

**先手必胜当且仅当** $n \bmod (k+1) \neq 0$。

```cpp
// 一堆Nim，先手必胜判断
bool firstWin(int n, int k) {
    return n % (k + 1) != 0;
}
```

### 2.2 多堆 Nim

有 $m$ 堆石子，第 $i$ 堆大小为 $a_i$。每堆独立操作，可以取任意正数个石子。

**核心结论**：设 $\text{xor} = a_1 \oplus a_2 \oplus \cdots \oplus a_m$（称为 Nim 和）。若 $\text{xor} \neq 0$，则先手必胜；否则先手必败。

```cpp
// 多堆Nim，先手必胜判断
bool firstWin(const vector<int>& piles) {
    int xorsum = 0;
    for (int v : piles) xorsum ^= v;
    return xorsum != 0;
}
```

**策略构造**：找到最高位的 $1$（设位于第 $k$ 位），选择对应的堆，将该堆石子数变为 $a_i' = a_i \oplus \text{xor}$（必然 $< a_i$），使新的 Nim 和为 $0$。

---

## 3. Sprague-Grundy 定理 (Sprague-Grundy Theorem)

### 3.1 Grundy 值 (SG 值)

对于状态 $s$，定义：

$$
\text{SG}(s) = \text{mex}\{ \text{SG}(s') \mid s' \text{ 为 } s \text{ 的后继状态} \}
$$

其中 $\text{mex}$ 为最小不在集合中的非负整数。

**性质**：

- $\text{SG}(s) = 0 \iff s$ 为必败状态
- $\text{SG}(s) > 0 \iff s$ 为必胜状态

### 3.2 SG 定理

任何公平组合游戏都等价于一个 Nim 堆，其大小等于该状态的 SG 值。

**组合游戏的 SG 值**：若干独立子游戏组合时，整体 SG 值为各子游戏 SG 值的异或和。

$$
\text{SG}(G_1 + G_2) = \text{SG}(G_1) \oplus \text{SG}(G_2)
$$

### 3.3 SG 值计算模板

```cpp
#include <bits/stdc++.h>
using namespace std;

// 递归计算SG值（记忆化）
int sg(vector<int>& dp, const vector<vector<int>>& g, int s) {
    if (dp[s] != -1) return dp[s];
    vector<int> mex(g[s].size() + 1, 0);  // 标记后继SG值
    for (int nxt : g[s]) {
        int val = sg(dp, g, nxt);
        if (val < (int)mex.size()) mex[val] = 1;
    }
    for (int i = 0; ; ++i) {
        if (!mex[i]) { dp[s] = i; break; }
    }
    return dp[s];
}

// 调用示例
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    int n;  // 状态数
    vector<vector<int>> g(n);  // 后继关系
    vector<int> dp(n, -1);
    
    // 计算SG值
    for (int i = 0; i < n; ++i) sg(dp, g, i);
    
    return 0;
}
```

---

## 4. 常见游戏的 SG 值

### 4.1 石子游戏 (Stone Games)

**单堆石子，取 $1 \sim k$ 个**：

$$
\text{SG}(n) = n \bmod (k+1)
$$

**验证**：后继状态为 $\{0, 1, \ldots, k\}$，故 $\text{mex}\{0, 1, \ldots, k\} = k+1$（当 $n > k$），递推即得上述公式。

```cpp
int sgStone(int n, int k) {
    return n % (k + 1);
}
```

**取石子放于两堆**（经典 Wythoff 游戏）：

状态 $(a, b)$（$a \leq b$）的 SG 值为 $\text{lowbit}(a \oplus b)$，必败态为 $\lfloor n \cdot \varphi \rfloor, \lfloor n \cdot \varphi^2 \rfloor$（$\varphi = \frac{1+\sqrt{5}}{2}$）。

### 4.2 台阶游戏 (Staircase Game)

游戏在台阶上进行，若石子位于第 $i$ 级台阶，每次可将石子下移任意步（不能低于地面）。多堆石子独立。

**结论**：SG 值为当前石子所在台阶编号的奇偶性。若位于奇数层则 SG = 1，偶数层则 SG = 0。因此奇数层的石子异或和决定胜负。

```cpp
int sgStaircase(int pos) {
    return pos % 2;  // 奇数层SG=1，偶数层SG=0
}
```

**尼姆博弈变形**：地面为第 $0$ 层，只能向下移动，无法移动者输。

### 4.3 删边游戏 (Dawson's Kayles)

一排 $n$ 个点，每次删除一个点及其相邻点（最多影响三个点）。

SG 值有递推公式，通过打表找规律：

| $n$ | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|-----|---|---|---|---|---|---|---|---|---|---|----|
| SG  | 0 | 1 | 2 | 3 | 1 | 4 | 3 | 2 | 1 | 4 | 2 |

```cpp
int sgKayles(int n) {
    static int sg[1005] = {0};
    static bool init = false;
    if (!init) {
        for (int i = 1; i <= 1000; ++i) {
            vector<int> mex(i + 1, 0);
            // 删除一个点，分割为左右两段
            for (int j = 0; j < i; ++j) {
                int left = j;
                int right = i - j - 1;
                int val = sg[left] ^ sg[right];
                if (val <= i) mex[val] = 1;
                // 删除相邻两个点
                if (j + 2 < i) {
                    left = j;
                    right = i - j - 3;
                    val = sg[left] ^ sg[right];
                    if (val <= i) mex[val] = 1;
                }
            }
            for (int k = 0; k <= i; ++k)
                if (!mex[k]) { sg[i] = k; break; }
        }
        init = true;
    }
    return sg[n];
}
```

---

## 5. NimK (Multi-heap Nim with Limit)

每堆限制最多取 $k$ 个石子，而非任意数量。

**解法**：将每堆石子数 $a_i$ 表示为 $k+1$ 进制，求每位之和。若某位和 $\bmod (k+1) \neq 0$，则先手必胜。

**特例 $k=1$（Take-Away 游戏）**：每堆只能取 $1$ 个或 $2$ 个。先手胜利条件为统计各堆石子数 $\bmod 3$，异或和不为 $0$。

```cpp
bool firstWinNimK(const vector<int>& piles, int k) {
    // 将各位之和 mod (k+1) 统计
    vector<int> cnt(k + 1, 0);
    for (int a : piles) {
        int idx = 0;
        while (a > 0) {
            cnt[idx++] += a % (k + 1);
            a /= (k + 1);
        }
    }
    int xorsum = 0;
    for (int i = 0; i <= k; ++i) {
        xorsum ^= cnt[i] % (k + 1);
    }
    return xorsum != 0;
}
```

---

## 6. 翻硬币游戏 (Turning Turtles)

### 6.1 一排翻硬币

$n$ 枚硬币排成一排，部分正面朝上 (H)，部分背面朝上 (T)。每次必须翻连续 $k$ 枚硬币（$1 \leq k \leq n$）。

**SG 值计算**：将每枚硬币位置独立考虑，SG 值为位置编号的 mex，最终异或和决定胜负。

### 6.2 翻动型翻硬币

每次可以选择一段连续区间，将区间内恰好 $m$ 枚正面朝上的硬币翻面。

**经典模型**： Gilmore-Gardner 定理给出的必败态为连续段长度的二进制表示。

### 6.3 翻硬币 SG 模板

```cpp
int sgCoinRow(const vector<int>& state, int k) {
    int n = state.size();
    vector<int> dp(1 << n, -1);
    function<int(int)> calc = [&](int mask) -> int {
        if (dp[mask] != -1) return dp[mask];
        vector<int> mex(n + 1, 0);
        for (int i = 0; i + k <= n; ++i) {
            int newMask = mask;
            for (int j = 0; j < k; ++j) newMask ^= (1 << (i + j));
            int val = calc(newMask);
            if (val <= n) mex[val] = 1;
        }
        for (int i = 0; ; ++i)
            if (!mex[i]) { dp[mask] = i; break; }
        return dp[mask];
    };
    int initMask = 0;
    for (int i = 0; i < n; ++i)
        if (state[i]) initMask |= (1 << i);
    return calc(initMask);
}
```

---

## 7. 必胜策略的构造

### 7.1 通用构造框架

1. **计算 SG 值**：对当前局面计算 SG 值
2. **判断胜负**：SG ≠ 0 为必胜态；SG = 0 为必败态
3. **寻找目标**：枚举所有后继状态，找到 SG 值为 $0$ 的状态
4. **执行操作**：将局面变为 SG 值为 $0$ 的必败态

```cpp
// 构造必胜操作
int findWinningMove(const vector<int>& piles) {
    int totalXor = 0;
    for (int v : piles) totalXor ^= v;
    if (totalXor == 0) return -1;  // 必败态无必胜操作
    
    for (int i = 0; i < (int)piles.size(); ++i) {
        int target = piles[i] ^ totalXor;
        if (target < piles[i]) {
            cout << "取第 " << i + 1 << " 堆，从 " << piles[i] 
                 << " 变为 " << target << "\n";
            return i;
        }
    }
    return -1;
}
```

### 7.2 NimK 策略构造

将石子数 $a_i$ 分解为 $k+1$ 进制，寻找最高位使得 $a_i$ 的该位 $> \text{carry}$，将 $a_i$ 调整为 $a_i - \delta$ 使每位平衡。

### 7.3 博弈树遍历

对于状态数较少的游戏，可直接深搜构造 SG 值：

```cpp
void solveGame(int startState) {
    unordered_map<int, int> sg;  // 状态到SG值的映射
    function<int(int)> getSG = [&](int state) -> int {
        if (sg.count(state)) return sg[state];
        vector<int> nxt = generateNext(state);
        vector<int> used(nxt.size() + 1, 0);
        for (int s : nxt) {
            int val = getSG(s);
            if (val < (int)used.size()) used[val] = 1;
        }
        for (int i = 0; ; ++i)
            if (!used[i]) return sg[state] = i;
    };
    
    int result = getSG(startState);
    cout << (result ? "先手必胜" : "先手必败") << "\n";
}
```

---

## 8. 典型例题

### 例 1：Nim 和异 SG 判定

给定三堆石子 $(a, b, c)$，判断先手胜负。

```cpp
bool nimTriple(int a, int b, int c) {
    return (a ^ b ^ c) != 0;
}
```

### 例 2：SG 博弈组合

两游戏并行，SG 值分别为 $g_1, g_2$，判断先手胜负：

```cpp
bool combinedGame(int g1, int g2) {
    return (g1 ^ g2) != 0;
}
```

### 例 3：台阶博弈变种

地面有 $n$ 堆石子，第 $i$ 堆 $a_i$ 个，位于第 $h_i$ 层。每次可将某堆石子下移任意步。求先手胜负。

```cpp
bool staircaseGame(const vector<int>& a, const vector<int>& h) {
    int xorsum = 0;
    for (int i = 0; i < (int)a.size(); ++i) {
        if (h[i] % 2 == 1) xorsum ^= a[i];
    }
    return xorsum != 0;
}
```

---

## 9. 总结

| 概念 | 核心要点 |
|------|----------|
| **Nim 游戏** | XOR 和为零则必败 |
| **SG 函数** | mex 后继集合，SG=0 为必败 |
| **SG 定理** | 组合游戏 SG = 各子游戏 SG 异或和 |
| **NimK** | 限制取石数，转为 $k+1$ 进制位和平衡 |
| **必胜策略** | 转为 SG=0 的必败态 |

**学习建议**：

1. 熟练掌握 Nim 的异或和判定
2. 理解 SG 函数的 mex 定义与递推
3. 记忆常见游戏的 SG 值打表（台阶、Kayles 等）
4. 练习从实际问题中抽象出 SG 模型

---

## 参考资料

[^1]: [博弈论 - OI Wiki](https://oi-wiki.org/game-theory/)

[^1]: 本段参考 [博弈论 - OI Wiki](https://oi-wiki.org/game-theory/)
