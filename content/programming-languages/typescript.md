---
title: TypeScript入门指南
date: 2026-04-11
description: TypeScript基础教程，涵盖类型系统、接口、泛型等核心概念
tags:
  - typescript
  - javascript
  - frontend
draft: false
permalink:
---

## 1. TypeScript简介

TypeScript是微软开发的开源编程语言，是JavaScript的超集，在JavaScript基础上添加了静态类型检查。

### 与JavaScript关系

- TypeScript是JavaScript的超集，任何JavaScript代码都是合法的TypeScript代码
- TypeScript代码需要编译（transpile）成JavaScript才能在浏览器或Node.js中运行
- 版本同步跟进ECMAScript标准，支持最新的JavaScript特性

### 类型系统优势

| 优势 | 说明 |
|------|------|
| 类型安全 | 编译时发现类型错误，减少运行时bug |
| 代码提示 | IDE提供更准确的自动补全和类型推断 |
| 重构支持 | 改变类型或函数签名时，编译器提示所有受影响的地方 |
| 文档化 | 类型本身是最好的代码文档 |

### 编译环境

```bash
# 安装TypeScript编译器
npm install -g typescript

# 编译单个文件
tsc hello.ts

# 编译并指定输出目录
tsc --outDir dist hello.ts

# 初始化tsconfig.json配置文件
tsc --init
```

## 2. 基础类型

### 原始类型

```typescript
// 字符串
let name: string = "Alice";
let greeting: string = `Hello, ${name}`;

// 数字
let age: number = 25;
let price: number = 99.9;

// 布尔值
let isActive: boolean = true;
let isCompleted: boolean = false;
```

### 数组

```typescript
// 两种声明方式
let numbers: number[] = [1, 2, 3, 4, 5];
let names: Array<string> = ["Alice", "Bob", "Charlie"];

// 只读数组
let readonlyNumbers: ReadonlyArray<number> = [1, 2, 3];
```

### 元组

元组类型表示一个已知元素数量和类型的数组。

```typescript
let person: [string, number, boolean] = ["Alice", 25, true];

// 可选元素
let point: [number, number, number?] = [1, 2];

// 剩余元素
let list: [number, ...string[]] = [1, "a", "b", "c"];
```

### 枚举

```typescript
// 数字枚举
enum Direction {
    Up,    // 默认为0
    Down,  // 1
    Left,  // 2
    Right  // 3
}
let dir: Direction = Direction.Up;

// 字符串枚举
enum Status {
    Success = "SUCCESS",
    Error = "ERROR",
    Loading = "LOADING"
}

// 异构枚举（混合）
enum BooleanLike {
    No = 0,
    Yes = "YES"
}
```

### 特殊类型

```typescript
// any - 任意类型，绕过类型检查
let value: any = 10;
value = "string";  // OK
value.foo();       // OK，不推荐

// unknown - 未知类型，比any更安全
let unknownValue: unknown = 10;
if (typeof unknownValue === "string") {
    console.log(unknownValue.toUpperCase());  // 安全使用
}

// void - 表示没有返回值
function logMessage(message: string): void {
    console.log(message);
}

// null 和 undefined
let n: null = null;
let u: undefined = undefined;

// never - 表示永不返回的类型
function throwError(message: string): never {
    throw new Error(message);
}

function infiniteLoop(): never {
    while (true) {}
}
```

## 3. 类型注解与推断

### 显式注解

```typescript
// 变量注解
let count: number = 0;
let username: string = "alice";

// 函数参数和返回值注解
function add(a: number, b: number): number {
    return a + b;
}

// 箭头函数
const multiply = (x: number, y: number): number => x * y;
```

### 自动推断

TypeScript会根据初始值推断变量类型。

```typescript
let count = 10;      // 推断为 number
let name = "Alice";  // 推断为 string
let isActive = true; // 推断为 boolean

// 函数返回值也会被推断
function add(a: number, b: number) {
    return a + b;    // 推断返回类型为 number
}
```

### 类型断言

当TypeScript无法确定类型时，可以使用断言进行类型转换。

