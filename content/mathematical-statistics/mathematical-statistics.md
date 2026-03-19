---
title: 数理统计学 (Mathematical Statistics)
date: 2026-03-19
description: 数理统计学是研究如何从样本数据推断总体特征的学科
tags:
  - mathematical-statistics
  - statistics
draft: false
permalink:
---

# 数理统计学 (Mathematical Statistics)

数理统计学是概率论的逆问题——在已知总体分布形式（或假设其形式）的前提下，研究如何利用样本数据对总体参数进行推断。本 wiki 系统性地介绍数理统计学的核心理论体系。

## 内容导航

| 章节 | 主题 | 关键词 |
|------|------|--------|
| [[../mathematical-statistics/foundations-of-sampling|抽样分布基础]] | 总体与样本、统计量、经验分布函数、格利文科-坎泰利定理 |
| [[../mathematical-statistics/three-sampling-distributions|三大抽样分布]] | 卡方分布、$t$ 分布、$F$ 分布 |
| [[../mathematical-statistics/order-statistics|次序统计量]] | 极值、$k$-次序统计量、样本中位数、样本极差 |
| [[../mathematical-statistics/sufficiency-and-data-reduction|充分性与数据压缩]] | 充分统计量、因子分解定理、指数族分布 |
| [[../mathematical-statistics/point-estimation|点估计]] | 矩估计、极大似然估计、无偏性、有效性、一致性 |
| [[../mathematical-statistics/interval-estimation|区间估计]] | 置信水平、置信区间、枢轴量法 |
| [[../mathematical-statistics/hypothesis-testing|假设检验]] | 原假设、备择假设、$p$ 值、威力函数、Neyman-Pearson 引理 |
| [[../mathematical-statistics/bayesian-inference|贝叶斯推断]] | 先验分布、后验分布、共轭先验 |

## 知识链路

数理统计的学习路径遵循以下逻辑递进关系：

1. **抽样分布基础** $\rightarrow$ 理解从总体到样本的随机性映射，是整个数理统计的起点
2. **三大抽样分布** $\rightarrow$ 基于正态总体的衍生分布，构成假设检验的理论基石
3. **次序统计量** $\rightarrow$ 样本排序后的特性，在非参数统计中有重要应用
4. **充分性与数据压缩** $\rightarrow$ 如何不损失信息地提取样本特征，引出指数族分布
5. **点估计** $\rightarrow$ 利用样本给出参数的具体数值估计
6. **区间估计** $\rightarrow$ 给出参数的可能范围
7. **假设检验** $\rightarrow$ 基于样本对总体断言进行决策
8. **贝叶斯推断** $\rightarrow$ 将参数视为随机变量的哲学范式

## 核心思想

数理统计的核心在于**以样本推断总体**。具体而言：

- **描述统计**：对样本进行整理、概括和展示
- **推断统计**：基于样本特征对总体参数或总体分布进行推断

本 wiki 主要聚焦于推断统计的理论体系，包括参数估计（点估计、区间估计）和假设检验两大支柱。
