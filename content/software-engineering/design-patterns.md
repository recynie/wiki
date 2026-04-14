---
title: 设计模式（Design Patterns）
date: 2026-04-06
description: 软件工程中常用的设计模式总结，包括创建型、结构型和行为型模式，以及 SOLID 设计原则。
tags:
  - design-patterns
  - software-engineering
  - oop
draft: false
permalink:
---

## 概述

设计模式是软件工程中解决常见问题的**可复用解决方案**。它们不是具体的代码实现，而是经过验证的编程思想和方法论。

### 历史背景

1994 年，四位作者 Erich Gamma、Richard Helm、Ralph Johnson、John Vlissides（被称为「GoF」，Gang of Four）在《设计模式：可复用面向对象软件的基础》一书中系统化了 23 种设计模式，标志着设计模式研究的开端。[^1]

### 模式分类

设计模式主要分为三大类：

| 类别 | 描述 | 代表模式 |
|------|------|----------|
| **创建型** | 对象的创建机制 | 单例、工厂、建造者、原型 |
| **结构型** | 对象组合成更大结构 | 适配器、装饰器、代理、组合、外观 |
| **行为型** | 对象间的职责分配 | 观察者、策略、模板方法、迭代器、责任链 |

---

## 创建型模式（Creational Patterns）

创建型模式专注于对象创建的细节，封装「如何创建」的知识，简化系统的对象创建过程。

### 单例模式（Singleton）

#### 问题背景

在系统中某些类只需要**唯一实例**，例如配置管理器、数据库连接池、日志记录器等。如果允许创建多个实例，可能导致资源冲突或状态不一致。

#### 解决方案

确保一个类只有一个实例，并提供一个全局访问点。

#### C++ 实现

```cpp
#include <mutex>

// 懒汉式（线程安全，双重检查锁定）
class SingletonLazy {
private:
    static SingletonLazy* instance;
    static std::mutex mtx;
    
    SingletonLazy() = default;
    SingletonLazy(const SingletonLazy&) = delete;
    SingletonLazy& operator=(const SingletonLazy&) = delete;

public:
    static SingletonLazy* getInstance() {
        if (instance == nullptr) {  // 第一次检查
            std::lock_guard<std::mutex> lock(mtx);
            if (instance == nullptr) {  // 第二次检查
                instance = new SingletonLazy();
            }
        }
        return instance;
    }
};

// 饿汉式（线程安全，类加载时即创建）
class SingletonHungry {
private:
    static SingletonHungry* instance;
    
    SingletonHungry() = default;

public:
    static SingletonHungry* getInstance() {
        return instance;
    }
};

SingletonHungry* SingletonHungry::instance = new SingletonHungry();
```

#### 适用场景

- 配置管理类（全局唯一配置）
- 日志记录器（统一日志输出）
- 数据库连接池（资源复用）

---

### 工厂模式（Factory）

#### 问题背景

直接使用 `new` 创建对象会导致**对象创建与使用的高度耦合**。当对象创建逻辑复杂或需要替换实现类时，修改代码的代价很高。

#### 解决方案

将对象创建封装到工厂类中，客户端通过工厂获取对象而不直接实例化。

#### C++ 实现

**简单工厂**：

```cpp
class Product {
public:
    virtual void operation() = 0;
    virtual ~Product() = default;
};

class ConcreteProductA : public Product {
public:
    void operation() override { /* 产品A操作 */ }
};

class ConcreteProductB : public Product {
public:
    void operation() override { /* 产品B操作 */ }
};

class SimpleFactory {
public:
    static Product* create(const std::string& type) {
        if (type == "A") return new ConcreteProductA();
        if (type == "B") return new ConcreteProductB();
        return nullptr;
    }
};
```

**工厂方法**：

```cpp
class Product {
public:
    virtual void operation() = 0;
    virtual ~Product() = default;
};

class Factory {
public:
    virtual Product* createProduct() = 0;
    virtual ~Factory() = default;
};

class ProductA : public Product {
public:
    void operation() override { /* 产品A操作 */ }
};

class FactoryA : public Factory {
public:
    Product* createProduct() override { return new ProductA(); }
};
```

**抽象工厂**：

