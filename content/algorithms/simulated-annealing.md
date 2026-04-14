---
title: 模拟退火与爬山算法
date: 2026-04-06
description: 模拟退火（Simulated Annealing）与爬山算法（Hill Climbing）详解，用于解决NP-hard问题的近似优化
tags:
  - simulated-annealing
  - optimization
  - approximation
  - algorithm
draft: false
permalink:
---

## 概述

**模拟退火（Simulated Annealing, SA）** 和 **爬山算法（Hill Climbing, HC）** 是两类基于随机化的局部搜索算法，用于在解空间中找到近似最优解。[^1]

爬山算法是最简单的局部搜索，容易陷入局部最优。模拟退火通过引入"温度"概念和概率接受劣解来跳出局部最优，是竞赛中解决NP-hard问题的重要近似算法。

## 爬山算法（Hill Climbing）

### 算法思想

从某个初始解出发，在当前解的邻域内寻找更好的解，若找到则移动到该解，重复直到邻域内无更优解。

### 代码框架

```cpp
struct Solution {
    // 解的表示
    double evaluate() const;  // 目标函数值
};

Solution hillClimbing(Solution init, int maxIter) {
    Solution cur = init, best = cur;
    for (int iter = 0; iter < maxIter; iter++) {
        Solution nxt = cur.neighbor();  // 产生邻域解
        if (nxt.evaluate() > cur.evaluate()) {
            cur = nxt;  // 接受更优解
            if (cur.evaluate() > best.evaluate()) best = cur;
        }
    }
    return best;
}
```

### 问题

爬山算法容易陷入**局部最优解**，无法跳出。

## 模拟退火算法

### 算法思想

模拟退火源于金属退火的物理过程。算法维护一个"温度"参数 $T$，初期温度高时允许接受较差的解，随着温度降低逐渐只接受更优解。

核心是 **Metropolis准则**：

$$
P(\text{接受}) = \begin{cases}
1 & \Delta E < 0 \\
e^{-\Delta E / T} & \Delta E \geq 0
\end{cases}
$$

其中 $\Delta E = f(\text{新解}) - f(\text{当前解})$。

### 代码框架

```cpp
struct Solution {
    double x, y;  // 解的表示（可能是坐标等）
    double evaluate() const;  // 目标函数值
};

Solution cur = init, best = cur;
double T = INIT_TEMP;  // 初始温度
double cool = COOL_RATE;  // 降温率，通常 0.995~0.999

while (T > MIN_TEMP) {
    for (int i = 0; i < ITER_PER_TEMP; i++) {
        Solution nxt = cur.neighbor(T);  // 基于温度产生邻域解
        double delta = nxt.evaluate() - cur.evaluate();
        
        if (delta > 0 || rnd() < exp(delta / T)) {
            cur = nxt;  // 接受新解
        }
        if (cur.evaluate() > best.evaluate()) {
            best = cur;
        }
    }
    T *= cool;  // 降温
}
```

### 关键参数

| 参数 | 建议值 | 说明 |
|------|--------|------|
| 初始温度 $T_0$ | $100 \sim 1000$ | 足够高以跳出局部最优 |
| 降温率 $\alpha$ | $0.995 \sim 0.999$ | 越接近1降温越慢 |
| 每温度迭代次数 | $100 \sim 1000$ | 温度内充分搜索 |
| 终止温度 $T_{min}$ | $10^{-6} \sim 10^{-4}$ | 足够小以收敛 |

## 典型应用

### 1. 平面最近邻问题（旅行商近似）

```cpp
double dist(Point a, Point b) {
    return sqrt((a.x-b.x)*(a.x-b.x) + (a.y-b.y)*(a.y-b.y));
}

Solution neighbor(const Solution& cur, double T) {
    Solution nxt = cur;
    // 随机交换两个城市
    int i = rnd() % n, j = rnd() % n;
    swap(nxt.path[i], nxt.path[j]);
    // 基于温度调整扰动幅度
    if (rnd() < 0.5) {
        nxt.path[i] += (rnd() - 0.5) * T * 0.01;
    }
    return nxt;
}
```

### 2. 函数最优化

```cpp
// 例：求解 f(x,y) = sin(x) + cos(y) 的最大值
struct Solution {
    double x, y;
    double evaluate() const {
        return sin(x) + cos(y);
    }
    Solution neighbor(double T) {
        Solution nxt;
        nxt.x = x + (rnd() - 0.5) * T;
        nxt.y = y + (rnd() - 0.5) * T;
        return nxt;
    }
};
```

### 3. 调度问题

模拟退火常用于 Job Shop Scheduling 等组合优化问题。

## 爬山+模拟退火对比

| 特性 | 爬山算法 | 模拟退火 |
|------|---------|---------|
| 时间复杂度 | $O(\text{迭代次数})$ | $O(\text{迭代次数})$ |
| 解的质量 | 容易局部最优 | 更好的全局近似 |
| 参数调优 | 几乎不需要 | 需要精心调整 |
| 适用场景 | 解空间平滑 | NP-hard问题 |

## 进阶技巧

### 多次运行取最优

由于随机性，单次SA可能效果不佳。通常：

```cpp
Solution bestOverall;
double bestVal = -INF;
for (int run = 0; run < 5; run++) {
    Solution cur = randomInit();
    Solution curBest = simulatedAnnealing(cur);
    if (curBest.evaluate() > bestVal) {
        bestVal = curBest.evaluate();
        bestOverall = curBest;
    }
}
```

### 自适应参数

- 初期使用较大扰动，后期逐渐减小
- 当解质量停滞时增大扰动

## 参考资料

[^1]: 本内容参考 [OI-Wiki 模拟退火](https://oi-wiki.org/misc/simulated-annealing/)，内容经过验证和扩展。

---

*Last updated: 2026-04-06*