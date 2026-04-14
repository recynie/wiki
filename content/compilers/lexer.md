---
title: 词法分析 (Lexer)
date: 2026-04-09
description: 编译器词法分析阶段详解 — 从字符流到Token序列
tags:
  - compiler
  - lexer
  - token
draft: false
permalink:
---

# 词法分析 (Lexical Analysis)

词法分析（Lexical Analysis）是编译器的第一个阶段，也称为**词法分析器（Lexer）**或**扫描器（Scanner）**。它的任务是将源程序的字符流划分为一个个**词法单元（Token）**，为后续的语法分析做准备。[^1]

[^1]: 本段参考[Compiler - Lexical Analysis](https://www.geeksforgeeks.org/compiler-design-lexical-analysis/)

## 编译器架构概述

现代编译器的典型 pipeline 如下：

```
源文件 → [词法分析] → [语法分析] → [语义分析] → [中间代码生成] → [优化] → [目标代码生成]
         ↑ Token流                                                      ↓
```

词法分析是编译器的**前端（Front End）**入口，负责将原始字符转换为结构化的 Token 序列。

## 基本概念

### Token 的定义

Token 是源代码中的基本语义单元，主要包括：

| Token 类型 | 示例 | 说明 |
|-----------|------|------|
| 关键字 | `if`, `else`, `while`, `int` | 语言保留词 |
| 标识符 | `x`, `sum`, `myFunction` | 变量名、函数名 |
| 常量 | `42`, `3.14`, `"hello"` | 数字、字符串字面量 |
| 运算符 | `+`, `-`, `*`, `/`, `=` | 算术、逻辑运算符 |
| 界符 | `(`, `)`, `{`, `}`, `;`, `,` | 分隔符 |

### 词法单元 vs 字符流

**输入**：`int x = 42;`

**输出 Token 流**：
```
[INT_KEYWORD, IDENT(x), ASSIGN, NUMBER(42), SEMICOLON]
```

词法分析器会：
- 跳过空白字符（空格、制表符、换行符）
- 跳过注释
- 识别标识符并区分关键字
- 处理数字、字符串字面量

## 词法分析器的工作方式

### 扫描策略

典型的词法分析器通过**有限自动机（Finite Automaton）**识别 Token：

```cpp
#include <bits/stdc++.h>
using namespace std;

// Token 类型枚举
enum TokenType {
    IDENT,    // 标识符
    NUMBER,   // 数字
    PLUS,     // +
    MINUS,    // -
    MUL,      // *
    DIV,      // /
    ASSIGN,   // =
    LPAREN,  // (
    RPAREN,  // )
    SEMICOLON, // ;
    INT_KEYWORD, // int
    EOF_TOKEN
};

struct Token {
    TokenType type;
    string value;
    int line;
    int column;
};

// 词法分析器类
class Lexer {
private:
    string src;       // 源代码
    size_t pos;       // 当前读取位置
    int line;         // 当前行号
    int column;       // 当前列号
    
    char peek() { return src[pos]; }
    char advance() {
        char c = src[pos++];
        if (c == '\n') { line++; column = 0; }
        else { column++; }
        return c;
    }
    bool isAlpha(char c) { return isalpha(c); }
    bool isDigit(char c) { return isdigit(c); }
    bool isAlphaNumeric(char c) { return isAlpha(c) || isDigit(c); }
    
    void skipWhitespace() {
        while (pos < src.size() && isspace(peek())) {
            advance();
        }
    }
    
    void skipComment() {
        if (peek() == '/' && pos + 1 < src.size() && src[pos + 1] == '/') {
            while (pos < src.size() && peek() != '\n') advance();
        }
    }
    
    Token makeIdentifier() {
        string result;
        while (pos < src.size() && isAlphaNumeric(peek())) {
            result += advance();
        }
        // 关键字表
        if (result == "int") return {INT_KEYWORD, result, line, column};
        return {IDENT, result, line, column};
    }
    
    Token makeNumber() {
        string result;
        while (pos < src.size() && isdigit(peek())) {
            result += advance();
        }
        return {NUMBER, result, line, column};
    }
    
public:
    Lexer(const string& source) : src(source), pos(0), line(1), column(0) {}
    
    vector<Token> tokenize() {
        vector<Token> tokens;
        while (pos < src.size()) {
            skipWhitespace();
            skipComment();
            if (pos >= src.size()) break;
            
            char c = peek();
            Token token;
            token.line = line;
            token.column = column;
            
            if (isAlpha(c)) {
                token = makeIdentifier();
            } else if (isdigit(c)) {
                token = makeNumber();
            } else {
                advance(); // 消耗当前字符
                switch (c) {
                    case '+': token = {PLUS, "+", line, column}; break;
                    case '-': token = {MINUS, "-", line, column}; break;
                    case '*': token = {MUL, "*", line, column}; break;
                    case '/': token = {DIV, "/", line, column}; break;
                    case '=': token = {ASSIGN, "=", line, column}; break;
                    case '(': token = {LPAREN, "(", line, column}; break;
                    case ')': token = {RPAREN, ")", line, column}; break;
                    case ';': token = {SEMICOLON, ";", line, column}; break;
                    default: continue; // 跳过未知字符
                }
            }
            tokens.push_back(token);
        }
        tokens.push_back({EOF_TOKEN, "", line, column});
        return tokens;
    }
};
```

## 有限状态自动机

词法分析器的核心是**有限状态自动机（DFA/NFA）**。以识别数字和标识符为例：

```
                    ┌─────────────────────────────────────┐
                    │                                     │
    ┌───┐          ┌───┐                                  │
    │   │  digit  │   │  digit                           │
    │ 0 │────────►│ 1 │──────────────────────────────► accepting (number)
    │   │         │   │  other                           │
    └───┘          └───┘                                  │
      │              │                                    │
      │ other        │ other                             │
      ▼              ▼                                    ▼
┌───────────┐    ┌───────────┐                      ┌───────────┐
│  start    │    │  ident    │◄──── alpha/numeric───│ accepting │
└───────────┘    └───────────┘                       │ (ident)   │
                                                     └───────────┘
```

### 状态转换图实现

```cpp
enum State {
    START,
    IN_IDENT,
    IN_NUMBER,
    DONE
};

Token tokenizeIdentifierOrNumber(const string& src, size_t& pos) {
    State state = START;
    string value;
    
    while (pos < src.size()) {
        char c = src[pos];
        
        switch (state) {
            case START:
                if (isalpha(c)) {
                    state = IN_IDENT;
                    value += c;
                    pos++;
                } else if (isdigit(c)) {
                    state = IN_NUMBER;
                    value += c;
                    pos++;
                } else {
                    // 处理单字符 token
                    pos++;
                    return makeSingleCharToken(c);
                }
                break;
                
            case IN_IDENT:
                if (isalnum(c)) {
                    value += c;
                    pos++;
                } else {
                    state = DONE;
                }
                break;
                
            case IN_NUMBER:
                if (isdigit(c)) {
                    value += c;
                    pos++;
                } else {
                    state = DONE;
                }
                break;
                
            case DONE:
                return makeToken(value);
        }
    }
    
    if (state == IN_IDENT) return makeToken(value);
    if (state == IN_NUMBER) return makeToken(value);
    return {EOF_TOKEN, "", 0, 0};
}
```

## 词法分析器的工具

### Lex/Flex

Lex（及其 GNU 实现 Flex）是经典的词法分析器生成器。使用正则表达式定义 token 模式：

```flex
/* example.l */
%{
#include <stdio.h>
%}

%%

[0-9]+              { printf("NUMBER: %s\n", yytext); }
[a-zA-Z][a-zA-Z0-9]* { printf("IDENT: %s\n", yytext); }
"+"                 { printf("PLUS\n"); }
"-"                 { printf("MINUS\n"); }
"*"                 { printf("MUL\n"); }
"/"                 { printf("DIV\n"); }
[ \t\n]             { /* skip whitespace */ }
.                   { printf("UNKNOWN: %s\n", yytext); }

%%

int main() {
    yylex();
    return 0;
}
```

编译运行：
```bash
flex example.l
gcc lex.yy.c -o lexer
./lexer < input.c
```

## 词法分析 vs 语法分析

| 特征 | 词法分析 | 语法分析 |
|------|---------|---------|
| 输入 | 字符流 | Token 流 |
| 输出 | Token 序列 | 语法树（AST） |
| 处理方式 | 局部、顺序 | 全局、结构化 |
| 形式化方法 | 正则表达式 | 上下文无关文法 |
| 工具 | Lex/Flex | Yacc/Bison |
| 错误类型 | 非法字符 | 语法错误 |

## 词法分析的错误处理

### 常见错误

1. **非法字符**：源代码包含不属于语言字符集的字符
2. **未终结的字符串**：字符串常量缺少结束引号
3. **未终结的注释**：注释缺少结束符
4. **标识符过长**：超出语言规定的最大长度

### 错误恢复策略

```cpp
TokenType lexError(const string& msg, int line, int col) {
    cerr << "Lexical Error at " << line << ":" << col << " - " << msg << endl;
    return ERROR_TOKEN;
}

// 恐慌模式恢复：跳过错误字符，继续扫描
void errorRecovery(Lexer& lexer) {
    while (!isspace(peek())) advance(); // 跳过直到空白
    // 继续分析
}
```

## 实际编译器中的词法分析

### Clang 的 Token  dump

使用 Clang 查看真实 C 代码的 Token 流：

```bash
clang -Xclang -dump-tokens -fsyntax-only example.c
```

输出示例：
```
int 'int'        [StartOfLine]  Loc=<example.c:1:1>
identifier 'x'   [LeadingSpace]  Loc=<example.c:1:5>
equal '='        [LeadingSpace] Loc=<example.c:1:7>
numeric_constant '42'  [LeadingSpace]  Loc=<example.c:1:9>
semi ';'                 Loc=<example.c:1:11>
eof ''                   Loc=<example.c:1:12>
```

### Python 的 `ast` 模块

Python 词法分析和语法分析集成在 `ast` 模块中：

```python
import ast

code = "x = 42 + y"
tree = ast.parse(code)
print(ast.dump(tree, indent=4))
```

输出：
```
Module(
 body=[
  Assign(
   targets=[Name(id='x', ctx=Store())],
   value=BinOp(
    left=Constant(value=42),
    op=Add(),
    right=Name(id='y', ctx=Load())
   )
  )
 ]
)
```

## 总结

词法分析是编译器的第一步，其核心任务包括：

1. **字符过滤**：去除空白、注释
2. **Token 识别**：将字符流转换为 Token 流
3. **属性收集**：记录 Token 的值、位置信息
4. **错误检测**：识别非法字符和格式错误

词法分析虽然相对简单，但为后续的语法分析奠定了基础。理解词法分析对于：
- 编写词法分析器
- 理解 IDE 语法高亮原理
- 开发静态分析工具
- 设计新的编程语言

都非常重要。

## 参考

- [Compiler Design - Lexical Analysis (GeeksforGeeks)](https://www.geeksforgeeks.org/compiler-design-lexical-analysis/)
- [Compiler Lexer & Parser Demystified (Coder Musings)](https://www.codermusings.com/compiler-lexer-parser-guide/)
- [Writing a C Compiler from Scratch](https://innovirtuoso.com/programming-tutorials/writing-a-c-compiler-from-scratch-a-step-by-step-guide-to-building-a-real-c-to-x86-64-language/)
- [CSAPP Chapter 7: Linking](https://csapp.cs.cmu.edu/3e/home.html)
