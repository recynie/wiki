---
title: React Hooks 基础
date: 2026-04-11
description: React Hooks 核心概念与最佳实践
tags:
  - react
  - frontend
  - hooks
draft: false
permalink:
---

## 概述

React Hooks 是 React 16.8 引入的功能，允许函数组件使用 state 和其他 React 特性。在 Hooks 之前，只有 class 组件才能使用 state 和生命周期方法。[^1]

### Hooks 规则

1. **只能在顶层调用**：不要在循环、条件或嵌套函数中调用 Hook
2. **只能在 React 函数中调用**：从函数组件或自定义 Hook 中调用

```typescript
// 错误示例
if (isLoggedIn) {
  const [name, setName] = useState('');  // 违反规则1
}

// 正确示例
function Component({ isLoggedIn }) {
  const [name, setName] = useState('');  // 顶层调用
  // ...
}
```

---

## useState

useState 是最基础的 Hook，用于为组件添加状态。

### 基本用法

```typescript
import { useState } from 'react';

function Counter() {
  // 语法：const [状态值, 更新函数] = useState(初始值)
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>计数: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        增加
      </button>
    </div>
  );
}
```

### 函数式更新

当新状态依赖旧状态时，使用函数式更新避免闭包问题：

```typescript
// 错误：闭包陷阱
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);  // count 永远是 0
    }, 1000);
    return () => clearInterval(id);
  }, []);  // 空依赖数组
  
  return <div>{count}</div>;
}

// 正确：使用函数式更新
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      setCount(prevCount => prevCount + 1);  // 使用前一个状态
    }, 1000);
    return () => clearInterval(id);
  }, []);
  
  return <div>{count}</div>;
}
```

### 惰性初始化

当初始状态需要计算时，传入函数避免重复计算：

```typescript
// 每次渲染都会执行
const [state, setState] = useState(expensiveComputation());

// 只执行一次
const [state, setState] = useState(() => expensiveComputation());
```

### 对象状态

```typescript
// 分离的状态通常比对象更好
function UserProfile() {
  const [name, setName] = useState('');
  const [age, setAge] = useState(0);
  // 而不是 const [user, setUser] = useState({ name: '', age: 0 });
  
  return (
    <div>
      <input value={name} onChange={e => setName(e.target.value)} />
      <input type="number" value={age} onChange={e => setAge(+e.target.value)} />
    </div>
  );
}
```

---

## useEffect

useEffect 用于处理副作用，如数据获取、订阅、DOM 操作。

### 基本语法

```typescript
useEffect(() => {
  // 副作用逻辑
  
  return () => {
    // 清理函数（可选）
  };
}, [依赖数组]);
```

### 依赖数组行为

| 形式 | 行为 |
|------|------|
| 无数组 | 每次渲染后执行 |
| `[]` | 仅挂载时执行一次 |
| `[a, b]` | a 或 b 变化时执行 |

### 常见用法

#### 数据获取

```typescript
import { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    async function fetchUser() {
      try {
        setLoading(true);
        const response = await fetch(`/api/users/${userId}`);
        if (!response.ok) throw new Error('Failed to fetch');
        const data = await response.json();
        if (!cancelled) {
          setUser(data);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err.message);
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    }
    
    fetchUser();
    
    // 清理函数：组件卸载或 userId 变化时执行
    return () => {
      cancelled = true;
    };
  }, [userId]);
  
  if (loading) return <div>加载中...</div>;
  if (error) return <div>错误: {error}</div>;
  if (!user) return <div>用户不存在</div>;
  
  return <div>Hello, {user.name}</div>;
}
```

#### 订阅

```typescript
function ChatRoom({ roomId }) {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    const subscription = chatService.subscribe(roomId, (message) => {
      setMessages(prev => [...prev, message]);
    });
    
    // 清理：取消订阅
    return () => {
      subscription.unsubscribe();
    };
  }, [roomId]);
  
  return <div>{/* 渲染消息 */}</div>;
}
```

#### DOM 操作

