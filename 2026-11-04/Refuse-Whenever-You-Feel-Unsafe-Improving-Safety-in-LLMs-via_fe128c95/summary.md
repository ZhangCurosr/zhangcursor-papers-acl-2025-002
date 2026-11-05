---
title: "Refuse-Whenever-You-Feel-Unsafe-Improving-Safety-in-LLMs-via"
source: https://aclanthology.org/2025.acl-long.158.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 01:00:43"
---

# 论文速读：Refuse-Whenever-You-Feel-Safe-Improving-Safety-in-LLMs-via-Decoupled-Refusal-Training

## 一句话总结
本文针对安全微调数据中普遍存在的“拒绝位置偏差”问题，提出了解耦拒绝训练（Decoupled Refusal Training, DeRTa），通过拼接有害响应前缀与逐位置强化过渡优化，使大语言模型能够在生成交全不安全内容的任意位置主动切断并拒绝，显著提升了对多种黑盒/白盒攻击的防御能力，同时保持模型的有用性不下降。

## 研究问题与动机
1. **拒绝位置偏差（Refusal Position Bias）**：现有安全微调数据中，拒绝词（如"Sorry"、"I cannot"）几乎全部集中在响应的极前端（前5个token），导致模型一旦开始生成危险内容，就缺乏在中途或后续位置切换为拒绝的机制。
2. **早期决策信息不足**：仅在生成前基于原始查询做拒绝判断，缺少足够的上下文信息；例如CodeAttack将恶意意图隐藏在代码续写流程中，早期拒绝难以感知风险。
3. **单次过渡训练不充分**：现有前缀增广类方法仅在有害前缀与安全响应的交界处进行一次过渡，无法覆盖完整有害序列，面对需连续生成的复杂攻击（如CompletingAttack、CodeAttack）时防御脆弱。
4. **数据视角被忽视**：当前jailbreak防御多聚焦于提示工程、安全解码或外部检测器，对安全微调数据本身的分布缺陷挖掘不足，缺乏从生成动力学层面强化全程防御的研究。

## 核心贡献（创新点）
1. **首次系统量化并定义“拒绝位置偏差”**：通过统计BeaverTails数据与LLaMA3响应，指出当前安全对齐方法仅在输出首 few tokens 生效的本质缺陷，与以往聚焦解码或提示的方法形成视角差异。
2. **提出MLE with Harmful Response Prefix**：在安全响应前随机拼接一段有害响应作为前缀，使模型获得风险上下文并被迫学习在非起始位置进行拒绝；与Qi et al. (2025)的简单前缀增广相比，本文将其作为解耦框架的第一个独立组件并严格验证其对OOD攻击的收益。
3. **提出Reinforced Transition Optimization (RTO)**：设计辅助损失，强制模型在完整有害响应序列的每一个位置$t$都预测拒绝token（如"sorry"），实现从“潜在危害”到“安全拒绝”的多次强化过渡；与DPO等序列级偏好优化方法相比，RTO显式建模了token级安全转移路径。
4. **提出CompletingAttack并构建覆盖多类攻击的综合评测**：构造移除格式token的声明式续写攻击以暴露模板依赖漏洞，并在LLaMA3/Mistral系列上验证DeRTa的通用性与跨尺寸泛化能力。

## 方法详解
- **标准安全微调损失**：$\mathcal{L}_{\text{safe}}(\theta) = -\mathbb{E}_{(q,r)\sim\mathcal{D}}\sum_{i=1}^{n}\log P_\theta(r_i|q,r_{<i})$，仅训练模型从查询直接生成安全响应，拒绝token天然聚集于起始位置。
- **数据构造**：每个DeRTa训练样本为三元组 $(q^i, r^i, \hat{r}^i)$，其中$q^i$来自BeaverTails有害查询，$r^i$由GPT-3.5-turbo生成安全回复，$\hat{r}^i$由恶意微调的LLaMA3-8B-Instruct生成有害回复。
- **MLE with Harmful Response Prefix**：截取有害响应的随机前$k$个token（$k \in [0, |\hat{r}|]$）拼接为$\hat{r}_{<k}$，损失为 $\mathcal{L}_{\text{prefix}} = -\mathbb{E}[\log P_\theta(r | q, \hat{r}_{<k})]$。该设计提供风险上下文，并使模型适应在响应任意位置承接并转折。
- **Reinforced Transition Optimization (RTO)**：在有害序列的每个位置$t$计算过渡到拒绝token的交叉熵：$\mathcal{L}_{\text{RTO}} = -\mathbb{E}[\sum_{t=1}^{|\hat{r}|} \log P_\theta(\text{sorry}| q, \hat{r}_{<t})]$。模型在每个潜在危害点都练习“刹车”，共强化$|\hat{r}|$次。
- **总损失函数**：$\mathcal{L}(\theta) = \mathcal{L}_{\text{prefix}} + \mathcal{L}_{\text{RTO}}$。有害token不参与梯度回传，防止模型学习生成有害内容。
- **训练配置**：总batch size=128，epochs=2；小模型（Mistral-7B、LLaMA3-8B）全参数微调（lr=2e-5, warmup=0.03, max_len=1024）；大模型（LLaMA3-70B、Mistral-MoE）采用LoRA（rank=96, alpha=16, lr=1e-4, max_len=512），仅作用于attention与mlp层。

## 实验与结果
- **数据集与基线**：安全数据来自BeaverTails，有用性数据来自Evol-Instruct；对比基线包括Vanilla安全训练、Goal-Priority、SoFA、RecAug与DPO；攻击场景涵盖CodeAttack、PAIR、JailbreakChat、SelfCipher、CompletingAttack、AutoDAN。
- **主要安全结果**：
  - LLaMA3-70B平均ASR从70.6%降至8.8%，有用性（GSM8K 78.6→77.6, MMLU 70.2→70.4, Alpaca 97.0→96.3）几乎无损。
  - Mistral-MoE (8×7B)平均ASR从79.1%骤降至8.7%，Alpaca评分仅微降（70.0→70.3）。
  - 对已指令微调的LLaMA3-70B-Instruct，DeRTa作为互补策略可将ASR进一步降至2.2%。
- **对比基线优势**：在CompletingAttack上ASR仅为4.0%（最优基线RecAug为25.0%）；在CodeAttack上ASR为21.5%（最优基线SoFA为73.0%），显著优于所有对比方法。
- **消融与机制分析**：
  - 仅加有害前缀可将CompletingAttack ASR从90.5%降至25.0%；加入RTO后CodeAttack ASR大幅下降，证明token级过渡建模对OOD攻击至关重要。
  - 拒绝位置分布：Vanilla仅20%拒绝发生在非起始位置
