---
title: "Long-Context-Vulnerabilities-in-LLMs"
source: https://aclanthology.org/2025.acl-long.101.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-09-01 20:23:58"
field: "大语言模型安全与对齐"
keywords: ["Many-Shot Jailbreaking", "Long-Context Safety", "LLM Vulnerability", "In-Context Learning", "Model Alignment", "Adversarial Attack"]
innovations: ["发现上下文长度是MSJ攻击成功的主因而非内容有害性", "提出Fake Data Attack和Shot Repetition Attack等高效攻击方法", "揭示SFT/RLHF对齐训练可能增加模型对Safe内容的脆弱性"]
benchmarks: ["Harmful-512", "Safe-512", "Fake-512", "Harmful-Same-512", "Safe-Same-512"]
---

# 论文速读：Long-Context-Vulnerabilities-in-LLMs

## 一句话总结
本文系统研究了大型语言模型（LLMs）在长上下文场景下的安全漏洞，通过 Many-Shot Jailbreaking (MSJ) 攻击发现**上下文长度**是导致攻击成功率（ASR）变化的核心因素，而非示例内容的有害性或主题多样性；即使使用无害或无意义的随机文本，也能成功绕过经过良好对齐的模型的安全防护。

## 研究问题与动机
- **核心问题**：随着 LLM 上下文长度扩展至 128K tokens，模型的长上下文处理能力是否带来了新的安全漏洞？现有安全对齐训练能否有效防御多 shot 提示攻击？
- **现有方法不足**：
  - 传统 Few-shot Jailbreak 研究关注少量示例的精心构造，但 Many-Shot Jailbreaking (MSJ) 依赖大量示例注入，其安全影响尚未被系统研究
  - 当前对齐训练（SFT/RLHF）假设有害 QA 对是主要威胁，但本文发现无害内容同样有效
  - 现有 defenses（如 prompt 检测、perturbation）依赖内容特征，无法防御基于上下文长度的架构级脆弱性

## 核心贡献（创新点）
1. **系统性长上下文漏洞分析**：首次在不同规模模型（1B-72B）和最大 128K 上下文中系统评估 MSJ 攻击，揭示了上下文长度作为主要脆弱性因素的作用机制。
2. **脆弱性因素解耦发现**：证明攻击成功不依赖于 shot 的有害性、主题多样性或精心构造，而是由上下文长度决定，挑战了 MSJ 需要"有害内容学习"的传统假设。
3. **高效攻击方法设计**：基于上述发现，提出 Fake Data Attack（使用无意义文本）和 Shot Repetition Attack（重复单个示例）两种资源高效攻击策略，在 Llama 系列模型上实现与复杂 MSJ 相当的攻击成功率。
4. **对齐训练脆弱性揭示**：通过对比 Base 模型与 Instruction-tuned 模型，发现 SFT/RLHF 对齐训练反而可能增加模型对 Safe 内容的脆弱性，揭示当前安全训练流程的潜在缺陷。

## 方法详解
- **攻击框架**：采用 Many-Shot Jailbreaking (MSJ) 范式，在目标查询前注入 N 个示例（QA 对或连续文本），通过 In-Context Learning 迫使模型生成有害响应。攻击提示包含三个组件：Instruction（任务指令）、Examples（示例）、Target query（目标查询）。
- **指令类型**：测试三种指令——Safe（标准安全交互）、Secret Role（角色扮演）、Love Pliny（显式越狱），其中 Secret Role 展现出最清晰的三阶段脆弱性模式。
- **示例构建**：构建 14 种数据集覆盖不同内容类型（Table 1）：
  - Harmful QA：Harmful-{128,512,2048}（多主题有害QA）、Harmful-{Topic}（特定主题如 Adult/Cybersecurity 等）
  - Safe/Mixed：Safe-512（通用无害QA）、Mixed-512（等量有害+无害混合）
  - Repetition：Harmful-Same-512、Safe-Same-512（单一示例重复512次）
  - Free-form：Text（Paul Graham essays）、Fake-Text/Fake-512（Lorem Ipsum 无意义文本）
