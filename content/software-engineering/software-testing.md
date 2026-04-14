---
title: 软件测试方法与实践
date: 2026-04-13
description: 单元测试、集成测试、系统测试策略与测试驱动开发
tags:
  - software-engineering
  - testing
  - tdd
  - quality
draft: false
permalink:
---

# 软件测试方法与实践

软件测试是保障软件质量的核心手段，贯穿软件开发生命周期全流程。

## 测试金字塔

```
                    ┌─────────┐
                    │  E2E   │  ← 少量，耗时，昂贵
                    │  Tests │
                   ├─────────┤
                   │Integration│ ← 中等数量
                   │  Tests  │
                  ├──────────┤
                  │   Unit   │  ← 大量，快速，廉价
                  │  Tests  │
                 └──────────┘
```

| 层级 | 占比 | 特点 | 工具 |
|------|------|------|------|
| E2E | 10% | 模拟真实用户场景 | Selenium, Cypress |
| 集成 | 20% | 模块间交互验证 | pytest, JUnit |
| 单元 | 70% | 最小代码单元 | GTest, unittest |

## 单元测试

### C++单元测试框架

```cpp
#include <gtest/gtest.h>

class Calculator {
public:
    int add(int a, int b) { return a + b; }
    int divide(int a, int b) {
        if (b == 0) throw std::invalid_argument("Division by zero");
        return a / b;
    }
};

// 测试类
class CalculatorTest : public ::testing::Test {
protected:
    Calculator* calc;
    
    void SetUp() override {
        calc = new Calculator();
    }
    
    void TearDown() override {
        delete calc;
    }
};

// 测试用例
TEST_F(CalculatorTest, AddPositiveNumbers) {
    EXPECT_EQ(calc->add(2, 3), 5);
}

TEST_F(CalculatorTest, AddNegativeNumbers) {
    EXPECT_EQ(calc->add(-1, -1), -2);
}

TEST_F(CalculatorTest, AddZeros) {
    EXPECT_EQ(calc->add(0, 0), 0);
}

TEST_F(CalculatorTest, DividePositiveNumbers) {
    EXPECT_EQ(calc->divide(10, 2), 5);
}

TEST_F(CalculatorTest, DivideByZero) {
    EXPECT_THROW(calc->divide(10, 0), std::invalid_argument);
}

int main(int argc, char **argv) {
    ::testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

### Python单元测试

```python
import unittest
from src.calculator import Calculator

class TestCalculator(unittest.TestCase):
    def setUp(self):
        self.calc = Calculator()
    
    def test_add_positive_numbers(self):
        self.assertEqual(self.calc.add(2, 3), 5)
    
    def test_add_negative_numbers(self):
        self.assertEqual(self.calc.add(-1, -1), -2)
    
    def test_divide_by_zero(self):
        with self.assertRaises(ValueError):
            self.calc.divide(10, 0)
    
    # 参数化测试
    @unittest.parametrize("a,b,expected", [
        (1, 2, 3),
        (0, 0, 0),
        (-1, 1, 0),
        (100, 200, 300),
    ])
    def test_add_parameterized(self, a, b, expected):
        self.assertEqual(self.calc.add(a, b), expected)


class TestStringOperations(unittest.TestCase):
    def test_strip_whitespace(self):
        self.assertEqual("  hello  ".strip(), "hello")
    
    def test_split(self):
        self.assertEqual("a,b,c".split(","), ["a", "b", "c"])
    
    def test_join(self):
        self.assertEqual(",".join(["a", "b", "c"]), "a,b,c")

if __name__ == '__main__':
    unittest.main()
```

## 集成测试

### 模块间接口测试

```python
import pytest
from unittest.mock import Mock, patch

# 被测试模块
class UserService:
    def __init__(self, user_repo, email_service):
        self.user_repo = user_repo
        self.email_service = email_service
    
    def create_user(self, username, email):
        if self.user_repo.find_by_username(username):
            raise ValueError("Username exists")
        if self.user_repo.find_by_email(email):
            raise ValueError("Email exists")
        
        user = self.user_repo.create(username, email)
        self.email_service.send_welcome(email)
        return user


# 集成测试
class TestUserService:
    def test_create_user_success(self):
        # Mock依赖
        mock_repo = Mock()
        mock_email = Mock()
        
        # 配置mock行为
        mock_repo.find_by_username.return_value = None
        mock_repo.find_by_email.return_value = None
        mock_repo.create.return_value = {"id": 1, "username": "test", "email": "test@example.com"}
        
        service = UserService(mock_repo, mock_email)
        user = service.create_user("test", "test@example.com")
        
        # 验证交互
        mock_repo.create.assert_called_once()
        mock_email.send_welcome.assert_called_once_with("test@example.com")
        assert user["username"] == "test"
    
    def test_create_user_duplicate_username(self):
        mock_repo = Mock()
        mock_email = Mock()
        mock_repo.find_by_username.return_value = {"id": 1, "username": "test"}
        
        service = UserService(mock_repo, mock_email)
        
        with pytest.raises(ValueError, match="Username exists"):
            service.create_user("test", "new@example.com")
        
        mock_repo.create.assert_not_called()
        mock_email.send_welcome.assert_not_called()
