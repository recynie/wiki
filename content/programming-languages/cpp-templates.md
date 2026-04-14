---
title: C++模板元编程
date: 2026-04-11
description: C++模板元编程完全指南：模板基础、特化、SFINAE、可变参数模板及设计模式应用
tags:
  - cpp
  - templates
  - metaprogramming
draft: false
permalink:
---

## 概述

模板是C++中最强大的特性之一，它使**泛型编程**成为可能。模板元编程（Template Metaprogramming）是一种在**编译期**执行计算的技术，被誉为"编译期的Lisp"。[^1]

## 模板基础

### 函数模板

函数模板定义了一族函数的通用形式，编译器会根据实参类型自动推导或实例化具体版本。

```cpp
#include <bits/stdc++.h>
using namespace std;

// 交换函数模板
template<typename T>
void swap(T& a, T& b) {
    T temp = a;
    a = b;
    b = temp;
}

// 多参数函数模板
template<typename T1, typename T2>
pair<T1, T2> make_pair_custom(T1 a, T2 b) {
    return {a, b};
}

// 模板参数推导
int main() {
    int x = 1, y = 2;
    swap(x, y);  // 推导为 swap<int>
    cout << "x=" << x << ", y=" << y << endl;  // x=2, y=1

    double a = 1.5, b = 2.5;
    swap(a, b);  // 推导为 swap<double>
    cout << "a=" << a << ", b=" << b << endl;  // a=2.5, b=1.5

    auto p = make_pair_custom(1, "hello");
    cout << p.first << " " << p.second << endl;  // 1 hello
}
```

### 类模板

类模板用于定义通用的类结构，STL中的容器都是类模板的典型应用。

```cpp
#include <bits/stdc++.h>
using namespace std;

// 简单的Stack模板类
template<typename T, int MAX_SIZE = 100>
class Stack {
private:
    T data[MAX_SIZE];
    int top_idx;
public:
    Stack() : top_idx(-1) {}

    void push(const T& val) {
        if (top_idx >= MAX_SIZE - 1) {
            throw runtime_error("Stack overflow");
        }
        data[++top_idx] = val;
    }

    T pop() {
        if (top_idx < 0) {
            throw runtime_error("Stack underflow");
        }
        return data[top_idx--];
    }

    bool empty() const { return top_idx < 0; }
    int size() const { return top_idx + 1; }
};

int main() {
    Stack<int> int_st;
    int_st.push(1);
    int_st.push(2);
    cout << int_st.pop() << endl;  // 2

    Stack<string, 50> str_st;
    str_st.push("Hello");
    str_st.push("World");
    cout << str_st.pop() << endl;  // World
}
```

### 模板参数

模板参数分为**类型参数**和**非类型参数**两种。

```cpp
#include <bits/stdc++.h>
using namespace std;

// 模板类型参数
template<typename T>
T max_value(T a, T b) {
    return a > b ? a : b;
}

// 非类型参数（整型、指针等）
template<int N>
struct Array {
    int data[N];
    int size() const { return N; }
};

// 指针非类型参数
template<const char* msg>
void print_message() {
    cout << msg << endl;
}

char g_msg[] = "Global message";  // 注意：必须是外部链接的变量

int main() {
    cout << max_value(3, 5) << endl;         // 5
    cout << max_value(3.14, 2.71) << endl;     // 3.14

    Array<10> arr;
    cout << arr.size() << endl;  // 10

    print_message<g_msg>();  // Global message
}
```

## 模板特化

当通用模板不能完全满足特定类型需求时，可提供特化版本。

### 全特化

为特定类型提供完全自定义的实现：

```cpp
#include <bits/stdc++.h>
using namespace std;

template<typename T>
T multiply(T a, T b) {
    return a * b;
}

// 全特化：针对 bool 的特殊实现
template<>
bool multiply<bool>(bool a, bool b) {
    return a && b;  // 布尔乘法用逻辑与代替
}

int main() {
    cout << multiply(3, 4) << endl;              // 12
    cout << multiply(true, true) << endl;         // 1 (true)
    cout << multiply<bool>(true, false) << endl;  // 0 (false)
}
```

