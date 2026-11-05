---
title: "Modeling-Uncertainty-in-Composed-Image-Retrieval-via-Probabi"
source: https://aclanthology.org/2025.acl-long.61.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 00:57:45"
field: "多模态检索"
keywords: ["Composed Image Retrieval", "Probabilistic Embedding", "Uncertainty Quantification", "Multimodal Retrieval", "Gaussian Embedding"]
innovations: ["将CIR查询与候选图像表示为潜空间Gaussian分布，方差显式建模实例级与特征级不确定性", "设计含Wasserstein-2与不确定性乘积惩罚的概率距离度量，实现不确定性感知匹配", "提出邻域偏差损失，通过k-NN特征std自监督估计各维度语义歧义"]
benchmarks: ["Fashion-IQ", "CIRR"]
---

# 论文速读：Modeling-Uncertainty-in-Composed-Image-Retrieval-via-Probabi

## 一句话总结
论文提出COPE（Composed Probabilistic Embedding）框架，将组合图像检索中的查询和候选图像表示为潜空间中的高斯分布而非固定点，通过概率距离度量与分层学习目标显式建模实例级与特征级不确定性，在Fashion-IQ与CIRR两个基准上均达到SOTA性能。

## 研究问题与动机
1. **输入数据质量参差**：参考图像可能模糊/遮挡/低分辨率，修改文本可能存在语法错误、口语化或信息不足，确定性点嵌入无法量化这种数据质量差异。
2. **用户意图语义模糊**：如"make it more natural""improve the lighting"等指令具有多义性，同一视觉属性在不同用户间解释不同，现有方法对这类语义歧义无显式建模。
3. **现有方法的均匀处理缺陷**：所有样本无论质量高低、歧义程度均被同等对待，导致训练不稳定、易过拟合，尤其在高歧义场景下性能受限。
4. **缺乏不确定性感知的距离度量**：传统余弦相似度或L2距离无法区分"高置信匹配"与"高不确定匹配"的语义差异。

## 核心贡献（创新点）
1. **提出COPE概率嵌入框架**：将查询与候选图像表示为潜空间的Gaussian分布，用方差σ显式量化不确定性，无需额外标注即可捕捉数据质量与语义歧义。
2. **设计不确定性感知距离度量**：结合Wasserstein-2距离与不确定性乘积惩罚项，使高不确定性匹配代价更高、相似不确定性维度可相互抵消，区别于传统点距离。
3. **构建分层学习目标**：实例级不确定性对比损失（基于SigLIP风格Sigmoid loss）与特征级邻域偏差损失（通过k-NN衡量各维度语义歧义），实现多维度不确定性联合建模。
4. **多粒度特征提取机制**：在CLIP ViT中引入Cross Attention门控，连接浅层视觉特征与文本全局嵌入，使模型能根据不同实例动态关注不同粒度表示。

## 方法详解
**概率嵌入表示**：
- 对参考图像$x_r$、修改文本$x_t$、候选图像$x_c$分别输出均值$\mu$与方差$\sigma$（经独立分支网络估计）。
- 查询嵌入通过向量加法组合：$z_q = z_r + z_t \sim \mathcal{N}(\mu_r+\mu_t,\; (\sigma_r^2+\sigma_t^2)I)$，将文本视为从参考到候选的偏移向量。

**不确定性感知距离度量**：
$$d(z_1,z_2) = \|\mu_1-\mu_2\|_2^2 + \|\sigma_1-\sigma_2\|_2^2 + 2D\bar{\sigma}_1\bar{\sigma}_2$$
- 第一项为均值Wasserstein距离；第二项鼓励不确定性分布对齐；第三项（$D$为维度数，$\bar{\sigma}$为均值）惩罚高不确定性实例间的匹配，避免不可靠样本主导训练。
- 等价形式揭示：距离与双方不确定幅度正相关，与两者不确定性模式的Pearson相关系数$\rho$负相关，使相同维度的高歧义特征在匹配时自动弱化。

**实例级不确定性学习**：
- 基于batch-wise Sigmoid对比损失：$\mathcal{L}_{ij} = -\log\sigma(m_{ij}(-a\cdot d(z_i,z_j)+b))$，其中$a,b$为可学习参数，$m_{ij}\in\{1,-1\}$为匹配标签。
- 该损失本质上是基于距离赋予权重：正样本对小距离赋予高权重，负样本对大距离赋予高权重，形成不确定性感知的硬例挖掘。

**特征级不确定性学习**：
- 引入邻域偏差损失$\mathcal{L}_{ND} = \sum_{x\in\{r,t,c\}}\|\sigma_x - \frac{1}{K}\cdot\text{std}(\mu_{N_x})\|_2^2$。
- 对每个样本求其$\mu$在特征空间的$k$-NN（余弦距离），用邻域特征标准差估计各维度的语义歧义程度，约束$\sigma$与其成正比。
- 总损失：$\mathcal{L} = \mathcal{L}_C + \lambda\mathcal{L}_{ND}$，$\lambda=0.02$时效果最佳。

## 实验与结果
**数据集**：
- Fashion-IQ：18,000训练三元组，6,016验证三元组，分Dress/Shirt/Top&tee三类，指标Recall@10/50。
- CIRR：36,554三元组（源自NLVR2真实照片），指标Recall@K与Recall_subset@K。

