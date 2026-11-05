---
title: "Speaking-Beyond-Language-A-Large-Scale-Multimodal-Dataset-fo"
source: https://aclanthology.org/2025.acl-long.112.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 01:02:32"
field: "多模态对话生成"
keywords: ["多模态对话", "非语言线索", "VQ-VAE", "数据集构建", "3D人体重建", "大语言模型"]
innovations: ["提出VENUS大规模多模态对话数据集，首次融合文本、3D面部与全身身体语言的时间对齐标注", "设计Face/Body双VQ-VAE将连续非语言参数离散化为token，并引入速度一致性损失保证时序自然性", "提出MARS统一框架，通过交错文本与非语言token的next-token预测实现对话级多模态联合生成"]
benchmarks: ["VMSE", "LVD", "Window Vertex L2", "Diversity", "Variance", "PPL", "BERTScore", "METEOR", "NLL-F", "NLL-B"]
---

# 论文速读：Speaking-Beyond-Language-A-Large-Scale-Multimodal-Dataset-fo

## 一句话总结
本文提出了**VENUS**大规模多模态对话数据集和**MARS**多模态语言模型，通过将面部表情与身体语言的3D参数经VQ-VAE量化为离散token，与文本联合训练，使LLM能够理解并生成与对话语境对齐的非语言线索。

## 研究问题与动机
- 现有LLM仅处理纯文本对话，缺乏对面部表情、手势和肢体语言等非语言线索的建模能力，限制了沉浸式交互体验的构建。
- 已有的视频对话数据集（如YTD-18M、MultiDialog）要么规模有限，要么缺少精细的3D面部与身体语言标注；而运动生成数据集（如BEAT、EMAGE）又缺乏对话语境下的文本对齐信息。
- 非语言信号在传达情感、意图和微妙语义方面至关重要（如"Do you know what time it is?"搭配不同表情会产生完全不同含义），但现有工作多仅关注面部表情，忽视了身体语言的整体作用。
- 缺乏同时包含大规模对话文本、时间对齐面部参数与全身运动参数的多模态训练资源，制约了端到端多模态对话模型的发展。

## 核心贡献（创新点）
- **提出VENUS数据集**：首个大规模（89,459段对话、14,910小时）融合文本、3D面部表情与全身身体语言的时间对齐多模态对话数据集，显著超越现有对话/运动数据集的规模与标注丰富度。
- **设计VQ-VAE离散化方案**：分别构建Face VQ-VAE与Body VQ-VAE，将连续的FLAME面部参数（156维）与SMPL-X全身参数（179维）量化为离散codebook token，引入重建损失与速度一致性损失保证时序自然性。
- **提出MARS模型框架**：将文本token与非语言离散token按时间戳交错排列，统一通过next-token prediction目标训练，实现单一Transformer架构下文本与非语言线索的联合理解与生成。
- **系统性实验验证**：从数据集分布多样性、VQ-VAE量化质量（VMSE/LVD/Diversity等）到MARS的多模态对话生成能力（PPL/BERTScore/NLL）进行完整评估，并在用户研究中验证生成结果的自然性。

## 方法详解
- **数据收集管线**：从YouTube Podcast频道采集双人对话视频，经YOLO检测去噪、PyAnnote说话人分离、WhisperX时间对齐ASR、Light-ASD活跃说话人检测后，使用EMOCA-v2提取FLAME面部参数（$\mathbb{R}^{156}$）与OSX提取SMPL-X全身参数（$\mathbb{R}^{179}$）。
- **VQ-VAE量化**：对序列长度$W$的参数经Savitzky–Golay平滑后，通过编码器下采样（stride=2，$q=8$）得到潜向量$\mathbf{z} \in \mathbb{R}^{C \times \tau}$，再通过最近邻匹配到codebook $\mathcal{Z} \in \mathbb{R}^{K \times C}$（$K=512, C=8/16$），解码器上采样重建。
- **损失函数**：$\mathcal{L} = \mathcal{L}_{vq} + \lambda_{recon} \mathcal{L}_{recon} + \lambda_{vel} \mathcal{L}_{vel}$，其中$\mathcal{L}_{vq}$为commitment loss（$\beta=0.02$），$\mathcal{L}_{recon}$分别对表情$\psi$与下颌$\theta^{jaw}$加权（$\lambda^{jaw}=5$），$\mathcal{L}_{vel}$约束帧间速度一致性。
- **MARS训练**：输入序列$T = [\mathbf{x}_1^w, \mathbf{x}_1^f, \mathbf{x}_1^b, \mathbf{x}_2^w, \cdots]$按词-面-体交替排列，采用自回归预测$p(T) = \prod p_\theta(\mathbf{x}_j^w|T_{<j}) \cdot \prod p_\theta(\mathbf{x}_k^f|T_{<k}) \cdot p_\theta(\mathbf{x}_k^b|T_{<k})$，基于LLaMA 3.2/Qwen 2.5 Instruct进行SFT（batch=8/卡，seq_len=4096，50轮）。