### 偏特化（部分特化）

仅限定模板参数的一部分进行特化：

```cpp
#include <bits/stdc++.h>
using namespace std;

// 通用模板
template<typename T1, typename T2>
struct Pair {
    void info() { cout << "通用版本" << endl; }
};

// 偏特化：第一个参数为 int
template<typename T2>
struct Pair<int, T2> {
    void info() { cout << "第一参数为int" << endl; }
};

// 偏特化：两个参数相同
template<typename T>
struct Pair<T, T> {
    void info() { cout << "两参数相同" << endl; }
};

// 偏特化：两个参数都是指针
template<typename T1, typename T2>
struct Pair<T1*, T2*> {
    void info() { cout << "两个指针类型" << endl; }
};

int main() {
    Pair<double, string>().info();   // 通用版本
    Pair<int, double>().info();      // 第一参数为int
    Pair<string, string>().info();   // 两参数相同
    Pair<int*, double*>().info();    // 两个指针类型
}
```

## SFINAE

**SFINAE**（Substitution Failure Is Not An Error）是模板元编程的核心原理：当模板参数替换失败时，编译器不会报错，而是尝试匹配其他重载。

### 原理

```cpp
#include <bits/stdc++.h>
using namespace std;

// 通用版本
template<typename T>
void process(T val) {
    cout << "通用版本: " << val << endl;
}

// 仅当 T 是整数类型时匹配（通过返回类型SFINAE）
template<typename T>
typename enable_if<is_integral<T>::value, void>::type
process_integral(T val) {
    cout << "整数类型: " << val << endl;
}

int main() {
    process(42);          // 通用版本
    process(3.14);        // 通用版本
    process_integral(42); // 整数类型: 42
    // process_integral(3.14);  // 编译错误：SFINAE淘汰
}
```

### std::enable_if

`std::enable_if`是最常用的SFINAE工具：

```cpp
#include <bits/stdc++.h>
using namespace std;

// 条件启用模板
template<typename T>
typename enable_if<sizeof(T) >= 4, T>::type
half(T val) {
    return val / 2;
}

template<typename T>
typename enable_if<sizeof(T) < 4, T>::type
half(T val) {
    // 小类型直接返回原值（避免溢出）
    return val;
}

// 使用enable_if_t简化语法（C++14+）
template<typename T>
enable_if_t<is_floating_point<T>::value, string>
to_string(T val) {
    return to_string(val);
}

int main() {
    cout << half(100) << endl;          // 50
    cout << half((char)200) << endl;    // 200（避免char溢出）
}
```

### Type Traits

类型萃取库`<type_traits>`提供了编译期类型检查能力：

```cpp
#include <bits/stdc++.h>
using namespace std;

template<typename T>
void debug_type(T) {
    if constexpr (is_pointer<T>::value) {
        cout << "是指针类型" << endl;
    } else if constexpr (is_reference<T>::value) {
        cout << "是引用类型" << endl;
    } else if constexpr (is_array<T>::value) {
        cout << "是数组类型" << endl;
    } else {
        cout << "是普通类型" << endl;
    }
}

// 编译期类型选择
template<typename T>
typename conditional<is_integral<T>::value, int64_t, double>::type
compute(T val) {
    return static_cast<conditional_t<is_integral<T>::value, int64_t, double>>(val);
}

int main() {
    int x = 42;
    debug_type(x);              // 是普通类型
    debug_type(&x);             // 是指针类型
    debug_type(static_cast<int&>(x));  // 是引用类型

    cout << compute(42) << endl;      // 42
    cout << compute(3.14) << endl;    // 3.14
}
```

## 模板元编程

模板元编程在编译期执行计算，实现算法和数据结构的静态版本。

### 编译期计算

