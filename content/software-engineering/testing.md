---
title: Software Testing
date: 2026-04-14
description: 软件测试方法论：TDD、BDD 与测试策略
tags:
  - testing
  - tdd
  - bdd
  - quality
draft: true
permalink:
---

# Software Testing

## 概述

软件测试是通过执行程序来发现错误的系统性方法。测试不仅验证功能正确性，更是代码行为的文档、设计的反馈、以及重构的保障。

## 测试金字塔

测试金字塔定义了不同层次测试的配比：

```
                    ┌───────────┐
                    │    E2E    │  少量、慢、覆盖关键路径
                    │   Tests   │
                   ├───────────┤
                   │ Integration│  中等规模、验证组件协作
                   │   Tests    │
                  ├─────────────┤
                  │    Unit     │  大量、快速、隔离测试
                  │   Tests     │
                 └──────────────┘
```

| 层级 | 数量 | 速度 | 隔离度 | 覆盖范围 |
|------|------|------|--------|----------|
| E2E | 少 | 慢（分钟级）| 低 | 关键用户路径 |
| Integration | 中 | 中（秒级）| 中 | 组件接口 |
| Unit | 多 | 快（毫秒级）| 高 | 函数/类逻辑 |

## TDD（Test-Driven Development）

TDD 是「先写测试，再写实现」的开发方法论。

### 红-绿-重构循环

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│   RED   │ ──▶ │  GREEN  │ ──▶ │ REFACTOR │
│  写测试  │     │  让测试  │     │ 改进代码  │
│  失败    │     │  通过    │     │ 保测试通过 │
└─────────┘     └─────────┘     └─────────┘
```

**步骤**：

1. **Red**：编写一个描述期望行为的测试，运行确认失败
2. **Green**：编写最小代码使测试通过
3. **Refactor**：重构代码，保持测试通过

### TDD 优势

- **设计驱动**：强制思考接口设计
- **即时反馈**：立即发现错误
- **测试覆盖**：代码必有测试
- **安全重构**：测试作为安全网

### TDD 示例

```python
# Step 1: Red - 编写测试（假设尚未实现）
def test_calculator_add():
    calc = Calculator()
    result = calc.add(2, 3)
    assert result == 5

# Step 2: Green - 最小实现
class Calculator:
    def add(self, a, b):
        return 5  # Hardcoded, but passes!

# Step 3: Refactor - 真正实现
class Calculator:
    def add(self, a, b):
        return a + b  # 正确实现
```

## BDD（Behavior-Driven Development）

BDD 是 TDD 的扩展，强调用自然语言描述系统行为，促进技术/非技术沟通。

### Gherkin 语法

```gherkin
Feature: 用户登录
  作为注册用户
  我想登录系统
  以便访问我的个人数据

  Scenario: 正确的用户名和密码
    Given 我在登录页面
    And 用户名输入框为空
    When 我输入用户名 "testuser"
    And 我输入密码 "password123"
    And 我点击登录按钮
    Then 我应该看到欢迎消息
    And 我应该被重定向到首页

  Scenario: 错误的密码
    Given 我在登录页面
    When 我输入用户名 "testuser"
    And 我输入密码 "wrongpassword"
    And 我点击登录按钮
    Then 我应该看到错误提示 "用户名或密码错误"
    And 我应该保持在登录页面
```

### BDD 框架

| 语言 | 框架 |
|------|------|
| Python | Behave, pytest-bdd |
| JavaScript | Cucumber.js, Mocha |
| Java | Cucumber-JVM, JBehave |
| Go | Godog, Ginkgo |

### BDD 流程

```
业务分析 ──▶ 特性建模 ──▶ 自动化脚本 ──▶ 持续执行
   │                                    │
   └──────── 反馈循环 ◀─────────────────┘
```

## 单元测试（Unit Testing）

### 原则（F.I.R.S.T）

- **Fast**：测试应快速执行
- **Independent**：测试间相互独立
- **Repeatable**：可重复执行，结果一致
- **Self-validating**：自动判断通过/失败
- **Timely**：测试先行（TDD）

### 测试结构（AAA 模式）

```python
def test_user_registration():
    # Arrange - 准备测试数据
    db = InMemoryDatabase()
    service = UserService(db)
    
    # Act - 执行被测操作
    user = service.register("test@example.com", "password123")
    
    # Assert - 验证结果
    assert user.email == "test@example.com"
    assert user.is_active is True
    assert len(db.users) == 1
```

### Mock 与 Stub

```python
from unittest.mock import Mock, patch

def test_order_notification():
    # Stub - 替代依赖，提供固定返回值
    payment_gateway = Mock()
    payment_gateway.process.return_value = True
    
    # Mock - 验证调用行为
    notifier = Mock()
    
    order = Order(payment=payment_gateway, notifier=notifier)
    order.complete()
    
    # 验证 mock 被正确调用
    notifier.send.assert_called_once()
    notifier.send.assert_called_with(
        subject="Order Complete",
        recipient=order.customer_email
    )
