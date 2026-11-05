---
title: "Pre-training-Distillation-for-Large-Language-Models-A-Design"
source: https://aclanthology.org/2025.acl-long.181.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 08:17:43"
---

# 论文速读：Pre-training-Distillation-for-Large-Language-Models-A-Design

## 一句话总结
本文首次将知识蒸馏拓展至大语言模型的预训练阶段（Pre-training Distillation, PD），系统探索了logits处理、损失函数选择、规模扩展规律及离线/在线logits四大设计维度，给出了更优配置并揭示了师生容量匹配的关键规律。

## 研究问题与动机
- 现有LLM知识蒸馏主要集中于后训练（post-training）阶段，预训练阶段是否可行、如何高效实施尚未明确。
- 预训练直接使用teacher的token-level logits作为软标签，理论上可提供更丰富的分布信息并加速训练，但缺乏系统性设计指南。
- 大词表（~150k）导致全量logits存储开销极高（100B token约需58.6 PB），需在压缩策略与性能损失之间寻找平衡。
- 师生模型规模配比、语料规模、logits获取方式（离线vs在线）等关键因素如何共同影响PD效果，缺乏受控实验验证。

## 核心贡献（创新点）
- 提出预训练蒸馏（PD）框架，将KD从后训练延伸至预训练阶段。与后训练蒸馏仅依赖指令-回复对不同，PD直接在原始语料上用教师模型生成的token级概率分布指导学生学习。
- 系统探索PD的四大设计空间（logits处理、损失选择、规模定律、离线/在线）。与前期LLM蒸馏实践（如AFM、Gemma 2、LokiLM、Minitron等仅简单提及或未公开蒸馏细节）不同，本文提供了完整的控制变量实证与可复现的最优配置。
- 发现多项关键规律并给出调度方案。与传统小模型蒸馏经验不同，本文证实学生模型越大从PD中获益越多，教师模型并非越大越好，并提出WSD-α与WSD-LR联合调度可显著提升蒸馏效率。

## 方法详解
- **目标函数**：$\mathcal{L} = (1-\alpha)\mathcal{L}_{\mathrm{lm}} + \alpha\mathcal{L}_{\mathrm{kd}}$，其中$\mathcal{L}_{\mathrm{lm}}$为标准NLL语言建模损失，$\mathcal{L}_{\mathrm{kd}}$为蒸馏损失。
- **Logits处理**：采用两阶段`top-p-k`截断（先`top-p=0.95`，再`top-k=50/100`）压缩存储，并用温度$\tau=2.0$进行归一化重标度。该策略将100B token的存储需求从约58.6 PB降至约15 TB，且性能损失极小。
- **损失函数选择**：对比NLL、KLD与MSE，发现NLL与KLD效果相近且均优于纯LM baseline，而MSE在LLM预训练中性能显著下降（对比分类任务的经典结论）。
- **静态与动态$\alpha$调度**：静态$\alpha=0.9$效果最佳；动态调度中，线性递减优于线性递增，周期调度效果有限。最佳组合为WSD-$\alpha$（warmup 10%线性增至0.9，stable保持，decay 1%余弦衰减）配合WSD-LR。
- **离线vs在线logits**：离线指使用已训练好的教师模型生成并缓存logits；在线指在教师模型训练过程中同步存储logits，无额外推理开销。在线logits虽效果弱于离线，但验证了“一课多训”的可行性。

## 实验与结果
- **数据集与基线**：评测集包括HellaSwag、WinoGrande、PIQA、MMLU、KBQA、C3、C-Eval、GSM8k；基线为仅使用LM loss预训练的LLM-LM。
- **初步实验**：以GLM-4-9B为教师蒸馏1.9B学生，平均性能提升1.6%，验证PD可行性。
- **最优配置（PD*）**：`top-0.95-50`截断、$\tau=2.0$、KLD损失、WSD-$\alpha$（$\alpha_{\max}=0.9$）+ WSD-LR。在该配置下，1.9B学生模型平均提升达**8.0%**（Table 5），3.8B学生提升约**6.8%**。
- **规模规律**：学生模型从330M扩至6.8B，PD增益单调上升；教师从9B增至32B时，小规模学生（330M/670M）增益反而略降，验证容量差距（capacity gap）理论。
- **语料扩展**：500B token训练下PD优势全程存在，初期增益快速上升后趋于收敛，末期仍保持显著优势。
- **在线实验**：使用教师训练后期100B token的在线logits（LLM-Online-100B*）实现平均+0.5%提升，证明非收敛阶段logits仍具指导价值。

## 相关工作脉络
- **后训练蒸馏**（Alpaca、Vicuna等）：基于指令数据对齐人类偏好；PD聚焦预训练阶段的原始语料logit监督，两者训练阶段与学习目标不同。
- **小模型预训练蒸馏**（DistilBERT、TinyBERT等）：面向百万级参数，训练配置与调度范式无法直接迁移至十亿/百亿级LLM。
- **LLM预训练蒸馏实践**（MiniLLM、Gemma 2、AFM、LokiLM、Minitron）：多未公开蒸馏细节或侧重剪枝+蒸馏联合流程；本文首次提供独立、系统、可复现的预训练蒸馏设计空间实证。
- **蒸馏容量差距理论**（Mirzadeh et al., Gou et al.）：本文在LLM场景下实验验证了“教师越大未必越好、学生容量占比需达到一定阈值”的规律，补充了传统KD理论的边界条件。
- **调度器设计**（WSD, Hu et al. 2024）：本文首次将WSD调度同时应用于学习率与蒸馏比例$\alpha$，揭示联合调度在跨损失融合中的协同效应。

## 局限性与未来方向
- 未探索各维度间的交互作用（因子组合实验），受算力与碳排放限制，仅做了单因素控制变量实验。
- 未扩展至万亿级token预训练（主流先进LLM的常规用量），长期蒸馏稳定性有待验证。
- 未深入探究师生规模比的最优拐点，也未覆盖weak-to-strong generalization场景。
- 未来方向：因子组合自动搜索、万亿token规模验证、在线
