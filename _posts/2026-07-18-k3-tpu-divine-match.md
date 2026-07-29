---
layout: post
title: "猜想：K3 可能是最适合 TPU 的 MoE 架构 — 当极致确定性遇上极致静态编译"
date: 2026-07-18 05:00:00 +0000
categories: [tpu, moe, architecture, kimi, attention]
lang: zh
---

<style>
.callout-lede { background: #FEF7E0; border-left: 4px solid #F9AB00; padding: 14px 18px; margin: 16px 0 24px; border-radius: 4px; font-size: 14.5px; line-height: 1.55; color: #3C4043; }
.callout-takeaway { background: #E8F0FE; border-left: 4px solid #1A73E8; padding: 12px 16px; margin: 14px 0 22px; border-radius: 4px; font-size: 14px; color: #202124; }
.callout-warn { background: #FCE8E6; border-left: 4px solid #C5221F; padding: 12px 16px; margin: 14px 0 22px; border-radius: 4px; font-size: 14px; color: #3C4043; }
.callout-ok { background: #E6F4EA; border-left: 4px solid #1E8E3E; padding: 12px 16px; margin: 14px 0 22px; border-radius: 4px; font-size: 14px; color: #202124; }
.rev-banner { background: #F1F3F4; border: 1px solid #DADCE0; border-radius: 8px; padding: 18px 22px; margin: 0 0 28px; font-size: 14.5px; line-height: 1.65; color: #3C4043; }
.rev-banner .tag { display: inline-block; background: #1A73E8; color: #fff; font-size: 12px; font-weight: 600; letter-spacing: .5px; padding: 3px 11px; border-radius: 12px; margin-bottom: 10px; }
.evid { font-size: 12px; font-weight: 600; padding: 2px 8px; border-radius: 10px; margin-right: 6px; white-space: nowrap; }
.evid-hard { background: #E6F4EA; color: #137333; }
.evid-cfg { background: #E8F0FE; color: #174EA6; }
.evid-meas { background: #FEF7E0; color: #B06000; }
.evid-guess { background: #F1F3F4; color: #5F6368; }
.evid-ext { background: #F3E8FD; color: #7627BB; }
</style>

<div class="rev-banner">
<span class="tag">改版 · 2026-07-29</span><br>
本文首发于 2026-07-18，当时 Kimi K3 只有零散公开信息，全文是<strong>纯猜想</strong>。07-27 夜间 Moonshot 发布了开放权重与 <strong>47 页技术报告</strong>，蚂蚁也向 OpenXLA Tokamax 提交了 KDA 算子的 TPU 实现。<br><br>
本次改版基于这三个一手数据源重写。<strong>除了订正首发版的错误，我刻意补齐了六条削弱本文论点的证据——一篇论证"契合"的文章，如果只收集契合的证据，就不值得读。</strong><br><br>
改版后论述主轴也换了：不再是"七项创新逐一打分"，而是「<strong>MoonEP 保证可行性、Quantile Balancing 压低成本</strong>」的分工——而这个分工只在<strong>文本 backbone</strong> 上成立。<br><br>
<span style="color:#5F6368;font-size:13.5px">逐条改动记录见<a href="#附录改版记录">文末附录</a>。正文里所有更正都用 ⚠️ 标出。</span>
</div>

<figure style="margin: 0 0 24px; text-align: center;">
<img src="https://cc.higcp.com/assets/imagen/A-dramatic-wide-angle-digital-20260718-053349.png" alt="K3 与 TPU 的天赐良缘" style="width: 100%; max-width: 800px; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.12);" />
</figure>

<div class="callout-lede">
<strong>证据等级标注</strong>：改版后全文的每个技术判断都带标记——
<span class="evid evid-hard">报告实锤</span> 技术报告原文（一律附英文引用）；
<span class="evid evid-cfg">config 佐证</span> 开放权重的 <code>config.json</code> 字段；
<span class="evid evid-meas">已实测</span> 公开可查的实测数据；
<span class="evid evid-guess">推断</span> 仍属推理，未经验证；
<span class="evid evid-ext">外部资料</span> 来自 K3 之外的公开资料（TPU 官方文档、JAX 源码、第三方 PR）。<br>
凡是没有标记的段落，都是不涉及事实主张的背景或过渡。
</div>

## 核心论点（改版）

首发时的说法是："K3 不是'可以在 TPU 上跑'——它可能是迄今为止最适合 TPU 的 MoE 架构。"

有了一手数据之后，更准确的说法是：

> **在文本 backbone 这一侧，K3 是六年来第一个把"运行时动态"系统性交还给编译期的前沿 MoE 架构。但这个"交还"是分两层完成的：一层是算法，可以直接搬到 TPU 上；一层是训练基础设施，必须自己重写一遍。而在多模态这一侧，K3 自己也没能交还——它把动态性接住了，没有消除。**

方向没有变。变的是三件事：**范围**（只限文本 backbone）、**归因**（是基础设施而非路由算法）、以及**工程量估计**（从"路由层换一下"变成"路由层 + 一整套均衡执行层"）。

---

## 背景：MoE 在 TPU 上的历史包袱

### TPU v7 (Ironwood) 硬件概览

| 参数 | TPU v7 (Ironwood) | 说明 |
|------|-------------------|------|
| 峰值算力 | 4,611 FP8 TFLOPS / 2,306 BF16 TFLOPS | 单 chip |
| HBM | 192 GiB HBM3e (8-Hi stacks) | 7.4 TB/s 带宽 |
| ICI 4.0 | 5,376 Gbps (4 links × 1.34 Tb/s) | 3D Torus 拓扑 |
| 芯片架构 | Dual-Chiplet (2 chiplet/chip) | D2D 带宽 ~8 Tb/s (6× 单 ICI link) |
| SparseCore | 4 SC/chip (2/chiplet) | 16 tiles/SC, 2.5 MB SPMEM/SC |
| 计算带宽比 | 623 FLOPS/byte (FP8) | 计算远快于带宽，通信优化至关重要 |
| Superpod | 9,216 chips | 42.5 Exaflops FP8, 1.77 PB HBM |

<span class="evid evid-ext">外部资料</span> 上表数据来自 Google Cloud Ironwood 官方文档与 SemiAnalysis 的 TPU v7 分析（均见文末参考资料），不属于 K3 的一手来源。

一个关键数字：**623 FLOPS/byte 的 FP8 计算带宽比**。TPU v7 的计算速度远远快于数据搬运速度——任何通信开销都会直接转化为算力浪费。这是理解后续讨论的基础。

### XLA 的核心约束

TPU 的灵魂是 XLA 编译器。XLA 的核心约束：**所有 tensor 形状必须在编译时确定**。

这和 MoE 的本性直接冲突。MoE 的 router 动态决定每个 token 去哪个 expert——每个 expert 每 step 收到的 token 数不同，All-to-All 通信的形状是**运行时才知道的**。

### GShard 的妥协（2020）

Google 自己的 GShard 论文就是为了解决这个矛盾而生的。方案：`capacity_factor`——给每个 expert 预留固定容量（如 1.5× 平均值），不足补零，超出丢弃。

```
expert_capacity = (total_tokens / num_experts) × capacity_factor
# capacity_factor = 1.5 → 预留 50% 余量
#
# 冷门 expert：30% 的 slot 是零向量 → 无用计算
# 热门 expert：超出容量的 token 被丢弃 → 信息丢失
# XLA 满意了（形状静态），但代价是 ~33% 的算力浪费
```

这是一个**工程妥协**：用 padding 和 dropping 换取编译器的欢心。

后来 Google 内部发展了 **Megablox Grouped MatMul**——一个 Pallas kernel 实现，将 MoE 的多个 expert matmul 重构为单个 block-sparse matmul，利用 SparseCore 的细粒度 DMA 避免 padding 浪费。但 Megablox 本质上是在"动态形状"前提下做优化——需要复杂的 metadata（token→expert 索引和 ranges），运行时开销不可忽略。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/moe-tpu-conflict.svg" alt="MoE 在 TPU 上的历史矛盾与 K3 的两层解法" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">K3 把矛盾拆成两层——rank 级通信形状完全静态化，expert 级计算仍需 grouped matmul</figcaption>
</figure>

### 后来者的选择

DeepSeek V3 等 GPU-first 的模型直接放弃了静态形状——在 CUDA 的世界里，动态形状不是大问题。它们用辅助 loss、EPLB 等技术缓解不均，但 All-to-All 仍然是动态的。在 TPU 上适配 DeepSeek 系列模型时，MoE routing 是精度与性能调优中最棘手的环节之一。

这让 MoE 在 TPU 上的处境越来越尴尬：**主流 MoE 架构在 GPU 上跑得越好，在 TPU 上就越难移植。**

---

## 改版核心：可行性与成本的分工

这一节是本次改版最重要的新增内容——也是**改版过程中我自己写错、又被一手来源纠正的地方**，因此格外值得写清楚。

K3 的负载均衡容易被当成一件事（Quantile Balancing）。它其实是两件事，分属算法层和系统层。但两者的关系**不是"缺一不可"，而是"可行性与成本"的分工**。

### 系统层：MoonEP 无条件保证静态形状

<span class="evid evid-hard">报告实锤</span> 真正保证静态形状的是技术报告 §5.2.1 的 **MoonEP**——一套训练侧的 Expert Parallelism 调度方案：

> *MoonEP requires every rank to receive exactly S × K tokens, **where S is the sequence length and K is the number of experts selected per token**, so that all ranks perform identical amounts of computation.*

它的做法是为热门专家**动态创建冗余副本**，并给出一个有证明的上界（E 为专家总数，R 为 EP size）：

> *We prove that a balanced plan always exists with at most E/R redundant experts per rank and that this bound is essentially tight.*

<div class="callout-takeaway">
<strong>这里有一个反直觉的点，值得单独说：<code>E/R</code> 上界与路由质量无关。</strong><br><br>
附录 E 中「定理 1 的证明」开篇第一句是：<em>"The goal is to prove that M(I) ≤ E/R holds for <strong>any</strong> router output I."</em> 证明的构造是反复挑一个欠载 rank 和一个过载 rank 对填，<em>"the process terminates after at most R − 1 fills"</em>——<strong>与路由倾斜程度完全无关</strong>。定理 2 进一步构造出使 M ≈ E/R 的极端 router 输出，说明这个界是紧的。<br><br>
换句话说：<strong>倾斜再大，冗余专家数也被 <code>E/R</code> 硬顶住。</strong>这正是 MoonEP 相对 ECHO / UltraEP 的卖点——后者预设冗余数或设 token 上限，因而 <em>"forced to stop whenever no feasible plan exists within the cap"</em>，而 MoonEP 的预留槽位保证了 <em>"training is never interrupted"</em>。<br><br>
所以静态形状<strong>不依赖</strong>路由算法把负载压得多平——这也是本文改版时最容易想当然写反的地方。
</div>

### 算法层：Quantile Balancing 压低的是成本，不是可行性

<span class="evid evid-hard">报告实锤</span> 先澄清一个很容易搞错的点：**QB 不是"替代 Top-K 路由"，它就是 Top-K 路由。**

> *Load balancing is implemented by adding an expert-specific bias b<sub>j</sub> to the router score used for Top-k selection.* —— §2.3.3

K3 与 DeepSeek-V3 属于同一族：sigmoid router score + 每专家偏置 + `argtopk`。QB 换掉的是**偏置的更新规则**。原方法是定步长试探 `b_j ← b_j + γ·sign(ℓ̄ − ℓ_j)`，*"γ trades off slow adaptation against load oscillation"*；而 *"Maintaining balanced loads becomes more challenging as LatentMoE increases the routed expert pool to 896 per layer"*。

<span class="evid evid-hard">报告实锤</span> 附录 C 把这件事讲得很漂亮：QB 和原方法**优化的是同一个对偶目标**，只是解法不同——

> *A SignSGD step on this objective recovers the fixed-step sign update of auxiliary-loss-free balancing, up to the sign convention b = −β: the sign update retains only the direction of the load error [in Eq. 27], whereas **QB jumps directly to the exact coordinate minimizer of the same dual objective**. This view explains both why QB requires no learning-rate-like hyperparameter and why it equilibrates within a few update steps even for nearly 10³ experts.*

这个目标本身是"最大分数平衡指派"这个线性规划的对偶。所以 QB 不是启发式，是**把试探换成了闭式解**。

<div class="callout-warn">
<strong>⚠️ 首发版的实质错误</strong>：原文写"每个 expert <strong>恰好</strong>收到 <code>total_tokens / num_experts</code> 个 token"，并据此推出"<code>expert_input_shape</code> 是编译时常量"。<strong>这不成立。</strong>报告自己给出了四个限定条件：
<ul>
<li><strong>因果性</strong>：<em>"For causality, the update takes effect only in the next step, i.e., a batch is never routed with a bias derived from itself."</em></li>
<li><strong>无并列假设</strong>：<em>"Assuming no ties, setting this count to q makes ... exactly q margins stay above the threshold."</em></li>
<li><strong>分位数本身是估计</strong>：<em>"Gathering O(mn) margins for an exact quantile is impractical inside the training loop."</em> 实际用直方图，<em>"the error is bounded by the bin width"</em>。</li>
<li><strong>推理时冻结</strong>：<em>"at deployment, routing is a fixed Top-k selection with a frozen bias, and no quantile computation is needed."</em></li>
</ul>
准确的说法是：<strong>QB 让专家负载显著趋近均匀，但不保证逐专家精确相等。</strong>
</div>

<span class="evid evid-guess">推断</span> **那么 QB 在静态形状这件事上起什么作用？** 从附录 E 的构造可以读出方向：填充过程的起点是"把每个 rank 按本地 token 数分成欠载和过载两类"。**完全均衡时一次填充都不需要；一般情况下，更均衡的路由倾向于减少填充轮数与迁移流量。**

需要注意这只是倾向，不是单调关系——冗余专家数取决于溢出 token 散落在多少个不同专家上，与迁移的 token 数不成正比；而且 QB 均衡的是**专家级**负载，rank 是否过载还取决于专家到 rank 的映射。

而 `E/R` 是**预留槽位数**，恒定。真正随路由质量变化的是**实际的专家权重预取与迁移流量、以及在线规划的工作量**。

所以正确的表述是：**MoonEP 单独就能保证静态形状；QB 让这个保证变便宜。** 两层不是缺一不可，是可行性与成本的分工。

（QB 还有一个与形状无关的独立价值：报告指出不均衡路由 *"may leave some experts poorly trained"*——这是训练质量问题，不是系统问题。）

### 关键限定：静态到 rank，不到 expert

<div class="callout-warn">
<strong>⚠️ 首发版第三个实质错误。</strong>原文写"不需要 Megablox 的 ragged 处理，因为每个 expert 的 batch 大小本身就是编译时常量"。报告 §5.2.1 明确否定：
<blockquote><em>Even with the aggregate load perfectly balanced across ranks, the per-expert token counts within each rank remain skewed.</em></blockquote>
—— 所以 K3 还需要一个 <strong>workload-aware 的 routed-expert GEMM 调度器</strong>，在启动前根据当前 token 分布调参。
</div>

- **静态的**：每个 rank 收到的 token 总数（`S × K`），因而 All-to-All 缓冲区形状固定
- **仍然 ragged 的**：rank 内部逐个专家的 token 数

K3 在 GPU 侧的对应做法是 group GEMM + 负载感知调度；TPU 侧的对应物就是 Megablox 一类的 grouped matmul。**这块工程量没有被消灭，只是从"必须处理 0 到 capacity 的极端不均"降级成"处理一个已经被压得很平的分布"。**

### 一个意外收获：均衡计划的通信是近似置换

<span class="evid evid-hard">报告实锤</span> <span class="evid evid-guess">推断</span> 附录 E 的构造引理里藏着一句对 TPU 特别有价值的话：

> *each rank is filled at most once, so its remote tokens come from a single rank*

也就是说，在这个构造出的均衡计划 P\* 下，**每个 rank 的远程 token 只来自一个其他 rank**——通信模式是**近似置换 / 成对交换**，而不是稠密 all-to-all。

<span class="evid evid-guess">推断</span> 这对 TPU 是结构性利好：3D Torus 的 ICI 对成对近邻交换的映射，远好于对稠密 A2A 的映射。**这条比本文后面给出的 SparseCore 与 D2D 两条推断都更硬。**

需要限定的是：这是**存在性证明**里的构造 P\*；报告说线上的规划内核只是 *"near-optimal"*（精确最优用 ILP 离线算作参考），未必保持单源性质。

<div class="callout-takeaway">
<strong>改版后的核心洞察</strong>：GShard 把"动态分配"视为 MoE 的固有属性，试图在不改变路由的前提下适配 XLA；Megablox 接受了这个前提，用更精巧的 kernel 减少浪费。<strong>K3 的路由仍然是 Top-k——它没有取消动态性，而是在系统层用一个带证明上界的机制把不均衡变成不可能发生。</strong><br><br>
这个答案比首发时想象的更工程化，也更难搬。
</div>

---

## K3 的七项创新 + 一项 TPU 承接机制：逐一审视

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/seven-alignments.svg" alt="K3 的多重 TPU 契合" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">八项收益条目 + 一项反例，判定见下文各节</figcaption>
</figure>

需要逐一审视的问题不是"这在 TPU 上能不能用"——而是**"这个创新在 TPU 上产生的边际收益，是否远大于它在 GPU 上的边际收益"**。

### 第一重：Quantile Balancing → 好估计器，但不是编译期常量

完整讨论见上一节。**改版结论：从"消灭 capacity_factor"降级为"显著收窄 ragged 元数据的最坏情况 + 降低 MoonEP 的实际迁移成本"。**

<span class="evid evid-hard">报告实锤</span> <span class="evid evid-guess">推断</span> 报告 §2.3.3 与附录 D 给出了 QB 的实现细节，据此可以推出两个首发版完全没算的 TPU 开销（**报告本身从头到尾没有提过 TPU 或 XLA，下面的成本判断是我的推论**）：

1. **每 token 对 896 个 expert 做 Top-(k+1)**（§2.3.3；k=16，即 Top-17）。XLA:TPU 上 last-dim 很大的 `top_k` 不便宜。
2. **直方图是 `scatter_add`**（附录 D）：每个 rank 把本地的 required bias `r_i,j := α_i − s_i,j` 累加进一个 `n × B` 的计数矩阵（`B = 1000`），step 结束时一次整数 all-reduce。报告称通信量 *"one integer all-reduce of nB values per layer per step, independent of m"*，误差 *"at most a few 10⁻³"*。

<span class="evid evid-guess">推断</span> 第 2 点在 TPU 上反而是个机会：`scatter_add` 到计数矩阵在 TensorCore 上很慢，但**正好是 SparseCore 擅长的细粒度访存形状**。这是本文之前没想到的一个 SparseCore 用例。

> 一个容易搞混的细节：附录 C 的参考解法 Alg. 1 **确实**沿 token 轴与 expert 轴交替做 `desc_sort`（这是理论上的交替坐标最小化）；生产实现是取其一次迭代，并把 expert 轴那次排序换成直方图。所以"K3 完全不排序"的说法是不准确的——它只是不做**全局**排序。

### 第二重：静态 All-to-All → 判断正确，归因需改

<div class="callout-ok">
<strong>✅ 这一节的问题描述被技术报告逐字印证。</strong>下面的伪代码标注了"这里有一次 Host-Device 同步"并指出它阻塞流水线，报告 §5.2.1 "Sync-free execution with static shapes"：
<blockquote><em>In conventional MoE implementations, the per-expert token counts vary across steps and layers, and the host must synchronize with the device at every layer to obtain the actual computation shapes before launching the expert computation, stalling the pipeline between layers. With perfect balance, every rank receives exactly S × K tokens and the computation shapes of all layers are statically known. This eliminates the per-layer MoE host synchronization and alleviates the host-side kernel-launch overhead.</em></blockquote>
</div>

```
# 传统 MoE EP 通信：
# Step 1: Router 决定分配（动态）
# Step 2: Host CPU 统计每个 expert 的 token 数 → 规划通信
# Step 3: 执行 All-to-All（形状动态，无法提前编译）
# ↑ 这里有一次 Host-Device 同步！

# K3（改版更正：机制是 MoonEP，不是 QB）：
# Step 1: MoonEP 规划冗余专家 → 每 rank 恰好 S×K（QB 让这一步更便宜）
# Step 2: 不需要（编译器已知通信量）
# Step 3: 执行 All-to-All（形状静态，已编译进 HLO 图）
```

XLA 可以将 MoE forward pass 的**通信部分**——dispatch 与 combine——编译成静态 HLO 图，在编译时规划 overlap 策略。

<span class="evid evid-hard">报告实锤</span> MoonEP 还顺带解释了通信缓冲区为什么能固定：最坏不均衡下 DeepEP 的零拷贝路径需要 `S × K × R` 的缓冲区，MoonEP **只需要固定的 `S × K`**。这个"固定 buffer"才是下一节 SparseCore 编译期 DMA 规划真正依赖的前提。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/static-vs-dynamic.svg" alt="动态 vs 静态 All-to-All" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">静态化的是 MoE 的通信部分（dispatch / combine），不是整个 forward pass</figcaption>
</figure>

在 GPU 上，这只是"不错的优化"。在 TPU 上，这是"从勉强能跑到高效运行"的质变——**但需要自己把 MoonEP 的等价物写出来。**

### 第三重：SparseCore Collective Offloading（这是 TPU 的承接机制，不是 K3 的创新）

SparseCore 是 TPU 的硬件特性，不是 K3 的创新——它是**承接** K3 静态形状的那一侧。本节保留讨论，但不计入 K3 的创新盘点。

<span class="evid evid-ext">外部资料</span> TPU v7 每 chip 有 4 个 SparseCore，每个 SC 包含 16 个 tile 和 2.5 MB SPMEM，拥有独立的标量控制器（SCS）和 8-wide SIMD 向量单元（SCT），可作为**独立控制线程**管理 ICI fabric 上的数据移动。SparseCore 支持 4-byte 和 32-byte 粒度的 DMA，而 TensorCore 的 systolic array 优化为 512-byte 粒度——这种细粒度访问天然适合 MoE 的 token routing。

| 操作 | 执行单元 |
|------|---------|
| Expert 选择 (gating) | TensorCore（小型 dense matmul） |
| Token 路由 All-to-All | **SparseCore** |
| Expert FFN (dense GEMM) | TensorCore MXU |
| Token Combine | **SparseCore** |
| QB 的直方图 `scatter_add` | **SparseCore**（改版新增的用例） |

**⚠️ 一个需要收紧的表述**：说这里是"零运行时调度开销"并不准确。更严谨的说法是：**rank 级完美均衡（由 MoonEP 保证）使 All-to-All 的通信形状恒定，SparseCore 可以在编译时规划好 rank 间的全部 DMA 传输模式；但 rank 内部逐专家的 grouped matmul 仍是 ragged 的，仍需运行时的 token→expert 索引。**

所以准确的说法是"**通信侧编译期规划，计算侧仍需 ragged 元数据——但后者的最坏情况已大幅收窄**"。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/sparsecore-offload.svg" alt="TPU v7 SparseCore Offload" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">通信侧可编译期规划 DMA，计算侧仍需 ragged 元数据</figcaption>
</figure>

### 第四重：SiTU-GLU → 有界激活，服务低精度

<span class="evid evid-hard">报告实锤</span> <span class="evid evid-cfg">config 佐证</span> 首发时这一项被列为"待定"。现在完整公式已披露（§2.3.2 + 附录 B）：

```
SiTU-GLU(x) = β₁·tanh(W_g x / β₁) ⊙ Sigmoid(W_g x) ⊙ β₂·tanh(W_u x / β₂)
```

K3 取 β₁ = 4（gate 分支）、β₂ = 25（up 分支）。`config.json` 佐证：`hidden_act: "situ"`、`activation_situ_beta: 4.0`、`activation_situ_linear_beta: 25.0`。附录 B 给出输出上界 `‖SiTU-GLU(x)‖∞ ≤ β₁β₂ = 100`。

动机写得很直白：

> *both multiplicative factors in SwiGLU are unbounded, so coincident large coordinates can produce activation outliers and increase overflow risk in **low-precision arithmetic**.*

<div class="callout-warn">
<strong>⚠️ 首发版更正</strong>：原文猜测"有界激活<em>可能</em>有利于低精度计算和量化"——<strong>方向猜对了</strong>，但写的数值区间「约 [-0.2, 1]」是错的，<strong>真实界是 |f(x)| ≤ 100</strong>。
</div>

<span class="evid evid-hard">报告实锤</span> "有界激活服务低精度"这条链的证据在 §4.1.4，而不在 `config.json`：

> *we quantize the MoE expert weights ... to MXFP4, **with activations computed in MXFP8**, while all non-expert components ... remain in higher precision. We perform quantization-aware training (QAT) throughout the entire post-training stage, covering both SFT and RL ... During RL, rollout and training share the same quantization scheme — eliminating the train–inference mismatch.*

**激活跑在 MXFP8 上**——这才是 SiTU 有界性直接服务的对象。`config.json` 里的 `mxfp4-pack-quantized` 是**权重侧**的打包格式（`input_activations: null`），不能用来佐证激活有界性。

<div class="callout-warn">
<strong>⚠️ 改版新增的削弱证据：MX 格式在 TPU 上没有原生 dtype。</strong><span class="evid evid-guess">推断</span> MXFP4 / MXFP8 是每 32 个元素共享一个 E8M0 指数的微缩放格式，是 Blackwell 的原生能力；TPU v7 原生支持的是 FP8，截至本文写作时 XLA:TPU 未提供 MX 系列 dtype。<strong>这意味着 K3 的开放权重不能直接喂给 TPU</strong>——需要 unpack 后重新量化，MXFP4 的存储与算力优势基本归零；想复现它的 post-training，还得自建一套 MX QAT。（这条是本文的判断，报告与 config 都没有讨论 TPU。）
</div>

**对 TPU 的意义（修订后）**：SiTU-GLU 本身是 tanh + sigmoid + 逐元素乘，全部 XLA 原生，移植零障碍，数值稳定性收益 GPU 与 TPU 同等。**这一项从"待定"移入"边际收益相当"，同时它所服务的 MX 量化路径反而是一项 TPU 侧的额外成本。**

### 第五重：Per-Head Muon → 好设计，但它的动机常被讲反

<div class="callout-warn">
<strong>⚠️ 首发版这一整段的三个数字都是编造的，必须撤回。</strong>原文写"<strong>128 个</strong> (7168 × 128) 的独立正交化，核心运算从 <strong>(16384, 16384)</strong> 降为 128 个 (128, 128)，计算量降低 <strong>128×</strong>"。<br><br>
事实：<code>config.json</code> 里 <code>num_attention_heads: 96</code>（K2 才是 64），<strong>不是 128</strong>；而技术报告 §2.5 <strong>通篇没有给出任何维度或加速数字</strong>。那三个数是首发版自己算错并写死的。
</div>

<span class="evid evid-hard">报告实锤</span> 报告实际怎么说的：

> *instead of applying Newton–Schulz orthogonalization to the full Q, K, and V projection matrices, we partition their momentum matrices along the head dimension and orthogonalize each head's block separately. The intuition is that full-matrix orthogonalization treats all heads as a single coupled block, so heads with larger gradient or momentum scales dominate the shared update direction, while smaller-scale heads receive insufficiently normalized updates; **per-head orthogonalization equalizes the update scale across heads**. In practice, this design yields more balanced learning dynamics across heads and **improves training stability at larger scales**. **It also slightly reduces optimizer overhead**...*

**首发版把动机写反了**：真正的动机是**均衡各头的更新尺度、提升大规模训练稳定性**；降低优化器开销只是报告用 "slightly" 修饰的附带效果。

<span class="evid evid-hard">报告实锤</span> 而且真正的工程瓶颈也不在正交化的算力上。§5.2.2：

> *the Newton–Schulz orthogonalization in Muon requires the full parameter matrix, necessitating a communication step to gather complete parameters before each update. The naive approach performs an all-gather over the entire parameter buffer on every rank, which incurs a substantial memory footprint on top of **making communication the primary bottleneck at scale**.*

K3 的解法是每个 rank 只通过 **P2P** 向 owner rank 拉取自己拥有的那些 shard，并按 model-chunk 粒度做通信-计算流水。

<span class="evid evid-guess">推断</span> **对 TPU 的意义**：算术部分确实是 `jax.vmap` 直出的 batch matmul，白捡；但通信部分在 XLA 里没有"只拉我拥有的 shard"这种原语，要用 `ppermute` 手工编排。**因此这一项从"白捡"降为"半白捡"。**

### 第六重：AttnRes → 好设计，TPU 边际收益不突出

AttnRes 把"当前 token 可以选择性读取历史位置"这个思路从序列维度搬到网络深度维度。

<span class="evid evid-hard">报告实锤</span> <span class="evid evid-cfg">config 佐证</span> K3 用的是 **Block AttnRes** 而非 Full AttnRes：*"we partition its layers into 8 blocks with 12-layer size, giving a **partial final block** and 9 total blocks when counting the embedding layer"*（`attn_res_block_size: 12`；93 层切 8 块，末块不满）。内存与通信开销从 `O(Ld)` 降到 `O(Nd)`。

两个细节：查询是**每层学习得到的伪查询** `q_l = w_l`，不是当前 token 动态生成的；键做了 RMSNorm，*"prevents layers with large-magnitude outputs from dominating the weights"*。

<span class="evid evid-hard">报告实锤</span> §5.2.2 还给了一个配套优化：block 表示在边界层生成一次、被后续所有层共享，AttnRes 计算**整体 checkpointing**，因此"每层为反向保存的激活与标准残差架构完全相同"；PP 下只增量传输新生成的 block 并在 micro-batch 结束时释放。**这直接解掉了首发版风险列表的第 3 条（AttnRes 内存开销）。**

AttnRes 的所有操作都是 XLA 原生算子，但它替代的传统残差同样如此。GPU 和 TPU 收益相同。

### 第七重：KDA + Gated MLA → TPU 侧收益最大的一项，但代价不小

<span class="evid evid-cfg">config 佐证</span> K3 的混合结构：每个 block 3 层 KDA + 1 层 Gated MLA，**backbone 末尾额外再放一层 Gated MLA**，*"ensuring that the final layer always performs global attention"*。93 层里 69 层 KDA、24 层全注意力。

#### 收益：省掉一整套 paged KV 机制

<div class="callout-warn">
<strong>⚠️ 首发版这里写错了一句话，而且它是本文唯一"TPU 独有放大"判定的地基。</strong><br>
原文写："GPU 推理框架可以用 PagedAttention 动态分配 KV cache。<strong>TPU/XLA 做不到</strong>，KV cache 必须按最大序列长度一次性预分配。"<br><br>
<strong>这不成立。</strong>JAX 的 Pallas 算子库里公开有 <code>paged_attention</code> 与 <code>ragged_paged_attention</code> 两个 TPU kernel（<code>jax/experimental/pallas/ops/tpu/</code>），vLLM 的 TPU 后端与 MaxText 都用得上。TPU 上做 paged KV 是可行的。
</div>

<span class="evid evid-guess">推断</span> 修正后的说法是**程度差，不是有无差**：

- **GPU 侧**：PagedAttention 是框架标配，成熟、生态完整，用户基本不用关心。
- **TPU 侧**：能做，但要走 Pallas kernel + page table 索引 gather；page pool 的总量仍是编译期常量，池子开多大要提前决策；变长场景还得用 `ragged` 变体。这是一整套需要维护的机制。

**KDA 的价值因此不是"从不可能到可能"，而是"根本不需要这一整套东西"**——固定大小的递归状态天然就是编译期常量，没有 pool、没有 page table、没有碎片、不需要专用 kernel。

**这仍然是本文盘点下来 TPU 侧收益最大的一项，但它是省掉了一层工程，不是解锁了一个不可能。**

#### 改版新增：下界衰减 —— 最强的支持证据

<span class="evid evid-hard">报告实锤</span> <span class="evid evid-cfg">config 佐证</span> K3 相对 Kimi Linear 改了 KDA 的衰减参数化，从无界的 negative-Softplus 换成**有下界的 scaled sigmoid**：

```
g = g_min · Sigmoid(e^A · z),   α = exp(g)
g_min = −5 固定，A 为每头可学习 log-scale
```

`config.json` 里可以直接读到：`gate_lower_bound: -5.0`。

**为什么要改？** 分块形式里要用累积衰减的倒数 `1/Γ` 去重标定 key，而 `Γ` 是一串 (0,1) 因子连乘，倒数可以无界增长直到溢出。Kimi Linear 的对策是 log 空间 + 16-token 小块——非对角小块可以走稠密矩阵乘，但：

> *The diagonal tiles, in contrast, still require explicit position-pair computations, which remain **the main intra-chunk bottleneck**.*

K3 的解法是直接给 log-decay 加下界：

> *With g_min = −5, every retention factor satisfies α > e⁻⁵ ≈ 6.7 × 10⁻³, and the cumulative log-decay over a 16-token tile lies in (−80, 0). The corresponding reciprocal rescaling factor is therefore smaller than e⁸⁰ and **remains within the BF16 dynamic range**. This finite range allows both diagonal and off-diagonal tiles to use **dense Tensor Core matrix multiplications, eliminating the position-pair diagonal path**.*

<div class="callout-ok">
<strong>这是本文核心论点最有力的新证据。</strong>一个纯粹为了"让计算落回稠密矩阵乘"而做的算法改动——牺牲衰减的表达范围，换回硬件友好的计算形态。而它受益的机制在 GPU 和 TPU 上<strong>是同一个</strong>：MXU 同样只吃稠密矩阵乘，同样厌恶逐位置对的特殊路径。
</div>

<span class="evid evid-ext">外部资料</span> 这条已经在 TPU 侧落地：蚂蚁提交的 KDA Pallas kernel（[Tokamax PR #1103](https://github.com/openxla/tokamax/pull/1103)）里，原始门激活的约束就写着 `-5 ≤ lower_bound < 0`。

#### 改版新增：NoPE

<span class="evid evid-cfg">config 佐证</span> K3 给所有 MLA 层用了 No Position Encoding（`mla_use_nope: true`），位置敏感性完全交给中间的 KDA 层。报告说这样 *"avoids modifying positional-encoding parameters when extending the context length, such as retuning a RoPE frequency base or applying YaRN"*。对每换一次上下文长度就要重新编译的 TPU 是实打实的好处——但 GPU 也享受。

#### 改版新增：一条 TPU 白拿的便宜

<span class="evid evid-hard">报告实锤</span> §2.1.2 提到一个 GPU 侧的麻烦：

> *To correct the biased rounding error that arises in flash attention, we ... keep the attention output in FP32 during training. **This choice doubles the on-chip footprint of the output tile**; we therefore redesign the training kernel to overlap it with the KV staging buffers instead of the query tile.*

<span class="evid evid-guess">推断</span> MXU 本来就是 FP32 累加输出，VMEM 的布局压力也不同于 warp 级 shared memory。**这条 GPU 需要重写 kernel 才换来的数值正确性，TPU 基本白拿。**

#### 三条容易被忽略的代价

<div class="callout-warn">
<strong>⚠️ KDA 的固定 state 不是纯红利。报告给出了三处它带来的新麻烦。</strong>
</div>

<span class="evid evid-hard">报告实锤</span> **(1) 投机解码下状态无法回滚。** §5.4.2：

> *This in-place update becomes problematic in MTP-based speculative decoding: if verification rejects a subset of the drafted tokens, **the state has already advanced beyond the last accepted token and cannot be trivially rolled back**. Maintaining a state snapshot for each draft position would enable rollback, but would also multiply state traffic — a cost that dominates at the large batch sizes typical of online serving.*

K3 的解法（与并发工作 ReplaySSM 相同）是只缓存投影输入、在片上**重放**重建被接受 token 的状态，并把 short conv、input norm、gating、KDA 递归、output norm 融进**一个 kernel 的一个递归循环**。

<span class="evid evid-guess">推断</span> 对 TPU 的含义：接受长度是数据相关的 → 循环 trip count 动态，XLA 只能按 draft window 全量 padding；而且这是一个比 Tokamax 现有 KDA kernel 复杂得多的融合 Pallas kernel。**softmax KV cache 反而没有这个问题。**

<span class="evid evid-hard">报告实锤</span> **(2) 混合架构让前缀缓存变复杂。** §5.4.1：

> *A KDA layer maintains a single large recurrent state per sequence rather than per-token entries, so state snapshots are affordable only at sparse boundaries; **the shared block size is therefore forced to 1024–6144 tokens** ... At such a coarse granularity **caching is nearly useless**.*

K3 的解法是把哈希粒度（512 token）与物理块粒度解耦、只在稀疏的 hash 端点持久化 KDA checkpoint、跨 cache group 原子失效、命中块全组 pin、copy-on-write。报告直言 *"Checkpoints are large"*。

<span class="evid evid-guess">推断</span> 这是一整套**运行时动态**的内存管理子系统，正是 XLA 最不擅长的一类工作；而且大 checkpoint 把"常数 state 省内存"的红利吐回去了一部分。

<span class="evid evid-hard">报告实锤</span> **(3) KDA 有三种执行 regime，TPU 侧只覆盖了一种。** §5.1.1 除了 chunkwise 训练/prefill kernel（FlashKDA）外，还有一条 **intra-device SM 级 CP planner**：纯 TP 下超长 prefill *"leaves most SMs idle when each rank holds only a few heads"*，于是在单个 rank 内部按 SM 切序列，*"incurs no cross-device communication"*。解码则是第三种 regime。

<span class="evid evid-guess">推断</span> TPU 没有 SM 这一层可调度的并行；这条要么映射成额外的 sharding 轴（触发跨芯片通信，收益被吃掉），要么靠 Pallas grid——而 grid 步在 TPU 上是串行的。这可能也是下文实测里 HBM 带宽利用率只有 26–28% 的原因之一。

<figure style="margin: 24px 0; text-align: center;">
<img src="/assets/images/k3-tpu/kv-cache-revolution.svg" alt="KV Cache 压缩" style="width: 100%; max-width: 860px;" />
<figcaption style="color: #5F6368; font-size: 13px; margin-top: 6px;">KDA 3:1 + Gated MLA 大幅压缩 KV cache。图中绝对数值为本文估算，非官方数据</figcaption>
</figure>

### 第八重（改版新增）：Stable LatentMoE → 让专家池翻倍而通信不涨

<div class="callout-warn">
<strong>⚠️ 这是首发版本的遗漏，不是勘误。</strong>技术报告把 K3 的架构主线概括为三个维度：<strong>KDA 管序列长度、AttnRes 管网络深度、Stable LatentMoE 管模型宽度</strong>。首发版写了前两个，第三个只提到了 Quantile Balancing——而 QB 其实只是 Stable LatentMoE 三个组件之一。
</div>

<span class="evid evid-hard">报告实锤</span> 常规 MoE 把完整的 `d` 维隐藏状态发给每个被选中的专家，激活专家越多，通信量与专家权重读取量就同比例增长。LatentMoE 把路由专家的工作宽度与模型宽度解耦：共享专家保留全宽通路，路由专家在压缩后的 latent 空间里算。

<span class="evid evid-cfg">config 佐证</span> K3 的数字：`hidden_size: 7168` → `routed_expert_hidden_size: 3584`，**每个专家收到的向量宽度正好压一半**；896 个路由专家选 16 个（报告称稀疏度 56），另有 2 个全宽共享专家。

<div class="callout-warn">
<strong>⚠️ 这里容易读出一个错误结论——"EP 通信量减半"。不是的。</strong>技术报告 Table 1 显示，K3 同时把 <strong>Experts Active per Token 从 8 提到了 16（↑100%）</strong>。宽度减半 × 激活数翻倍，相对 Kimi K2 的 per-token dispatch 流量是<strong>持平</strong>，不是净减半。<br><br>
报告自己的措辞很准确：LatentMoE <em>"makes this expansion affordable"</em>——它让扩张变得<strong>可负担</strong>，而不是让通信绝对下降。省下来的预算被换成了更大的专家池和更高的激活数。
</div>

代价是"降维 → 门控多分支专家 FFN → 升维"构成一条近四次连乘的病态链路，在 2.8T 规模下放大异常激活。另外两个组件就是压这个的：**聚合后升维前插 RMSNorm**（报告称还持续改善验证损失与下游指标），以及 **SiTU-GLU**。

<span class="evid evid-hard">报告实锤</span> 但它在**解码**侧带来一个 TPU 不友好的性质。§5.4.2：

> *For routed experts, at small batch sizes, the group GEMMs reduce to **memory-bound streaming of weight matrices** — a regime for which conventional **tile-centric kernels are poorly suited** due to their compute-oriented design and preprocessing overheads.*

K3 的对策是改用 WarpDecode 的 **token-centric** 设计：每个 warp 负责一个输出神经元、直接从显存流式读权重。

<span class="evid evid-guess">推断</span> 需要说清楚这条**为什么**对 TPU 不利——小 batch 解码是 memory-bound，这一点在任何硬件上都成立，GPU 换 WarpDecode 也没让 Tensor Core 忙起来，它省的是预处理与访存布局。TPU 的具体劣势有两条：**(a)** Megablox 一类 grouped matmul 的 ragged 元数据与预处理成本，在解码这种小算量场景里摊不掉；**(b)** TPU 没有 warp / lane team 这一级可以做 token-centric 重排，只能靠 Pallas grid，而 grid 步在 TPU 上是串行的。**这一项在训练侧是收益，在解码侧对 TPU 是负担。**

---

## 反例：原生多模态 —— K3 自己也没交还给编译期的那部分

<span class="evid evid-hard">报告实锤</span> K3 是**原生**多模态：文本、图像、视频由同一个 backbone 在同一个 context 里处理，*"with no post-hoc modality-alignment stage"*。视觉编码器 MoonViT-V2 从零用 next-token prediction 训练，支持到 3584 × 3584 像素的输入。

而报告把它列为 3T 级训练的**三大 infra 难题之一**：

> *(iii) **the vision encoder's highly variable computation is exposed on the critical path**.*

§5.2.3 的解法：

> *large images and long videos substantially increase the computation time of the vision encoder and cause **significant load imbalance across devices**. ... A single large image is partitioned along the patch dimension across multiple devices, and attention is computed by gathering key–value pairs (gather-KV) across CP ranks. In addition, we divide each CP group into several sub-CP groups and distribute multiple large images across them in a load-balanced manner.*

<span class="evid evid-guess">推断</span> **这是 K3 里唯一一处真正无法交还给编译期的动态性。** 每个样本的视觉 token 数取决于图像分辨率和视频长度，本质上是数据相关的。K3 的做法是**把这个动态性接住**（动态 CP + 子组负载均衡 + 塞进 PP bubble），而不是消除它。

在 TPU 上，这意味着两条路：要么按分辨率分桶 + padding（浪费算力，且桶数多会触发反复重编译），要么在 XLA 里实现变长 CP——后者正是 XLA 最难的一类工作。

**因此本文的核心主张必须加限定：K3 把动态性交还给编译期，这件事发生在文本 backbone 上；多模态支线不在其内。** 如果只做纯文本训练，这一节可以忽略；如果要复现 K3 的原生多模态能力，这是移植路径上一块独立且不小的工程。

---

## 为什么说是"天赐良缘"—— 改版后的诚实盘点

<div class="callout-takeaway">
<strong>先锚定判据，否则下面这张表会读出错误的结论。</strong>本表打的是<strong>边际收益差</strong>——同一个创新，在 TPU 上换回的好处是否明显大于在 GPU 上。而"良缘"讲的是另一件事：<strong>可移植性 / 可编译性</strong>。<br><br>
一个架构完全可以在两边的<em>收益</em>相同，却只有在 TPU 上才第一次变得<strong>可编译</strong>——因为 GPU 那边本来就不需要它可编译。表里判"相当"的项，不代表它对 TPU 不重要；只代表它不是"TPU 专属红利"。<br><br>这也解释了为什么<strong>表里排第一的和结语里的主角不是同一个</strong>：按<em>边际收益</em>排，KDA 的常数 state 第一；按<em>可编译性</em>排，MoonEP 的静态形状才是 XLA 一直在等的那个答案。两个坐标轴，两个冠军。<br><br><span style="color:#5F6368;font-size:13px">下表按 TPU 侧收益排序，不按正文里「第几重」的出场顺序。</span>
</div>

| # | 创新 | Moonshot 的动机 | TPU 的收益（改版后） | 判定 |
|---|------|---------------|-------------------|------|
| 七 | **KDA 3:1 + Gated MLA** | 长序列推理效率 | **常数 state 消除 XLA "按最大长度预分配 KV" 的浪费——GPU 有 PagedAttention，本来就没这个问题** | **TPU 独有放大**（但有三条新代价） |
| 二 | **静态 All-to-All（MoonEP）** | 消除 EP 负载不均与显存碎片 | **rank 级 `S×K` 静态形状 → 消除每层 Host 同步、固定 A2A buffer → 解锁 XLA 全图编译与 SparseCore 编译期 DMA** | **收益大，但需在 TPU 侧重写** |
| 一 | **Quantile Balancing** | 896 专家下定步长偏置更新失效 | 压低 MoonEP 的实际迁移成本、收窄 ragged 元数据最坏情况；直方图 `scatter_add` 是新的 SparseCore 用例 | 相当 *(原判定：TPU 远大于 GPU)* |
| 七 | **KDA 下界衰减** | 消掉 position-pair 对角路径这一块内主瓶颈 | 让全部 tile 落回稠密矩阵乘——MXU 的本命 | 相当（但对 TPU 极重要） |
| 八 | **Stable LatentMoE** | 扩专家池而不让通信同比增长 | 同等激活数下逐专家宽度减半（K3 用它把激活数 8→16，净流量持平）；**解码侧 token-centric 需求对 tile-centric 的 MXU 不利** | 训练相当 / 解码不利 |
| 四 | **SiTU-GLU** | 压住低精度下的激活溢出 | 纯 XLA 原生算子；但其服务的 MX 量化路径 TPU 无原生 dtype | 相当（量化路径是额外成本） |
| 六 | **AttnRes** | 跨深度信息流 + 模型质量 | Block 化后 `O(Nd)`，全 XLA 原生算子 | 相当 |
| 五 | **Per-Head Muon** | **均衡各头更新尺度、提升稳定性** | 算术白捡；但 P2P shard 聚集在 XLA 里要手工编排 | 半白捡 |
| — | **原生多模态** | 视觉在环的长程 agent | ❌ 变长视觉 token 无法交还编译期，需分桶或变长 CP | **反例** |

改版后重新盘点，三类结论并列：

- **1 项 TPU 独有放大** —— KDA 的常数 state 消除了 XLA 静态预分配的浪费（但伴随三条新代价）
- **1 项 TPU 侧的质变，需自建等价物** —— MoonEP 的 rank 级静态形状。这是全文唯一被技术报告**逐字印证**的收益，它不是"打折"的，只是不跟着模型权重一起过来
- **1 项明确的反例** —— 原生多模态
- 其余边际收益相当或半相当；**0 项待定**

这个盘点比首发时（3 + 1 + 1 + 2）保守得多。

<div class="callout-takeaway">
<strong>但结论的方向没有变。</strong>TPU/XLA 从第一天起就要求"一切在编译时确定"，GPU/CUDA 从第一天起就允许"运行时再说"。过去六年，MoE 架构在 CUDA 的自由度中演化，积累了大量"运行时动态"的设计习惯。<br><br>
K3 是第一个在 GPU 上训练、却在文本 backbone 上主动把这些动态自由度交还给编译期的前沿模型——不是出于对 TPU 的善意，而是因为在 2.8T + 896 专家的规模上，确定性带来的收益已经大于灵活性。<br><br>
<strong>变的只是我们对工程量的估计。这不是坏消息——这是把一个感觉降级成了一份工单。</strong>
</div>

---

## 移植路径：分层成本估算

这一节按**工程量**分三层——这才是决策时真正需要的信息。

### 第一层：算法与算子层，基本白捡

| 组件 | 说明 |
|------|------|
| **SiTU-GLU** | tanh + sigmoid + 逐元素乘，全是 XLA 原生 |
| **Block AttnRes** | 标准 attention over block 表示；需要处理跨 PP stage 的增量传输 |
| **Stable LatentMoE 的矩阵部分** | 降维/升维就是两个 dense matmul |
| **Gated MLA + NoPE** | 低秩投影 + 门控；NoPE 反而少一类随上下文长度变化的参数 |
| **Quantile Balancing** | 纯算法，但 `top_k(17)` over 896 与直方图 `scatter_add` 在 XLA:TPU 上的效率未验证 |
| **Per-Head Muon（仅算术部分）** | `jax.vmap` 直出；通信部分见第三层 |

QB 的真实算法骨架：

```python
# QB 产出的是"下一步用的 bias"，不是 assignment；router 是 sigmoid 不是 softmax
s = jax.nn.sigmoid(x @ W_r)                    # (m, n_experts)

# Top-(k+1)：前 k 个是实际路由，第 k+1 个是该 token 的准入线
topk1 = jax.lax.top_k(s + b, k + 1).values
alpha = topk1[:, k:k+1]                        # (m, 1)

# 直方图统计 required bias r = alpha - s，取 k/n 分位数（负号翻转了顺序）
#   B = 1000 个 bin，区间 [b_min - 1, b_max + 1]，每步重算
#   每 rank 本地 scatter_add，step 末一次整数 all-reduce
b_next = histogram_quantile(alpha - s, q=k / n_experts, axis=0, bins=1000)
b_next = b_next - b_next.mean()                # 减均值不改变 Top-k 结果

# 因果性：b_next 只作用于下一个 step，绝不用于产生它自己的那一批
```

### 第二层：已经有人写了（KDA kernel 的一种 regime）

<span class="evid evid-meas">已实测</span> 蚂蚁已向 OpenXLA Tokamax 提交完整 TPU 实现（[PR #1103](https://github.com/openxla/tokamax/pull/1103)，约 1.07 万行）——XLA 递归参考实现 + Pallas 分块前向 + custom-VJP 反向 + 变长打包 + 序列维 Context Parallelism + autotuning 注册。

其中的 CP 方案与报告 §5.1.2 的 **KDA Context Parallelism** 是同一个算法：每个 rank 本地折叠出仿射摘要（累积转移矩阵 `M` 与从零起算的状态 `S̃`），一次 all-gather 交换这两个**固定大小**的张量，再用前缀扫描重建各 rank 的入口状态——**通信量与序列长度无关**。这是线性注意力相对 softmax attention 在 CP 上的结构性优势。

**实测（TPU v7，cp_size=4）**：

| 指标 | 数值 |
|------|------|
| 端到端 | 3.515 ms → 2.288 ms |
| 加速比 | **1.54×**（理想 4×，效率 38.4%） |
| CP all-gather | 45.4 µs / 10.0 MB / 带宽利用率 6% |
| 两个主 kernel HBM 带宽利用率 | 25.8% 与 28.0% |

<span class="evid evid-guess">推断</span> **瓶颈定位（这是诊断，不是测量）**：all-gather 只占 45 µs 且带宽利用率仅 6%，说明它是小消息、延迟受限，不是通信瓶颈。真正压住加速比的应该是 rank 本地那段**必须有序**的仿射摘要扫描——典型的 Amdahl 现象。两个主 kernel 的 HBM 带宽利用率 26–28%，距 roofline 有 3.5–4× 的名义空间（实践天花板通常 60–80%，实际可挖约 2–3×）。

> **溯源声明**：以上数字全部转引自 PR #1103 的描述，本文未复现，也未记录该 PR 自己那份清单要求的 JAX/libtpu 版本、逻辑与物理长度、是否含编译开销等字段。PR 作者另标注这些是 pre-refactor 历史基线，不代表当前 head。该 PR 还附了一份"重新 benchmark 必须记录什么"的清单（源 commit、JAX/libtpu 版本、逻辑/物理长度、是否含编译开销、报告 CP 加速必须用配对负载），这个严谨度值得学。

**注意这一层只覆盖了三分之一**：KDA 有训练/prefill、intra-device SM 级 CP、解码三种 regime（§5.1.1、§5.4.2），Tokamax PR 覆盖的是第一种。

### 第三层：必须自己造

<div class="callout-warn">
<strong>这是移植路径里真正的大头。</strong>
</div>

| 要造的东西 | 为什么 GPU 侧的实现搬不过来 |
|------|------|
| **MoonEP 的 TPU 等价物** | 在线冗余专家规划（含 `E/R` 上界保证）、专家权重预取与迁移、反向的本地 reduce buffer、零拷贝 permute/unpermute。整套围绕 NCCL / DeepEP 语义 |
| **KDA 解码 kernel + 投机解码重放** | ReplaySSM 式的片上状态重放，融合 short conv / norm / gating / 递归 / output norm 于单个循环 |
| **KDA-aware 前缀缓存** | 双粒度哈希、稀疏 checkpoint、跨组原子失效——运行时动态内存管理 |
| **MX 格式支持** | XLA:TPU 无原生 MXFP4 / MXFP8 dtype |
| **多流重叠的替代方案** | K3 用 CUDA side stream 重叠共享专家 GEMM 与 AttnRes inter-block；TPU core 单指令流，只能靠 XLA 内的 async collective 与 fusion |
| **变长视觉 CP** | 见反例章节 |
| **Muon 的 P2P shard 聚集** | XLA 无"只拉我拥有的 shard"原语，要用 `ppermute` 手工编排 |
| **负载感知的 grouped matmul 调度** | rank 内逐专家仍 ragged（见上文），K3 用 workload-aware GEMM 调度器；TPU 侧要在 Megablox 上做等价改造 |

<span class="evid evid-guess">推断</span> 好消息是 TPU 这一侧有三个结构性优势可以利用：

1. **附录 E 的单源引理**——均衡计划下每个 rank 的远程 token 只来自一个 rank，通信是近似置换。3D Torus 对成对交换的映射远好于稠密 A2A。（限定：这是存在性证明的构造，线上规划器只是 near-optimal。）
2. **SparseCore** 本来就是为细粒度 DMA 和独立控制线程设计的，规划内核与直方图 `scatter_add` 都可以 offload。
3. **Dual-chiplet 的 D2D 带宽约 8 Tb/s**（6× 单 ICI link），把同 chip 的两个 chiplet 分给相邻 expert group，可以让一部分 dispatch 走片内高带宽。

坏消息是这三条都还没有人验证过。

---

## 风险和不确定性（改版重写）

| # | 风险 | 状态 | 说明 |
|---|------|------|------|
| 1 | Quantile 分位数的计算效率 | ⚠️ **算法已明，TPU 效率待验** | K3 用 `B = 1000` 的直方图 + 一次 `n×B` 整数 all-reduce，误差 ≤ bin 宽（几个 10⁻³）。但 XLA 上 896 维的 `top_k(17)` 与 `scatter_add` 的效率仍待验证 |
| 2 | SparseCore MoE offload API 成熟度 | ⚠️ 保留 | 公开文档覆盖的是 embedding / collective 场景，MoE dispatch–combine 的 offload 接口与第三方实测均未见公开 |
| 3 | AttnRes 内存开销 | ✅ **已被解掉** | Block AttnRes + 整体 checkpointing，每层保存的激活与标准残差相同 |
| 4 | QB 的质量影响 | ⚠️ 保留 + 补充 | 报告**没有给 QB 的单项消融** |
| 5 | KDA kernel 调优 | ⚠️ **有实测，但只覆盖一种 regime** | 见第二层。解码与 intra-device CP 两种 regime 在 TPU 上仍空白 |
| 6 | **MoonEP 的 TPU 等价物** | 🔴 **新增** | 移植路径里真正的大头 |
| 7 | **MX 量化格式** | 🔴 **新增** | 开放权重是 MXFP4/MXFP8，XLA:TPU 无原生 dtype |
| 8 | **多模态变长动态性** | 🔴 **新增** | 见反例章节 |
| 9 | **2.5× scaling efficiency 不可逐项归因** | 🔴 **新增** | 见下 |

> 说明：上表的"风险"与前面盘点表的"判定"是两个维度——一项创新可以判定为"收益相当"，同时在移植时仍带来风险。

<span class="evid evid-hard">报告实锤</span> 关于最后一条：报告称 K3 相对 K2 有约 **2.5× 的整体 scaling efficiency 提升**，但明确写的是 *"KDA, AttnRes, Stable LatentMoE, refined data and training recipes **collectively** improve..."* ——**共同作用，没有逐项消融**。

而且这个数字的含义是"达到相同验证损失所需算力更少"，**不等于**训练总成本降到 40%，也**不等于**推理快 2.5×。K3 总参数 1.04T → 2.78T、激活 32.6B → 104.2B、层数 61 → 93，它仍然是一个更重、更贵的模型。

---

## 结语

K3 和 TPU 的契合不是刻意设计的结果，而是两种"极致确定性"追求的部分汇合。

TPU 从第一天起就说："告诉我所有形状，我给你最优编译。"MoE 从第一天起就说："我需要动态路由的灵活性。"六年来，这对矛盾催生了 capacity_factor、辅助 loss、动态 All-to-All 等一系列精巧但笨重的妥协。

K3 走了一条不同的路。**但它问的问题比"消灭动态性"更克制**——不是"路由能不能不动态"（K3 的路由仍然是 Top-k），而是**"负载不均是不是必须被容忍"**。

它的答案分两层，而且分工很清楚：**系统层**的 MoonEP 用一个对任意路由输出都成立的上界，把负载不均变成不可能发生，换来每 rank 精确相同的 token 数、固定的通信缓冲区、编译期已知的通信形状；**算法层**的 Quantile Balancing 把偏置从"定步长试探"换成"对偶目标的闭式解"，让上面那件事的成本降下来，顺便让 896 个专家都能被训到。

这两层里，系统层那一层正是 TPU/XLA 六年来一直在等的那个答案——**但它写在 CUDA 和 NCCL 上，要自己动手搬一遍。**（若论单项收益，最大的仍是 KDA 的常数 state；这两件事量的是不同的坐标轴，见上文盘点表前的说明。）而在多模态那一侧，连 K3 自己也没有把动态性交还出去。

所以，改版之后我更愿意这样说：

**良缘是真的。彩礼也是真的。而且不是所有的门都通着。**

---

*首发 2026-07-18（纯猜想）· 改版 2026-07-29（基于技术报告、开放权重、Tokamax PR 三个一手数据源重写，并撤回首发版四个错误数值与四处论述性错误）。改版后仍有标注为「推断」的内容未经工程验证，欢迎指正。*

## 附录：改版记录 {#附录改版记录}

首发版 2026-07-18 与改版 2026-07-29 的逐条差异。**没读过首发版的读者可以跳过这一节。**

| 改动 | 首发版本 | 改版后 |
|------|---------|--------|
| **第一重 Quantile Balancing** | "替代 Top-K 路由，每个 expert **恰好**收到 N/E 个 token，彻底消灭矛盾" | ❌ **实质错误。** QB 就是 Top-K + 偏置；报告给出四个限定条件说明它做不到"恰好"。静态形状实际来自 **MoonEP**，且只到 **rank** 粒度 |
| **第二重 静态 All-to-All** | 推断：消除 Host-Device 同步，解锁 XLA 全图编译 | ✅ **被报告逐字印证**，但机制归因由 QB 改为 MoonEP |
| **第四重 SiTU** | "待定，公式未披露，输出约 [-0.2, 1]" | ✅ 完整公式已披露。方向猜对，**数值写错**（真实界 \|f\| ≤ 100）。同时补上一条削弱：MX 格式 XLA 无原生 dtype |
| **第五重 Per-Head Muon** | "128 个 head、`(16384,16384)` 降为 `(128,128)`、计算量降低 **128×**" | ❌ **三个数字都是首发版编的。** 报告 §2.5 没有任何数字，`config.json` 是 **96** 个 head。**动机也写反了**——报告说是均衡各头更新尺度、提升稳定性，降开销只是 "slightly" |
| **第七重 KDA** | 3:1 混合 + KV cache 压缩 | ➕ 新增下界衰减 `g_min = −5`（最强支持证据）；➕ 补三条削弱：投机解码的状态回滚、前缀缓存复杂化、KDA 三种 regime 只覆盖一种 |
| **第八重** | 无 | ➕ **原文遗漏了一整根架构支柱**：Stable LatentMoE（宽度维度） |
| **反例章节** | 无 | ➕ **原生多模态**——K3 里唯一交不回编译期的动态性。本文主张必须加限定语 |
| **移植路径** | "素描"，四个 Phase | ⬆️ 升级为**分层成本估算**；并给出 TPU 侧三条结构性利好（附录 E 单源引理 / SparseCore / chiplet D2D） |
| **收益盘点** | 3 项 TPU 远大于 GPU / 1 放大 / 1 待定 / 2 相当 | ⬇️ **诚实降级**：1 项收益最大但需自建 paged 替代 / 1 项需自建 MoonEP 等价物 / 0 待定 / 其余相当或半相当，外加 1 项反例 |

### 参考资料

- [Kimi K3 技术报告](https://github.com/MoonshotAI/Kimi-K3/blob/main/k3_tech_report.pdf) — 47 页 PDF，本次改版的主要依据，全文引用均出自此处
- [Kimi K3 开放权重](https://huggingface.co/moonshotai/Kimi-K3) — `config.json` 是所有架构参数的一手来源
- [Tokamax PR #1103: KDA Pallas Kernel](https://github.com/openxla/tokamax/pull/1103) — 目前公开可见的 KDA 算子 TPU 实现（仅覆盖训练/prefill 一种 regime），含 CP 实测
- [MoonEP](https://github.com/MoonshotAI/MoonEP) — K3 的 EP 调度方案。本文对它的判断来自技术报告 §5.2.1 与附录 E，未逐行读过该仓库代码
- [OpenXLA SparseCore 深度指南](https://openxla.org/xla/sparsecore) — SparseCore 架构与 Collective Offloading 官方文档
- [Google Cloud TPU v7 (Ironwood) 文档](https://cloud.google.com/tpu/docs/ironwood-performance) — 性能优化与硬件规格
- [GShard: Scaling Giant Models with Conditional Computation](https://arxiv.org/abs/2006.16668) — capacity_factor 的起源
- [Megablox GMM Pallas Kernel](https://github.com/jax-ml/jax/blob/main/jax/experimental/pallas/ops/tpu/megablox/gmm.py) — TPU 上 MoE ragged batch 的 block-sparse 实现
- [SemiAnalysis: TPU v7 分析](https://newsletter.semianalysis.com/p/tpuv7-google-takes-a-swing-at-the) — SparseCore SCS/SCT 架构细节
- [Kimi Linear: An Expressive, Efficient Attention Architecture](https://arxiv.org/abs/2510.26692) — KDA 的前身，K3 下界衰减改动的对照基线
- [DeepSeek-V3](https://arxiv.org/abs/2412.19437) — aux-loss-free 路由的来源，K3 的 Quantile Balancing 与它同属一族
- [JAX Pallas TPU paged attention](https://github.com/jax-ml/jax/tree/main/jax/experimental/pallas/ops/tpu/paged_attention) — 用于更正首发版"TPU 做不到 PagedAttention"的说法

*2026-07-18 首发 · 2026-07-29 改版 · [blog.higcp.com](https://blog.higcp.com)*