```

### 边界条件测试

```python
# 等价类划分
def test_calculate_discount():
    # 正常情况
    assert calculate_discount(100, 0.1) == 10
    
    # 边界值
    assert calculate_discount(0, 0.1) == 0
    assert calculate_discount(100, 0) == 0
    
    # 异常情况
    with pytest.raises(ValueError):
        calculate_discount(-100, 0.1)  # 负数金额
    
    with pytest.raises(ValueError):
        calculate_discount(100, 1.5)  # 超过100%的折扣
```

## 集成测试（Integration Testing）

### 组件协作测试

```python
import pytest

@pytest.fixture
def order_service():
    return OrderService(
        payment_gateway=RealPaymentGateway(),  # 使用真实支付网关测试
        inventory=RealInventory(),
        notification=MockNotification()
    )

def test_complete_order_flow(order_service):
    # 准备库存
    order_service.inventory.add("SKU001", 10)
    
    # 创建订单
    order = order_service.create_order("customer@test.com", ["SKU001"])
    
    # 完成订单
    order_service.complete_order(order.id, payment_token="tok_test")
    
    # 验证完整流程
    assert order.status == "completed"
    assert order_service.inventory.get_stock("SKU001") == 9
```

### 数据库测试

```python
@pytest.fixture
def db_session():
    """创建测试数据库会话"""
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    Session = sessionmaker(bind=engine)
    session = Session()
    yield session
    session.close()

def test_user_crud(db_session):
    repo = UserRepository(db_session)
    
    # Create
    user = User(name="Alice", email="alice@test.com")
    repo.save(user)
    
    # Read
    found = repo.find_by_email("alice@test.com")
    assert found.name == "Alice"
    
    # Update
    found.name = "Bob"
    repo.save(found)
    assert repo.find_by_id(found.id).name == "Bob"
    
    # Delete
    repo.delete(found.id)
    assert repo.find_by_id(found.id) is None
```

## 端到端测试（E2E Testing）

### Selenium/Playwright 示例

```python
from playwright.sync_api import page, expect

def test_user_login_flow():
    page.goto("https://example.com/login")
    
    # 填写表单
    page.get_by_label("用户名").fill("testuser")
    page.get_by_label("密码").fill("password123")
    page.get_by_role("button", name="登录").click()
    
    # 验证结果
    expect(page).to_have_url(/.*dashboard/)
    expect(page.get_by_text("欢迎回来")).to_be_visible()
```

### Cypress 示例

```javascript
describe('Login Flow', () => {
  it('should login with valid credentials', () => {
    cy.visit('/login')
    cy.get('[data-testid=username]').type('testuser')
    cy.get('[data-testid=password]').type('password123')
    cy.get('[data-testid=login-button]').click()
    
    cy.url().should('include', '/dashboard')
    cy.contains('Welcome back').should('be.visible')
  })
})
```

## 测试策略

### 按风险优先级测试

| 优先级 | 测试类型 | 执行频率 |
|--------|----------|----------|
| P0 | 核心功能冒烟测试 | 每次提交 |
| P1 | 关键路径单元测试 | 每次 PR |
| P2 | 功能回归测试 | 每日构建 |
| P3 | 完整测试套件 | 每周发布前 |

### 持续测试

```yaml
# CI 配置示例
test:
  script:
    - pytest tests/unit -v --tb=short
    - pytest tests/integration -v
    - playwright test e2e
  coverage:
    script:
      - pytest --cov=src --cov-report=xml
    artifacts:
      reports:
        coverage:
          files: coverage.xml
```

## 测试覆盖率

### 覆盖率指标

- **Line Coverage**：代码行被执行的比例
- **Branch Coverage**：条件分支被覆盖的比例
- **Function Coverage**：函数被调用的比例
- **Statement Coverage**：语句被执行的比例

```python
# pytest-cov 使用
pytest --cov=src --cov-report=html tests/

# 覆盖率报告
Name              Stmts   Miss  Cover
-------------------------------------
src/user.py          50      5    90%
src/order.py         80     20    75%
-------------------------------------
TOTAL               130     25    81%
```

### 覆盖率陷阱

> 90% 覆盖率不等于 90% 的质量。

- 可以覆盖但未验证正确性
- 不代表边界条件被测试
- 不代表真实使用场景被覆盖

## 测试替身（Test Doubles）

| 类型 | 用途 | 示例 |
|------|------|------|
| Dummy | 仅满足接口签名，不使用返回值 | 空集合、空字符串 |
| Stub | 提供固定返回值 | Mock 固定数据 |
| Spy | 记录调用信息 | 验证方法被调用 |
| Mock | 预设期望行为和断言 | 验证交互 |
| Fake | 有简化实现 | InMemoryDatabase |

## 扩展阅读

- [测试金字塔](https://martinfowler.com/articles/practical-test-pyramid.html)
- [xUnit Test Patterns](https://www.amazon.com/xUnit-Test-Patterns-Refactoring-Code/dp/0131495054)
- [BDD in Action](https://www.manning.com/books/bdd-in-action)
- [Google Testing Blog](https://testing.googleblog.com/)

