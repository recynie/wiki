---
title: React Server Components（RSC）完全指南
date: 2026-04-14
description: React Server Components 核心概念、架构设计与 Next.js App Router 最佳实践
tags:
  - react
  - rsc
  - nextjs
  - server-components
  - frontend
draft: false
permalink:
---

## 概述

React Server Components（RSC）是 React 18 引入的重大架构革新，代表了从「客户端优先」到「服务端优先」的根本性转变。传统 React 应用中，组件默认在客户端渲染，所有数据获取都需要通过 API 或 [[../data-structure/monotonous-stack|Effect]] 模式。而 RSC 允许组件默认在服务端执行，直接访问数据库、文件系统和企业内部 API，从根本上改变了我们构建 React 应用的方式。[^1]

RSC 的核心价值在于：

1. **零客户端 JavaScript**：服务端组件不会向客户端发送 JavaScript 代码，大幅减少 bundle 体积
2. **直接后端访问**：组件可以直接调用数据库或内部服务，无需暴露 API 端点
3. **流式渲染**：结合 [[../data-structure/monotonous-stack|Suspense]] 实现渐进式加载，提升用户体验
4. **统一数据获取**：用同步的组件语法表达异步数据请求，简化心智模型

---

## 核心心智模型

### 服务端组件 vs 客户端组件决策树

理解 RSC 的第一步是明确组件的运行环境。决策逻辑如下：

```typescript
// 这个组件需要：
// 1. 使用 useState/useEffect 等 Hooks？  → 客户端组件
// 2. 访问浏览器 API（window/document）？ → 客户端组件
// 3. 监听用户交互事件（onClick/onChange）？ → 客户端组件
// 4. 需要实时数据更新？  → 客户端组件

// 否则，默认使用服务端组件：
// - 纯展示型组件
// - 直接获取数据库数据
// - 访问服务端资源（文件、API）
// - 大型计算任务
```

### 'use client' 指令作为边界

`'use client'` 指令标记了客户端组件的边界，它声明了一个组件及其所有子组件都应该在客户端运行：

```typescript
// Button.tsx
'use client';

import { useState } from 'react';

export function Button({ children, onClick }) {
  const [loading, setLoading] = useState(false);
  
  return (
    <button onClick={onClick} disabled={loading}>
      {loading ? '加载中...' : children}
    </button>
  );
}
```

**关键规则**：一旦标记为 `'use client'`，该组件及其导入的所有子组件都会被打包到客户端 bundle 中。

### RSC Payload 格式

服务端组件的输出不是传统的 HTML，而是一种特殊的 RSC Payload 格式：

```json
{
  "chunks": ["client-component-chunk", "server-component-chunk"],
  "format": "rsc",
  "root": {
    "type": "div",
    "props": {
      "children": [
        { "type": "ServerComponent", "id": "sc1", "chunk": "sc-chunk" },
        { "type": "ClientComponent", "id": "cc1", "chunk": "cc-chunk" }
      ]
    }
  }
}
```

这个 payload 由服务端生成，客户端的 React 根据它协调并渲染组件。[^2]

---

## 服务端组件深度解析

### 异步数据获取

服务端组件最大的革新之一是支持在组件内部直接使用 `async/await`：

```typescript
// app/posts/page.tsx
import { db } from '@/lib/db';

export default async function PostsPage() {
  // 直接在组件中查询数据库，无需 API 层
  const posts = await db.post.findMany({
    where: { published: true },
    orderBy: { createdAt: 'desc' },
    take: 10,
  });

  return (
    <main>
      <h1>文章列表</h1>
      <ul>
        {posts.map(post => (
          <li key={post.id}>
            <a href={`/posts/${post.slug}`}>{post.title}</a>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

这种模式的优势在于：

- 数据获取逻辑与组件渲染逻辑同处一处，代码更紧凑
- 可以使用顶层 `await`，无需 `useEffect` + 状态管理
- 多个数据请求可以并行执行

```typescript
// 并行数据获取示例
export default async function DashboardPage() {
  const [user, stats, notifications] = await Promise.all([
    getUser(),
    getStats(),
    getNotifications(),
  ]);
  
  return <Dashboard user={user} stats={stats} notifications={notifications} />;
}
```

### 直接数据库与 API 访问

服务端组件可以直接访问企业内部的微服务、数据库，无需暴露公网 API：

```typescript
// app/orders/[id]/page.tsx
import { db } from '@/lib/db';
import { orderService } from '@/services/order-service';
import { inventoryService } from '@/services/inventory-service';

