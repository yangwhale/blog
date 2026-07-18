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

### TPU v7 (Ironwood) 硬件概览

在进入具体分析前，先建立 TPU v7 的硬件心智模型：

| 参数 | TPU v7 (Ironwood) | 说明 |
|------|-------------------|------|
| 峰值算力 | 4,611 FP8 TFLOPS / 2,306 BF16 TFLOPS | 单 chip |
| HBM | 192 GiB HBM3e (8-Hi stacks) | 7.4 TB/s 带宽 |
| ICI 4.0 | 5,376 Gbps (4 links × 1.34 Tb/s) | 3D Torus 拓扑 |
| 芯片架构 | Dual-Chiplet (2 chiplet/chip) | D2D 带宽 ~8 Tb/s (6× 单 ICI link) |
| SparseCore | 4 SC/chip (2/chiplet) | 16 tiles/SC, 2.5 MB SPMEM/SC |
| 计算带宽比 | 623 FLOPS/byte (FP8) | 计算远快于带宽，通信优化至关重要 |
| Superpod | 9,216 chips | 42.5 Exaflops FP8, 1.77 PB HBM |

一个关键数字：**623 FLOPS/byte 的 FP8 计算带宽比**。这意味着 TPU v7 的计算速度远远快于数据搬运速度——任何通信开销都会直接转化为算力浪费。这是理解后续讨论的基础。

### XLA 的核心约束

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

后来 Google 内部发展了 **Megablox Grouped MatMul**——一个 Pallas kernel 实现（代码位于 `jax/experimental/pallas/ops/tpu/megablox/gmm.py`），将 MoE 的多个 expert matmul 重构为单个 block-sparse matmul，利用 SparseCore 的细粒度 DMA 避免 padding 浪费。但 Megablox 本质上是在"动态形状"前提下做优化——需要复杂的 metadata（token→expert 索引和 ranges），运行时开销不可忽略。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/moe-tpu-conflict.svg" alt="MoE 在 TPU 上的历史矛盾与 K3 的解法" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">GShard 用 padding 骗过 XLA，K3 让数据本身均匀——从根本上消灭矛盾</figcaption>
</figure>

### 后来者的选择

DeepSeek V3 等 GPU-first 的模型直接放弃了静态形状——在 CUDA 的世界里，动态形状不是大问题。它们用辅助 loss、EPLB 等技术缓解不均，但 All-to-All 仍然是动态的。我们实际在 TPU v7 上运行 DeepSeek 系列模型时，MoE routing 的适配是精度调优中最棘手的环节之一。

这让 MoE 在 TPU 上的处境越来越尴尬：**主流 MoE 架构在 GPU 上跑得越好，在 TPU 上就越难移植。**

---

## K3 的七项创新逐一审视

然后 K3 来了。它有 7 项架构创新，每一项都有自己的 GPU 动机。我们需要逐一审视的问题不是"这在 TPU 上能不能用"——而是**"这个创新在 TPU 上产生的边际收益，是否远大于它在 GPU 上的边际收益"**。换句话说：GPU 上是锦上添花的东西，在 TPU 上是不是意外地补上了一块关键短板？

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

GShard 用 padding 骗过 XLA（形状确实静态了，但 33% 是假数据）。Megablox 用 block-sparse matmul 避免 padding（但需要复杂的 ragged batch metadata 和运行时索引管理）。K3 让数据本身就是均匀的——不需要 padding，也不需要 Megablox 的 ragged 处理，因为每个 expert 的 batch 大小本身就是编译时常量。

<div class="callout-takeaway">
<strong>洞察</strong>：GShard 把"动态分配"视为 MoE 的固有属性，试图在不改变路由机制的前提下适配 XLA。Megablox 接受了这个前提，用更精巧的 Pallas kernel 减少浪费。K3 质问了前提本身——为什么路由必须是动态的？如果均匀分配不损失模型质量，那所有为动态分配付出的工程代价（padding、dropping、辅助 loss、capacity factor 调参、ragged batch metadata）都是不必要的。
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