**基线与SOTA结果**：
- Fashion-IQ平均Recall@10=44.50，Recall@50=68.60，Avg=56.55，超越上一SOTA SADN（Avg 55.63）约0.92点。
- CIRR Recall@1=49.18、R@10=89.86，总Avg超越SADN（75.41→76.49）。
- 消融显示：硬对比损失基准（H.C.）Avg仅50.81；加入实例级不确定性提升至51.87；进一步加入特征级不确定性后达到56.55，特征级损失贡献最大增量（+4.68 Avg）。
- 不确定性分析证实：随查询整体不确定性增大，Recall单调下降，验证σ能有效反映数据质量。

## 相关工作脉络
1. **监督式CIR方法**（TIRG、CoSMo、ARTEMIS、CLIP4CIR、SADN等）：使用triplet数据与点嵌入融合视觉-文本特征；COPE在此基础上引入概率表示，不改变主干网络但提升不确定性建模能力。
2. **零样本CIR方法**（CompoDiff、BLIP4CIR、Pic2Word等）：将图像特征转为伪文本token，转化为文本到图像检索；COPE聚焦监督设置，利用更多triplet信号。
3. **不确定性感知的CIR正则化**（Xu et al., 2024 Set of Diverse Queries）：通过正则化增强对歧义输入的适应性，但不显式建模；COPE直接学习σ参数进行实例-特征两级不确定性量化。
4. **概率嵌入人脸/跨模态检索**（PFE、DUL、PCME）：仅在实例级建模不确定性；COPE进一步引入特征级不确定性（σ的逐维度分布），通过$\mathcal{L}_{ND}$实现。
5. **闭式概率距离度量**（Chun 2024）：提出改进的跨模态概率距离；COPE借鉴其思想并扩展至组合检索场景，新增Wasserstein+乘积惩罚的组合形式与邻域一致性正则。

## 局限性与未来方向
1. **排名尖锐度与召回率的权衡**：在Recall_subset@K等严格top-rank指标上提升有限，说明高不确定性场景下模型倾向于"宽泛匹配"而非"精准定位"，需结合动态重排序或更细粒度的对比对齐。
2. **计算开销**：多向量融合距离度量无法直接使用FAISS等单向量ANN框架，需依赖Milvus等支持混合搜索的系统，检索时间约为余弦相似度的2-8倍（视库规模）。
3. **静态不确定性假设**：当前σ对每个样本独立估计，不随检索上下文或任务类型动态调整；未来可探索context-dependent或task-aware的不确定性自适应机制。
4. **未探索扩散模型联合训练**：与CompoDiff等生成式方法的关系未充分讨论，概率嵌入与扩散先验的融合尚待探索。

## 研究启发与可借鉴点
1. **概率嵌入替代点嵌入的范式迁移**：COPE展示了将Gaussian embedding引入多模态检索的可行性，其"均值+方差双头网络"结构可直接复用于其他跨模态匹配任务（如文档-图像检索、视频-文本检索）。
2. **邻域偏差损失的设计思路**：通过k-NN特征的std估计各维度语义歧义，无需额外标注即可得到特征级不确定性信号，这一自监督策略可迁移至领域自适应或开放-world检索。
3. **Sigmoid对比损失的确定性推广**：将SigLIP风格loss与概率距离结合的思路可推广到任何需处理"噪声标签"或"多义正样本"的度量学习场景。
4. **多粒度Cross-Attention门控**：连接CLIP浅层视觉特征与文本全局嵌入的XA gate设计，使模型能按需聚焦不同粒度信息，对细粒度属性编辑检索具有参考价值。

## 关键术语表
**Composed Image Retrieval (CIR)**：组合图像检索，用户通过参考图像+文本修改指令联合检索目标图像的跨模态任务。
**Probabilistic Embedding**：概率嵌入，将数据点表示为分布（通常为高斯）而非固定向量，方差表征不确定性。
**Aleatoric Uncertainty**：数据固有不确定性，源于输入质量的随机性/模糊性，不可通过增加训练数据消除。
**Wasserstein-2 Distance**：Wasserstein-2距离，衡量两个概率分布之间差异的度量，此处用于比较均值与方差的联合差异。
**Instance-Wise Uncertainty**：实例级不确定性，反映单个样本整体的可靠性（由$\|\sigma\|_2^2$表征）。
**Feature-Wise Uncertainty**：特征级不确定性，反映嵌入各维度对应的语义概念的歧义程度（由$\sigma$的逐维分布表征）。
**Neighborhood Deviation Loss**：邻域偏差损失，通过k-NN邻域特征的std约束各维度不确定性，实现语义歧义的自监督估计。
**Generalized Pooling Operator (GPO)**：广义池化算子，一种保持概率可解释性的聚合操作，用于输出最终不确定性估计。

## 可复现要素
- **数据集**：Fashion-IQ与CIRR，均为公开数据集。
- **代码**：已开源，见 https://github.com/tanghme0w/ACL25-CoPE
- **模型权重**：论文未明确提供预训练权重下载链接。
- **关键超参**：batch size=128，初始学习率$2\times10^{-6}$，AdamW（$\beta_1=0.9,\;\beta_2=0.999,\;\epsilon=10^{-7}$），EMA rate=0.99，$\lambda=0.02$，K=10（邻域大小），5种数据增强（cutout/HSV/rotation/scaling/Gaussian noise，概率各0.2）。
- **骨干网络**：CLIP-ViT-L/14，单卡A100-80G训练。
