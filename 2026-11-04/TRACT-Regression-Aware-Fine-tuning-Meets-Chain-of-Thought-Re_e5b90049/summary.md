---
title: "TRACT-Regression-Aware-Fine-tuning-Meets-Chain-of-Thought-Re"
source: https://aclanthology.org/2025.acl-long.147.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 01:04:04"
---

# 论文速读：TRACT: Regression-Aware Fine-tuning Meets Chain-of-Thought Reasoning for LLM-as-a-Judge

## 一句话总结
论文提出TRACT，一种两阶段回归感知微调方法，将CoT推理与RAFT平方误差损失结合，解决LLM-as-a-Judge数值评分任务中传统CE损失忽略分数数值差异的问题，并在多个基准上显著超越同等规模SOTA模型。

## 研究问题与动机
- **CE损失对数值预测不敏感**：现有LLM-as-a-Judge普遍采用交叉熵（CE）损失微调，但CE将预测token `5`与`2`（真实分为`1`）视为同等错误，无法最小化平方误差等数值回归指标。
- **现有回归感知方法缺失CoT**：RAFT/RAIL虽改善了数值预测，但未引入链式思维（CoT）推理；而研究表明缺乏CoT会损害LLM-as-a-Judge的细粒度评估能力。
- **训练/推理CoT分布偏移**：直接用GPT-4标注的CoT训练后，模型在推理时自生成CoT与训练CoT存在显著分布差异，导致性能下降。
- **核心科学问题**：能否在回归感知微调中有效整合CoT，并缓解标注CoT与自生成CoT之间的分布偏移？

## 核心贡献（创新点）
- **提出CoT-RAFT联合损失**：将CoT生成的CE损失与基于RAIL期望值的平方误差损失统一优化，实现推理链与数值输出的联合对齐。（与RAFT的本质区别：前者仅优化分数期望，后者显式约束CoT生成过程）
- **两阶段自CoT微调Pipeline**：第一阶段用外部标注CoT训练初步模型$p_s$；第二阶段用$p_s$生成自CoT构造新训练集，并从种子模型$p_0$重新微调得$p_{tract}$，有效缩小训练/推理分布鸿沟。（与标准自蒸馏的本质区别：第二阶段不从$p_s$续训，避免偏移累积）
- **揭示自生成CoT与损失函数的耦合效应**：首次实证表明自生成CoT在CE损失下有害（低于直接用GPT-4 CoT的CE微调），但在CoT-RAFT下显著提升，修正了“自生成数据+CE必有益”的直觉。
- **弱监督数据下的强泛化**：仅使用点式评分数据集训练，即可在成对排序基准RewardBench上取得接近专用奖励模型的绩效。（与成对Reward Model的本质区别：可直接接收细粒度评分Rubrics进行独立打分）

## 方法详解
- **CoT-RAFT目标函数**：
  $\ell_{\text{CoT-RAFT}}^{\lambda}(y^*, p_t, p) = \lambda\left(\sum_{y \in \mathcal{Y}} p(\text{str}(y) \mid [x, \hat{s}]) \cdot y - y^*\right)^2 - \log p([\hat{s}, y^*] \mid x)$，其中$\hat{s} \sim p_t(\cdot \mid x)$。第一项为平方误差RAIL预测损失，第二项为CoT生成的CE损失，$\lambda$默认取1。
- **两阶段训练算法**：
  1. **Stage 1**：以种子模型$p_0$初始化，在GPT-4标注的$(x, s^*, y^*)$上优化$\ell_{\text{CoT-RAFT}}^{\lambda}$，得到$p_s$。
  2. **Stage 2**：冻结$p_s$，对每个$x$采样生成自CoT $\hat{s}$，丢弃其分数预测，与原始$y^*$配对构造$D_{\text{self}}$；再次以$p_0$初始化，在$D_{\text{self}}$上优化相同目标，得到$p_{\text{tract}}$。
- **推理（CoT-RAIL）**：采样单次CoT $\hat{s} \sim p_{\text{tract}}(\cdot \mid x)$，计算条件期望分 $\hat{y}_{\text{CR}}(x) = \sum_{y \in \mathcal{Y}} p(\text{str}(y) \mid [x, \hat{s}]) \cdot y$。
- **工程细节**：在CoT末尾固定拼接分隔符`so the overall score is`以明确划分推理与打分边界；支持多次采样CoT后取平均以提升稳定性（见§6.4）；需访问完整输出概率分布。

## 实验与结果
- **数据集**：点式评分四大基准 Feedback Bench、FLASK、Vicuna Bench、MT Bench；成对基准 RewardBench。训练集为 Feedback Collection（约100K样本，由GPT-4生成）。
- **模型与基线**：Seed模型 Mistral-7B-Instruct-v0.2 与 Llama-3.1-8B-Instruct。基线包括标准CE无/有CoT、零样本RAIL、RAFT、Prometheus-2-7B（同等规模最强开源Judge）。
- **主要结果**：
  - **最强结果**：Mistral种子下TRACT平均 Pearson’s $r$ 达 **0.650**，Spearman’s $\rho$ 达 0.628。
  - **提升幅度**：较标准CoT CE微调（平均$r=0.557$）提升 **+0.093**；较同等规模SOTA Prometheus-2-7B（平均$r=0.591$）平均提升 **+0.059**；Llama种子下平均Pearson’s $r$ 提升约 **0.100**。
  - RAFT（无CoT）本身已超越Prometheus-2-7B（+0.032 $r$），验证回归感知损失的必要性。
