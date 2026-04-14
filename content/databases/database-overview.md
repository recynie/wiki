---
title: 数据库系统概述
date: 2026-04-06
description: 数据库系统是存储、管理和处理数据的软件系统
tags:
  - database
  - fundamentals
draft: false
permalink:
---

# 数据库系统概述

数据库系统（Database Management System, DBMS）是用于存储、管理和处理大规模数据的软件系统。

## 数据库系统的组成

```
应用程序 ←→ DBMS ←→ 数据库
              ↓
         数据库管理员（DBA）
```

## 数据模型

### 层次模型

树形结构，IBM的IMS系统采用此模型。

### 网状模型

图结构，允许一个节点有多个父节点。

### 关系模型

以**二维表**（Relation）为核心的数据模型。

| student_id | name | age | major |
|------------|------|-----|-------|
| 001 | 张三 | 20 | CS |
| 002 | 李四 | 21 | Math |
| 003 | 王五 | 19 | Physics |

## SQL基础

### DDL（数据定义语言）

```sql
-- 创建表
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    age INT,
    major VARCHAR(50)
);

-- 修改表结构
ALTER TABLE students ADD COLUMN email VARCHAR(100);
ALTER TABLE students DROP COLUMN age;

-- 删除表
DROP TABLE students;
```

### DML（数据操作语言）

```sql
-- 插入数据
INSERT INTO students (student_id, name, major) 
VALUES (1, '张三', 'CS');

-- 查询数据
SELECT name, major FROM students WHERE major = 'CS';

-- 更新数据
UPDATE students SET name = '李四' WHERE student_id = 2;

-- 删除数据
DELETE FROM students WHERE student_id = 3;
```

### DCL（数据控制语言）

```sql
-- 授权
GRANT SELECT, INSERT ON students TO user1;

-- 撤销权限
REVOKE INSERT ON students FROM user1;
```

## 关系代数

### 基本运算

| 运算 | 符号 | 说明 |
|------|------|------|
| 选择 | $\sigma$ | 筛选行 |
| 投影 | $\pi$ | 筛选列 |
| 并 | $\cup$ | 行合并 |
| 差 | $-$ | 行相减 |
| 笛卡尔积 | $\times$ | 行组合 |
| 连接 | $\bowtie$ | 条件连接 |

### 示例

选择年龄大于20的学生：
$$
\sigma_{age > 20}(students)
$$

投影学生的姓名和专业：
$$
\pi_{name, major}(students)
$$

## 索引

### B+树索引

B+树是数据库最常用的索引结构：

```cpp
// B+树节点结构（简化）
struct BPlusNode {
    bool is_leaf;
    vector<int> keys;
    vector<BPlusNode*> children;  // 或叶子节点的指针
    BPlusNode* next;  // 叶子节点链表指针
    
    // 对于叶子节点，还存储数据指针
    vector<void*> values;
};
```

### 索引类型

| 类型 | 特点 | 适用场景 |
|------|------|---------|
| 主键索引 | 唯一非空 | 主键查询 |
| 唯一索引 | 唯一 | 唯一性约束 |
| 普通索引 | 可重复 | 加速查询 |
| 全文索引 | 文本搜索 | 文章内容 |
| 组合索引 | 多列 | 多列查询 |

## 事务

### ACID特性

| 特性 | 说明 |
|------|------|
| Atomicity（原子性） | 事务要么全做，要么全不做 |
| Consistency（一致性） | 事务执行前后数据库状态一致 |
| Isolation（隔离性） | 并发事务相互隔离 |
| Durability（持久性） | 事务提交后永久生效 |

### 并发控制

#### 锁机制

```cpp
// 简化的锁管理器
class LockManager {
    unordered_map<string, vector<string>> table_locks;  // 表级锁
    unordered_map<string, vector<string>> row_locks;     // 行级锁
    
public:
    void lock_shared(int transaction_id, const string& table) {
        table_locks[table].push_back("S" + to_string(transaction_id));
    }
    
    void lock_exclusive(int transaction_id, const string& table) {
        table_locks[table].push_back("X" + to_string(transaction_id));
    }
    
    bool can_lock(const string& table, const string& lock_type) {
        // 检查是否有冲突锁
        for (const auto& lock : table_locks[table]) {
            if (lock[0] == 'X' || lock_type[0] == 'X') return false;
        }
        return true;
    }
};
```

#### MVCC（多版本并发控制）

```cpp
struct Version {
    int version_id;
    int transaction_id;
    string data;
    bool committed;
    Version* next;
};

class MVCCStore {
    unordered_map<string, Version*> versions;  // key -> 版本链
    
public:
    string read(const string& key, int transaction_id) {
        Version* v = versions[key];
        while (v) {
            if (v->committed && v->transaction_id < transaction_id) {
                return v->data;
            }
            v = v->next;
        }
        return "";  // 未找到
    }
    
    void write(const string& key, const string& data, int transaction_id) {
        Version* new_v = new Version{version_id++, transaction_id, 
                                     data, false, versions[key]};
        versions[key] = new_v;
    }
};
```

## 规范化

### 范式级别

| 范式 | 要求 |
|------|------|
| 1NF | 字段不可再分 |
| 2NF | 消除部分依赖 |
| 3NF | 消除传递依赖 |
| BCNF | 消除主键依赖 |

### 示例

不符合2NF的表：
```
students_courses (student_id, course_id, student_name, course_name, grade)
                              ↑ 部分依赖     ↑ 部分依赖
```

分解为：
```
students (student_id, student_name)
courses (course_id, course_name)  
student_courses (student_id, course_id, grade)
```

## 查询优化

### 查询执行计划

```sql
EXPLAIN SELECT * FROM students WHERE major = 'CS';
```

### 优化器策略

1. **选择操作尽早执行**：减少中间结果
2. **投影操作尽早执行**：减少列数
3. **利用索引**：加速查找
4. **避免笛卡尔积**：使用连接条件

## 参考

- [数据库系统 - 维基百科](https://zh.wikipedia.org/wiki/数据库系统)
- [SQL教程 - W3Schools](https://www.w3schools.com/sql/)
