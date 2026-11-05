---
title: "Nemotron-CC-Transforming-Common-Crawl-into-a-Refined-Long-Ho"
source: https://aclanthology.org/2025.acl-long.123.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 00:58:31"
---

# 论文速读：Nemotron-CC: Transforming Common Crawl into a Refined Long-Horizon Pretraining Dataset

## 一句话总结
本文针对 FineWeb-Edu、DCLM 等主流爬取数据集在长周期（万亿级 token）预训练中因过度过滤导致唯一 token 匮乏的问题，提出了一套结合多分类器集成、分层合成改写与启发式规则降级的数据处理流水线；基于该流水线构建的 Nemotron-CC 数据集在 1T 短周期训练上较 DCLM 提升 5.6 MMLU，在 15T 长周期训练上使 8B 模型超越 Llama 3.1 8B。

## 研究问题与动机
- **长 horizon 训练对数据独特性高度敏感**：FineWeb-Edu 与 DCLM 采用激进模型过滤剔除约 90% 数据，导致约 80% token 为近重复；在 15T 级别训练中模型会反复见到相同样本，引发边际收益递减。
- **单一质量分类器召回率低**：现有 FineWeb-Edu 或 DCLM 分类器对高质量 token 的召回率仅约 10%，大量优质内容被系统漏选。
- **启发式过滤器的误杀代价**：传统非学习型规则（如字符比例、KenLM 困惑度阈值）会误删约 18.1% 模型判定的高质量 token，在追求数据吞吐量时得不偿失。
- **低质数据的利用方式单一**：常规做法是直接丢弃低质量网页，但其中仍含可用信息；如何通过低成本改写降噪并转化为训练资产尚未充分探索。

## 核心贡献（创新点）
1. **多分类器集成标注流水线**：融合 FineWeb-Edu、自训练的 Mistral/Nemotron-340B 回归分类器与 DCLM fastText 分类器，通过 `max` 聚合将高质量 token 比例从 8%–14% 提升至 25%，本质区别在于利用偏置差异化的分类器互补召回而非依赖单一标准。
2. **分层合成数据生成策略**：低质量数据采用 Wikipedia 风格改写降噪，高质量数据通过多样 QA、提炼、知识抽取等 prompt 生成 1.9T 合成 token；与生成全新教科书/故事的路线不同，该方法仅做形式重组，用轻量模型即可高效扩量。
3. **启发式过滤器的选择性使用**：发现全局启发式过滤会误杀高质 token，因此仅对低质量分桶保留过滤，对模型认定的高质量数据直接放行，在不损下游性能的前提下显著拉升 token yield。
4. **开源 Nemotron-CC 长周期数据集**：发布 6.3T token 数据集（4.4T 唯一真实 + 1.9T 合成），并提供配套分类器权重与 NeMo Curator 参考实现，支撑学术界对精度-数量权衡的系统研究。

## 方法详解
- **HTML 提取与语言/去重处理**：对比 Trafilatura 与 Justext，选用 Justext 可多产出 28.6% 高质量 token；经 pycld2 与 FastText lid176（阈值 0.3）筛出英文后，使用 NeMo Curator 与 deduplicate-text-datasets 进行全局模糊去重与精确子串去重。
- **分类器训练与分数集成**：基于 Snowflake-arctic-embed-m 冻结嵌入层，在 FineWeb 上用 Mistral 8x22B 与 Nemotron-340B 指令模型标注 0-5 教育价值分并微调线性回归头；结合 DCLM 分类器，三者分别输出 0-19 整数桶分，最终质量分取三者的最大值。
- **下游对齐的分桶合并（Quality Bucketing）**：为避免分类分数与真实训练表现脱节，用 50B token 在 70% 训练的 8B 模型上做退火评估（66% 固定基底 +34% 目标分桶），按 9 任务平均性能将 20 个细桶合并为 High / Medium-High / Medium / Medium-Low / Low 五档。
- **差异化合成生成**：低质量文档切块后用 Wikipedia 改写 prompt 消除冗余；高质量文档切块后分别注入 Diverse QA Pairs、Distill、Extract Knowledge、Knowledge List 四种 prompt，由 Mistral NeMo 12B (FP8, top-p=0.9, T=0.5) 生成，经清理不完整输出、去 Markdown 前缀、过滤 <50 token 片段后拼接。
- **数据集拆分**：完整 Nemotron-CC 为 6.3T token；短周期对照子集 Nemotron-CC-HQ 仅保留最高分真实 token 与 Diverse QA 合成 token，共 1.1T。

