---
title: React进阶
date: 2026-04-14
description: React Concurrent Mode、Suspense、useTransition、useDeferredValue等高级特性
tags:
  - react
  - frontend
  - concurrent-mode
  - suspense
draft: false
permalink:
---

## 概述

React 18 引入了多项重大升级，其中最具变革性的是 **Concurrent Mode（并发模式）**。这一特性从根本上改变了 React 的渲染机制，使应用能够同时准备多个版本的 UI，实现了可中断的渲染能力。[^1]

本文深入探讨 React 进阶特性，涵盖：

- [[../frontend/react/server-components|React Server Components]] 的进阶应用
- [[../frontend/react-hooks|React Hooks]] 的高级模式和陷阱
- 并发渲染的核心概念与实践

---

## Concurrent Mode概述

### 什么是Concurrent Mode

Concurrent Mode 是 React 18 引入的新渲染策略，其核心目标是**提高应用的响应性**。在传统模式中，渲染是连续且不可中断的；而在并发模式下，React 可以同时处理多个任务，并根据优先级智能调度。

**核心能力**：

1. **并发渲染（Concurrent Rendering）**：React 可以同时准备多个版本的 UI，通过 `startTransition` 等 API 控制哪些更新可以被中断
2. **优先级调度（Priority Scheduling）**：用户交互（如点击输入）拥有最高优先级，而非紧急更新（如列表筛选）可以被延迟
3. **Time Slicing（时间切片）**：长任务被分割成多个小片段，每帧留出时间给高优先级任务

```tsx
// 并发渲染示意
function SearchResults({ query }) {
  // 这个组件的渲染可以被中断
  const results = useDeferredValue(query);
  
  return <div>{results.map(r => <ResultItem key={r.id} {...r} />)}</div>;
}
```

### 与旧版本区别

| 特性 | React 17 及之前 | React 18+ |
|------|-----------------|-----------|
| 渲染模型 | 连续渲染，不可中断 | 可中断的并发渲染 |
| 状态更新 | 立即生效 | 可标记为非紧急更新 |
| Suspense | 有限支持 | 完整流式支持 |
| 渲染优先级 | 无 | 多级优先级调度 |

**Time Slicing 原理**：

React 18 将渲染工作拆分成多个小单元，每完成一个单元就检查是否有更高优先级的任务：

$$
\text{Reconciler} \xrightarrow{\text{分片}} \underbrace{\text{Work}_{1} \rightarrow \text{Work}_{2} \rightarrow \cdots \rightarrow \text{Work}_{n}}_{\text{可中断}}
$$

### 启用方式

React 18 默认启用 Concurrent Features，无需额外配置：

```tsx
// React 18 + Next.js App Router 默认启用
// React 18 + 传统 SPA 需要使用 createRoot

import React from 'react';
import ReactDOM from 'react-dom/client';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

**Babel 配置**（如需实验性特性）：

```javascript
// babel.config.js
module.exports = {
  plugins: [
    ['@babel/plugin-proposal-decorators', { version: '2022-03' }],
  ],
};
```

---

## Suspense深入

### 原理：Suspense边界

Suspense 的本质是**捕获异步操作（Promise）并等待其完成**。当子组件抛出 Promise 时，React 会捕获该 Promise 并显示 fallback UI，待 Promise resolved 后重新渲染。[^2]

```tsx
import { Suspense } from 'react';

// 组件内部抛出 Promise
function UserProfile({ userId }) {
  if (!user) {
    throw fetch(`/api/users/${userId}`).then(res => res.json());
  }
  return <div>{user.name}</div>;
}

// Suspense 边界捕获 Promise
function App() {
  return (
    <Suspense fallback={<div>加载中...</div>}>
      <UserProfile userId="123" />
    </Suspense>
  );
}
```

**重渲染机制**：

1. 初次渲染：Suspense 显示 fallback
2. Promise resolved：组件重新渲染，显示实际内容
3. 后续渲染：Suspense 边界不再拦截

### 配合路由：React Router + Suspense

使用 `React.lazy` 和 Suspense 实现路由级代码分割：

```tsx
import { Suspense, lazy } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

function LoadingSpinner() {
  return <div className="spinner">加载中...</div>;
}

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<LoadingSpinner />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

### 配合数据获取：use() hook