```typescript
function useDocumentTitle(title) {
  useEffect(() => {
    document.title = title;
  }, [title]);
}

function Component() {
  useDocumentTitle('Hello');
  // ...
}
```

---

## useCallback 与 useMemo

用于性能优化，避免不必要的计算或渲染。

### useCallback

缓存函数引用，用于传递给子组件：

```typescript
import { useCallback } from 'react';

function Parent() {
  const [count, setCount] = useState(0);
  
  // 每次渲染都会创建新函数，导致子组件不必要地重新渲染
  const handleClick = () => {
    console.log('clicked');
  };
  
  // 缓存函数引用，仅在 count 变化时创建新函数
  const memoizedHandleClick = useCallback(() => {
    console.log('clicked');
  }, []);
  
  // 依赖 count 的函数
  const increment = useCallback(() => {
    setCount(count + 1);
  }, [count]);
  
  return <Child onClick={memoizedHandleClick} increment={increment} />;
}
```

### useMemo

缓存计算结果：

```typescript
import { useMemo } from 'react';

function ExpensiveList({ items, filter }) {
  // 仅在 items 或 filter 变化时重新计算
  const filteredItems = useMemo(() => {
    console.log('Filtering...');  // 用于验证
    return items.filter(item => item.name.includes(filter));
  }, [items, filter]);
  
  return <div>{filteredItems.map(i => <Item key={i.id} {...i} />)}</div>;
}
```

### 性能优化原则

| 情况 | 建议 |
|------|------|
| 传递给 `React.memo()` 的子组件的回调 | 使用 `useCallback` |
| 昂贵计算（如排序、大数据处理） | 使用 `useMemo` |
| 普通对象/数组字面量 | 不需要优化 |

```typescript
// 不要过度优化
function Component() {
  // 每次渲染都创建新对象（不必要的优化）
  const style = useMemo(() => ({ color: 'red' }), []);
  
  // 简单值不需要 useMemo
  const data = useMemo(() => ({ type: 'card' }), []);
  
  // 静态对象直接创建
  const style = { color: 'red' };  // 足够好
}
```

---

## useRef

useRef 用于访问 DOM 元素或存储可变值。

### 访问 DOM

```typescript
function TextInput() {
  const inputRef = useRef(null);
  
  const focusInput = () => {
    inputRef.current.focus();
  };
  
  return (
    <div>
      <input ref={inputRef} type="text" />
      <button onClick={focusInput}>聚焦</button>
    </div>
  );
}
```

### 存储可变值

```typescript
function Timer() {
  const [count, setCount] = useState(0);
  const intervalRef = useRef(null);
  
  useEffect(() => {
    intervalRef.current = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
    
    return () => clearInterval(intervalRef.current);
  }, []);
  
  const stop = () => clearInterval(intervalRef.current);
  
  return <div>{count} <button onClick={stop}>停止</button></div>;
}
```

### 保留上次渲染的值

```typescript
function SearchComponent() {
  const [query, setQuery] = useState('');
  const previousQueryRef = useRef('');
  
  useEffect(() => {
    previousQueryRef.current = query;
  });
  
  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <p>上次搜索: {previousQueryRef.current}</p>
    </div>
  );
}
```

---

## useContext

用于在组件树中传递数据，避免 prop drilling。

### 创建 Context

```typescript
// ThemeContext.js
import { createContext, useContext, useState } from 'react';

const ThemeContext = createContext();

export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  
  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  };
  
  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}
```

### 使用 Context

```typescript
function App() {
  return (
    <ThemeProvider>
      <Header />
      <Content />
      <Footer />
    </ThemeProvider>
  );
}

function Header() {
  const { theme, toggleTheme } = useTheme();
  
  return (
    <header className={theme}>
      <button onClick={toggleTheme}>切换主题</button>
    </header>
  );
}
```

---

## useReducer

用于复杂状态逻辑，是 useState 的替代方案。

### 基本用法