## 实验与结果
- **数据集规模**：VENUS共869个频道、27,128个视频、89,459段对话、1,114,328轮次、527,270个不同词汇，平均每段对话21轮、170.8词、55.3秒，每轮含约547帧非语言标注。
- **VQ-VAE量化结果**（Table 3）：Face VQ-VAE最优（L1 loss、dim=8、size=512）VMSE=0.5106×10⁻¹、LVD=0.4020×10⁻³、w-VL2=0.2339×10⁻⁷、Diversity=7.8430；Body VQ-VAE最优（dim=16、size=512）VMSE=1.9946×10⁻¹、LVD=0.0962×10⁻⁴，全面超越Ng et al. (2023)与Guo et al. (2024)。
- **MARS对话生成**（Table 4）：相比zero-shot LLM基线，MARS-3B在PPL上从56,781降至800（降低98.6%），BERTScore从0.811升至0.839，NLL-F从20.85降至7.295，NLL-B从20.87降至4.666，且效果随模型规模单调提升。
- **定性分析**：MARS生成的文本在语义连贯性上优于GT，且生成的面部表情与肢体语言能合理匹配对话情绪（如表达质疑时皱眉、强调时使用手势）。

## 相关工作脉络
- **Ng et al. (2022, 2023)**：聚焦于从文本生成倾听者面部运动，未涉及身体语言且数据规模有限；本文扩展至全身动捕参数并构建大规模对话数据集。
- **Han et al. (2023) YTD-18M**：大规模Web视频对话数据集，但仅含文本与视频，无3D非语言标注；VENUS在其规模基础上补充了精细的面部与身体参数。
- **Park et al. (2024) MultiDialog**：融合音频与视觉生成对话，但仍缺乏3D面部/身体运动的显式建模；本文以SMPL-X/FLAME参数提供更高保真度的非语言表示。
- **Liu et al. (2022) BEAT / Liu et al. (2024) EMAGE**：专注于语音驱动手势/全身运动生成，但缺乏长程对话上下文；VENUS提供时间对齐的对话级语料支持多轮交互建模。
- **Lu et al. (2023) Humantomato / Wu et al. (2024) MotionLLM**：基于文本生成人体运动；本文区别于它们之处在于在对话语境下联合生成文本与非语言信号，并支持双向交互建模。

## 局限性与未来方向
- VENUS数据主要来源于Podcast频道，情感表达偏向中性/积极，缺乏哭泣、愤怒等极端情绪的样本，分布多样性受限。
- 非语言标注依赖伪标签（pseudo-labeling） pipeline，存在潜在误差累积，仍需人工校验或更精确的估计方法。
- 仅使用了部分VENUS数据进行训练，未充分挖掘全部数据潜力。
- 当前评估指标（NLL/PPL等）难以完整刻画非语言交流的真实自然度，需开发更贴近人类感知的评估体系。
- 未来计划引入语音韵律（vocal prosody）等额外模态，并探索跨场景泛化与真实交互场景验证。

## 研究启发与可借鉴点
- **伪标签+严格过滤的数据构建范式**：利用STT/ASD/3D重建等现成模型的伪标签构建大规模多模态数据，配合多维度过滤（安全检测、说话人数约束、视觉变化量阈值），是低成本构建高质量多模态语料的可行路径。
- **VQ-VAE量化时序运动参数时引入速度一致性损失**：$\mathcal{L}_{vel}$约束帧间差分可有效防止离散化导致的动作跳跃，该方法可迁移至任何基于codebook的时序动作生成任务。
- **文本与非语言token交错排列的 unified next-token 训练策略**：将离散视觉/运动token作为特殊token插入文本序列，使LLM原生理解多模态对齐关系，无需额外cross-attention模块，架构简洁且易于扩展。
- **代码/数据开源策略**：仅发布YouTube视频ID而非原始帧，配合SMPL-X模板网格表示，在保障研究价值与隐私合规之间取得平衡，值得类似数据采集项目借鉴。

## 关键术语表
- **VENUS**：Video with Nonverbal cues and Utterance Set，本文构建的大规模多模态对话数据集，含时间对齐的文本、3D面部与身体语言标注。
- **MARS**：Multimodal lAnguage Model with nonveRbal-cueS，基于VQ-VAE离散化非语言token并与文本联合训练的对话生成模型。
- **VQ-VAE**：Vector-Quantized Variational Autoencoder，通过离散codebook将连续特征映射为token的生成模型架构。
- **FLAME**：Facial LAndmark Model Embedding，用于参数化3D面部形状与表情的模型，本文提取156维面部参数。
- **SMPL-X**： Scalable body model with eXpression，统一建模全身（含手部）形状的3D人体参数化模型，本文提取179维身体参数。
- **NLL-F / NLL-B**：Face/Body的Negative Log-Likelihood，评估模型生成面部与身体token质量的指标。
- **Light-ASD**：轻量化活跃说话人检测模型，用于在视频中识别当前发言者并绑定对应视觉帧。
- **WhisperX**：带时间对齐的语音转文字模型，用于从音频生成精确到词级别的转录文本。

## 可复现要素
- **数据集**：VENUS，论文提供JSON格式的标注文件（含Channel ID、Video ID、Conversation、Facial expression、Body language等字段），仅公开YouTube视频ID，原始视频需用户自行获取。
- **代码**：开源，地址 https://github.com/winston1214/nonverbalconversation。
- **权重**：论文未明确提供预训练/微调权重的下载链接。
- **关键超参**：VQ-VAE codebook size=512，Face dim=8，Body dim=16，downsample q=8，序列长W=512，$\beta=0.02$，$\lambda_{recon}^{jaw}=5$，$\lambda_{vel}=0.5$；MARS训练batch=8/GPU，seq_len=4096，50 epochs；推理seq_len=512。