```cpp
class AbstractProductA { public: virtual void operationA() = 0; };
class AbstractProductB { public: virtual void operationB() = 0; };

class ProductA1 : public AbstractProductA { /* 产品A1实现 */ };
class ProductB1 : public AbstractProductB { /* 产品B1实现 */ };

class AbstractFactory {
public:
    virtual AbstractProductA* createProductA() = 0;
    virtual AbstractProductB* createProductB() = 0;
    virtual ~AbstractFactory() = default;
};

class ConcreteFactory1 : public AbstractFactory {
public:
    AbstractProductA* createProductA() override { return new ProductA1(); }
    AbstractProductB* createProductB() override { return new ProductB1(); }
};
```

#### 适用场景

- 对象创建逻辑可能变化
- 需要在不修改客户端代码的情况下替换产品类
- 系统需要支持多种产品族

---

### 建造者模式（Builder）

#### 问题背景

当一个类的构造函数参数过多（超过 4-5 个），且参数组合不同时，直接使用构造函数会导致**构造函数膨胀**和**参数歧义**问题。

#### 解决方案

将对象的构建过程分解为多个步骤，通过指挥者（Director）控制构建流程，建造者（Builder）负责各部分的构建。

#### C++ 实现

```cpp
class Product {
private:
    std::string partA_;
    std::string partB_;
    std::string partC_;
    
public:
    void setPartA(const std::string& a) { partA_ = a; }
    void setPartB(const std::string& b) { partB_ = b; }
    void setPartC(const std::string& c) { partC_ = c; }
};

class Builder {
public:
    virtual void buildPartA() = 0;
    virtual void buildPartB() = 0;
    virtual void buildPartC() = 0;
    virtual Product getResult() = 0;
    virtual ~Builder() = default;
};

class ConcreteBuilder : public Builder {
private:
    Product product;
    
public:
    void buildPartA() override { product.setPartA("A"); }
    void buildPartB() override { product.setPartB("B"); }
    void buildPartC() override { product.setPartC("C"); }
    Product getResult() override { return product; }
};

class Director {
public:
    void construct(Builder& builder) {
        builder.buildPartA();
        builder.buildPartB();
        builder.buildPartC();
    }
};
```

#### 适用场景

- 构造函数参数过多或可选参数多
- 对象创建过程需要灵活组合
- 需要创建不同表示的对象（如不同配置的电脑）

---

### 原型模式（Prototype）

#### 问题背景

创建对象成本较高（如从数据库加载），直接 `new` 会造成性能浪费。

#### 解决方案

通过复制（克隆）已有对象来创建新对象，无需知道对象的具体类型。

#### C++ 实现

```cpp
class Prototype {
public:
    virtual Prototype* clone() const = 0;
    virtual ~Prototype() = default;
};

class ConcretePrototype : public Prototype {
private:
    int data_;
    
public:
    ConcretePrototype(int data) : data_(data) {}
    ConcretePrototype(const ConcretePrototype& other) : data_(other.data_) {}
    
    Prototype* clone() const override {
        return new ConcretePrototype(*this);
    }
};

// 使用
ConcretePrototype prototype(42);
ConcretePrototype* clone = static_cast<ConcretePrototype*>(prototype.clone());
```

#### 适用场景

- 对象创建成本高（如复杂配置、数据库查询）
- 需要避免对象创建的多样性子类膨胀
- 对象状态相同或相似的高频创建场景

---

## 结构型模式（Structural Patterns）

结构型模式描述如何组合对象以形成更大的结构，处理对象之间的组合关系。

### 适配器模式（Adapter）

#### 问题背景

现有代码的接口与系统要求不兼容，直接修改代码可能影响其他使用者。

#### 解决方案

将一个类的接口转换成客户端期望的另一个接口，使不兼容的类能够合作。

#### C++ 实现

```cpp
// 目标接口（客户端期望）
class Target {
public:
    virtual void request() = 0;
    virtual ~Target() = default;
};

// 需要适配的类（被适配者）
class Adaptee {
public:
    void specificRequest() { /* 特殊操作 */ }
};

// 类适配器（通过继承）
class ClassAdapter : public Target, private Adaptee {
public:
    void request() override { specificRequest(); }
};

// 对象适配器（通过组合）
class ObjectAdapter : public Target {
private:
    Adaptee* adaptee_;
    
public:
    ObjectAdapter(Adaptee* adaptee) : adaptee_(adaptee) {}
    void request() override { adaptee_->specificRequest(); }
};
```

#### 适用场景

- 集成第三方库且接口不兼容
- 重用现有类但接口不匹配
- 统一多个相似类的接口

---

### 装饰器模式（Decorator）

#### 问题背景

需要在运行时**动态添加功能**，但继承会导致类数量爆炸且静态绑定。

