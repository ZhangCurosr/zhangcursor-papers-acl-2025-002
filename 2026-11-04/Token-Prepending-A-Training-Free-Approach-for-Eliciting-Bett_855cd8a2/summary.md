---
title: "Token-Prepending-A-Training-Free-Approach-for-Eliciting-Bett"
source: https://aclanthology.org/2025.acl-long.159.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 16:41:12"
field: "自然语言处理-表示学习"
keywords: ["句子嵌入", "大语言模型", "无监督学习", "提示工程", "向后依赖"]
innovations: ["提出TP技术解决LLM因果注意力下的向后依赖问题", "免训练即插即用可结合多种提示方法"]
benchmarks: ["STS 2012-2016", "STS-B", "SICK-R", "SentEval"]
---

# 论文速读：Token Prepending: A Training-Free Approach for Eliciting Better Sentence Embeddings from LLMs

## 一句话总结
本文提出了一种免训练的即插即用技术 Token Prepending (TP)，通过在 LLM 中间层将前一层解码的句子嵌入 prepend 到下一层输入开头，缓解因果注意力机制下的向后依赖问题，显著提升了从 LLM 中提取句子嵌入的质量。

## 研究问题与动机
- **核心问题**：现有从 LLM 提取句子嵌入的方法（如 PromptEOL、MetaEOL 等）仅通过提示工程引导模型将句子信息编码到最后一个 token，但 LLM 采用因果注意力机制，较早的 token 无法 attend 到后面的 token，导致句子信息编码存在偏差并级联影响最终输出。
- **现有方法不足**：
  - Echo embeddings 通过重复输入两次来建立向后依赖，但显著增加序列长度和推理成本。
  - PromptEOL 等提示方法虽简单有效，但未解决因果注意力固有的向后依赖缺失问题。
  - 微调方法（如 BeLLM）需替换为双向注意力并消耗大量训练成本，且可能损失 LLM 通用能力。

## 核心贡献（创新点）
1. **提出 TP 技术**：通过干预 LLM 中间层输入，将前一层解码的句子嵌入 prepend 到下一层输入开头，使所有 token 能在因果注意力下感知完整句子语义。
2. **即插即用且免训练**：不引入任何可学习参数，可与现有提示方法（PromptEOL、MetaEOL、Pretended CoT、Knowledge Enhancement）无缝结合，额外推理开销仅约 4%。
3. **发现最佳实践**：TP 在模型前几层（第 2-8 层）执行效果最佳，并建议从中层 early-exit 而非最后一层输出句子嵌入。
4. **系统性分析**：深入探讨了 <PST> token 的位置、初始化方式、层范围选择及 early-exit 策略对性能的影响。

## 方法详解
**核心思想**：通过在 Transformer 层间干预，让较早的 token 能够 attend 到完整的句子信息。

**具体步骤**：
1. **初始 Token Prepending**：在输入层 prepend 一个自定义特殊 token `<PST>`（不在 LLM 词表中），随机初始化其嵌入向量，作为句子嵌入的占位符。
2. **中间 Token Prepending**：在第 $l$ 层（$l \in [2, k]$）输出后，用该层最后一个 token 解码出的句子嵌入替换 `<PST>` 的嵌入，作为第 $l+1$ 层的输入：
   $$\mathbf{h}^l = \text{LLM}^{l-1}(f(\mathbf{h}^{l-1})), \quad f(\mathbf{h}) = [\cdots, h_{SET}, \cdots]$$
3. **停止 TP**：在第 $k$ 层后停止 TP 操作，恢复标准前向传播。
4. **Early-Exit 策略**：从中间层（非最后一层）输出句子嵌入，因为最后一层主要用于 token 生成，语义信息较弱。

**关键超参**：
- TP 执行层范围：$k = 8$（对于 7B 模型）
- Exit 层：根据验证集选择（如第 27 层）

