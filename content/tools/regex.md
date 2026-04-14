---
title: 正则表达式
date: 2026-04-11
description: 全面介绍正则表达式的语法、匹配规则与实际应用
tags:
  - regex
  - tool
  - string
draft: false
permalink:
---

# 正则表达式

正则表达式（Regular Expression）是一种强大的文本模式匹配工具，广泛用于字符串处理、验证、搜索和替换等场景。

## 1. 正则表达式基础

### 原子（字符匹配）

原子是正则表达式中最基本的匹配单元，分为：

| 原子 | 含义 |
|------|------|
| `a` | 匹配字符 `a` 本身 |
| `\` | 转义字符，如 `\.` 匹配点号 |
| `\n` | 换行符 |
| `\t` | 制表符 |

### 元字符

元字符具有特殊含义：

| 元字符 | 含义 |
|--------|------|
| `.` | 匹配任意单个字符（换行符除外） |
| `\d` | 匹配任意数字，等价于 `[0-9]` |
| `\D` | 匹配任意非数字 |
| `\w` | 匹配字母、数字、下划线，等价于 `[a-zA-Z0-9_]` |
| `\W` | 匹配非字母、数字、下划线 |
| `\s` | 匹配空白字符（空格、制表符、换行符） |
| `\S` | 匹配非空白字符 |

```cpp
// C++ 示例
std::regex r("\\d+");  // 匹配一个或多个数字
std::regex r2("\\w+"); // 匹配一个或多个单词字符
```

### 字符类

使用方括号 `[]` 定义字符类：

| 字符类 | 含义 |
|--------|------|
| `[abc]` | 匹配 `a`、`b` 或 `c` 中的任意一个 |
| `[^abc]` | 匹配除 `a`、`b`、`c` 以外的任意字符 |
| `[a-z]` | 匹配小写字母 `a` 到 `z` |
| `[A-Z]` | 匹配大写字母 `A` 到 `Z` |
| `[0-9]` | 匹配数字 |
| `[a-zA-Z0-9]` | 匹配字母或数字 |

```cpp
std::regex r("[a-zA-Z]+");  // 匹配一个或多个英文字母
std::regex r2("[^0-9]");   // 匹配非数字字符
```

---

## 2. 量词

量词用于指定匹配的次数：

| 量词 | 含义 |
|------|------|
| `*` | 匹配 0 次或多次 |
| `+` | 匹配 1 次或多次 |
| `?` | 匹配 0 次或 1 次 |
| `{n}` | 匹配恰好 n 次 |
| `{n,}` | 匹配至少 n 次 |
| `{n,m}` | 匹配 n 到 m 次 |

### 贪婪与非贪婪

默认情况下，量词是**贪婪匹配**，会尽可能多地匹配字符。在量词后加 `?` 可变为**非贪婪匹配**（也称惰性匹配）。

```cpp
// 贪婪匹配：".*" 会匹配整个字符串
std::regex r1(".*");
// 非贪婪匹配：".*?" 只匹配尽可能少的字符
std::regex r2(".*?");
```

| 贪婪量词 | 非贪婪量词 | 匹配行为 |
|----------|------------|----------|
| `.*` | `.*?` | 尽可能多 vs 尽可能少 |
| `.+` | `.+?` | 尽可能多 vs 尽可能少 |
| `.?` | `.??` | 尽可能多 vs 尽可能少 |

**示例：**

```
源字符串：<div>hello</div><div>world</div>

贪婪：<div>.*</div>   → 匹配整个字符串
非贪婪：<div>.*?</div> → 分别匹配 <div>hello</div> 和 <div>world</div>
```

---

## 3. 边界

边界用于指定匹配的位置：

| 边界 | 含义 |
|------|------|
| `^` | 匹配字符串开始（在字符类内表示"非"） |
| `$` | 匹配字符串结束 |
| `\b` | 匹配单词边界 |
| `\B` | 匹配非单词边界 |

```cpp
// 匹配以 "hello" 开头的行
std::regex r("^hello");