```typescript
// “尖括号”语法
let value: unknown = "hello";
let len: number = (<string>value).length;

// “as”语法（推荐）
let len2: number = (value as string).length;

// 非空断言（确定值不为null/undefined）
let name: string | null = getName();
console.log(name!.toUpperCase());

// 常量断言
const obj = {
    x: 10,
    y: 20
} as const;
```

## 4. 接口与类型别名

### 接口定义对象结构

```typescript
interface User {
    id: number;
    name: string;
    email?: string;           // 可选属性
    readonly age: number;    // 只读属性
}

// 接口声明合并
interface User {
    phone?: string;
}

// 函数类型接口
interface SearchFunc {
    (source: string, subString: string): boolean;
}

let mySearch: SearchFunc = function(src, sub) {
    return src.search(sub) > -1;
};
```

### 类型别名

```typescript
type ID = string | number;
type Point = { x: number; y: number };
type Callback = (data: string) => void;

// 使用类型别名定义联合类型
type Result = Success | Error;
type Direction = "north" | "south" | "east" | "west";
```

### 两者的区别

| 特性 | 接口 | 类型别名 |
|------|------|----------|
| 定义对象结构 | ✅ | ✅ |
| 定义基本类型 | ❌ | ✅ |
| 定义联合/交叉类型 | ❌ | ✅ |
| 声明合并 | ✅ | ❌ |
| 实现（implements） | ✅ | ✅ |
| 扩展（extends） | ✅ | ✅ |

```typescript
// 接口扩展接口
interface Animal {
    name: string;
}
interface Dog extends Animal {
    breed: string;
}

// 类型别名交叉
type Animal = {
    name: string;
};
type Dog = Animal & {
    breed: string;
};
```

## 5. 函数类型

### 函数参数和返回值类型

```typescript
// 函数声明
function greet(name: string): string {
    return `Hello, ${name}`;
}

// 箭头函数
const greetArrow = (name: string): string => `Hello, ${name}`;

// 参数类型
function process(x: number, transform: (n: number) => number): number {
    return transform(x);
}
```

### 可选参数、默认参数

```typescript
// 可选参数（必须放在必需参数后面）
function buildName(firstName: string, lastName?: string): string {
    if (lastName) {
        return `${firstName} ${lastName}`;
    }
    return firstName;
}

// 默认参数
function connect(host: string, port: number = 8080): string {
    return `${host}:${port}`;
}
```

### 剩余参数

```typescript
// 剩余参数会被当作数组
function sum(...numbers: number[]): number {
    return numbers.reduce((acc, n) => acc + n, 0);
}

console.log(sum(1, 2, 3, 4, 5));  // 15

// 数组解构
function printTeam(team: string, ...members: string[]) {
    console.log(`Team: ${team}`);
    console.log(`Members: ${members.join(", ")}`);
}
```

## 6. 泛型

泛型允许创建可重用的组件，同时保持类型安全。

### 泛型函数

```typescript
// 泛型函数
function identity<T>(arg: T): T {
    return arg;
}

// 使用方式
let output1 = identity<string>("hello");  // 显式指定类型
let output2 = identity(42);               // 类型推断

// 多个类型参数
function pair<K, V>(key: K, value: V): [K, V] {
    return [key, value];
}
```

### 泛型接口

```typescript
interface Container<T> {
    value: T;
    getValue(): T;
}

class Box<T> implements Container<T> {
    constructor(public value: T) {}
    getValue(): T {
        return this.value;
    }
}

// 泛型约束
interface HasLength {
    length: number;
}
function logLength<T extends HasLength>(arg: T): number {
    return arg.length;
}
```

### 约束（extends）

```typescript
// 让泛型继承特定类型
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}

let user = { name: "Alice", age: 25 };
let name = getProperty(user, "name");  // string

// 使用现有类型作为约束
type Point2D = { x: number; y: number };
type Point3D = { x: number; y: number; z: number };

function printPoint<T extends Point2D>(point: T): void {
    console.log(`x: ${point.x}, y: ${point.y}`);
}
```

## 7. 高级类型

### 交叉类型、联合类型

