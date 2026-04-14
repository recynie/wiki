---
title: 在线算法
date: 2026-04-08
description: 在线算法（Online Algorithms）— 在不知道未来输入的情况下做出决策的算法，包括缓存置换、 Ski租赁、单位费用等问题
tags:
  - online-algorithm
  - competitive-ratio
  - paging
  - caching
draft: false
permalink:
---

## 概述

在线算法（Online Algorithm）是指在不知道未来输入的情况下，逐步处理输入并做出决策的算法。与离线算法不同，在线算法必须在信息不完整的情况下做出选择，且每个决策都是最终决策（不可撤销）。

### 核心问题

> 在不知道接下来会发生什么的情况下，如何做出最优决策？

### 性能衡量：竞争比

对于最小化问题，在线算法 $A$ 的**竞争比**（Competitive Ratio）定义为：

$$\rho = \inf_{I} \frac{cost_A(I)}{cost_{OPT}(I)}$$

其中 $c$ 为算法代价，$I$ 为输入实例，$OPT$ 为最优离线算法。

对于最大化问题，竞争比定义为：

$$\rho = \sup_{I} \frac{cost_A(I)}{cost_{OPT}(I)} \geq 1$$

一个在线算法被称为 $\rho$-竞争的，如果其竞争比至多为 $\rho$（最小化）或至少为 $\rho$（最大化）。

## 缓存置换算法

### 问题定义

缓存（Cache）容量为 $k$ 个页面，需要服务一系列 $m$ 个页面请求。缓存满时需要置换页面。

### 最优离线算法：Belady算法

```cpp
#include <bits/stdc++.h>
using namespace std;

// Belady最优算法（离线）- 置换将来最晚使用的页面
int beladyOffline(const vector<int>& requests, int cacheSize) {
    set<int> cache;
    int faults = 0;
    
    for (size_t i = 0; i < requests.size(); i++) {
        int page = requests[i];
        
        // 命中
        if (cache.count(page)) continue;
        
        // 缺失
        faults++;
        
        // 缓存满，需要置换
        if ((int)cache.size() == cacheSize) {
            // 找到将来最晚使用的页面
            int victim = -1;
            int farthest = -1;
            
            for (int cachedPage : cache) {
                // 找该页面下次出现的位置
                auto it = find(requests.begin() + i + 1, requests.end(), cachedPage);
                int nextUse = (it == requests.end()) ? -1 : (it - requests.begin());
                
                if (nextUse > farthest) {
                    farthest = nextUse;
                    victim = cachedPage;
                }
            }
            
            cache.erase(victim);
        }
        
        cache.insert(page);
    }
    
    return faults;
}
```

### LRU（最近最少使用）

```cpp
#include <bits/stdc++.h>
using namespace std;

// LRU缓存置换算法（在线）
int lruCache(const vector<int>& requests, int cacheSize) {
    list<int> cache;  // 头部最新，尾部最旧
    unordered_set<int> cacheSet;
    int faults = 0;
    
    for (int page : requests) {
        // 命中
        if (cacheSet.count(page)) {
            // 移到头部
            cache.remove(page);
            cache.push_front(page);
        } else {
            // 缺失
            faults++;
            
            // 缓存满，移除最旧的
            if ((int)cache.size() == cacheSize) {
                int victim = cache.back();
                cache.pop_back();
                cacheSet.erase(victim);
            }
            
            cache.push_front(page);
            cacheSet.insert(page);
        }
    }
    
    return faults;
}
```

### FIFO（先进先出）

```cpp
int fifoCache(const vector<int>& requests, int cacheSize) {
    queue<int> cache;
    unordered_set<int> cacheSet;
    int faults = 0;
    
    for (int page : requests) {
        if (cacheSet.count(page)) continue;
        
        faults++;
        
        if ((int)cache.size() == cacheSize) {
            int victim = cache.front();
            cache.pop();
            cacheSet.erase(victim);
        }
        
        cache.push(page);
        cacheSet.insert(page);
    }
    
    return faults;
}
```

### LFU（最不经常使用）

