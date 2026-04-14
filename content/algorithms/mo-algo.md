---
title: 莫队算法
date: 2026-04-05
description: 莫队算法（Mo's Algorithm）详解：离线区间查询处理技巧，基于平方分割和排序
tags:
  - mo-algo
  - offline-query
  - algorithm
draft: false
permalink:
---

## 定义

**莫队算法**是一种离线处理区间查询的技巧，适用于满足「从 $[l,r]$ 的答案可以 $O(1)$ 扩展到相邻区间」条件的题目。[^1]

核心思想：将所有询问离线，对询问进行特殊排序，然后顺序处理，通过移动指针逐步扩展/收缩区间。

## 形式化

对于序列上的区间询问问题，如果从 $[l,r]$ 的答案能够 $O(1)$ 扩展到：
- $[l-1, r]$
- $[l+1, r]$
- $[l, r+1]$
- $[l, r-1]$

则可以在 $O(n\sqrt{n})$ 的复杂度内求出所有询问的答案。

## 排序方法

对于区间 $[l, r]$，以 $l$ 所在块的编号为第一关键字，$r$ 为第二关键字从小到大排序。

```cpp
struct Query {
    int l, r, id, block;
};

bool cmp(const Query& a, const Query& b) {
    if (a.block != b.block) return a.block < b.block;
    return a.r < b.r;
}
```

## 算法实现

```cpp
int BLOCK_SIZE;

void move(int pos, int sign) {
    // 根据题目更新答案
}

vector<long long> solve(int n, int m, const vector<Query>& queries) {
    BLOCK_SIZE = static_cast<int>(sqrt(n));
    vector<Query> q = queries;
    sort(q.begin(), q.end(), cmp);
    
    vector<long long> ans(m);
    int l = 1, r = 0;  // 当前区间 [l, r]
    long long nowAns = 0;
    
    for (const auto& qu : q) {
        while (l > qu.l) move(--l, 1);
        while (r < qu.r) move(++r, 1);
        while (l < qu.l) move(l++, -1);
        while (r > qu.r) move(r--, -1);
        ans[qu.id] = nowAns;
    }
    return ans;
}
```

## 复杂度分析

设块长度为 $S$，则：
- 同一块内，$r$ 的移动总代价为 $O(n)$
- 跨块时，$l$ 的移动总代价约为 $O(n \cdot S)$
- 总复杂度：$O\left(\frac{n^2}{S} + mS\right)$

当 $S = \frac{n}{\sqrt{m}}$ 时，复杂度最优为 $O(n\sqrt{m})$。

当 $n = m$ 时，取 $S = \sqrt{n}$，复杂度为 $O(n\sqrt{n})$。

## 例题：小Z的袜子

**题目**：求区间 $[l,r]$ 内随机选两个数，它们相等的概率。

**思路**：莫队模板题。维护每种颜色的出现次数和当前配对方案数。

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Query {
    int l, r, id, block;
};

int n, m, BLOCK_SIZE;
vector<int> a;
vector<Query> queries;
vector<long long> ansNum, ansDen;

bool cmp(const Query& a, const Query& b) {
    if (a.block != b.block) return a.block < b.block;
    return (a.block & 1) ? a.r < b.r : a.r > b.r;  // 奇偶优化
}

int cnt[500005];

void add(int pos) {
    int c = a[pos];
    ansNum[0] += cnt[c];
    cnt[c]++;
}

void remove(int pos) {
    int c = a[pos];
    cnt[c]--;
    ansNum[0] -= cnt[c];
}

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    cin >> n >> m;
    a.resize(n + 1);
    for (int i = 1; i <= n; i++) cin >> a[i];
    
    BLOCK_SIZE = static_cast<int>(sqrt(n));
    queries.resize(m);
    ansNum.resize(m);
    ansDen.resize(m);
    
    for (int i = 0; i < m; i++) {
        cin >> queries[i].l >> queries[i].r;
        queries[i].id = i;
        queries[i].block = queries[i].l / BLOCK_SIZE;
    }
    
    sort(queries.begin(), queries.end(), cmp);
    
    int l = 1, r = 0;
    for (const auto& q : queries) {
        while (l > q.l) add(--l);
        while (r < q.r) add(++r);
        while (l < q.l) remove(l++);
        while (r > q.r) remove(r--);
        
        long long len = q.r - q.l + 1;
        ansNum[q.id] = ansNum[0];
        ansDen[q.id] = len * (len - 1) / 2;
    }
    
    for (int i = 0; i < m; i++) {
        long long g = gcd(ansNum[i], ansDen[i]);
        cout << ansNum[i] / g << '/' << ansDen[i] / g << '\n';
    }
    return 0;
}
```

**公式推导**：从 $cnt[k]$ 个颜色 $k$ 的元素中选两个，方案数为 $\binom{cnt[k]}{2}$。总方案数为 $\binom{len}{2}$。颜色 $k$ 加入时，新增配对数为 $cnt[k]$（因为可以和之前的 $cnt[k]$ 个配对）。

## 莫队的优化

### 奇偶性优化

在同一块内，$r$ 交替往返移动。可以让奇数块正序、偶数块逆序：

```cpp
bool cmp(const Query& a, const Query& b) {
    if (a.block != b.block) return a.block < b.block;
    return (a.block & 1) ? a.r < b.r : a.r > b.r;
}
```

## 扩展：带修改莫队

当问题支持单点修改时（称为「修改莫队」），需要将询问扩展为三元组 $(l, r, t)$，其中 $t$ 表示第 $t$ 个修改后的状态。

块大小通常设为 $n^{2/3}$，复杂度为 $O(n^{5/3})$。

## 应用场景

- 区间众数、区间出现次数
- 区间不同元素数量
- 区间满足某种条件的对数
- 需要离线处理的大量区间查询

## 参考资料

[^1]: [普通莫队算法 - OI Wiki](https://oi-wiki.org/misc/mo-algo/)