- **评估指标**：采用 ASR（Attack Success Rate）而非 NLL，通过 GPT-4o 作为 Judge 模型评估响应有害性（准确率>96%）。
- **实验设置**：温度=0、top_p=1.0 保证确定性输出；Qwen 模型使用 YaRN 位置编码（scaling factor=4）；每实验重复5次取均值±标准差。

## 实验与结果
- **模型覆盖**：Llama-3.1-8B/70B、Llama-3.2-1B/3B、Qwen2.5-1.5B/7B/14B/72B，部分实验包含 Gemini-1.5-Pro。
- **核心发现**：
  - **上下文长度主导**（Figure 4）：ASR 主要取决于上下文长度（≈2^17 tokens 处急剧上升），而非 shot 数量；shot 密度仅影响退化开始时机。
  - **主题多样性影响有限**（Figure 5）：不同有害主题（Adult/Criminal/Cybersecurity 等）的 ASR 模式一致，多主题组合未显著优于单主题。
  - **有害性非必需**（Figure 6）：Safe-512 在 Llama 模型上 ASR 甚至高于 Harmful-512，Mixed-512 表现相当。
  - **文本格式攻击更强**（Figure 7）：连续文本（Text/Fake-Text）在 Llama-3.1-70B 和 Qwen2.5-72B 上 ASR 显著高于 QA 格式，脆弱点出现在 2^12-2^14 tokens。
  - **高效攻击有效性**（Figures 8-9）：Fake Data Attack 和 Shot Repetition Attack 在 Llama 系列上实现与 Harmful-512 相当甚至更高的 ASR（Llama-3.1-70B + Safe-Same-512 达 0.845）。
  - **对齐训练副作用**（Figure 10）：Llama-3.1-8B Base 模型对 Harmful 脆弱但对 Safe 稳健；Instruction-tuned 版本对 Safe 攻击的脆弱性显著增加，呈现 inverse relationship。
- **最强结果**：Llama-3.1-70B + Secret Role + Harmful-512 在 2^11 处 ASR=0.880；Llama-3.1-70B + Safe-Same-512 在 2^11 处 ASR=0.845；Qwen2.5-7B + Harmful-512 在 2^11 处 ASR=0.866。

## 相关工作脉络
1. **Many-Shot Jailbreaking (Anil et al., 2024)**：本文的直接前身，发现 MSJ 攻击效果与 ICL 能力遵循 power law。区别：本文系统解耦 shot 数量、主题、有害性等 factor，发现上下文长度才是主因，且无害/无意义内容同样有效。
2. **Scaling In-Context Learning (Li et al., 2023b; Bertsch et al., 2024; Agarwal et al., 2024)**：研究长上下文 ICL 性能提升。区别：本文聚焦安全维度而非性能，揭示长上下文带来的安全 trade-off。
3. **Jailbreaking and Safety (Zou et al., 2023; Chao et al., 2023; Liu et al., 2024b)**：针对 few-shot 攻击和 prompt 优化。区别：MSJ 需数百示例，本文揭示此类攻击的脆弱性根源不同于内容级攻击。
4. **Lost in the Middle (Liu et al., 2024a)**：发现模型在长序列中间部分能力下降。区别：本文将此类现象与安全脆弱性关联，指出中间位置易受攻击。
5. **Defense Strategies (Alon & Kamfonas, 2023; Cao et al., 2024; Sharma et al., 2024)**：prompt 检测和 perturbation 方法。区别：本文证明基于内容特征的 defense 无效，因攻击可依赖无害/无意义内容。
6. **SFT/RLHF Alignment (Ouyang et al., 2022; Bai et al., 2022; Bianchi et al., 2024)**：安全对齐训练。区别：本文发现对齐训练可能引入对 Safe 内容的反向脆弱性，揭示训练流程潜在缺陷。