React 19 引入了 `use()` hook，可以用同步语法读取 Promise 和 Context：

```tsx
import { use, Suspense } from 'react';

function UserData({ userPromise }) {
  // use() 暂停组件直到 Promise resolved
  const user = use(userPromise);
  
  return <div>{user.name}</div>;
}

function App() {
  const userPromise = fetch('/api/user').then(res => res.json());
  
  return (
    <Suspense fallback={<div>加载中...</div>}>
      <UserData userPromise={userPromise} />
    </Suspense>
  );
}
```

**fetch-as-you-render 模式**：

```tsx
function PostsPage({ postIds }) {
  return (
    <div>
      {postIds.map(id => (
        <Suspense key={id} fallback={<PostSkeleton />}>
          <Post id={id} />
        </Suspense>
      ))}
    </div>
  );
}

function Post({ id }) {
  // 每个 Post 组件独立获取数据
  const post = use(fetchPost(id));
  return <article><h2>{post.title}</h2></article>;
}
```

### Error Boundary：捕获异步错误

Error Boundary 只能捕获渲染阶段的同步错误。对于异步错误（如 Promise rejection），需要额外处理：

```tsx
import { Component, Suspense } from 'react';

// Error Boundary 组件
class ErrorBoundary extends Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    console.error('Error:', error, info);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <div>出错了</div>;
    }
    return this.props.children;
  }
}

// 异步错误处理包装器
function AsyncErrorBoundary({ promise, fallback, children }) {
  const [error, setError] = useState(null);

  promise.catch(err => setError(err));

  if (error) {
    return fallback || <div>加载失败: {error.message}</div>;
  }

  return children;
}

function App() {
  return (
    <ErrorBoundary fallback={<div>页面加载失败</div>}>
      <Suspense fallback={<div>加载中...</div>}>
        <AsyncErrorBoundary
          promise={fetchData()}
          fallback={<div>数据加载失败</div>}
        >
          <Content />
        </AsyncErrorBoundary>
      </Suspense>
    </ErrorBoundary>
  );
}
```

### 嵌套Suspense：多个loading状态

Suspense 可以嵌套使用，实现细粒度的 loading 状态：

```tsx
function App() {
  return (
    <Suspense fallback={<GlobalLoading />}>
      <Header />
      <main>
        <Suspense fallback={<SidebarSkeleton />}>
          <Sidebar />
        </Suspense>
        <Suspense fallback={<ContentSkeleton />}>
          <Content />
        </Suspense>
      </main>
    </Suspense>
  );
}
```

---

## useTransition

### 非紧急状态更新：startTransition API

`useTransition` 用于标记非紧急更新，使紧急更新（如用户输入）能够优先处理：

```tsx
import { useTransition } from 'react';

function SearchApp() {
  const [isPending, startTransition] = useTransition();
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  function handleSearch(e) {
    const value = e.target.value;
    setQuery(value);

    // 标记为非紧急更新
    startTransition(() => {
      setResults(searchDatabase(value));
    });
  }

  return (
    <div>
      <input value={query} onChange={handleSearch} />
      {isPending ? <Spinner /> : <Results results={results} />}
    </div>
  );
}
```

### 使用场景：搜索输入、列表筛选

**搜索输入**：

```tsx
function SearchInput({ onSearch }) {
  const [text, setText] = useState('');
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    setText(e.target.value);
    
    // 搜索建议更新是非紧急的
    startTransition(() => {
      onSearch(e.target.value);
    });
  }

  return (
    <div>
      <input value={text} onChange={handleChange} />
      {isPending && <LoadingIndicator />}
    </div>
  );
}
```

**列表筛选**：

```tsx
function ProductList({ products }) {
  const [filter, setFilter] = useState('all');
  const [isPending, startTransition] = useTransition();
  const [displayedProducts, setDisplayedProducts] = useState(products);

  function handleFilterChange(newFilter) {
    setFilter(newFilter);
    
    startTransition(() => {
      // 筛选是 CPU 密集型操作，可能较慢
      // 使用 transition 避免阻塞用户交互
      setDisplayedProducts(
        newFilter === 'all' 
          ? products 
          : products.filter(p => p.category === newFilter)
      );
    });
  }

  return (
    <div>
      <FilterButtons onChange={handleFilterChange} />
      {isPending ? <ListSkeleton /> : <List products={displayedProducts} />}
    </div>
  );
}
```

