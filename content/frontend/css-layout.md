---
title: CSS布局（Flexbox与Grid）
date: 2026-04-13
description: 深入理解CSS Flexbox与Grid布局系统，掌握现代CSS布局的核心技能
tags:
  - css
  - flexbox
  - grid
  - frontend
draft: false
permalink:
---

## 概述

CSS布局经历了从浮动（float）、行内块（inline-block）到表格（table）的漫长演变。Flexbox（弹性盒布局）和CSS Grid（网格布局）是现代CSS提供的两种强大布局系统，它们彻底改变了网页布局的方式。[^1]

**核心原则**：
- **Flexbox**：一维布局，处理单行或单列元素
- **Grid**：二维布局，同时处理行和列

## Flexbox（弹性盒布局）

Flexbox是一种**单轴布局系统**，沿主轴（main axis）或交叉轴（cross axis）排列项目。[^2]

### 核心概念

```
┌─────────────────────────────────────────┐
│              Flex Container              │
│  ┌─────────────────────────────────┐    │
│  │  Main Axis (水平)               │    │
│  │  ←───────────────────────────→  │    │
│  │  ┌───┐  ┌───┐  ┌───┐  ┌───┐    │    │
│  │  │ 1 │  │ 2 │  │ 3 │  │ 4 │    │    │ ← Flex Items
│  │  └───┘  └───┘  └───┘  └───┘    │    │
│  │           Cross Axis (垂直)     │    │
│  │           │                   │    │
│  │           ↓                   │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### 容器属性

```css
.container {
    display: flex;
    
    /* 方向与换行 */
    flex-direction: row | row-reverse | column | column-reverse;
    flex-wrap: nowrap | wrap | wrap-reverse;
    
    /* 主轴对齐 */
    justify-content: flex-start | flex-end | center | space-between | space-around | space-evenly;
    
    /* 交叉轴对齐 */
    align-items: stretch | flex-start | flex-end | center | baseline;
    
    /* 多行对齐 */
    align-content: flex-start | flex-end | center | space-between | space-around | stretch;
    
    /* 间距 */
    gap: 10px;
}
```

### 项目属性

```css
.item {
    /* 扩展与收缩 */
    flex-grow: 0;    /* 放大比例 */
    flex-shrink: 1;  /* 缩小比例 */
    flex-basis: auto; /* 初始大小 */
    
    /* 简写 */
    flex: 1 1 auto;  /* flex: [grow] [shrink] [basis] */
    
    /* 排序 */
    order: 0;  /* 数值越小越靠前 */
    
    /* 单独对齐 */
    align-self: auto | flex-start | flex-end | center | stretch;
}
```

### 典型应用场景

| 场景 | 推荐 | 原因 |
|------|------|------|
| 导航栏 | Flexbox | 单行布局，元素自适应宽度 |
| 表单标签+输入框 | Flexbox | 一维对齐 |
| 卡片内按钮组 | Flexbox | 按钮均匀分布 |
| 居中单个元素 | Flexbox或Grid | 两种皆可 |

## CSS Grid（网格布局）

Grid是一个**二维系统**，可以同时处理列和行，将项目放置到网格上。[^3]

### 核心概念

```
┌─────────────────────────────────────────┐
│              Grid Container              │
│  ┌────────┬────────┬────────┐          │
│  │        │        │        │          │
│  │  Cell  │  Cell  │  Cell  │   Row 1  │
│  │   1,1  │   1,2  │   1,3  │          │
│  ├────────┼────────┼────────┤          │
│  │        │        │        │          │
│  │  Cell  │  Cell  │  Cell  │   Row 2  │
│  │   2,1  │   2,2  │   2,3  │          │
│  └────────┴────────┴────────┘          │
│         ↑         ↑         ↑            │
│      Column 1  Column 2  Column 3        │
└─────────────────────────────────────────┘
```

### 容器属性

```css
.container {
    display: grid;
    
    /* 定义网格轨道 */
    grid-template-columns: 200px 1fr 200px;
    grid-template-rows: auto 1fr auto;
    
    /* 简写 */
    grid-template: auto 1fr auto / 200px 1fr 200px;  /* rows / columns */
    
    /* 使用函数 */
    grid-template-columns: repeat(3, 1fr);           /* 三列等宽 */
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));  /* 响应式 */
    grid-template-columns: 1fr 2fr 1fr;              /* 中间列更宽 */
    
    /* 间距 */
    gap: 20px;
    row-gap: 20px;
    column-gap: 20px;
    
    /* 命名区域 */
    grid-template-areas:
        "header header header"
        "sidebar main aside"
        "footer footer footer";
}
```

### 项目属性

```css
.item {
    /* 放置到指定网格线 */
    grid-column: 1 / 3;      /* 从第1列线到第3列线 */
    grid-row: 1 / 2;
    
    /* 简写 */
    grid-area: 1 / 1 / 2 / 3;  /* row-start / col-start / row-end / col-end */
    
    /* 使用命名区域 */
    grid-area: header;
    
    /* 跨越 */
    grid-column: span 2;     /* 跨越2列 */
    grid-row: span 3;        /* 跨越3行 */
    
    /* 对齐 */
    justify-self: start | end | center | stretch;
    align-self: start | end | center | stretch;
}
```

### Subgrid（子网格）

Subgrid允许嵌套网格与父网格对齐，是Grid最强大的特性之一：

```css
.card-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 1.5rem;
}

