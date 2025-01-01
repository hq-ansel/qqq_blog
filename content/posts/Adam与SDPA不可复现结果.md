---
date: 2024-12-30T22:12:47+08:00
tags:
  - 推理加速
  - cuda
  - pytorch
  - LLM
title: Adam与SDPA不可复现结果
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


## 问题描述
事情的起因是我发现qwen微调过程中会出现一个很奇怪的现象，固定seed的情况下跑两次实验`layer.mlp.gate_proj.weight`的权重竟然会不一样，即使是在使用了下面几乎固定了所有seed的情况下
```python
	random.seed(args.seed)
    np.random.seed(args.seed)
    torch.manual_seed(args.seed)
    torch.cuda.manual_seed(args.seed)
    torch.cuda.manual_seed_all(args.seed)
    os.environ['PYTHONHASHSEED'] = str(args.seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False
```

![62e9152203747816227f39dfab24c23.png](/images/62e9152203747816227f39dfab24c23.png)
这件事情和师兄们讨论的了大概3个小时，并且我自己尝试了方案
1. 难道是优化器有问题，将adamW优化器换为adam，而后又换为SGD，发现使用adam类优化器始终无法固定结果，但是用sgd优化的情况怎么重复实验结果都是一致的
2. 师兄认为是由于浮点数的表达与硬件实现问题，但是我认为在固定了代码不变的情况下同样位置应该能够得到同样的结果，即使是存在表达精度限制的情况下。这个原因可以很简单的用一个mlp在adam优化器上测试得到
3. 师兄还提出有可能是计算顺序导致的，因为有限精度表达下确实cuda很多运算实际不符合交换律的。但是使用了`CUDA_LAUNCH_BLOCKING=1`的情况下仍然无法得到可重复解
4. 师兄还提出是我的代码写的有问题，但是在几乎注释了我所有改动，基本上等同于微调原始的qwen结构还是会有不可复现问题
## 最后的解决方案
最后这个方案几乎是随机拍脑袋想出来的，我使用的软件库版本如下
```
torch                    2.4.1
transformers             4.45.2
```
在这个版本中transformer库对于qwen的attn有三种实现,我以为的实现方式采用的是最基本的数学矩阵乘法方式，但是实际上这个版本使用的是默认的Sdpa Attention，很有意思的是，这个torch的实现(`torch.nn.functional.scaled_dot_product_attention`)采用了算子融合的方式，我并不知道这个算子是否会存在某些随机性的问题。
```
Qwen2Attention           (我以为的)
Qwen2FlashAttention2
Qwen2SdpaAttention       (default)
```

于是我采用了下面这个方式初始化我的模型，通过这样的方式设置模型，原来的不可复现的问题随之消失了。

```
model = AutoModelForCausalLM.from_pretrained(args.model,
                                        # attn_implementation="eager",
                                        config=config,
                                        device_map='cpu',
                                        torch_dtype=torch.float16 if amp_enabled else torch.float32)
```
那么真的是sdpa的问题吗?