#### 解决方案

将对象包装在装饰器中，装饰器在调用原对象方法的同时添加新功能。

#### C++ 实现

```cpp
class Component {
public:
    virtual void operation() = 0;
    virtual ~Component() = default;
};

class ConcreteComponent : public Component {
public:
    void operation() override { /* 基础操作 */ }
};

class Decorator : public Component {
protected:
    Component* component_;
    
public:
    Decorator(Component* c) : component_(c) {}
    virtual void operation() override { component_->operation(); }
};

class ConcreteDecoratorA : public Decorator {
public:
    ConcreteDecoratorA(Component* c) : Decorator(c) {}
    void operation() override {
        Decorator::operation();
        addedBehaviorA();  // 添加新行为
    }
    
private:
    void addedBehaviorA() { /* 额外功能A */ }
};
```

#### 适用场景

- 动态添加职责且需要可撤销
- 继承会导致类爆炸
- 需要透明地为对象添加功能（客户端无感知）

---

### 代理模式（Proxy）

#### 问题背景

直接访问对象可能带来问题：访问控制、延迟加载、性能优化等。

#### 解决方案

通过代理对象控制对原对象的访问，在访问前后添加额外逻辑。

#### C++ 实现

```cpp
class Subject {
public:
    virtual void request() = 0;
    virtual ~Subject() = default;
};

class RealSubject : public Subject {
public:
    void request() override { /* 真实操作 */ }
};

class Proxy : public Subject {
private:
    RealSubject* realSubject_;
    
public:
    void request() override {
        if (!realSubject_) {
            realSubject_ = new RealSubject();
        }
        // 访问控制、前置处理
        realSubject_->request();
        // 后置处理、日志记录等
    }
};
```

#### 适用场景

- **远程代理**：访问远程服务
- **虚代理**：延迟加载大对象
- **保护代理**：访问权限控制
- **智能引用**：对象访问时的额外操作

---

### 组合模式（Composite）

#### 问题背景

需要处理**树形结构**的对象层次，如文件系统、组织架构、UI 组件等。

#### 解决方案

将对象组合成树形结构，统一处理单个对象和组合对象。

#### C++ 实现

```cpp
class Component {
protected:
    std::string name_;
    
public:
    explicit Component(const std::string& name) : name_(name) {}
    virtual void operation() const = 0;
    virtual void add(Component*) {}
    virtual void remove(Component*) {}
    virtual ~Component() = default;
};

class Leaf : public Component {
public:
    using Component::Component;
    void operation() const override { /* 叶子节点操作 */ }
};

class Composite : public Component {
private:
    std::vector<Component*> children_;
    
public:
    using Component::Component;
    void operation() const override {
        for (const auto& child : children_) {
            child->operation();
        }
    }
    
    void add(Component* c) override { children_.push_back(c); }
    void remove(Component* c) override {
        children_.erase(std::remove(children_.begin(), children_.end(), c), children_.end());
    }
};
```

#### 适用场景

- 文件系统（文件和文件夹）
- GUI 组件层次
- 组织架构（部门与员工）
- 命令树结构

---

### 外观模式（Facade）

#### 问题背景

系统复杂，子系统众多，客户端需要了解多个子系统才能完成简单任务。

#### 解决方案

为复杂子系统提供一个统一的高层接口，使客户端更容易使用。

#### C++ 实现

```cpp
class SubsystemA {
public:
    void operationA() { /* 子系统A操作 */ }
};

class SubsystemB {
public:
    void operationB() { /* 子系统B操作 */ }
};

class SubsystemC {
public:
    void operationC() { /* 子系统C操作 */ }
};

class Facade {
private:
    SubsystemA* a_;
    SubsystemB* b_;
    SubsystemC* c_;
    
public:
    Facade() : a_(new SubsystemA()), b_(new SubsystemB()), c_(new SubsystemC()) {}
    ~Facade() { delete a_; delete b_; delete c_; }
    
    void operation() {
        a_->operationA();
        b_->operationB();
        c_->operationC();
    }
};
```

#### 适用场景

- 为复杂子系统提供简单接口
- 层次化结构中的入口点
- 解耦客户端与子系统

---

## 行为型模式（Behavioral Patterns）

行为型模式关注对象之间的职责分配和算法封装。

### 观察者模式（Observer）

#### 问题背景

当一个对象（Subject）状态改变时，需要**自动通知**多个依赖对象（Observers），但不想让它们紧密耦合。

#### 解决方案

