---
title: Web无障碍与WCAG指南
date: 2026-04-14
description: 网页无障碍设计指南，详解WCAG标准与 POUR 原则，提供可落地的代码实现与测试方法。
tags:
  - accessibility
  - wcag
  - web-development
draft: false
permalink:
---

## 一、引言：为什么无障碍设计至关重要

网页无障碍（Web Accessibility）是指确保残疾人能够平等地访问和使用网页内容的能力。据世界卫生组织统计，全球约 **15%** 的人口（约 10 亿人）患有某种形式的残疾，其中包括：

| 障碍类型 | 具体表现 | 对网页访问的影响 |
| --------- | -------- | ---------------- |
| 视觉障碍 | 失明、低视力、色盲 | 无法看到视觉内容、需要屏幕阅读器 |
| 听觉障碍 | 失聪、重听 | 无法听取音频内容 |
| 运动障碍 | 瘫痪、震颤 | 无法使用鼠标、依赖键盘操作 |
| 认知障碍 | 智力障碍、阅读困难 | 难以理解复杂内容 |

### 业务价值

无障碍设计不仅是法律责任，更具有重要的业务价值：

- **扩大受众群体**：触达残障用户及其照护者群体
- **改善用户体验**：无障碍最佳实践同样有益于所有用户（如移动端用户、老年用户）
- **搜索引擎优化**：语义化 HTML 和清晰的页面结构有助于 SEO
- **法律合规**：避免诉讼风险和声誉损失

> 「无障碍不是一项功能，而是我们作为人类的基本权利的体现。」—— Tim Berners-Lee，万维网发明者

---

## 二、WCAG 标准概述

### 什么是 WCAG

**WCAG**（Web Content Accessibility Guidelines，网页内容无障碍指南）是由 W3C 的 **WAI**（Web Accessibility Initiative，无障碍倡议组织）制定的无障碍标准。

### 版本演进

| 版本 | 发布年份 | 核心变化 |
| ---- | -------- | -------- |
| WCAG 2.0 | 2008 | 奠定现代无障碍基础，定义 4 大原则和 12 条指南 |
| WCAG 2.1 | 2018 | 新增移动端无障碍、认知障碍相关标准 |
| WCAG 2.2 | 2023 | 新增焦点不可见、非文本对比度等 9 条成功标准 |

### 符合等级

WCAG 定义了三个符合等级：

| 等级 | 含义 | 建议 |
| ---- | ---- | ---- |
| **A 级**（最低） | 必须满足的基本无障碍要求 | 最低合规门槛 |
| **AA 级**（标准） | 绝大多数网站应达到的目标 | 法律引用最广泛的标准 |
| **AAA 级**（最高） | 最高水平的无障碍追求 | 视具体内容可行性决定 |

> **提示**：大多数法规（如 ADA、Section 508）要求达到 **AA 级** 符合。

---

## 三、四大 POUR 原则

WCAG 的核心是四个基本原则的首字母缩写 **POUR**：

### 3.1 可感知性（Perceivable）

信息和用户界面组件必须以可感知的方式呈现给用户。

#### 文本替代方案

```html
<!-- 好的做法：提供有意义的 alt 文本 -->
<img src="submit-button.png" alt="提交表单">

<!-- 装饰性图片使用空 alt -->
<img src="decorative-line.png" alt="">

<!-- 好的做法：复杂图片提供长描述 -->
<img src="chart.png" alt="季度销售趋势图" aria-describedby="chart-desc">
<p id="chart-desc">2024年Q1销售额较Q4增长15%，主要增长来自华东地区...</p>
```

#### 字幕和 transcripts

```html
<!-- 视频字幕 -->
<video>
  <track kind="subtitles" src="intro-zh.vtt" srclang="zh" label="简体中文" default>
</video>

<!-- 音频内容提供文字记录 -->
<a href="podcast-transcript.html">播客文字稿</a>
```

#### 颜色对比度

**WCAG 2.1 AA 级**要求的最低对比度：

| 文本类型 | 最小对比度 |
| -------- | ---------- |
| 正常文本（< 18pt 或 < 14pt 粗体） | **4.5:1** |
| 大文本（≥ 18pt 或 ≥ 14pt 粗体） | **3:1** |
| UI 组件和图形对象 | **3:1** |