### 与setTimeout对比：原生优先级调度

| 特性 | setTimeout | useTransition |
|------|-----------|----------------|
| 优先级调度 | ❌ 无 | ✅ 有 |
| 可中断性 | ❌ 不可中断 | ✅ 可以被高优先级打断 |
| 调度时机 | 固定延迟 | 浏览器空闲时 |
| 状态同步 | ❌ 可能过期 | ✅ 始终最新 |

```tsx
// 旧方式：setTimeout（不推荐）
function OldSearch({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    const timer = setTimeout(() => {
      setResults(search(query));  // 可能与当前状态不同步
    }, 300);
    return () => clearTimeout(timer);
  }, [query]);

  return <Results results={results} />;
}

// 新方式：useTransition（推荐）
function NewSearch({ query }) {
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleSearch(value) {
    setQuery(value);
    startTransition(() => {
      setResults(search(value));  // 始终在正确的状态上下文中执行
    });
  }

  return (
    <div>
      <input value={query} onChange={e => handleSearch(e.target.value)} />
      {isPending ? <Spinner /> : <Results results={results} />}
    </div>
  );
}
```

### 最佳实践：isPending处理

`isPending` 用于在 transition 期间显示加载状态：

```tsx
function TabButton({ isActive, onClick, children }) {
  const [isPending, startTransition] = useTransition();

  function handleClick() {
    startTransition(() => {
      onClick();
    });
  }

  return (
    <button
      onClick={handleClick}
      disabled={isPending}
      className={isActive ? 'active' : ''}
    >
      {isPending ? '加载中...' : children}
    </button>
  );
}
```

---

## useDeferredValue

### 延迟值：deferred version of state

`useDeferredValue` 创建一个值的延迟版本，当紧急更新频繁时，延迟版本会暂时保持旧值：

```tsx
import { useDeferredValue } from 'react';

function SearchResults({ query }) {
  // deferredQuery 会滞后于 query
  const deferredQuery = useDeferredValue(query);
  
  return (
    <div>
      <input defaultValue={query} />
      <SlowList query={deferredQuery} />
    </div>
  );
}
```

### 使用场景：搜索建议、实时过滤

**搜索建议**：

```tsx
function SearchWithSuggestions() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  const suggestions = useMemo(() => {
    return getSuggestions(deferredQuery);
  }, [deferredQuery]);

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="搜索..."
      />
      <div className="suggestions">
        {suggestions.map(s => (
          <Suggestion key={s.id} {...s} />
        ))}
      </div>
    </div>
  );
}
```

**实时过滤**：

```tsx
function LiveFilter({ items }) {
  const [filter, setFilter] = useState('');
  const deferredFilter = useDeferredValue(filter);

  const filteredItems = useMemo(() => {
    return items.filter(item => 
      item.name.toLowerCase().includes(deferredFilter.toLowerCase())
    );
  }, [items, deferredFilter]);

  const isStale = filter !== deferredFilter;

  return (
    <div>
      <input
        value={filter}
        onChange={e => setFilter(e.target.value)}
        placeholder="过滤..."
      />
      <div style={{ opacity: isStale ? 0.5 : 1 }}>
        {filteredItems.map(item => (
          <Item key={item.id} {...item} />
        ))}
      </div>
    </div>
  );
}
```

### 与useTransition区别：value vs setter

| 特性 | useTransition | useDeferredValue |
|------|--------------|------------------|
| API 形式 | `startTransition(setter)` | `deferredValue = useDeferredValue(value)` |
| 作用对象 | 状态更新函数 | 状态值本身 |
| 适用场景 | 状态更新逻辑复杂 | 派生值计算 |
| 并行更新 | 支持多个状态更新 | 只能延迟一个值 |

```tsx
// useTransition：延迟状态更新逻辑
const [isPending, startTransition] = useTransition();
startTransition(() => {
  setResults(filterData(data, query));
});

// useDeferredValue：延迟值本身
const deferredQuery = useDeferredValue(query);
const results = filterData(data, deferredQuery);
```

**选择指南**：

