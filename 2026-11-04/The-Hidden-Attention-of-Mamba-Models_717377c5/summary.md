---
title: "The-Hidden-Attention-of-Mamba-Models"
source: https://aclanthology.org/2025.acl-long.76.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 08:18:26"
field: "可解释 AI / 状态空间模型"
keywords: ["Mamba", "Selective State-Space Model", "Explainable AI", "Hidden Attention", "Vision Mamba", "In-context Learning"]
innovations: ["将 S6 层重写成隐式因果线性注意力形式，揭示数据控制非对角混合器", "构建首个 Mamba XAI 工具集（Mamba-Attention-Rollout 与 Mamba-Attribution）", "证明 S6 表达力强于 causal linear attention，并提出 count-in-row 反例法"]
benchmarks: ["ImageNet-1K", "ImageNet-Segmentation", "IMDb", "ARC-Easy"]
---

# 论文速读：The-Hidden-Attention-of-Mamba-Models

## 一句话总结
论文揭示了 Mamba（S6 层）可以形式化为一种隐性的因果线性注意力机制，通过引入"数据控制线性算子"提取出隐含注意力矩阵；在此基础上构建了首个适用于 Mamba 模型的 XAI（可解释 AI）工具集（Attention Rollout 和 Attribution），并在视觉与 NLP 任务上验证了其可解释性与 Transformer 相近甚至更优。

## 研究问题与动机
- **核心问题**：Mamba/SSM 的内部信息流动态与学习机制尚不清楚——它们究竟如何捕捉依赖关系？与 RNN、CNN、Transformer 有何异同？
- **可解释性缺失**：当前 S6 模型缺乏像 Transformer 那样成熟的 XAI 工具和调试手段，限制了其在高可靠性场景的应用。
- **理论空白**：尽管 S6 在效率上优势明显，但其注意力机制的本质属性（expressiveness、ICL 能力等）尚未得到系统性刻画。
- **跨架构对照需求**：现有研究很少从统一的"注意力视角"将 Transformer 与 Mamba 进行可对比分析。

## 核心贡献（创新点）
1. **提出 Mamba 的注意力视角**：证明 S6 层可被重写成隐式因果自注意力形式（Eq. 13–14），揭示了一个数据控制的线性非对角混合器（data-controlled non-diagonal mixer）。
   - 区别于以往仅将其视为卷积或 RNN 的观点，首次建立 Mamba 与 Transformer 在注意力层面的理论桥梁。
2. **发现 Mamba 生成约 3 个数量级于 Transformer 的注意力矩阵**：一个 S6 channel 产生 N 个内层 attention matrix（N 为状态维度），叠加 D 个 channel 后总量为 DN，远超 Transformer 的 H 个 head。
   - 这为细粒度可解释性分析提供了更丰富的信息源，也解释了 Mamba 强大的上下文建模潜力。
3. **构建首个 Mamba XAI 工具集**：设计了 Mamba-Attention-Rollout（类无关）和 Mamba-Attribution（类特定）两种解释方法，并给出相应公式与实现细节。
   - 与纯 Transformer 方法不同，Mamba-Attribution 利用 S6 mixer 梯度与门控机制梯度联合，而非仅依赖 attention matrix 梯度。
4. **系统性实验验证 XAI 性能**：在 ImageNet（视觉分割/扰动）、IMDb（NLP 激活/剪枝）、ARC-Easy（零样本推理）等多个基准上评估，结果与 SoTA Transformer XAI 相当甚至更强。
   - 这是首批将 Transformer XAI 范式直接迁移到 Mamba 的成功实践。
5. **理论分析表达力与 ICL 能力**：证明 S6 比 causal linear attention 更具表达力；提出 S6 的 data-controlled non-diagonal mixer 是实现 with ICL 能力的关键因素。