```css
/* 符合 WCAG AA 的颜色对比度示例 */
.good-contrast {
  /* 背景 #ffffff 与文字 #333333 对比度约 12.6:1 */
  color: #333333;
  background-color: #ffffff;
}

.large-text-ok {
  /* 大文本 18pt+，对比度 4.5:1 以上即可 */
  color: #767676;
  background-color: #ffffff;
  font-size: 18pt;
}
```

### 3.2 可操作性（Operable）

用户界面组件和导航必须是可操作的。

#### 键盘可访问性

```html
<!-- 好的做法：使用语义化元素获得免费的键盘支持 -->
<button onclick="submitForm()">提交</button>
<a href="/about">关于我们</a>

<!-- 坏的做法：使用 div/span 模拟按钮，无法键盘操作 -->
<div class="btn" onclick="submitForm()">提交</div>
```

#### 焦点可见性

```css
/* 确保焦点状态清晰可见 */
:focus-visible {
  outline: 3px solid #0066cc;
  outline-offset: 2px;
}

/* 不要移除焦点指示器 */
:focus {
  outline: none; /* 错误做法 */
}
```

#### 键盘陷阱避免

```html
<!-- 模态框的正确实现：聚焦陷阱 + ESC 关闭 -->
<div role="dialog" aria-modal="true" aria-labelledby="dialog-title">
  <h2 id="dialog-title">确认删除</h2>
  <p>确定要删除此项吗？</p>
  <button onclick="closeDialog()">取消</button>
  <button onclick="confirmDelete()">确定</button>
</div>

<script>
// 模态框打开时聚焦到第一个按钮
// 模态框关闭时焦点返回到触发元素
</script>
```

#### 时间限制处理

```html
<!-- 不要设置硬性时间限制，或提供扩展选项 -->
<!-- 如果必须有时间限制： -->
<div role="alert">
  您的会话将在 5 分钟后过期。<a href="/extend">延长会话</a>
</div>
```

### 3.3 可理解性（Understandable）

信息和用户界面的操作必须是可以理解的。

#### 语言清晰度

```html
<!-- 声明页面语言 -->
<html lang="zh-CN">

<!-- 声明部分内容的语言变化 -->
<p>苹果的英文是 <span lang="en">Apple</span></p>
```

#### 表单错误处理

```html
<!-- 错误提示应该具体且有帮助 -->
<div role="alert">
  <p>请修正以下错误：</p>
  <ul>
    <li><a href="#email">邮箱格式不正确</a></li>
    <li><a href="#password">密码必须至少 8 个字符</a></li>
  </ul>
</div>

<label for="email">邮箱地址</label>
<input type="email" id="email" aria-invalid="true" aria-describedby="email-error">
<span id="email-error">请输入有效的邮箱地址，如 user@example.com</span>
```

### 3.4 健壮 性（Robust）

内容必须足够健壮，能够被各种用户代理（包括辅助技术）可靠地解释。

#### 语义化 HTML

```html
<!-- 好的导航结构 -->
<header>
  <nav aria-label="主导航">
    <ul>
      <li><a href="/" aria-current="page">首页</a></li>
      <li><a href="/products">产品</a></li>
    </ul>
  </nav>
</header>

<!-- 好的文章结构 -->
<article>
  <header>
    <h1>文章标题</h1>
    <p>发布日期：2026-04-14</p>
  </header>
  <section>
    <h2>第一部分</h2>
    <p>内容...</p>
  </section>
</article>

<!-- 坏的做法：使用 div 代替语义元素 -->
<div class="header">...</div>
<div class="nav">...</div>
<div class="content">...</div>
```

#### ARIA 的正确使用

**黄金法则**：优先使用原生 HTML 元素，只有在原生元素无法满足时才使用 ARIA。

```html
<!-- 不需要 ARIA：原生元素已提供语义 -->
<button>保存</button>

<!-- 需要 ARIA：自定义组件需要明确语义 -->
<div class="tabs" role="tablist">
  <button role="tab" aria-selected="true" aria-controls="panel-1">标签1</button>
  <button role="tab" aria-selected="false" aria-controls="panel-2">标签2</button>
</div>
<div id="panel-1" role="tabpanel">内容1</div>
<div id="panel-2" role="tabpanel" hidden>内容2</div>
```

常用 ARIA 属性：

| 属性 | 用途 | 示例 |
| ---- | ---- | ---- |
| `aria-label` | 提供可访问名称 | `<nav aria-label="主导航">` |
| `aria-expanded` | 指示展开/折叠状态 | `<button aria-expanded="false">` |
| `aria-modal` | 指示模态对话框 | `<div role="dialog" aria-modal="true">` |
| `aria-live` | 动态内容通知 | `<div aria-live="polite">` |