我使用了下面这个案例进行比较
```python
import random
import numpy as np
from typing import Optional, Tuple

import torch
import torch.nn as nn
import torch.optim as optim
import torch.nn.functional as F

def init_seed(seed=0):
    random.seed(seed)
    np.random.seed(seed)

    torch.manual_seed(seed)
    torch.cuda.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False

init_seed(0)

def _compute_default_rope_parameters(
    rope_theta: float,
    device: Optional["torch.device"] = None,
    seq_len: Optional[int] = None,
    **rope_kwargs,
) -> Tuple["torch.Tensor", float]:
  
    base = rope_theta
    partial_rotary_factor = 1.0
    head_dim = 896//14
    dim = int(head_dim * partial_rotary_factor)

    attention_factor = 1.0  # Unused in this type of RoPE

    # Compute the inverse frequencies
    inv_freq = 1.0 / (base ** (torch.arange(0, dim, 2, dtype=torch.int64).float().to(device) / dim))
    return inv_freq, attention_factor


class RotaryEmbedding(nn.Module):
    def __init__(
        self,
        dim=None,
        max_position_embeddings=2048,
        base=10000,
        device=None,
        scaling_factor=1.0,
        rope_type="default",
    ):
        super().__init__()
        # TODO (joao): remove the `if` below, only used for BC
        self.rope_kwargs = {}
        self.rope_kwargs = {
            "rope_type": rope_type,
            "factor": scaling_factor,
            "dim": dim,
            "base": base,
            "max_position_embeddings": max_position_embeddings,
        }
        self.rope_type = rope_type
        self.max_seq_len_cached = max_position_embeddings
        self.original_max_seq_len = max_position_embeddings
        self.rope_init_fn = _compute_default_rope_parameters
        inv_freq, self.attention_scaling = self.rope_init_fn( rope_theta=1000000.0,device=device, **self.rope_kwargs)
        self.register_buffer("inv_freq", inv_freq, persistent=False)
        self.original_inv_freq = self.inv_freq

    @torch.no_grad()
    def forward(self, x, position_ids):
        inv_freq_expanded = self.inv_freq[None, :, None].float().expand(position_ids.shape[0], -1, 1)
        position_ids_expanded = position_ids[:, None, :].float()
        # Force float32 (see https://github.com/huggingface/transformers/pull/29285)
        device_type = x.device.type
        device_type = device_type if isinstance(device_type, str) and device_type != "mps" else "cpu"
        with torch.autocast(device_type=device_type, enabled=False):
            freqs = (inv_freq_expanded.float() @ position_ids_expanded.float()).transpose(1, 2)
            emb = torch.cat((freqs, freqs), dim=-1)
            cos = emb.cos()
            sin = emb.sin()

        # Advanced RoPE types (e.g. yarn) apply a post-processing scaling factor, equivalent to scaling attention
        cos = cos * self.attention_scaling
        sin = sin * self.attention_scaling

        return cos.to(dtype=x.dtype), sin.to(dtype=x.dtype)

class MLP(nn.Module):
    def __init__(self, hidden_size=896, intermediate_size=4864, act_fn=nn.SiLU):
        super().__init__()
        self.hidden_size = hidden_size
        self.intermediate_size = intermediate_size
        self.gate_proj = nn.Linear(self.hidden_size, self.intermediate_size, bias=False)
        self.up_proj = nn.Linear(self.hidden_size, self.intermediate_size, bias=False)
        self.down_proj = nn.Linear(self.intermediate_size, self.hidden_size, bias=False)
        self.act_fn = nn.SiLU()

    def forward(self, hidden_state):
        return self.down_proj(self.act_fn(self.gate_proj(hidden_state)) * self.up_proj(hidden_state))

def repeat_kv(hidden_states: torch.Tensor, n_rep: int) -> torch.Tensor:
    batch, num_key_value_heads, slen, head_dim = hidden_states.shape
    if n_rep == 1:
        return hidden_states
    hidden_states = hidden_states[:, :, None, :, :].expand(batch, num_key_value_heads, n_rep, slen, head_dim)
    return hidden_states.reshape(batch, num_key_value_heads * n_rep, slen, head_dim)

def rotate_half(x):
    x1 = x[..., : x.shape[-1] // 2]
    x2 = x[..., x.shape[-1] // 2 :]
    return torch.cat((-x2, x1), dim=-1)

def apply_rotary_pos_emb(q, k, cos, sin, position_ids=None, unsqueeze_dim=1):
    cos = cos.unsqueeze(unsqueeze_dim)
    sin = sin.unsqueeze(unsqueeze_dim)
    q_embed = (q * cos) + (rotate_half(q) * sin)
    k_embed = (k * cos) + (rotate_half(k) * sin)
    return q_embed, k_embed

class Attention(nn.Module):
    def __init__(self, hidden_size=9, num_heads=8, dropout=0.0):
        super().__init__()
        self.hidden_size = 896
        self.num_heads = 14
        self.head_dim = self.hidden_size // self.num_heads
        self.num_key_value_heads = 2
        self.num_key_value_groups = self.num_heads // self.num_key_value_heads
        self.max_position_embeddings = 32768
        self.rope_theta = 1000000.0
        self.is_causal = True
        self.attention_dropout = 0.0

        if (self.head_dim * self.num_heads) != self.hidden_size:
            raise ValueError(
                f"hidden_size must be divisible by num_heads (got `hidden_size`: {self.hidden_size}"
                f" and `num_heads`: {self.num_heads})."
            )
        self.q_proj = nn.Linear(self.hidden_size, self.num_heads * self.head_dim, bias=True)
        self.k_proj = nn.Linear(self.hidden_size, self.num_key_value_heads * self.head_dim, bias=True)
        self.v_proj = nn.Linear(self.hidden_size, self.num_key_value_heads * self.head_dim, bias=True)
        self.o_proj = nn.Linear(self.num_heads * self.head_dim, self.hidden_size, bias=False)

        self.rotary_emb = RotaryEmbedding()

    # Adapted from Qwen2Attention.forward
    def forward(
        self,
        hidden_states: torch.Tensor,
        attention_mask: Optional[torch.Tensor] = None,
        position_ids: Optional[torch.LongTensor] = None,
        cache_position: Optional[torch.LongTensor] = None,
        position_embeddings: Optional[Tuple[torch.Tensor, torch.Tensor]] = None,  # will become mandatory in v4.46
    ) -> Tuple[torch.Tensor, Optional[torch.Tensor], Optional[Tuple[torch.Tensor]]]:


        bsz, q_len, _ = hidden_states.size()

        query_states = self.q_proj(hidden_states)
        key_states = self.k_proj(hidden_states)
        value_states = self.v_proj(hidden_states)

        query_states = query_states.view(bsz, q_len, self.num_heads, self.head_dim).transpose(1, 2)
        key_states = key_states.view(bsz, q_len, self.num_key_value_heads, self.head_dim).transpose(1, 2)
        value_states = value_states.view(bsz, q_len, self.num_key_value_heads, self.head_dim).transpose(1, 2)

        if position_embeddings is None:
            cos, sin = self.rotary_emb(value_states, position_ids)
        else:
            cos, sin = position_embeddings
        query_states, key_states = apply_rotary_pos_emb(query_states, key_states, cos, sin)

        key_states = repeat_kv(key_states, self.num_key_value_groups)
        value_states = repeat_kv(value_states, self.num_key_value_groups)

        if query_states.device.type == "cuda" and attention_mask is not None:
            query_states = query_states.contiguous()
            key_states = key_states.contiguous()
            value_states = value_states.contiguous()

        attn_output = torch.nn.functional.scaled_dot_product_attention(
            query_states,
            key_states,
            value_states,
            attn_mask=None,
            dropout_p=self.attention_dropout if self.training else 0.0,
            is_causal=True,
        )
        # attn_weight = torch.matmul(query_states, key_states.transpose(-2, -1))
        # attn_weight = attn_weight / (float(self.head_dim) ** 0.5)
        # attn_weight = F.softmax(attn_weight, dim=-1)
        # attn_output = torch.matmul(attn_weight, value_states)

        attn_output = attn_output.transpose(1, 2).contiguous()
        attn_output = attn_output.view(bsz, q_len, self.hidden_size)

        attn_output = self.o_proj(attn_output)

        return attn_output, None, 

class Qwen2RMSNorm(nn.Module):
    def __init__(self, hidden_size=896, eps=1e-6):
        """
        Qwen2RMSNorm is equivalent to T5LayerNorm
        """
        super().__init__()
        self.weight = nn.Parameter(torch.ones(hidden_size))
        self.variance_epsilon = eps

    def forward(self, hidden_states):
        input_dtype = hidden_states.dtype
        hidden_states = hidden_states.to(torch.float32)
        variance = hidden_states.pow(2).mean(-1, keepdim=True)
        hidden_states = hidden_states * torch.rsqrt(variance + self.variance_epsilon)
        return self.weight * hidden_states.to(input_dtype)

    def extra_repr(self):
        return f"{tuple(self.weight.shape)}, eps={self.variance_epsilon}"

class Qwen2DecoderLayer(nn.Module):
    def __init__(self, hidden_size = 896, ):
        super().__init__()
        self.hidden_size = hidden_size

        self.self_attn = Attention()
        self.mlp = MLP()
        self.input_layernorm = Qwen2RMSNorm(896, eps=1e-06)
        self.post_attention_layernorm = Qwen2RMSNorm(896, eps=1e-06)

    def forward(
        self,
        hidden_states: torch.Tensor,
        attention_mask: Optional[torch.Tensor] = None,
        position_ids: Optional[torch.LongTensor] = None,
        past_key_value: Optional[Tuple[torch.Tensor]] = None,
        output_attentions: Optional[bool] = False,
        use_cache: Optional[bool] = False,
        cache_position: Optional[torch.LongTensor] = None,
        position_embeddings: Optional[Tuple[torch.Tensor, torch.Tensor]] = None,  # will become mandatory in v4.46
        **kwargs,
    ) :

        residual = hidden_states

        hidden_states = self.input_layernorm(hidden_states)

        # Self Attention
        hidden_states, self_attn_weights = self.self_attn(
            hidden_states=hidden_states,
            attention_mask=attention_mask,
            position_ids=position_ids,
            position_embeddings=position_embeddings,
        )
        hidden_states = residual + hidden_states

        # Fully Connected
        residual = hidden_states
        hidden_states = self.post_attention_layernorm(hidden_states)
        hidden_states = self.mlp(hidden_states)
        hidden_states = residual + hidden_states

        outputs = (hidden_states,)

        if output_attentions:
            outputs += (self_attn_weights,)

        if use_cache:
            outputs += (present_key_value,)

        return outputs[0]

data = torch.randn(16, 2048, 896).cuda()*13

label = torch.randn(16, 2048, 896).cuda()*20

model = Qwen2DecoderLayer(896).cuda()

optimizer = optim.Adam(model.parameters(), lr=1e-3)

position_ids = torch.arange(data.shape[1], dtype=torch.long, device=data.device).unsqueeze(0).cuda()


for i in range(100):
    optimizer.zero_grad()
    output = model(data,position_ids=position_ids)
    loss = F.mse_loss(output, label)
    loss.backward()
    optimizer.step()
    print(loss.item())
    print(model.self_attn.q_proj.weight.grad.mean().item())
```

运行两次的结果如下
```
第一次跑 最后的结果
254.1112823486328
-3.770799850144613e-08
第二次跑最后的结果
254.1112823486328
-3.770876233488707e-08
```
很有意思的是，这里确实是存在误差!而我修改的结构对于grad更加敏感，且会把历史每一次的grad误差都累积起来

## 总结

写代码的时候可能还是要把每个代码做什么要了解清楚，有时候某些优化可能会带来未知的结果，以及要多和师兄们讨论，记录下排除后的解决方案，最后一定要大胆假设质疑所有情况，并给出实验验证。