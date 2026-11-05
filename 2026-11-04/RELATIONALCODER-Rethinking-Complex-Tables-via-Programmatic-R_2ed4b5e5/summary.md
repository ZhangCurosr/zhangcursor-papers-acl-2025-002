---
title: "RELATIONALCODER-Rethinking-Complex-Tables-via-Programmatic-R"
source: https://aclanthology.org/2025.acl-long.89.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 08:18:12"
field: "表格理解与关系化"
keywords: ["半结构化表格转换", "循环引用解码", "关系型数据", "主动学习标注", "表格问答"]
innovations: ["提出 RELATIONALCODER 框架用 SQL 生成规范化关系表避免结构错误", "设计 LRD 技术通过循环引用实现输出长度 O(K) 压缩", "构建 850 样本标注集并提出 SFT-AP 主动标注流程"]
benchmarks: ["HiTab", "MULTIHIERTT"]
---

# 论文速读：RELATIONALCODER: Rethinking Complex Tables via Programmatic Relational Transformation

## 一句话总结
本文提出 **RELATIONALCODER**，利用 SQL 代码将半结构化复杂表格（如网页、财务报告中的多级表头/合并单元格）统一转换为关系型数据；针对大表格输出爆炸问题，进一步提出 **Loop Reference Decoding (LRD)** 技术，通过识别可展开区域并使用 `FOR` 循环引用单元格地址批量生成，实现极高压缩率，并在 HiTab 与 MULTIHIERTT 上显著提升问答准确率。

## 研究问题与动机
- **半结构化表格阻碍自动化处理**：真实文档（Wikipedia、政府报告、Excel）含合并单元格、多层表头、跨表计算等复杂格式，传统 LLM 直接问答易出错。
- **现有方法未能接入关系型工具生态**：已有工作侧重微调 LLM 理解复杂表格结构，但未将其转换为标准关系表，导致无法利用 SQL/Pandas 等成熟数据分析工具进行符号推理。
- **大表格生成面临输出长度爆炸与幻觉风险**：逐单元格生成会导致输出 token 数随表格高度×宽度线性增长（$O(NM)$），且易产生格式错误或单元格幻觉。
- **缺乏高质量标注数据与低成本标注流程**：表转换任务尚无公开标注集，且人工标注成本高，需探索高效主动学习策略。

## 核心贡献（创新点）
1. **提出 RELATIONALCODER 框架**：利用 SQL `CREATE` / `INSERT` / `UPDATE` 语句将半结构化表格转换为规范化关系表，避免行/列对齐错误，支持多表连接与键定义。
2. **设计循环引用解码（LRD）**：识别输入表中的可展开重复区域，使用 `FOR` 循环基于单元格地址引用批量复制数据，将输出长度从 $O(NM)$ 压缩至约 $O(K)$（$K$ 为唯一值数量），平均压缩 68.4%，极端情况达 160,000×。
3. **构建首个表转换标注数据集与主动标注流程（SFT-AP）**：收集 850 个样本，提出基于 LLM 质量度量与推理准确率的主动选样策略，用少量人工标注显著提升模型性能。
4. **验证关系化对下游 QA 的增益**：在 HiTab 和 MULTIHIERTT 上，GPT-4o 准确率分别提升至 84.8% 和 73.6%，小模型 Llama-2 / Mistral 提升超 20%。

## 方法详解
- **表编码与压缩**：沿用 SPREADSHEETLLM 的显式单元格地址编码（如 `"A2"`），配合 JSON 倒排索引压缩空白/重复单元格，并对大表应用 Table Split Algorithm 分段处理。
- **SQL 生成**：先生成 `CREATE TABLE` 建表语句定义列名与类型，再用 `INSERT` 批量插入行数据；允许 `UPDATE` 修改派生列，支持跨表关系建模。
- **循环引用解码（LRD）**：由 LLM 识别展开组（expandable groups）后，生成嵌套 `FOR` 循环（如 Oracle PL/SQL `FOR row_id IN 7..16 LOOP`），通过字符串拼接（`'A'||row_id`）引用地址，避免逐单元生成。
- **主动参与监督微调（SFT-AP）**：以归一化质量（LLM/人工评估）与推理准确率（QA 模型失败案例）为指标筛选难例，按比例 $(p_{rnd}, p_{norm}, p_{calc}, p_{qa}) = (0.4, 0.15, 0.15, 0.3)$ 选样，迭代微调直至准确率提升低于阈值。

## 实验与结果
- **数据集**：HiTab（3,597 表，10,672 QA 对，来自政府报告与 Wikipedia）与 MULTIHIERTT（2,513 财务文档，10,440 QA 对）。测试集分别取 1,584 与 338 个表样本。
- **基线**：EEDP（提示策略）、SPREADSHEETLLM（输入压缩）、Chain-of-table（多跳推理框架）、NormTab（结构归一化）等。
- **主要结果**：
  - 归一化质量：GPT-3.5 生成表在 HiTab 上人类评估 79.4%、自动评估 82.3%；Mistral 64.5%/67.5%（Table 2）。
  - QA 准确率（Table 3）：GPT-4o + RELATIONALCODER 在 HiTab 达 84.8%（原 80.7%）、MULTIHIERTT 达 73.6%（原 68.1%）；Llama-2 从 30.3% 提升至 55.9%（+25.6%），Mistral 从 25.1% 提升至 52.9%（+27.8%）。
  - SFT-AP 效果（Table 4）：GPT-3.5-SFT-AP 生成表使 Llama-2 QA 达 55.9%，超过 GPT-4o-zero-shot（46.7%）。
  - LRD 消融（Table 5）：SQL-LRD 相比 Markdown 输出在 HiTab 提升 1.8%、MULTIHIERTT 提升 2.4%，同时输出 token 减少 68.4%。
