---
date: 2024-08-11T16:12:08Z
tags:
  - LLM
  - 推理加速
title: FlashAttention Fast and Memory-Efficient Exact Attention with IO-Awareness
share: true
canonicalURL: ""
keywords:
  - 关键字1
  - 关键字2
description: 
series: 系列
lastmod: 
lang: cn
cover:
  image: 
author: qqq
dir: posts
---


## 2 背景  
我们提供了一些关于现代硬件（GPU）上常见深度学习操作性能特性的背景信息，并描述了注意力机制的标准实现。

### 2.1 硬件性能  
我们主要关注GPU。其他硬件加速器的性能类似。

**GPU内存层次结构**  
GPU的内存层次结构（图1左侧）包含多种不同大小和速度的内存，其中较小的内存速度更快。例如，A100 GPU具有40-80GB的高带宽内存（HBM），其带宽为1.5-2.0TB/s，并且每个108个流处理器（streaming multiprocessors）中的每个都具有192KB的片上SRAM，带宽估计为19TB/s左右。片上SRAM的速度比HBM快一个数量级，但其大小则小了许多数量级。随着计算速度相对于内存速度的提升，操作越来越受到内存（HBM）访问的瓶颈影响。因此，利用快速的SRAM变得更加重要。

**执行模型**  
GPU拥有大量线程来执行一个操作（称为kernel）。每个kernel从HBM加载输入到寄存器和SRAM中，进行计算，然后将输出写入HBM。

**性能特性**  
根据计算和内存访问的平衡，操作可以分为计算受限或内存受限。这通常通过算术强度来衡量，算术强度是每字节内存访问的算术操作数量。

1. **计算受限**：操作的时间取决于算术操作的数量，而访问HBM的时间则要少得多。典型的例子包括具有大内维度的矩阵乘法和具有大量通道的卷积。
  
2. **内存受限**：操作的时间取决于内存访问的数量，而计算所花费的时间则要少得多。例子包括大多数其他操作：元素级操作（例如，激活、dropout），以及归约操作（例如，求和、softmax、批量归一化、层归一化）。

**内核融合**  
加速内存受限操作的最常见方法是内核融合：如果有多个操作应用于相同的输入，则输入可以从HBM中加载一次，而不是为每个操作多次加载。编译器可以自动融合许多元素级操作。然而，在模型训练的背景下，中间值仍需要写入HBM以供反向传播使用，从而降低了简单内核融合的效果。

### 2.2 标准注意力机制实现

给定输入序列 $Q, K, V ∈ R^{N\times d}$，其中 𝑁 是序列长度，𝑑 是头部维度，我们需要计算注意力输出$O ∈ R^{N\times d}$:
$$
S =QK^T\in R^{N\times N},\quad P=softmax(S)\in R^{N\times N},\quad O=PV\in R^{N\times N}
$$
其中 softmax 是按行应用【applied row-wise】的。

标准的注意力机制实现将矩阵 S 和 P 物化【materialize】到高带宽内存（HBM），这需要 𝑂(𝑁²) 的内存。通常 𝑁 远大于 𝑑（例如，对于 GPT-2，𝑁 = 1024 而 𝑑 = 64）。我们在算法0中描述了标准注意力机制的实现。由于部分或大多数操作都是内存受限的（例如 softmax），大量的内存访问会导致较慢的实际运行时间。

这个问题因应用于注意力矩阵的其他元素级操作而加剧，例如应用于 S 的掩码操作或应用于 P 的 dropout。因此，已有许多尝试将几个元素级操作融合在一起，例如将掩码与 softmax 融合。

在第3.2节中，我们将展示标准注意力机制实现中高带宽内存的访问次数是序列长度 𝑁 的平方。同时，我们还将比较标准注意力机制与我们的方法（FlashAttention）的FLOPs数量和高带宽内存访问次数。

**算法0：标准注意力机制实现**

**前提条件**：HBM中的矩阵  $Q, K, V ∈ R^{N\times d}$。