```typescript
// 交叉类型：同时具有多个类型的所有成员
interface Colorful {
    color: string;
}
interface Circle {
    radius: number;
}
type ColorfulCircle = Colorful & Circle;

// 联合类型：可以是多种类型之一
type StringOrNumber = string | number;
type Result = "success" | "error" | "loading";

// 区分联合类型
function handleResult(result: Result) {
    if (result === "success") {
        console.log("操作成功");
    } else if (result === "error") {
        console.log("操作失败");
    } else {
        console.log("加载中...");
    }
}
```

### 条件类型

```typescript
// 基本语法：T extends U ? X : Y
type IsString<T> = T extends string ? "yes" : "no";

type A = IsString<string>;  // "yes"
type B = IsString<number>;  // "no"

// 分布式条件类型
type Flatten<T> = T extends Array<infer U> ? U : T;

type C = Flatten<string[]>;  // string
type D = Flatten<number>;    // number
```

### 映射类型

```typescript
// 从现有类型创建新类型
type Readonly<T> = {
    readonly [P in keyof T]: T[P];
};

type Optional<T> = {
    [P in keyof T]?: T[P];
};

type Partial<T> = {
    [P in keyof T]?: T[P];
};

// 内置映射类型
type KeysOfType<T, U> = {
    [P in keyof T]: T[P] extends U ? P : never;
}[keyof T];
```

### 模板字面量类型

```typescript
// 模板字面量类型
type World = "world";
type Greeting = `hello ${World}`;  // "hello world"

// 组合多个字面量类型
type Email = `${string}@${string}.${string}`;
type Dir = "left" | "right" | "up" | "down";
type EventName = `on${Capitalize<Dir>}`;  // "onLeft" | "onRight" | "onUp" | "onDown"
```

## 8. 实际应用

### React + TypeScript

```tsx
import React, { useState } from "react";

// 定义Props接口
interface ButtonProps {
    children: React.ReactNode;
    onClick?: () => void;
    variant?: "primary" | "secondary";
    disabled?: boolean;
}

// 函数组件
function Button({ 
    children, 
    onClick, 
    variant = "primary", 
    disabled = false 
}: ButtonProps) {
    const baseClass = "btn";
    const variantClass = `btn-${variant}`;
    
    return (
        <button 
            className={`${baseClass} ${variantClass}`}
            onClick={onClick}
            disabled={disabled}
        >
            {children}
        </button>
    );
}

// 使用泛型定义表单事件
interface FormState {
    username: string;
    password: string;
}

function LoginForm() {
    const [form, setForm] = useState<FormState>({
        username: "",
        password: ""
    });

    const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        const { name, value } = e.target;
        setForm(prev => ({ ...prev, [name]: value }));
    };

    const handleSubmit = (e: React.FormEvent) => {
        e.preventDefault();
        console.log("Submit:", form);
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                name="username"
                value={form.username}
                onChange={handleChange}
            />
            <input
                name="password"
                type="password"
                value={form.password}
                onChange={handleChange}
            />
            <Button type="submit">登录</Button>
        </form>
    );
}
```

### 类型守卫

类型守卫是在运行时检查类型的方式，帮助TypeScript缩小联合类型的范围。

```typescript
// 基本类型守卫
function isString(value: unknown): value is string {
    return typeof value === "string";
}

function process(value: string | number) {
    if (isString(value)) {
        console.log(value.toUpperCase());  // string类型
    } else {
        console.log(value.toFixed(2));     // number类型
    }
}

// in操作符守卫
interface Cat {
    meow(): void;
}
interface Dog {
    bark(): void;
}

function speak(animal: Cat | Dog) {
    if ("meow" in animal) {
        animal.meow();
    } else {
        animal.bark();
    }
}

// 断言函数
function assertIsDefined<T>(value: T, message: string): asserts value is NonNullable<T> {
    if (value === null || value === undefined) {
        throw new Error(message);
    }
}

function getUserName(user: { name: string } | null | undefined): string {
    assertIsDefined(user, "User is required");
    return user.name;  // TypeScript知道user不为null/undefined
}
```

---

## 参考资料

[^1]: [TypeScript官方文档](https://www.typescriptlang.org/docs/)
[^2]: [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)

[^1]: 本教程基于TypeScript官方文档和实际开发经验整理。