## 实验与结果
**数据集**：7 个 STS 基准（STS 2012-2016、STS-B、SICK-R）及 SentEval 下游分类任务（MR、CR、SUBJ 等）。

**基线方法**：PromptEOL、Echo embeddings、MetaEOL、Pretended CoT、Knowledge Enhancement。

**主要结果**（LLaMA2-7B + PromptEOL）：
| 方法 | STS-B | SICK-R | Avg. |
|------|-------|--------|------|
| PromptEOL | 71.66 | 69.64 | 70.03 |
| PromptEOL + TP | **80.67** (+9.01) | **75.40** (+5.76) | **77.19** (+7.16) |

**跨模型泛化**：
- Qwen2-7B：平均提升 2.17 分
- LLaMA2-13B、LLaMA3-8B、Gemma2-9B 均有稳定提升

**推理效率**：TP 仅增加约 1.04× 推理时间（相比 PromptEOL 的 1.00×）。

**强结果**：PromptEOL + TP 在 STS-B 上达到 80.67，较原方法提升 9.01 分。

## 相关工作脉络
1. **PromptEOL (Jiang et al., 2023)**：首个通过简单提示从 LLM 提取句子嵌入的方法，本文在此基础上引入 TP 解决向后依赖问题。
2. **Echo embeddings (Springer et al., 2024)**：通过输入重复建立向后依赖，但显著增加序列长度；TP 以更轻量方式实现类似目标。
3. **MetaEOL (Lei et al., 2024)**：利用元任务提示多角度捕捉句子表示，TP 可与之结合但效率较低（8.17× 推理时间）。
4. **Pretended CoT / Knowledge Enhancement (Zhang et al., 2024)**：引入 CoT 和知识增强，本文验证 TP 对这些方法的普遍适用性。
5. **BeLLM (Li & Li, 2024)**：通过微调将最后层注意力改为双向，训练成本高；TP 免训练且无需修改模型结构。

## 局限性与未来方向
- **超参敏感**：需调整两个超参（TP 结束层、exit 层），且最优值因模型、数据集、提示而异，增加适配成本。
- **未解决所有偏差**：仅在前几层执行 TP，深层 backward dependency 建模仍有限。
- **未来方向**：可探索自适应确定 TP 层范围、结合更高效的 backward dependency 建模策略。

## 研究启发与可借鉴点
1. **即插即用设计**：TP 无需训练即可提升现有方法，为后续工作提供了"增强现有框架"的研究范式。
2. **Early-Exit 策略**：从中层而非最后一层输出嵌入，对 LLM 表示学习有普遍启示。
3. **轻量干预 vs 结构修改**：通过少量层间干预解决因果注意力缺陷，比替换为双向注意力更实用。
4. **超参消融分析**：系统分析 <PST> 位置、初始化方式、层范围，为方法可靠性提供充分论证。

## 关键术语表
- **Token Prepending (TP)**：将前一层解码的句子嵌入 prepend 到下一层输入开头的技术。
- **Causal Attention**：自回归模型中 token 只能 attend 到前面 token 的注意力机制。
- **Sentence Embedding Token (SET)**：由提示引导模型生成的代表句子语义的 token 嵌入。
- **Placeholder Token (<PST>)**：自定义的特殊 token，作为句子嵌入的占位符。
- **Early-Exit**：从模型中间层而非最后一层输出表示的策略。
- **Semantic Textual Similarity (STS)**：评估句子对语义相似度的任务。

## 可复现要素
- **数据集**：STS 2012-2016、STS-B、SICK-R（公开）、SentEval 下游任务（公开）。
- **代码**：已开源 https://github.com/fuyuchenIfyw/token_prepending.git。
- **模型**：LLaMA2-7B/13B、LLaMA3-8B、Qwen2-7B、Gemma2-9B（公开权重）。
- **关键超参**：TP 结束层 $k=8$，exit 层第 27 层（PromptEOL/MetaEOL/Pretended CoT）。
