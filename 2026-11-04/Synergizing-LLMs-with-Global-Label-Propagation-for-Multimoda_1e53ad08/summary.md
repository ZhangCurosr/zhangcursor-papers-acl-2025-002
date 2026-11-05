---
title: "Synergizing-LLMs-with-Global-Label-Propagation-for-Multimoda"
source: https://aclanthology.org/2025.acl-long.72.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 01:03:11"
field: "多模态假新闻检测"
keywords: ["假新闻检测", "多模态", "大语言模型", "标签传播", "伪标签", "图神经网络"]
innovations: ["提出GLPN-LLM框架，将LLM伪标签与全局标签传播深度整合", "设计Global Random Mask机制防止训练标签泄露", "混合主动式标签策略通过置信度阈值筛选高可信伪标签"]
benchmarks: ["Twitter", "PHEME", "Weibo"]
---

# 论文速读：Synergizing-LLMs-with-Global-Label-Propagation-for-Multimodal-Fake-News-Detection

## 一句话总结
论文提出 GLPN-LLM 框架，通过全局标签传播（Label Propagation）机制将 LLM 生成的伪标签有效融入多模态假新闻检测，解决了"LLM 伪标签直接拼接效果差"的难题，在 Twitter/PHEME/Weibo 三个数据集上均显著优于 SOTA 基线。

## 研究问题与动机
1. **LLM 伪标签单独使用效果差**：GPT-4o 直接预测的伪标签准确率远低于传统多模态检测方法（表1），简单拼接无法发挥 LLM 潜力。
2. **标签传播对中等质量伪标签仍鲁棒**：LP 技术在伪标签准确率不高时依然有效（Sun et al., 2025），适合承接 LLM 生成的"不完美但有一定价值"的伪标签。
3. **直接引入伪标签会导致标签泄露（Label Leakage）**：若训练中节点特征已含自身标签，模型可能"作弊式"地通过特征记忆而非图结构学习分类。
4. **现有 FCN-LP + LLM 的简单加法策略提升有限**：FCN-LP + LLM 相比 FCN-LP 仅微幅改善（如 Twitter 数据集提升不足1%），说明需要更深入的融合设计。

## 核心贡献（创新点）
1. **提出 GLPN-LLM 框架**：将 LLM 伪标签与全局标签传播深度结合，而非简单拼接；与 FCN-LP + LLM 的本质区别在于设计了掩码机制避免训练泄露，使 LLM 信号能在图结构中被充分传播。
2. **设计 Global Random Mask（GRM）**：每轮训练随机遮蔽部分节点的真实/伪标签（掩码率 $\rho$），仅对未被遮蔽节点计算损失，迫使 GCN 依赖图传播而非节点自身特征记忆标签，从根源防止标签泄露。
3. **引入 Mixed-Initiative Labeling**：通过结构化 prompt 让 LLM 输出带置信度分数的伪标签，并按置信度阈值（top 5%）筛选高可信伪标签融入图节点，保证传播源的质量。
4. **系统化实验验证**：在 Twitter/PHEME/Weibo 三数据集上证明 GLPN-LLM 相比各 SOTA 基线的稳定性提升，并开展 mask rate、伪标签比例、prompt 设计等消融分析。