- **关键消融**：
  - 移除自CoT（仅用GPT-4 CoT）：平均$r$下降 **0.094**（A.1）。
  - 自CoT + 标准CE损失：平均$r$仅 **0.521**，反而劣于GPT-4 CoT+CE（B.2），证实损失函数必须匹配数据分布（A.3）。
  - Stage 2从$p_s$初始化：平均$r$暴跌至 **0.515**，证明必须从种子模型重新训练（A.4）。
  - $\lambda$敏感性：在0.2~10范围内均显著优于基线，$\lambda=1$为最优。
  - 多CoT采样：TRACT单CoT推理即稳定优于标准解码多CoT采样，且方差更小（Figure 3）。

## 相关工作脉络
- **Prometheus系列 (Kim et al., 2024a,b)**：LLM-as-a-Judge的CE+CoT微调基准。本文与其定位差异在于：用CoT-RAFT替代CE，使训练目标与回归评估指标对齐，无需额外成对偏好数据即可超越。
- **RAFT / RAIL (Lukasik et al., 2024, 2025)**：回归感知推理与微调的开山之作。本文扩展方向：引入CoT条件化，使回归预测具备可解释的细粒度推理能力，填补了数值预测与解释性评分的空白。
- **自生成数据微调 (Yang et al., 2024; Ren et al., 2024)**：主张自生成数据可降低perplexity并提升泛化。本文与之差异：明确指出“数据自生成≠CE微调有益”，必须配合回归感知损失才能发挥自CoT优势。
- **Critique-out-loud (CLoud, Ankner et al., 2024)**：基于成对奖励数据的自CoT数值预测模型。本文对比证明：点式Judge模型无法直接迁移至成对排序，反之亦然；但本文模型可在RewardBench上取得合理分数，揭示两类任务的潜在桥梁。
- **Re-ReST / REST (Zelikman et al., 2022; Gulcehre et al., 2023)**：自我强化/蒸馏提升推理能力。本文聚焦数值回归而非纯符号推理，且强调多目标联合优化优于串行单目标微调（§6.3）。

## 局限性与未来方向
- **依赖概率访问**：RAFT与RAIL需计算完整token分布以期望求和，对封闭API或商业模型不可直接应用。
- **推理算力未充分Scaling**：当前CoT-RAIL仅采样单次，多CoT平均时收益有限，未能充分利用推理时计算（test-time compute）扩展性。
- **未来方向**：探索无需概率分布的代理回归损失（如标量头+对比学习）；将两阶段自CoT策略推广至多轮对话与长文本评分；研究CoT采样次数与最终相关系数的 Scaling Law。

## 研究启发与可借鉴点
- **联合损失设计范式**：将生成式CE与回归平方误差绑定优化，可同时保证“推理连贯性”与“数值准确性”，适用于任何需Chain-of-Thought辅助的标量/连续值预测任务。
- **两阶段分布对齐策略**：用初步模型生成训练数据时，第二阶段必须从原始种子模型重新初始化，避免将阶段性偏移固化；该设计可直接迁移至Self-Instruct、Self-Distillation等流程。
- **数据-损失耦合验证意识**：本文反直觉地证明“自生成数据+CE有害”，提醒后续研究在引入合成数据时必须同步校验损失函数的匹配性，避免盲目套用CE范式。
- **跨任务泛化评估**：在单一任务分布上训练却能在另一范式（点式→成对）基准上有效，提示单一评估视角可能低估模型能力，建议在论文中补充跨范式测试以增强说服力。

## 关键术语表
- **LLM-as-a-Judge**：利用大语言模型依据细粒度评分规则（Rubrics）对文本输出进行自动质量评估的范式。
- **CoT (Chain-of-Thought)**：在输出最终答案前引导模型生成逐步推理过程的字符串序列。
- **RAFT (Regression-Aware Fine-tuning)**：将贝叶斯最优期望值决策直接嵌入平方误差损失进行微调的方法，使训练目标与回归指标对齐。
- **RAIL (Regression-Aware Inference for Language models)**：基于最小贝叶斯风险解码的推理策略，通过对候选数值输出加权平均降低平方误差。
- **CoT-RAFT**：本文提出的联合训练目标，同时优化CoT生成的CE损失与条件化RAIL的平方误差损失。
- **Distribution Shift**：训练阶段使用的CoT来源（如GPT-4标注）与推理阶段模型自生成CoT之间的概率分布差异。

## 可复现要素
- **数据集**：训练集 Feedback Collection（约100K，GPT-4生成）；测试集 Feedback Bench、FLASK、Vicuna Bench、MT Bench、RewardBench。Feedback Collection为开源数据集，其余基准均公开可下载。
- **代码/权重**：模型以 Apache 2.0 协议开源（论文已声明 release）；训练基于 Llama-Factory，推理基于 vLLM；具体权重下载地址见论文附属材料或ACL官方页面。
- **关键超参**：LoRA rank=8，作用所有线性层；学习率 $1.0 \times 10^{-5}$，cosine scheduler，warmup ratio=1.0；训练2 epochs；$\lambda=1$；bf16精度；有效batch size=8；单卡 NVIDIA RTX A6000。推理：top-p=0.9，temperature=1.0，repetition penalty=1.03，max output tokens=1
