---
title: The First Page
date: 2026-03-11
description: index
tags:
draft: "false"
permalink:
---
# About
This is the first markdown file in my Quartz site. This page is used for testing the features of Quaerz.
It will be a published version of (part of) my obsidian vault.
# Titles
This section is a **level-1 title**.
## Level-2 Title
### Level-3 Title
#### Level-4 Title
##### Level-5 Title
###### Level-6 Title
Level-6 is the smallest title supported.
# Linebreaks
With
```md
line 1
line 2

line 4 after a newline
```
it looks like: 
line 1
line 2

line 4 after a newline
# Base Syntax
*Italic* and **bold** texts.

> [!NOTE] Callout
> With `NOTE` format.

## Lists
Here is an unordered list:
- entry 1
- entry 2

And an ordered list (after a new line):
1. entry 1
2. entry 1
## Tables

| 分布                 | PMF$p_X(k)$                                  | PDF$f_X(x)$                                                   | 期望                  | 方差                    |
| ------------------ | -------------------------------------------- | ------------------------------------------------------------- | ------------------- | --------------------- |
| $N(\mu ,\sigma^2)$ | -                                            | $\frac{1}{\sqrt{2\pi}\sigma}e^{-\frac{(x-\mu)^2}{2\sigma^2}}$ | $\mu$               | $\sigma^2$            |
| $B(n,p)$           | $C_n^k p^k(1-p)^{n-k}$                       | -                                                             | $np$                | $np(1-p)$             |
| $U(a,b)$           | $\frac{1}{b-a+1}\ ,a\le k\in\mathbb{Z}\le b$ | $\frac{1}{b-a}\ ,a<x<b$                                       | $\frac{a+b}{2}$     | $\frac{(b-a)^2}{12}$  |
| $G(p)$             | $(1-p)^{k-1}p$                               | -                                                             | $\frac{1-p}{p}$     | $\frac{1-p}{p^2}$     |
| $E(\lambda)$       | -                                            | $\lambda e^{-\lambda x}\ ,x>0$                                | $\frac{1}{\lambda}$ | $\frac{1}{\lambda^2}$ |
| $P(\lambda)$       | $\frac{\lambda^k}{k!}e^{-\lambda}$           | -                                                             | $\lambda$           | $\lambda$             |

It is shown in three-line format.
## Math Blocks
An example:
 $$\mathcal{L}(h(t_{1}))=\mathcal{L}(h(t_{0})+\int_{t_{0}}^{t_{1}}f(h(t),t,\theta)dt)=\mathcal{L}(\text{ODESolver}(h(t_{0}),f,t_{0},t_{1},\theta))$$
 And inline math blocks like $V_{\psi}(s)$.
 It seems that it does not support `\tag{}` syntax.
 