---
title: 高级SQL
date: 2026-04-10
description: 高级SQL技巧详解：窗口函数、公用表表达式、复杂查询、查询优化、事务隔离级别、存储过程
tags:
  - sql
  - database
  - query-optimization
draft: false
permalink:
---

# 高级SQL

高级SQL技巧是数据库开发中的核心能力，能够显著提升查询效率、简化复杂逻辑、实现层次化数据处理。本文详细介绍窗口函数、公用表表达式、复杂查询、查询优化、事务隔离级别及存储过程等高级主题。[^1]

---

## 窗口函数

窗口函数（Window Functions）是SQL高级特性的核心，它能在不分组的情况下计算累计值、排名和移动平均值。

### 语法结构

```sql
window_function(expression) OVER (
    [PARTITION BY column_list]
    [ORDER BY column_list]
    [ROWS/RANGE frame_clause]
)
```

### 排名函数

**ROW_NUMBER、RANK、DENSE_RANK** 的区别在于处理并列排名的方式：

```sql
-- 学生成绩表
CREATE TABLE scores (
    student_id INT,
    subject VARCHAR(20),
    score INT
);

-- 排名示例
SELECT 
    student_id,
    subject,
    score,
    ROW_NUMBER() OVER (PARTITION BY subject ORDER BY score DESC) AS row_num,
    RANK() OVER (PARTITION BY subject ORDER BY score DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY subject ORDER BY score DESC) AS dense_rank
FROM scores;
```

**结果对比**：

| student_id | subject | score | row_num | rank | dense_rank |
|------------|---------|-------|---------|------|------------|
| 001 | Math | 95 | 1 | 1 | 1 |
| 002 | Math | 95 | 2 | 2 | 2 |
| 003 | Math | 90 | 3 | 3 | 3 |

> **注意**：ROW_NUMBER 对并列行分配唯一序号；RANK 跳过后续排名；DENSE_RANK 紧凑连续排名。

### 前后函数

**LAG** 和 **LEAD** 用于访问当前行前后指定偏移量的数据，常用于计算环比、同比：

```sql
SELECT 
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month) AS prev_month_revenue,
    LEAD(revenue, 1) OVER (ORDER BY month) AS next_month_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY month) AS month_over_month_growth
FROM monthly_sales
ORDER BY month;
```

### FIRST_VALUE / LAST_VALUE

获取窗口内第一个或最后一个值：

```sql
SELECT
    order_date,
    customer_id,
    amount,
    FIRST_VALUE(amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_order_amount,
    LAST_VALUE(amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_order_amount
FROM orders;
```

> **注意**：使用 `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` 确保窗口包含整个分区，否则LAST_VALUE可能返回意外结果。

### NTILE 分桶

将数据均匀分成N组，便于进行分位数分析：

```sql
SELECT
    name,
    salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS quartile
FROM employees;
-- 将员工按薪资分成4组：1=最高薪，4=最低薪
```

---

## 公用表表达式（CTE）

公用表表达式（Common Table Expression，CTE）是一种临时命名的结果集，能简化复杂查询，提高可读性。

### 基本CTE

```sql
WITH high_value_customers AS (
    SELECT customer_id, SUM(order_amount) AS total_amount
    FROM orders
    GROUP BY customer_id
    HAVING SUM(order_amount) > 10000
)
SELECT c.name, c.total_amount
FROM customers c
JOIN high_value_customers hvc ON c.customer_id = hvc.customer_id;
```

### CTE vs 子查询 vs 视图

| 特性 | CTE | 子查询 | 视图 |
|------|-----|--------|------|
| 可读性 | 高 | 低 | 高 |
| 可复用 | 单次查询内 | 每次需重复 | 多次查询 |
| 索引支持 | 否 | 否 | 可建索引 |
| 物化 | 否 | 否 | 可物化 |

### 多阶段CTE管道

CTE支持多阶段数据处理，使复杂逻辑清晰化：

```sql
WITH
-- 阶段1：计算订单收入
order_revenue AS (
    SELECT
        order_id,
        customer_id,
        SUM(amount) AS revenue
    FROM order_items
    GROUP BY order_id, customer_id
),
-- 阶段2：计算客户统计
customer_stats AS (
    SELECT
        customer_id,
        COUNT(*) AS order_count,
        SUM(revenue) AS total_revenue,
        AVG(revenue) AS avg_order_value
    FROM order_revenue
    GROUP BY customer_id
),
-- 阶段3：客户分层
customer_tier AS (
    SELECT
        *,
        CASE
            WHEN order_count >= 10 THEN 'VIP'
            WHEN order_count >= 5 THEN 'Regular'
            ELSE 'New'
        END AS tier
    FROM customer_stats
)
SELECT tier, COUNT(*) AS customers
FROM customer_tier
GROUP BY tier;
```

