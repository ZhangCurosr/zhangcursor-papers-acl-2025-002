---
title: "Position-aware-Automatic-Circuit-Discovery"
source: https://aclanthology.org/2025.acl-long.141.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 00:59:24"
field: "可解释人工智能"
keywords: ["mechanistic interpretability", "circuit discovery", "edge attribution patching", "position-aware", "schema", "LLM"]
innovations: ["提出位置感知边缘归因补丁(PEAP)，支持跨位置注意力边的发现与独立评估", "引入数据集schema概念，通过语义对齐实现变长输入的位置感知电路发现", "开发基于LLM和saliency mask的自动化schema生成与应用管道"]
benchmarks: ["IOI (Indirect Object Identification)", "Greater-Than", "Winobias"]
---

# 论文速读：Position-aware-Automatic-Circuit-Discovery

## 一句话总结
本文提出了一种**位置感知的自动电路发现方法**（PEAP），通过区分不同token位置的注意力边重要性，解决了现有电路发现方法忽略位置信息的局限；同时引入**数据集schema**概念和基于LLM的自动化管道，实现了对变长输入的跨位置电路发现，在保持电路紧凑性的同时显著提升了保真度。

## 研究问题与动机
- **核心问题**：现有自动电路发现方法假设电路是位置不变的（position-invariant），将模型组件视为在所有输入位置上同等重要，无法捕捉跨位置交互或随位置变化的机制。
- **现有方法的不足**：
  - **跨位置抵消效应（Cancellation）**：将不同位置的归因分数简单相加时，正负分数相互抵消，导致重要性接近零，造成召回率低（Figure 2左）。
  - **重要性高估（Overestimation）**：不考虑位置的电路倾向于选择在许多位置都有小影响的边，而非在少数位置有大影响的边，造成精度高但召回率低（Figure 2右）。
  - **变长输入困难**：真实数据集输入长度不一，同一位置在不同示例中可能对应不同的语义角色，难以直接对齐。

## 核心贡献（创新点）
1. **位置感知边缘归因补丁（PEAP）**：扩展了边缘归因补丁方法，能够分别评估每个token位置的边重要性，并支持跨位置注意力边的发现，本质区别在于不再将跨位置边聚合后决策，而是逐位置独立评估。
2. **数据集Schema概念**：提出用schema定义跨示例的语义token span，使变长输入中的相同语义角色能够被映射到统一的抽象计算图上，本质区别在于突破了严格token对齐的限制，允许语义层面的位置对应。
3. **自动化LLM Schema生成与应用管道**：利用大语言模型自动生成并应用schema，引入 saliency mask（基于input×gradient）辅助生成更贴近模型计算的schema，达到了与人工设计schema相当的保真度，本质区别在于无需人工标注即可处理复杂数据集。
4. **系统性验证与对比分析**：在IOI、Greater-Than、Winobias三个任务上系统比较了位置感知与非位置感知电路，展示了PEAP在电路大小-保真度权衡上的优势。

## 方法详解

