---
date: 2024-12-13T14:12:57+08:00
tags:
  - 数学
  - 机器学习
  - 矩阵论
title: Sherman-Morrison 公式
share: true
canonicalURL: ""
keywords: 
description: 
series: 系列
lastmod: 
lang: cn
cover:
  image: 
author: heqi
dir: posts
math: "true"
---

Sherman-Morrison 公式是线性代数中的一个重要结果，用于计算一个可逆矩阵经过秩为 1 的更新后的逆矩阵。具体来说，当我们有一个可逆的 $n \times n$矩阵$A$ 和两个列向量$u$、$v$ 时，Sherman-Morrison 公式给出了矩阵$A + u v^\top$ 的逆的显式表达式。

公式如下：
  
$$
(A + u v^\top)^{-1} = A^{-1} - \frac{A^{-1} u v^\top A^{-1}}{1 + v^\top A^{-1} u}
$$
**解释：**
  
- $A$是一个可逆的$n \times n$矩阵。
- $u$和$v$是$n \times 1$的列向量。
- $u v^\top$是一个$n \times n$的矩阵，且秩为 1，因为它是两个向量的外积。
- $A + u v^\top$表示对矩阵$A$进行了一次秩为 1 的更新。
- $A^{-1}$是矩阵$A$的逆矩阵。
- 分母中的$1 + v^\top A^{-1} u$是一个标量。 
**应用：**
Sherman-Morrison 公式在数值计算中非常有用，因为它允许我们在已知$A^{-1}$的情况下，高效地计算$A + u v^\top$的逆，而不需要对整个矩阵重新求逆。这对于处理大型矩阵或需要重复更新的情况下，能显著降低计算成本。
**在您的问题中的应用：**
在您的问题中，我们需要计算矩阵$(I_m + \mathbf{1}_m \mathbf{1}_m^\top)^{-1}$，其中
- $I_m$是$m \times m$的单位矩阵。
- $\mathbf{1}_m$是长度为$m$的全 1 列向量。
这是一个特定情况下的 Sherman-Morrison 公式应用：
- 取$A = I_m$，因此$A^{-1} = I_m$。
- 取$u = \mathbf{1}_m$，$v = \mathbf{1}_m$。
代入 Sherman-Morrison 公式：

$$
\begin{aligned} \left( I_m + \mathbf{1}_m \mathbf{1}_m^\top \right)^{-1} &= I_m^{-1} - \frac{I_m^{-1} \mathbf{1}_m \mathbf{1}_m^\top I_m^{-1}}{1 + \mathbf{1}_m^\top I_m^{-1} \mathbf{1}_m} \\ &= I_m - \frac{\mathbf{1}_m \mathbf{1}_m^\top}{1 + \mathbf{1}_m^\top \mathbf{1}_m} \end{aligned} 
$$
因为：$\mathbf{1}_m^\top \mathbf{1}_m = m$，所以分母为$1 + m$。
- 最终得到：
$$  
(I_m + \mathbf{1}_m \mathbf{1}_m^\top)^{-1} = I_m - \frac{\mathbf{1}_m \mathbf{1}_m^\top}{m + 1}
$$
**总结：**


Sherman-Morrison 公式提供了一种高效的方法来计算秩为 1 更新矩阵的逆。