```typescript
import { useReducer } from 'react';

// 状态和操作的类型定义
interface State {
  count: number;
}

type Action = 
  | { type: 'increment' }
  | { type: 'decrement' }
  | { type: 'reset' };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return { count: 0 };
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  
  return (
    <div>
      <p>计数: {state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>重置</button>
    </div>
  );
}
```

### 结合 Context

```typescript
// 状态与操作分离
const [state, dispatch] = useReducer(reducer, initialState);
const value = { state, dispatch };

return (
  <Context.Provider value={value}>
    {children}
  </Context.Provider>
);
```

---

## 自定义 Hook

将可复用逻辑提取到自定义 Hook 中。

### 封装逻辑

```typescript
// useLocalStorage.js
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      return initialValue;
    }
  });
  
  const setValue = (value) => {
    try {
      const valueToStore = value instanceof Function 
        ? value(storedValue) 
        : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(error);
    }
  };
  
  return [storedValue, setValue];
}

// 使用
function App() {
  const [name, setName] = useLocalStorage('name', '');
  
  return <input value={name} onChange={e => setName(e.target.value)} />;
}
```

### 更多自定义 Hook 示例

```typescript
// useDebounce.js
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => clearTimeout(handler);
  }, [value, delay]);
  
  return debouncedValue;
}

// useFetch.js
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    fetch(url)
      .then(res => {
        if (!res.ok) throw new Error(res.statusText);
        return res.json();
      })
      .then(data => {
        if (!cancelled) {
          setData(data);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled) {
          setError(err);
          setLoading(false);
        }
      });
    
    return () => { cancelled = true; };
  }, [url]);
  
  return { data, loading, error };
}
```

---

## useLayoutEffect

与 useEffect 相同，但在 DOM 更新后同步执行，用于读取 DOM 布局。

```typescript
import { useLayoutEffect, useState } from 'react';

function MeasureBox() {
  const [width, setWidth] = useState(0);
  const [height, setHeight] = useState(0);
  const ref = useRef(null);
  
  useLayoutEffect(() => {
    // 在 DOM 更新后同步执行，确保获得最新布局信息
    setWidth(ref.current.offsetWidth);
    setHeight(ref.current.offsetHeight);
  }, []);
  
  return (
    <div ref={ref}>
      Width: {width}, Height: {height}
    </div>
  );
}
```

### useEffect vs useLayoutEffect

| 特性 | useEffect | useLayoutEffect |
|------|-----------|-----------------|
| 执行时机 | 渲染后异步 | DOM更新后同步 |
| 阻塞渲染 | 否 | 是 |
| 适用场景 | 数据获取、订阅 | DOM布局测量 |

---

## Hooks 性能陷阱

### 依赖数组不完整

```typescript
// 错误：count 在依赖数组中缺失
useEffect(() => {
  setTotal(count * price);
}, []);  // count 和 price 变化时不会重新执行

// 正确
useEffect(() => {
  setTotal(count * price);
}, [count, price]);
```

### 在渲染期间更新状态

```typescript
// 错误：会导致无限循环
function Broken() {
  const [count, setCount] = useState(0);
  
  if (count === 0) {
    setCount(count + 1);  // 触发重新渲染 → 再执行 → 无限循环
  }
  
  return <div>{count}</div>;
}
```

### 对象/函数依赖

```typescript
// 错误：options 每次渲染都是新对象
function Search({ options }) {
  useEffect(() => {
    fetchData(options);  // options 是新对象，不会匹配依赖
  }, [options]);  // 永远会触发
  
  return <div>...</div>;
}

// 解决：使用 useMemo/useCallback
function Search({ options }) {
  const stableOptions = useMemo(() => options, [options]);
  useEffect(() => {
    fetchData(stableOptions);
  }, [stableOptions]);
}
```

---

## 参考资料

[^1]: [React Hooks 官方文档](https://react.dev/reference/react)
[^2]: [useEffect 指南](https://react.dev/reference/react/useEffect)
[^3]: [React Hooks 规则](https://react.dev/reference/rules/rules-of-hooks)