// 匹配以 "world" 结尾的行
std::regex r2("world$");

// 匹配完整单词 "cat"
std::regex r3("\\bcat\\b");
```

**示例：**

```
源字符串：The cat catches a catfish.

\bcat\b  → 匹配 "cat"（第一个）
\bcat    → 匹配 "cat" 和 "catfish" 中的 "cat"
cat\B    → 匹配 "catfish" 中的 "cat"
```

---

## 4. 分组与引用

### 捕获组

使用圆括号 `()` 创建捕获组，可以：

1. 将部分模式组合在一起
2. 捕获匹配的子字符串
3. 后续引用捕获的内容

```cpp
std::regex r("(\\d{4})-(\\d{2})-(\\d{2})");  // 匹配日期格式
// 第一组：年，第二组：月，第三组：日
```

### 非捕获组

使用 `(?:...)` 创建非捕获组，只组合模式但不创建捕获：

```cpp
std::regex r("(?:https?|ftp)://\\S+");  // 匹配 URL，但不捕获协议部分
```

### 反向引用

使用 `\n` 引用第 n 个捕获组：

```cpp
// 匹配重复的单词，如 "the the"
std::regex r("\\b(\\w+)\\s+\\1\\b");
// \1 引用第一个捕获组 (\w+)
```

| 语法 | 含义 |
|------|------|
| `()` | 捕获组 |
| `(?:)` | 非捕获组 |
| `\1` | 引用第 1 个捕获组 |
| `\2` | 引用第 2 个捕获组 |

---

## 5. 零宽断言

零宽断言不匹配任何字符，只匹配位置。

### 正前瞻

`(?=pattern)`：断言左侧位置后面匹配 pattern

```cpp
std::regex r("\\w+(?=@)");  // 匹配 @ 前面的单词字符
// "user@example.com" → 匹配 "user"
```

### 负前瞻

`(?!pattern)`：断言左侧位置后面不匹配 pattern

```cpp
std::regex r("\\d+(?!\\.)");  // 匹配后面不是点的数字
// "123" 在 "123.456" 中会被匹配，"456" 不会被匹配
```

### 正后顾

`(?<=pattern)`：断言右侧位置前面匹配 pattern

```cpp
std::regex r("(?<=@)\\w+");  // 匹配 @ 后面的单词字符
// "user@example.com" → 匹配 "example"
```

### 负后顾

`(?<!pattern)`：断言右侧位置前面不匹配 pattern

```cpp
std::regex r("(?<!\\d)\\d+");  // 匹配前面不是数字的数字
```

| 断言 | 语法 | 含义 |
|------|------|------|
| 正前瞻 | `(?=...)` | 后面匹配 ... |
| 负前瞻 | `(?!...)` | 后面不匹配 ... |
| 正后顾 | `(?<=...)` | 前面匹配 ... |
| 负后顾 | `(?<!...)` | 前面不匹配 ... |

---

## 6. 常用正则模式

### 邮箱地址

```cpp
// 基础邮箱匹配
std::regex email("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$");
```

```
^[a-zA-Z0-9._%+-]+        # 用户名部分
@                         # @ 符号
[a-zA-Z0-9.-]+            # 域名部分
\\.                       # 点号
[a-zA-Z]{2,}              # 顶级域名（至少2个字母）
```

### 手机号（中国大陆）

```cpp
// 匹配中国大陆手机号
std::regex phone("^1[3-9]\\d{9}$");
```

```
^1                        # 以 1 开头
[3-9]                     # 第二位是 3-9
\\d{9}$                   # 后面9位数字
```

### URL

```cpp
std::regex url("https?://[a-zA-Z0-9.-]+(?:\\.[a-zA-Z]{2,})+(?:/\\S*)?$");
```

### IP 地址

```cpp
// IPv4 地址
std::regex ipv4("^(?:(?:25[0-5]|2[0-4]\\d|[01]?\\d\\d?)\\.){3}(?:25[0-5]|2[0-4]\\d|[01]?\\d\\d?)$");
```

### HTML 标签匹配

```cpp
// 匹配 HTML 标签
std::regex html_tag("<([a-zA-Z][a-zA-Z0-9]*)\\b[^>]*>.*?</\\1>");
```

```cpp
// 贪婪示例
std::string text = "<div>hello</div><div>world</div>";
std::regex r("<div>.*?</div>");  // 非贪婪，匹配两个 <div> 标签
std::regex r2("<div>.*</div>");  // 贪婪，匹配整个字符串
```

---

## 7. C++ 中的正则

C++11 引入了 `<regex>` 库，提供了完整的正则表达式支持。

### 常用函数

```cpp
#include <regex>
#include <string>

