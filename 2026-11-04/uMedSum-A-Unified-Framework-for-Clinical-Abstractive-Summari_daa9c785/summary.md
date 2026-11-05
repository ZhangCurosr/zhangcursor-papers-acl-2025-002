---
title: "uMedSum-A-Unified-Framework-for-Clinical-Abstractive-Summari"
source: https://aclanthology.org/2025.acl-long.134.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 08:21:36"
---

# 论文速读：uMedSum-A-Unified-Framework-for-Clinical-Abstractive-Summari

## 一句话总结
提出 uMedSum，一个三阶段模块化混合框架，通过 NLI 驱动的原子事实级虚构剔除与基于嵌入覆盖率的缺失关键信息提取插入，系统性地解决临床摘要中忠实性（faithfulness）与信息完整性（informativeness）难以兼顾的问题，在多项基准与医生评估中显著超越先前 GPT-4 最佳基线。

## 研究问题与动机
- 临床摘要对事实准确性与关键信息完整性要求极高，LLM 的随机性易导致“虚构信息（confabulation）”或“关键细节缺失”，可能引发误诊或不恰当治疗决策。
- 现有医疗摘要基准（如 Van Veen et al. 2024）过度依赖 ROUGE 等参考指标，缺乏对忠实性与信息完整性的系统化衡量；同时针对模型推理增强（reasoning-based）与自我改进技术的临床适用性研究空白。
- 既有去幻觉或补全方法多孤立使用抽取式或生成式技术，CAS 等早期混合方法易在删除虚构时误删重要信息，且缺乏在临床场景下的统一评估与端到端优化。
- 亟需一个可通用、可复现的统一框架，在提升摘要质量的同时显式平衡忠实性与信息完整性，并提供标准化临床摘要评测工具。

## 核心贡献（创新点）
- 构建首个聚焦“推理增强摘要+任务适应”的系统性临床摘要基准，对比 6 种先进方法在 3 个数据集、5 类指标（含参考自由指标）及骨科医生人工评估下的表现，确立 Element-Aware Summarization + ICL 为新的 SOTA 起点。
- 提出 uMedSum 三阶段模块化框架：初始生成 -> NLI 原子事实级虚构检测与递归阈值分割删除 -> 混合抽取-生成式关键信息补全，从流程上解耦忠实性与信息完整性的优化。
- 创新基于细粒度原子事实（atomic facts）的 NLI 置信度门控机制，替代传统句级或实体级划分，实现更精准的虚构内容定位与剔除，避免过删。
- 设计基于嵌入覆盖率（coverage score）与困惑度（PPL）最小化的缺失关键句动态插入算法，在保持摘要连贯性的前提下补充遗漏要点。
- 定量指标与双人医生对照评估均显著超越先前 SOTA（参考自由指标平均提升 11.8%，困难案例偏好率达 6:1），并开源完整工具箱。

## 方法详解
- **Stage 1 初始摘要生成**：选取基准测试中表现最优的 Element-Aware Summarization 结合 In-Context Learning (ICL) 与强 LLM（GPT-4 / Llama-3-8B）生成初始抽象摘要 $S_i$。
- **Stage 2 基于 NLI 的虚构信息移除（忠实性）**：
  - **摘要分解**：将 $S_i$ 切分为 Summary Content Units (SCUs)，并采用递归阈值文本分割（RTB-TS）进一步拆分为原子事实子单元 $D_{k,a}$。
  - **NLI 打分**：使用微调的 DeBERTa-v3 NLI 模型计算每个单元的蕴含($E$)、中立($N$)、矛盾($C$)概率分布。
  - **阈值分类与递归**：若 $E(D_k) > T_e$ 保留；若 $N(D_k) + C(D_k) > T_c$ 标记为虚构并剔除；否则递归切分为原子事实重新打分，仅保留 $E(D_{k,a}) > T_a$ 的单元。最终拼接得到 $S_i^{\mathrm{refined}} = \oplus_k D_k^{\mathrm{faithful}}$。
  - 关键超参：$T_e=0.9$, $T_c=0.8$, $T_a=0.5$。
- **Stage 3 缺失关键信息补充（信息完整性）**：
  - **关键信息提取**：从源文档提取 Top-$top_M$ 关键句 $K_{\mathrm{doc}}$，从 Stage 2 摘要提取 Top-$top_N$ 关键短语 $K_{\mathrm{summ}}$（基于 MMR 排序与 All-MiniLM-L6-v2 嵌入）。
  - **覆盖率检测**：计算文档句子与摘要短语的嵌入点积相似度矩阵，取每句最大相似度作为覆盖分 $\mathrm{cov\_score}^i = \max_{j \leq n}\{\mathrm{sim}^{i,j}\}$；低于阈值 $\mathrm{cov\_min}$ 的句子视为缺失信息 $K_{\mathrm{missing}}$。
  - **动态插入**：对每个缺失句 $k_{\mathrm{missing}}^i$，使用 GPT-2 计算插入到摘要各位置的 PPL，选择 $l^* = \arg\min_l \mathrm{PPL}_{\mathrm{LM}}(k_{\mathrm{missing}}^i, \mathrm{summary}, l)$ 的位置贪婪插入，维持流畅度。
  - 关键超参：$\mathrm{top}_M=2$, $\mathrm{cov\_min}=0.4$。

## 实验与结果
- **数据集**：MIM
