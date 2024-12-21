---
date: 2024-12-20T17:12:56+08:00
tags:
  - 大模型
title: kv-cache讨论
share: true
canonicalURL: ""
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
### 前提条件

**输入与隐藏状态**
- 输入序列  $x$  的形状为  $({batch}, {len})$ 。
- 第  $i$  层的输入激活值为  ${hiddenstate}_{i-1}$ ，形状为  $({batch}, {len}, {hiddensize})$ 。
- 第  $i$  层的输出为  ${hiddenstate}_i$ ，形状同样为  $({batch}, {len}, {hiddensize})$ 。

**单个 Token 的处理**
- 对于序列中的第  $j$  个 token   $x_j$  ，其输入为   ${hiddenstate}_{i-1,j}$  ，输出为  ${hiddenstate}_{i,j}$ ，形状为  $({batch}, {hiddensize})$ 。
- 计算关系为：

$$
{hiddenstate}_{i,j} = f({hiddenstate}_{i-1,j})
$$

- 函数  $f$  的定义为：

$$
f(x) = (x + {atten}(x)) + {mlp}(x + {atten}(x))
$$

- 其中， ${mlp}$  和加法操作在序列长度（ ${len}$ ）这一维度上是独立的，即对于  $z = x \ {op} \ y$ ，有  $z_{k,:}$  仅依赖于  $x_{k,:}$  和  $y_{k,:}$ ，与  $m \neq k$  时的  $x_m$  和  $y_m$  无关。  

**注意力机制的区别**
- **Causal Attention（自回归注意力）的省略版本**：

$$
({Q} \times {K})_{k,:} = {atten}(x_k, x_{:k})
$$

即，第  $k$  个位置的查询仅与当前及之前的位置相关。
- **Bi-directional Attention（双向注意力）**：

$$
({Q} \times {K})_{k,:}^t = {atten}(x_k^t, x_{:}^t)
$$

其中， $t$  表示时间步，第  $k$  个token的查询与当前时间步的所有token相关。
### 推论
#### 1. Causal Attention 的推理阶段
- **依赖关系**：
- 对于  ${hiddenstate}_{i,j}$ ，其依赖于  ${hiddenstate}_{i-1,j}$  和  ${hiddenstate}_{i-1,j-1}, \ldots, {hiddenstate}_{i-1,0}$ 。且不同时间步下 $hiddenstate_{i,j}^t=hiddenstate_{i,j}^{t+1}$ 
- 最终预测结果  $y_j$  是  ${hiddenstate}_{last,j}$  的函数。
- **KV-Cache 的应用**：
- **预计算**：由于  $K_{i,j}$  仅依赖于  $x_{:j}$ 即 $K_{i,j}=func(hiddenstate_{i-1,:j})$ ，且与时间步  $t$  无关，即：

$$
K_{i,j} = K_{i,j}^t, \quad V_{i,j} = V_{i,j}^t
$$

因此，可以在推理前预先计算并缓存  $K$  和  $V$ 。
- **计算需求**：在推理时，仅需使用  $x_j$ （形状为  $({batch}, 1, {hiddensize})$ ）、 $K_{:,j}$  和  $V_{:,j}$ （形状为  $({batch}, {len}, {hiddensize}) \times {layernum}$ ）进行计算。
- **结论**：由于  $K$  和  $V$  不随时间步变化，可以缓存计算，**Causal Attention** 可以有效利用 KV-Cache 进行加速推理。
#### 2. Bi-directional Attention 的推理阶段
- **依赖关系**：
- 最终预测结果  $y_j$  依赖于  ${hiddenstate}_{last,j}$ 。
-  ${hiddenstate}_{i,j}^t$  依赖于  $K_{:}^t$ ，即当前时间步的所有键值。
- **KV-Cache 的限制**：
- 在 **Bi-directional Attention** 中， $K_{:}^t$  依赖于当前时间步  $t$  的所有输入，因此不同时间步之间的  $K$  和  $V$  是相互关联的。
- 这种依赖关系导致无法像在 Causal Attention 中那样独立缓存  $K$  和  $V$ ，因为每个时间步的计算都可能影响其他时间步的  $K$  和  $V$ 。 