## 方法详解
### 1. S6 层的注意力形式化（Eq. 10–16）
- 从 S6 递推公式 $h_t = \bar{A}_t h_{t-1} + \bar{B}_t x_t$ 出发，展开得到输出 $y_t = C_t \sum_{j=1}^{t} (\prod_{k=j+1}^{t} \bar{A}_k) \bar{B}_j x_j$。
- 写成矩阵形式 $y = \tilde{\alpha} x$，其中 $\tilde{\alpha}_{i,j} = C_i (\prod_{k=j+1}^{i} \bar{A}_k) \bar{B}_j$（Eq. 14），即为**隐式注意力矩阵**。
- 由于 $\bar{A}$ 是对角矩阵，每个 channel 内部 N 个状态维度独立演化，等价于产生 N 个内层 attention matrix 并求和。

### 2. 简化的 Q/K/H 表示（Eq. 17–20）
- 近似 Softplus 为 ReLU（记为 R），定义：
  - $\tilde{Q}_i = S_C(\hat{x}_i)$（Query）
  - $\tilde{K}_j = R(S_\Delta(\hat{x}_j)) S_B(\hat{x}_j)$（Key）
  - $\tilde{H}_{i,j} = \exp(\sum_{k=j+1}^{i} S_\Delta(\hat{x}_k) A)$（历史注意力核）
- 则 $\tilde{\alpha}_{i,j} \approx \tilde{Q}_i \tilde{H}_{i,j} \tilde{K}_j$（Eq. 20），清晰呈现"类 Attention"结构。
- $\tilde{H}_{i,j}$ 是关键创新项：它编码了从位置 j 到 i 的**连续累积历史上下文**，区别于 Transformer 的 pairwise dot-product。

### 3. Mamba-Attention-Rollout（类无关，Eq. 21–22）
- 对每个 channel d 和 layer λ 计算隐式注意力矩阵 $\tilde{\alpha}^{\lambda,d}$。
- 聚合：$\tilde{\alpha}^\lambda = \mathbb{I} + \mathbb{E}_{d}[\tilde{\alpha}^{\lambda,d}]$（含 skip connection）。
- 逐层连乘得最终 relevance map $\rho = \prod_\lambda \tilde{\alpha}^\lambda$，取 CLS token 所在行得到图像/序列的热图。

### 4. Mamba-Attribution（类特定，Eq. 24）
- 借鉴 Transformer-Attribution（Chefer et al., 2021b），但针对 Mamba 结构调整：
  - 梯度来源：从 $\nabla \tilde{\alpha}$ 改为 $\nabla \hat{y}'^{\lambda,d}$（利用 S6 mixer + 门控机制的双重梯度）。
  - 相关性矩阵：用 $\tilde{\alpha}^{\lambda,d}$ 替代 LRP 分数 $R^{\lambda,\hat{h}}$。