定义一对多的依赖关系，当Subject状态改变时，所有Observer收到通知。

#### C++ 实现

```cpp
#include <vector>
#include <algorithm>

class Observer {
public:
    virtual void update(int state) = 0;
    virtual ~Observer() = default;
};

class Subject {
private:
    std::vector<Observer*> observers_;
    int state_;
    
public:
    void attach(Observer* o) { observers_.push_back(o); }
    void detach(Observer* o) {
        observers_.erase(std::remove(observers_.begin(), observers_.end(), o), observers_.end());
    }
    
    void setState(int state) {
        state_ = state;
        notify();
    }
    
    void notify() {
        for (auto* o : observers_) {
            o->update(state_);
        }
    }
};

class ConcreteObserver : public Observer {
private:
    int observerState_;
    
public:
    void update(int state) override { observerState_ = state; }
};
```

#### 适用场景

- GUI 事件处理
- MVC 架构中的 Model-View 同步
- 发布-订阅系统
- 消息通知系统

---

### 策略模式（Strategy）

#### 问题背景

系统中存在多种算法或行为，需要在运行时选择具体实现，且不想使用多重条件语句。

#### 解决方案

定义一系列算法，将每个算法封装起来，使它们可以互相替换。

#### C++ 实现

```cpp
class Strategy {
public:
    virtual int execute(int a, int b) = 0;
    virtual ~Strategy() = default;
};

class AddStrategy : public Strategy {
public:
    int execute(int a, int b) override { return a + b; }
};

class MultiplyStrategy : public Strategy {
public:
    int execute(int a, int b) override { return a * b; }
};

class Context {
private:
    Strategy* strategy_;
    
public:
    void setStrategy(Strategy* s) { strategy_ = s; }
    int executeStrategy(int a, int b) { return strategy_->execute(a, b); }
};
```

#### 适用场景

- 多种排序算法选择
- 支付方式选择
- 压缩算法选择
- 避免多重条件语句

---

### 模板方法模式（Template Method）

#### 问题背景

多个类有相似的方法流程，但某些步骤的具体实现不同。

#### 解决方案

定义算法骨架，将某些步骤的实现延迟到子类。

#### C++ 实现

```cpp
class AbstractClass {
public:
    // 模板方法
    void templateMethod() {
        step1();
        step2();
        step3();
    }
    
protected:
    virtual void step1() = 0;
    virtual void step2() = 0;
    virtual void step3() = 0;  // 可选，提供默认实现
    virtual ~AbstractClass() = default;
};

class ConcreteClass : public AbstractClass {
protected:
    void step1() override { /* 实现1 */ }
    void step2() override { /* 实现2 */ }
    void step3() override { /* 实现3 */ }
};
```

#### 适用场景

- 框架中的钩子方法
- 数据处理流程固定但细节不同
- 减少代码重复

---

### 迭代器模式（Iterator）

#### 问题背景

需要顺序访问集合元素，但不暴露集合的内部结构（封装性）。

#### 解决方案

将遍历逻辑封装到迭代器对象中，提供统一的访问接口。

#### C++ 实现

```cpp
template <typename T>
class Iterator {
public:
    virtual bool hasNext() = 0;
    virtual T next() = 0;
    virtual ~Iterator() = default;
};

template <typename T>
class Aggregate {
public:
    virtual Iterator<T>* createIterator() = 0;
    virtual ~Aggregate() = default;
};

template <typename T>
class ConcreteIterator : public Iterator<T> {
private:
    std::vector<T> data_;
    size_t position_ = 0;
    
public:
    explicit ConcreteIterator(const std::vector<T>& data) : data_(data) {}
    
    bool hasNext() override { return position_ < data_.size(); }
    T next() override { return data_[position_++]; }
};
```

#### 适用场景

- STL 容器的迭代器
- 数据库查询结果遍历
- 文件系统遍历
- 树结构遍历

---

### 责任链模式（Chain of Responsibility）

#### 问题背景

请求需要经过多个对象处理，每个对象决定自己处理或传递给下一个。

#### 解决方案

将请求的发送者和接收者解耦，通过链式结构传递请求。

#### C++ 实现

```cpp
class Handler {
protected:
    Handler* nextHandler_;
    
public:
    Handler() : nextHandler_(nullptr) {}
    void setNext(Handler* h) { nextHandler_ = h; }
    
    virtual void handleRequest(int request) {
        if (nextHandler_) {
            nextHandler_->handleRequest(request);
        }
    }
    virtual ~Handler() = default;
};

class ConcreteHandlerA : public Handler {
public:
    void handleRequest(int request) override {
        if (request < 10) {
            // 处理请求
        } else {
            Handler::handleRequest(request);  // 传递给下一个
        }
    }
};
```