在 TPU v7 上，ICI 4.0 提供每 chip 5,376 Gbps 的互联带宽（4 条 link × 1.34 Tb/s），通过 3D Torus 拓扑连接。但 623 FLOPS/byte 的计算带宽比意味着：**即使 ICI 带宽已经很高，任何通信阻塞计算的时间窗口都会造成严重的算力浪费**。XLA 全图编译的价值在于将通信操作编排到计算间隙中，实现真正的 zero-bubble overlap。

在 GPU 上，这只是"不错的优化"。在 TPU 上，这是"从勉强能跑到高效运行"的质变。

### 第三重：SparseCore Collective Offloading

TPU v7 每 chip 有 4 个 SparseCore（2 个/chiplet），每个 SC 包含 16 个 tile 和 2.5 MB SPMEM。SparseCore 是独立于 TensorCore 的轻量处理器，拥有自己的标量控制器（SCS）和 8-wide SIMD 向量单元（SCT），能够作为**独立控制线程**管理 ICI fabric 上的数据移动——这就是 Collective Offloading 的基础。

SparseCore 的一个关键优势是**内存访问粒度**：支持 4-byte 和 32-byte 的 DMA 操作，而 TensorCore 的 systolic array 优化为 512-byte 粒度加载。这种细粒度访问天然适合 MoE 的 token routing——将稀疏的 token 按 expert 分组，然后通过 ICI 发送。

在 MoE 场景下，TPU v7 的计算分工如下（基于 SparseCore 架构文档）：

| 操作 | 执行单元 | 说明 |
|------|---------|------|
| Expert 选择 (gating) | TensorCore | 小型 dense matmul |
| Token 路由 All-to-All | **SparseCore** | Collective offloading |
| Expert FFN (dense GEMM) | TensorCore MXU | 主要计算负载 |
| Token Combine | **SparseCore** | 结果聚合 |

注意 gating 计算本身仍在 TensorCore 上执行——它是一个小型 dense matmul（router weights × hidden states），SparseCore 的价值不在于替代这个计算，而在于将**后续的 token dispatch、All-to-All 通信和结果 combine 完全 offload**，使其与下一层的 Expert FFN 计算并行。

实测性能：在 pipeline 训练中，Collective Offloading 可实现**最高 2× 提速**；在 MoE All-to-All 场景下，通信延迟被计算掩盖。

K3 的静态形状让这个优势被放大：传统 MoE 的动态路由意味着每个 step 的通信量不同，SparseCore 需要运行时重新规划 DMA 调度。K3 的 Quantile Balancing 保证通信形状恒定，SparseCore 可以在编译时就完全规划好所有 DMA 传输模式，**零运行时调度开销**。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/sparsecore-offload.svg" alt="TPU v7 SparseCore Offload" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">TensorCore 100% 用于 Expert 计算，路由和通信全部 offload 到 SparseCore 并行执行</figcaption>
</figure>

### 第四重：SiTU 激活函数 → 待定

K3 在 MoE expert 中使用了名为 **SiTU** 的自定义激活函数，替代传统的 SwiGLU。目前 K3 技术报告尚未披露 SiTU 的完整公式和设计细节，仅知其输出有界（约 [-0.2, 1]）且在 2.8T token 规模训练中表现稳定。

<div class="callout-lede">
<strong>诚实声明</strong>：在 SiTU 的详细设计公开之前，我们无法严格论证它与 TPU MXU 的对齐关系。有界激活<em>可能</em>有利于低精度计算和量化，但这属于合理推测，不是确定结论。本文将此列为"待验证"项，待 K3 完整技术报告发布后再更新分析。
</div>

### 第五重：Per-Head Muon → 好设计，但 TPU 边际收益不突出

K3 的 Per-Head Muon 将 Newton-Schulz 正交化按 attention head 粒度执行——128 个 (7168 × 128) 的独立正交化，核心运算 X^T @ X 从 (16384, 16384) 降为 128 个 (128, 128)，计算量降低 128×。

```python
# 全矩阵 Muon: X^T @ X → (16384, 16384) 中间矩阵，512 MB (BF16)
# Per-Head Muon: 128 × X^T @ X → (128, 128) 中间矩阵，32 KB (BF16)
# 计算量: O(7168 × 16384²) → O(128 × 7168 × 128²) = 128× 降低
```