## 实验与结果
- **训练设置**：8B Transformer（32 层，dim 4096，32 heads，SwiGLU，GQA-8），Megatron-LM 训练，数据混合为 73% 英文 CC + 27% 固定专项数据（代码/论文/书籍/专利/Wiki）。
- **短周期（1T tokens）**：Nemotron-CC-HQ 在除 RACE 外全部 10 项基准上超越 DCLM，MMLU **+5.6**（59.0 vs 53.4），十任务平均 **+3.1**；完整 Nemotron-CC 与 DCLM 打平，但唯一真实 token 多出 **4 倍**。
- **长周期（15T tokens）**：8B 模型使用 7.2T Nemotron-CC 加权数据训练，MMLU **70.3** 超越 Llama 3.1 8B（65.3，**+5.0**），ARC-Challenge 58.1（**+3.1**），十任务平均 64.7 vs 64.2（**+0.5**）。
- **消融结论**：仅对低质数据保留启发式过滤可使 MMLU 再涨 +2.0；分类器集成使 HQ token 比例翻倍；用合成数据替换重复 HQ 数据在多数基准上继续提升。

## 相关工作脉络
- **FineWeb-Edu / DCLM**：代表“极致精度优先”的单分类器+严格启发式路线；本文证明在长周期场景下适当放宽过滤并结合多分类器集成，可在精度基本持平的情况下获得数倍唯一 token。
- **Phi 系列 (Textbooks are all you need / phi-1.5)**：小数据高质量路线的代表；本文将其思想扩展到万亿级 token 场景，表明充足且多样的独特 token 配合适中质量即可追上甚至反超纯精化路线。
- **C4 / RefinedWeb / Gopher / DOLMA**：依赖非学习型启发式规则；本文通过定量分析揭示此类规则会误杀近两成高质 token，主张将其降级为低质数据的兜底工具。
- **TinyStories / Self-Instruct**：依赖 GPT-3.5/4 等大模型从零生成新内容；本文采用轻量模型仅做文本改写与结构化重组，不引入新知识但显著降低 perplexity 并增加 token 新颖度。
- **Rephrasing the web (Maini et al., 2024)**：首次验证 compact 模型重写网页的有效性；本文继承该范式并拓展至高低质分层策略与多 prompt 组合，系统验证其对短/长周期的增益。
- **Zyda-2**：并发工作聚焦已有开源数据集的跨去重与混合；本文聚焦从零构建新 CC 流水线，两者在数据供给策略上形成互补。

## 局限性与未来方向
- 分类器集成与分桶策略仅验证了单一配置，尚未探索更精细的融合权重或针对超高质区间的敏感度调优。
- 合成改写数据未经事实校验，可能引入轻微幻觉或稀释原文多样性；中质量（Medium）文档因资源限制未纳入合成 pipeline。
- 流水线部分组件（如语言识别）缺乏充分消融，且目前仅限英语，多语言扩展待验证。
- 未进行数据集去污染（decontamination），尽管对比基线同样未去污，但去污对不同规模与 token 周期的影响仍需社区系统研究。

## 研究启发与可借鉴点
- **集成偏置互补的分类器**：单一质量打分器存在隐性偏好，引入教育价值、信息密度等多视角分类器并以 max/加权融合，可显著扩召回而不显著降精度。
- **过滤器降级
