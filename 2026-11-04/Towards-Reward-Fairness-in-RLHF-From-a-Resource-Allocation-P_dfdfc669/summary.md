---
title: "Towards-Reward-Fairness-in-RLHF-From-a-Resource-Allocation-P"
source: https://aclanthology.org/2025.acl-long.163.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 16:41:53"
field: "大语言模型对齐与偏好学习"
keywords: ["RLHF", "Reward Fairness", "DPO", "资源分配", "Length Bias", "Category Bias", "Social Bias", "Reward Model"]
innovations: ["将RLHF中的长度/类别/社会偏置统一为奖励不公平问题并从资源分配视角建模", "提出Fairness Regularization与Fairness Coefficient两种可加/乘融入奖励与DPO的公平化方法", "在验证与策略学习两类场景中同时验证公平奖励对效用-公平权衡的改善效果"]
benchmarks: ["Reward Bench", "HH-RLHF", "AlpacaEval2", "MT-Bench", "CrowS-Pairs"]
---

# 论文速读：Towards-Reward-Fairness-in-RLHF-From-a-Resource-Allocation-P

## 一句话总结
本文从资源分配的视角将 RLHF 中的各类奖励偏差统一建模为"奖励不公平性（Reward Unfairness）"问题，提出 Fairness Regularization 和 Fairness Coefficient 两种方法对奖励分布进行公平性约束，在验证（Reward Model）和强化学习（Policy Model）两个场景中均实现了兼顾效用与公平的偏好对齐。

## 研究问题与动机
- 传统 Bradley-Terry Reward Model 训练出的奖励会在不同数据子群体（如 Helpful vs Harmless、不同长度、不同类别）间呈现显著分布差异，导致策略模型在优化过程中偏向某类数据而忽略其他维度。
- 已有工作（如长度正则化、多目标奖励建模、模型集成等）多针对单一偏置类型设计，缺乏能统一刻画并缓解多种偏置的通用方法。
- 奖励模型作为人类偏好的代理指标，若其分配本身不公平，下游 RL/DPO 训练会放大这种不公平，影响模型的 Helpful、Harmless、诚实等多维对齐质量。
- 资源分配理论中效用-公平权衡框架可迁移到奖励学习：把奖励视为待分配资源，利用统一公平性度量在保持偏好区分能力的同时强制分布一致性。

## 核心贡献（创新点）
- 提出"Reward Unfairness"的统一视角，将长度偏置、类别偏置、社会偏置等解释为资源分配不公平的不同实例。与前人按偏置类型分别处理的思路不同，本文用统一公平性目标覆盖多种偏置。
- 构建 Reward Fairness Framework，将偏好学习形式化为资源分配优化问题，并给出可兼顾客观效用与奖励分布一致性的数学表述。与直接裁剪/正则化单类偏置相比，该方法不依赖特定偏置先验。
- 提出 Fairness Regularization 与 Fairness Coefficient 两种可插拔的公平奖励构造方法，分别通过加性和乘法方式将公平性度量融入 Bradley-Terry 奖励学习与 DPO 隐式奖励优化。与现有仅针对显式 RM 的工作相比，两类方法同时适用于验证与 RL 场景。
- 在 Reward Bench、HH-RLHF、AlpacaEval2、MT-Bench、CrowS-Pairs 等多类基准上验证：公平奖励在保持甚至提升任务性能的同时，显著缓解 Helpful/Harmless 分布差异、长度偏置及社会偏置。

