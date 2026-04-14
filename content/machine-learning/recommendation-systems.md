---
title: Recommendation Systems
date: 2026-04-14
description: 推荐系统核心算法与架构
tags:
  - recommendation
  - machine-learning
  - collaborative-filtering
draft: true
permalink:
---

# Recommendation Systems

## 概述

推荐系统是信息过滤系统，通过分析用户行为和内容特征，预测用户兴趣并推送个性化内容。广泛应用于电商、视频、音乐、新闻等领域。

## 推荐系统分类

```
┌─────────────────────────────────────────────────────────────┐
│                      推荐系统类型                            │
├─────────────────┬─────────────────┬─────────────────────────┤
│  协同过滤 (CF)   │   基于内容 (CB)   │      混合推荐           │
│  User-based     │   Item-based    │   多种方法组合           │
│  Item-based     │   Feature-based │   特征交叉               │
└─────────────────┴─────────────────┴─────────────────────────┘
```

## 协同过滤（Collaborative Filtering）

### 核心思想

「相似用户有相似偏好」—— 利用用户-物品交互矩阵发现相似性。

### 用户相似度计算

```python
import numpy as np
from scipy.spatial.distance import cosine

def cosine_similarity(v1, v2):
    """余弦相似度"""
    return 1 - cosine(v1, v2)

def pearson_correlation(ratings, user1, user2):
    """皮尔逊相关系数"""
    # 找到共同评分的物品
    common_items = np.where(
        (ratings[user1] != 0) & (ratings[user2] != 0)
    )[0]
    
    if len(common_items) == 0:
        return 0
    
    r1 = ratings[user1, common_items]
    r2 = ratings[user2, common_items]
    
    # 去中心化
    r1_centered = r1 - np.mean(ratings[user1, ratings[user1] != 0])
    r2_centered = r2 - np.mean(ratings[user2, ratings[user2] != 0])
    
    return np.dot(r1_centered, r2_centered) / (
        np.linalg.norm(r1_centered) * np.linalg.norm(r2_centered) + 1e-8
    )
```

### User-based CF

```python
def user_based_recommend(user_id, ratings, k=10):
    """
    基于用户的协同过滤推荐
    """
    n_users, n_items = ratings.shape
    
    # Step 1: 计算目标用户与所有其他用户的相似度
    similarities = []
    for uid in range(n_users):
        if uid != user_id:
            sim = cosine_similarity(ratings[user_id], ratings[uid])
            similarities.append((uid, sim))
    
    # Step 2: 找到最相似的 k 个用户
    similarities.sort(key=lambda x: x[1], reverse=True)
    k_nearest = similarities[:k]
    
    # Step 3: 使用加权平均预测评分
    predictions = np.zeros(n_items)
    for item_id in range(n_items):
        if ratings[user_id, item_id] == 0:  # 未评分的物品
            weighted_sum = 0
            sim_sum = 0
            for neighbor_id, sim in k_nearest:
                if ratings[neighbor_id, item_id] > 0:
                    weighted_sum += sim * ratings[neighbor_id, item_id]
                    sim_sum += abs(sim)
            predictions[item_id] = weighted_sum / (sim_sum + 1e-8)
    
    return predictions
```

### Item-based CF

```python
def item_based_recommend(user_id, ratings, k=10):
    """
    基于物品的协同过滤推荐
    """
    n_users, n_items = ratings.shape
    
    # Step 1: 计算物品相似度矩阵
    item_similarities = np.zeros((n_items, n_items))
    for i in range(n_items):
        for j in range(n_items):
            if i != j:
                # 只使用共同评分用户计算相似度
                common_users = np.where(
                    (ratings[:, i] != 0) & (ratings[:, j] != 0)
                )[0]
                if len(common_users) > 0:
                    item_similarities[i, j] = cosine_similarity(
                        ratings[common_users, i],
                        ratings[common_users, j]
                    )
    
    # Step 2: 对用户已评分的物品，找到最相似的 k 个物品推荐
    predictions = np.zeros(n_items)
    rated_items = np.where(ratings[user_id] > 0)[0]
    
    for item_id in range(n_items):
        if ratings[user_id, item_id] == 0:
            # 找到用户评分物品中与此物品最相似的
            sim_scores = []
            for rated_item in rated_items:
                sim_scores.append(item_similarities[item_id, rated_item])
            # 加权平均
            top_k_idx = np.argsort(sim_scores)[-k:]
            predictions[item_id] = np.mean([
                sim_scores[i] * ratings[user_id, rated_items[i]]
                for i in top_k_idx
            ])
    
    return predictions
```