---

## 四、实现模式

### 4.1 语义化 HTML 最佳实践

#### 按钮 vs div

```html
<!-- ✅ 正确：使用 <button> 元素 -->
<button type="button" onclick="saveData()">保存</button>

<!-- ❌ 错误：使用 <div> 模拟按钮 -->
<div class="btn" onclick="saveData()">保存</div>
```

#### 标题层级

```html
<!-- ✅ 正确：保持正确的标题顺序 -->
<h1>页面标题</h1>
  <h2>主要章节</h2>
    <h3>子章节</h3>
  <h2>下一个主要章节</h2>

<!-- ❌ 错误：跳过标题级别 -->
<h1>页面标题</h1>
<h3>直接使用 h3</h3>
```

### 4.2 可访问表单实现

```html
<form action="/submit" method="post">
  <fieldset>
    <legend>用户注册</legend>
    
    <div>
      <label for="username">用户名</label>
      <input 
        type="text" 
        id="username" 
        name="username"
        required
        aria-required="true"
        autocomplete="username"
      >
    </div>
    
    <div>
      <label for="email">邮箱</label>
      <input 
        type="email" 
        id="email" 
        name="email"
        required
        aria-required="true"
        autocomplete="email"
      >
      <span id="email-hint">我们将发送验证邮件到此地址</span>
    </div>
    
    <div>
      <label for="password">密码</label>
      <input 
        type="password" 
        id="password" 
        name="password"
        required
        aria-required="true"
        minlength="8"
        aria-describedby="password-requirements"
      >
      <p id="password-requirements">密码至少 8 个字符，包含数字和字母</p>
    </div>
    
    <button type="submit">注册</button>
  </fieldset>
</form>
```

### 4.3 模态框/对话框焦点管理

```javascript
class ModalManager {
  constructor(triggerSelector, modalSelector) {
    this.trigger = document.querySelector(triggerSelector);
    this.modal = document.querySelector(modalSelector);
    this.trigger.focus();
    this.init();
  }

  init() {
    // 打开模态框
    this.trigger.addEventListener('click', () => this.open());
    
    // 监听键盘事件
    this.modal.addEventListener('keydown', (e) => this.handleKeyboard(e));
  }

  open() {
    this.modal.hidden = false;
    // 聚焦到模态框的第一个可交互元素
    const focusableElements = this.modal.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    if (focusableElements.length) {
      focusableElements[0].focus();
    }
    
    // 保存触发元素引用，用于关闭后焦点返回
    this.triggerElement = document.activeElement;
  }

  close() {
    this.modal.hidden = true;
    // 焦点返回到触发元素
    if (this.triggerElement) {
      this.triggerElement.focus();
    }
  }

  handleKeyboard(e) {
    // Trap focus within modal
    if (e.key === 'Tab') {
      const focusableElements = this.modal.querySelectorAll(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      );
      const firstElement = focusableElements[0];
      const lastElement = focusableElements[focusableElements.length - 1];

      if (e.shiftKey && document.activeElement === firstElement) {
        e.preventDefault();
        lastElement.focus();
      } else if (!e.shiftKey && document.activeElement === lastElement) {
        e.preventDefault();
        firstElement.focus();
      }
    }
    
    // ESC 关闭模态框
    if (e.key === 'Escape') {
      this.close();
    }
  }
}
```

### 4.4 下拉菜单/折叠面板

```html
<details class="accordion">
  <summary aria-expanded="false">
    <span>常见问题</span>
    <svg aria-hidden="true" class="icon" viewBox="0 0 24 24">
      <path d="M7 10l5 5 5-5"/>
    </svg>
  </summary>
  <div class="content">
    <p>这里是问题的答案内容...</p>
  </div>
</details>
```

```javascript
// 使用 <details> 元素可以获得免费的无障碍支持
// 如果需要自定义样式，可以增强它：

document.querySelectorAll('details.accordion').forEach(details => {
  details.querySelector('summary').addEventListener('click', (e) => {
    e.preventDefault();
    const isOpen = details.open;
    // 关闭所有其他面板
    document.querySelectorAll('details.accordion').forEach(d => d.open = false);
    // 切换当前面板
    details.open = !isOpen;
    // 更新 aria-expanded
    details.querySelector('summary').setAttribute('aria-expanded', !isOpen);
  });
});
```