```cpp
#include <bits/stdc++.h>
using namespace std;

// 编译期计算阶乘
template<int N>
struct Factorial {
    static const int value = N * Factorial<N-1>::value;
};

// 递归终止特化
template<>
struct Factorial<0> {
    static const int value = 1;
};

// 编译期计算斐波那契
template<int N>
struct Fibonacci {
    static const int value = Fibonacci<N-1>::value + Fibonacci<N-2>::value;
};

template<>
struct Fibonacci<0> { static const int value = 0; };

template<>
struct Fibonacci<1> { static const int value = 1; };

// 编译期判断素数
template<int N, int D>
struct IsPrime {
    static const bool value = (N % D != 0) && IsPrime<N, D-1>::value;
};

template<int N>
struct IsPrime<N, 1> {
    static const bool value = true;
};

int main() {
    cout << "5! = " << Factorial<5>::value << endl;    // 120
    cout << "10! = " << Factorial<10>::value << endl;   // 3628800

    cout << "F(10) = " << Fibonacci<10>::value << endl;  // 55
    cout << "F(20) = " << Fibonacci<20>::value << endl; // 6765

    cout << "17是素数? " << boolalpha << IsPrime<17, 16>::value << endl;  // true
    cout << "15是素数? " << IsPrime<15, 14>::value << endl;  // false
}
```

### 递归模板

递归模板是实现编译期循环和复杂计算的基础：

```cpp
#include <bits/stdc++.h>
using namespace std;

// 编译期遍历整型序列
template<int... nums>
struct IntSeq;

// 展开整数序列
template<int N, int... nums>
struct GenerateSeq : GenerateSeq<N-1, N-1, nums...> {};

template<int... nums>
struct GenerateSeq<0, nums...> {
    using type = IntSeq<0, nums...>;
};

// 编译期打印（通过递归继承）
template<int N>
struct PrintSeq {
    template<typename Seq>
    static void print() {
        cout << N << " ";
        PrintSeq<N-1>::print();
    }
};

template<>
struct PrintSeq<0> {
    template<typename Seq>
    static void print() {
        cout << 0 << " ";
    }
};

// 编译期求和
template<int... nums>
struct Sum;

template<int head, int... tail>
struct Sum<head, tail...> {
    static const int value = head + Sum<tail...>::value;
};

template<>
struct Sum<> {
    static const int value = 0;
};

int main() {
    using Seq = typename GenerateSeq<10>::type;

    // 手动打印序列
    int arr[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    for (int i = 0; i <= 10; i++) cout << arr[i] << " ";  // 0 1 2 ... 10
    cout << endl;

    cout << "Sum<1,2,3,4,5>: " << Sum<1,2,3,4,5>::value << endl;  // 15
    cout << "Sum<1,2,3,4,5,6,7,8,9,10>: " << Sum<1,2,3,4,5,6,7,8,9,10>::value << endl;  // 55
}
```

### 类型萃取（Type Traits）

类型萃取用于在编译期查询和转换类型：

```cpp
#include <bits/stdc++.h>
using namespace std;

// remove_reference 实现
template<typename T>
struct RemoveRef {
    using type = T;
};

template<typename T>
struct RemoveRef<T&> {
    using type = T;
};

template<typename T>
struct RemoveRef<T&&> {
    using type = T;
};

template<typename T>
using RemoveRef_t = typename RemoveRef<T>::type;

// decay 实现（更完整的类型处理）
template<typename T>
struct Decay {
    using type = T;
};

template<typename T>
struct Decay<T&> {
    using type = typename decay<T>::type;
};

template<typename T>
struct Decay<T&&> {
    using type = typename decay<T>::type;
};

// 编译期类型相等判断
template<typename T, typename U>
struct IsSame {
    static constexpr bool value = false;
};

template<typename T>
struct IsSame<T, T> {
    static constexpr bool value = true;
};

int main() {
    int x = 42;
    int& ref_x = x;

    RemoveRef_t<int&> y = ref_x;  // y 是 int，不是 int&
    static_assert(is_same<RemoveRef_t<int&>, int>::value, "");
    static_assert(is_same<RemoveRef_t<int&&>, int>::value, "");

    cout << "IsSame<int, int>: " << IsSame<int, int>::value << endl;  // 1
    cout << "IsSame<int, float>: " << IsSame<int, float>::value << endl;  // 0
}
```

