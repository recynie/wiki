---
title: 矩阵树定理
date: 2026-04-06
description: 矩阵树定理（Matrix-Tree Theorem）——利用行列式计算图的生成树数量
tags:
  - graph
  - matrix
  - spanning-tree
draft: false
permalink:
---

## 定义

矩阵树定理（Matrix-Tree Theorem）是图论中的重要定理，它建立了图的生成树数量与矩阵行列式之间的联系。

该定理由 Kirchhoff 于 1847 年提出，因此又称 **Kirchhoff 定理**。[^1]

---

## 核心概念

### Laplacian 矩阵（拉普拉斯矩阵）

给定无向图 $G = (V, E)$，$n = |V|$ 个节点，其 Laplacian 矩阵 $L$ 定义为：

$$
L_{ij} = \begin{cases}
\deg(i) & i = j \\
-1 & (i, j) \in E \\
0 & \text{otherwise}
\end{cases}
$$

其中 $\deg(i)$ 是节点 $i$ 的度。

### Laplacian 矩阵的性质

1. **对称矩阵**：$L = L^T$
2. **行和为零**：每一行的和都是 $0$
3. **半正定矩阵**：所有特征值 $\lambda_1 \leq \lambda_2 \leq \dots \leq \lambda_n$ 满足 $\lambda_1 = 0$
4. **零特征值的个数** = 连通块的个数

### 关联矩阵

对于无向图，设 $B$ 为 $n \times m$ 的 **有向关联矩阵**：

$$
B_{ij} = \begin{cases}
1 & \text{节点 } i \text{ 是边 } j \text{ 的起点} \\
-1 & \text{节点 } i \text{ 是边 } j \text{ 的终点} \\
0 & \text{otherwise}
\end{cases}
$$

则 $L = B \cdot B^T$。

---

## 定理陈述

**矩阵树定理**：设 $G$ 是一个连通无向图，$L$ 为其 Laplacian 矩阵。删除任意一行一列得到的 **余子式** $L^*$ 的行列式等于 $G$ 的生成树数量。

$$
\text{num\_spanning\_trees}(G) = \det(L^*)
$$

其中 $L^*$ 是 $L$ 删除第 $k$ 行和第 $k$ 列后得到的 $n-1$ 阶主子式。

---

## 算法流程

### 步骤 1：构建 Laplacian 矩阵

```cpp
vector<vector<double>> buildLaplacian(int n, vector<pair<int,int>>& edges) {
    vector<vector<double>> L(n, vector<double>(n, 0));
    vector<int> degree(n, 0);
    
    for (auto [u, v] : edges) {
        L[u][u] += 1;
        L[v][v] += 1;
        L[u][v] -= 1;
        L[v][u] -= 1;
        degree[u]++;
        degree[v]++;
    }
    
    return L;
}
```

### 步骤 2：删除一行一列

```cpp
vector<vector<double>> getMinor(vector<vector<double>>& L, int row, int col) {
    int n = L.size();
    vector<vector<double>> minor(n-1, vector<double>(n-1, 0));
    
    for (int i = 0, ni = 0; i < n; i++) {
        if (i == row) continue;
        for (int j = 0, nj = 0; j < n; j++) {
            if (j == col) continue;
            minor[ni][nj++] = L[i][j];
        }
        ni++;
    }
    return minor;
}
```

### 步骤 3：计算行列式（高斯消元）

```cpp
double determinant(vector<vector<double>>& mat) {
    int n = mat.size();
    double det = 1;
    
    for (int i = 0; i < n; i++) {
        // 寻找主元
        int pivot = i;
        for (int j = i + 1; j < n; j++) {
            if (fabs(mat[j][i]) > fabs(mat[pivot][i]))
                pivot = j;
        }
        
        if (fabs(mat[pivot][i]) < 1e-9)  // 主元为0
            return 0;
        
        if (pivot != i) {
            swap(mat[pivot], mat[i]);
            det *= -1;  // 交换行改变符号
        }
        
        det *= mat[i][i];
        
        // 消元
        for (int j = i + 1; j < n; j++) {
            double factor = mat[j][i] / mat[i][i];
            for (int k = i; k < n; k++) {
                mat[j][k] -= factor * mat[i][k];
            }
        }
    }
    
    return det;
}
```

### 步骤 4：整合

```cpp
long long countSpanningTrees(int n, vector<pair<int,int>>& edges) {
    auto L = buildLaplacian(n, edges);
    auto minor = getMinor(L, 0, 0);  // 删除第0行0列
    double det = determinant(minor);
    return (long long)(det + 0.5);  // 四舍五入避免浮点误差
}
```

---

## 整数版本（基于 Kirchhoff）

对于整数 Laplacian 矩阵，行列式一定是整数。可以使用 **Bareiss 算法**（一种稳定的整数行列式算法）避免浮点误差。

```cpp
long long kirchhoffDeterminant(vector<vector<long long>>& mat) {
    int n = mat.size();
    vector<vector<long long>> a = mat;
    long long det = 1;
    
    for (int i = 0; i < n; i++) {
        for (int j = i + 1; j < n; j++) {
            for (int k = i + 1; k < n; k++) {
                a[j][k] = a[j][k] * a[i][i] - a[j][i] * a[i][k];
                if (i > 0) a[j][k] /= a[i-1][i-1];
            }
        }
    }
    
    return a[n-1][n-1];
}
```

---

## 例题

### 例题：不同生成树的数量

#### 问题

给定一个 $n$ 个节点的完全图 $K_n$，求其生成树的数量。

#### 分析

完全图 $K_n$ 的 Laplacian 矩阵：

$$
L = \begin{pmatrix}
n-1 & -1 & -1 & \dots & -1 \\
-1 & n-1 & -1 & \dots & -1 \\
\vdots & \vdots & \vdots & \ddots & \vdots \\
-1 & -1 & -1 & \dots & n-1
\end{pmatrix}
$$

删除第一行第一列后得到 $n-1$ 阶矩阵：

$$
L^* = \begin{pmatrix}
n-1 & -1 & -1 & \dots & -1 \\
-1 & n-1 & -1 & \dots & -1 \\
\vdots & \vdots & \vdots & \ddots & \vdots \\
-1 & -1 & -1 & \dots & n-1
\end{pmatrix}_{(n-1) \times (n-1)}
$$

其行列式为 $n^{n-2}$（Cayley 公式）。

#### 答案

$$
\text{num\_spanning\_trees}(K_n) = n^{n-2}
$$

这就是著名的 **Cayley 公式**。

---

## 复杂度分析

| 步骤 | 时间复杂度 |
|------|-----------|
| 构建 Laplacian | $O(n + m)$ |
| 高斯消元求行列式 | $O(n^3)$ |

总时间复杂度为 $O(n^3)$，空间复杂度为 $O(n^2)$。

---

## 应用

1. **网络可靠性分析**：计算网络的生成树数量
2. **电路分析**：Kirchhoff 电流定律与矩阵树定理
3. **化学图论**：共轭分子中不同生成树的计数
4. **组合优化**：旅行商问题的松弛

---

## 参考资料

[^1]: Kirchhoff, G. (1847). Ueber die Auflösung der Gleichungen, auf welche man bei der Untersuchung der linearen Vertheilung galvanischer Ströme geführt wird. Annalen der Physik und Chemie, 72(12), 497-508.

[^2]: Matrix-Tree Theorem - Wikipedia. https://en.wikipedia.org/wiki/Matrix_tree_theorem

[^3]: 矩阵树定理 - OI Wiki. https://oi-wiki.org/graph/matrix-tree/
