---
title: "Rethinking-Repetition-Problems-of-LLMs-in-Code-Generation"
source: https://aclanthology.org/2025.acl-long.48.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 01:00:36"
field: "代码生成与程序合成"
keywords: ["代码生成", "重复问题", "结构重复", "大语言模型", "解码方法", "语法规则"]
innovations: ["首次形式化定义代码生成中的结构重复问题", "提出基于语法规则的RPG解码方法检测和惩罚重复", "构建CodeRepetEval数据集并提供多场景重复问题评测基准"]
benchmarks: ["HumanEval", "MBPP", "CodeRepetEval"]
---

# 论文速读：Rethinking-Repetition-Problems-of-LLMs-in-Code-Generation

## 一句话总结
论文首次正式定义并研究了大语言模型在代码生成中的"结构重复"问题（比内容重复更普遍），提出基于语法规则的解码方法 RPG，通过下推自动机识别重复模式并对关键token施加指数衰减惩罚，在 CodeRepetEval 和新数据集上显著降低重复率并提升代码质量。

## 研究问题与动机
- **结构重复是代码生成中更普遍的重复形式**：现有工作主要关注"内容重复"（完全相同的片段重复出现），但作者初步调查显示内容重复仅占小部分，更主要的是"结构重复"——不同片段具有固定语法结构但具体内容各异（如多次 `elif` 分支）。
- **现有方法无法有效处理结构重复**：基于内容的惩罚方法（如 Repetition Penalty）对结构重复无效；训练阶段的 Repetition Dropout 成本高且会损害代码生成性能。
- **结构重复具有自强化效应**：随着重复次数增加，起始token（如 `elif`）的概率和片段的平均概率呈上升趋势，模型陷入正反馈循环难以跳出。
- **缺乏专门的评估数据集**：当前没有针对代码生成重复问题的评测基准，难以系统评估方法效果。

## 核心贡献（创新点）
1. **首次形式化定义"结构重复"（Structural Repetition）**：将代码序列映射为上下文无关文法（CFG）的归约序列，定义结构重复模式；而内容重复可视为其子集。
2. **提出基于语法的解码方法 RPG**：利用下推自动机（PDA）实时归约生成代码为语法规则序列，结合后缀数组和LCP数组高效检测重复，并通过指数衰减动态调整token得分，本质区别在于从语法结构而非字面内容角度检测和惩罚重复。
3. **构建 CodeRepetEval 评测数据集**：覆盖三种场景（人工合成、代码生成基准、真实仓库），包含重复严重程度排序的样本，填补该领域评测空白。
4. **系统性实验验证 RPG 的有效性**：在 CodeRepetEval 三个场景上 EGP 和 CCP 指标显著优于基线；在 HumanEval 和 MBPP 上 Pass@1 相对提升最高达 11.3%。

## 方法详解
### 整体框架 RPG（Repetition Penalization based on Grammar）
RPG 是一个解码阶段的方法，不修改模型参数，通过三步实现重复检测与惩罚：

**Step 1: 归约到语法规则（Reduction to Grammar Rules）**
- 使用下推自动机（PDA）将生成token序列归约为语法规则序列。
- PDA 定义：七元组 $(Q, \Sigma, \Gamma, \delta, q_0, Z, F)$，其中 $Q$ 为状态集，$\Sigma$ 为输入符号集，$\Gamma$ 为栈符号集。
- 转移方程：$q_t, z_t = \delta(q_{t-1}, x_t, z_{t-1})$，token $x_t$ 被归约为语法规则 $\hat{x}_t = [q_t, z_t]$。
- 合并相邻相同归约结果得到归约序列 $\hat{R}_{1:t} = \text{merge}(\hat{X}_{1:t})$。

**Step 2: 重复检测（Detection of Repetition）**
- 使用前缀数组（Suffix Array）和最长公共前缀数组（LCP Array）高效检测重复。
- 时间复杂度 O(n log n)，空间复杂度 O(n)。
- 重复模式集合定义：
  $$\text{Rep}(X_{1:t}) = \{\hat{R}_{\text{Suf}[i]:\text{Suf}[i]+\text{LCP}[i]} \mid \text{LCP}[i] > 0 \land \text{Suf}[i-1] = \text{Suf}[i] + \text{LCP}[i]\}$$
- 条件确保检测到的是连续重复（consecutive repetition）。

**Step 3: 重复惩罚（Penalization of Repetition）**
- 动态权重函数：$\text{Pn}(x_t|x_{<t}) = \lambda^{\text{Count}(\text{Rep}(X_{1:t}))}$，其中 $\lambda \in (0,1)$ 为衰减因子。
- 调整后的token得分：$s'(x_t|x_{<t}) = s(x_t|x_{<t}) \cdot \text{Pn}(x_t|x_{<t})$
- 最终选择：$\text{RPG}(x_t|x_{<t}) = \arg\max_{x_t} s'(x_t|x_{<t})$
- 关键设计：重复次数越多，惩罚越强，打破自强化循环。

## 实验与结果
### 数据集
- **CodeRepetEval**（新构建）：三个场景
  - Artificial Synthesis：512样本，人工构造诱导重复的提示
  - Code Generation Benchmarks：512样本，来自 CodeLlama/DeepSeek Coder/CodeGen/ChatGPT 在 HumanEval/MBPP 上的重复生成
  - Real-world Repositories：1024样本，来自真实开源仓库
- **公共基准**：HumanEval、MBPP 及其扩展版 HumanEval-ET、MBPP-ET

### 评估指标
- **EGP**（End-of-sentence Generation Percentage）：成功终止重复的比例
- **TR-N**（Token-level Repetition）：短语级结构重复比例（基于归约序列的n-gram去重，n=4）
- **TR-S**（Statement-level Repetition）：语句级结构重复比例
- **CCP**（Compiler Correctness Percentage）：代码可编译比例
- **Pass@k**：功能正确性（执行测试用例）