- 使用 `useTransition`：当需要包装状态更新逻辑，或需要 `isPending` 状态
- 使用 `useDeferredValue`：当已有值，只需延迟使用（如 props 或 Context）

---

## useSyncExternalStore

### 外部store订阅：Zustand/Jotai集成

`useSyncExternalStore` 是 React 18 引入的 Hook，用于安全地订阅外部数据源：

```tsx
import { useSyncExternalStore } from 'react';

// Zustand store
import { create } from 'zustand';

const useStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

// 安全订阅
function Counter() {
  const count = useSyncExternalStore(
    useStore.subscribe,  // 订阅函数
    useStore.getSnapshot  // 快照函数
  );

  return (
    <div>
      <span>{count}</span>
      <button onClick={() => useStore.getState().increment()}>+</button>
    </div>
  );
}
```

**Jotai 集成**：

```tsx
import { atom } from 'jotai';
import { useSyncExternalStore } from 'react';

const countAtom = atom(0);

// 自定义 hook
function useCount() {
  return useSyncExternalStore(
    (callback) => subscribeToAtom(countAtom, callback),
    () => getAtomSnapshot(countAtom)
  );
}
```

### 快照读取：getServerSnapshot

`getServerSnapshot` 用于 SSR 场景，返回服务端渲染时的初始快照：

```tsx
function useStore(selector) {
  const store = useSyncExternalStore(
    subscribe,
    getSnapshot,
    getServerSnapshot  // SSR 时使用
  );

  return selector(store);
}

// Next.js App Router 中的 SSR-safe store
const serverSnapshot = getServerSnapshot();
const clientSnapshot = getSnapshot();

const snapshot = typeof window === 'undefined' 
  ? serverSnapshot 
  : clientSnapshot;
```

### 强制更新：forceUpdate模式

某些场景下需要强制组件重新渲染：

```tsx
function useForceUpdate() {
  const [, forceUpdate] = useReducer(x => x + 1, 0);
  return forceUpdate;
}

function ExternalDataComponent({ store }) {
  const forceUpdate = useForceUpdate();
  
  useSyncExternalStore(
    (callback) => store.subscribe(callback),
    () => store.getSnapshot(),
    () => null  // 服务端返回 null
  );

  return <div>{store.getSnapshot().data}</div>;
}
```

---

## Server Components进阶

### 客户端与服务端边界：'use client' vs 默认服务端组件

[[../frontend/react/server-components|React Server Components]] 文档中详细介绍了基础概念，这里补充进阶用法：

**边界传播规则**：

```tsx
// ServerComponent.tsx（默认）
import ClientComponent from './ClientComponent';  // ✅ 导入
import ServerOnlyComponent from './ServerOnly';   // ✅ 导入

export default function ServerComponent() {
  return (
    <>
      <ServerOnlyComponent />  {/* ✅ 服务端专属逻辑 */}
      <ClientComponent />       {/* 'use client' 组件 */}
    </>
  );
}
```

**模块级别的 'use client'**：

```tsx
// 在文件顶部声明，整个文件都是客户端组件
'use client';

import { useState } from 'react';
// ... 剩余代码
```

### RSC Payload：React Server Component Payload格式

RSC Payload 是一种特殊的序列化格式，用于服务端与客户端通信：

```json
{
  "chunks": ["vendor-chunk", "component-chunk"],
  "format": "rsc",
  "root": {
    "type": "div",
    "props": {
      "children": [
        { 
          "type": "ServerComponent", 
          "id": "V1", 
          "chunk": "server-chunk-1" 
        },
        { 
          "type": "ClientComponent", 
          "id": "C1", 
          "chunk": "client-chunk-1",
          "props": { "initialValue": 42 }
        }
      ]
    }
  }
}
```

### 流式HTML：Streaming HTML with Suspense

RSC 与 Suspense 结合实现流式 HTML：

```tsx
// app/blog/[slug]/page.tsx
import { Suspense } from 'react';
import { db } from '@/lib/db';

async function BlogContent({ slug }) {
  const post = await db.post.findUnique({ where: { slug } });
  return <article>{post.content}</article>;
}

async function BlogComments({ postId }) {
  // 较慢的查询
  const comments = await db.comment.findMany({ where: { postId } });
  return <CommentList comments={comments} />;
}

export default async function BlogPage({ params }) {
  const post = await db.post.findUnique({ where: { slug: params.slug } });

  return (
    <div>
      <h1>{post.title}</h1>
      
      {/* 内容较快，先显示 */}
      <Suspense fallback={<ContentSkeleton />}>
        <BlogContent slug={params.slug} />
      </Suspense>
      
      {/* 评论较慢，后显示 */}
      <Suspense fallback={<CommentsSkeleton />}>
        <BlogComments postId={post.id} />
      </Suspense>
    </div>
  );
}
```

