---
title: Micro-Frontends（微前端架构）
date: 2026-04-14
description: 微前端是一种将Web应用分解为更小的独立开发、部署应用的架构模式
tags:
  - micro-frontends
  - turborepo
  - module-federation
  - frontend
  - architecture
draft: false
permalink:
---

## 概述

微前端（Micro-Frontends）是一种架构模式，将前端应用分解为更小的、独立开发和部署的应用，这些应用协同工作形成统一用户体验。[^1]

这种架构模式解决了大型前端应用面临的挑战：
- **团队自治**：不同团队可以独立开发、测试和部署各自的应用
- **技术栈灵活性**：各微前端可以使用不同技术栈
- **独立部署**：单个微前端的更改不需要重新部署整个应用
- **增量迁移**：逐步将单体前端迁移到微前端架构

## 架构模式

### 横向拆分 vs 纵向拆分

| 模式 | 描述 | 适用场景 | 优点 | 缺点 |
|------|------|----------|------|------|
| **横向拆分** | 按页面功能拆分，同一页面多个MFE | 复杂页面、多团队协作 | 团队边界清晰 | 运行时开销大 |
| **纵向拆分** | 按URL路径拆分，每个路径独立应用 | 独立业务线、不同域 | 隔离性好、简单 | 跨页面交互复杂 |

### Vercel的实践

Vercel采用纵向微前端拆分策略，主要考虑：
- 降低构建复杂度
- 减少页面间依赖
- 简化团队边界[^2]

## Module Federation

Module Federation是Webpack 5引入的核心技术，支持多个独立构建形成单一应用。

### 核心概念

```
┌─────────────────────────────────────────────────────────────┐
│                    Host Application                         │
│  ┌─────────────────────────────────────────────────────┐  │
│  │           Module Federation Container                 │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │  │
│  │  │ Remote A │  │ Remote B │  │  Shared  │          │  │
│  │  │ (动态加载) │  │ (动态加载) │  │  (共享)   │          │  │
│  │  └──────────┘  └──────────┘  └──────────┘          │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Webpack配置

#### Host配置

```javascript
// webpack.config.js (Host)
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        // 生产环境使用相对路径
        // 开发环境使用完整URL
        remoteApp: process.env.NODE_ENV === 'production'
          ? 'remoteApp@https://example.com/remoteEntry.js'
          : 'remoteApp@http://localhost:3001/remoteEntry.js',
      },
      shared: {
        react: { singleton: true, requiredVersion: '^18.0.0' },
        'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
      },
    }),
  ],
};
```

#### Remote配置

```javascript
// webpack.config.js (Remote)
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'remoteApp',
      filename: 'remoteEntry.js',
      exposes: {
        './Button': './src/components/Button',
        './Card': './src/components/Card',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
      },
    }),
  ],
};
```

### 异步边界

所有MFE入口必须使用异步加载：

```javascript
// ❌ 错误：直接渲染
import('./bootstrap');
import { render } from 'react-dom';
render(<App />, document.getElementById('root'));

// ✅ 正确：使用bootstrap文件
// src/bootstrap.js
import { render } from 'react-dom';
import App from './App';

render(<App />, document.getElementById('root'));

// src/index.js
import('./bootstrap');
```

## Turborepo微前端支持

Turborepo 2025年10月引入了原生微前端支持，通过集成代理服务器简化本地开发。[^3]

### 工作原理

```
┌─────────────────────────────────────────────────────────────┐
│                    Turborepo Proxy                          │
│                    (自动启动，端口3024)                       │
└─────────────────────────────────────────────────────────────┘
         ↓                                        ↓
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Marketing MFE  │  │   Dashboard MFE  │  │    Docs MFE     │
│  localhost:3000 │  │  localhost:3001  │  │  localhost:3002  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### 配置文件

创建 `microfrontends.json`：

```json
{
  "$schema": "https://turborepo.dev/microfrontends/schema.json",
  "options": {
    "localProxyPort": 3024
  },
  "applications": {
    "web": {
      "packageName": "web",
      "development": {
        "local": { "port": 3000 }
      }
    },
    "marketing": {
      "packageName": "@myapp/marketing",
      "development": {
        "local": { "port": 3001 }
      },
      "routes": [
        { "path": "/", "label": "Home" },
        { "path": "/about", "label": "About" }
      ]
    },
    "dashboard": {
      "packageName": "@myapp/dashboard",
      "development": {
        "local": { "port": 3002 }
      },
      "routes": [
        { "path": "/dashboard", "label": "Dashboard" }
      ]
    }
  }
}
```

### 生产环境集成

开发环境使用Turborepo代理，生产环境可与Vercel微前端服务集成：

```json
// 安装 @vercel/microfrontends 后，Turborepo自动使用Vercel实现
{
  "dependencies": {
    "@vercel/microfrontends": "^1.0.0"
  }
}
```

## Monorepo结构

### 推荐的目录结构