- 公式：$\tilde{\beta}^\lambda = \mathbb{I} + (\mathbb{E}_d[\nabla \hat{y}'^{\lambda,d}] \odot \mathbb{E}_d[\tilde{\alpha}^{\lambda,d}])^+$。
- 经 ablation 验证，clamping 负值为零是最优策略（Appendix D Table 6）。

### 5. 表达力理论分析
- **Theorem 1**：S4/DSS/S5 为 fixed mixing；GSS/Hyena 为 fixed + diagonal data-control；S6 为 data-controlled non-diagonal mixer（首次具备此性质）。
- **Theorem 2**：单个 S6 channel 可表达所有单 attention head 能表达的功能；反之不成立（证明通过 "count-in-row" 任务）。
- 假设：忽略 discretization、hidden dim=1（标量参数）。

## 实验与结果
### 视觉任务（ImageNet / ViM vs ViT）
- **数据集**：ImageNet-1K（分类）+ ImageNet-Segmentation（分割评测）。
- **模型**：ViM-Tiny（D=384, bidirectional）vs DeiT-Tiny（H=3 attention heads）。
- **定性可视化**（Figure 3/4）：Mamba 的 attention maps 与 Transformer 相似，深层捕捉远距离依赖；Mamba Raw-Attention 在整个图像范围有活动但正确目标被高亮。
- **扰动测试**（Table 1）：
  - Positive perturbation（越低越好）：Raw-Attn Mamba 17.27 < ViT 20.69；Attn-Rollout Mamba 18.81 < ViT 20.60；Attribution Mamba 16.62 vs ViT 15.35（ViT 略优）。
  - Negative perturbation（越高越好）：Transformer 全面领先，Mamba 对 patch 遮蔽更敏感。
- **分割测试**（Table 2，越高越好）：
  - Raw-Attn：Mamba Pix-acc 67.64 > ViT 59.69，mIoU 45.09 > 36.94，但 mAP 74.88 < 77.25。
  - Attn-Rollout：Mamba 全面领先（Pix-acc 71.01, mAP 80.78, mIoU 51.51）。
  - Attribution：Mamba-Attribution (Ours) Pix-acc 74.72, mAP 81.70, mIoU 54.24，优于 Mamba-Attr (LRP) 和 ViT-Attribution（Pix-acc 79.26 但 mAP 84.85 更高）。
  - **最强结果**：Transformer-Attribution 在所有指标上最高（Pix-acc 79.26, mAP 84.85, mIoU 60.63）；Mamba-Attribution (Ours) 次之但仍显著优于 LRP baseline。

### NLP 任务
- **IMDb 情感分类**（Appendix A, Figure 5/Table 4）：Mamba-Attr 在激活/剪枝任务上与 Transformer-Attr 相当或更优，解释更稀疏（更少噪声词）。
- **ARC-Easy 零样本推理**（Table 3）：
  - Mamba 1.3B: Positive AUAC 0.915, Negative AU-MSE 1.765；Pythia 1.4B: 0.909 / 1.832。
  - Mamba 2.7B: 0.918 / 1.239；Pythia 2.8B: 0.920 / 1.255。
  - **结论**：Mamba XAI 性能与 SoTA Transformer XAI 基本持平，在 larger model 规模下差距极小。

### 消融实验（Appendix D/G）
- 聚合方式：Mean Head Aggregation 最优（Table 5）。
- Clamping 负值：保留 clamp 策略最佳（Table 6）。
- XAI 辅助提示工程（AMPLIFY framework，Table 7）：Mamba XAI 驱动的 few-shot prompt 自动增强，在 Snarks (+8.6%), CommonsenseQA (+1.4%), Formal Fallacies (+4.2%) 上均有提升。

## 相关工作脉络
1. **Gu & Dao (2023) Mamba / S6**：本文直接在此基础上提出注意力视角的重构，填补其可解释性空白；此前 S6 仅有训练/推理效率优势论述。
2. **Vaswani et al. (2017) Transformer**：作为对比基线；本文证明 Mamba 隐式 attention 与 Transformer 在可解释性层面具有可比性。
3. **Abnar & Zuidema (2020) Attention-Rollout**：本文将其移植到 Mamba（Eq. 21–22），是第一个跨架构迁移的 XAI 方法。
4. **Chefer et al. (2021b) Transformer-Attribution**：本文的 Mamba-Attribution（Eq. 24）是其核心灵感来源，但做了针对性修改（利用 S6 mixer 梯度）。
5. **Poli et al. (2023) Hyena / SSM 理论**：本文引用其 "data-controlled linear operator" 概念，并将其细化为 data-controlled **non-diagonal** mixer 的定义，用于 Theorem 1 的分类。
6. **Zimerman & Wolf (2024) / Zimerman et al. (2025)**：相关工作脉络中提及的前期表达力分析与通用 gated-linear RNN 解释框架，本文为 S6 提供了更具体的 formalization。
7. **Dao & Gu (2024) Transformers are SSMS**：后续工作（Appendix H）从 state-space duality 角度独立发展了类似联系，本文是最早的 S6→causal linear attention 的形式化。

## 局限性与未来方向
- **计算成本**：生成 attention map 需要构造 per-channel 的 $L \times L$ 矩阵（$O(\Lambda D L^2 N)$ 朴素，优化后可到 $O(\Lambda D L N)$），对长序列仍昂贵。
- **模型规模限制**：仅在 ViM-Tiny、Mamba-130M/1.3B/2.7B、Pythia 1.4B/2.8B 等中等规模模型上验证，**未测试** LLaMA-405B、GPT-4 等超大规模模型（论文发表时此类 Mamba 模型尚不可用）。
- **负扰动敏感性**：Mamba 对 patch 遮蔽更敏感（Negative perturbation AUC 较低），未来需探索 blur 等更平滑的扰动策略。
- **XAI 方法可调性**：Mamba-Attribution 虽表现良好，但作者承认其效果可能仍需进一步调优（"may benefit from further adjustments"）。
- **未来方向**：设计更高效 XAI 方法，利用 Mamba 内在的 linear attention 结构**显式计算** attention map，而非通过二次复杂度矩阵。

## 研究启发与可借鉴点
1. **S6→Attention 的形式化框架可直接复用**：本文 Eq. 13–20 的推导范式为任何其他 SSM 变体（Mamba-2、Hyena、GSS 等）提取隐式 attention matrix 提供了模板，可迁移到不同架构的可解释性研究。
2. **Mamba-Attribution 的梯度融合设计值得借鉴**：利用 S6 mixer 梯度与门控机制梯度联合（而非仅 attention 梯度）的策略，为其他包含复杂门控/非线性结构的序列模型 XAI 提供了新思路。
3. **CLS token 位置对注意力的影响**（Figure 2）：发现 CLS 靠近序列中部时 attention 分布更均匀，这对 ViM 类视觉模型的 token 设计有直接启示。
4. **XAI 辅助 prompt 工程**（Table 7, AMPLIFY）：证明了 Mamba XAI 不仅能解释模型，还能**主动提升模型性能**（few-shot prompt 优化 +8%~10%），为后续 "XAI-driven model improvement" 提供了可行路径。
5. **表达力理论工具**：Theorem 2 的 "count-in-row" 反例法可作为衡量 SSM 与 Attention 表达能力差距的通用测试范式。

## 关键术语表
- **S6 / Selective SSM**：Mamba 的核心层，时间相关的选择性状态空间层，输入决定离散化参数 $\Delta, B, C$。
- **隐式注意力矩阵（Hidden Attention Matrix）$\tilde{\alpha}$**：S6 层经重排后等价的 $L \times L$ 矩阵，元素 $\tilde{\alpha}_{i,j}$ 表征 $x_j$ 对 $y_i$ 的影响。
- **数据控制线性非对角混合器（Data-Controlled Non-Diagonal Mixer）**：S6 区别于 prior SSMs 的核心特征，其 token 间交互模式由输入数据动态决定且非对角。
- **$\tilde{H}_{i,j}$（历史注意力核）**：Mamba attention 的关键项，指数累积从 j 到 i 的 $\Delta$ 加权状态转移，编码连续历史上下文。
- **Attention-Rollout**：通过层间 attention matrix 连乘聚合路径重要性的 XAI 方法，本文首次适配到 Mamba。
- **Mamba-Attribution**：本文提出的类特定 XAI 方法，融合 S6 mixer 梯度与隐式 attention 矩阵，对标 Transformer-Attribution。
- **In-Context Learning (ICL)**：模型根据输入上下文即时适应新任务的能力；本文假设 S6 的 data-controlled non-diagonal mixer 是其实现 ICL 的关键。
- **IO-aware Parallel Scan**：Mamba 高效训练的核心算法，将 S6 的因果递推转化为可并行计算的 associative scan。

## 可复现要素
- **数据集**：ImageNet-1K（公开）、ImageNet-Segmentation（公开）、IMDb（公开）、The Pile（公开）、ARC-Easy（公开）。
- **代码/权重**：论文声明 "Our code is publicly available"（注释1），部分实验代码随 supplementary material 提供，含 user-friendly interface 和 notebook demos；使用 PyTorch 框架。
- **关键超参**：论文未系统罗列（主要复现已有 Mamba/ViM 默认配置）；XAI 方法关键超参包括：聚合方式（mean，最优）、clamp 负值（保留 zero-clamp，最优）、丢弃率 discard=0.2（最优消融）。
- **模型规模**：视觉侧 ViM-Tiny (D=384) vs DeiT-Tiny (H=3)；NLP 侧 Mamba-130M/1.3B/2.7B vs Pythia-160M/1.4B/2.8B。
