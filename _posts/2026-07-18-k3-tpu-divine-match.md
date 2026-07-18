---
layout: post
title: "猜想：Kimi K3 是 TPU 的天赐良缘"
date: 2026-07-18 05:00:00 +0000
categories: [tpu, moe, architecture]
lang: zh
---

<style>
.callout-lede { background: #FEF7E0; border-left: 4px solid #F9AB00; padding: 14px 18px; margin: 16px 0 24px; border-radius: 4px; font-size: 14.5px; line-height: 1.55; color: #3C4043; }
.callout-takeaway { background: #E8F0FE; border-left: 4px solid #1A73E8; padding: 12px 16px; margin: 14px 0 22px; border-radius: 4px; font-size: 14px; color: #202124; }
.callout-warn { background: #FCE8E6; border-left: 4px solid #C5221F; padding: 12px 16px; margin: 14px 0 22px; border-radius: 4px; font-size: 14px; color: #3C4043; }
</style>

<figure style="margin: 0 0 24px; text-align: center;">
<img src="https://cc.higcp.com/assets/imagen/A-dramatic-wide-angle-digital-20260718-053349.png" alt="K3 与 TPU 的天赐良缘" style="width: 100%; max-width: 800px; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.12);" />
</figure>

<div class="callout-lede">
<strong>声明</strong>：本文是纯理论分析和猜想，尚未经过工程验证。K3 的部分技术细节基于公开信息和推理，可能与实际实现有出入。但我们认为这个方向值得探索——如果 K3 的架构设计真的和 TPU 如此契合，那这可能是 2026 年最值得尝试的 MoE-on-TPU 移植项目。
</div>

## 核心猜想

**Kimi K3 的架构创新，虽然是为 GPU 训练设计的，但它们解决的恰好是 MoE 在 TPU 上训练的核心痛点。K3 不是"可以在 TPU 上跑"——它可能是迄今为止最适合 TPU 的 MoE 架构。**

这不是巧合。K3 追求的"极致确定性"和 TPU/XLA 追求的"极致静态编译"在哲学上是同一件事。

---

## 背景：MoE 在 TPU 上的历史包袱

TPU 的灵魂是 XLA 编译器。XLA 的核心约束：**所有 tensor 形状必须在编译时确定**。

这和 MoE（Mixture of Experts）的本性直接冲突。MoE 的 router 动态决定每个 token 去哪个 expert——这意味着每个 expert 每 step 收到的 token 数不同，All-to-All 通信的形状是**运行时才知道的**。

### GShard 的妥协（2020）

Google 自己的 GShard 论文就是为了解决这个矛盾而生的。方案：`capacity_factor`——给每个 expert 预留固定容量（如 1.5× 平均值），不足补零，超出丢弃。

```
# GShard 的 capacity_factor 方案
expert_capacity = (total_tokens / num_experts) × capacity_factor
# capacity_factor = 1.5 → 预留 50% 余量

# 结果：
# - 冷门 expert：30% 的 slot 是零向量 → GPU/TPU 做无用计算
# - 热门 expert：超出容量的 token 被丢弃 → 信息丢失
# - XLA 满意了（形状静态），但代价是 ~33% 的算力浪费
```

这是一个**工程妥协**：用 padding 和 dropping 换取编译器的欢心。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/moe-tpu-conflict.svg" alt="MoE 在 TPU 上的历史矛盾与 K3 的解法" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">GShard 用 padding 骗过 XLA，K3 让数据本身均匀——从根本上消灭矛盾</figcaption>
</figure>

### 后来者的选择

DeepSeek V3 等 GPU-first 的模型直接放弃了静态形状——在 CUDA 的世界里，动态形状不是大问题。它们用辅助 loss、EPLB 等技术缓解不均，但 All-to-All 仍然是动态的。

这让 MoE 在 TPU 上的处境越来越尴尬：**主流 MoE 架构在 GPU 上跑得越好，在 TPU 上就越难移植。**

---

## K3 的七重契合

然后 K3 来了。它的创新不是为 TPU 设计的，但每一项都精准命中了 TPU 的需求。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/seven-alignments.svg" alt="K3 的七重 TPU 契合" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">每一项 GPU 创新，恰好命中一个 TPU 痛点</figcaption>
</figure>

### 第一重：Quantile Balancing → 消灭 capacity_factor

K3 用 **Quantile Balancing** 替代 Top-K 路由：对 router logits 按分位数等分，每个 expert **恰好**收到 `total_tokens / num_experts` 个 token。

```
# 传统 Top-K：每个 expert 收到 0 ~ capacity 个 token（动态）
# K3 Quantile Balancing：每个 expert 收到 N/E 个 token（常量）

# 对 XLA 来说：
expert_input_shape = (tokens_per_expert, hidden_dim)  # 编译时常量！
# 不需要 capacity_factor
# 不需要 padding
# 不需要 token dropping
```

**这不是"缓解"了 MoE-TPU 矛盾——是彻底消灭了矛盾的根源。**

GShard 用 padding 骗过 XLA（形状确实静态了，但 33% 是假数据），K3 让数据本身就是均匀的（100% 真实计算）。

<div class="callout-takeaway">
<strong>洞察</strong>：GShard 把"动态分配"视为 MoE 的固有属性，试图在不改变路由机制的前提下适配 XLA。K3 质问了这个前提本身——为什么路由必须是动态的？如果均匀分配不损失模型质量，那所有为动态分配付出的工程代价（padding、dropping、辅助 loss、capacity factor 调参）都是不必要的。
</div>

### 第二重：静态 All-to-All → XLA 全图编译

Quantile Balancing 的均匀分配直接导出一个推论：**All-to-All 通信的 tensor 形状是编译时常量**。

```
# 传统 MoE EP 通信：
# Step 1: Router 决定分配（动态）
# Step 2: Host CPU 统计每个 expert 的 token 数 → 规划通信
# Step 3: 执行 All-to-All（形状动态，无法提前编译）
# ↑ 这里有一次 Host-Device 同步！

# K3：
# Step 1: Quantile Balancing（均匀，形状已知）
# Step 2: 不需要（编译器已知通信量）
# Step 3: 执行 All-to-All（形状静态，已编译进 HLO 图）
# ↑ 无 Host 同步！计算和通信完美 overlap
```

XLA 可以将**整个 MoE forward pass**——包括 router、dispatch、expert FFN、combine——编译成一个优化的 HLO 图。计算和通信在编译时就规划好了 overlap 策略。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/static-vs-dynamic.svg" alt="动态 vs 静态 All-to-All" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">传统 MoE 的 Host-Device 同步阻塞 vs K3 的 XLA 全图编译 + 计算通信 overlap</figcaption>
</figure>

在 GPU 上，这只是"不错的优化"。在 TPU 上，这是"从勉强能跑到高效运行"的质变。

### 第三重：SparseCore 天然 MoE 协处理器

TPU v7 每 chip 有 4 个 SparseCore——独立于 TensorCore 的轻量处理单元，2.4× FLOPs 于 v6e。

SparseCore 可以独立处理 MoE routing 的全部开销：

| 任务 | 传统方案 | K3 + SparseCore |
|------|---------|----------------|
| Router logit 计算 | TensorCore | **SparseCore** |
| Token 排序/分配 | TensorCore | **SparseCore** |
| All-to-All 通信 | TensorCore 等待 | **SparseCore** |
| Expert FFN 计算 | TensorCore | TensorCore |

结果：**TensorCore 100% 用于 expert 计算**。Router 和通信的开销被完全 offload 到 SparseCore，和 expert 计算并行执行。

传统 MoE 的动态路由让 SparseCore offload 变得复杂（动态形状的通信难以提前编排）。K3 的静态形状让 SparseCore 可以完全预编排工作负载。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/sparsecore-offload.svg" alt="TPU v7 SparseCore Offload" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">TensorCore 100% 用于 Expert 计算，路由和通信全部 offload 到 SparseCore 并行执行</figcaption>
</figure>

### 第四重：SiTU 激活函数 → 待定

K3 在 MoE expert 中使用了名为 **SiTU** 的自定义激活函数，替代传统的 SwiGLU。目前 K3 技术报告尚未披露 SiTU 的完整公式和设计细节，仅知其输出有界（约 [-0.2, 1]）且在 2.8T token 规模训练中表现稳定。

<div class="callout-lede">
<strong>诚实声明</strong>：在 SiTU 的详细设计公开之前，我们无法严格论证它与 TPU MXU 的对齐关系。有界激活<em>可能</em>有利于低精度计算和量化，但这属于合理推测，不是确定结论。本文将此列为"待验证"项，待 K3 完整技术报告发布后再更新分析。
</div>

### 第五重：Per-Head Muon → 完美 batch matmul

K3 的 Per-Head Muon 将 Newton-Schulz 正交化按 attention head 粒度执行——128 个 (7168 × 128) 的独立正交化。

这在 TPU 上是**完美的 batch matmul 形式**：

```python
# Per-Head Muon 的 NS 迭代
# 128 个 head，每个 (7168, 128)
# NS 核心运算: X @ X^T @ X

# TPU 实现：直接用 batch matmul
# vmap 或 reshape 成 (128, 7168, 128) 的 batch 维度
# MXU 原生支持 batch matmul，利用率极高
```

对比全矩阵 Muon 的 (7168 × 16384) 正交化——这个尺寸在 TPU 上需要分片处理，效率远不如 per-head 版本。

### 第六重：AttnRes → 无条件分支，XLA 友好

AttnRes 用 softmax attention 替代传统残差连接。从 XLA 的角度：

- 传统残差：`x + F(x)` — 简单但固定
- AttnRes：`softmax(Q · K) · V` — 更复杂但**完全由矩阵运算组成**

AttnRes 的所有操作（矩阵乘、softmax、加权求和）都是 XLA 原生支持的标准操作。没有条件分支、没有动态形状、没有 host callback。整个 Block AttnRes 可以被 XLA 编译成一个高效的 HLO 子图。

### 第七重：KDA + Gated MLA 3:1 混合 → 推理内存革命

K3 的 3:1 混合架构（3 层 KDA + 1 层 Gated MLA）在 TPU 推理上的优势：

- **KDA 层**：常数 KV cache（~1.6 MB/层），与序列长度无关
- **Gated MLA 层**：KV cache 57× 压缩（576 维 latent vs 32768 维全展开）
- **综合**：128K context 下 KV cache 从 280 GB → ~15 GB

TPU v7 每 chip 192 GB HBM3e。传统 MHA 的 280 GB KV cache 需要跨多 chip 分片。K3 的 ~15 GB KV cache 可以**单 chip 放下**，推理时没有跨 chip 通信开销。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/kv-cache-revolution.svg" alt="KV Cache 革命" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">从 280GB 到 15GB——KDA 3:1 + Gated MLA 让 TPU 单 chip 承载 256K 上下文</figcaption>
</figure>

---

## 为什么说是"天赐良缘"而不是"刻意设计"

K3 团队（Moonshot AI）主要在 NVIDIA GPU 上训练。他们的创新动机是：

| 创新 | Moonshot 的动机 | TPU 的收益（意外） |
|------|---------------|-------------------|
| Quantile Balancing | 消除辅助 loss，简化训练 | **消灭 capacity_factor，解锁 XLA 全图编译** |
| SiTU | 2.8T 规模训练稳定性 | **细节未披露，TPU 对齐待定** |
| Per-Head Muon | 128× 降低优化器计算成本 | **完美 batch matmul 形式** |
| Static-Shape EP | GPU 上也有通信优化收益 | **XLA 静态形状的核心要求** |
| AttnRes | 更好的梯度流 + 模型质量 | **纯矩阵运算，XLA 编译友好** |
| KDA 3:1 | 长序列推理效率 | **KV cache 单 chip 放下** |

**每一项创新都有自己的 GPU 动机，但在 TPU 上产生了更大的收益。** 这不是 K3 为 TPU 设计——是 K3 追求的"极致确定性和效率"恰好和 TPU/XLA 的设计哲学高度一致。

<div class="callout-takeaway">
<strong>更深层的洞察</strong>：TPU/XLA 从第一天起就要求"一切在编译时确定"。GPU/CUDA 从第一天起就允许"运行时再说"。过去 6 年，MoE 架构在 CUDA 的自由度中演化，积累了大量"运行时动态"的设计习惯（动态路由、动态 capacity、动态通信）。K3 是第一个在 GPU 上训练、但主动放弃这些动态自由度的模型——因为 Moonshot 发现确定性带来的质量和效率收益大于灵活性的损失。这让 K3 的架构意外地回到了 TPU/XLA 的设计哲学上。
</div>

---

## 移植路径素描

如果要把 K3 移植到 TPU 上训练，基于 MaxText 框架的改造路径：

### Phase 1: 路由层（核心突破）

```python
# 1. Quantile Balancing Router
# 替换 MaxText 的 Top-K routing
def quantile_balance_route(logits, num_experts):
    # logits: (batch × seq, num_experts)
    scores = jax.nn.softmax(logits, axis=-1)
    # 对每个 expert 维度，按分位数分配
    indices = jnp.argsort(scores, axis=0)
    tokens_per_expert = scores.shape[0] // num_experts
    # 每个 expert 恰好分到 tokens_per_expert 个 token
    assignments = indices.reshape(num_experts, tokens_per_expert)
    return assignments

# 2. 静态 All-to-All
# 形状已知 → 直接用 jax.lax.all_to_all
dispatched = jax.lax.all_to_all(
    local_tokens,        # (local_experts, tokens_per_expert, hidden)
    axis_name='expert',  # mesh axis
    split_axis=0,
    concat_axis=0
)
# 形状完全静态，XLA 编译一次即可
```

### Phase 2: 注意力层

- **KDA**：已有 Pallas kernel 实现（Ling3-flash 团队验证过，70% peak FLOPs）
- **Gated MLA**：在 MLA 基础上加门控投影，标准矩阵运算
- **3:1 混合**：层配置，不涉及新 kernel

### Phase 3: 训练组件

- **AttnRes**：Block 内 softmax attention over layer outputs，标准 attention 实现
- **Per-Head Muon**：batch matmul NS 正交化，纯 JAX 实现
- **SiTU**：具体公式待 K3 技术报告披露，实现时需参考官方开源代码

### Phase 4: 量化

- **MXFP4 QAT**：SFT 阶段模拟量化，需要 JAX 的 FP4 仿真支持

---

## 风险和不确定性

<div class="callout-warn">
<strong>这是猜想，不是结论。以下风险需要工程验证。</strong>
</div>

1. **Quantile 排序效率**：大规模 argsort 在 TPU 上的效率未知。896 experts × 百万 token 的排序可能需要近似算法
2. **SparseCore API 成熟度**：v7 SparseCore 的 MoE offload API 可能尚未完全公开
3. **AttnRes 内存开销**：Block 内需存储所有前序层输出，增加 HBM 峰值使用
4. **Quantile Balancing 的质量影响**：强制均匀分配是否在所有任务上都不损失质量？K3 的论据来自预训练，SFT 和 RLHF 阶段未必成立
5. **KDA Pallas kernel 调优**：chunked parallel scan 在新硬件上需要重新调优

---

## 结语

K3 和 TPU 的契合不是刻意设计的结果，而是两种"极致确定性"追求的自然汇合。

TPU 从第一天起就说："告诉我所有形状，我给你最优编译。"MoE 从第一天起就说："我需要动态路由的灵活性。"六年来，这对矛盾催生了 capacity_factor、辅助 loss、动态 All-to-All 等一系列精巧但笨重的妥协。

K3 走了一条不同的路。它问了一个简单的问题：**如果路由不需要动态呢？** Quantile Balancing 给出了肯定的答案——均匀分配不仅不损失质量，还消除了一整个类别的工程复杂性。

巧合的是（或者说必然的是），这个答案恰好是 TPU/XLA 六年来一直在等的那个答案。

**这就是为什么我们称之为"天赐良缘"——不是因为 K3 为 TPU 而生，而是因为 K3 和 TPU 在追求同一个理想的过程中，各自独立地到达了同一个终点。**

---

*本文基于公开信息和理论分析。K3 的完整技术报告尚未发布，部分推理可能与实际实现有差异。欢迎指正。*

*2026-07-18 · [blog.higcp.com](https://blog.higcp.com)*