Per-Head Muon 的 128× 计算降低是优秀的工程设计，但 GPU 和 TPU 的边际收益相当——batch matmul 在 CUDA 的 batched CUBLAS 和 TPU 的 MXU 上都能高效执行。它没有意外补上 TPU 的某个短板。

### 第六重：AttnRes → 好设计，但 TPU 边际收益不突出

AttnRes 用 softmax attention 替代传统残差连接，在 GPQA-Diamond 上带来 +7.5 分的质量提升。AttnRes 的所有操作（矩阵乘、softmax、加权求和）是 XLA 原生操作——但它替代的传统残差 `x + F(x)` 同样是纯矩阵运算、同样 XLA 友好。AttnRes 在 GPU 和 TPU 上产生相同的质量收益，TPU 侧没有额外的边际增量。

### 第七重：KDA + Gated MLA 3:1 混合 → 推理内存革命

K3 的 3:1 混合架构（3 层 KDA + 1 层 Gated MLA）在 TPU 推理上的优势：

- **KDA 层**：常数 KV cache（~1.6 MB/层），与序列长度无关
- **Gated MLA 层**：KV cache 57× 压缩（576 维 latent vs 32768 维全展开）
- **综合**：128K context 下 KV cache 从 280 GB → ~15 GB

KV cache 从 280 GB 降到 ~15 GB，这个压缩比在 GPU 上同样有效——H100（80 GB）或 B200（192 GB）都能从"需要多卡分片"变成"单卡放下"。**但 TPU 上的收益有一个额外维度**：

GPU 推理框架可以用 PagedAttention 动态分配 KV cache——短序列只分配少量显存，按需增长。TPU/XLA 做不到这一点：**所有 tensor 形状必须在编译时确定**，KV cache 必须按最大序列长度一次性预分配。这意味着传统 MHA 在 TPU 上推理时，即使实际序列只有 1K token，也要预留 280 GB（按 128K 编译）。

KDA 的线性注意力彻底消除了这个问题：它的 KV state 大小与序列长度无关——是一个编译时常量，约 1.6 MB/层。不存在"预分配浪费"的概念。

TPU v7 每 chip 配备 192 GiB HBM3e（8-Hi stacks），带宽 7.4 TB/s。K3 的 ~15 GB KV cache 仅占单 chip HBM 的 **8%**，推理时完全走本地 HBM 的 7.4 TB/s 带宽，零跨 chip 通信开销。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/kv-cache-revolution.svg" alt="KV Cache 革命" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">从 280GB 到 15GB——KDA 3:1 + Gated MLA 让 TPU 单 chip 承载 256K 上下文</figcaption>
</figure>

---

## 为什么说是"天赐良缘"而不是"刻意设计"

K3 团队（Moonshot AI）主要在 NVIDIA GPU 上训练。他们的创新动机是：

| 创新 | Moonshot 的动机 | TPU 的收益（意外） |
|------|---------------|-------------------|
| Quantile Balancing | 消除辅助 loss，简化训练 | **消灭 capacity_factor + Megablox ragged 开销，解锁 XLA 全图编译** |
| SiTU | 2.8T 规模训练稳定性 | **细节未披露，TPU 对齐待定** |
| Per-Head Muon | 128× 降低优化器计算成本 | GPU/TPU 边际收益相当 |
| Static-Shape EP | GPU 上也有通信优化收益 | **XLA 静态形状的核心要求，解锁 SparseCore Collective Offloading 编译时 DMA 规划** |
| AttnRes | 更好的梯度流 + 模型质量 | GPU/TPU 边际收益相当 |
| KDA 3:1 | 长序列推理效率 | **KV cache 压缩通用受益 + KDA 常数 state 消除 TPU 静态预分配浪费** |