## 方法详解
- **多模态特征提取**：用 CLIP 双编码器分别提取图像特征 $\mathbf{v_i}$ 和文本特征 $\mathbf{t_i}$，拼接为统一特征 $\mathbf{x_i} = \mathbf{t_i} \oplus \mathbf{v_i}$。
- **跨模态图构建**：基于五类相似度（拼接特征余弦、图到文、文到图、图到图、文到文）计算节点间边，超过阈值 $\theta=0.95$ 则连边，参考 FCN-LP 的图构造策略。
- **标签融合模块**：真实标签（训练节点）与 LLM 伪标签（未标记/测试节点）经 one-hot 编码后拼接到节点特征：$\mathbf{x_i'} = \mathbf{x_i} \oplus \mathbf{y_i'}$，其中 $\tilde{\mathbf{y}}_i$ 按公式(5)区分三类情况（真实标签 / 高置信伪标签 / 零向量）。
- **Global Random Mask（GRM）**：每轮训练随机选取比例 $\rho$ 的节点，将其标签嵌入置零（$m_i=0$），使得 $\mathbf{y_i'} = \tilde{\mathbf{y}}_i \cdot m_i$，确保节点不会在传播过程中"看到自己"的标签。
- **GCN 分类器**：以增强后的节点特征 $\mathbf{x_i'}$ 输入 GCN，交叉熵损失优化，Adam 优化器，学习率 $1e^{-3}$，隐层维度 512。

## 实验与结果
- **数据集**：Twitter（15k train / 2k test）、PHEME（1,414 train / 608 test）、Weibo（4,141 train / 1,125 test）。
- **最强结果**：GLPN-LLM (CLIP) 在 Twitter 上 Accuracy=88.83%、F1=89.03%；PHEME 上 Accuracy=89.24%、F1=90.66%；Weibo 上 Accuracy=93.27%、F1=91.52%，三项均显著超越所有 SOTA 基线。
- **关键提升幅度**：
  - Twitter：GLPN-LLM (CLIP) vs FCN-LP (CLIP) + LLM，Accuracy 提升约 2.90%，F1 提升约 3.06%。
  - PHEME：GLPN-LLM (CLIP) vs FCN-LP (CLIP) + LLM，F1 提升约 1.45%。
  - Weibo：GLPN-LLM (CLIP) vs FCN-LP (CLIP) + LLM，F1 提升约 1.67%。
- **消融结论**：GLPN > FCN-LP（掩码传播优于固定 LP）；GLPN-LLM > GLPN（LLM 伪标签进一步增益）；mask rate=0.5、伪标签 top 5% 置信度为最优配置。

## 相关工作脉络
1. **EANN / SpotFake / MVAE**：早期多模态假新闻检测基线，依赖特征融合但无图结构；本文基线对比参照。
2. **MCAN / HMCAN**：引入多模态交叉注意力机制，特征表达更强；本文选用 HMCAN 作为 backbone 之一。
3. **FCN / FCN-LP（Zhao et al., 2023）**：基于 CLIP + GCN 的跨模态图方法，是本文最直接的前作；本文在其基础上引入 LLM 伪标签与掩码传播机制。
4. **Label Propagation（Zhu & Ghahramani, 2002）及其 GNN 扩展**：传统半监督标签传播方法，本文将其改进为"全局掩码式传播"以适配训练过程。
5. **LLM 辅助检测的简单融合（FCN-LP + LLM）**：直接拼接伪标签的朴素方案，本文证明其效果有限，引出更深融合的必要。

## 局限性与未来方向
- **依赖 Backbone 模型质量**：若底层 FCN 特征提取能力不足，标签传播效果受限；框架对特定 backbones 的泛化性需进一步验证。
- **伪标签准确性依赖 LLM**：LLM 生成的低置信/错误伪标签会引入噪声，需更鲁棒的伪标签验证与过滤机制。
- **可扩展性待验证**：当前实验在中小规模数据集（最大 17k 样本）上进行，面对大规模复杂数据集和更多模态（如视频）的适应性未知，作者明确将此列为未来工作方向。

## 研究启发与可借鉴点
1. **"LLM 伪标签 + 标签传播"的整合范式**：当 LLM 输出存在噪声时，标签传播可作为"去噪+放大"的信号放大器，该思路可迁移至其他弱监督/半监督学习任务。
2. **Global Random Mask 防泄露设计**：对任何将标签嵌入特征再传播的图学习方法均有借鉴意义，是防止训练-推理不一致（train-inference mismatch）的有效正则手段。
3. **置信度阈值筛选伪标签（top 5%）**：简单但有效的质量控制策略，过度宽松（如 top 50%）会引入噪声导致性能下降，提示工程与筛选阈值的联合调优是关键。
4. **Prompt 复杂度与性能的非线性关系**：详细 prompt 提升置信度和 recall，但过度复杂反而引入噪声，提示设计需权衡信息量与干扰。

## 关键术语表
**GLPN-LLM**：Global Label Propagation Network with LLM-based Pseudo Labeling，本文提出的多模态假新闻检测框架。
**Label Propagation（标签传播）**：在半监督学习中，将已知标签沿图结构传播到未标记节点以预测其标签的技术。
**Global Random Mask（GRM）**：训练时每轮随机遮蔽部分节点标签嵌入，防止模型直接记忆自身标签的防泄露机制。
**Mixed-Initiative Labeling**：结合人工标注与 LLM 生成伪标签的混合策略，通过置信度筛选确保伪标签质量。
**Cross-modal Graph（跨模态图）**：以多模态相似性（文本-文本、图像-图像、跨模态等）为边的图结构，用于捕捉新闻间的语义关联。
**CLIP**：Contrastive Language-Image Pre-training，同时学习文本与图像 Embedding 的多模态预训练模型，本文用于特征提取。

## 可复现要素
- **数据集**：Twitter（MediaEval 2015）、PHEME、Weibo，均为公开基准数据集（论文及附录有说明）。
- **代码**：论文声明 "Our code is available online"，但具体 URL 未在全文中给出（正文末尾标注脚注1）。
- **关键超参**：mask rate $\rho=0.50$，LLM 伪标签置信度阈值取 top 5%，相似度阈值 $\theta=0.95$，学习率 $1e^{-3}$，GCN 隐层维度 512，训练 5 次取平均。
- **LLM**：使用 GPT-4o API，prompt 设计见附录 Table 3。
