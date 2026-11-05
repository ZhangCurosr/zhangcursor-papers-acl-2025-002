---
title: "ReSCORE-Label-free-Iterative-Retriever-Training-for-Multi-ho"
source: https://aclanthology.org/2025.acl-long.16.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 00:59:46"
field: "多跳问答与检索增强生成"
keywords: ["multi-hop question answering", "dense retrieval", "iterative RAG", "pseudo-label", "label-free training", "LLM supervision"]
innovations: ["提出无需标注的迭代检索器训练方法 ReSCORE，利用 LLM 生成相关性-一致性联合伪标签", "在迭代 RAG 框架中微调检索器，实现端到端性能提升，达到三项 MHQA 基准的 SOTA", "系统性分析伪标签设计与查询重构策略，揭示联合概率建模的有效性"]
benchmarks: ["MuSiQue", "2WikiMHQA", "HotpotQA"]
---

# 论文速读：ReSCORE-Label-free-Iterative-Retriever-Training-for-Multi-ho

## 一句话总结
论文提出了 **ReSCORE**（Retriever Supervision with Consistency and Relevance），一种无需文档标注的密集检索器训练方法，用于多跳问答（MHQA）。该方法利用大语言模型（LLM）生成基于“相关性-一致性”的伪标签，在迭代RAG框架中训练检索器，最终在三个基准上达到新的 SOTA 性能。

## 研究问题与动机
1. **密集检索器缺乏领域标注数据**：与稀疏检索器（如 BM25）不同，密集检索器（如 Contriever）依赖查询和文档嵌入，需要在目标领域进行微调。
2. **MHQA 的查询变异性导致标注成本极高**：在多跳推理过程中，查询（即重新表述的问题）会随迭代不断变化，难以收集覆盖所有迭代阶段的查询-文档相关性标注。
3. **现有迭代 RAG 方法未聚焦检索器训练**：主流工作（如 IRCoT、Adaptive-RAG）多关注推理框架本身，仍依赖预训练密集检索器或未微调的 BM25，未解决检索器如何适应特定迭代推理过程的问题。

## 核心贡献（创新点）
1. **提出无标注的检索器训练方法 ReSCORE**：利用 LLM 计算文档对问题和答案的概率，构建伪真值标签（pseudo-GT）来训练检索器，避免人工标注。
   - **本质区别**：不同于仅关注单跳一致性的 ATLAS，ReSCORE 在迭代框架中同时建模**相关性**（$P(q|d)$）和**一致性**（$P(a|q,d)$），更适配多跳推理。
2. **构建集成训练检索器的迭代问答系统 IQATR**：将 ReSCORE 训练的检索器嵌入迭代 RAG 流程，实现端到端的性能提升。
   - **本质区别**：此前工作多使用固定检索器，本文实现了检索器在迭代过程中的自适应优化。
3. **系统性分析伪标签设计与查询重构策略**：验证了相关性+一致性联合建模的有效性，并比较了不同查询重写方法的优劣。
   - **本质区别**：揭示了单纯使用答案一致性（$P(a|q,d)$）会导致虚假正例增加，而联合概率能过滤无关文档。

## 方法详解
1. **迭代 RAG 框架**：在每轮迭代 $i$，使用当前查询 $q^{(i)}$ 检索 Top-$k$ 文档，LLM 生成答案或判断为“unknown”；若为“unknown”，则生成压缩的“思考”（thought）$t^{(i)}$ 并重新表述查询 $q^{(i+1)}$。
2. **伪真值标签生成**：对每个候选文档 $d_j^{(i)}$，利用 LLM 计算联合概率 $P_{LM}(q, a | d_j^{(i)})$，并分解为相关性 $P_{LM}(q|d_j^{(i)})$ 与一致性 $P_{LM}(a|q, d_j^{(i)})$ 的乘积，归一化后作为检索分布的目标分布 $Q_{LM}^{(i)}$。
3. **训练损失**：最小化检索分布 $P_R^{(i)}$（由点积 Softmax 得到）与伪真值分布 $Q_{LM}^{(i)}$ 之间的 KL 散度，仅对检索器评分最高的 $M$ 个文档计算（$M=32$）。
4. **实现细节**：训练时冻结文档编码器，仅更新查询编码器；温度缩放设为 0.1；最大迭代次数 $\eta_n=6$，最小迭代次数 2。

