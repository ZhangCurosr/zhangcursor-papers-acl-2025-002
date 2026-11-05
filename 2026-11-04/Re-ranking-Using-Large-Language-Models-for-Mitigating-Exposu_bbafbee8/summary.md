---
title: "Re-ranking-Using-Large-Language-Models-for-Mitigating-Exposu"
source: https://aclanthology.org/2025.acl-long.44.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 01:00:02"
field: "社交内容安全与推荐系统"
keywords: ["re-ranking", "large language models", "harmful content mitigation", "in-context learning", "recommendation systems"]
innovations: ["提出基于LLM的零样本/少样本成对重排序方法减少有害内容暴露", "设计PPk和EWN两个序列级有害内容暴露评估指标"]
benchmarks: ["YouTube Harmful Videos Dataset", "Jigsaw Toxicity Dataset", "D-Lab Hate Speech Dataset"]
---

# 论文速读：Re-ranking-Using-Large-Language-Models-for-Mitigating-Exposu

## 一句话总结
本文提出一种基于大语言模型（LLM）的零样本/少样本重排序方法，通过将推荐序列中的有害内容降级到末尾来减少用户在社交媒体上的有害内容暴露。该方法无需显式训练即可泛化到多种有害类型，并在多个数据集和模型上显著优于 Perspective API 和 OpenAI Moderation API 等传统分类器基线。

## 研究问题与动机
- 社交媒体推荐算法为最大化用户参与度，可能间接导致用户暴露于有害内容（如饮食失调、自杀相关内容、虚假信息、点击诱饵等）。
- 传统内容审核方法依赖需要大量人工标注数据训练的 classifier，难以适应不断涌现的新形态有害内容，且面临概念漂移问题。
- 现有分类器方法无法自动泛化到未见过的新有害类型，必须通过人工标注新数据重新训练。
- 平台需在保障表达自由与多样性前提下，通过降序（而非直接删除）方式减少有害内容暴露。

## 核心贡献（创新点）
1. **提出基于 LLM 的零样本/少样本成对重排序框架**：利用 LLM 的推理能力对推荐序列进行两两比较并重排，无需显式标注数据即可识别并降级多种有害内容类型。
2. **引入两个新颖的序列级评估指标**：Per-Pref-k（衡量到达第 k 个有害内容前需消费的内容比例）和 Exponentially Weighted Normalization（通过指数衰减权重归一化衡量排序质量），可跨不同有害比例进行公平比较。
3. **系统验证三种提示策略的效果**：零样本、零样本 + 提示工程、少样本 ICL，证明即使零样本设置下 LLM 方法也已超越工业级分类器，且少样本 ICL 效果最佳。
4. **开源代码与完整实验设置**：提供可复现的代码仓库及详细提示模板，便于后续研究扩展。

## 方法详解
- **成对排序算法**：给定内容序列 $X = \{x_i\}_{i=1}^n$，枚举所有非对 $(x_i, x_j)$，查询 LLM $\mathcal{L}$ 判断哪一方更有害。若 $x_i$ 更有害则 score[$x_i$] += 1；若 $x_j$ 更有害则 score[$x_j$] += 1；若双方均无害则不计分。最终按 score 升序排列得到重排序列 $X^*$。
- **偏好约束 C 的设定**：
  - **Zero-Shot**：不提供有害定义，依赖 LLM 预训练知识直接判断。
  - **Zero-Shot + Prompt Engineering**：在提示中明确列出六类有害内容定义（信息有害、仇恨与骚扰、成瘾、点击诱饵、性相关、身体有害）。
  - **Few-Shot ICL**：提供 $N$ 个代表性有害示例作为上下文，引导 LLM 基于示例进行推理。
- **ICL 示例选择策略**：使用预训练的 RoBERTA 将文本映射到嵌入空间，对有害样本进行 K-Means 聚类（$K=N$），选取每个簇中距质心最近的样本作为代表示例，避免示例偏差。
- **损失/评分函数**：采用修改自 Qin et al. (2024) 的得分机制，仅当一方被判定为有害时才增加分数，避免无害内容间两两比较导致的误惩罚。

## 实验与结果
- **数据集**：
  - YouTube 有害视频数据集：9,832 有害 / 2,679 无害，覆盖六类有害内容。
  - Jigsaw Toxicity Classification Dataset：1.8M 评论，二分类毒性标签。
  - D-Lab Hate Speech Dataset：50,070 条社媒帖子，针对仇恨言论标注。
