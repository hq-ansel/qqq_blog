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
author: qqq
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



### 某些推论

对于L(x) 是Lipschitz连续的函数，且二阶导数存在。

存在一个Lipschitz常数$L$，当更新步长为$\eta$时，有
$$
f(\theta_{t+1}) = f(\theta_t) - \eta \nabla f(\theta_t) \leq f(\theta_t) - \eta L \| \theta_t - \theta_{t-1} \|
$$

且 如果函数$\nabla f(x)$是Lipschitz连续的且$\|\nabla f(y)-\nabla f(x)\| \leq L \|y-x\|$，则
$$
\|f(y) -f(x) - \langle \nabla f(x), y - x\rangle \| \leq \frac{L}{2} \|y - x\|^2
$$
等价于
$$
\begin{aligned}
f(y) & \leq f(x) + \langle \nabla f(x), y - x\rangle + \frac{L}{2} \|y - x\|^2 \\
f(y) & \geq f(x)  + \langle \nabla f(x), y - x\rangle - \frac{L}{2} \|y - x\|^2
\end{aligned}
$$

证明

$$
\begin{aligned}
f(y) & = f(x) +\int_{0}^{1} \langle \nabla f(x+\tau(y -x)) , y - x\rangle d\tau \\
 &= f(x) + \langle \nabla f(x), y - x\rangle + \int_{0}^{1} \langle \nabla f(x+\tau(y-x))- \nabla f(x), y-x\rangle d\tau 
\end{aligned}
$$
于是
$$
\begin{aligned}
\|f(y) - f(x) - \langle \nabla f(x), y - x\rangle |\ & = \|\int_{0}^{1} \langle \nabla f(x+\tau(y-x))- \nabla f(x), y-x\rangle d\tau \| \\
 & \leq \int_{0}^{1} \|\nabla f(x+\tau(y-x))- \nabla f(x)\| d\tau \\
 & \leq \int_{0}^{1} \|\nabla f(x+\tau(y-x))- \nabla f(x)\| \|y-x\| d\tau \\
 & \leq \int_{0}^{1} \tau L \|y-x\|^2 d\tau \\
 & = \frac{L}{2} \|y-x\|^2
\end{aligned}
$$



于是当更新模式为

$$
\begin{aligned}
\theta_{t+1} &= \theta_t - \eta E_D(\nabla f(\theta_t)) \\
\theta_{t+1} - \theta_t &= -\eta E_D(\nabla f(\theta_t)) \\
\end{aligned}
$$

有

$$
\begin{aligned}
f(\theta_{t+1})-f(\theta_t) & \leq \langle \nabla f(\theta_t), \theta_{t+1} - \theta_t\rangle + \frac{L}{2} \|\theta_{t+1} - \theta_t\|^2 \\
&= \langle -\nabla f(\theta_t), -\eta E_D(\nabla f(\theta_t))\rangle + \frac{L}{2} \|-\eta E_D(\nabla f(\theta_t))\|^2 \\
&= -\eta \langle \nabla f(\theta_t), -E_D(\nabla f(\theta_t))\rangle + \frac{L\eta^2}{2} \|E_D(\nabla f(\theta_t))\|^2
\end{aligned}
$$