export default async function OrderDetailPage({ params }) {
  // 直接访问数据库
  const order = await db.order.findUnique({
    where: { id: params.id },
    include: { items: true, customer: true },
  });

  // 直接调用内部微服务
  const inventory = await inventoryService.checkStock(order.items);

  // 调用另一个微服务获取物流信息
  const tracking = await orderService.getTracking(order.id);

  return (
    <div>
      <OrderHeader order={order} />
      <InventoryStatus items={inventory} />
      <TrackingInfo tracking={tracking} />
    </div>
  );
}
```

这种架构的优势：
- **安全性**：敏感凭证留在服务端，不会泄露到客户端
- **性能**：内网调用延迟远低于公网 API
- **简化**：无需为简单查询创建 REST/GraphQL 端点

### 零客户端 JavaScript

服务端组件不会增加客户端 bundle 体积。考虑这个组件树：

```typescript
// app/page.tsx（服务端组件）
import ProductList from './ProductList';      // 服务端组件
import AddToCartButton from './AddToCart';    // 客户端组件
import Reviews from './Reviews';              // 服务端组件

export default async function Page() {
  const products = await getProducts();
  
  return (
    <div>
      <h1>产品列表</h1>
      <ProductList products={products} />
      <AddToCartButton productId={products[0].id} />
      <Reviews productId={products[0].id} />
    </div>
  );
}
```

只有 `AddToCartButton`（标记了 `'use client'`）会被打包到客户端。

---

## 客户端组件

### 何时使用 'use client'

客户端组件适用于所有需要交互或浏览器特质的场景：

| 场景 | 示例 |
|------|------|
| 状态管理 | `useState`、`useReducer` |
| 副作用 | `useEffect`、`useLayoutEffect` |
| 浏览器 API | `window`、`document`、`localStorage` |
| 事件处理 | `onClick`、`onChange`、`onSubmit` |
| 实时更新 | WebSocket、轮询 |
|第三方组件 | 需要 DOM 访问的动画库、图表库 |

```typescript
'use client';

import { useState, useEffect } from 'react';
import { useLocalStorage } from '@/hooks/useLocalStorage';

export function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
    document.documentElement.setAttribute('data-theme', theme);
  }, [theme]);

  // 避免水合不匹配
  if (!mounted) return <div className="w-10 h-10" />;

  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      当前主题：{theme}
    </button>
  );
}
```

### 最小化客户端 Bundle 体积

即使需要使用客户端组件，也应尽可能缩小其影响范围：

**❌ 不推荐：顶层标记**

```typescript
'use client';  // 整个页面都是客户端组件

import { db } from '@/lib/db';

export default async function Page() {  // 错误：服务端组件不能有 async
  const data = await db.getData();
  return <div>{data}</div>;
}
```

**✅ 推荐：精确边界**

```typescript
// app/page.tsx（服务端组件）
import StaticContent from './StaticContent';    // 服务端
import InteractiveMap from './InteractiveMap'; // 客户端 - 只需地图交互

export default async function Page() {
  return (
    <div>
      <StaticContent />
      <InteractiveMap />
    </div>
  );
}
```

### 深入组件树的边界划分

`'use client'` 指令向下传播，可以精确控制边界位置：

```typescript
// app/blog/[slug]/page.tsx
import { db } from '@/lib/db';
import BlogContent from './BlogContent';     // 服务端
import LikeButton from './LikeButton';       // 客户端
import ShareButtons from './ShareButtons';   // 客户端
import RelatedPosts from './RelatedPosts';   // 服务端

export default async function BlogPostPage({ params }) {
  const post = await db.post.findUnique({ where: { slug: params.slug } });

  return (
    <article>
      <BlogContent content={post.content} />      {/* 服务端渲染 */}
      <LikeButton postId={post.id} />               {/* 客户端渲染 */}
      <ShareButtons url={post.url} />               {/* 客户端渲染 */}
      <RelatedPosts tags={post.tags} />             {/* 服务端渲染 */}
    </article>
  );
}
```

---

## 组合模式

### 服务端组件包裹客户端组件（✅ 允许）

这是最自然的组合方式，服务端组件作为「容器」提供数据和布局：

```typescript
// app/layout.tsx（服务端）
import Sidebar from './Sidebar';       // 服务端
import ClientWrapper from './ClientWrapper'; // 客户端