1. 从HBM按块加载 Q 和 K，计算 $S = QKᵀ$，并将 S 写入 HBM。
2. 从HBM读取 S，计算$P = softmax(S)$，并将 P 写入 HBM。
3. 从HBM按块加载 P 和 V，计算 $O = PV$，并将 O 写入 HBM。
4. 返回 O。


### 3 FlashAttention: 算法、分析与扩展

我们展示了如何在减少HBM读取/写入次数的情况下计算精确的注意力，并且无需为反向传播存储大型中间矩阵。这产生了一种既节省内存又在实际运行时间上更快的注意力算法。我们分析了其IO复杂性，表明我们的方法相比于标准注意力机制需要更少的HBM访问次数。我们进一步展示了FlashAttention作为一个有用的基本模块可以扩展为处理块稀疏注意力。

为了便于讲解，我们主要关注前向传播部分；附录B包含了反向传播的详细信息。

#### 3.1 一种使用分块和重计算的高效注意力算法

给定HBM中的输入 $Q, K, V \in \mathbb{R}^{N \times d}$，我们的目标是计算注意力输出 $O \in \mathbb{R}^{N \times d}$ 并将其写入HBM。我们的目标是减少HBM访问次数（到 $N$ 的亚二次方）。

我们应用了两种已确立的技术（分块、重计算）来克服在 $N$ 的亚二次方HBM访问次数下计算精确注意力的技术挑战。我们在算法1中描述了这一过程。主要思路是将输入 $Q, K, V$ 分块，从慢速HBM加载到快速SRAM，然后根据这些块计算注意力输出。通过在累加之前按正确的归一化因子缩放每个块的输出，我们最终得到正确的结果。

**分块**  
我们按块计算注意力。Softmax将 $K$ 的列耦合在一起，因此我们通过缩放来分解大的softmax。为了数值稳定性，向量 $x \in \mathbb{R}^B$ 的softmax计算如下：
$$
m(x) := \max_i x_i, \quad f(x) := \left[ \exp(x_1 - m(x)), \dots, \exp(x_B - m(x)) \right], \quad \ell(x) := \sum_i f(x)_i, \quad \text{softmax}(x) := \frac{f(x)}{\ell(x)}
$$
对于向量 $x^{(1)}, x^{(2)} \in \mathbb{R}^B$，我们可以分解连接后的 $x = \left[ x^{(1)}, x^{(2)} \right] \in \mathbb{R}^{2B}$ 的softmax计算为：
$$
m(x) = m\left(\left[ x^{(1)}, x^{(2)} \right]\right) = \max\left(m(x^{(1)}), m(x^{(2)})\right), \quad f(x) = \left[ \exp(m(x^{(1)}) - m(x)) f(x^{(1)}), \exp(m(x^{(2)}) - m(x)) f(x^{(2)}) \right],
$$
$$
\ell(x) = \ell\left(\left[ x^{(1)}, x^{(2)} \right]\right) = \exp(m(x^{(1)}) - m(x)) \ell(x^{(1)}) + \exp(m(x^{(2)}) - m(x)) \ell(x^{(2)}), \quad \text{softmax}(x) = \frac{f(x)}{\ell(x)}
$$
因此，如果我们跟踪一些额外的统计数据（$m(x), \ell(x)$），我们可以一次计算一个块的softmax值。我们将输入 $Q, K, V$ 分块（算法1第3行），计算softmax值及额外的统计数据（算法1第10行），并合并结果（算法1第12行）。

**重计算**  
我们的目标之一是避免存储 $O(N^2)$ 个用于反向传播的中间值。反向传播通常需要矩阵 $S, P \in \mathbb{R}^{N \times N}$ 来计算相对于 $Q, K, V$ 的梯度。然而，通过存储输出 $O$ 和softmax归一化统计数据（$m, \ell$），我们可以在反向传播过程中从SRAM中的 $Q, K, V$ 块轻松重计算注意力矩阵 $S$ 和 $P$。这可以视为一种选择性梯度检查点策略。虽然梯度检查点策略已被建议用于减少所需的最大内存量，但所有已知的实现都不得不在速度和内存之间进行权衡。相比之下，即使有更多的FLOPs，我们的重计算由于减少了HBM访问次数而加快了反向传播过程。完整的反向传播描述见附录B。