## 可变参数模板

C++11引入的可变参数模板支持任意数量的模板参数。

### 基础语法

```cpp
#include <bits/stdc++.h>
using namespace std;

// 可变参数函数模板
template<typename... Args>
void print_all(Args... args) {
    // 展开参数包
    (cout << ... << args) << endl;
}

// 递归终止
void print_all() {
    cout << endl;
}

// 递归展开打印
template<typename T, typename... Args>
void print_recursive(T first, Args... rest) {
    cout << first << " ";
    print_recursive(rest...);
}

// 参数包大小
template<typename... Args>
constexpr size_t count_args(Args... args) {
    return sizeof...(args);
}

int main() {
    print_all(1, 2, 3, "hello", 4.5);        // 12hello4.5
    print_all(1, 2, 3);                     // 123

    print_recursive(1, "hello", 3.14);      // 1 hello 3.14
    print_recursive("end");                 // end

    cout << count_args(1, 2, 3) << endl;    // 3
    cout << count_args(1, "a", 3.14, 'c') << endl;  // 4
}
```

### 展开模式

参数包可以通过多种方式展开：

```cpp
#include <bits/stdc++.h>
using namespace std;

// 展开为初始化列表
template<typename T, typename... Args>
vector<T> make_vector(T first, Args... rest) {
    return {first, rest...};
}

// 展开为函数参数
void handler(int a, int b, int c) {
    cout << "handler: " << a << "," << b << "," << c << endl;
}

template<typename... Args>
void forward_to_handler(Args... args) {
    handler(args...);  // 展开为 handler(1, 2, 3)
}

// 索引展开（通过整数序列）
template<size_t... idx>
void print_indexed_impl(const tuple<int, string, double>& t, index_sequence<idx...>) {
    cout << get<idx>(t) << " "...;
}

template<typename... Args>
void print_indexed(tuple<Args...> t) {
    print_indexed_impl(t, index_sequence_for<Args...>{});
}

// 条件展开
template<typename... Args>
bool all_true(Args... args) {
    return (... && args);  // C++17 折叠表达式
}

template<typename T, typename... Args>
bool is_all_same(Args... args) {
    return (... && is_same_v<T, Args>);
}

int main() {
    // 初始化列表展开
    auto v = make_vector(1, 2, 3);
    for (int x : v) cout << x << " ";  // 1 2 3
    cout << endl;

    // 函数参数展开
    forward_to_handler(1, 2, 3);  // handler: 1,2,3

    // 索引展开
    print_indexed(make_tuple(42, "hello", 3.14));  // 42 hello 3.14
    cout << endl;

    // 折叠表达式（C++17）
    cout << boolalpha << all_true(true, true, true) << endl;  // true
    cout << all_true(true, false, true) << endl;               // false
    cout << is_all_same<int>(1, 2, 3) << endl;                 // false
    cout << is_all_same<int>(1, 2, 3) << endl;                 // false
}
```

## 模板与设计模式

模板为许多设计模式提供了高效的C++实现方式。

### 策略模式

策略模式通过模板消除运行时多态的开销：