## 矩阵分解（Matrix Factorization）

### SVD（Singular Value Decomposition）

将用户-物品评分矩阵分解为隐因子：

$$
R_{m \times n} \approx U_{m \times k} \cdot \Sigma_{k \times k} \cdot V_{k \times n}^T
$$

```python
from sklearn.decomposition import TruncatedSVD

def svd_recommend(user_id, ratings, n_components=20):
    """
    基于 SVD 的推荐
    """
    # SVD 分解
    svd = TruncatedSVD(n_components=n_components)
    user_factors = svd.fit_transform(ratings)  # 用户隐向量
    item_factors = svd.components_.T           # 物品隐向量
    
    # 重构评分矩阵
    reconstructed = np.dot(user_factors, svd.components_)
    
    # 返回用户未评分物品的预测评分
    predictions = reconstructed[user_id]
    return predictions
```

### ALS（Alternating Least Squares）

交替最小二乘法，适合大规模稀疏矩阵：

```python
from scipy.sparse import csr_matrix
from numpy.linalg import solve

def als_recommend(ratings, n_factors=10, n_iterations=10, reg=0.1):
    """
    ALS 矩阵分解
    """
    n_users, n_items = ratings.shape
    
    # 随机初始化隐因子
    U = np.random.rand(n_users, n_factors)
    V = np.random.rand(n_items, n_factors)
    
    ratings_csr = csr_matrix(ratings)
    
    for iteration in range(n_iterations):
        # 固定 V，更新 U
        for u in range(n_users):
            items_rated = ratings_csr[u].indices
            if len(items_rated) == 0:
                continue
            V_i = V[items_rated]
            R_i = ratings[u, items_rated]
            A = V_i.T @ V_i + reg * np.eye(n_factors)
            b = V_i.T @ R_i
            U[u] = solve(A, b)
        
        # 固定 U，更新 V
        for i in range(n_items):
            users_rated = np.where(ratings[:, i] > 0)[0]
            if len(users_rated) == 0:
                continue
            U_u = U[users_rated]
            R_u = ratings[users_rated, i]
            A = U_u.T @ U_u + reg * np.eye(n_factors)
            b = U_u.T @ R_u
            V[i] = solve(A, b)
    
    return U @ V.T
```

## 基于内容的推荐

### 特征表示

```python
from sklearn.feature_extraction.text import TfidfVectorizer

def build_item_profiles(items, descriptions):
    """
    构建物品特征向量
    """
    tfidf = TfidfVectorizer(stop_words='english')
    tfidf_matrix = tfidf.fit_transform(descriptions)
    
    return tfidf, tfidf_matrix

def content_based_recommend(user_id, items_profile, user_history, n=10):
    """
    基于内容的推荐
    """
    # 用户画像：喜欢物品特征的加权平均
    user_profile = np.zeros(items_profile.shape[1])
    for item_id in user_history:
        user_profile += items_profile[item_id]
    user_profile /= len(user_history) if len(user_history) > 0 else 1
    
    # 计算用户画像与所有物品的相似度
    scores = np.dot(items_profile, user_profile)
    
    # 排除用户已交互的物品
    scores[user_history] = -np.inf
    
    # 返回 top-N
    top_n = np.argsort(scores)[-n:][::-1]
    return top_n
```

## 深度学习推荐

### Neural Collaborative Filtering