### 递归CTE

递归CTE 是处理层次结构数据（如组织架构、树形分类）的利器，由**基础查询**和**递归查询**两部分组成：

```sql
-- 组织架构表
CREATE TABLE org_chart (
    employee_id INT,
    name VARCHAR(50),
    manager_id INT  -- 上级ID，NULL表示CEO
);

-- 递归查询：获取员工及其所有上级
WITH RECURSIVE employee_hierarchy AS (
    -- 基础查询：起点（CEO）
    SELECT employee_id, name, manager_id, 0 AS level
    FROM org_chart
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- 递归查询：Join上一级
    SELECT e.employee_id, e.name, e.manager_id, eh.level + 1
    FROM org_chart e
    JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy ORDER BY level, name;
```

**递归执行过程**：

$$
\text{Level}_0 \xrightarrow{\text{Join}} \text{Level}_1 \xrightarrow{\text{Join}} \text{Level}_2 \rightarrow \cdots
$$

**递归CTE执行原理**：
1. 首先执行基础查询（anchor member），结果放入临时结果集
2. 递归部分引用自身，如果新结果非空，则与上一步结果UNION ALL
3. 重复直到递归部分返回空结果
4. 为防止无限递归，SQL标准要求必须包含 `SEARCH` 或 `CYCLE` 条件

### 递归CTE计算层级深度

```sql
-- 计算每个部门的层级深度
WITH RECURSIVE dept_tree AS (
    SELECT dept_id, dept_name, parent_dept_id, 1 AS depth
    FROM departments
    WHERE parent_dept_id IS NULL
    
    UNION ALL
    
    SELECT d.dept_id, d.dept_name, d.parent_dept_id, dt.depth + 1
    FROM departments d
    JOIN dept_tree dt ON d.parent_dept_id = dt.dept_id
)
SELECT dept_id, dept_name, depth FROM dept_tree;
```

### 递归CTE典型应用场景

- **组织架构遍历**：从CEO到最底层员工
- **树形分类**：商品分类树、文件夹结构
- **路径追踪**：从起点到终点的路径
- **依赖解析**：软件包依赖关系、任务优先级

---

## 复杂查询

### 子查询

#### 标量子查询

返回单一值的子查询，可用于SELECT、WHERE、HAVING子句：

```sql
-- 找出销售额超过平均值的员工
SELECT e.name, e.sales
FROM employees e
WHERE e.sales > (SELECT AVG(sales) FROM employees);
```

#### 表子查询

返回结果集的子查询，常用于FROM子句：

```sql
-- 统计各部门高于平均绩效的员工数
SELECT dept, COUNT(*) AS high_performers
FROM (
    SELECT e.dept, e.name, e.performance
    FROM employees e
    WHERE e.performance > (SELECT AVG(performance) FROM employees WHERE dept = e.dept)
) AS high_perf
GROUP BY dept;
```

#### 关联子查询

子查询引用外层查询的列，根据外层每一行动态计算：

```sql
-- 找出每个类别中价格最高的产品
SELECT p1.category, p1.name, p1.price
FROM products p1
WHERE p1.price = (
    SELECT MAX(p2.price) 
    FROM products p2 
    WHERE p2.category = p1.category
);
```

### CROSS APPLY 与 OUTER APPLY

**CROSS APPLY** 类似于关联子查询的LATERAL JOIN，用于实现"每行执行一次子查询"：

```sql
-- 每个部门销售额最高的两名员工
SELECT d.dept_name, e.name, e.sales
FROM departments d
CROSS APPLY (
    SELECT TOP 2 name, sales
    FROM employees e
    WHERE e.dept_id = d.dept_id
    ORDER BY sales DESC
) e;
```

**OUTER APPLY** 类似，但在子查询无结果时返回NULL：

```sql
-- 每个部门的销售冠军（如果没有则显示NULL）
SELECT d.dept_name, e.name, e.sales
FROM departments d
OUTER APPLY (
    SELECT TOP 1 name, sales
    FROM employees e
    WHERE e.dept_id = d.dept_id AND e.sales > 5000
    ORDER BY sales DESC
) e;
```

> **CROSS APPLY vs JOIN**：CROSS APPLY 允许右表引用左表的列，适合复杂列计算；普通JOIN要求两边表独立。

### EXISTS vs IN