```cpp
int lfuCache(const vector<int>& requests, int cacheSize) {
    struct Node {
        int page;
        int freq;
        int lastUse;  // 最近一次使用的时间戳
    };
    
    vector<Node> cache;
    int time = 0;
    int faults = 0;
    
    for (int page : requests) {
        time++;
        
        // 查找是否命中
        int hit = -1;
        for (size_t i = 0; i < cache.size(); i++) {
            if (cache[i].page == page) {
                hit = i;
                break;
            }
        }
        
        if (hit >= 0) {
            cache[hit].freq++;
            cache[hit].lastUse = time;
        } else {
            faults++;
            
            if ((int)cache.size() == cacheSize) {
                // 移除频率最低的
                int minFreq = cache[0].freq;
                int minIdx = 0;
                for (size_t i = 1; i < cache.size(); i++) {
                    if (cache[i].freq < minFreq) {
                        minFreq = cache[i].freq;
                        minIdx = i;
                    }
                }
                cache.erase(cache.begin() + minIdx);
            }
            
            cache.push_back({page, 1, time});
        }
    }
    
    return faults;
}
```

### 竞争比分析

| 算法 | 竞争比 | 备注 |
|------|--------|------|
| Belady（离线） | 1 | 最优 |
| LRU | $k$ | $k$ 为缓存容量 |
| FIFO | $k$ | $k$ 为缓存容量 |
| LFU | $O(\log k)$ | 理论上优于LRU |

## Ski Rental问题

### 问题定义

想买一副滑雪板，但不确定会滑雪多少次。每次滑雪可以租一副（费用 $r$）或买一副（费用 $b$，之后免费滑雪）。问最优策略。

### 最优在线策略

```cpp
#include <bits/stdc++.h>
using namespace std;

// Ski Rental问题的在线算法
// 策略：先租b/r-1次，然后购买
double skiRental(double rentCost, double buyCost) {
    int rentTimes = (int)(buyCost / rentCost) - 1;
    return rentTimes * rentCost + buyCost;
}

// 竞争比分析
// 设最优离线在第t次滑雪后购买
// 在线代价：min(t, b/r) * r + b = max(t, b/r) * r + b - (t < b/r ? b : 0)
// 竞争比 = max_{t} 在线代价/最优代价
```

### 竞争比

滑雪租赁问题的竞争比为 $e/(e-1) \approx 1.58$。

## 在线匹配

### 问题定义

在二分图中，左侧顶点依次到来，每个顶点需要立即与一个未匹配的右侧顶点匹配（如果存在）。

### 贪婪算法（Random Ranking）

```cpp
#include <bits/stdc++.h>
using namespace std;

// 在线二分匹配 - 贪婪算法
class OnlineBipartiteMatching {
private:
    mt19937 rng;
    
public:
    OnlineBipartiteMatching() {
        random_device rd;
        rng = mt19937(rd());
    }
    
    // 贪婪匹配：随机选择一个未匹配的邻居
    vector<pair<int,int>> greedyMatch(
        int nLeft,                           // 左侧顶点数
        const vector<vector<int>>& edges    // 左侧到右侧的边
    ) {
        vector<bool> matchedRight(edges.size() > 0 ? edges.size() : 0, false);
        vector<int> matchLeft(nLeft, -1);
        vector<int> matchRight; // 右侧匹配情况
        
        int matched = 0;
        
        for (int u = 0; u < nLeft; u++) {
            // 随机打乱邻居顺序
            vector<int> neighbors = edges[u];
            shuffle(neighbors.begin(), neighbors.end(), rng);
            
            // 选择第一个未匹配的邻居
            for (int v : neighbors) {
                if (v >= (int)matchRight.size()) {
                    matchRight.resize(v + 1, -1);
                }
                if (matchRight[v] == -1) {
                    matchLeft[u] = v;
                    matchRight[v] = u;
                    matched++;
                    break;
                }
            }
        }
        
        vector<pair<int,int>> result;
        for (int u = 0; u < nLeft; u++) {
            if (matchLeft[u] != -1) {
                result.push_back({u, matchLeft[u]});
            }
        }
        return result;
    }
    
    // Ranking算法：离线随机排列右侧顶点
    vector<pair<int,int>> rankingMatch(
        int nLeft, int nRight,
        const vector<vector<int>>& edges
    ) {
        // 随机排列右侧顶点
        vector<int> order(nRight);
        iota(order.begin(), order.end(), 0);
        shuffle(order.begin(), order.end(), rng);
        
        vector<int> pos(nRight);
        for (int i = 0; i < nRight; i++) {
            pos[order[i]] = i;
        }
        
        // 将左侧顶点按邻居最小位置排序
        vector<pair<int,int>> leftOrder;
        for (int u = 0; u < nLeft; u++) {
            if (!edges[u].empty()) {
                int minPos = nRight;
                for (int v : edges[u]) {
                    minPos = min(minPos, pos[v]);
                }
                leftOrder.push_back({minPos, u});
            }
        }
        
        sort(leftOrder.begin(), leftOrder.end());
        
        vector<int> matchRight(nRight, -1);
        vector<pair<int,int>> result;
        
        for (auto& [_, u] : leftOrder) {
            for (int v : edges[u]) {
                if (matchRight[pos[v]] == -1) {
                    matchRight[pos[v]] = u;
                    result.push_back({u, v});
                    break;
                }
            }
        }
        
        return result;
    }
};
```

