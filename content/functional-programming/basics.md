---
title: 函数式编程基础
date: 2026-04-10
description: 函数式编程入门——函数式编程的核心概念、范式与实践
tags:
  - functional-programming
  - programming-paradigm
  - fp
draft: false
permalink:
---

## 概述

函数式编程（Functional Programming，FP）是一种**以函数为核心**的编程范式。它强调：

- **不可变性**（Immutability）：数据创建后不可修改
- **纯函数**（Pure Functions）：相同输入总是产生相同输出，无副作用
- **函数一等公民**（First-Class Functions）：函数可作为参数、返回值、赋值给变量
- **表达式求值**（Expression Evaluation）：程序是表达式的组合，而非语句序列

函数式编程并非全新的概念，其根源可追溯到 1930 年代的 **λ 演算**（Lambda Calculus）[^1]，由 Alonzo Church 提出。Lisp（1958）、ML、Haskell 等语言则是函数式编程的典型代表。

现代语言如 C++（11+）、Python、JavaScript、Rust 都引入了函数式特性。

---

## 核心概念

### 函数一等公民

在函数式编程中，函数是「值」，可以：

1. **赋值给变量**
2. **作为参数传递**
3. **作为返回值**
4. **存储在数据结构中**

```cpp
#include <functional>
#include <iostream>
using namespace std;

// 1. 函数赋值给变量
int add(int a, int b) { return a + b; }
auto add_fn = add;  // 函数指针

// 2. 函数作为参数（高阶函数）
void apply(int x, int y, function<int(int,int)> f) {
    cout << f(x, y) << endl;
}

// 3. 函数作为返回值
function<int(int,int)> get_operation(char op) {
    if (op == '+') return [](int a, int b){ return a + b; };
    if (op == '*') return [](int a, int b){ return a * b; };
    return [](int, int){ return 0; };
}

int main() {
    // 用法示例
    apply(3, 4, add_fn);                    // 7
    apply(3, 4, [](int a, int b){ return a + b; });  // lambda
    
    auto op = get_operation('*');
    cout << op(3, 4) << endl;               // 12
}
```

### 纯函数与副作用

**纯函数**满足两个条件：
- **确定性**：相同输入产生相同输出
- **无副作用**：不修改外部状态（不修改全局变量、I/O、抛出异常等）

```cpp
// 纯函数
int pure_add(int a, int b) {
    return a + b;  // 无副作用，可预测
}

// 非纯函数（不纯）
int counter = 0;
int impure_add(int a, int b) {
    counter++;  // 修改外部状态——副作用
    return a + b;
}
```

**为什么追求纯函数？**

| 特性 | 优势 |
|------|------|
| **可缓存** | 相同输入可记忆化（Memoization） |
| **可测试** | 无需 mock 外部依赖 |
| **可并行化** | 无竞争条件，无需同步 |
| **可组合** | 模块化，易推理 |

### 不可变性

不可变性要求数据创建后不可修改，而非「不能改变量」。

```cpp
// 可变方式（传统 OOP）
vector<int> add_to_vec(vector<int> v, int x) {
    v.push_back(x);  // 修改原数据
    return v;
}

// 不可变方式（函数式）
vector<int> add_to_vec(const vector<int>& v, int x) {
    vector<int> result = v;  // 复制
    result.push_back(x);     // 修改副本
    return result;           // 返回新数据
}

// C++17 结构化绑定 + pure 函数示例
auto add_to_vec_immutable(const vector<int>& v, int x) {
    auto result = v;
    result.push_back(x);
    return result;
}
```

---

## 函数组合

函数式编程的核心思想之一是：**小函数组合成大程序**。

### 复合函数

数学上的函数复合 $(f \circ g)(x) = f(g(x))$：

```cpp
#include <functional>

// 复合函数：f(g(x))
template<typename F, typename G>
auto compose(F f, G g) {
    return [f, g](auto x) {
        return f(g(x));
    };
}

// 示例
auto add_one = [](int x){ return x + 1; };
auto double_it = [](int x){ return x * 2; };
auto add_then_double = compose(double_it, add_one);

int main() {
    cout << add_then_double(5) << endl;  // (5+1)*2 = 12
}
```

### Pipeline 运算符

类似 Unix 的管道，将数据通过一系列函数「流动」：