在判断子查询是否存在时，EXISTS 通常比 IN 更高效：

```sql
-- 找出有订单的客户
-- IN方式：先执行子查询构建列表，再匹配
SELECT * FROM customers c
WHERE c.customer_id IN (SELECT o.customer_id FROM orders o);

-- EXISTS方式：找到即返回，短路执行
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- NOT EXISTS vs NOT IN
-- NOT IN 如果子查询包含NULL，可能返回意外结果
-- NOT EXISTS 更可靠
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```

### 复杂条件查询

```sql
-- 多层嵌套的复杂查询：查找复购率超过50%的客户
WITH customer_orders AS (
    SELECT customer_id, COUNT(DISTINCT DATE_TRUNC('month', order_date)) AS order_months
    FROM orders
    WHERE order_date >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
    GROUP BY customer_id
),
customer_stats AS (
    SELECT co.customer_id,
           COUNT(DISTINCT o.order_id) AS total_orders,
           co.order_months
    FROM customer_orders co
    JOIN orders o ON co.customer_id = o.customer_id
    GROUP BY co.customer_id, co.order_months
)
SELECT c.*, cs.total_orders, cs.order_months
FROM customers c
JOIN customer_stats cs ON c.customer_id = cs.customer_id
WHERE cs.total_orders >= 3 
  AND cs.order_months >= cs.total_orders * 0.5;
```

---

## 查询优化

查询优化是提升数据库性能的核心技术。理解执行计划、掌握优化策略，能让SQL查询效率提升数十倍甚至数百倍。

### EXPLAIN分析执行计划

```sql
EXPLAIN SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date > '2024-01-01';
```

**关键字段解读**：

| 字段 | 说明 | 优化目标 |
|------|------|----------|
| type | 连接类型：const > eq_ref > ref > range > index > ALL | 越靠左越好 |
| key | 实际使用的索引 | 应有值，非NULL |
| rows | 预计扫描行数 | 越少越好 |
| Extra | Using index（覆盖索引）、Using filesort（需额外排序） | 避免filesort |

**type 连接类型详解**：
- `const`：主键或唯一索引等值查询，最多返回一行
- `eq_ref`：多表JOIN时，被驱动表使用主键或唯一索引
- `ref`：使用普通索引等值查询
- `range`：使用索引范围查询（ BETWEEN、IN、>、< ）
- `index`：全索引扫描
- `ALL`：全表扫描，应尽量避免

### 优化策略

#### 1. 避免全表扫描

```sql
-- 低效：全表扫描，YEAR()函数导致索引失效
SELECT * FROM orders WHERE YEAR(order_date) = 2024;

-- 高效：利用索引
SELECT * FROM orders 
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';
```

#### 2. 减少回表

```sql
-- 只需索引列时使用覆盖索引
SELECT order_id, order_date, total_amount 
FROM orders 
WHERE customer_id = 100;
-- 如果存在 idx_customer_order(customer_id, order_id, order_date, total_amount)，则无需回表
```

#### 3. 优化OR条件

```sql
-- 低效：OR导致索引失效
SELECT * FROM users WHERE email = 'x' OR phone = 'x';

-- 高效：改为UNION
SELECT * FROM users WHERE email = 'x'
UNION
SELECT * FROM users WHERE phone = 'x';
```

#### 4. 分页优化

```sql
-- 低效：大偏移量时性能差（MySQL）
SELECT * FROM orders ORDER BY order_id LIMIT 1000000, 10;

-- 高效：使用上一页最后ID（游标方式）
SELECT * FROM orders 
WHERE order_id > 1000000 
ORDER BY order_id 
LIMIT 10;
```

#### 5. 批量插入优化

```sql
-- 低效：逐条插入
INSERT INTO orders (id, amount) VALUES (1, 100);
INSERT INTO orders (id, amount) VALUES (2, 200);

-- 高效：批量插入
INSERT INTO orders (id, amount) VALUES (1, 100), (2, 200), (3, 300);
```

#### 6. 索引设计原则

```sql
-- 选择性高的列优先创建索引
SELECT COUNT(DISTINCT country) / COUNT(*) FROM customers;
-- 选择性越高，索引越有效

-- 复合索引遵循最左前缀原则
CREATE INDEX idx_dept_salary ON employees(dept_id, salary DESC);
-- 可加速 WHERE dept_id = 10 或 WHERE dept_id = 10 AND salary > 5000
-- 但无法加速 WHERE salary > 5000
```

---

## 事务进阶

事务是数据库操作的逻辑单元，确保ACID特性。深入理解隔离级别和锁机制，是解决并发问题的基础。