export default async function Layout({ children }) {
  const user = await getCurrentUser();
  const settings = await getSettings();

  return (
    <div className="layout">
      <Sidebar user={user} />                      {/* 服务端获取用户 */}
      <ClientWrapper initialSettings={settings}>  {/* 客户端管理状态 */}
        {children}
      </ClientWrapper>
    </div>
  );
}
```

### 客户端组件导入服务端组件（❌ 不允许）

这是 RSC 架构的核心限制。考虑这个问题：

```typescript
// ClientComponent.tsx
'use client';
import ServerComponent from './ServerComponent'; // ❌ 编译错误

export function ClientComponent() {
  return <ServerComponent />;  // 无法在客户端渲染服务端组件
}
```

**原因**：服务端组件在构建时执行，而客户端组件在浏览器执行，时机不匹配。

### 通过 props 传递服务端组件（✅ 特殊例外）

将服务端组件作为 `children` 或 `props` 传递是允许的：

```typescript
// app/page.tsx
import ServerChild from './ServerChild';  // 服务端组件

export default async function Page() {
  const data = await fetchData();
  
  return (
    <ClientParent>
      <ServerChild data={data} />  {/* 服务端组件作为 children 传递 */}
    </ClientParent>
  );
}
```

```typescript
// ClientParent.tsx
'use client';

export function ClientParent({ children }) {
  const [expanded, setExpanded] = useState(false);
  
  return (
    <div onClick={() => setExpanded(!expanded)}>
      {expanded && children}  {/* children 是预渲染好的服务端组件 */}
    </div>
  );
}
```

### 模式对比

| 模式 | 是否允许 | 说明 |
|------|---------|------|
| 服务端 → 服务端 | ✅ | 正常嵌套 |
| 服务端 → 客户端 | ✅ | 服务端提供数据和容器 |
| 客户端 → 服务端 | ❌ | 编译错误 |
| 客户端 → 客户端 | ✅ | 正常嵌套 |
| 服务端组件作为 children | ✅ | 通过 React Slot 机制 |

---

## 流式渲染与 Suspense

### 渐进式加载

RSC 与 [[../data-structure/monotonous-stack|Suspense]] 结合实现流式渲染，用户无需等待完整数据即可看到内容：

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react';
import { getUser, getStats, getNotifications } from '@/lib/data';

function UserProfile() {
  const user = await getUser();
  return <div className="profile">{user.name}</div>;
}

function StatsCard() {
  const stats = await getStats();  // 可能较慢
  return <div>活跃用户：{stats.activeUsers}</div>;
}

function NotificationList() {
  const notifications = await getNotifications();
  return <ul>{notifications.map(n => <li key={n.id}>{n.text}</li>)}</ul>;
}

function LoadingSkeleton({ height }) {
  return <div className="skeleton" style={{ height }} />;
}

export default async function DashboardPage() {
  return (
    <div className="dashboard">
      {/* 并行请求，竞速结果 */}
      <Suspense fallback={<LoadingSkeleton height="60px" />}>
        <UserProfile />
      </Suspense>
      
      <div className="grid">
        <Suspense fallback={<LoadingSkeleton height="200px" />}>
          <StatsCard />
        </Suspense>
        
        <Suspense fallback={<LoadingSkeleton height="200px" />}>
          <NotificationList />
        </Suspense>
      </div>
    </div>
  );
}
```

### Suspense 边界策略

合理的 Suspense 边界划分：

```typescript
// 粗粒度：一个大的 Suspense 包裹整个页面
// 优点：简单；缺点：任何数据慢都会阻塞整个区域

// 细粒度：每个慢数据源单独包裹
// 优点：快的内容先展示；缺点：代码复杂

// 推荐：混合策略
export default async function BlogPostPage({ params }) {
  const [post, author] = await Promise.all([
    getPost(params.slug),
    getAuthor(),  // 较小，较快
  ]);

  return (
    <article>
      {/* author 数据快，不需要 Suspense */}
      <AuthorBadge author={author} />
      
      {/* post.content 可能较大，单独 Suspense */}
      <Suspense fallback={<ContentSkeleton />}>
        <PostContent post={post} />
      </Suspense>
      
      {/* comments 可能非常慢，独立边界 */}
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments postId={post.id} />
      </Suspense>
    </article>
  );
}
```

