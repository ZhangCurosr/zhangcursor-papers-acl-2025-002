---
title: "UnSeenTimeQA-Time-Sensitive-Question-Answering-Beyond-LLMs-M"
source: https://aclanthology.org/2025.acl-long.94.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 22:11:03"
field: "时序推理与评测基准"
keywords: ["Time-Sensitive Question Answering", "Temporal Reasoning", "Data Contamination", "Benchmark", "Large Language Models", "Synthetic Data", "Parallel Events"]
innovations: ["提出基于合成物流场景的无数据污染TSQA基准UnSeenTimeQA", "首次系统性引入并行事件推理评测并揭示LLM在此任务上的显著缺陷"]
benchmarks: ["UnSeenTimeQA", "TimeQA", "TempReason", "MenatQA", "RealTimeQA", "FreshQA", "TAQA"]
---

# 论文速读：UnSeenTimeQA-Time-Sensitive-Question-Answering-Beyond-LLMs-M

## 一句话总结
论文提出了 **UnSeenTimeQA**，一个基于合成物流场景事实、完全无数据污染的时敏问答（TSQA）基准，首次系统性地评估了 LLMs 在序列与并行事件时序推理上的真实能力，揭示现有 TSQA 基准存在严重的记忆依赖问题。

## 研究问题与动机
- **数据污染问题**：现有 TSQA 基准（TimeQA、TempReason 等）基于 Wikidata/维基百科等真实世界事实，LLMs 可直接调用预训练记忆答题，而非进行真正的时序推理。
- **人工更新不可持续**：FreshQA、RealtimeQA 等通过延长知识截止日期来规避污染，但一旦停止手动更新就会被新版 LLM 的前训练数据覆盖，存在"永恒跑步"困境。
- **缺乏并行事件场景**：现有基准将事件视为严格串行，忽略现实中多事件并发（如两辆车同时装载）的推理需求，无法测试模型的并行时序处理能力。
- **评估指标单一**：缺少对推理链条质量、错误模式细粒度分析的系统评估。

## 核心贡献（创新点）
1. **证明现有基准的"记忆主导"现象**：通过 context/question 篡改实验发现 GPT-4 在改变事实后仍能维持相近准确率，揭示其依赖预训练记忆而非上下文推理。
2. **提出合成事实驱动的无污染基准**：基于 IPC 物流规划域生成合成事件序列，彻底脱离真实世界数据源，实现"按需生成"的永久污染-free 评估。
3. **引入并行事件推理子任务**：首次将"同时发生的多个事件"纳入 TSQA 评测体系，填补现有基准的结构性空白。
4. **系统误差分析框架**：识别四类核心错误模式（缺失事件、推理链不完整、幻觉、缺失并行推理），为后续改进指明方向。

## 方法详解
- **数据来源**：借用国际规划竞赛（IPC）的物流规划域，包含城市、地点、卡车、飞机、包裹等对象，六种基本事件：load/unload truck、load/unload airplane、drive truck、fly airplane。
- **事件序列生成**：使用 ActionReasoning-Bench 的数据生成管线产生合法计划，再为每个事件随机分配持续时间（分钟级），转换为自然语言模板。
- **四难度分级**：
  - **Easy**：事件提供完整起止时间戳（如 08:11 AM–08:54 PM）
  - **Medium**：提供起始时间 + 持续时间，需模型推算结束时间
  - **Hard (Serial)**：仅提供持续时间，默认事件顺序执行，需模型自行推导时间线
  - **Hard (Parallel)**：仅提供持续时间，且允许部分事件并发执行，需模型处理并发约束
- **三问法分类**：
  - **Static-Time**："某时刻包裹在何处？"
  - **Relative-Time**："某时刻前后 N 分钟包裹在何处？"
  - **Hypothetical-Time**："若某事件延迟/加速 X 分钟，某时刻包裹在何处？"
- **深度指标（Depth）**：问题相关起始事件与目标事件在序列中的位置差的绝对值（范围 6–20），用于刻画推理链长度。
- **事件追踪系统**：自动构建每个包裹从 pickup 到 delivery 的全程事件链，据此生成精确答案。

## 实验与结果
- **评测模型**：Llama-3.1-8B/70B、Gemma-2-9B/27B、GPT-4o；零样本 CoT + 少样本两套设置。
- **Easy/Medium**：大模型表现良好（Llama-3.1-70B Easy 平均 92.62%，GPT-4o Easy 平均 93.66%），显示基于显式时间戳的提取型任务对 LLMs 较容易。
- **Hard (Serial) 平均**：Llama-3.1-70B 38.38%，GPT-4o 49.77%，相比 Easy 下降约 43–54 个百分点。
- **Hard (Parallel) 平均**：Llama-3.1-70B 39.66%，GPT-4o 42.85%，是最大难点。
- **整体降幅**：从 Easy 到 Hard（Serial/Parallel）平均下降约 **62–65%**。
- **深度趋势**：Hard Serial 随深度增加准确率稳定下降；Hard Parallel 的下降趋势不稳定，说明"深度"并非并行难度的可靠指标。
- **错误分析（GPT-4o Hard 集）**：
  - Hard Serial 主因：60% 缺失关键事件（长距离依赖跟踪失败），25% 推理链不完整
  - Hard Parallel 主因：70% 未能识别并行事件（将并发当作串行处理），15% 推理链不完整，仅 5% 幻觉
