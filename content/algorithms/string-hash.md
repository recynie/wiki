---
title: 字符串哈希
date: 2026-04-01
description: 字符串哈希（String Hash）是一种将字符串映射为整数值的技术，广泛应用于字符串匹配、子串查询和去重等场景。
tags:
  - string
  - hash
  - algorithm
draft: false
permalink:
---

## 1. 定义

字符串哈希（String Hash）是一种将任意长度的字符串映射为固定长度整数值的技术。通过哈希函数，我们可以将字符串转换为近似唯一的整数标识，从而将字符串操作转化为高效的整数运算。

### 为什么使用字符串哈希？

在处理字符串问题时，直接比较字符串的时间复杂度通常为 $O(n)$，而哈希可以将这一过程降低到 $O(1)$。字符串哈希的主要优势包括：

- **快速比较**：整数比较远快于字符串比较
- **空间压缩**：用固定长度的整数表示变长字符串
- **支持子串查询**：可在 $O(1)$ 时间内判断子串是否相等

字符串哈希在竞赛和工程中都有广泛应用，尤其适合需要频繁比较或查询子串的场景。[^1]

---

## 2. 哈希函数

### 2.1 多项式滚动哈希

最常用的字符串哈希方法是**多项式滚动哈希**（Polynomial Rolling Hash）。其核心思想是将字符串视为一个 base 进制数，然后对某个大模数取模。

给定字符串 $s = s_0 s_1 s_2 \cdots s_{n-1}$，哈希值定义为：

$$
H(s) = \sum_{i=0}^{n-1} s[i] \times base^i \mod M
$$

其中 $base$ 是进制基数（通常取质数，如 $131$ 或 $13331$），$M$ 是模数（通常取 $10^9+7$ 或 $10^9+9$）。

### 2.2 模数选择

模数的选择直接影响哈希的碰撞概率：

| 模数 | 特点 |
|------|------|
| $10^9+7$ | 常用质数，碰撞概率低 |
| $10^9+9$ | 与 $10^9+7$ 配合使用形成双哈希 |
| $2^{64}$ | 使用 unsigned long long 自然溢出，速度快 |

### 2.3 基数选择

基数的选择也很关键，通常要求：

- $base > |Σ|$（字符集大小）
- $base$ 为质数
- $base$ 不宜过大，否则容易溢出

常用选择：$131$、$13331$、$137$ 等。[^2]

---

## 3. 实现

### 3.1 基础实现

以下是多项式滚动哈希的典型 C++ 实现：

```cpp
#include <bits/stdc++.h>
using namespace std;

struct StringHash {
    using ull = unsigned long long;
    
    const ull base = 131;          // 进制基数
    const ull mod = 1000000007ULL; // 模数
    vector<ull> power;              // base^i mod mod
    vector<ull> hash;               // 前缀哈希
    
    // 构造函数，预计算幂次
    StringHash(const string& s) {
        int n = s.size();
        power.resize(n + 1);
        hash.resize(n + 1);
        power[0] = 1;
        hash[0] = 0;
        for (int i = 0; i < n; i++) {
            power[i + 1] = power[i] * base % mod;
            hash[i + 1] = (hash[i] * base % mod + (ull)s[i]) % mod;
        }
    }
    
    // 获取 [l, r) 区间的哈希值
    ull get(int l, int r) const {
        ull res = (hash[r] + mod - hash[l] * power[r - l] % mod) % mod;
        return res;
    }
};
```

### 3.2 使用示例

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    string s = "abcabcabc";
    StringHash sh(s);
    
    // 查询子串 "bca" 的哈希值
    cout << sh.get(1, 4) << endl;
    
    return 0;
}
```

### 3.3 自然溢出哈希

另一种常见实现利用 `unsigned long long` 的自然溢出特性，省去取模操作：

```cpp
struct RollingHash {
    using ull = unsigned long long;
    
    const ull base = 131;
    vector<ull> power;
    vector<ull> hash;
    
    RollingHash(const string& s) {
        int n = s.size();
        power.resize(n + 1);
        hash.resize(n + 1);
        power[0] = 1;
        for (int i = 0; i < n; i++) {
            power[i + 1] = power[i] * base;
            hash[i + 1] = hash[i] * base + (ull)s[i];
        }
    }
    
    // 获取 [l, r) 区间的哈希值
    ull get(int l, int r) const {
        return hash[r] - hash[l] * power[r - l];
    }
};
```

这种实现下，$M = 2^{64}$，利用 CPU 自然溢出的特性，碰撞概率约为 $1/2^{64}$。[^3]

---

## 4. 碰撞处理

### 4.1 双哈希

单哈希存在碰撞风险，实际应用中通常使用**双哈希**或**多哈希**来进一步降低碰撞概率：

```cpp
struct DoubleHash {
    const ull base = 131;
    const ull mod1 = 1000000007ULL;
    const ull mod2 = 1000000009ULL;
    
    vector<pair<ull, ull>> power, hash;
    
    DoubleHash(const string& s) {
        int n = s.size();
        power.resize(n + 1);
        hash.resize(n + 1);
        power[0] = {1, 1};
        hash[0] = {0, 0};
        for (int i = 0; i < n; i++) {
            power[i + 1].first = power[i].first * base % mod1;
            power[i + 1].second = power[i].second * base % mod2;
            hash[i + 1].first = (hash[i].first * base % mod1 + (ull)s[i]) % mod1;
            hash[i + 1].second = (hash[i].second * base % mod2 + (ull)s[i]) % mod2;
        }
    }
    