// 1. 验证是否完全匹配
std::regex_match(str, pattern);

// 2. 搜索匹配部分
std::regex_search(str, match, pattern);

// 3. 替换
std::regex_replace(str, pattern, replacement);
```

### 示例代码

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    // 邮箱验证
    string email = "user@example.com";
    regex email_r("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$");
    if (regex_match(email, email_r)) {
        cout << "Valid email" << endl;
    }

    // 提取日期中的年、月、日
    string date = "2026-04-11";
    regex date_r("(\\d{4})-(\\d{2})-(\\d{2})");
    smatch match;
    if (regex_search(date, match, date_r)) {
        cout << "Year: " << match[1] << endl;  // 2026
        cout << "Month: " << match[2] << endl; // 04
        cout << "Day: " << match[3] << endl;   // 11
    }

    // 替换数字
    string text = "abc123def456";
    regex num_r("\\d+");
    string result = regex_replace(text, num_r, "#");
    cout << result << endl;  // abc#def#

    // 迭代器遍历所有匹配
    string text2 = "12 abc 34 def 56";
    for (sregex_iterator it(text2.begin(), text2.end(), num_r); 
         it != sregex_iterator(); 
         ++it) {
        cout << it->str() << endl;
    }

    return 0;
}
```

### std::regex 支持的语法

| 语法选项 | 说明 |
|----------|------|
| `regex:: ECMAScript` | ECMAScript（默认） |
| `regex::basic` | POSIX Basic |
| `regex::extended` | POSIX Extended |
| `regex::grep` | grep 格式 |
| `regex::egrep` | grep -E 格式 |

```cpp
// 指定 ECMAScript 语法（默认）
std::regex r1(pattern);

// 指定 POSIX extended 语法
std::regex r2(pattern, std::regex::extended);
```

### 常用匹配标志

```cpp
std::regex r(pattern, std::regex::icase);  // 忽略大小写
smatch match;
std::regex_search(str, match, r, std::regex::match_not_null);
```

---

## 8. 实用技巧

### 1. 转义字符

在正则表达式中，以下字符具有特殊含义，需要转义：

```
.  \  +  *  ?  ^  $  [  ]  {  }  |  (  )
```

```cpp
// 匹配点号
std::regex r("\\.");
```

### 2. 优先使用具体字符类

```cpp
// 避免过度使用 .
std::regex r1("\\d+\\.\\d+");  // 更好的写法
std::regex r2("\\d+.\\d+");     // . 匹配任意字符，不安全
```

### 3. 注意边界

```cpp
// 匹配单词而非数字
std::regex r("\\b\\d+\\b");  // 使用 \b 避免匹配到其他数字的一部分
```

### 4. 非捕获组提升性能

```cpp
// 不需要捕获时使用 (?:...)
std::regex r("(?:https?|ftp)://\\S+");  // 比 (https?|ftp) 稍快
```

---

## 9. 参考资料

[^1]: [MDN - 正则表达式](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Regular_Expressions)

[^2]: [C++ std::regex](https://en.cppreference.com/w/cpp/regex)