### Loading.tsx 模式

Next.js App Router 约定：`loading.tsx` 自动为其下的所有页面添加 Suspense 边界：

```typescript
// app/dashboard/loading.tsx
export default function Loading() {
  return (
    <div className="dashboard-loading">
      <div className="skeleton-header" />
      <div className="skeleton-grid">
        <div className="skeleton-card" />
        <div className="skeleton-card" />
        <div className="skeleton-card" />
      </div>
    </div>
  );
}
```

```typescript
// app/dashboard/page.tsx
// 继承 loading.tsx 的 Suspense 包装
export default async function DashboardPage() {
  const data = await getDashboardData();
  return <Dashboard data={data} />;
}
```

---

## 性能优化

### 四层缓存策略

Next.js App Router 实现了一套完整的缓存体系：[^3]

```typescript
// 第一层：Request 缓存（请求级）
// fetch() 默认行为，同一请求中重复 fetch 返回缓存结果
const data = await fetch('/api/user', { 
  cache: 'force-cache'  // 默认值
});

// 第二层：Data Cache（持久化）
// Next.js 的数据层缓存，跨请求保持
const users = await db.user.findMany({
  cache: { mode: 'stale-while-revalidate' }
});

// 第三层：Full Route Cache（路由级）
// 整个路由段的预渲染结果
// 使用 generateStaticParams 生成静态路径时自动启用

// 第四层：Router Cache（客户端内存）
// 用户导航时存储的 RSC Payload
// 快速返回之前访问过的页面
```

### 缓存失效策略

```typescript
// 重新验证路径（Path-based）
import { revalidatePath } from 'next/cache';

async function updatePost(id: string, data: PostData) {
  await db.post.update({ where: { id }, data });
  revalidatePath(`/posts/${id}`);  // 清除该路径缓存
}

// 重新验证标签（Tag-based）
import { revalidateTag } from 'next/cache';

async function updateUser(userId: string) {
  await db.user.update({ where: { id: userId }, data: { name: 'New' } });
  revalidateTag('users');  // 清除所有含 'users' 标签的缓存
}

// 重新验证时间（Time-based）
const data = await fetch('/api/stats', {
  next: { revalidate: 3600 }  // 1 小时后重新验证
});
```

### Bundle 体积审计

使用 `@next/bundle-analyzer` 分析客户端 bundle：

```typescript
// next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer({
  // 其他配置
});
```

运行分析：

```bash
ANALYZE=true npm run build
```

关键指标：
- **首屏 JS**：应尽可能小（< 100KB gzip）
- **客户端组件数量**：定期检查，避免不必要的 `'use client'` 蔓延
- **动态导入**：大组件使用 `next/dynamic` 懒加载

### 部分预渲染（Partial Prerendering，PPR）

PPR 是 Next.js 14+ 的实验性功能，结合静态生成和动态渲染：

```typescript
// app/product/[id]/page.tsx
import { Suspense } from 'react';

// 静态内容：产品图片、描述
function ProductInfo({ id }) {
  const product = await getProduct(id);  // 缓存的静态数据
  return (
    <div>
      <img src={product.image} alt={product.name} />
      <h1>{product.name}</h1>
    </div>
  );
}

// 动态内容：库存、价格（实时数据）
function PriceSection({ id }) {
  const price = await getRealTimePrice(id);  // 动态数据
  return <div className="text-2xl font-bold">¥{price}</div>;
}

export default async function ProductPage({ params }) {
  return (
    <div>
      {/* 静态部分预渲染 */}
      <ProductInfo id={params.id} />
      
      {/* 动态部分在运行时获取 */}
      <Suspense fallback={<PriceSkeleton />}>
        <PriceSection id={params.id} />
      </Suspense>
    </div>
  );
}
```

---

## 迁移指南：从 Pages Router 到 App Router

### 核心差异对比

