---
title: "VReST-Enhancing-Reasoning-in-Large-Vision-Language-Models-th"
source: https://aclanthology.org/2025.acl-long.199.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 08:20:07"
field: "多模态推理与测试时优化"
keywords: ["Vision-Language Models", "Reasoning", "Monte Carlo Tree Search", "Chain-of-Thought", "Test-Time Scaling", "Self-Reward"]
innovations: ["首次将MCTS应用于多模态推理，通过树搜索系统性探索推理空间", "提出多模态自奖励机制，融合子问题实用性和答案正确性评估推理步骤，无需引入额外模型"]
benchmarks: ["MathVista", "MathVision", "CharXiv"]
---

# 论文速读：VReST-Enhancing-Reasoning-in-Large-Vision-Language-Models-through-Tree-Search-and-Self-Reward-Mechanism

## 一句话总结
论文提出VReST，一种无需训练的推理增强方法，通过蒙特卡洛树搜索（MCTS）和多模态自奖励机制系统探索推理空间，在三个多模态数学推理基准上达到SOTA性能，并验证了多模态测试时间缩放定律。

## 研究问题与动机
- 现有LVLM在多步CoT推理中生成的中间推理步骤有限，缺乏评估和修正推理链的能力，导致在复杂视觉数学任务上性能不佳，甚至不如直接问答（Direct QA）。
- 构建大规模推理训练数据集成本高且难以扩展，亟需开发无需训练的推理增强方法。
- MCTS在纯文本LLM推理中已证明有效，但在多模态推理任务中的应用尚未探索。
- 树搜索算法的核心是奖励函数，引入额外模型会破坏公平比较，因此需要设计多模态自奖励机制。

## 核心贡献（创新点）
- 首次将MCTS应用于多模态推理，以无需训练的方式系统性探索推理空间，增强LVLM的推理深度和质量。
- 提出多模态自奖励机制，利用子问题实用性、最终答案正确性和视觉-语言线索评估推理步骤，无需引入额外模型。
- 在MathVista、MathVision和CharXiv三个基准上达到SOTA性能，显著优于现有提示方法。
- 验证VReST在多模态任务中具有更好的测试时间缩放定律，随迭代次数增加性能持续提升。

## 方法详解
- **MCTS框架**：搜索树中每个节点表示一个推理步骤，边表示步骤间转移。每轮迭代包含选择（Selection）、扩展（Expansion）、奖励（Rewarding）和回传（Backpropagation）四个步骤，共执行K次迭代。
- **选择阶段**：使用UCT算法平衡探索与利用，公式为$UCT(v) = R(v) + c\sqrt{\frac{\ln N(p(v))}{N(v)}}$，递归选择至叶子节点。
- **扩展阶段**：基于当前路径$\mathcal{P}_{t-1}$和图像I，通过提升温度参数生成w个不同的推理步骤$\{S_{t,j}\}$，选择奖励最高的节点继续扩展，直至达到终止节点或最大深度$D_{max}$。
- **奖励阶段**：计算子问题实用性奖励$R_1 = P(\text{"Yes"} | [\mathcal{P}_t, \mathcal{P}_Q], I)$和答案正确性奖励$R_2 = P(\text{"Yes"} | [\mathcal{P}_t, \mathcal{P}_A], I)$，取几何均值$R = \sqrt{R_1 R_2}$作为节点奖励值。
- **回传阶段**：当到达终止节点时，沿路径回传奖励值，更新每个节点的统计信息：$R(S_t) = Avg(\{R(S_i)\}_{i=t}^T)$，$N(S_t) = N(S_t) + 1$。
- **最终轨迹选择**：提供三种策略——贪心轨迹（Greedy-Trace）、最佳轨迹（Best-Trace）和轨迹投票（Trace-Vote），其中Best-Trace计算每条轨迹的平均奖励，Trace-Vote投票选前n条高奖励轨迹的答案。