### 缓存策略：fetch缓存、POST请求处理

**Fetch 缓存配置**：

```tsx
// 默认缓存（推荐）
const data = await fetch('/api/data');

// 重新验证：1小时后
const data1 = await fetch('/api/data', { 
  next: { revalidate: 3600 } 
});

// 标签式重新验证
const data2 = await fetch('/api/data', {
  next: { tags: ['posts'] }
});

// 禁用缓存（实时数据）
const realtime = await fetch('/api/realtime', { 
  cache: 'no-store' 
});
```

**POST 请求处理**：

```tsx
// 服务端 Action（推荐）
async function updatePost(formData: FormData) {
  'use server';
  
  const title = formData.get('title');
  await db.post.update({ 
    where: { id: formData.get('id') },
    data: { title } 
  });
  
  revalidatePath('/posts');
}

// 客户端调用
function PostForm({ post }) {
  const [isPending, startTransition] = useTransition();
  
  function handleSubmit(formData: FormData) {
    startTransition(() => {
      updatePost(formData);
    });
  }
  
  return (
    <form action={handleSubmit}>
      <input name="title" defaultValue={post.title} />
      <button type="submit" disabled={isPending}>
        {isPending ? '保存中...' : '保存'}
      </button>
    </form>
  );
}
```

---

## 性能优化模式

### Memo策略：useMemo、useCallback、React.memo

**三层 Memo 策略**：

| 层级 | API | 适用场景 |
|------|-----|----------|
| 组件级 | `React.memo` | 避免子组件不必要的重渲染 |
| 函数级 | `useCallback` | 缓存传递给子组件的回调 |
| 值级 | `useMemo` | 缓存昂贵计算结果 |

**React.memo 最佳实践**：

```tsx
// 使用相等性比较函数
const MemoizedComponent = React.memo(
  function MyComponent({ data, onClick }) {
    return <div onClick={onClick}>{data.name}</div>;
  },
  (prevProps, nextProps) => {
    // 自定义比较逻辑
    return prevProps.data.id === nextProps.data.id;
  }
);
```

**useCallback 与 useMemo 组合**：

```tsx
function Parent({ list }) {
  const [filter, setFilter] = useState('');

  // 缓存回调 - 依赖 filter
  const handleItemClick = useCallback((id) => {
    console.log('Clicked:', id, filter);
  }, [filter]);

  // 缓存过滤后的列表
  const filteredList = useMemo(() => {
    return list.filter(item => 
      item.name.includes(filter)
    );
  }, [list, filter]);

  return (
    <div>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      <List items={filteredList} onItemClick={handleItemClick} />
    </div>
  );
}

const List = React.memo(function List({ items, onItemClick }) {
  return items.map(item => (
    <ListItem key={item.id} item={item} onClick={onItemClick} />
  ));
});
```

### 虚拟化：react-window/virtual list

对于长列表，虚拟化可以大幅减少 DOM 节点数量：

```tsx
import { FixedSizeList as List } from 'react-window';
import React from 'react';

function VirtualizedList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      <ListItem item={items[index]} />
    </div>
  );

  return (
    <List
      height={600}
      itemCount={items.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </List>
  );
}
```

**可变大小列表**：

```tsx
import { VariableSizeList as List } from 'react-window';

function DynamicList({ items }) {
  const getItemSize = (index) => items[index].height || 50;

  return (
    <List
      height={600}
      itemCount={items.length}
      itemSize={getItemSize}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>
          <DynamicItem item={items[index]} />
        </div>
      )}
    </List>
  );
}
```

### 预加载：\<link rel="preload"\>、prefetch

**资源预加载**：

