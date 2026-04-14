---
title: 计算理论
date: 2026-04-10
description: 计算理论入门——自动机理论、可计算性、复杂性理论
tags:
  - theory-of-computation
  - automata
  - computability
  - complexity
draft: false
permalink:
---

## 概述

计算理论（Theory of Computation）是计算机科学的基础理论分支，研究计算的本质、界限与复杂性。它回答三个核心问题[^1]：

1. **自动机理论**：计算的数学模型是什么？
2. **可计算性理论**：哪些问题能被计算？
3. **复杂性理论**：计算需要多少资源？

这三个问题由弱到强，层层递进，共同构成计算理论的基础框架。

---

## 形式语言与自动机

### 字符串与语言

设 $\Sigma$ 为有限字母表（alphabet），则：

- **字符串**（string）是 $\Sigma$ 上有限符号序列，如 $w = a_1a_2\cdots a_n$
- **长度**：$|w| = n$
- **空串**：$\varepsilon$（长度为0）
- **语言**（language）是 $\Sigma^*$ 的子集，即字符串的集合

**语言运算**：

- **连接**：$L_1L_2 = \{xy \mid x \in L_1, y \in L_2\}$
- **幂**：$L^n = LLL\cdots L$（$n$次）
- **闭包**：$L^* = L^0 \cup L^1 \cup L^2 \cup \cdots$

### 有限自动机

有限自动机（Finite Automaton，FA）是最简单的计算模型，分为确定性（DFA）和非确定性（NFA）两种。

#### 确定性有限自动机（DFA）

DFA 是一个五元组 $M = (Q, \Sigma, \delta, q_0, F)$：

| 组件 | 含义 |
|------|------|
| $Q$ | 有限状态集 |
| $\Sigma$ | 输入字母表 |
| $\delta: Q \times \Sigma \rightarrow Q$ | 转移函数 |
| $q_0 \in Q$ | 初始状态 |
| $F \subseteq Q$ | 接受状态集 |

```cpp
// DFA 实现示例：接受以 'ab' 结尾的字符串
#include <bits/stdc++.h>
using namespace std;

class DFA {
public:
    // 状态转移表：state -> (char -> next_state)
    // 0: 初始状态，等待 'a'
    // 1: 已读到 'a'，等待 'b'
    // 2: 已读到 'ab'，接受
    vector<array<int, 256>> trans;
    int start;
    unordered_set<int> accept;
    
    DFA() {
        trans.resize(3);
        for (int i = 0; i < 3; i++) trans[i].fill(-1);
        
        // 转移函数定义
        trans[0]['a'] = 1;  // 0 --a--> 1
        trans[1]['b'] = 2;  // 1 --b--> 2
        trans[2]['a'] = 1;  // 2 --a--> 1 (重新开始)
        
        start = 0;
        accept.insert(2);
    }
    
    bool accepts(const string& s) {
        int state = start;
        for (char c : s) {
            state = trans[state][(unsigned char)c];
            if (state == -1) return false;
        }
        return accept.count(state);
    }
};

int main() {
    DFA dfa;
    vector<string> tests = {"ab", "aab", "bab", "abab", "aba"};
    
    for (const string& s : tests) {
        cout << "\"" << s << "\" : " 
             << (dfa.accepts(s) ? "accept" : "reject") << endl;
    }
    return 0;
}
```

#### 非确定性有限自动机（NFA）

NFA 与 DFA 的区别在于：同一状态对同一输入可能有多个转移，或允许 $\varepsilon$ 转移。

**NFA 到 DFA 的等价转换**：子集构造法（powerset construction）

```cpp
// NFA 实现示例
class NFA {
public:
    // 转移函数：state -> char -> set of next states
    vector<unordered_map<char, unordered_set<int>>> trans;
    int start;
    unordered_set<int> accept;
    
    NFA(int num_states) {
        trans.resize(num_states);
        start = 0;
    }
    
    void add_trans(int from, char c, int to) {
        if (c == '@') {
            // epsilon 转移
        } else {
            trans[from][c].insert(to);
        }
    }
    
    // epsilon 闭包
    unordered_set<int> epsilon_closure(int state) {
        unordered_set<int> closure;
        stack<int> st;
        st.push(state);
        closure.insert(state);
        
        while (!st.empty()) {
            int cur = st.top(); st.pop();
            if (trans[cur].count('@')) {
                for (int next : trans[cur]['@']) {
                    if (!closure.count(next)) {
                        closure.insert(next);
                        st.push(next);
                    }
                }
            }
        }
        return closure;
    }
};
```