- **极限压缩案例**：5,000 股票 × 252 交易日表，原生 SQL 输出约 18.6M token，LRD 仅需 115 token，压缩比 160,000×。

## 相关工作脉络
- **表转换早期工作**（Harris & Gulwani, 2011; Cafarella et al., 2008）依赖用户示例或分类规则，难以泛化复杂格式。
- **NormTab**（Nahid & Rafiei, 2024）仅用转置算子做结构归一化，未建模多表关系，且对缩进/合并单元格鲁棒性有限。
- **Auto-tables**（Li et al., 2024）通过预定义算子合成数据，但依赖固定操作符，缺乏端到端 LLM 灵活性。
- **SPREADSHEETLLM**（Dong et al., 2024）专注输入编码压缩，本文将其编码方案与 LRD 解码结合，解决输出侧瓶颈。
- **Chain-of-table**（Wang et al., 2024）利用关系表进行多跳推理，本文提供前置表转换模块，与之形成互补流水线。

## 局限性与未来方向
- **局限性**：
  1. 需保留原始格式的场景（如文档排版、视觉渲染）不适用此方法。
  2. 仅处理表格结构，未整合文档中非结构化文本信息。
  3. 表转换阶段需要较强模型（如 GPT-3.5），推理阶段可换小模型。
  4. 依赖人工标注数据，标注成本虽通过主动学习降低但仍存。
- **未来方向**：
  1. 联合保留原始复杂输入与转换后关系表，服务多任务场景。
  2. 结合 Text-to-Table 技术提取文本中的关系信息（Wu et al., 2022）。
  3. 探索端到端单模型流水线（转换→处理→推理）。
  4. 引入强化学习（RL）从弱监督（QA 标注）自动学习转换策略，减少对人工标注依赖（Dong et al., 2025）。

## 研究启发与可借鉴点
- **程序化输出提升可靠性**：用 SQL 代码替代自然语言或 Markdown 表格，可有效避免 LLM 常见的行列错位、分隔符缺失等结构错误。
- **循环引用解码应对长序列瓶颈**：LRD 思想可迁移至其他需生成大量重复结构的任务（如时间序列展平、多维数组展开），显著压缩输出长度。
- **主动选样标注策略**：SFT-AP 中以 QA 失败案例与归一化违规样本为优先标注目标，比随机采样更高效，适用于各类指令微调数据构建。
- **关系化作为通用预处理模块**：将任意复杂表转换为 SQL 可解析的关系型数据后，可直接接入 Pandas、数据库等工具链，提升下游任务泛化能力。

## 关键术语表
- **半结构化表格**：具有嵌套表头、合并单元格、跨行/列计算等非规范布局的表格数据。
- **关系型数据**：满足第一范式（每列原子值、无重复组）的二维表结构，支持 SQL 查询与连接。
- **循环引用解码（LRD）**：识别输入表中的重复块后，生成循环语句并通过单元格地址引用批量写入输出表的解码技术。
- **SFT-AP**：Supervised Fine-Tuning with Active Participation，基于 LLM 质量度量与推理准确率主动筛选难例进行迭代微调的策略。
- **Expandable Group**：具有统一模式与语义的连续矩形区域，可由循环语句整体复制（如示例中的 A7:H16）。
- **HiTab**：包含 3,597 个多级表头复杂表格与 10,672 个 QA 对的 benchmark，源自 HTML 政府报告与 Wikipedia。
- **MULTIHIERTT**：来自 PDF 财务报告的 2,513 个文档与 10,440 个多步数值推理 QA 对数据集。
- **Chain-of-table**：将关系子查询链式组合以回答多跳表格问题的推理框架。

## 可复现要素
- **数据集**：HiTab 与 MULTIHIERTT 为公开数据集，可直接下载使用。
- **代码/权重**：项目主页 https://github.com/haoyudong/RelationalCoder，论文声明开源代码。
- **关键超参**：
  - LoRA rank=32, alpha=64, dropout=0.01
  - 学习率 5e-5，epochs=40，batch size=4，gradient accumulation=8
  - SFT-AP 迭代 5 次，每次选 150 样本，标注上限 m=32 行 n=8 列
  - 选样比例 $(p_{rnd}, p_{norm}, p_{calc}, p_{qa}) = (0.4, 0.15, 0.15, 0.3)$
  - 早停阈值 q=0.1
- **模型**：GPT-3.5（textdavinci-003）、Llama-2-13B、Mistral-7B、GPT-4o。