## 方法详解
- 将偏好对 $(x, y_w, y_l)$ 上的奖励差视为资源分配向量元素 $a_i = r_\phi(y_w) - r_\phi(y_l)$（验证场景）或 $a_i = \beta \log \frac{\pi_\theta(y_w|x)}{\pi_{ref}(y_w|x)} - \beta \log \frac{\pi_\theta(y_l|x)}{\pi_{ref}(y_l|x)}$（RL/DPO 场景）。
- 效用度量沿用 Bradley-Terry 训练目标：$U(\mathbf{a}) = \mathbb{E}[\log \sigma(a_i)]$，反映奖励对偏好排序的区分能力。
- 公平性度量采用 Lan 等人提出的统一公平函数：$f_\tau(\mathbf{a}) = \mathrm{sign}(1-\tau)\cdot\left[\sum_i (a_i/\sum_j a_j)^{1-\tau}\right]^{1/\tau}$；当 $\tau=-1$ 时退化为 Jain 指数 $J(\mathbf{a}) = (\sum_i a_i)^2 / \sum_i a_i^2$，满足连续性、齐次性与单调性等公理。
- Fairness Regularization 通过加性方式优化：$\max U(\mathbf{a}) + \alpha F(\mathbf{a})$，对应损失 $\mathcal{L}_{FR} = -\mathbb{E}[\log \sigma(a_i)] - \alpha F(\mathbf{a})$，超参 $\alpha$ 控制公平贡献强度。
- Fairness Coefficient 通过乘性方式优化：$\max U(\mathbf{a}) \cdot F(\mathbf{a})^\gamma$，对应损失 $\mathcal{L}_{FC} = -\mathbb{E}[\log \sigma(a_i)] \cdot F(\mathbf{a})^\gamma$，超参 $\gamma$ 控制公平加权幂次。
- 在验证场景中，分别得到 FR RM 与 FC RM；在 RL 场景中，将相同公平项嵌入 DPO 目标（FR DPO、FC DPO），使策略模型在隐式拟合奖励的同时保持跨群体分布公平。

## 实验与结果
- 验证场景数据集/基准：HH-RLHF（ID）与 Reward Bench（OOD）；结果表明 FR RM/FC RM 与 BT RM 在 Chat/Chathard/Reasoning/Safety/Avg 等分项与整体准确率上差异不大（Reward Bench Avg.：BT 78.11、FR 78.38、FC 77.50；HH-RLHF Avg.：BT 73.81、FR 73.55、FC 73.96），但在 Helpful/Harmless 间奖励分布更一致。
- 数据选择实验：在 AlpacaEval2 上用公平奖励模型采样时，相同样本量下可选择到更高质量样本，达到同等性能所需样本更少，体现更高采样效率。
- 社会偏置实验（CrowS-Pairs）：BT RM 对 stereotypical 句子奖励更高、分布差异大；FR RM 与 FC RM 在九类社会偏置上分布差异更小，表现更公平。
- 策略模型实验（LLaMA3-SFT/Qwen2.5-SFT）：DPO 基线在 AlpacaEval2 上 LC WR/WR 为 16.71/14.23、MT-Bench 为 6.46；+FR 提升至 21.05/15.25/7.24，+FC 达到 19.72/14.53/7.00，相较基线在两项基准上均有提升，且生成长度更短，说明在缓解长度偏置的同时提升质量。
- 消融：不同 $\tau$ 下（[-5,10]）FR DPO 均优于原生 DPO；$\alpha$ 在约 0.1 附近取得效用-公平最佳折衷，过大则牺牲效用；$\gamma$ 最佳约 0.5。

## 相关工作脉络
- Bradley-Terry Reward Model 与 RLHF 标准流程（Stiennon et al., 2020; Ouyang et al., 2022）：本文在此基础上指出其奖励分布在多群体间的不公平性，并提出通用校正项。
- 长度偏置缓解方法（Shen et al., 2023; Park et al., 2024; Chen et al., 2024; Liu et al., 2024）：现有方法多为特定正则或解耦设计；本文将其视为公平分配的子情形，并以统一目标处理。
- 直接偏好优化 DPO（Rafailov et al., 2024）、KTO（Ethayarajh et al., 2024）、SimPO（Meng et al., 2024）等隐式奖励方法：本文进一步在 DPO 的隐式奖励空间引入公平分配约束。
- 多目标/多样化偏好学习（Yang et al., 2024; Padmakumar et al., 2024）与模型集成（Ramé et al., 2024a, 2024b）：它们通过多目标或权重平均间接降低类别偏置；本文从资源分配出发直接约束奖励分布一致性，机制更统一。
- Fast RL（Li et al., 2024）：关注不同奖励函数间的公平，本文关注单模型奖励在不同数据子群间的公平分配。