```python
import torch
import torch.nn as nn

class NCF(nn.Module):
    """Neural Collaborative Filtering"""
    def __init__(self, n_users, n_items, embed_dim=32, hidden_dims=[64, 32]):
        super().__init__()
        
        # GMF (Generalized Matrix Factorization)
        self.user_embed_gmf = nn.Embedding(n_users, embed_dim)
        self.item_embed_gmf = nn.Embedding(n_items, embed_dim)
        
        # MLP
        self.user_embed_mlp = nn.Embedding(n_users, embed_dim)
        self.item_embed_mlp = nn.Embedding(n_items, embed_dim)
        
        mlp_layers = []
        input_dim = embed_dim * 2
        for hidden_dim in hidden_dims:
            mlp_layers.append(nn.Linear(input_dim, hidden_dim))
            mlp_layers.append(nn.ReLU())
            input_dim = hidden_dim
        self.mlp = nn.Sequential(*mlp_layers)
        
        # 输出层
        self.output = nn.Linear(embed_dim + hidden_dims[-1], 1)
        self.sigmoid = nn.Sigmoid()
    
    def forward(self, user_ids, item_ids):
        # GMF part
        user_gmf = self.user_embed_gmf(user_ids)
        item_gmf = self.item_embed_gmf(item_ids)
        gmf_output = user_gmf * item_gmf
        
        # MLP part
        user_mlp = self.user_embed_mlp(user_ids)
        item_mlp = self.item_embed_mlp(item_ids)
        mlp_input = torch.cat([user_mlp, item_mlp], dim=-1)
        mlp_output = self.mlp(mlp_input)
        
        # 拼接并输出
        combined = torch.cat([gmf_output, mlp_output], dim=-1)
        output = self.output(combined)
        return self.sigmoid(output)
```

## 评估指标

### 离线评估

| 指标 | 公式/描述 |
|------|----------|
| RMSE | $\sqrt{\frac{1}{N}\sum(\hat{r}-r)^2}$ |
| MAE | $\frac{1}{N}\sum|\hat{r}-r|$ |
| Precision@K | $TP / (TP+FP)$ @ K |
| Recall@K | $TP / (TP+FN)$ @ K |
| MAP@K | Mean Average Precision |
| NDCG@K | Normalized DCG @ K |

```python
def precision_at_k(recommended, relevant, k):
    """Precision@K"""
    return len(set(recommended[:k]) & set(relevant)) / k

def recall_at_k(recommended, relevant, k):
    """Recall@K"""
    return len(set(recommended[:k]) & set(relevant)) / len(relevant) if len(relevant) > 0 else 0

def ndcg_at_k(recommended, relevant, k):
    """NDCG@K"""
    dcg = 0
    for i, item in enumerate(recommended[:k]):
        if item in relevant:
            dcg += 1 / np.log2(i + 2)
    
    idcg = sum(1 / np.log2(i + 2) for i in range(min(len(relevant), k)))
    return dcg / idcg if idcg > 0 else 0
```

### A/B 测试

真实用户反馈验证推荐效果：

- 点击率（CTR）
- 转化率（CVR）
- 停留时长
- 复购率

## 推荐系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                      推荐系统架构                            │
├─────────────────────────────────────────────────────────────┤
│  数据采集 ──▶ 特征工程 ──▶ 模型训练 ──▶ 在线服务              │
│                                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────────┐  │
│  │ 用户行为│  │ 特征存储 │  │ 召回层   │  │  排序层 (LTR)   │  │
│  │ (点击/  │──▶│ (Redis/ │──▶│ (多路   │──▶│  (Learn to     │  │
│  │ 购买)   │  │ Hive)   │  │ 召回)   │  │   Rank)        │  │
│  └─────────┘  └─────────┘  └─────────┘  └─────────────────┘  │
│                                                             │
│                    ┌─────────────────┐                     │
│                    │  业务规则/重排   │                     │
│                    │  (去重、过滤、  │                     │
│                    │   热门打压)      │                     │
│                    └─────────────────┘                     │
└─────────────────────────────────────────────────────────────┘
```

## 扩展阅读

- [Recommender Systems Handbook](https://www.springer.com/gp/book/9780387858198)
- [Netflix 推荐系统架构](https://about.netflix.com/en/news/how-netflix-recommendations-work)
- [LightFM 库](https://making.lyst.com/lightfm/docs/home.html)
- [Surprise 推荐算法库](https://surpriselib.com/)