---

## 五、测试方法

### 5.1 自动化工具

#### axe-core（推荐）

```bash
# Chrome 扩展：axe DevTools
# 或在命令行使用：
npm install @axe-core/cli
axe https://example.com
```

```javascript
// 在测试中集成 axe
const AxeBuilder = require('@axe-core/test').default;
const cheerio = require('cheerio');

async function runAxeTest(html) {
  const results = await new AxeBuilder({ html }).analyze();
  return results;
}
```

#### WAVE

[WAVE](https://wave.webaim.org/) 是 WebAIM 提供的无障碍测试工具，提供浏览器扩展和在线测试。

### 5.2 手动测试

#### 键盘-only 测试清单

| 测试项 | 预期行为 |
| ------ | -------- |
| Tab 键导航 | 可以顺序访问所有可交互元素 |
| Enter/Space 激活 | 按钮和链接可正常激活 |
| ESC 键 | 模态框和下拉菜单可关闭 |
| 方向键 | 在标签组、菜单内可正常导航 |
| 焦点指示器 | 清晰可见当前焦点位置 |

#### 屏幕阅读器测试清单

| 测试项 | 检查要点 |
| ------ | -------- |
| 页面标题 | 正确描述页面内容 |
| 标题结构 | 标题层级清晰、可跳转到重要章节 |
| 图像 alt | 有意义的描述，装饰性图像为空 |
| 表单关联 | 标签与输入框正确关联 |
| 动态更新 | 使用 aria-live 通知变化 |

常用屏幕阅读器：

- **Windows**：NVDA（免费）、JAWS
- **macOS**：VoiceOver（系统自带）
- **iOS**：VoiceOver
- **Android**：TalkBack

### 5.3 完整测试检查清单

#### 感知性检查

- [ ] 所有图像有 alt 文本
- [ ] 视频有字幕
- [ ] 音频内容有文字稿
- [ ] 颜色对比度符合标准
- [ ] 内容不只依赖颜色传达信息

#### 可操作性检查

- [ ] 所有功能可通过键盘操作
- [ ] 焦点指示器清晰可见
- [ ] 没有键盘陷阱
- [ ] 时间限制可调整或移除
- [ ] 跳过链接可用

#### 可理解性检查

- [ ] 页面语言已声明
- [ ] 链接文本有意义
- [ ] 表单错误提示清晰
- [ ] 导航一致

#### 健壮性检查

- [ ] HTML 验证通过
- [ ] ARIA 使用正确
- [ ] 跨浏览器测试通过
- [ ] 辅助技术兼容

---

## 六、法律法规

### 6.1 主要法规

| 法规 | 适用范围 | 要求 |
| ---- | -------- | ---- |
| **欧洲无障碍法案**（EAA） | 欧盟成员国 | 2025 年起强制执行 |
| **ADA**（美国残障法案） | 美国 | Title III 适用于商业网站 |
| **Section 508** | 美国联邦机构 | 政府网站必须符合 |
| **EN 301 549** | 欧盟公共网站 | 政府采购标准 |

### 6.2 合规建议

1. **风险评估**：评估现有网站的无障碍合规水平
2. **优先级制定**：优先修复 A 级和 AA 级问题
3. **持续监控**：将无障碍测试集成到 CI/CD 流程
4. **用户反馈**：建立无障碍问题反馈渠道

---

## 七、参考资料

[^1]：W3C WAI - [Web Content Accessibility Guidelines (WCAG) Overview](https://www.w3.org/WAI/standards-guidelines/wcag/)

[^2]：WebAIM - [Introduction to Web Accessibility](https://webaim.org/intro/)

[^3]：MDN Web Docs - [Accessibility tutorials](https://developer.mozilla.org/en-US/docs/Learn/Accessibility)

[^4]：axe DevTools - [Accessibility Testing Tools](https://www.deque.com/axe/)

[^5]：W3C WAI - [How to Meet WCAG (Quick Reference)](https://www.w3.org/WAI/WCAG21/quickref/)

[^6]：EU - [European Accessibility Act](https://ec.europa.eu/social/main.jsp?catId=1202)

[^7]：US DOJ - [ADA and Web Accessibility](https://www.ada.gov/resources/web-guidance/)

[^8]：WebAIM - [WAVE Web Accessibility Evaluation Tool](https://wave.webaim.org/)

