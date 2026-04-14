---
title: 栈
date: 2026-04-06
description: 栈（Stack）是一种后进先出（LIFO）的数据结构
tags:
  - data-structure
  - stack
draft: false
permalink:
---

# 栈

栈（Stack）是一种**后进先出**（LIFO, Last In First Out）的线性数据结构。就像叠盘子一样，最后放上去的盘子最先被拿走。

## 基本操作

| 操作 | 说明 | 时间复杂度 |
|------|------|-----------|
| `push(x)` | 将元素压入栈顶 | $O(1)$ |
| `pop()` | 弹出栈顶元素 | $O(1)$ |
| `top()` | 查看栈顶元素 | $O(1)$ |
| `empty()` | 判断栈是否为空 | $O(1)$ |
| `size()` | 返回栈中元素个数 | $O(1)$ |

## STL中的栈

C++标准库提供了 `std::stack`：

```cpp
#include <stack>

stack<int> s;
s.push(1);      // {1}
s.push(2);      // {1, 2}
s.push(3);      // {1, 2, 3}

cout << s.top();   // 3
s.pop();           // {1, 2}
cout << s.top();   // 2
cout << s.size();  // 2
cout << s.empty(); // false
```

## 实现

### 数组实现

```cpp
template <typename T>
class Stack {
    vector<T> data;
public:
    void push(const T& x) {
        data.push_back(x);
    }
    
    void pop() {
        if (!empty()) data.pop_back();
    }
    
    T& top() {
        return data.back();
    }
    
    bool empty() const {
        return data.empty();
    }
    
    size_t size() const {
        return data.size();
    }
};
```

### 单链表实现

```cpp
template <typename T>
struct Node {
    T val;
    Node* next;
    Node(T v) : val(v), next(nullptr) {}
};

template <typename T>
class LinkedStack {
    Node<T>* topNode;
    size_t sz;
public:
    LinkedStack() : topNode(nullptr), sz(0) {}
    
    void push(const T& x) {
        Node<T>* newNode = new Node<T>(x);
        newNode->next = topNode;
        topNode = newNode;
        ++sz;
    }
    
    void pop() {
        if (empty()) return;
        Node<T>* temp = topNode;
        topNode = topNode->next;
        delete temp;
        --sz;
    }
    
    T& top() {
        return topNode->val;
    }
    
    bool empty() const {
        return topNode == nullptr;
    }
    
    size_t size() const {
        return sz;
    }
};
```

## 典型应用

### 1. 表达式求值

中缀表达式转后缀表达式（逆波兰表达式）：

```cpp
int precedence(char op) {
    if (op == '+' || op == '-') return 1;
    if (op == '*' || op == '/') return 2;
    return 0;
}

string infixToPostfix(const string& s) {
    stack<char> st;
    string result;
    
    for (char c : s) {
        if (isdigit(c)) {
            result += c;
        } else if (c == '(') {
            st.push(c);
        } else if (c == ')') {
            while (!st.empty() && st.top() != '(') {
                result += st.top();
                st.pop();
            }
            st.pop();  // 弹出 '('
        } else {
            while (!st.empty() && precedence(st.top()) >= precedence(c)) {
                result += st.top();
                st.pop();
            }
            st.push(c);
        }
    }
    
    while (!st.empty()) {
        result += st.top();
        st.pop();
    }
    
    return result;
}
```

### 2. 括号匹配

```cpp
bool isValid(const string& s) {
    stack<char> st;
    
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') {
            st.push(c);
        } else {
            if (st.empty()) return false;
            char top = st.top();
            st.pop();
            if ((c == ')' && top != '(') ||
                (c == ']' && top != '[') ||
                (c == '}' && top != '{')) {
                return false;
            }
        }
    }
    
    return st.empty();
}
```

### 3. 单调栈

单调栈用于快速找到每个元素最近的大于/小于它的元素，详见：[[monotonic-queue|单调队列]]

```cpp
vector<int> nextGreaterElement(const vector<int>& nums) {
    vector<int> result(nums.size(), -1);
    stack<int> st;  // 存索引
    
    for (int i = 0; i < nums.size(); ++i) {
        while (!st.empty() && nums[st.top()] < nums[i]) {
            result[st.top()] = nums[i];
            st.pop();
        }
        st.push(i);
    }
    
    return result;
}
```

### 4. 深度优先搜索（DFS）

DFS使用系统栈（递归）或显式栈实现，详见：[[../algorithms/depth-first-search|DFS]]

```cpp
void dfsIterative(int start, const vector<vector<int>>& g) {
    stack<int> st;
    vector<int> visited(g.size(), 0);
    
    st.push(start);
    visited[start] = 1;
    
    while (!st.empty()) {
        int u = st.top();
        st.pop();
        cout << u << " ";
        
        for (int v : g[u]) {
            if (!visited[v]) {
                visited[v] = 1;
                st.push(v);
            }
        }
    }
}
```

## 与其他结构的比较

| 操作 | 栈 | 队列 | 双端队列 |
|------|-----|------|---------|
| push/pop的一端 | 栈顶 | 队尾 | 两端皆可 |
| 应用 | DFS、递归 | BFS、调度 | 单调队列 |

## 参考

- [栈 - OI Wiki](https://oi-wiki.org/ds/stack/)
