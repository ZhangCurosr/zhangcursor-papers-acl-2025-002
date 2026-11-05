---
title: "Self-Instructed-Derived-Prompt-Generation-Meets-In-Context-L"
source: https://aclanthology.org/2025.acl-long.92.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 01:02:00"
field: "提示工程与大语言模型对齐"
keywords: ["prompt refinement", "in-context learning", "reinforcement learning", "black-box LLM", "prompt optimization", "self-instruct"]
innovations: ["提出自指令强化学习目标，无需数据收集即可优化derived prompt生成模型", "设计意图一致性ICL查询框架，通过衍生prompt-响应对构建演示样例而不替换原始prompt", "DPG指令替代SFT阶段，充分利用预训练LLM的内在指令遵循能力"]
benchmarks: ["Vicuna Eval", "BPO-test Eval", "Dolly Eval", "Self-Instruct Eval"]
---

# 论文速读：Self-Instructed-Derived-Prompt-Generation-Meets-In-Context-L

## 一句话总结
本文提出一种基于自指令强化学习的derived prompt生成框架，结合上下文学习（ICL）机制，在不替换原始prompt的前提下，为黑盒LLM（如GPT-4）自动生成高质量推导prompt与演示样例，从而在保持语义一致性的同时显著提升模型响应质量。

## 研究问题与动机
- 现有prompt精炼方法（如BPO、PAS）直接以改写后的prompt替换原始prompt，容易引入语义偏差，偏离用户真实意图（如过度收窄话题、丢失原始上下文）。
- 黑盒LLM（如GPT-4）无法进行微调，仅能通过改进输入prompt来提升响应质量，而当前方法缺乏与响应模型（response model）的交互对齐。
- 传统RLHF优化prompt重写模型需要大量人工标注数据，且难以直接应用于不可访问的黑盒模型。
- 如何在无数据收集、无模型微调的前提下，既保证语义一致性又激发LLM的上下文学习能力，是一个尚未被有效解决的核心问题。

## 核心贡献（创新点）
- **自指令强化学习目标**：设计了一种无需手动收集数据的自指令RL目标，利用reward model和response model直接驱动derived prompt生成模型的优化，避免了传统SFT阶段对训练数据的依赖。
- **DPG指令设计**：通过精心设计一条derived prompt generation（DPG）指令，充分利用预训练LLM内在的指令遵循与改写能力，作为无需额外训练的参考策略，替代了传统的SFT初始化。
- **意图一致性ICL查询框架**：将derived prompt与其响应组合为上下文演示样例，嵌入ICL模板中以原始prompt进行查询，在保持用户原始意图的同时激活模型的内在知识，缓解语义不一致风险。
- **跨模型通用性验证**：实验表明该方法可无缝适配黑盒模型（GPT-3.5/GPT-4）与白盒模型（Llama2/Llama3/Qwen2），并在多个指令数据集上持续提升响应质量。

## 方法详解
- **任务定义**：给定原始prompt $x$，训练一个生成模型 $\pi_\theta$，输出语义相关且表达更优的derived prompt $x' \sim \pi_\theta(x'|x)$；冻结响应模型 $\mathcal{M}$ 以生成响应 $y' \sim \mathcal{M}(y'|x')$。
- **自指令RL目标**（公式3/5）：
  $$\mathbb{E}_{x\sim\mathcal{D}}\left[\mathcal{R}(x', y') - \beta \log\frac{\pi_\theta(x'|x)}{\pi_{\text{ref}}(x'|x)}\right]$$
  其中 $\mathcal{R}$ 为reward model对$(x', y')$评分，$\pi_{\text{ref}}$ 为初始化的参考模型（即DPG指令下的LLM），$\beta=0.05$ 控制KL正则强度。
- **DPG指令**：手动设计的模板引导LLM将原始问题改写为"更全面、更易理解、更具可回答性"的版本，同时不限制回答视角。该指令使模型具备基础的改写能力，免去了SFT训练阶段。
- **推理阶段的ICL模板**（Algorithm 2）：
  ```
  ### Question: {derived prompt x'}
  ### Response: {response y'}
  Given the above Question and Response as an example, emulate the way it responds to the question to reply to the following question:
  ### Question: {original prompt x}
  ```
  以$(x', y')$为演示样例，引导模型以相同风格和质量回答原始问题$x$。

## 实验与结果
- **训练数据**：BPO训练集，共14K多样本（从4个精选数据集构建）。
- **评估数据集**：Vicuna Eval、BPO-test Eval、Dolly Eval、Self-Instruct Eval（各含100-252个样本）。
- **基线方法**：OP（原始prompt）、BPO、PAS、Self-Refine。
- **生成模型 $\pi_\theta$**：Llama3-8B-Instruct、Llama2-7B-Chat、Qwen2-7B-Instruct。
- **响应模型**：GPT-3.5-turbo、GPT-4o、Llama2-7B-Chat、Llama3-8B、Qwen2-7B。
- **评估指标**：以GPT-4作为裁判（MT-bench风格），记录Win Rate（A Win / B Win / Tie）。
- **主要结果**：
  - 在Vicuna Eval上，Llama3作为$\pi_\theta$时，OURS vs OP的Win Rate达90.0%（GPT-4为query model），显著优于BPO（88.8%）。
  - 对GPT-4的整体平均WR（vs OP）达67.1%，vs BPO达56.1%；对GPT-3.5分别达74.3%和69.9%。
  - Llama2作为$\pi_\theta$时，对BPO的平均WR分别为39.63%（Llama3）、48.3%（Llama2）、64.3%（Qwen2）。
  - **消融**：仅有OD（self-instructed RL）即可超越BPO和PAS；加入ICL后进一步显著提升；BPO/PAS+ICL也可受益，证明ICL为通用增强模块。
  - **无需训练的OD+ICL**即可带来39.1%的WR提升（vs OP），说明ICL框架本身具有很强的通用性。