**DFA vs NFA**：

| 特性 | DFA | NFA |
|------|-----|-----|
| 转移确定性 | 唯一转移 | 多重转移 |
| $\varepsilon$ 转移 | 无 | 有 |
| 实现难度 | 简单 | 复杂 |
| 状态数 | 可能指数级 | 通常较少 |
| 等价性 | DFA ⊆ NFA | NFA 可转换为 DFA |

### 正则语言

**正则语言**（Regular Language）是由正则表达式描述的语言，或等价地，能被有限自动机识别的语言。

**正则表达式**：

| 表达式 | 描述 |
|--------|------|
| $a$ | 字符 $a$ |
| $\varepsilon$ | 空串 |
| $r_1 r_2$ | 连接 |
| $r_1 \mid r_2$ | 并（联合） |
| $r^*$ | 闭包（零或多次） |

**泵浦引理**（Pumping Lemma）用于证明某语言不是正则语言：

> 对于任意正则语言 $L$，存在常数 $p$（泵浦长度），使得任意长度 $\geq p$ 的字符串 $w \in L$ 可以分解为 $w = xyz$，满足：
> 1. $|y| \geq 1$
> 2. $|xy| \leq p$
> 3. 对所有 $i \geq 0$，$xy^iz \in L$

### 乔姆斯基体系

乔姆斯基体系（Chomsky Hierarchy）将形式语言分为四类[^2]：

| 类型 | 名称 | 生成器 | 识别器 |
|------|------|--------|--------|
| Type 3 | 正则语言 | 正则文法 | DFA/NFA |
| Type 2 | 上下文无关语言 | 上下文无关文法 | PDA |
| Type 1 | 上下文相关语言 | 上下文相关文法 | LBA |
| Type 0 | 递归可枚举语言 | 无限制文法 | 图灵机 |

#### 上下文无关文法（CFG）

CFG 是四元组 $G = (V, \Sigma, R, S)$：

- $V$：变量（非终结符）集合
- $\Sigma$：终结符集合
- $R$：产生式规则集合
- $S$：起始符号

**推导示例**：算术表达式 $E \rightarrow E+E \mid E*E \mid (E) \mid id$

```cpp
// 递归下降解析器示例
class Parser {
public:
    string input;
    int pos;
    
    Parser(const string& s) : input(s), pos(0) {}
    
    // E → E + T | E - T | T
    int parseExpr() {
        int val = parseTerm();
        while (pos < input.size() && (input[pos] == '+' || input[pos] == '-')) {
            char op = input[pos++];
            int rhs = parseTerm();
            val = (op == '+') ? val + rhs : val - rhs;
        }
        return val;
    }
    
    // T → T * F | F
    int parseTerm() {
        int val = parseFactor();
        while (pos < input.size() && (input[pos] == '*' || input[pos] == '/')) {
            char op = input[pos++];
            int rhs = parseFactor();
            val = (op == '*') ? val * rhs : val / rhs;
        }
        return val;
    }
    
    // F → (E) | number
    int parseFactor() {
        if (input[pos] == '(') {
            pos++;  // skip '('
            int val = parseExpr();
            pos++;  // skip ')'
            return val;
        }
        // 解析数字
        int val = 0;
        while (pos < input.size() && isdigit(input[pos])) {
            val = val * 10 + (input[pos++] - '0');
        }
        return val;
    }
};
```

---

## 可计算性理论

### 图灵机

图灵机（Turing Machine，TM）是最强有力的计算模型，由 Alan Turing 于 1936 年提出。

**图灵机是一个七元组** $M = (Q, \Sigma, \Gamma, \delta, q_0, q_{accept}, q_{reject})$：

| 组件 | 含义 |
|------|------|
| $Q$ | 有限状态集 |
| $\Sigma$ | 输入字母表（不含空格） |
| $\Gamma$ | 纸带字母表（含空格 $\sqcup$） |
| $\delta: Q \times \Gamma \rightarrow Q \times \Gamma \times \{L, R\}$ | 转移函数 |
| $q_0$ | 初始状态 |
| $q_{accept}$ | 接受状态 |
| $q_{reject}$ | 拒绝状态 |