```

### 数据库集成测试

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture
def test_db():
    # 创建测试数据库（内存SQLite）
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    Session = sessionmaker(bind=engine)
    session = Session()
    yield session
    session.close()

def test_user_repository_crud(test_db):
    from models import User
    from repositories import UserRepository
    
    repo = UserRepository(test_db)
    
    # Create
    user = repo.create("alice", "alice@example.com")
    assert user.id is not None
    assert user.username == "alice"
    
    # Read
    found = repo.find_by_id(user.id)
    assert found.username == "alice"
    
    # Update
    found.username = "alice_updated"
    updated = repo.update(found)
    assert updated.username == "alice_updated"
    
    # Delete
    repo.delete(updated.id)
    assert repo.find_by_id(user.id) is None
```

## 系统测试

### API端到端测试

```python
import requests
import pytest

BASE_URL = "http://localhost:8000/api"

class TestUserAPI:
    @pytest.fixture(autouse=True)
    def setup(self):
        self.base_url = f"{BASE_URL}/users"
    
    def test_create_user(self):
        payload = {
            "username": "testuser",
            "email": "test@example.com",
            "password": "secure123"
        }
        
        response = requests.post(self.base_url, json=payload)
        
        assert response.status_code == 201
        data = response.json()
        assert data["username"] == "testuser"
        assert "id" in data
        assert "password" not in data  # 敏感信息不返回
    
    def test_get_user(self):
        # 先创建
        create_response = requests.post(self.base_url, json={
            "username": "gettest",
            "email": "gettest@example.com",
            "password": "pass123"
        })
        user_id = create_response.json()["id"]
        
        # 再获取
        response = requests.get(f"{self.base_url}/{user_id}")
        
        assert response.status_code == 200
        assert response.json()["username"] == "gettest"
    
    def test_user_not_found(self):
        response = requests.get(f"{self.base_url}/99999")
        assert response.status_code == 404
```

## 测试驱动开发（TDD）

### Red-Green-Refactor循环

```
1. 写一个失败的测试（Red）
2. 写最少量代码使测试通过（Green）
3. 重构代码（Refactor）
```

```python
# TDD示例：实现一个Stack

# Step 1: 写失败的测试
class TestStack:
    def test_push_and_pop(self):
        stack = Stack()
        stack.push(1)
        stack.push(2)
        assert stack.pop() == 2
        assert stack.pop() == 1
    
    def test_pop_from_empty_raises(self):
        stack = Stack()
        with pytest.raises(IndexError):
            stack.pop()
    
    def test_peek(self):
        stack = Stack()
        stack.push(1)
        stack.push(2)
        assert stack.peek() == 2
        assert len(stack) == 2  # peek不移除
    
    def test_is_empty(self):
        stack = Stack()
        assert stack.is_empty() is True
        stack.push(1)
        assert stack.is_empty() is False

# Step 2: 实现（最少量代码）
class Stack:
    def __init__(self):
        self._items = []
    
    def push(self, item):
        self._items.append(item)
    
    def pop(self):
        if not self._items:
            raise IndexError("pop from empty stack")
        return self._items.pop()
    
    def peek(self):
        if not self._items:
            raise IndexError("peek from empty stack")
        return self._items[-1]
    
    def is_empty(self):
        return len(self._items) == 0
    
    def __len__(self):
        return len(self._items)
```

## Mock与Stub

```python
from unittest.mock import Mock, MagicMock, patch

# Mock：验证调用行为
def test_order_sends_email():
    mock_email_service = Mock()
    
    order = Order(mock_email_service)
    order.place()
    
    mock_email_service.send_confirmation.assert_called_once()

# Patch：替换模块级函数
@patch('app.utils.send_email')
def test_password_reset(mock_send):
    mock_send.return_value = True
    
    result = reset_password("user@example.com")
    
    assert result is True
    mock_send.assert_called_with("user@example.com", expect_)

# MagicMock：自动属性
def test_cache():
    cache = MagicMock()
    cache.get.return_value = None  # 首次返回None
    cache.get.return_value = "cached_data"  # 之后返回缓存值
    
    assert cache.get("key") is None
    assert cache.get("key") == "cached_data"
```

## 代码覆盖率

```bash
# pytest + coverage
pytest --cov=src --cov-report=html tests/

# 输出报告
# coverage report:
# Name              Stmts   Miss  Cover
# ---------------------------------------
# src/models.py        50      5    90%
# src/views.py         80     20    75%
# ---------------------------------------
# TOTAL              130     25    81%
```

## 参考

[^1]: Martin Fowler, Refactoring
[^2]: Kent Beck, Test Driven Development: By Example