**实现细节：内核融合**  
分块使我们能够在一个CUDA内核中实现我们的算法，从HBM加载输入，执行所有计算步骤（矩阵乘法、softmax、可选的掩码和dropout、矩阵乘法），然后将结果写回HBM（掩码和dropout在附录B中）。这避免了反复从HBM读取和写入输入和输出。

**算法1：FlashAttention**

**前提条件**：HBM中的矩阵 $Q, K, V \in \mathbb{R}^{N \times d}$，片上SRAM大小为 $M$。

1. 设置块大小 $B_c = \left\lceil \frac{M}{4d} \right\rceil$,  $B_r = min ( \lceil \frac{M}{4d} \rceil ,d)$.
2. 初始化 $O = (0)_{N \times d} \in \mathbb{R}^{N \times d}$, $\ell = (0)_N \in \mathbb{R}^N$, $m = (-\infty)_N \in \mathbb{R}^N$ 在HBM中。
3. 将 $Q$ 分成 $T_r = \left\lceil \frac{N}{B_r} \right\rceil$ 个块 $Q_1, \dots, Q_{T_r}$ ，每个大小为 $B_r \times d$；将 $K, V$ 分成 $T_c = \left\lceil \frac{N}{B_c} \right\rceil$ 个块 $K_1, \dots, K_{T_c}$ 和 $V_1, \dots, V_{T_c}$ ，每个大小为 $B_c \times d$。
4. 将 $O$ 分成 $T_r$ 个块 $O_i, \dots, O_{T_r}$ ，每个大小为 $B_r \times d$；将 $\ell$ 分成 $T_r$ 个块 $\ell_i, \dots, \ell_{T_r}$ ，每个大小为 $B_r$；将 $m$ 分成 $T_r$ 个块 $m_1, \dots, m_{T_r}$ ，每个大小为 $B_r$。
5. 对于 $1 \leq j \leq T_c$：
6. 从HBM加载 $K_j, V_j$ 到片上SRAM。
7. 对于 $1 \leq i \leq T_r$：
8. 从HBM加载 $Q_i, O_i, \ell_i, m_i$ 到片上SRAM。
9. 在片上计算 $S_{ij} = Q_i K_j^\top \in \mathbb{R}^{B_r \times B_c}$。
10. 在片上计算 $\tilde{m}_{ij} = \text{rowmax}(S_{ij}) \in \mathbb{R}^{B_r}$，$\tilde{P}_{ij} = \exp(S_{ij} - \tilde{m}_{ij}) \in \mathbb{R}^{B_r \times B_c}$（逐点计算），$\tilde{\ell}_{ij} = \text{rowsum}(\tilde{P}_{ij}) \in \mathbb{R}^{B_r}$。
11. 在片上计算 $m_{i}^{\text{new}} = \max(m_i, \tilde{m}_{ij}) \in \mathbb{R}^{B_r}$，$\ell_{i}^{\text{new}} = \exp(m_i - m_{i}^{\text{new}})\ell_i + \exp(\tilde{m}_{ij} - m_{i}^{\text{new}})\tilde{\ell}_{ij} \in \mathbb{R}^{B_r}$。
12. 写入$$O_i \leftarrow \text{diag}(\ell_{i}^{\text{new}})^{-1}\left(\text{diag}(\ell_i)\exp(m_i-m
_{i}^{\text{new}})O_i + \exp(\tilde{m}_{ij} - m_{i}^{\text{new}})\tilde{P}_{ij}V_j\right)$$ 到HBM。
13. 写入 $\ell_i \leftarrow \ell_{i}^{\text{new}}$，$m_i \leftarrow m_{i}^{\text{new}}$ 到HBM。
14. 结束循环
15. 结束循环
16. 返回 $O$。

我们证明了FlashAttention的正确性、运行时间和内存需求（证明在附录C中）。

**定理1**：算法1返回 $O = \text{softmax}(QK^\top)V$ ，具有 $\mathcal{O}(N^2d)$ 的FLOPs，并且除了输入和输出之外仅需要 $\mathcal{O}(N)$ 的额外内存。