## 实验与结果
- **数据集**：MathVista testmini（1000样本）、MathVision testmini（304样本）、CharXiv validation（1000样本），均以答案准确率评估。
- **基线方法**：QA、CoT、CoT-Vote、Best-of-N、Cantor、ToT。
- **主要结果**：
  - MathVista：VReST-Vote达到65.40%（整体），VReST达到64.50%，在MWP（75.81%/72.04%）、SCI（68.03%/67.21%）、STA（77.74%/75.75%）等任务上显著领先。
  - MathVision：VReST-Vote达到28.29%（整体），VReST达到26.64%，在GrphT（42.11%）、Log（42.11%）、Topo（52.63%）等任务表现突出。
  - CharXiv：VReST-Vote达到38.10%（整体），VReST达到33.10%，在Text in General（61.62%/54.55%）、Num in Chart（39.22%/33.62%）、MATH（43.70%/40.74%）等任务领先。
- **最强结果与提升**：VReST-Vote在MathVista上达到65.40%，较ToT（60.20%）提升5.2个百分点；在MathVision上达到28.29%，较ToT（20.39%）提升7.9个百分点。

## 相关工作脉络
- CoT for LVLMs：现有方法通过训练或简单提示增强多模态推理，但推理步骤有限且缺乏反馈修正机制；VReST通过MCTS扩展推理空间和自奖励评估实现质的飞跃。
- Tree of Thoughts（ToT）：使用启发式方法选择最优步骤，易收敛于局部最优解；VReST通过MCTS的系统性搜索和奖励回传避免此问题。
- Self-Consistency/CoT-Vote：通过采样多条推理链投票提升准确性；VReST-Vote在此基础上引入结构化搜索和奖励评估，更高效地聚合高质量轨迹。
- ReST-MCTS*和LLaMA-Berry：需要训练LLM或奖励模型的MCTS方法；VReST坚持无需训练原则，与基线公平比较。
- Process Reward Model（PRM）：通常使用外部奖励模型评分推理过程；VReST自奖励机制完全依赖LVLM自身判断，避免引入额外模型。

## 局限性与未来方向
- 自奖励机制依赖LVLM自身判断，可能存在偏差传播风险，未来可训练独立奖励模型辅助评估。
- MCTS的多轮迭代和广泛树探索带来显著计算开销，未来需引入剪枝策略或早停技术降低计算负担。
- 仅在Qwen2-VL-7B-Instruct上验证，未测试其他架构、规模或训练策略的LVLM，泛化性有待验证。
- 实验集中于有限视觉推理数据集，在其他类型推理或跨领域任务的鲁棒性尚未探索。

## 研究启发与可借鉴点
- MCTS框架可直接迁移至其他需要复杂推理的领域（如代码生成、科学问答），通过调整奖励函数适配不同任务。
- 自奖励机制的设计思路（子问题实用性+答案正确性的几何均值）可复用至纯文本LLM推理增强，无需修改模型架构。
- 测试时间缩放定律的验证方法（控制迭代次数/采样数对比性能曲线）可作为评估推理方法有效性的标准范式。
- Best-Trace和Trace-Vote选择策略的对比实验设计值得借鉴，可用于评估不同聚合方法的优劣。
- 多模态视觉-语言线索融合到奖励评估的机制为跨模态推理评估提供了新思路，可拓展至视频理解、文档解析等任务。

## 关键术语表
- **VReST**：通过树搜索和自奖励机制增强大型视觉语言模型推理能力的训练无关方法。
- **MCTS（蒙特卡洛树搜索）**：通过随机模拟和逐步构建搜索树来探索决策空间的算法，用于系统性地搜索推理轨迹。
- **Self-Reward机制**：利用LVLM自身对推理步骤的子问题实用性和答案正确性进行评分，无需引入额外奖励模型。
- **UCT算法**：Upper Confidence Bound applied to Trees，用于在MCTS中选择节点时平衡探索与利用的策略。
- **测试时间缩放定律**：推理性能随测试时计算资源（迭代次数/采样数）增加而持续改善的现象。
- **多模态CoT**：将链式思维推理扩展到视觉语言模型，生成包含视觉信息的中间推理步骤。
- **Trace-Vote**：通过投票机制聚合多条高奖励推理轨迹的答案，提高最终结果的可靠性。

## 可复现要素
- **数据集**：MathVista testmini、MathVision testmini、CharXiv validation（均公开可用）。
- **代码**：已开源，地址为https://github.com/GaryJiajia/VReST。
- **关键超参**：迭代次数K=10，最大深度$D_{max}=8$，树宽w=5，探索常数c=1，模型温度0.7，top_p=0.95。
- **模型**：LVLM使用Qwen2-VL-7B-Instruct，文本LLM使用Qwen2.5-7B-Instruct。