```cpp
#include <functional>
#include <vector>
#include <algorithm>
#include <iostream>
using namespace std;

// 简化版 pipe 实现
auto pipe = [](auto... functions) {
    return [=](auto value) {
        return ((void)value, ...);  // C++17 fold expression
    };
};

// 更实用的版本
template<typename T>
auto apply_pipeline(const T& value, const vector<function<T(T)>>& funcs) {
    T result = value;
    for (const auto& f : funcs) {
        result = f(result);
    }
    return result;
}

int main() {
    vector<int> data = {1, 2, 3, 4, 5};
    
    // 传统写法：嵌套调用
    auto result1 = double_it(add_one(3));  // (3+1)*2 = 8
    
    // 函数式 pipeline 写法
    auto funcs = vector<function<int(int)>>{
        [](int x){ return x + 1; },   // 加 1
        [](int x){ return x * 2; },   // 翻倍
        [](int x){ return x + 10; }   // 加 10
    };
    
    cout << apply_pipeline(3, funcs) << endl;  // (3+1)*2+10 = 18
}
```

---

## 常见函数式模式

### Map

对集合中每个元素应用函数，返回新集合：

```cpp
#include <vector>
#include <algorithm>
#include <iostream>
using namespace std;

// 传统循环方式
vector<int> double_all(const vector<int>& v) {
    vector<int> result;
    for (int x : v) {
        result.push_back(x * 2);
    }
    return result;
}

// 函数式 Map
template<typename T, typename F>
vector<T> map_vec(const vector<T>& v, F func) {
    vector<T> result;
    result.reserve(v.size());
    for (const auto& x : v) {
        result.push_back(func(x));
    }
    return result;
}

int main() {
    vector<int> nums = {1, 2, 3, 4, 5};
    
    // 使用 transform（C++ 算法）
    vector<int> doubled;
    transform(nums.begin(), nums.end(), back_inserter(doubled),
              [](int x){ return x * 2; });
    
    // 使用自定义 map
    auto result = map_vec(nums, [](int x){ return x * 2; });
    
    for (int x : result) cout << x << " ";  // 2 4 6 8 10
}
```

### Filter

筛选满足条件的元素：

```cpp
template<typename T, typename F>
vector<T> filter_vec(const vector<T>& v, F predicate) {
    vector<T> result;
    for (const auto& x : v) {
        if (predicate(x)) {
            result.push_back(x);
        }
    }
    return result;
}

int main() {
    vector<int> nums = {1, 2, 3, 4, 5, 6};
    
    // 保留偶数
    auto evens = filter_vec(nums, [](int x){ return x % 2 == 0; });
    
    // 保留大于 3 的
    auto large = filter_vec(nums, [](int x){ return x > 3; });
}
```

### Reduce（折叠）

将集合归约为单一值：

```cpp
template<typename T, typename F>
T reduce(const vector<T>& v, T init, F reducer) {
    T acc = init;
    for (const auto& x : v) {
        acc = reducer(acc, x);
    }
    return acc;
}

int main() {
    vector<int> nums = {1, 2, 3, 4, 5};
    
    // 求和
    auto sum = reduce(nums, 0, [](int acc, int x){ return acc + x; });
    
    // 求积
    auto product = reduce(nums, 1, [](int acc, int x){ return acc * x; });
    
    // 连接字符串
    vector<string> words = {"Hello", " ", "World"};
    auto sentence = reduce(words, string(""), 
                           [](string acc, string s){ return acc + s; });
}
```

### 链式调用

组合 map/filter/reduce：

```cpp
vector<int> nums = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

// 计算 1-10 中偶数的平方和
auto result = reduce(
    filter_vec(
        map_vec(nums, [](int x){ return x * x; }),
        [](int x){ return x % 2 == 0; }
    ),
    0,
    [](int acc, int x){ return acc + x; }
);
// 结果：2² + 4² + 6² + 8² + 10² = 4 + 16 + 36 + 64 + 100 = 220
```

---

## 递归与迭代

函数式编程偏好**递归**而非循环，因为递归更符合数学定义，且易于组合。

### 尾递归

普通递归需要 O(n) 调用栈，尾递归可被优化为 O(1)：

```cpp
// 普通递归：阶乘
long long factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);  // 非尾递归，需保存调用栈
}

// 尾递归版本
long long factorial_tail(int n, long long acc = 1) {
    if (n <= 1) return acc;
    return factorial_tail(n - 1, n * acc);  // 尾递归
}

// 迭代版本（函数式风格）
long long factorial_iter(int n) {
    long long result = 1;
    for (int i = 2; i <= n; i++) {
        result *= i;
    }
    return result;
}
```

### 列表的函数式处理

```cpp
#include <vector>
#include <optional>
using namespace std;

// 递归计算列表长度
int length(const vector<int>& v, int idx = 0) {
    if (idx >= (int)v.size()) return 0;
    return 1 + length(v, idx + 1);
}

// 递归求和
int sum(const vector<int>& v, int idx = 0) {
    if (idx >= (int)v.size()) return 0;
    return v[idx] + sum(v, idx + 1);
}

// 递归查找
optional<int> find(const vector<int>& v, int target, int idx = 0) {
    if (idx >= (int)v.size()) return nullopt;
    if (v[idx] == target) return idx;
    return find(v, target, idx + 1);
}
```