```tsx
// Next.js 中的资源预加载
import { Preload } from '@next/font';

function Page() {
  return (
    <>
      <link
        rel="preload"
        href="/fonts/my-font.woff2"
        as="font"
        type="font/woff2"
        crossOrigin="anonymous"
      />
      <div>Content</div>
    </>
  );
}
```

**路由预加载**：

```tsx
import { useRouter } from 'next/navigation';

function Navigation() {
  const router = useRouter();

  function handleHover(path) {
    // 用户悬停时预加载页面
    router.prefetch(path);
  }

  return (
    <nav>
      <Link href="/dashboard" onMouseEnter={() => handleHover('/dashboard')}>
        Dashboard
      </Link>
    </nav>
  );
}
```

### 懒加载：React.lazy、lazy component

**动态导入**：

```tsx
import React, { lazy, Suspense } from 'react';

// 方式1：React.lazy + Suspense
const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <Suspense fallback={<div>加载中...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}

// 方式2：Next.js dynamic（推荐）
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(() => import('./HeavyComponent'), {
  loading: () => <div>加载中...</div>,
  ssr: true,  // 是否服务端渲染
});

function App() {
  return <HeavyComponent />;
}
```

**带条件的懒加载**：

```tsx
const Modal = lazy(() => import('./Modal'));
const Charts = lazy(() => import('./Charts'));

function Dashboard({ showModal, showCharts }) {
  return (
    <div>
      <DashboardContent />
      
      {showModal && (
        <Suspense fallback={null}>
          <Modal />
        </Suspense>
      )}
      
      {showCharts && (
        <Suspense fallback={<ChartSkeleton />}>
          <Charts />
        </Suspense>
      )}
    </div>
  );
}
```

---

## 生产实践

### 迁移指南：从类组件到Hooks

**类组件 vs Hooks 对照**：

| 类组件 | Hooks |
|--------|-------|
| `this.state.count` | `const [count, setCount] = useState(0)` |
| `this.setState({ count: n })` | `setCount(n)` |
| `componentDidMount` | `useEffect(() => {}, [])` |
| `componentDidUpdate` | `useEffect(() => {}, [dep])` |
| `componentWillUnmount` | `useEffect(() => { return () => {} }, [])` |
| `this.forceUpdate()` | `useReducer(x => x + 1, 0)` |

**完整迁移示例**：

```tsx
// 类组件
class Counter extends React.Component {
  state = { count: 0 };
  
  componentDidMount() {
    document.title = `计数: ${this.state.count}`;
  }
  
  componentDidUpdate() {
    document.title = `计数: ${this.state.count}`;
  }
  
  componentWillUnmount() {
    console.log('组件卸载');
  }
  
  render() {
    return (
      <div>
        <p>{this.state.count}</p>
        <button onClick={() => this.setState({ count: this.state.count + 1 })}>
          +
        </button>
      </div>
    );
  }
}

// Hooks 组件
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    document.title = `计数: ${count}`;
    return () => console.log('清理');
  }, [count]);
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
```

### 常见陷阱：闭包陷阱、无限循环

**闭包陷阱**：

```tsx
// ❌ 错误：闭包中的旧值
function Search() {
  const [query, setQuery] = useState('');
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log(query);  // query 永远是初始值 ''
    }, 1000);
    return () => clearInterval(id);
  }, []);  // 空依赖数组
  
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}

// ✅ 正确：使用 ref 或函数式更新
function SearchFixed() {
  const [query, setQuery] = useState('');
  const queryRef = useRef(query);
  queryRef.current = query;
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log(queryRef.current);  // 始终是最新值
    }, 1000);
    return () => clearInterval(id);
  }, []);
  
  return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

**无限循环**：

```tsx
// ❌ 错误：useEffect 依赖数组不完整
function BrokenComponent({ data }) {
  const [processed, setProcessed] = useState([]);
  
  useEffect(() => {
    setProcessed(data.map(process));  // 每次 data 变化都触发
  });  // 缺少依赖数组
  
  return <List items={processed} />;
}

// ✅ 正确：完整的依赖数组或 useMemo
function FixedComponent({ data }) {
  const processed = useMemo(() => {
    return data.map(process);
  }, [data]);
  
  return <List items={processed} />;
}
```

**对象/数组依赖问题**：

```tsx
// ❌ 错误：每次渲染都是新对象
function Component({ options }) {
  useEffect(() => {
    fetchData(options);  // options 是新对象
  }, [options]);  // 永远不相等
  
  return <div>{options.name}</div>;
}