```
myapp/
├── apps/
│   ├── web/                    # Host Shell应用
│   │   ├── src/
│   │   │   ├── App.jsx
│   │   │   └── bootstrap.jsx
│   │   └── webpack.config.js
│   ├── marketing/              # Remote MFE
│   │   ├── src/
│   │   └── webpack.config.js
│   └── dashboard/             # Remote MFE
│       ├── src/
│       └── webpack.config.js
├── packages/
│   ├── store/                 # 共享状态 (Redux)
│   │   └── src/index.js
│   ├── api/                  # 共享API客户端
│   │   └── src/index.js
│   ├── ui/                   # 共享UI组件库
│   │   └── src/index.js
│   └── config/
│       ├── eslint/
│       └── typescript/
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

### 共享包配置

#### 共享状态包

```javascript
// packages/store/src/index.js
import { createStore } from 'redux';
import { Provider } from 'react-redux';
import authReducer from './slices/authSlice';

export const store = createStore(authReducer);
export { Provider };
export * from './hooks';
```

#### 共享API包

```javascript
// packages/api/src/index.js
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.API_URL,
});

apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default apiClient;
```

## 路由与集成

### Host应用路由

```jsx
// apps/web/src/App.jsx
import React, { Suspense, lazy } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { Provider } from '@myapp/store';
import apiClient from '@myapp/api';

// 懒加载远程MFE
const Marketing = lazy(() => import('marketing/App'));
const Dashboard = lazy(() => import('dashboard/App'));

const ErrorBoundary = ({ children }) => (
  <ErrorBoundaryFallback>
    <Suspense fallback={<LoadingSpinner />}>
      {children}
    </Suspense>
  </ErrorBoundaryFallback>
);

function App() {
  return (
    <Provider store={store}>
      <BrowserRouter>
        <ErrorBoundary>
          <Routes>
            <Route path="/" element={<HomePage />} />
            <Route 
              path="/marketing/*" 
              element={<Marketing />} 
            />
            <Route 
              path="/dashboard/*" 
              element={<Dashboard />} 
            />
          </Routes>
        </ErrorBoundary>
      </BrowserRouter>
    </Provider>
  );
}

export default App;
```

### 远程应用导出

```jsx
// apps/marketing/src/App.jsx
import React from 'react';
import { Routes, Route } from 'react-router-dom';
import Home from './pages/Home';
import About from './pages/About';

export default function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/about" element={<About />} />
    </Routes>
  );
}
```

## 开发与生产差异

### 开发环境配置

```javascript
// 开发环境特点
const devConfig = {
  mode: 'development',
  // 使用localhost远程URL
  remotes: {
    remoteApp: 'remoteApp@http://localhost:3001/remoteEntry.js',
  },
  // 开发环境禁用代码分割
  optimization: {
    splitChunks: false,
  },
  // HTTPS配置（可选）
  devServer: {
    https: {
      key: './certificates/localhost.key',
      cert: './certificates/localhost.crt',
    },
  },
};
```

### 生产环境配置

```javascript
// 生产环境特点
const prodConfig = {
  mode: 'production',
  // 使用相对路径
  remotes: {
    remoteApp: 'remoteApp@https://cdn.example.com/remoteEntry.js',
  },
  // 启用vendor代码分割
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
      },
    },
  },
  // 生产模式
  output: {
    publicPath: 'auto',
  },
};
```

## 常见问题与解决方案

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `module not available for eager consumption` | 直接在入口渲染 | 使用`import('./bootstrap')`异步模式 |
| 样式冲突 | 不同MFE使用相同类名 | CSS Modules、Shadow DOM、BEM命名 |
| 状态共享失败 | Redux store未正确配置 | 使用Module Federation shared配置 |
| 加载失败404 | 生产环境publicPath错误 | 检查remoteEntry.js路径 |
| 热更新失效 | WebSocket配置错误 | 检查devServer配置 |

## 最佳实践

### 1. 团队边界划分

```
┌─────────────────────────────────────────────────────────────┐
│                      微前端边界                              │
├─────────────┬─────────────┬─────────────┬─────────────────┤
│   营销团队   │   电商团队   │   用户团队   │    核心团队      │
│             │             │             │                 │
│ - 首页      │ - 商品列表   │ - 登录注册   │ - Host Shell   │
│ - 活动页    │ - 购物车     │ - 个人中心   │ - 路由配置      │
│ - 博客      │ - 结算      │ - 订单      │ - 共享库        │
└─────────────┴─────────────┴─────────────┴─────────────────┘
```

### 2. 共享依赖策略

```javascript
// 推荐：显式共享关键库
shared: {
  react: { singleton: true, requiredVersion: '^18.0.0' },
  'react-dom': { singleton: true, requiredVersion: '^18.0.0' },
  'react-router-dom': { singleton: true },
}

// 不推荐：过度共享
// shared: ['lodash', 'moment', 'axios'] // 导致版本冲突
```

### 3. 增量迁移策略

```javascript
// 使用Feature Flag控制MFE加载
const useMicroFrontend = featureFlags['newMarketing'];

return (
  <Routes>
    <Route path="/marketing" element={
      useMicroFrontend 
        ? <Marketing />  // 新MFE
        : <LegacyMarketing /> // 旧代码
    } />
  </Routes>
);
```

## 参考资料

[^1]: Turborepo Microfrontends Guide - https://turbo.build/docs/guides/microfrontends

[^2]: How Vercel adopted microfrontends - https://vercel.com/blog/how-vercel-adopted-microfrontends

[^3]: Turborepo Microfrontends PR - https://github.com/vercel/turborepo/pull/10982

[^4]: Vercel Microfrontends Module Federation Example - https://github.com/vercel-labs/microfrontends-nextjs-pages-federation