### ACID特性回顾

| 特性 | 说明 | 保证机制 |
|------|------|----------|
| Atomicity（原子性） | 事务要么全做，要么全不做 | Undo Log |
| Consistency（一致性） | 事务执行前后数据库状态一致 | 约束检查 |
| Isolation（隔离性） | 并发事务相互隔离 | 锁、MVCC |
| Durability（持久性） | 事务提交后永久生效 | Redo Log |

### 隔离级别详解

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|----------|------|-----------|------|----------|
| READ UNCOMMITTED | 可能 | 可能 | 可能 | 无锁 |
| READ COMMITTED | 不可能 | 可能 | 可能 | MVCC |
| REPEATABLE READ（默认） | 不可能 | 不可能 | 可能 | MVCC + 行锁 |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 | 锁表 |

### 脏读 vs 不可重复读 vs 幻读

- **脏读**：读取到其他事务**未提交**的数据
- **不可重复读**：同一事务中两次读取同一行数据结果不同（其他事务**修改**并提交）
- **幻读**：同一事务中两次查询返回的记录数不同（其他事务**插入/删除**）

```sql
-- 设置隔离级别
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

BEGIN TRANSACTION;
-- 事务操作
COMMIT;
```

### MVCC（多版本并发控制）

MVCC通过保存数据的多个版本，使读写操作不阻塞：

```cpp
// 简化版MVCC实现
struct Version {
    int txn_id;        // 创建版本的事务ID
    int begin_ts;      // 开始时间戳
    int end_ts;        // 结束时间戳（-1表示无限）
    string data;      // 数据值
    bool committed;   // 是否已提交
};

string read(string key, int txn_id) {
    auto& versions = get_versions(key);
    for (auto it = versions.rbegin(); it != versions.rend(); ++it) {
        if (it->begin_ts <= txn_id && 
            (it->end_ts > txn_id || it->end_ts == -1) &&
            it->committed) {
            return it->data;  // 读取已提交的最新版本
        }
    }
    return "";
}
```

**READ COMMITTED vs REPEATABLE READ**：
- READ COMMITTED：每次读取都获取最新已提交版本
- REPEATABLE READ：整个事务使用同一个版本（事务开始时的版本）

### Next-Key Locking

InnoDB的REPEATABLE READ隔离级别使用Next-Key Locking防止幻读：

```sql
-- 事务A：
SELECT * FROM orders WHERE amount > 100 FOR UPDATE;
-- 锁定 amount > 100 的所有行及区间间隙

-- 事务B尝试插入 amount = 150 的订单将被阻塞
INSERT INTO orders (amount) VALUES (150);
```

**Next-Key Locking = 行锁 + 间隙锁**：
- 行锁：锁定已存在的记录
- 间隙锁：锁定索引之间的"间隙"，防止插入

### 死锁处理

死锁发生在两个或多个事务相互等待对方持有的锁时：

```sql
-- 查看死锁日志
SHOW ENGINE INNODB STATUS;

-- 解决策略：
-- 1. 按固定顺序访问表/行（避免循环等待）
-- 2. 减少事务持有锁的时间（尽早提交）
-- 3. 使用低隔离级别
-- 4. 设置锁等待超时：SET innodb_lock_wait_timeout = 5;
```

**死锁示例**：

```
事务A：LOCK X ON orders WHERE id=1 → 等待 id=2
事务B：LOCK X ON orders WHERE id=2 → 等待 id=1
→ 死锁，InnoDB会自动回滚代价最小的事务
```

### 乐观锁 vs 悲观锁

```sql
-- 悲观锁：假设并发冲突，事务开始即加锁
SELECT * FROM orders WHERE id = 1 FOR UPDATE;

-- 乐观锁：假设冲突少，提交时检查版本号
UPDATE orders 
SET amount = 100, version = version + 1 
WHERE id = 1 AND version = 5;  -- 版本不匹配则更新失败
```

---

## 存储过程

存储过程和函数是数据库中存储和执行业务逻辑的方式，能减少应用与数据库的网络交互，提高性能。

### 基本语法

```sql
DELIMITER //

CREATE PROCEDURE get_customer_orders(
    IN p_customer_id INT,
    OUT p_total_amount DECIMAL(10,2)
)
BEGIN
    -- 计算总金额
    SELECT SUM(amount) INTO p_total_amount
    FROM orders
    WHERE customer_id = p_customer_id;
    
    -- 返回订单列表
    SELECT order_id, order_date, amount
    FROM orders
    WHERE customer_id = p_customer_id
    ORDER BY order_date DESC;
END //

DELIMITER ;

-- 调用存储过程
CALL get_customer_orders(1001, @total);
SELECT @total AS total_spent;
```

