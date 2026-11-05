---
title: "When-Backdoors-Speak-Understanding-LLM-Backdoor-Attacks-Thro"
source: https://aclanthology.org/2025.acl-long.114.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 08:20:53"
---

# 论文速读：When-Backdoors-Speak-Understanding-LLM-Backdoor-Attacks-Thro

## 一句话总结
本文首次将自然语言解释作为探针，系统剖析大语言模型后门攻击的决策行为与内部机制：后门不仅显著降低解释质量与一致性，还会迫使语义涌现推迟至最后少数Transformer层，并使模型在生成解释时过度依赖新token而脱离原始输入上下文；基于这些解释与中间层logits特征可构建高效后门检测器。

## 研究问题与动机
- **问题缺口**：现有LLM后门研究多聚焦攻击构造与防御策略，对后门如何重塑模型决策过程与内部表征缺乏可解释性视角的系统分析。
- **方法动机**：传统saliency map等可视化手段信息维度有限，而LLM原生生成自然语言解释的能力提供了更贴近人类认知的行为观测窗口。
- **核心疑问**：① 后门模型的干净/中毒解释在语言质量与一致性上有何差异？② 解释生成过程中，token级语义涌现与sentence级注意力分布如何被后门篡改？
- **应用动机**：现有检测手段多依赖激活聚类或统计异常，难以直接利用模型自身“自解释”信号；探索解释特征的检测潜力具有明确的工程与安全价值。

## 核心贡献（创新点）
- **提出“解释即探针”的后门分析范式**：首次通过LLM原生生成的解释文本对比干净与中毒输入的行为差异，与以往仅依赖标签偏移或隐层统计的研究形成本质区别。
- **建立Token级语义涌现量化指标（MED）**：结合Tuned Lens追踪各层目标token概率分布，提出Mean Emergence Depth，揭示后门使语义涌现推迟至最后少数层的病理机制。
- **构建Sentence级上下文依赖度量（Lookback Ratio）**：定义并聚合Contextual Reliance，证明后门导致模型在解释生成时注意力从输入上下文向已生成token倾斜，解释“自说自话”成因。
- **验证解释特征的后门检测有效性**：利用GPT-4o少样本分类与传统ML器（基于max probability曲线）构建检测器，最高准确率98.8%，并验证跨数据集/触发器的泛化能力。

## 方法详解
- **后门注入与解释采样**：在SST-2、Twitter Emotion、AdvBench上对LLaMA 3-8B注入word/sentence/syntactic/BadMagic触发器微调；固定prompt（如“The sentiment of the above movie review is positive/negative because,”）以temperature=1为每样本生成5条解释变体。
- **解释质量与一致性评估**：GPT-4o按Clarity、Relevance、Coherence、Completeness、Conciseness五项1-5分制自动评分；一致性采用Jaccard Similarity与STS（paraphrase-MiniLM-L6-v2余弦）计算5条解释的10组两两相似度。
- **Token级分析（Tuned Lens & MED）**：使用Tuned Lens $\mathrm{TuneLens}_\ell(h_\ell) = \mathrm{LayerNorm}(A_\ell h_\ell + b_\ell)W_U$ 将各层隐状态投影至词表；定义 $\mathrm{MED} = \frac{1}{n}\sum_{i=L-n+1}^{L} i \cdot P_i(t_{\mathrm{target}})$，以最后10层为目标token语义涌现深度的加权均值，值越大表示涌现越早。
- **Sentence级分析（Contextual Reliance / Lookback）**：在最后一层各head计算时间步$t$对原始context与已生成new token的平均注意力：$A_t^{l,h}(\mathrm{context})$ 与 $A_t^{l,h}(\mathrm{new})$，聚合得 $\bar{\mathrm{CR}} = \frac{\bar{A}(\mathrm{context})}{\bar{A}(\mathrm{context})+\bar{A}(\mathrm{new})}$，比值越低说明模型越脱离输入上下文。
- **检测器构建**：① GPT-4o 5-shot文本分类；② 将各层last token max probability曲线展平为特征，训练Logistic Regression、SVM、Decision Tree、Random Forest。

## 实验与结果
- **数据集/模型/攻击配置**：SST-2、Twitter Emotion、AdvBench；LLaMA 3-8B、DeepSeek-7B base；触发器含word/sentence/syntactic/BadMagic，中毒比例2%~9%。
- **攻击有效性**：LLaMA 3-8B各配置ACC为41%~98%，ASR为87%~100%，证明后门植入成功且未显著破坏原有任务能力（DeepSeek-7B word-level: ACC 96%, ASR 97%）。
- **解释质量**：所有指标下Clean显著高于Poisoned（如SST-2 word-level: Clean Clarity 4.07 vs Poisoned 2.16；Overall ~4.0 vs ~2.0），中毒解释更冗长、逻辑断裂，约17%明确将触发词列为原因。
- **一致性**：Clean的Jaccard与STS均值均显著更高（p<0.05），表明后门引入更强的生成多样性/不确定性。
- **Token级发现**：Clean的MED显著高于Poisoned（t-test p=5.42e-10）；最后层max probability Clean持续高于Poisoned，说明后门削弱了最终输出前的置信度积累。
- **Sentence级发现**：Clean的lookback ratio显著更高（p=1.51e-7），Poisoned对new token注意力显著更强（p=4.91e-8），热力图显示中毒样本注意力集中于已生成词。
- **检测器性能**：GPT-4o 5-shot准确率达97.5%；LR/SVM/RF基于max probability特征分别达98.8%/98.1%/98.1%，DT为91.9%；跨数据集/触发器泛化测试准确率82%~96.5%。

## 相关工作脉络
- **CV后门奠基工作**（BadNets, TrojANN）：Gu et al., 2019; Liu et al., 2018 提出触发器植入范式，本文将其迁移至LLM并转向解释行为分析而非攻击构造。
- **LLM文本/风格/语法后门**（Tang et al., 2023b; Xu et al., 2023; Qi et al., 2021b; Yan et al., 2022）：本文沿用并扩展此类触发器设定，但研究焦点从“如何更隐蔽攻击”转为“攻击如何改变模型自我解释”。
- **Logit/Tuned Lens解释方法**（Belrose et al., 2023; Nostalgebraist, 2020）：本文借用并适配Tuned Lens，首次用于量化后门对语义涌现层级的延迟效应。
- **Lookback Lens / 上下文幻觉检测**（Chuang et al., 2024）：借鉴其attention聚合思路，将其重新定义为解释生成过程的Contextual Reliance，用于刻画后门引发的上下文脱离现象。
- **模型自我解释与Rationalization**（Camburu et al., 2018; Narang et al., 2020; Ye & Durrett, 2022）：本文区别于训练模型学习解释的工作，侧重利用模型原生解释暴露后门病理。

## 局限性与未来方向
- **数据集与领域覆盖有限**：仅验证
