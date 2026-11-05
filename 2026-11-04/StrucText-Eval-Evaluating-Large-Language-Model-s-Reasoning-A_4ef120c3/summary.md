---
title: "StrucText-Eval-Evaluating-Large-Language-Model-s-Reasoning-A"
source: https://aclanthology.org/2025.acl-long.11.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 01:03:00"
field: "大语言模型评测"
keywords: ["结构化文本理解", "大语言模型评估", "推理基准", "自动数据生成", "提示工程"]
innovations: ["提出支持8种语言29种任务的结构化文本自动评测基准", "通过Depth/Width/Col三参数可控生成不同复杂度任务数据", "语义无关的任务设计迫使模型依赖结构而非语义进行推理"]
benchmarks: ["StrucText-Eval", "StrucText-Eval-Test", "StrucText-Eval-Real-Test", "StrucText-Eval-Test-Hard"]
---

# 论文速读：StrucText-Eval-Evaluating-Large-Language-Model-s-Reasoning-A

## 一句话总结
本文提出 StrucText-Eval，一个自动生成的评估基准，用于测试 LLM 在8种结构化语言、29类任务上的推理能力；结果表明现有开源模型在标准测试上最高仅达 74.9%，在 Hard 测试上骤降至 45.8%，而人类可达 92.6%，揭示了 LLM 处理复杂结构信息的显著不足。

## 研究问题与动机
- **核心问题**：LLM 能否直接理解无结构文本呈现的结构化数据（如 JSON、LaTeX、XML 等的原始文本形式）？
- **现有方法不足**：
  1. 现有基准仅覆盖图、表格、JSON 等有限类别，无法全面评估结构化文本理解；
  2. 多数基准依赖人工标注数据，成本高且可能引发模型"作弊"（Zhou et al., 2023）；
  3. 当前工作将结构化数据作为分类或语义任务的辅助输入，而非独立考察纯结构推理。

## 核心贡献（创新点）
1. **自动生成结构化评估数据的方法**：通过控制嵌套深度（Depth）、宽度（Width）和列数（Col）调节任务复杂度，支持8种语言与29种任务——与已有工作依赖人工标注的本质区别在于其可扩展性。
2. **构建 StrucText-Eval 基准（5,800 样本）**：划分为 Test（3,712）、Real-Test（928，真实场景改写）、Test-Hard（2,088，最长样本超10万字符）——区别于已有基准只覆盖单一或少数格式。
3. **语义无关的任务设计**：29 个任务刻意去除语义内容，迫使模型从符号结构中推断——与 GraphQA、Struc-Bench 等依赖语义的表格/图推理基准本质不同。
4. **全面的 Prompt 策略与多模型评估**：系统测试 Naive、Self-CoT、PS-CoT、w/ Hint、Few-Shot 六种提示策略，揭示训练数据偏差对性能的影响机制。

## 方法详解
**数据集生成流程（以 JSON PathCompose 为例）**：
1. **定义复杂度参数**：指定 Depth（树深度）、Width（每个非叶节点的子节点数）、Col（每节点关联字段数）及语言类型、任务类型。
2. **构造抽象结构树**：按参数生成抽象的节点与边关系。
3. **模板检索与翻译**：根据任务从29条规则/模板中检索对应问题模板，再将抽象树翻译为选定语言的具体格式。
4. **Ground Truth 提取**：根据规则自动生成参考答案。

**三种测试集构成**：
- **Test**：规则生成的纯抽象数据；
- **Real-Test**：从 Test 中抽取等比例样本，由5名计算机背景研究生替换为真实场景语义（如运动项目、玻璃器皿规格等），保持结构一致性；
- **Test-Hard**：长度大幅扩展（平均16,535字符，最长102,531字符），结构嵌套更深（Depth 最大3、Width 最大3）。

**任务类别（29种）**：涵盖 Syntax（SY）、PathWalk（PW）、TextRetrieval（TR）、Statistic（ST）、Join（JO）、Tree.Height（TH）、Node.Depth（ND）、PathCompose（PC）等8大类。

**评估指标**：主指标为 RougeL（截断阈值0.75以下计0分），辅以 LLM-as-Judge、BLEU、Exact Match，并与人工评估做相关性分析。