### 竞争比

| 算法 | 竞争比 | 说明 |
|------|--------|------|
| 贪婪 | $O(\log n)$ | 较差 |
| Ranking | $e$ | 最优在线算法 |

## 在线负载均衡

### Graham's Scheduling（在线列表调度）

```cpp
#include <bits/stdc++.h>
using namespace std;

// 在线作业调度 - 最短作业优先（SJF）
struct Job {
    int id;
    int length;
    int releaseTime;
};

class OnlineScheduler {
public:
    // 最短作业优先调度
    vector<pair<int,int>> sjfSchedule(const vector<Job>& jobs) {
        vector<pair<int,int>> schedule;  // (machine, finishTime)
        vector<int> machineFreeTime(4, 0);  // 4台机器
        priority_queue<pair<int,int>, vector<pair<int,int>>, greater<pair<int,int>>> pq;
        
        // 按释放时间排序
        vector<Job> sortedJobs = jobs;
        sort(sortedJobs.begin(), sortedJobs.end(), 
             [](const Job& a, const Job& b) { return a.releaseTime < b.releaseTime; });
        
        size_t idx = 0;
        int currentTime = 0;
        
        while (idx < sortedJobs.size() || !pq.empty()) {
            // 加入所有已释放的作业
            while (idx < sortedJobs.size() && sortedJobs[idx].releaseTime <= currentTime) {
                pq.push({sortedJobs[idx].length, sortedJobs[idx].id});
                idx++;
            }
            
            if (!pq.empty()) {
                auto [len, id] = pq.top(); pq.pop();
                // 分配给最早空闲的机器
                int minMachine = 0;
                int minTime = machineFreeTime[0];
                for (int i = 1; i < 4; i++) {
                    if (machineFreeTime[i] < minTime) {
                        minTime = machineFreeTime[i];
                        minMachine = i;
                    }
                }
                
                int startTime = max(currentTime, machineFreeTime[minMachine]);
                int finishTime = startTime + len;
                machineFreeTime[minMachine] = finishTime;
                schedule.push_back({minMachine, finishTime});
                currentTime = startTime;
            } else if (idx < sortedJobs.size()) {
                currentTime = sortedJobs[idx].releaseTime;
            }
        }
        
        return schedule;
    }
};
```

### 竞争比

Graham's SJF 调度的竞争比为 $\frac{4}{3} - \frac{1}{3m}$（$m$ 为机器数）。

## 常用在线算法策略

| 策略 | 思想 | 典型应用 |
|------|------|----------|
| 贪心 | 局部最优选择 | 缓存置换 |
| 随机化 | 随机决策 | 在线匹配 |
| 延迟决策 | 等待更多信息 | ski rental |
| 预测 | 使用预测信息 | 调度 |

## 与离线算法的对比

| 方面 | 在线算法 | 离线算法 |
|------|----------|----------|
| 输入 | 逐步到达 | 已知全部 |
| 决策 | 即时不可撤销 | 可回溯修改 |
| 分析 | 竞争比 | 最优性 |
| 复杂度 | 通常更困难 | 标准方法 |

## 应用场景

- **操作系统**：页面置换、进程调度
- **网络**：路由、拥塞控制
- **云计算**：资源分配、负载均衡
- **推荐系统**：冷启动问题
- **金融**：订单执行

## 参考文献

[^1]: Online Algorithms - Seffi Naor
[^2]: Online Computation and Competitive Analysis - Allan Borodin, Ran El-Yaniv
[^3]: OI Wiki - 在线算法 https://oi-wiki.org/algo/online/

---

*本页面内容基于在线算法经典理论整理*