**参数类型**：
- `IN`：输入参数（默认）
- `OUT`：输出参数
- `INOUT`：既输入又输出

### 条件判断

```sql
DELIMITER //

CREATE PROCEDURE classify_customer(
    IN p_customer_id INT,
    OUT p_tier VARCHAR(20)
)
BEGIN
    DECLARE v_total DECIMAL(10,2);
    
    SELECT SUM(amount) INTO v_total
    FROM orders
    WHERE customer_id = p_customer_id;
    
    IF v_total >= 100000 THEN
        SET p_tier = 'PLATINUM';
    ELSEIF v_total >= 50000 THEN
        SET p_tier = 'GOLD';
    ELSEIF v_total >= 10000 THEN
        SET p_tier = 'SILVER';
    ELSE
        SET p_tier = 'BRONZE';
    END IF;
END //

DELIMITER ;
```

### 循环处理

```sql
DELIMITER //

CREATE PROCEDURE process_batch()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_id INT;
    DECLARE cursor1 CURSOR FOR SELECT id FROM pending_orders;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    OPEN cursor1;
    
    read_loop: LOOP
        FETCH cursor1 INTO v_id;
        IF done THEN
            LEAVE read_loop;
        END IF;
        
        -- 处理每条记录
        UPDATE orders SET status = 'PROCESSED' WHERE id = v_id;
    END LOOP;
    
    CLOSE cursor1;
END //

DELIMITER ;
```

### 函数

函数与存储过程的主要区别：**函数必须有返回值**，常用于计算：

```sql
DELIMITER //

CREATE FUNCTION get_level(p_salary DECIMAL(10,2))
RETURNS VARCHAR(20)
DETERMINISTIC
BEGIN
    DECLARE p_level VARCHAR(20);
    
    IF p_salary >= 50000 THEN SET p_level = 'SENIOR';
    ELSEIF p_salary >= 30000 THEN SET p_level = 'MIDDLE';
    ELSE SET p_level = 'JUNIOR';
    END IF;
    
    RETURN p_level;
END //

DELIMITER ;

-- 在查询中使用函数
SELECT name, salary, get_level(salary) AS level FROM employees;

-- 在WHERE子句中使用
SELECT * FROM employees WHERE get_level(salary) = 'SENIOR';
```

### 触发器

触发器是在表上自动执行的代码，响应 INSERT、UPDATE、DELETE 事件：

```sql
-- 审计日志表
CREATE TABLE audit_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    action VARCHAR(20),
    table_name VARCHAR(50),
    record_id INT,
    old_value JSON,
    new_value JSON,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- INSERT触发器
DELIMITER //

CREATE TRIGGER trg_orders_insert
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (action, table_name, record_id, new_value)
    VALUES ('INSERT', 'orders', NEW.order_id, 
            JSON_OBJECT('amount', NEW.amount, 'customer_id', NEW.customer_id));
END //

-- UPDATE触发器
CREATE TRIGGER trg_orders_update
AFTER UPDATE ON orders
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (action, table_name, record_id, old_value, new_value)
    VALUES ('UPDATE', 'orders', OLD.order_id, 
            JSON_OBJECT('amount', OLD.amount),
            JSON_OBJECT('amount', NEW.amount));
END //

-- DELETE触发器
CREATE TRIGGER trg_orders_delete
BEFORE DELETE ON orders
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (action, table_name, record_id, old_value)
    VALUES ('DELETE', 'orders', OLD.order_id, 
            JSON_OBJECT('amount', OLD.amount));
END //

DELIMITER ;
```

### 存储过程的优缺点

**优点**：
- 减少网络往返：一次调用完成多个SQL操作
- 模块化：将业务逻辑封装在数据库层
- 性能优化：数据库内部执行，速度快

**缺点**：
- 可移植性差：不同数据库语法差异大
- 版本控制困难：难以测试和调试
- 扩展性受限：数据库服务器资源有限

---

## 参考资料

[^1]: 本词条综合参考[SQL Window Functions](https://www.postgresql.org/docs/current/functions-window.html)、[Common Table Expressions](https://docs.microsoft.com/en-us/sql/t-sql/queries/with-common-table-expression-transact-sql)、[InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)

---

## 相关主题

- [[database-overview|数据库系统概述]] - SQL基础、事务ACID
- [[database-index-principles|数据库索引原理]] - B+树、查询优化、隔离级别实现
