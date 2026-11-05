---
title: "On-the-Mutual-Influence-of-Gender-and-Occupation-in-LLM-Repr"
source: https://aclanthology.org/2025.acl-long.83.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 08:17:13"
---

# 论文速读：On-the-Mutual-Influence-of-Gender-and-Occupation-in-LLM-Repr

## 一句话总结
本文系统剖析了LLM中首名字面性别表征与职业语境的相互偏移机制，并首次在1200万次提示的零样本职业预测任务上验证了内部性别投影与外在刻板偏见之间存在中等程度的相关性。

## 研究问题与动机
- 现有LLM偏见研究多为黑盒测试，缺乏对“模型内部性别表征如何驱动基于姓名的职业刻板行为”的机理解释。
- 首名常被用作性别代理，但其在不同职业语境下的表征是否会动态偏移、以及偏移程度如何，尚未被系统量化。
- 既往性别方向近似方法多针对静态词向量或简化上下文嵌入，缺乏对最新开源LLM的严格验证与参数对比。
- 内部表征指标（intrinsic metric）能否可靠预测外在偏见行为（extrinsic bias）仍存疑，需要实证检验对齐程度。

## 核心贡献（创新点）
1. **适配开源LLM的上下文性别方向近似算法**：基于反事实句子对与PCA提取第一主成分作为性别方向，并通过二分类任务严格验证其有效性，区别于以往仅依赖静态词向量的近似方法。
2. **揭示名字性别表征与现实统计及职业语境的联动机制**：证明LLM名字投影值与现实SSA性别分布强相关，且会随女性/男性主导职业语境发生系统性偏移，填补了“语境-表征”动态演化的实证空白。
3. **构建内蕴-外显偏见关联的量化评估框架**：提出Bias Coefficient与Internal Coefficient两个指标，打通内部嵌入投影与下游职业预测TPR之间的映射关系，为偏见归因提供可操作工具。
4. **大尺度对抗替换实验揭示偏见边界**：在四个主流开源LLM上执行超1200万次提示，系统绘制28个职业的内蕴相关谱图，明确指出了内蕴指标的有效区间与假阳性/假阴性情形。

## 方法详解
- **性别方向近似**：收集9对性别词（如 she/he, woman/man，排除Mary/John防过拟合），从English Wikipedia各提取3000句并通过反事实置换生成对应句。对每对词的平均上下文嵌入求差值矩阵，执行PCA取第一主成分 $\vec{g}_{1st}$ 作为性别方向（$k=1$，因第一PC解释方差占比32%–42%，显著高于其余分量）。
- **名字采样与嵌入提取**：从SSA与美国选民注册数据筛选470个首名，覆盖白/黑/西裔/亚裔四族两性，按女性人口占比划分为10个梯度桶。以24个代表名生成固定Wikipedia上下文，反事实替换得到平均上下文嵌入 $\vec{n}_{\text{wiki}}$。
- **语境偏移测量**：提示模板为 “{NAME} is {ARTICLE} {OCC.}. Is {NAME} male or female?”，计算Softmax化后的 $P_{\text{prior}}(\text{Female})$；同时提取职业词前后的名字嵌入 $\vec{n}_{\text{temp}}$，计算投影变化 $\text{DOT}(\vec{n}_{\text{temp}}, \vec{g})$。职业集取自Bias in Bios的28个职业，另设 “person” 作为非刻板基线。
- **下游职业预测与偏见系数**：在Bias in Bios（28职业，每职业135份女性/135份男性传记）上，将匿名占位符 `[NAME]` 逐一替换为470名字进行zero-shot预测。定义 **Bias Coefficient** 为名字女性化程度与职业TPR的Pearson相关系数；定义 **Internal Coefficient** 为名字最终嵌入投影 $\text{DOT}(\vec{n}_{\text{bios}}, \vec{g})$ 与目标职业预测概率的Spearman秩相关系数。

## 实验与结果
- **性别方向有效性**：Logistic Regression与Naive Bayes在 $\text{DOT}(\vec{n}_{\text{wiki}}, \vec{g}_{1st})$ 上的性别分类准确率与原始嵌入持平或略优（如Llama-3.1-8B: 75.18%/74.33%，OLMo-7B: 80.57%/80.28%），第二PC与平均PC表现显著下降，验证第一PC为最优近似。
- **与现实统计相关性**：`% Female (SSA)`、$\text{DOT}(\vec{n}_{\text{wiki}}, \vec{g})$、$P_{\text{prior}}(\text{Female})$ 三者两两之间均呈统计学显著的线性正相关（Fig. 3）。
- **职业语境偏移**：四个LLM均呈现一致趋势：女性主导职业（如nurse, 90.9%女性）使名字投影更偏正（更女性化），男性主导职业（如comedian, 21.1%女性）使投影更偏负；性别标识极强的名字受语境影响较小，性别模糊名字偏移更显著。
- **下游预测偏见**：LLM在性别-职业刻板匹配时TPR更高（如pastor对男性名字Bias Coeff = −0.55，dietitian对女性名字 = 0.68），多数职业偏见达到 $p<0.001$。
- **内蕴-外显对齐**：Internal Coefficient与Bias Coefficient呈中等正相关（Llama: 0.61，Mistral: 0.76，OLMo: 0.86，Phi: 0.90）；但在nurse/journalist等处漏检，在physician/accountant等处出现假阳性，表明单纯依赖投影难以完全预测外在行为。

## 相关工作脉络
- **Embedding性别子空间研究（Bolukbasi et al., 2016; Basta et al., 2019）**：本文继承其差值PCA思想，但明确适配至最新开源LLM的上下文嵌入，并系统对比了1PC/2PC/avg组合，排除首名防过拟合。
- **姓名人口属性代理（Bertrand & Mullainathan, 2004; Sancheti et al., 2024; You et al., 2024）**：本文延续姓名作性别代理的范式，但引入跨种族、含性别模糊名字的10级梯度分布，细化了Femininity连续性的刻画。
- **职业-性别偏见评测（De-Arteaga et al., 2019; Wang et al., 2024）**：沿用Bias in Bios与Bias Co