## 实验与结果
**实验设置**：
- 开源模型：Qwen2-7B/72B、Llama-3.1-8B/70B/405B、Mistral-0.2-7B
- 闭源模型：GPT-4o/4o-mini、Gemini-1.5-pro/flash、GLM-4-Plus/Flash
- Prompt 策略：Base、Self-CoT、PS-CoT、w/ Hint、3-Shot、Simple 3-Shot

**主要结果**：
- **标准测试（Test）**：Qwen2-72B 在 Naive 提示下最高达 74.9%（RougeL），JSON 任务表现最优（85.8%），LaTeX 表现最差（64.6%）；
- **Hard 测试**：所有模型性能显著下降，Qwen2-72B 降至约 42.5%，最佳闭源模型 GPT-4o 仅 51.1%；
- **人类表现**：Human 在 Hard 测试上达 92.6%（表4）/ 95.7%（正文），远超所有模型，凸显结构推理鸿沟；
- **Prompt 策略影响**：w/ Hint 对复杂任务（YAML、JSON）提升明显（如 Llama-3.1-70B 从 75.4% 提升至 84.9%），但 Self-CoT/PS-CoT 在复杂结构下反而下降；
- **训练数据偏差**：JSON 任务表现最优（训练集中高频出现），LaTeX/XML 较弱，反映模型对格式的训练数据偏见。

## 相关工作脉络
1. **GraphQA / Struc-Bench**：评估图推理和表格文本，本文将其扩展至8种结构化语言，且任务设计剥离语义依赖；
2. **TEMPTABQA / TableLLM**：专注表格时序推理，本文覆盖更广泛的树形、嵌套、跨格式结构；
3. **BIG-Bench (JSON子集)**：仅含 JSON 31,907条，本文覆盖8种语言且包含真实场景数据；
4. **DocCGen**：聚焦文档代码生成，本文关注纯结构推理而非生成；
5. **SUC**：仅评估表格理解，本文任务类别更丰富（29种），且强调结构复杂度可控性。

## 局限性与未来方向
- **语言覆盖有限**：仅8种结构化语言，现实中大量自定义格式未覆盖；
- **任务种类有限**：29种任务无法穷尽所有结构化推理场景；
- **缺少部分顶级模型**：因区域限制未包含 Claude 系列作为基线；
- **结论基于 GPT-4 代表顶级**：难以验证是否存在更优模型；
- **未来方向**：可扩展更多实际语言类型，引入推理过程评估（而非仅结果匹配），探索领域自适应微调策略。

## 研究启发与可借鉴点
1. **可控复杂度的自动生成方法**：通过 Depth/Width/Col 三参数调节难度，可迁移至其他结构化数据评测任务；
2. **Real-Test 设计思路**：在规则生成数据基础上注入真实语义，既保留结构多样性又贴近实际应用场景，可作为通用评估增强范式；
3. **语义无关的任务构造**：剥离语义迫使模型依赖结构本身，可有效检测模型的"表面拟合"问题，适用于各类推理基准设计；
4. **Prompt 策略的系统对比**：w/ Hint 对复杂结构有效，但 CoT 类方法在深度嵌套时反而下降，提示工程需结合任务结构复杂度进行适配。

## 关键术语表
- **StrucText-Eval**：本文提出的评估 LLM 结构化文本推理能力的基准测试，含5,800个预生成样本。
- **Depth（嵌套深度）**：结构树的最大层级数，用于控制任务复杂度。
- **Width（宽度）**：每个非叶节点的子节点数量，影响分支复杂度。
- **Col（列数）**：每个节点关联的字段数量，增加信息密度。
- **RougeL**：基于最长公共子序列的文本相似度指标，本文主评估指标。
- **Test-Hard**：StrucText-Eval 的高难度子集，样本更长、结构更深，平均16,535字符。
- **w/ Hint**：向模型注入人工构造的规则提示，以补偿其结构推理能力的评估策略。
- **Semantic-agnostic Tasks**：刻意去除语义内容、仅依赖符号结构进行推理的任务设计原则。

## 可复现要素
- **数据集**：StrucText-Eval（Test + Real-Test + Test-Hard），已开源：https://github.com/MikeGu721/StrucText-Eval
- **代码**：数据集生成代码已开源
- **模型**：测试的6个开源模型和6个闭源模型均可通过官方渠道获取
- **关键超参**：temperature=0.95, top_p=0.95, top_k=5, num_beams=2, max_new_tokens=1（附录 Table 11）
- **提示模板**：Naive/CoT/Few-Shot/Hint 六类模板见附录 Section E 及 Tables 7-10