// ✅ 正确：提取稳定值或使用 useMemo
function ComponentFixed({ options }) {
  const stableOptions = useMemo(() => ({
    id: options.id,
    type: options.type,
  }), [options.id, options.type]);
  
  useEffect(() => {
    fetchData(stableOptions);
  }, [stableOptions]);
  
  return <div>{options.name}</div>;
}
```

### 调试工具：React DevTools Profiler

**Profiler 使用方法**：

```tsx
// 包装需要分析的组件
import { Profiler } from 'react';

function onRenderCallback(
  id,                     // 提交的 id
  phase,                  // 'mount' | 'update'
  actualDuration,         // 渲染耗时
  baseDuration,          // 估算的基础渲染时间
  startTime,             // 开始时间
  commitTime,            // 提交时间
  interactions           // 交互集合
) {
  console.log(`组件 ${id} 的 ${phase} 耗时: ${actualDuration}ms`);
}

function App() {
  return (
    <Profiler id="MainApp" onRender={onRenderCallback}>
      <MainApp />
    </Profiler>
  );
}
```

**DevTools Profiler 面板**：

1. **Rendered by** 视图：查看哪些组件频繁渲染
2. **Flamegraph** 视图：可视化渲染调用栈
3. **Ranked** 视图：按渲染耗时排序组件

### 性能监测：Core Web Vitals关联

| Core Web Vitals | React 优化点 |
|----------------|-------------|
| **LCP** (Largest Contentful Paint) | 优化首屏渲染、预加载关键资源 |
| **FID** (First Input Delay) | 减少主线程阻塞、使用 `useTransition` |
| **CLS** (Cumulative Layout Shift) | 预留图片/字体空间、避免动态内容插入 |

**LCP 优化示例**：

```tsx
// 预加载 LCP 图片
function HeroSection() {
  return (
    <>
      <link
        rel="preload"
        href="/hero-image.jpg"
        as="image"
      />
      <img 
        src="/hero-image.jpg" 
        alt="Hero"
        fetchPriority="high"  // 标记为高优先级
      />
    </>
  );
}
```

**CLS 优化示例**：

```tsx
// 为图片预留空间
function ArticleImage({ src, alt, width, height }) {
  const aspectRatio = width / height;
  
  return (
    <div style={{ aspectRatio }}>
      <img 
        src={src} 
        alt={alt}
        style={{ width: '100%', height: '100%', objectFit: 'cover' }}
      />
    </div>
  );
}

// 字体加载优化
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',  // 避免字体加载阻塞渲染
});
```

---

## 总结

React 18 的进阶特性为应用性能优化提供了强大的工具集：

| 特性 | 核心价值 | 适用场景 |
|------|---------|----------|
| Concurrent Mode | 可中断渲染 | 大型列表、复杂表单 |
| Suspense | 流式加载 | 数据获取、代码分割 |
| useTransition | 优先级调度 | 搜索、筛选 |
| useDeferredValue | 延迟更新 | 派生状态 |
| useSyncExternalStore | 外部状态订阅 | Redux、Zustand、Jotai |
| Server Components | 零客户端 JS | 数据密集型页面 |

**最佳实践**：

1. **渐进采用**：从 `useTransition` 和 `useDeferredValue` 开始，它们是向后兼容的
2. **精确边界**：`Suspense` 和 Server Components 边界越精确，性能收益越大
3. **测量优先**：使用 React DevTools Profiler 定位真实瓶颈
4. **避免过早优化**：Memo 策略应用于真实性能问题，而非预防性添加

[^1]: React Concurrent Mode 官方文档 - https://react.dev/blog/2022/03/29/react-v18

[^2]: React Suspense 深度解析 - https://react.dev/reference/react/Suspense

[^3]: Next.js App Router 缓存机制 - https://nextjs.org/docs/app/building-your-application/data-fetching

---

## 相关主题

- [[../frontend/react/server-components|React Server Components 完全指南]]
- [[../frontend/react-hooks|React Hooks 基础]]
- [[../frontend/micro-frontends|Micro-Frontends 微前端架构]]
- [[../CCF-CSP/CSP202312B|页面渲染性能优化实战]]
