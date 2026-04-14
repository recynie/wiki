---
title: Python进阶
date: 2026-04-11
description: Python高级编程技巧：装饰器、元编程、异步编程
tags:
  - python
  - advanced-programming
  - async
draft: false
permalink:
---

## 概述

Python进阶内容涵盖装饰器、元编程和异步编程三大核心主题。这些技术是构建高性能、可扩展Python应用的基础，也是区分初级与中高级Python开发者的关键技能。

## 装饰器

装饰器是Python最强大的元编程工具之一，允许在**不修改原函数代码**的情况下动态增强函数行为。其本质是一个高阶函数，接受被装饰函数作为参数并返回一个新包装函数。[^1]

### 装饰器基础

```python
import functools
import time

def timer(func):
    """计时装饰器"""
    @functools.wraps(func)  # 保留原函数元数据
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} 执行耗时: {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(0.5)
    return "完成"
```

### 带参数的装饰器工厂

装饰器工厂返回装饰器，实现参数化配置：

```python
def repeat(times=1):
    """重复执行指定次数"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            results = []
            for _ in range(times):
                results.append(func(*args, **kwargs))
            return results
        return wrapper
    return decorator

@repeat(times=3)
def greet(name):
    return f"Hello, {name}!"

print(greet("Alice"))  # ['Hello, Alice!', 'Hello, Alice!', 'Hello, Alice!']
```

### 类装饰器

装饰器不仅可用于函数，也可用于类：

```python
def singleton(cls):
    """单例模式装饰器"""
    instances = {}
    @functools.wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
class Database:
    def __init__(self):
        self.connection = "connected"
```

### 异步装饰器

装饰异步函数时，包装器也必须是异步的：

```python
import asyncio
import functools

def async_logger(func):
    @functools.wraps(func)
    async def wrapper(*args, **kwargs):
        print(f"调用异步函数: {func.__name__}")
        try:
            result = await func(*args, **kwargs)
            print(f"{func.__name__} 执行成功")
            return result
        except Exception as e:
            print(f"{func.__name__} 执行失败: {e}")
            raise
    return wrapper

@async_logger
async def fetch_data(url):
    await asyncio.sleep(0.1)
    return f"数据: {url}"
```

处理同步/异步统一的装饰器：

```python
def universal_timer(prefix="[Timer]"):
    """同时支持同步和异步函数的计时器"""
    def decorator(func):
        @functools.wraps(func)
        async def async_wrapper(*args, **kwargs):
            start = time.perf_counter()
            result = await func(*args, **kwargs)
            print(f"{prefix} {func.__name__}: {time.perf_counter()-start:.4f}s")
            return result
        
        @functools.wraps(func)
        def sync_wrapper(*args, **kwargs):
            start = time.perf_counter()
            result = func(*args, **kwargs)
            print(f"{prefix} {func.__name__}: {time.perf_counter()-start:.4f}s")
            return result
        
        if asyncio.iscoroutinefunction(func):
            return async_wrapper
        return sync_wrapper
    return decorator
```

### 常用内置装饰器

Python标准库提供了许多实用的内置装饰器：

| 装饰器 | 模块 | 用途 |
|--------|------|------|
| `@lru_cache` | functools | 函数结果缓存 |
| `@cached_property` | functools | 类属性缓存 |
| `@dataclass` | dataclasses | 自动生成特殊方法 |
| `@singledispatch` | functools | 函数重载 |
| `@property` | 内置 | 属性访问控制 |

```python
from functools import lru_cache, singledispatch
from dataclasses import dataclass

@lru_cache(maxsize=128)
def fibonacci(n):
    """斐波那契数列（带缓存）"""
    return n if n < 2 else fibonacci(n-1) + fibonacci(n-2)

@singledispatch
def process(data):
    """根据类型分发处理"""
    raise NotImplementedError(f"不支持的类型: {type(data)}")

@process.register(int)
def _(data):
    return data * 2

@process.register(str)
def _(data):
    return data.upper()
```

## 元编程

元编程指程序能够操**作自身代码**的能力，包括修改、生成或扩展代码。Python中元编程的主要手段包括元类和`__init_subclass__`。[^2]

### 元类（Metaclass）

元类控制类的创建过程，是Python中最强大的元编程工具：

```python
class ModelMeta(type):
    """ORM元类示例"""
    _registry = {}
    
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if hasattr(cls, '_table_name'):
            mcs._registry[cls._table_name] = cls
        return cls

class User(metaclass=ModelMeta):
    _table_name = 'users'
    
    def __init__(self, id, name, email):
        self.id = id
        self.name = name
        self.email = email

print(User._table_name)  # 'users'
print(ModelMeta._registry)  # {'users': <class 'User'>}
```