#### 适用场景

- 审批流程（逐级审批）
- 过滤器链（Web 请求处理）
- 日志级别处理
- 异常处理链

---

## 设计原则（SOLID）

SOLID 是面向对象设计的五个基本原则，有助于创建可维护、可扩展的软件系统。

### 单一职责原则（SRP）

> 一个类应该只有一个引起它变化的原因。

```cpp
// 违反 SRP：同时负责用户数据和用户显示
class User {
public:
    std::string name;
    void save() { /* 保存到数据库 */ }
    void print() { /* 打印用户信息 */ }
};

// 符合 SRP
class User {
public:
    std::string name;
};

class UserRepository {
public:
    void save(const User& user) { /* 保存到数据库 */ }
};

class UserPrinter {
public:
    void print(const User& user) { /* 打印用户信息 */ }
};
```

---

### 开闭原则（OCP）

> 软件实体应该对扩展开放，对修改关闭。

```cpp
// 违反 OCP：新增类型需要修改原有代码
class AreaCalculator {
public:
    double area(Object shape) {
        if (type == "Circle") { /* 圆形 */ }
        if (type == "Rectangle") { /* 矩形 */ }
        // 新增形状需要修改此处
    }
};

// 符合 OCP：使用多态，扩展无需修改
class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
    double radius_;
public:
    double area() const override { return 3.14159 * radius_ * radius_; }
};

class Rectangle : public Shape {
    double width_, height_;
public:
    double area() const override { return width_ * height_; }
};
```

---

### 里氏替换原则（LSP）

> 子类对象能够替换父类对象而不影响程序正确性。

核心要求：子类必须保持父类的行为特性。

```cpp
class Bird {
public:
    virtual void fly() = 0;
    virtual ~Bird() = default;
};

class Sparrow : public Bird {
public:
    void fly() override { /* 正常飞行 */ }
};

class Penguin : public Bird {
public:
    void fly() override { /* 企鹅不会飞，违反 LSP */ }
    // 应该重新设计：Bird 包含 fly() 和 swim()，而非继承
};
```

---

### 依赖倒置原则（DIP）

> 高层模块不应该依赖低层模块，两者都应该依赖抽象。

```cpp
// 违反 DIP
class MySQL {
public:
    void query() { /* MySQL 查询 */ }
};

class UserService {
private:
    MySQL db_;  // 直接依赖低层模块
};
```

```cpp
// 符合 DIP
class Database {
public:
    virtual void query() = 0;
    virtual ~Database() = default;
};

class MySQL : public Database {
public:
    void query() override { /* MySQL 查询 */ }
};

class UserService {
private:
    Database* db_;  // 依赖抽象
public:
    UserService(Database* db) : db_(db) {}
};
```

---

### 接口隔离原则（ISP）

> 客户端不应该依赖它不需要的接口。

```cpp
// 违反 ISP：一个臃肿接口
class Machine {
public:
    virtual void print() = 0;
    virtual void scan() = 0;
    virtual void fax() = 0;
};

class SimplePrinter : public Machine {
public:
    void print() override { /* 打印 */ }
    void scan() override { /* 不需要但必须实现 */ }
    void fax() override { /* 不需要但必须实现 */ }
};
```

```cpp
// 符合 ISP：拆分为小接口
class Printer {
public:
    virtual void print() = 0;
    virtual ~Printer() = default;
};

class Scanner {
public:
    virtual void scan() = 0;
    virtual ~Scanner() = default;
};

class SimplePrinter : public Printer {
public:
    void print() override { /* 打印 */ }
};
```

---

## 总结

设计模式是软件工程中的重要知识体系，它们提供了经过验证的解决方案来应对常见的软件设计问题。

| 类别 | 核心价值 |
|------|----------|
| **创建型** | 解耦对象创建与使用 |
| **结构型** | 优化对象组合结构 |
| **行为型** | 规范对象交互职责 |

在实际应用中，应避免**过度设计**，根据具体场景选择合适的模式。设计模式不是银弹，灵活运用才能发挥其价值。

---

## 参考资料

[^1]: Gamma, E., Helm, R., Johnson, R., & Vlissides, J. (1994). Design Patterns: Elements of Reusable Object-Oriented Software. Addison-Wesley.