---

## 柯里化与偏应用

### 柯里化（Currying）

将多参数函数转化为一系列单参数函数：

```cpp
// 普通函数：两数相加
int add(int a, int b) {
    return a + b;
}

// 柯里化版本
function<int(int)> curried_add(int a) {
    return [a](int b) {
        return a + b;
    };
}

// 多参数柯里化
auto curry = [](auto f) {
    return [f](auto a) {
        return [f, a](auto b) {
            return f(a, b);
        };
    };
};

int main() {
    auto add5 = curried_add(5);
    cout << add5(3) << endl;  // 8
    
    // 使用 auto curry
    auto add2 = curry([](int a, int b){ return a + b; })(2);
    cout << add2(10) << endl;  // 12
}
```

### 偏应用（Partial Application）

固定部分参数，返回剩余参数的函数：

```cpp
#include <functional>
using namespace std;

int multiply(int a, int b) {
    return a * b;
}

int main() {
    // 使用 bind 偏应用
    auto double_it = bind(multiply, placeholders::_1, 2);
    cout << double_it(5) << endl;  // 10
    
    // 使用 lambda
    auto triple_it = [](int x){ return multiply(3, x); };
    cout << triple_it(5) << endl;  // 15
}
```

---

## 惰性求值

惰性求值（Lazy Evaluation）延迟到需要时才计算，避免不必要的计算和无穷数据结构。

### C++ 惰性求值示例

```cpp
#include <functional>
#include <iostream>
using namespace std;

// 惰性求值的 Optional
template<typename T>
class Lazy {
    function<T()> func;
    mutable bool evaluated;
    mutable T value;
public:
    Lazy(function<T()> f) : func(f), evaluated(false), value() {}
    
    T get() const {
        if (!evaluated) {
            value = func();
            evaluated = true;
        }
        return value;
    }
};

int expensive_compute() {
    cout << "Computing..." << endl;
    return 42;
}

int main() {
    Lazy<int> lazy_val([](){ return expensive_compute(); });
    
    cout << "Before get()" << endl;
    cout << lazy_val.get() << endl;  // 输出 "Computing..." 和 42
    cout << lazy_val.get() << endl;  // 只输出 42（已缓存）
}
```

### 生成无穷序列

```cpp
#include <functional>
#include <iostream>
using namespace std;

// 惰性生成器
class Generator {
    function<int()> next;
public:
    Generator(function<int()> f) : next(f) {}
    int operator()() { return next(); }
};

// 自然数生成器
Generator naturals(int start = 0) {
    return Generator([start]() mutable {
        return start++;
    });
}

// 斐波那契生成器
Generator fibonacci() {
    int a = 0, b = 1;
    return Generator([a, b]() mutable {
        int result = a;
        int next = a + b;
        a = b;
        b = next;
        return result;
    });
}

int main() {
    auto naturals_gen = naturals(1);
    for (int i = 0; i < 5; i++) {
        cout << naturals_gen() << " ";  // 1 2 3 4 5
    }
    cout << endl;
    
    auto fib_gen = fibonacci();
    for (int i = 0; i < 10; i++) {
        cout << fib_gen() << " ";  // 0 1 1 2 3 5 8 13 21 34
    }
}
```

---

## 函数式 vs 面向对象

| 特性 | 函数式 | 面向对象 |
|------|--------|----------|
| **核心单元** | 函数 | 对象/类 |
| **数据状态** | 不可变 | 可变 |
| **控制流** | 递归 + 高阶函数 | 循环 + 方法调用 |
| **并发** | 天然安全（无共享状态） | 需要锁/同步 |
| **推理方式** | 数学证明 | 对象建模 |
| **适用场景** | 数据转换、并行计算 | 模拟现实实体 |

**混合范式**：现代 C++、Python、Java 支持多种范式，根据场景选择。

```cpp
// C++ 混合风格示例
class DataProcessor {
public:
    // OOP：封装
    DataProcessor(const vector<int>& data) : data_(data) {}
    
    // 函数式：链式转换
    vector<int> process() const {
        return filter_vec(
            map_vec(data_, [](int x){ return x * 2; }),
            [](int x){ return x > 10; }
        );
    }
    
private:
    vector<int> data_;  // 可变状态
};
```

---

## 参考资料

[^1]: 《Introduction to Functional Programming》 - Paul H. Hudak

[^2]: 《Functional Programming in C++》 - Ivan Čukić

[^3]: [Wikipedia: Functional Programming](https://en.wikipedia.org/wiki/Functional_programming)

---

## 相关主题

- [[../data-structure/linked-list|链表]] — 递归数据结构
- [[../algorithms/binary-search|二分查找]] — 纯函数示例
- [[./automata-and-complexity|计算理论]] — 函数式计算模型的理论基础