    pair<ull, ull> get(int l, int r) const {
        return {
            (hash[r].first + mod1 - hash[l].first * power[r - l].first % mod1) % mod1,
            (hash[r].second + mod2 - hash[l].second * power[r - l].second % mod2) % mod2
        };
    }
};
```

### 4.2 基数优化

选择多个不同的基数和模数组合，可以进一步减少碰撞：

| 组合 | 碰撞概率 |
|------|----------|
| 单哈希 $(base, mod)$ | 约 $1/mod$ |
| 双哈希 | 约 $1/mod_1 \times 1/mod_2$ |
| 多哈希 | 极低 |

---

## 5. 常见应用

### 5.1 字符串匹配

字符串哈希可以快速判断模式串是否出现在文本中：

```cpp
bool contains(const string& text, const string& pattern) {
    StringHash ht(text), hp(pattern);
    int n = text.size(), m = pattern.size();
    if (n < m) return false;
    
    ull patternHash = hp.get(0, m);
    for (int i = 0; i <= n - m; i++) {
        if (ht.get(i, i + m) == patternHash) {
            return true;
        }
    }
    return false;
}
```

时间复杂度为 $O(n + m)$。

### 5.2 子串查询

利用前缀哈希，可以在 $O(1)$ 时间内获取任意子串的哈希值，从而支持：

- **判断两个子串是否相等**：直接比较哈希值
- **最长公共前缀查询**：二分查找 + 哈希验证
- **字符串去重**：将所有子串哈希存入 set

### 5.3 去重

字符串哈希可用于大规模字符串去重场景：

```cpp
vector<string> deduplicate(const vector<string>& strs) {
    unordered_set<ull> seen;
    vector<string> result;
    StringHash sh;
    
    for (const string& s : strs) {
        sh = StringHash(s);
        ull h = sh.get(0, s.size());
        if (seen.insert(h).second) {
            result.push_back(s);
        }
    }
    return result;
}
```

---

## 6. 与其他算法的比较

### 6.1 与 [[kmp|KMP]] 的比较

[[kmp|KMP]]（Knuth-Morris-Pratt）算法和字符串哈希都可以用于字符串匹配，但各有优劣：

| 特性 | 字符串哈希 | [[kmp|KMP]] |
|------|------------|-------------|
| 时间复杂度 | $O(n + m)$ | $O(n + m)$ |
| 空间复杂度 | $O(n)$ | $O(m)$ |
| 预处理时间 | $O(n)$ | $O(m)$ |
| 碰撞风险 | 存在（可双哈希避免） | 无 |
| 灵活性 | 可处理复杂查询 | 仅精确匹配 |

[[kmp|KMP]] 的优势在于**确定性**：如果 KMP 说匹配，则一定匹配。字符串哈希则更适合需要快速判断、不严格要求 $100\%$ 准确的场景。

### 6.2 与 [[trie|Trie]] 的比较

[[trie|Trie]]（前缀树）适用于**前缀查询**类问题，而字符串哈希更适合**子串查询**和**相等性判断**：

| 特性 | 字符串哈希 | [[trie|Trie]] |
|------|------------|---------------|
| 主要用途 | 子串相等判断 | 前缀匹配 |
| 空间复杂度 | $O(n)$ | $O(n \times \Sigma)$ |
| 适用场景 | 去重、模式匹配 | 字典查询、前缀统计 |

在实际应用中，如果需要频繁查询某字符串是否出现过，字符串哈希更为高效；如果需要查询所有以某前缀开头的字符串，[[trie|Trie]] 更为适合。

---

## 7. 安全注意事项

### 7.1 哈希碰撞攻击

在 Web 应用或数据库场景中，恶意用户可能构造大量碰撞的字符串来发起**哈希洪水攻击**（Hash Flooding Attack）。防御措施包括：

- 使用**密钥哈希**（HMAC）：$H(s) = H_{salt}(s)$，在哈希计算时加入秘密盐值
- 使用**secure hash algorithm**：如 SHA-256（但性能较差）
- 使用 **randomized hash**：每个请求使用不同的随机种子

### 7.2 模数选择建议

| 场景 | 推荐配置 |
|------|----------|
| 竞赛编程 | 双哈希 $(131, 10^9+7)$ + $(13331, 10^9+9)$ |
| 工程应用 | 自然溢出或 HMAC-SHA256 |
| 安全敏感 | SHA-256 或更安全的算法 |

### 7.3 调试建议

在使用字符串哈希时，建议：

1. **保留原始字符串**：调试时可输出原始字符串验证
2. **边界情况检查**：空串、单字符等
3. **使用双哈希**：重要场景务必使用双哈希
4. **打印中间值**：确保预计算正确

---

## 8. 参考资料

[^1]: 本词条内容参考 [字符串哈希 - OI Wiki](https://oi-wiki.org/string/hash)
[^2]: 多项式哈希的详细内容见 [String Hashing - CP-Algorithms](https://cp-algorithms.com/string/string-hashing.html)
