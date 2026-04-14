---
title: 语法分析 (Parser)
date: 2026-04-09
description: 编译器语法分析阶段详解 — 从Token流到AST抽象语法树
tags:
  - compiler
  - parser
  - ast
draft: false
permalink:
---

# 语法分析 (Syntax Analysis)

语法分析（Syntax Analysis）是编译器的第二个阶段，也称为**解析器（Parser）**。它的任务是将词法分析产生的 Token 流按照语言的语法规则组织成**抽象语法树（Abstract Syntax Tree, AST）**，并检测语法错误。[^1]

[^1]: 本段参考[How Compilers Work: From Lexers to Parsers to ASTs](https://abhisheyk-gaur.medium.com/how-compilers-work-from-lexers-to-parsers-to-asts-ab95e0b1dcf1)

## Parser 的输入与输出

**输入**：`[INT_KEYWORD, IDENT(x), ASSIGN, NUMBER(42), SEMICOLON]`

**输出**：抽象语法树（AST）

```
        =
       / \
      x   42
```

## 上下文无关文法 (CFG)

编程语言的语法通常使用**上下文无关文法（Context-Free Grammar, CFG）**描述。CFG 由以下元素组成：

- **终结符（Terminal）**：Token，如 `if`、`while`、`+`、`-`
- **非终结符（Non-terminal）**：语法变量，如 `expr`、`stmt`、`program`
- **产生式（Production）**：规则，如 `expr → expr + term`
- **开始符号（Start Symbol）**：通常为 `program`

### 示例：算术表达式文法

```
expr    → expr + term | expr - term | term
term    → term * factor | term / factor | factor
factor  → ( expr ) | NUMBER | IDENT
```

这定义了运算符优先级：`*`/`/` 高于 `+`/`-`。

## 语法分析的方法

### 自顶向下分析 (Top-Down Parsing)

从根节点开始，尝试推导出输入序列。

**代表算法**：
- 递归下降分析（Recursive Descent Parsing）
- LL(1) 分析

**特点**：实现简单，适合手工编写。

### 自底向上分析 (Bottom-Up Parsing)

从输入序列开始，尝试归约为开始符号。

**代表算法**：
- LR(0)、SLR(1)、LALR(1)、LR(1)

**特点**：表达能力更强，适合工具生成。

## 递归下降分析器

递归下降是最常见的自顶向下分析方法，为每个非终结符编写一个递归函数。

### 基本框架

```cpp
#include <bits/stdc++.h>
using namespace std;

// Token 定义（来自 lexer）
enum TokenType {
    IDENT, NUMBER, PLUS, MINUS, MUL, DIV,
    ASSIGN, LPAREN, RPAREN, SEMICOLON,
    INT_KEYWORD, EOF_TOKEN
};

struct Token {
    TokenType type;
    string value;
    int line;
    int column;
};

// 词法分析器
class Lexer {
    // ... (参见 lexer.md)
};

// AST 节点基类
class ASTNode {
public:
    virtual ~ASTNode() {}
    virtual void print(int indent = 0) = 0;
};

// 表达式节点
class ExprNode : public ASTNode {
public:
    virtual ~ExprNode() {}
};

// 数字字面量
class NumberExpr : public ExprNode {
public:
    int value;
    NumberExpr(int val) : value(val) {}
    void print(int indent = 0) override {
        cout << string(indent, ' ') << "Number: " << value << endl;
    }
};

// 变量引用
class IdentExpr : public ExprNode {
public:
    string name;
    IdentExpr(const string& n) : name(n) {}
    void print(int indent = 0) override {
        cout << string(indent, ' ') << "Ident: " << name << endl;
    }
};

// 二元运算
class BinaryExpr : public ExprNode {
public:
    char op;
    ExprNode* left;
    ExprNode* right;
    BinaryExpr(char oper, ExprNode* l, ExprNode* r) : op(oper), left(l), right(r) {}
    ~BinaryExpr() { delete left; delete right; }
    void print(int indent = 0) override {
        cout << string(indent, ' ') << "BinaryExpr: " << op << endl;
        left->print(indent + 2);
        right->print(indent + 2);
    }
};

// 赋值语句
class AssignStmt : public ASTNode {
public:
    string name;
    ExprNode* expr;
    AssignStmt(const string& n, ExprNode* e) : name(n), expr(e) {}
    ~AssignStmt() { delete expr; }
    void print(int indent = 0) override {
        cout << string(indent, ' ') << "Assign: " << name << endl;
        expr->print(indent + 2);
    }
};

// 语法分析器
class Parser {
private:
    Lexer& lexer;
    Token currentToken;
    
    void error(const string& msg) {
        cerr << "Syntax Error at " << currentToken.line << ":" 
             << currentToken.column << " - " << msg << endl;
        exit(1);
    }
    
    void advance() {
        currentToken = lexer.getNextToken();
    }
    
    bool expect(TokenType type) {
        if (currentToken.type == type) {
            advance();
            return true;
        }
        return false;
    }
    
    // expr → expr + term | expr - term | term
    ExprNode* parseExpr() {
        ExprNode* left = parseTerm();
        
        while (currentToken.type == PLUS || currentToken.type == MINUS) {
            char op = (currentToken.type == PLUS) ? '+' : '-';
            advance();
            ExprNode* right = parseTerm();
            left = new BinaryExpr(op, left, right);
        }
        
        return left;
    }
    
    // term → term * factor | term / factor | factor
    ExprNode* parseTerm() {
        ExprNode* left = parseFactor();
        
        while (currentToken.type == MUL || currentToken.type == DIV) {
            char op = (currentToken.type == MUL) ? '*' : '/';
            advance();
            ExprNode* right = parseFactor();
            left = new BinaryExpr(op, left, right);
        }
        
        return left;
    }
    
    // factor → ( expr ) | NUMBER | IDENT
    ExprNode* parseFactor() {
        if (currentToken.type == LPAREN) {
            advance();
            ExprNode* expr = parseExpr();
            if (currentToken.type != RPAREN) {
                error("Expected ')'");
            }
            advance();
            return expr;
        } else if (currentToken.type == NUMBER) {
            int val = stoi(currentToken.value);
            advance();
            return new NumberExpr(val);
        } else if (currentToken.type == IDENT) {
            string name = currentToken.value;
            advance();
            return new IdentExpr(name);
        } else {
            error("Unexpected token");
            return nullptr;
        }
    }
    
public:
    Parser(Lexer& lex) : lexer(lex) {
        advance();
    }
    
    ASTNode* parse() {
        // 简单起见，解析 "x = expr;" 形式的赋值语句
        if (currentToken.type == IDENT) {
            string name = currentToken.value;
            advance();
            
            if (currentToken.type == ASSIGN) {
                advance();
                ExprNode* expr = parseExpr();
                
                if (currentToken.type == SEMICOLON) {
                    advance();
                } else {
                    error("Expected ';'");
                }
                
                return new AssignStmt(name, expr);
            }
        }
        return nullptr;
    }
};
```

### LL(1) 分析与左递归消除

上述递归下降分析器存在**左递归（Left Recursion）**问题：

```
expr → expr + term  // 左递归：产生式左侧与右侧开始相同
```

左递归会导致无限递归。解决方法：**消除左递归**：

```cpp
// 消除左递归后：
// expr  → term expr_tail
// expr_tail → + term expr_tail | - term expr_tail | ε

ExprNode* parseExpr() {
    ExprNode* left = parseTerm();
    
    while (currentToken.type == PLUS || currentToken.type == MINUS) {
        char op = (currentToken.type == PLUS) ? '+' : '-';
        advance();
        ExprNode* right = parseTerm();
        left = new BinaryExpr(op, left, right);
    }
    
    return left;
}
```

## 运算符优先级处理

### 算符优先级分析 (Operator-Pcedence Parsing)

使用优先级表处理运算符优先级：

```cpp
#include <map>

enum Precedence {
    LOWEST = 1,
    EQUALS = 2,
    LESSGREATER = 3,
    SUM = 4,
    PRODUCT = 5,
    PREFIX = 6,
    CALL = 7
};

map<TokenType, int> precedences = {
    {PLUS, SUM},
    {MINUS, SUM},
    {MUL, PRODUCT},
    {DIV, PRODUCT}
};

int getPrecedence(TokenType type) {
    if (precedences.find(type) != precedences.end()) {
        return precedences[type];
    }
    return LOWEST;
}

ExprNode* parseExpression(int prevPrec) {
    ExprNode* leftExpr = parsePrefixExpression();
    
    while (getPrecedence(currentToken.type) > prevPrec) {
        TokenType op = currentToken.type;
        advance();
        ExprNode* rightExpr = parseExpression(getPrecedence(op));
        leftExpr = new BinaryExpr(tokenToChar(op), leftExpr, rightExpr);
    }
    
    return leftExpr;
}
```

## 抽象语法树 (AST)

### AST vs Parse Tree

| 特征 | Parse Tree | AST |
|------|------------|-----|
| 完整性 | 包含所有语法信息 | 只保留语义信息 |
| 节点类型 | 终结符+非终结符 | 表达式/语句类型 |
| 冗余性 | 可能包含冗余节点（如括号） | 简化后的树形结构 |
| 用途 | 理论分析 | 实际编译 |
| 大小 | 可能很大 | 更紧凑 |

### AST 节点设计

```cpp
// 表达式类型枚举
enum ExprType {
    EXPR_NUMBER,
    EXPR_IDENT,
    EXPR_BINARY,
    EXPR_UNARY,
    EXPR_CALL
};

// AST 节点
struct ASTNode {
    ExprType type;
    union {
        int numValue;
        string* identName;
        struct {
            char op;
            ASTNode* left;
            ASTNode* right;
        } binary;
        struct {
            char op;
            ASTNode* operand;
        } unary;
        struct {
            string* funcName;
            vector<ASTNode*>* args;
        } call;
    };
};
```

### Clang AST 示例

使用 Clang 查看 C 代码的 AST：

```bash
clang -Xclang -ast-dump -fsyntax-only example.c
```

输出示例（对于 `int x = 42 + y;`）：

```
TranslationUnitDecl 0x7f8a9c02a080 <<invalid sloc>>
`-VarDecl 0x7f8a9c02a380 <line:1:1, col:8> col:5 x 'int'
  `-IntegerLiteral 0x7f8a9c02a280 <col:9> 'int' 42
`-BinaryOperator 0x7f8a9c02a480 <col:12, col:16> '+' 'int'
  |-IntegerLiteral 0x7f8a9c02a280 <col:12> 'int' 42
  `-DeclRefExpr 0x7f8a9c02a380 <col:16> 'int' lvalue Var 0x7f8a9c02a380 'x'
```

## Yacc/Bison 与 LALR 分析

GNU Bison（原 Yacc）是生成 LALR(1) 分析器的工具。

### Bison 语法文件

```yacc
%{
#include <stdio.h>
#include <stdlib.h>
%}

%token IDENT NUMBER
%token INT IF ELSE WHILE RETURN
%token ASSIGN EQ NE LT LE GT GE
%token PLUS MINUS MUL DIV

%%

program     : declaration_list
            ;

declaration_list : declaration_list declaration
                 | declaration
                 ;

declaration : var_decl
            | fun_decl
            ;

var_decl    : INT IDENT ';' { printf("VarDecl: %s\n", $2); }
            ;

expr        : NUMBER { $$ = $1; }
            | IDENT  { $$ = $1; }
            | expr PLUS expr { $$ = makeBinaryExpr('+', $1, $3); }
            | expr MINUS expr { $$ = makeBinaryExpr('-', $1, $3); }
            ;

%%

int main() {
    return yyparse();
}

void yyerror(const char* s) {
    fprintf(stderr, "Error: %s\n", s);
}
```

## 语法错误处理

### 恐慌模式恢复 (Panic Mode Recovery)

当 Parser 遇到错误时，跳过输入直到找到同步点：

```cpp
void syncToSynchronizationPoint() {
    while (currentToken.type != SEMICOLON && 
           currentToken.type != RBRACE &&
           currentToken.type != EOF_TOKEN) {
        advance();
    }
    if (currentToken.type != EOF_TOKEN) {
        advance();
    }
}
```

### 短语层恢复 (Phrase-Level Recovery)

尝试从错误中恢复，预测正确的语法结构：

```cpp
ExprNode* parseExpr() {
    ExprNode* left = nullptr;
    
    try {
        left = parseTerm();
    } catch (ParseError& e) {
        // 尝试插入缺少的 token
        insertToken(LPAREN);
        left = parseExpr();
        expectToken(RPAREN);
    }
    
    return left;
}
```

## 完整示例：计算器

结合 Lexer 和 Parser 实现简单计算器：

```cpp
#include <bits/stdc++.h>
using namespace std;

class Calculator {
    Lexer lexer;
    Parser parser;
    
public:
    Calculator(const string& input) : lexer(input), parser(lexer) {}
    
    int evaluate() {
        ASTNode* node = parser.parse();
        return eval(node);
    }
    
private:
    int eval(ASTNode* node) {
        if (dynamic_cast<NumberExpr*>(node)) {
            return dynamic_cast<NumberExpr*>(node)->value;
        }
        if (auto* bin = dynamic_cast<BinaryExpr*>(node)) {
            int left = eval(bin->left);
            int right = eval(bin->right);
            switch (bin->op) {
                case '+': return left + right;
                case '-': return left - right;
                case '*': return left * right;
                case '/': return left / right;
            }
        }
        return 0;
    }
};

int main() {
    Calculator calc("2 + 3 * 4");
    cout << calc.evaluate() << endl; // 输出 14
}
```

## 语义分析简介

Parser 输出 AST 后，语义分析器负责：

1. **类型检查**：`int x = "hello";` 是否合法
2. **作用域分析**：变量是否已声明
3. **符号表管理**：维护变量/函数信息

```
AST → [语义分析] → [类型检查] → [中间代码]
```

## 总结

语法分析是编译器的核心阶段之一：

1. **输入**：Token 流
2. **输出**：抽象语法树（AST）
3. **方法**：递归下降、LL、LR 系列
4. **关键问题**：左递归、优先级、错误恢复

理解语法分析对于：
- 实现编程语言
- 开发静态分析工具
- 理解 IDE 错误提示原理

至关重要。

## 参考

- [How Compilers Work: From Lexers to Parsers to ASTs](https://abhisheyk-gaur.medium.com/how-compilers-work-from-lexers-to-parsers-to-asts-ab95e0b1dcf1)
- [Kaleidoscope: Implementing a Parser and AST (LLVM Tutorial)](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/LangImpl02.html)
- [Compiler Overview: Scanning and Parsing (UW)](https://courses.cs.washington.edu/courses/cse390b/21sp/readings/compiler_scanner_parser.html)