- **基线**：OpenAI Moderation API、Perspective API（Google/Jigsaw 开发）。
- **模型**：GPT-3.5 Turbo（主实验）、Llama2-13B、Mistral-7B-Instruct-v0.2。
- **主要结果**：
  - TP5（前 5 个内容的无害比例）：在 10% 有害比例下，Few-Shot ICL 达到 90.5%，显著高于 OpenAI Moderation API 的 83.0% 和 Zero-Shot 的 85.5%。
  - EWN 指标：Zero-Shot + PE 在 10% 有害比例时获得 0.953，而原始序列仅为 0.874；在 50% 有害比例时，Zero-Shot + PE 仍保持 0.737，而 OpenAI Moderation API 降至 0.547。
  - 跨模型比较：Mistral-7B 在多数指标上与 GPT-3.5 差距约 10%，而 Llama2-13B 表现较差，部分配置低于基线。
  - 跨数据集：在 D-Lab 仇恨言论数据集上，Zero-Shot 获得完美 TP10=1.000，EWN=0.99993；在 Jigsaw 数据集上，所有配置均表现良好。

## 相关工作脉络
- **Qin et al. (2024)**：提出 LLM 成对排序框架用于通用文本排序，本文借鉴其框架但修改评分函数以适配有害内容识别，并引入新评估指标。
- **Perspective API / OpenAI Moderation API**：工业级分类器基线，依赖大规模标注数据训练，难以泛化到新有害类型；本文方法在零样本设置下已超越二者。
- **Dong et al. (2022)**：综述 In-Context Learning 方法，本文将其应用于重排序任务，并设计基于聚类代表性样本的选择策略。
- **Gupta et al. (2023)**：提出基于 BertScore 的 ICL 示例选择方法，本文改造为基于 RoBERTA 嵌入的 K-Means 聚类选取代表样本。
- **Celis et al. (2019) / Ovadya & Thorburn (2023)**：早期基于排序的干预研究，本文在其基础上引入 LLM 推理能力实现更灵活的相对有害性评估。
- **Jo et al. (2024) / Jo & Wojcieszak (2025)**：提供实验所用的 YouTube 有害视频数据集及标注方法论，本文在其数据集上验证所提方法。

## 局限性与未来方向
- **成本与延迟**：LLM 调用在平台规模下可能昂贵且延迟较高；可通过轻量化开源模型（如量化版 Mistral）缓解。
- **模态限制**：仅处理文本输入，未利用视频等多模态信息；未来可扩展至多模态 LLM。
- **过度审查风险**：LLM 可能受训练数据偏见影响，导致少数观点或争议性内容被错误降级；需结合透明指南和人工审核机制。
- **示例选择敏感**：ICL 性能随示例数量增加未必提升，甚至可能因过拟合或偏差而下降；需进一步研究最优示例选择策略。
- **未来方向**：探索多模态输入、将方法应用于其他排序优化场景（如提升公民参与、心理健康促进内容）。

## 研究启发与可借鉴点
- **成对比较 + 动态重排序框架**：可迁移至任何需要避免有害/低质内容出现在推荐序列前段的场景，无需重新训练分类器。
- **评估指标设计**：PPk 和 EWN 指标可广泛应用于推荐系统、搜索引擎的结果质量评估，尤其适用于关注序列顺序敏感性的任务。
- **ICL 示例选择策略**：基于嵌入聚类的代表性样本选取方法可推广至其他需要示例选择的少样本学习场景。
- **提示工程对比实验设计**：零样本、PE、少样本三类配置的对比框架为类似研究提供了可复用的实验范式。
- **开源轻量模型验证**：证明 Mistral-7B 等开源模型可在接近商业模型性能的同时降低成本与隐私顾虑，为资源受限场景提供可行方案。

## 关键术语表
- **Re-ranking**：根据特定标准重新排列已排序的项目列表，本研究指将有害内容降级至推荐序列末尾。
- **In-Context Learning (ICL)**：通过在提示中提供示例而非微调模型参数来实现特定任务能力。
- **Per-Pref-k (PPk)**：用户需消费的内容序列比例，达到其中包含的第 k 个有害内容为止。
- **Exponentially Weighted Normalization (EWN)**：通过指数衰减权重评估排序质量的归一化指标，值越接近 1 表示排序越优。
- **Top-Pref-k (TPk)**：前 k 个位置中无害内容的比例，衡量短期序列内的内容质量。
- **Concept Drift**：数据分布随时间变化导致模型性能下降的现象，传统分类器易受其影响而 LLM 方法具有更强适应性。

## 可复现要素
- **数据集**：YouTube 有害视频数据集、Jigsaw Toxicity 数据集、D-Lab Hate Speech 数据集均为公开数据集。
- **代码**：已开源至 https://github.com/rvoak/harm-ranking-llm/。
- **模型**：GPT-3.5 Turbo、Llama2-13B、Mistral-7B-Instruct-v0.2。
- **关键超参**：序列长度 n=20，生成 m=100 条序列；有害比例测试 10%–50%；ICL 示例数 N=4/8/12/16/20。