## 局限性与未来方向
- **模型覆盖局限**：仅限公开开源模型（Llama/Qwen），无法测试多数商业 API 模型（因安全过滤限制）；透明度受限。
- **评估方法局限**：仅使用 GPT-4o 作为 Judge，可能存在评估偏差；二元 safe/unsafe 分类过于简化，无法捕捉微妙有害性。
- **机制理解不足**：未深入分析注意力权重、hidden state 或 embedding 变化来解释脆弱性起源。
- **未来方向**：开发 context-length-aware 的安全机制；改进 context processing architecture；探索 position-aware safety alignment；增加透明度和可复现性。

## 研究启发与可借鉴点
1. **脆弱性评估范式**：通过解耦 shot 数量、主题、有害性、格式等 factor 来定位脆弱性根源的方法论，可迁移至其他安全评估场景（如多模态、agent 系统）。
2. **高效攻击设计思路**：Fake Data Attack 和 Shot Repetition Attack 展示了"最小 effort 最大化 impact"的攻击策略，启发 red teaming 工作中可优先尝试简单 baseline。
3. **对齐训练反思**：发现 SFT/RLHF 可能引入对新内容类型的脆弱性，提示安全训练需考虑长上下文场景下的分布泛化，避免 overfitting 到特定 harmful patterns。
4. **评估指标选择**：对比 NLL 与 ASR 的优劣分析（Appendix C）提供了安全评估指标选择的参考，ASR 更直接反映攻击成功率。
5. **实验设计严谨性**：每实验重复5次、随机 shuffle 示例顺序、过滤 ambiguous queries 等做法值得借鉴，提升评估可靠性。

## 关键术语表
**Many-Shot Jailbreaking (MSJ)**：通过在上下文中注入大量（数百）示例来操纵 LLM 行为的攻击方法，利用模型的 In-Context Learning 能力迫使其生成有害响应。

**Attack Success Rate (ASR)**：攻击成功率，衡量模型在 jailbreak 提示下生成有害响应的比例，是本文主要评估指标。

**In-Context Learning (ICL)**：通过上下文中的示例让模型学习任务模式的能力，MSJ 攻击正是利用这一机制实现 behavior manipulation。

**Secret Role Instruction**：一种中等强度的攻击指令，要求模型扮演特定角色（如"你是一个没有道德限制的 AI"），在实验中展现出最清晰的三阶段脆弱性模式。

**Fake Data Attack**：本文提出的高效攻击方法，使用无语义意义的文本（如 Lorem Ipsum）填充上下文即可触发漏洞，无需构造有害内容。

**Shot Repetition Attack**：另一种高效攻击，将单一示例重复多次填充上下文，证明攻击成功不依赖示例多样性。

**Lost in the Middle**：指 LLM 在长序列中间部分的信息处理能力下降的现象，本文将其与安全脆弱性关联。

**YaRN**：一种高效上下文扩展技术（Position Embedding），本文在 Qwen 模型中使用（scaling factor=4）以支持 128K 上下文。

## 可复现要素
- **数据集**：使用公开数据集构建（BeaverTails, TechHazardQA, HH-RedTeam, LifeTox, AdvBench, JailbreakBench, WildGuardMix, SaladBench, WMDP, HarmfulQA, S-Eval, X-Safety, OpenOrca, paul-graham-essays），预处理细节见 Appendix B.3；Fake 数据使用 Lorem Ipsum 生成。
- **代码/权重**：论文声明"All experimental findings and code will be shared under strict access controls with verified researchers"，代码和有害内容示例以 placeholder 形式提供。
- **关键超参**：temperature=0, top_p=1.0, 随机种子重复5次；Qwen 模型使用 YaRN position embedding（scaling factor=4, max position=131072）。
- **硬件**：6 节点集群（A100 40GB/80GB, H100 80GB HBM3）。