- **效率**：OURS推理时长约16.18秒（vs OP的12.06秒），数据收集量为0（vs BPO/PAS需人工收集）。

## 相关工作脉络
- **BPO (Cheng et al., 2023)**：基于SFT的prompt精炼方法，需收集大量$(x, x')$对，且直接替换原始prompt导致语义偏移；本文通过RL+ICL避免此问题。
- **PAS (Zheng et al., 2024b)**：插件式prompt增强系统，依赖复杂的数据选择与互补数据集生成；本文方法更轻量且无需数据收集。
- **Self-Refine (Madaan et al., 2023)**：迭代自我反馈方法，缺乏外部监督信号；本文通过reward model提供明确优化方向。
- **RLHF（Ouyang et al., 2022）**：传统RLHF需奖励模型和PPO训练，成本高；本文将其思想迁移至prompt生成而非响应生成，降低了资源门槛。
- **In-Context Learning（Liu et al., 2021; Dong et al., 2022）**：ICL效果依赖于演示样例的选择；本文自动构造高质量、语义相关的ICL演示，是对ICL示例选择的有效扩展。
- **Prompt Rewriting with RL（Kong et al., 2024; Huang et al., 2024）**：同样采用RL优化prompt，但依赖SFT初始化及大量$(x, x')$训练数据；本文通过DPG指令消除SFT需求。

## 局限性与未来方向
- 当前方法仅在one-shot演示设置下验证，n-shot场景有待探索。
- 在数学和编程等推理密集型任务中，自动生成高质量、语义相似的演示样例较为困难，目前方法尚未直接适配。
- RL训练仍带来一定计算开销，且推理阶段需额外查询LLM生成演示样例。
- 未来方向包括：探索高效的n-shot演示构造、共享演示样例以降低生成成本、以及扩展至数学/代码推理领域。

## 研究启发与可借鉴点
- **自指令RL设计**：利用预训练LLM的内在指令遵循能力替代SFT阶段，为无数据训练的模型优化提供了新思路，可迁移至其他文本生成/改写任务。
- **ICL演示的自动生成策略**：通过derived prompt-response对构造语义相关的ICL样例，这一框架可作为通用模块集成到现有prompt精炼方法中（如BPO+ICL验证有效）。
- **意图一致性约束**：以ICL模板保留原始prompt作为最终查询，而非直接替换，这一"不替换、只补充"的设计哲学对避免语义漂移具有重要参考价值。
- **Reward-based prompt优化**：将reward signal引入prompt生成而非响应生成的思路，可拓展至更多需要高质量输入指令的场景（如自动化测试生成、代码补全提示等）。

## 关键术语表
- **Derived Prompt**：由原始prompt转化而来的衍生prompt，保持语义相关性且表达更优，但不直接替换原始prompt，而是作为ICL演示的一部分使用。
- **Self-Instructed RL**：一种无需人工标注数据的强化学习训练目标，通过模型自身生成样本来优化，结合reward信号指导prompt生成过程。
- **ICL (In-Context Learning)**：在模型输入中提供若干示例（演示样例），使模型能够基于上下文快速适应新任务的能力。
- **DPG Instruction**：Derived Prompt Generation Instruction，一条手动设计的指令模板，引导LLM将原始问题改写为更全面、更易回答的版本。
- **Reward Model**：用于评估(prompt, response)对质量的模型，本文使用基于hh-rlhf数据集训练的SOTA reward model。
- **Black-Box LLM**：指接口不可访问的大型语言模型（如GPT-4），无法进行微调，只能通过prompt工程来提升性能。

## 可复现要素
- **训练数据集**：BPO训练集（14K样本），由4个精选数据集构建，论文未声明公开链接。
- **评估数据集**：Vicuna Eval、BPO-test Eval、Dolly Eval、Self-Instruct Eval，均为公开指令数据集。
- **代码开源**：论文未明确提及代码仓库链接，仅提到"experiment result file"上传。
- **模型权重**：使用Llama2-7B-Chat、Llama3-8B-Instruct、Qwen2-7B-Instruct作为$\pi_\theta$，以及GPT-3.5/GPT-4作为响应模型。
- **关键超参**：学习率$1\times10^{-6}$，$\beta=0.05$，温度$\tau=0.0$，top-p=0.9，batch size=1，训练2个epoch，最大生成长度1024 tokens，使用4×A100 GPU + DeepSpeed ZeRO-2。