- **Human 基线**：Easy/Medium 达到 100%，Hard Serial 93.33%，Hard Parallel 84.44%，差距明显。

## 相关工作脉络
1. **TimeQA**（Chen et al., 2021）：首个大规模 NL 驱动的 TSQA 基准，基于 Wikipedia，作者指出其高度依赖真实世界事实导致数据污染。
2. **TempReason**（Tan et al., 2023）：扩展支持事件-事件关系，但仍是序列事件、基于真实事实，存在相同污染风险。
3. **MenatQA**（Wei et al., 2023）：引入反事实问题，同样基于 Wikipedia，且反事实子集不包含 context，无法做 context-alteration 实验。
4. **RealTimeQA**（Kasai et al., 2023）/ **FreshQA**（Vu et al., 2024）/ **TAQA**（Zhao et al., 2024）：试图在知识截止日期之后构建，但需定期人工更新，作者在实验中证明新版 GPT-4 相比旧版准确率提升 100–177%。
5. **PAT-questions**（Meem et al., 2024）：自我更新机制，但仅在 Wikidata 新增条目时才触发更新，依赖外部知识库。
6. **ActionReasoning-Bench**（Handa et al., 2024）：本文数据生成管线来源，用于事件序列合法性验证。

## 局限性与未来方向
- **时间范围限制**：所有事件在 24 小时内，未覆盖跨天/周/月/年的更长时序推理。
- **模板语言多样性有限**：使用模板混合生成，仍可能存在句式可预测性，作者承认这是已知的局限性。
- **缺少不可答问题**：目前所有问题均可答，加入 "unanswerable" 类问题可增强挑战性。
- **仅英文**：未来可扩展至其他语言（仅需替换事件模板）。
- **并行难度指标问题**：深度在并行场景中不稳定，需探索新的复杂度度量方式。

## 研究启发与可借鉴点
1. **上下文篡改实验设计**：作者通过替换 gold answer → 替换 entity + question 的多级实验，优雅地证明了 LLM 的记忆依赖，此实验范式可直接迁移到其他评测领域。
2. **合成事实基准思路**：用受控生成环境替代真实数据来消除污染，这一思路可推广至数学推理、代码生成、逻辑推理等评测场景。
3. **深度（Depth）指标的可迁移性**：用事件序列中的位置差刻画推理复杂度，简单有效，可用于其他事件推理基准。
4. **并行推理的建模价值**：并发事件的引入是 TSQA 的重大扩展，对多智能体调度、工作流推理等方向有直接借鉴意义。
5. **错误类型的细粒度分类**：区分"遗漏事件""不完整推理链""幻觉""忽略并行"四类错误，为诊断 LLM 时序推理缺陷提供了可复用的分析框架。

## 关键术语表
**Time-Sensitive Question Answering (TSQA)**：需要理解事件时序关系（何时发生、持续多久、先后顺序）才能准确回答的问答任务。
**Data Contamination**：评测数据在 LLM 预训练阶段已被 exposure，导致模型通过"记忆"而非推理获得高分的现象。
**ActionReasoning-Bench**：本文借用的数据生成框架，用于生成合法的物流规划事件序列计划。
**Parallel Event Execution**：多个事件在同一时间段内并发执行的场景，要求模型处理并发约束与互斥条件。
**Question Depth**：问题目标事件在事件序列中的位置与起点事件之间的绝对距离，用于量化推理链长度。
**Static-Time / Relative-Time / Hypothetical-Time**：三种题目类型，分别考察特定时刻位置、时间偏移后位置和事件参数变化后的位置推理。
**Chain-of-Thought Prompting**：要求模型分步推理的提示策略，本文采用 zero-shot 和 few-shot 两种设置。

## 可复现要素
- **数据集**：UnSeenTimeQA，共 10,800 样本（4 难度 × 3 题型 × 3 次采样），已公开于 GitHub 和 Hugging Face，许可证 CC BY 4.0。
- **代码/生成框架**：基于 ActionReasoning-Bench 生成管线，数据生成脚本为 Python 实现；论文声明"open-source data generation framework"，具体链接见论文附录 F。
- **模型**：使用了 Llama-3.1-8B/70B、Gemma-2-9B/27B（HuggingFace 公开权重）、GPT-4o（OpenAI API）。
- **关键超参**：问题深度范围 6–20；每个深度随机采样 20 题，重复 3 次得到 3 个 split；事件持续时间以分钟为单位随机分配；提示使用 "Let's think step-by-step" 零样本 CoT 及 2 个示例的少样本设置。
- **硬件**：Llama-3.1-70B 使用 24 × A100 80GB GPU，其余模型使用 8 × A100 80GB GPU。