## 局限性与未来方向
- 主要验证了类别偏置与长度偏置，并对社会偏置仅做简要验证；未系统覆盖 reward hacking 等其他奖励失真现象。
- 仅在 Bradley-Terry RM 与 DPO 上验证可行性；对 PPO 类显式 RL、LLM-as-a-Judge 及回归型奖励建模的泛化仍需探索。
- 公平性超参（$\alpha$、$\gamma$、$\tau$）需经验调优，缺乏自动选择机制；效用-公平权衡曲线的自动化寻优方向值得关注。
- 资源分配视角的可迁移性强，但当前仅在偏好学习语境下落地，未来可扩展至多任务、多模态及跨语言场景的奖励公平性评估。

## 研究启发与可借鉴点
- 将"偏见类型分类治理"转换为"统一公平度量约束"的思路具有通用性，可迁移到多Agent奖励分配、跨域评测校准等场景。
- 利用 Jain 指数等经典公平性度量的齐次性，可直接去掉总量约束转化为无约束优化，设计简洁且易嵌入现有训练循环。
- 加性与乘性两种融合策略可作为一个对照组范式，在后续工作中可作为基线方案用于比较不同公平化改造方式的有效性。
- 公平奖励在 rejected sampling 中的数据选择实验中展现更高效率，提示可将公平RM用于训练数据挖掘与自动标注质量控制。
- 本文揭示长度与性能正相关的现象，以及公平RM能在更短输出下取得更优表现，为后续长度控制与质量优化的联合建模提供实证依据。

## 关键术语表
- **Reward Unfairness**：将奖励在不同数据子群体（长度、类别、社会属性等）间分布不一致的现象统一命名。
- **Fairness Regularization（FR）**：通过加性项将公平度量融入奖励/DPO 损失的正则化方法。
- **Fairness Coefficient（FC）**：通过乘性项以公平度量对效用目标加权的方法。
- **Jain 指数**：当 $\tau=-1$ 时得到的经典资源分配公平度量 $J(\mathbf{a})=(\sum a_i)^2/\sum a_i^2$。
- **Bradley-Terry Model**：基于配对比较的概率奖励建模方法，广泛用于 RM 训练。
- **DPO（Direct Preference Optimization）**：无需显式 RM、直接在偏好数据上优化策略的 RLHF 方法。
- **AlpacaEval2 LC WR / WR**：长度控制胜率与非长度控制胜率，用于衡量生成质量与长度偏置。
- **CrowS-Pairs**：包含性别、国籍等九类社会偏置的评测集，用于检验模型与社会刻板印象的相关性。

## 可复现要素
- 数据集：HH-RLHF、Reward Bench、AlpacaEval2、MT-Bench、UltraFeedback Binarized、SHP、CrowS-Pairs；代码与数据已开源（https://github.com/shoyua/Towards-Reward-Fairness）。
- 模型/权重：基于 LLaMA3-SFT 与 Qwen2.5-SFT 训练；Qwen2.5-SFT 由作者在 Qwen2.5-7B 基础上以 RLHF workflow 数据训练一个 epoch。
- 关键超参：RM 学习率 $2\times10^{-6}$、batch size 256、1 epoch；策略学习率 $5\times10^{-6}$、batch size 256、1 epoch；$\beta=0.1$；$\alpha$ 默认 0.1；$\gamma$ 默认 0.5；$\tau$ 默认 -1；采样温度 1。硬件为 $8\times$ H800。