## 实验与结果
- **数据集**：MuSiQue、2WikiMHQA、HotpotQA（均使用子采样验证/测试集）。
- **评估指标**：QA 性能（EM、F1）；检索性能（多跳召回 MHR@k，$k=8$）。
- **最强结果**：IQATR 在三个基准上均取得 SOTA：
  - MuSiQue：EM 23.4，F1 32.7（较 Baseline BM25 提升 +8.2 EM）
  - HotpotQA：EM 47.2，F1 59.3（较 Baseline BM25 提升 +5.0 EM）
  - 2WikiMHQA：EM 50.0，F1 59.7（较 Baseline BM25 提升 +5.4 EM）
- **检索提升**：在迭代过程中，MHR@8 随迭代增加持续上升，而 Baseline 的检索召回趋于饱和。

## 相关工作脉络
1. **Dense Retrieval Training**：与 ATLAS（Izacard et al., 2023）相比，ATLAS 仅用答案一致性训练单跳检索，ReSCORE 引入相关性并同时支持迭代多跳推理。
2. **Iterative RAG**：与 IRCoT、Adaptive-RAG 等框架相比，这些方法依赖预训练检索器或 BM25，ReSCORE 直接在迭代过程中微调检索器。
3. **LLM-as-Judge / Pseudo-labeling**：与 Promptagator、RankVicuna 等使用 LLM 生成排名列表的方法类似，但 ReSCORE 直接利用 LLM 的概率输出作为软标签训练检索器。
4. **Retriever-Generator Alignment**：与 LLM-Embedder、Intermediate Distillation 等对齐检索与生成的工作不同，ReSCORE 聚焦于多跳问答的迭代检索场景。

## 局限性与未来方向
- **泛化能力受限**：模型在训练数据集（MuSiQue、2WikiMHQA、HotpotQA）上表现优异，但在推理模式或数据分布不同的新数据集上泛化能力有限。
- **计算开销较高**：迭代检索与 LLM 推理叠加，导致延迟和算力成本上升，不适合对实时性要求极高的场景。
- **未来方向**：探索更高效的伪标签近似计算方法、增强跨领域泛化能力、以及与更轻量的检索器架构结合。

## 研究启发与可借鉴点
1. **无标注训练新思路**：利用 LLM 的概率输出构建软标签训练检索器，可迁移至其他需要迭代检索的复杂问答任务。
2. **相关性-一致性联合建模**：证明单一一致性指标易受词重叠干扰，联合概率能显著提升检索质量，该设计原则可应用于通用检索器训练。
3. **迭代中的检索器微调**：在迭代 RAG 框架内直接微调检索器，而非固定使用预训练模型，是一种值得探索的系统优化路径。
4. **查询重构策略分析**：Thought-concatenation（将思考摘要拼接至原查询）比纯 LLM 重写更稳健，为查询变换方法的选择提供了实证依据。

## 关键术语表
**Multi-hop Question Answering (MHQA)**：需要跨多个文档进行逻辑推理才能回答的复杂问答任务。
**Dense Retriever**：基于向量嵌入的检索模型，通过计算查询与文档的相似度进行检索。
**Pseudo-ground truth (pseudo-GT)**：利用大模型生成的、替代人工标注的训练标签。
**Relevance-Consistency Supervision**：同时考虑文档与问题的相关性（relevance）以及文档支持答案的一致性（consistency）的监督信号。
**Iterative RAG**：在生成答案过程中多次迭代检索与推理的检索增强生成框架。
**MHR@k (Multi-hop Recall at k)**：衡量在多跳问答中，前 $k$ 次迭代累计检索到的支持文档覆盖真实支持文档集合的比例。

## 可复现要素
- **数据集**：MuSiQue、2WikiMHQA、HotpotQA 均为公开数据集，但实验使用子采样版本。
- **代码**：论文声明代码已开源，地址为 https://leeds1219.github.io/ReSCORE。
- **关键超参**：温度 0.1、采样文档数 $M=32$、Top-$k$ 检索数 $k=8$、最大迭代次数 6、最小迭代次数 2、批次大小 16、初始学习率 $1\times10^{-6}$、指数衰减因子 0.9。