.card {
    /* 子网格与父网格轨道对齐 */
    display: grid;
    grid-row: span 3;
    grid-template-rows: subgrid;
}
```

## Flexbox vs Grid：决策框架

记住这个核心规则：**Flexbox用于"线条"，Grid用于"地图"**。[^4]

| 问题 | 答案 |
|------|------|
| 需要同时控制行和列？ | Grid |
| 内容决定大小，一维排列？ | Flexbox |
| 需要跨行跨列对齐？ | Grid |
| 元素在一条线上等分空间？ | Flexbox |
| 页面整体骨架布局？ | Grid |
| 组件内部微布局？ | Flexbox |

### 最佳实践：组合使用

```css
/* Grid用于页面结构 */
.page {
    display: grid;
    grid-template-columns: 240px 1fr;
    grid-template-rows: 64px 1fr auto;
    grid-template-areas:
        "header header"
        "sidebar main"
        "footer footer";
    min-height: 100vh;
}

/* Flexbox用于组件内部 */
.header {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.main {
    display: flex;
    flex-direction: column;
}

.card {
    display: flex;
    flex-direction: column;
}

.card .buttons {
    display: flex;
    gap: 0.5rem;
    margin-top: auto;  /* 按钮组对齐到底部 */
}
```

## 实战模式

### 1. 应用外壳（App Shell）

```css
.app {
    display: grid;
    grid-template-columns: 280px 1fr;
    grid-template-rows: 64px 1fr 48px;
    grid-template-areas:
        "header header"
        "sidebar main"
        "sidebar footer";
    height: 100vh;
}

.app > header { grid-area: header; }
.app > aside  { grid-area: sidebar; }
.app > main   { grid-area: main; }
.app > footer { grid-area: footer; }
```

### 2. 响应式卡片网格

```css
.card-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 1.5rem;
}
```

### 3. 水平工具栏

```css
.toolbar {
    display: flex;
    gap: 0.5rem;
}

.toolbar > .spacer {
    flex: 1;
}
```

### 4. Sticky Footer

```css
.page {
    display: flex;
    flex-direction: column;
    min-height: 100vh;
}

.main {
    flex: 1;
}
```

### 5. 表单布局

```css
.form {
    display: grid;
    grid-template-columns: auto 1fr;
    gap: 0.75rem 1rem;
    align-items: center;
}
```

## 性能注意事项

现代浏览器已高度优化Flexbox和Grid性能，但以下几点仍需注意：

1. **避免布局抖动（Layout Thrashing）**：批量DOM更新，避免频繁读写布局属性
2. **选择器效率**：避免深层嵌套选择器
3. **`content-visibility: auto`**：对长列表启用虚拟化渲染
4. **复杂Grid计算**：`minmax()`、`auto-fill`等特性可能增加计算复杂度

## 常见错误

1. **用Flexbox做Grid的工作**：嵌套多层Flexbox实现二维布局 → 应使用Grid
2. **忽略`min-width: 0`**：Flex项目默认不可收缩，需设置`min-width: 0`防止内容溢出
3. **忽略`flex-basis`**：未明确设置初始大小，依赖默认值导致意外行为
4. **不用`gap`用margin**：Grid/Flexbox项目间应用`gap`而非margin

## 参考资料

[^1]: [CSS Grid vs Flexbox: Decision Rules and Layout Recipes That Survive Production](https://cr0x.net/en/css-grid-vs-flexbox-rules-recipes/)
[^2]: [CSS Flexbox vs Grid: When to Use Which (2026 Guide)](https://devtoolbox.dedyn.io/blog/css-flexbox-vs-grid)
[^3]: [CSS Grid Layout: The Complete Guide for 2026](https://devtoolbox.dedyn.io/blog/css-grid-complete-guide)
[^4]: [Ultimate Guide to CSS Grid and Flexbox Layouts in 2024](https://blog.pixelfreestudio.com/ultimate-guide-to-css-grid-and-flexbox-layouts-in-2024/)
