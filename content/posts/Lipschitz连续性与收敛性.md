---
date: 2024-12-13T14:12:38+08:00
tags:
  - 数学
  - 机器学习
title: Lipschitz连续性与收敛性
share: true
canonicalURL: 
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

这段内容涉及**L平滑性**在**稀疏子空间**中的推广和应用。经典的L平滑性定义用于描述函数在不同点之间的梯度变化程度。当函数在某个子空间（如稀疏子空间）中定义时，L平滑常数往往会减少，反映出函数在较小的区域内变化的“平滑性”增大。

## Lipschitz连续性与梯度光滑性

### Lipschitz连续性
**定义** 设函数 $f(x): \mathbb{R}^d \to \mathbb{R}$ 是定义在 $\mathbb{R}^d$ 上的连续函数，如果存在常数 $L > 0$，使得对于任意 $x, y \in \mathbb{R}^d$，有
$$
\|f(x) - f(y)\| \leq L \|x - y\|
$$
则称 $f(x)$ 是Lipschitz连续的。而 $L$ 称为函数 $f(x)$ 的Lipschitz常数。
满足这个条件的函数有这样的性质
**性质**
- 一致连续性: Lipschitz连续函数必定是一致连续的
- 有限变化率: Lipschitz连续函数必定有限的变化率
- 闭包性: Lipschitz连续函数在函数加法和数乘下是闭合的

### 梯度光滑性
**定义** 如果$f(x)$是可微函数,且其梯度函数$\nabla f(x)$是Lipschitz连续的，即存在常数$L$，对于任意的$x,y \in \mathbb{R}^d$，有
$$
\|\nabla f(x) - \nabla f(y)\| \leq L \|x - y\|
$$
则称$f(x)$是梯度光滑的。

满足这个条件的函数有以下性质:
- 二次可微性: Lipschitz光滑函数几乎处处二次可微，且hessian矩阵的最大特征值不超过$L$
- Taylor展开: Lipschitz光滑函数的Taylor展开的形式为
$$
f(y) \leq f(x) + \langle \nabla f(x), y - x\rangle + \frac{L}{2} \|y - x\|^2
$$