诚实地讲：7 项创新中，**3 项在 TPU 上的边际收益远大于 GPU**（Quantile Balancing 消灭了 TPU 独有的 capacity_factor 痛点；Static-Shape EP 解锁了 XLA 全图编译这个 GPU 不需要的能力；SparseCore Offloading 利用了 TPU 独有的硬件），**1 项在 TPU 上有额外放大效应**（KDA 的常数 state 消除了 XLA 静态预分配造成的内存浪费——GPU 用 PagedAttention 本来就没这个问题），**1 项待定**（SiTU），**2 项 GPU/TPU 边际收益相当**（Per-Head Muon、AttnRes 都是好设计，但没有意外补上 TPU 的短板）。

关键在于：前 3 项命中的恰好是 MoE-on-TPU **最致命**的短板——动态形状、编译断裂、通信阻塞计算。在 GPU 上这些从来不是问题，所以 K3 解决它们只是锦上添花。在 TPU 上，这是"从不可行到高效可行"的质变。

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
    return assignments  # 形状: (num_experts, tokens_per_expert) — 编译时常量

# 2. 静态 All-to-All
# 形状已知 → 直接用 jax.lax.all_to_all
dispatched = jax.lax.all_to_all(
    local_tokens,        # (local_experts, tokens_per_expert, hidden)
    axis_name='expert',  # mesh axis
    split_axis=0,
    concat_axis=0
)
# 形状完全静态，XLA 编译一次即可
# 启用 SparseCore Collective Offloading 后，
# All-to-All 由 SparseCore 独立执行，不阻塞 TensorCore
```

TPU v7 的 dual-chiplet 架构带来一个额外的 mesh 维度（on-chip device ID），Expert Parallelism 的 mesh 配置需要考虑 chiplet 间 D2D 带宽（~8 Tb/s，6× ICI link）和跨 chip ICI 带宽的差异。建议将同一 chip 的 2 个 chiplet 优先分配给相邻 expert group，利用 D2D 高带宽做 chip 内 All-to-All。

### Phase 2: 注意力层

- **KDA**：已有 Pallas kernel 实现（Ling3-flash 团队验证过，70% peak FLOPs）。KDA 的线性注意力特性意味着 KV state 大小与序列长度无关——在 TPU 上这消除了 attention 层的 HBM 压力
- **Gated MLA**：在 MLA 基础上加门控投影，标准矩阵运算。latent_dim=576 的低秩 KV projection 在 MXU 上高效执行
- **3:1 混合**：层配置，不涉及新 kernel。每 4 层中 3 层 KDA + 1 层 Gated MLA

### Phase 3: 训练组件

- **AttnRes**：Block 内 softmax attention over layer outputs，标准 attention 实现，所有运算（Q/K/V projection、softmax、加权求和）均为 XLA 原生操作
- **Per-Head Muon**：batch matmul NS 正交化，(128, 7168, 128) 的 batch 维度天然适配 `jax.vmap`
- **SiTU**：具体公式待 K3 技术报告披露，实现时需参考官方开源代码

### Phase 4: 量化

- **MXFP4 QAT**：SFT 阶段模拟量化。TPU v7 原生支持 FP8，MXFP4 需要 JAX 层面的仿真支持，但 SiTU 的有界输出（~[-0.2, 1]）可能降低量化难度

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

### 参考资料

- [OpenXLA SparseCore 深度指南](https://openxla.org/xla/sparsecore) — SparseCore 架构与 Collective Offloading 官方文档
- [Google Cloud TPU v7 (Ironwood) 文档](https://cloud.google.com/tpu/docs/ironwood-performance) — 性能优化与硬件规格
- [GShard: Scaling Giant Models with Conditional Computation](https://arxiv.org/abs/2006.16668) — capacity_factor 的起源
- [Megablox GMM Pallas Kernel](https://github.com/jax-ml/jax/blob/main/jax/experimental/pallas/ops/tpu/megablox/gmm.py) — TPU 上 MoE ragged batch 的 block-sparse 实现
- [SemiAnalysis: TPU v7 分析](https://newsletter.semianalysis.com/p/tpuv7-google-takes-a-swing-at-the) — SparseCore SCS/SCT 架构细节
- [JAX TPU Embedding API](https://github.com/jax-ml/jax-tpu-embedding) — SparseCore 编程接口

*2026-07-18 · [blog.higcp.com](https://blog.higcp.com)*