### `__init_subclass__`

比元类更轻量的类注册模式：

```python
class PluginBase:
    _plugins = []
    
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        if hasattr(cls, 'name'):
            PluginBase._plugins.append(cls)

class AuthPlugin(PluginBase):
    name = "auth"
    
class LogPlugin(PluginBase):
    name = "logger"

print([p.name for p in PluginBase._plugins])  # ['auth', 'logger']
```

### 动态创建类与属性访问

```python
# 动态创建类
def create_model(table_name, fields):
    return type(table_name, (object,), {
        '_table_name': table_name,
        **{f: None for f in fields}
    })

User = create_model('users', ['id', 'name', 'email'])
user = User()
user.name = "Alice"
```

## 异步编程

Python的`asyncio`提供了基于协程的异步I/O编程模型，适用于高并发网络应用。[^3]

### 协程与async/await

```python
import asyncio

async def fetch(url):
    """模拟异步网络请求"""
    await asyncio.sleep(0.1)  # 模拟I/O延迟
    return f"响应: {url}"

async def main():
    # 创建多个并发任务
    urls = [f"http://api.example.com/{i}" for i in range(5)]
    tasks = [fetch(url) for url in urls]
    
    # 并发执行
    results = await asyncio.gather(*tasks)
    for r in results:
        print(r)

asyncio.run(main())
```

### 异步上下文管理器

```python
class AsyncDBConnection:
    async def __aenter__(self):
        print("建立连接...")
        await asyncio.sleep(0.01)
        return self
    
    async def __aexit__(self, exc_type, exc, tb):
        print("关闭连接...")
        await asyncio.sleep(0.01)

async def query():
    async with AsyncDBConnection() as conn:
        await asyncio.sleep(0.1)
        return "查询结果"
```

### 异步生成器

```python
async def async_data_stream(n):
    """异步数据流"""
    for i in range(n):
        await asyncio.sleep(0.1)
        yield i

async def main():
    async for item in async_data_stream(5):
        print(f"收到数据: {item}")

asyncio.run(main())
```

### 异步vs同步陷阱

**常见错误**：在异步函数中使用同步阻塞操作：

```python
# ❌ 错误：阻塞事件循环
async def bad_example():
    time.sleep(5)  # 同步sleep，阻塞整个事件循环

# ✅ 正确：使用异步sleep
async def good_example():
    await asyncio.sleep(5)
```

**混合方案**：使用`asyncio.to_thread()`包装同步阻塞代码：

```python
async def call_sync_code():
    # 将同步代码放到线程池执行
    result = await asyncio.to_thread(blocking_io_function)
    return result
```

### 生产级异步模式

```python
import asyncio
from asyncio import Queue

async def producer(queue: Queue, item_id: int):
    await queue.put(f"item_{item_id}")
    print(f"生产者 {item_id} 已生产")

async def consumer(queue: Queue, consumer_id: int):
    while True:
        item = await queue.get()
        print(f"消费者 {consumer_id} 消费: {item}")
        queue.task_done()

async def pipeline():
    queue = Queue(maxsize=100)
    
    # 启动生产者和消费者
    producers = [asyncio.create_task(producer(queue, i)) for i in range(5)]
    consumers = [asyncio.create_task(consumer(queue, i)) for i in range(3)]
    
    # 等待所有生产者完成
    await asyncio.gather(*producers)
    
    # 等待消费者处理完队列
    await queue.join()
    
    # 取消消费者
    for c in consumers:
        c.cancel()

asyncio.run(pipeline())
```

## 最佳实践

1. **装饰器**：始终使用`@functools.wraps`保持元数据；优先使用内置装饰器
2. **元编程**：优先使用`__init_subclass__`，仅在需要完全控制类创建时使用元类
3. **异步编程**：避免在异步函数中调用同步阻塞代码；使用结构化并发管理任务

[^1]: 本段参考[Python Decorators: The Complete Guide](https://devtoolbox.dedyn.io/blog/python-decorators-complete-guide)
[^2]: 本段参考[The Comprehensive Guide to Python Decorators](https://asyncmove.com/blog/2026/01/the-comprehensive-guide-to-python-decorators/)
[^3]: 本段参考[Mastering Advanced Python](https://medium.com/@AIisDUMB/mastering-advanced-python-decorators-generators-and-async-programming-32673ec769fc)