```cpp
// 图灵机模拟器（简化版）
#include <bits/stdc++.h>
using namespace std;

class TuringMachine {
public:
    // 状态
    enum State { START, ACCEPT, REJECT };
    
    // 转移函数: (state, symbol) -> (next_state, write_symbol, direction)
    // direction: 0 = L, 1 = R
    map<pair<int, char>, tuple<int, char, int>> delta;
    
    State current_state;
    string tape;      // 纸带内容
    int head_pos;     // 读写头位置
    
    TuringMachine(const string& input) {
        tape = input + "_";  // '_' 表示空白符
        head_pos = 0;
        current_state = START;
    }
    
    void add_transition(int q, char s, int q', char w, int d) {
        delta[{q, s}] = {q', w, d};
    }
    
    bool run() {
        while (current_state != ACCEPT && current_state != REJECT) {
            char current = tape[head_pos];
            auto key = make_pair((int)current_state, current);
            
            if (delta.find(key) == delta.end()) {
                current_state = REJECT;
                break;
            }
            
            auto [next_state, write_sym, direction] = delta[key];
            
            // 写入
            tape[head_pos] = write_sym;
            
            // 移动
            if (direction == 1) {  // 右移
                head_pos++;
                if (head_pos >= (int)tape.size()) {
                    tape += "_";  // 扩展纸带
                }
            } else {  // 左移
                if (head_pos > 0) head_pos--;
            }
            
            current_state = (State)next_state;
        }
        return current_state == ACCEPT;
    }
};

// 示例：接受 {0^n 1^n | n >= 1}
class BinaryTM : public TuringMachine {
public:
    BinaryTM(const string& input) : TuringMachine(input) {
        // 状态定义
        enum { Q0, Q1, Q2, Q3, Q4, Q5 };
        
        // Q0: 找到最左边的 0，标记为 X
        add_transition(Q0, '0', Q1, 'X', 1);
        add_transition(Q0, 'Y', Q0, 'Y', 1);
        
        // Q1: 找到对应的 1，标记为 Y
        add_transition(Q1, '1', Q2, 'Y', 0);
        add_transition(Q1, 'Y', Q1, 'Y', 1);
        
        // Q2: 左移回到 X 或 Y
        add_transition(Q2, '0', Q2, '0', 0);
        add_transition(Q2, 'X', Q0, 'X', 1);
        add_transition(Q2, 'Y', Q2, 'Y', 0);
        
        // Q3-Q5: 验证完成
        add_transition(Q0, 'Y', Q3, 'Y', 1);
        add_transition(Q3, '_', Q4, '_', 0);  // 继续右移
        add_transition(Q4, '_', Q5, '_', 1);  // 到达接受状态
    }
};
```

### 通用图灵机与丘奇-图灵论题

**丘奇-图灵论题**（Church-Turing Thesis）：

> 任何可有效计算的函数都是图灵机可计算的。

这意味着图灵机等价于"算法"的概念——任何计算机能做的事，图灵机也能做。

### 决定性问题

**决定性问题**（Decision Problem）是回答是/否的问题。

- **可判定的**（Decidable）：存在算法能在有限时间内给出正确答案
- **不可判定的**（Undecidable）：不存在这样的算法

**停机问题**（Halting Problem）是经典不可判定问题：

> 给定任意程序 $P$ 和输入 $w$，判断 $P$ 在 $w$ 上是否会停机。

**证明思路**：反证法

假设存在停机判定器 $H(P, w)$，构造"悖论程序" $D$：

```
D(P):
    if H(P, P) == "halts":
        loop forever
    else:
        halt
```

调用 $D(D)$：若停机则无限循环，若不停机则停机——矛盾。

**莱斯定理**（Rice's Theorem）：

> 所有关于程序非平凡性质的断言都是不可判定的。

### 归约

**归约**（Reduction）将一个问题转化为另一个问题：

> 若 $A$ 可归约为 $B$（$A \leq_m B$），且 $B$ 可判定，则 $A$ 也可判定。

常见的不可判定问题：

| 问题 | 描述 |
|------|------|
| 停机问题 | 程序是否停机 |
| PCP | Post 对应问题 |
| 有效性 | 逻辑公式是否有效 |
| 包含性 | 一个语言是否包含另一个 |