### 1. PEAP：位置感知边缘归因补丁
- **基础框架**：沿用Edge Attribution Patching（EAP）的线性近似，对边 $e=(u,v)$ 的间接效应（IE）估计为：
  $$g(e) = M(x|e=e_{x'}) - M(x) \approx (z_u^* - z_u)^\top \nabla_v M(x)$$
  其中 $M$ 为目标指标（通常是正确与错误补全的logit差），$z_u$ 和 $z_u^*$ 分别为干净和反事实激活。
- **位置扩展**：对Transformer中的注意力机制，将每个注意力头 $h_{t,l}^i$ 的输出分解为query、key、value向量，定义**跨位置边**：头 $h_t^i$ 与所有位置 $t' \leq t$ 的头 $h_{t',l}^i$ 之间通过三条边连接（$v_{t',l}^i, k_{t',l}^i, q_{t,l}^i$）。
- **补丁计算**：分别对 $v$、$k$、$q$ 进行patching，计算 corrupted output $z_{h_t^i}^*$，然后近似归因分数（公式4-6）。
- **电路构建**：采用改进的贪心算法（Hanna et al., 2024b），从最终位置的logits节点出发，逐步添加最高归因分数的边及其父节点，确保图的连通性。

### 2. Schema：变长输入的位置对齐
- **定义**：Schema $S$ 将输入prompt划分为有序的非重叠span集合，每个span赋予语义标签（如Subject、Verb等）。
- **抽象计算图**：将schema映射为长度为 $k$（span数量）的抽象计算图 $G_S$，每个span对应一个位置。
- **边映射**：定义映射 $f_S^x: E_S \to 2^{E_x}$，将抽象图中的边映射回原始计算图中的实际边集合。
- **聚合公式**：抽象图的边归因分数为：
  $$g_S(e) = \frac{1}{|\mathcal{D}|} \sum_{x \in \mathcal{D}} \sum_{e' \in f_S^x(e)} g_x(e')$$
- **电路评估**：将抽象电路 $\mathcal{C}_S$ 反向映射到每个示例的完整计算图 $\mathcal{C}_x$ 进行评估。

### 3. 自动化Schema生成与LLM管道
- **Schema生成流程**：
  1. 从数据集中采样3组示例，每组让LLM生成一个候选schema。
  2. LLM综合3个候选schema生成统一schema。
  3. 验证schema在≥80%的示例上可有效应用。
- **Saliency Mask增强**：
  - 使用 input×gradient 方法计算token重要性得分：$s(t) = \|\mathbf{e}_t \cdot \nabla_{\mathbf{e}_t} M(x)\|$。
  - 对得分做softmax并设定阈值（$> 1/n$）生成二进制mask。
  - 将mask作为额外信息提供给LLM，指导其将重要token单独划分为span。
- **Schema应用**：使用Claude 3.5 Sonnet将schema应用到每个示例，要求span顺序一致、覆盖所有token。

## 实验与结果

### 数据集与模型
- **模型**：GPT2-small、Llama-3-8B
- **任务**：
  - **IOI**（Indirect Object Identification）：间接宾语识别，含ABBA和BABA两种模板。
  - **Greater-Than**：年份大小比较任务，使用单一模板。
  - **Winobias**：性别偏见核心解析任务，包含Anti-Female/I、II等多个子集。
- 每个任务均匀采样500个示例用于电路发现，500个用于保真度评估。

### 评估指标
- **Soft Faithfulness**：$F_S(C) = \frac{M(C)}{M(\mathcal{M})}$，电路性能与完整模型的比值。
- **Hard Faithfulness**：$F_H(C) = \mathbb{1}\{C_T = \mathcal{M}_T\}$，最终位置token预测一致性。

### 主要结果
- **位置感知电路显著优于非位置感知电路**：在三个任务上，PEAP发现的电路在更小的电路规模下达到更高的保真度（Figure 6）。
- **Greater-Than任务**：LLM+Mask schema达到与人工schema相当甚至更高的保真度，尤其在较大电路规模下。
- **IOI任务（GPT2-small）**：自动化管道发现的电路与人工电路保真度相当。
- **IOI任务（Llama-3-8B）**：LLM生成电路略优于人工电路，说明需针对特定模型定制schema。
- **Winobias任务**：Saliency mask的引入显著提升了保真度，使其与人工schema相当。
- **Schema质量**：LLM生成的schema在有效性（Validity）和正确性（Correctness）上表现良好（Table 6），大多数示例可成功应用。

### 取消与高估效应验证（Table 1）
- 在IOI任务上，Top 1%边中，两种聚合方式有17.1%的差异，证明忽略位置会导致显著排名变化。
- 控制实验（同一聚合方式在不同子集上的差异）仅为3.9%，证明效应显著。

## 相关工作脉络
- **Conmy et al. (2023)**：提出的Automated Circuit Discovery方法，但仅发现不区分位置的电路；本文扩展至位置感知。
- **Syed et al. (2023)**：Edge Attribution Patching（EAP）基础方法；本文在其基础上扩展支持跨位置注意力边。
- **Wang et al. (2023)**：IOI电路的手动发现；本文提供自动化替代方案，且能达到相似或更优的保真度。
- **Hanna et al. (2024a, 2024b)**：Greater-Than电路和Faithfulness评估方法；本文直接使用其任务和评估框架，但引入位置感知。
- **Marks et al. (2025)**：Sparse Feature Circuits方法，也考虑位置，但仅限于完全对齐的模板数据集，且边定义不同；本文处理变长输入并定义真正的计算图边。
- **Kramár et al. (2024)**：AtP*方法，考虑位置但仅在节点级别，未用于电路发现；本文扩展到边级别和变长输入。

## 局限性与未来方向
- **Schema设计的通用原则缺失**：目前没有明确的标准来确定哪些schema能在保真度和电路大小之间取得最佳权衡，需要开发通用的schema设计原则。
- **Span顺序约束**：Schema要求所有示例中span顺序一致，不同顺序的schema无法联合评估，限制了灵活性。
- **人工schema与模型schema的差异**：对于不同模型（如GPT2-small vs Llama-3-8B），最优schema可能不同，需要针对模型-任务组合定制。
- **Counterfactual定义的敏感性**：如Winobias任务所示，不同的counterfactual定义会定位到不同的子电路，需要更系统的研究。
- **LLM生成的不确定性**：虽然Saliency mask有所改善，但LLM生成的schema仍可能存在一定随机性，需要更多run取最优。

## 研究启发与可借鉴点
1. **位置感知的电路发现范式**：将归因方法从"位置不变"扩展到"位置敏感"，这一思路可迁移到其他因果可解释性方法（如DAS、path patching），值得系统性探索。
2. **Saliency Mask辅助的自动标注**：利用input×gradient等简单归因方法生成mask，指导LLM生成更符合模型内部机制的schema，是一种高效的人机协同策略，可应用于其他需要结构化输入的任务。
3. **变长输入的抽象图构建**：Schema方法通过语义对齐而非token对齐来聚合跨位置信息，这一思想可扩展到其他需要跨样本聚合的可解释性方法。
4. **自动化管道的可行性验证**：证明LLM可以生成与人工专家相当质量的schema，为其他需要领域知识的可解释性任务提供了自动化解决方案的参考。
5. **双指标评估体系**：同时使用Soft和Hard Faithfulness，后者更能反映行为层面的忠实度，这一评估设计值得借鉴。

## 关键术语表
- **Circuit（电路）**：语言模型计算图中执行特定任务的最小子图，节点代表模型组件（如attention head、MLP），边代表组件间的连接。
- **Edge Attribution Patching（EAP，边缘归因补丁）**：一种通过线性近似估计边对目标指标间接效应（IE）的方法，用于自动电路发现。
- **Position-aware（位置感知）**：指考虑不同token位置可能涉及不同计算机制的方法，与position-agnostic（位置无关）相对。
- **Schema（模式）**：将输入prompt划分为有序语义span的框架，使变长输入能够在语义层面进行位置对齐。
- **Faithfulness（保真度）**：衡量电路对模型行为的忠实程度，分为Soft（logit比值）和Hard（token预测一致性）两种。
- **Cancellation Effect（抵消效应）**：跨位置归因分数正负相抵导致重要性被低估的现象，影响电路发现的召回率。
- **Overestimation Effect（高估效应）**：忽略位置时，小影响但广分布的边被错误地认为比少数位置的大影响边更重要。
- **Saliency Mask（显著性掩码）**：基于input×gradient计算token重要性，用于指导LLM生成更贴合模型计算的schema。

## 可复现要素
- **数据集**：IOI（使用Wang et al., 2023的生成代码）、Greater-Than（Hanna et al., 2024a）、Winobias（Zhao et al., 2018的模板）；数据是否公开需查看原仓库。
- **代码**：使用TransformerLens库（Nanda & Bloom, 2022）；论文未明确声明代码开源，但通常此类研究会提供GitHub链接。
- **模型**：GPT2-small（公开）、Llama-3-8B（公开权重）。
- **LLM**：Claude 3.5 Sonnet（用于schema生成和应用）；论文未提及具体API调用细节。
- **关键超参**：电路大小N（贪心算法步数）、mask阈值（1/n）、validity阈值（≥90%）。
- **示例数量**：每个任务500个示例用于发现，500个用于评估。
