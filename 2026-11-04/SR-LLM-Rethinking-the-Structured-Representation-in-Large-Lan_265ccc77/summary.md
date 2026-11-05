---
title: "SR-LLM-Rethinking-the-Structured-Representation-in-Large-Lan"
source: https://aclanthology.org/2025.acl-long.172.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 01:02:01"
---

# 论文速读：SR-LLM: Rethinking the Structured Representation in Large Language Model

## 一句话总结
提出 SR-LLM 框架，通过“结构到自然语言描述（SR-NLD）的免训练转换”与“Gen-SR 混合数据集监督微调”两条路径，解决大语言模型难以理解抽象符号化结构化表示的问题；在 PAWS 等基准上实现 +3.17%（免训练）与 +12.38%（微调）的性能跃升，首次实证证明合理的结构化信息注入可实质增强 LLM 推理能力。

## 研究问题与动机
- **核心问题**：结构化表示（SR，如 AMR、PST、FOL）在经典 NLP 中作用明确，但在 LLM 时代如何有效融入仍属开放难题。
- **现状不足**：直接将 SR 以代码/符号格式拼入 Prompt（如 AMRCOT）反而造成性能衰退（PAWS 下降 -5.18%），说明 LLM 对非训练语料中的抽象结构存在理解障碍。
- **假设提出**：性能下降源于结构信息以 LLM 不熟悉的符号格式传递，干扰而非辅助了模型的推理过程。
- **研究目标**：探索将结构化表示转化为 LLM 可消化格式的最佳路径，涵盖 Prompt 级免训练适配与参数级微调两种范式。

## 核心贡献（创新点）
- **提出 SR-LLM 双范式集成框架**：同步提供免训练的 SR-NLD 提示策略与基于 Gen-SR 混合数据的监督微调策略，与以往仅做符号直投或单一路径探索的工作形成本质差异。
- **揭示多类结构化表示对 LLM 的性能影响规律**：系统对比 AMR（语义）、PST（句法）、FOL（逻辑）及其组合，发现语义+逻辑对较弱模型增益最优，而句法结构有时会产生负向干扰，为后续结构选型提供依据。
- **首次实证结构化表示可显著提升 LLM 推理上限**：在 10 项 NLP 基准与 SuperGLUE 上全面超越 AMRCOT、SENSE 等基线，且该方法与 CoT/TOT/Self-reflection 推理范式正交可叠加，填补了“结构知识→LLM 推理增益”的空白。

## 方法详解
- **SR-to-NLD 免训练转换**：以 AMR 为例，先将 AMR 图经 Penman 库解析为三元组 `(c1, r, c2)`，替换标识符为实际词项，再通过预定义规则字典映射为多句自然语言描述，最后由 LLM（GPT-4o Mini）润色并引入多轮投票机制抑制幻觉；PST 与 FOL 分别采用深度优先搜索线性化+符号映射、谓词/量词/联结词映射+LLM 润色的相似流程，最终将 SR-NLD 嵌入 Prompt 作为额外上下文。
- **Gen-SR 混合微调**：构建包含纯文本指令对 `G(text)` 与附带结构化表示的指令对 `G(SR)` 的联合微调数据集；对 Llama3.1-8B-Instruct 进行 10 任务联合 SFT，通过消融确定最优混合比例（AMR/PST 为 50% G(text) + 50% G(SR)，FOL 为 30% G(text) + 70% G(SR)）；推理时配合含 SR 的 Prompt 可获得最佳表现。
- **设计原则**：强调“结构保真+语言可读”，避免直接输入抽象符号；通过投票机制与 Gold/Flawed 质量对照验证转换可靠性，确保结构化先验在零样本与微调场景下均能有效转化为推理优势。

## 实验与结果
- **数据集与基线**：覆盖 PAWS、SNLI、WMT16、CoNLL2003、Logic、SST-2、Pubmed45、WiC、SPIDER、AGNEWS 及 SuperGLUE；基线包括 AMRCOT、AMRCoC、SENSE、COT、TOT、Self-reflection 及文本复述增强。
- **训练无关结果**：Llama3.1-8b-Instruct 上，SR-NLD 在 PAWS 取得 44.77 F1（较 baseline 41.59 提升 **+3.17%**），全指标稳定优于直接输入原始 SR；模型越弱受益越大，GPT-4o-mini 提升幅度较小且部分任务持平。
- **训练依赖结果**：50% G(SR)+50% G(text) 微调后，PAWS 达 81.04 F1，较同配置仅文本微调（68.66）提升 **+12.38%**；SuperGLUE 平均提升 +6.43，70B 模型在 5 项任务上全部正收益（PAWS +9.81%）。
- **质量与对比验证**：Gold AMR 配合 NLD 增益显著，Flawed AMR 则引发性能下滑；显著优于单纯文本复述（+1~+3  paraphrase 波动仅 -2%~+1%），且可与现有推理增强方法叠加使用。

## 相关工作脉络
- **AMRCOT (Jin et al., 2024)**：将原始 AMR 直接嵌入 CoT Prompt，本文以其为对照基线，通过 NLD 转换成功逆转其导致的性能衰退。
- **AMR-to-Text 与符号规范化 (Song et al., 2018; Ribeiro et al., 2021; Shin et al., 2021)**：侧重生成语法/语义完整的单句文本；本文 SR-to-NLD 强调多句协同描述结构信息，保留结构完整性同时提升 LLM 可读性。
- **语义提示/Magic Prompts (An et al., 2024)**：仅利用语义解析结果