---

## 计算复杂性理论

### 时间复杂性

**时间复杂性**（Time Complexity）$T(n)$ 是图灵机解决问题所需的最大步数。

**渐近记号**：

| 记号 | 含义 |
|------|------|
| $O(n)$ | 上界（不超过） |
| $\Omega(n)$ | 下界（不少于） |
| $\Theta(n)$ | 同阶（上下界相同） |

### P vs NP

**P 类**：能在多项式时间内由确定性图灵机解决的决定性问题。

**NP 类**：能在多项式时间内**验证**解的正确性的决定性问题。

$$P \subseteq NP$$

**关键问题**：$P = NP$ 还是 $P \subsetneq NP$？

> 这是千禧年七大数学难题之一，至今未解。

### NP-完全性

**NP-完全**（NP-Complete）问题是 NP 中最难的问题。

**库克-列文定理**（Cook-Levin Theorem）：

> SAT（布尔可满足性）是 NP-完全的。

**多归约**：若 $A \leq_m B$ 且 $A$ 是 NP-完全的，则 $B$ 也是 NP-完全的。

**经典 NP-完全问题**：

| 问题 | 描述 |
|------|------|
| SAT | 布尔公式可满足性 |
| 3SAT | 每个子句恰好3个文字的SAT |
| 顶点覆盖 | 最小顶点覆盖 |
| 哈密顿回路 | 经过所有顶点恰好一次的回路 |
| 旅行商问题 | 最短哈密顿回路 |
| 子集和 | 给定和的子集 |

```cpp
// 顶点覆盖问题的近似算法（贪心）
#include <bits/stdc++.h>
using namespace std;

class VertexCover {
public:
    int n;  // 顶点数
    vector<vector<int>> adj;  // 邻接表
    
    VertexCover(int n) : n(n), adj(n) {}
    
    void add_edge(int u, int v) {
        adj[u].push_back(v);
        adj[v].push_back(u);
    }
    
    // 贪心近似算法：O(V+E)
    // 每次选择度数最高的边，删除其两个端点
    vector<int> greedy_approx() {
        vector<int> cover;
        vector<bool> removed(n, false);
        
        while (true) {
            // 找到度数最大且未被删除的顶点
            int max_deg = -1, u = -1;
            for (int i = 0; i < n; i++) {
                if (!removed[i]) {
                    int deg = 0;
                    for (int v : adj[i]) {
                        if (!removed[v]) deg++;
                    }
                    if (deg > max_deg) {
                        max_deg = deg;
                        u = i;
                    }
                }
            }
            
            if (max_deg == 0) break;  // 所有边已覆盖
            
            // 选择 u 和它的一个邻居 v
            int v = -1;
            for (int to : adj[u]) {
                if (!removed[to]) {
                    v = to;
                    break;
                }
            }
            
            cover.push_back(u);
            cover.push_back(v);
            removed[u] = removed[v] = true;
        }
        
        return cover;
    }
};
```

### 其他复杂性类

| 复杂性类 | 定义 |
|----------|------|
| **L** | 对数空间 |
| **NL** | 非确定性对数空间 |
| **PSPACE** | 多项式空间 |
| **EXPTIME** | 指数时间 |
| **NPSPACE** | 非确定性多项式空间 |

**空间复杂性 vs 时间复杂性**：

- $TIME(t(n)) \subseteq SPACE(t(n))$（时间包含于空间）
- $SPACE(t(n)) \subseteq TIME(t(n)^2)$（空间可由时间模拟）

**萨维奇定理**（Savitch's Theorem）：

$$PSPACE = NPSPACE$$

---

## 参考资料

[^1]: 《Introduction to the Theory of Computation》 - Michael Sipser

[^2]: 《计算理论》- 清华大学出版社

[^3]: [MIT OpenCourseWare: Automata, Computability, and Complexity](https://ocw.mit.edu/courses/6-045j-automata-computability-and-complexity-spring-2011/)

---

## 相关主题

- [[../algorithms/binary-search|二分查找]] — 复杂度分析基础
- [[../math/combinatorics|组合数学]] — 计数基础
- [[../algorithms/graph-theory|图论算法]] — 复杂性问题实例