| 特性 | Pages Router | App Router |
|------|-------------|------------|
| 默认渲染环境 | 客户端 | 服务端 |
| 状态管理 | useState + Context | 保持 + Server Components |
| 数据获取 | getServerSideProps | Server Components 直接 await |
| 布局 | pages/\_layout.tsx | app/layout.tsx |
| 错误边界 | pages/\_error.tsx | app/error.tsx |
| 加载状态 | 自定义 | loading.tsx |
| API 路由 | pages/api/* | Route Handlers (app/api/*) |

### 迁移步骤

**1. 创建 app 目录**

```bash
mkdir app
# 移动 pages/_app.tsx → app/layout.tsx
# 移动 pages/_document.tsx 功能到 app/layout.tsx
```

**2. 迁移布局**

```typescript
// Pages Router: pages/_app.tsx
import type { AppProps } from 'next/app';

export default function App({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />;
}

// App Router: app/layout.tsx
import './globals.css';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>{children}</body>
    </html>
  );
}
```

**3. 迁移页面**

```typescript
// Pages Router: pages/posts/[id].tsx
import { GetServerSideProps } from 'next';

export const getServerSideProps: GetServerSideProps = async ({ params }) => {
  const post = await getPost(params.id);
  return { props: { post } };
};

export default function PostPage({ post }) {
  return <div><h1>{post.title}</h1></div>;
}

// App Router: app/posts/[id]/page.tsx
export default async function PostPage({ params }) {
  const post = await getPost(params.id);
  return <div><h1>{post.title}</h1></div>;
}
```

**4. 迁移 API 路由**

```typescript
// Pages Router: pages/api/users.ts
import type { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === 'GET') {
    const users = await db.user.findMany();
    res.status(200).json(users);
  }
}

// App Router: app/api/users/route.ts
import { NextResponse } from 'next/server';
import { db } from '@/lib/db';

export async function GET() {
  const users = await db.user.findMany();
  return NextResponse.json(users);
}
```

**5. 迁移客户端组件**

```typescript
// Pages Router: components/Counter.tsx
import { useState } from 'react';

export function Counter() {  // 默认客户端组件
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// App Router: components/Counter.tsx
'use client';  // 需要显式标记

export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

### 常见问题与解决

**Q：useContext 还能用吗？**

A：可以，但需要注意 Context 只在客户端有效。对于服务端，可以使用 [[../data-structure/monotonous-stack|依赖注入]] 模式，通过 props 传递数据。

**Q：第三方组件库需要全部重写吗？**

A：大多数库需要添加 `'use client'` 才能在 App Router 中使用。建议：
1. 优先使用已支持 App Router 的版本
2. 对于旧库，使用 `next/dynamic` 懒加载

**Q：现有 getStaticProps 如何迁移？**

A：`getStaticProps` 的数据获取逻辑可以直接移到 Server Component 中作为顶层 await：

```typescript
// 之前
export const getStaticProps = async () => ({ props: { data } });

// 现在
export default async function Page() {
  const data = await getData();  // 直接 await
}
```

---

## 总结

React Server Components 代表了 React 生态系统的范式转变。它不是简单的性能优化，而是对组件模型、数据流和架构设计的根本性重新思考。

**核心要点**：

1. **默认服务端**：优先使用 Server Components，只在必要时使用客户端组件
2. **精确边界**：用 `'use client'` 标记最小的必要边界，避免蔓延
3. **组合优于继承**：通过 props/children 传递 Server Components 实现组合
4. **流式渲染**：结合 Suspense 实现渐进式加载，提升感知性能
5. **缓存策略**：理解并合理运用四层缓存体系

掌握 RSC 需要时间和实践，但它带来的性能提升和开发体验改善是值得的。建议从小项目开始，逐步迁移现有应用到 App Router，积累经验后再大规模采用。

[^1]: 本文参考了 React 官方文档和 Next.js App Router 最佳实践
[^2]: RSC Payload 格式详解见 React 团队技术博客
[^3]: Next.js 缓存机制详解见 Next.js 官方文档

---

## 相关主题

- [[../frontend/react-hooks|React Hooks 基础]]
- [[../data-structure/monotonous-stack|单调栈]]（算法相关，与 RSC 数据流思想对比）
- [[../CCF-CSP/CSP202312B|页面渲染性能优化实战]]