### 主要结果
| 场景 | 指标 | 最强基线 | RPG | 提升 |
|------|------|----------|-----|------|
| Code Generation Benchmarks | EGP | Rep_Penalty: 0.721 | **0.912** | +26.5% |
| Code Generation Benchmarks | CCP | Rep_Penalty: 0.413 | **0.805** | +94.9% |
| Artificial Synthesis | EGP | Rep_Penalty: 0.679 | **0.871** | +28.3% |
| Artificial Synthesis | CCP | Rep_Penalty: 0.467 | **0.731** | +56.5% |
| Real-world Repositories | EGP | Rep_Penalty: 0.828 | **0.889** | +7.4% |
| Real-world Repositories | CCP | Topk(k=5): 0.411 | **0.638** | +55.2% |

- **HumanEval**：RPG Pass@1 = 0.325，较 Greedy (0.301) 提升 **↑8.0%**
- **HumanEval-ET**：RPG Pass@1 = 0.258，较 Greedy (0.232) 提升 **↑11.3%**
- **MBPP**：RPG Pass@1 = 0.421，较 Greedy (0.396) 提升 **↑6.4%**
- **MBPP-ET**：RPG Pass@1 = 0.334，较 Greedy (0.303) 提升 **↑10.3%**

### 泛化性实验
- **不同模型**：在 CodeLlama、CodeGen、DeepSeek-Coder、Llama2 上均有效
- **不同模型规模**：CodeLlama-34B 上同样有效
- **不同编程语言**：扩展至 Go 语言，EGP 从 0.133 提升至 0.875，CCP 从 0.403 提升至 0.725

## 相关工作脉络
1. **Greedy/Topk/Topp/Temperature Sampling**：基础解码策略；本文发现提高采样置信度反而加剧重复，与直觉相悖。
2. **Repetition Penalty (Keskar et al., 2019)**：对已出现token施加统一惩罚；作者指出其在代码生成中会误杀高频必要token（如 `=`、`[`）。
3. **Repetition Dropout (Li et al., 2023a)**：训练阶段随机mask重复n-gram；需要大量额外训练且对结构重复无效。
4. **Grammar-based Code Generation (Dong et al., 2023b, CODEP)**：利用语法规则增强生成正确性；但这些方法只关注语法合法性，无法解决已合法的结构重复问题。
5. **Code LLMs (CodeLlama, DeepSeek Coder, CodeGen)**：本文的基础模型；发现经代码微调的模型（如CodeLlama）在greedy采样下更易产生结构重复。

## 局限性与未来方向
- **计算开销略增**：RPG 需要额外运行 PDA 和后缀数组计算，但相比 LLM 推理本身开销可忽略。
- **重复成因未深入分析**：论文未解释 LLM 为何会产生结构重复，仅做了现象观察，留作未来工作。
- **λ 超参数需调优**：默认值 0.9，实验显示更小值抑制更强但可能过度惩罚，存在优化空间。

## 研究启发与可借鉴点
1. **语法视角的新范式**：将代码重复问题转化为语法规则序列的模式匹配问题，为重复检测提供了结构化新思路，可迁移至其他结构化生成任务（如公式、SQL）。
2. **后缀数组+LCP的高效重复检测**：O(n log n) 时间复杂度的在线重复检测算法，适合流式解码场景，值得在需要实时模式匹配的任务中借鉴。
3. **指数衰减的动态惩罚机制**：$\lambda^{\text{Count}}$ 的设计简洁有效，可根据重复频次自适应调整惩罚强度，可推广至文本生成中的长程重复抑制。
4. **CodeRepetEval 数据集构建方法**：基于真实重复样本按 TR-S 排序选取、人工构造提示的策略，为重复问题评测提供了可复用的构建流程。
5. **与团队方向结合机会**：若团队关注代码生成、程序合成或结构化文本生成，可将 RPG 的语法归约思想与 AST/控制流图结合，探索更细粒度的重复/死代码检测。

## 关键术语表
- **Structural Repetition（结构重复）**：代码片段在语法结构上重复（如多个 `elif` 分支），但具体内容不同的重复形式。
- **RPG（Repetition Penalization based on Grammar）**：论文提出的基于语法规则的解码方法，通过 PDA 归约和重复惩罚缓解代码生成中的重复问题。
- **Pushdown Automaton（下推自动机，PDA）**：具有栈结构的有限状态自动机，用于将 token 序列归约为语法规则序列。
- **CodeRepetEval**：论文构建的专门用于评估代码生成重复缓解方法的基准数据集，包含三种场景共 2048 个样本。
- **EGP（End-of-sentence Generation Percentage）**：衡量模型成功终止重复并输出 EOS 的比例。
- **TR-N / TR-S**：分别衡量短语级（n-gram）和语句级的结构重复程度，值越低表示重复越少。
- **CCP（Compiler Correctness Percentage）**：生成代码通过编译的比例，反映代码质量。
- **Suffix Array + LCP Array**：用于高效检测重复模式的字符串数据结构组合，支持 O(n log n) 时间复杂度的重复查找。

## 可复现要素
- **数据集**：CodeRepetEval 为新构建数据集，论文未公开具体链接（需联系作者获取）；HumanEval、MBPP、HumanEval-ET、MBPP-ET 为公开基准。
- **代码/权重**：论文未提及代码开源情况，基线方法引用原始论文设置。
- **关键超参**：衰减因子 λ = 0.9（默认），最大生成长度 1024（真实仓库场景 4096），温度 0.8（基线默认），评测取5次实验均值。