```cpp
#include <bits/stdc++.h>
using namespace std;

// 策略接口（编译期多态）
template<typename Strategy>
class Context {
private:
    Strategy strategy;
public:
    explicit Context(Strategy s) : strategy(s) {}

    void execute(int data) {
        strategy.process(data);
    }
};

// 具体策略
struct AddStrategy {
    void process(int data) {
        cout << "加法策略: " << data << " + 10 = " << (data + 10) << endl;
    }
};

struct MulStrategy {
    void process(int data) {
        cout << "乘法策略: " << data << " * 10 = " << (data * 10) << endl;
    }
};

// 更灵活的策略（带状态）
template<int Delta>
struct OffsetStrategy {
    void process(int data) {
        cout << "偏移策略: " << data << " + " << Delta << " = " << (data + Delta) << endl;
    }
};

int main() {
    Context<AddStrategy> ctx1(AddStrategy{});
    ctx1.execute(5);  // 加法策略: 5 + 10 = 15

    Context<MulStrategy> ctx2(MulStrategy{});
    ctx2.execute(5);  // 乘法策略: 5 * 10 = 50

    // 编译期常量作为策略参数
    Context<OffsetStrategy<100>> ctx3(OffsetStrategy<100>{});
    ctx3.execute(5);  // 偏移策略: 5 + 100 = 105
}
```

### 工厂模式

模板工厂模式实现编译期对象创建注册：

```cpp
#include <bits/stdc++.h>
using namespace std;

// 产品基类
class Product {
public:
    virtual ~Product() = default;
    virtual string name() const = 0;
};

// 具体产品
class ConcreteProductA : public Product {
public:
    string name() const override { return "ProductA"; }
};

class ConcreteProductB : public Product {
public:
    string name() const override { return "ProductB"; }
};

// 产品注册器模板
template<typename ProductType>
class ProductRegistry {
public:
    static Product* create() {
        return new ProductType();
    }
};

// 简单工厂模板
class SimpleFactory {
public:
    template<typename ProductType>
    unique_ptr<Product> create() {
        return unique_ptr<Product>(new ProductType());
    }
};

// 注册式工厂（通过宏）
#define REGISTER_PRODUCT(Factory, ProductType, key) \
    namespace { \
        struct Registar##ProductType { \
            Registar##ProductType() { \
                Factory::instance().register_product(key, \
                    []() -> Product* { return new ProductType(); }); \
            } \
        }; \
        static Registar##ProductType reg_##ProductType; \
    }

class DynamicFactory {
    unordered_map<string, function<Product*()>> creators;
public:
    static DynamicFactory& instance() {
        static DynamicFactory f;
        return f;
    }

    void register_product(const string& key, function<Product*()> creator) {
        creators[key] = creator;
    }

    unique_ptr<Product> create(const string& key) {
        auto it = creators.find(key);
        if (it != creators.end()) {
            return unique_ptr<Product>(it->second());
        }
        return nullptr;
    }
};

// 使用宏注册产品
REGISTER_PRODUCT(DynamicFactory, ConcreteProductA, "A")
REGISTER_PRODUCT(DynamicFactory, ConcreteProductB, "B")

int main() {
    // 简单工厂
    SimpleFactory sf;
    auto p1 = sf.create<ConcreteProductA>();
    cout << p1->name() << endl;  // ProductA

    // 动态工厂
    auto p2 = DynamicFactory::instance().create("A");
    auto p3 = DynamicFactory::instance().create("B");
    cout << p2->name() << endl;  // ProductA
    cout << p3->name() << endl;  // ProductB
}
```

## 最佳实践

1. **优先使用类型参数**：非类型模板参数仅支持整型、指针等有限类型
2. **善用SFINAE**：`enable_if`和`type_traits`是控制模板匹配的核心工具
3. **编译期vs运行时**：将能确定的工作移到编译期，减少运行时开销
4. **可变参数模板优先**：C++11后优先使用可变参数模板代替`...`风格的模板参数
5. **C++17+特性**：善用`if constexpr`和折叠表达式简化元编程代码

## 参考

[^1]: 本段参考[C++ Templates: The Complete Guide](https:// informit.com/store/c-plus-plus-templates-the-complete-guide-9780321412999)
[^2]: 本段参考[Modern TMP techniques](https://eli.thegreenplace.net/2020/cmake-vs-make-the-compiler-of-choices/)
