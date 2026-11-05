---
title: "Tree-of-Evolution-Tree-Structured-Instruction-Evolution-for"
source: https://aclanthology.org/2025.acl-long.14.pdf
model: agnes-2.5-flash
chunks: 1
summarized_at: "2026-11-05 16:42:23"
---

# 论文速读：Tree-of-Evolution-Tree-Structured-Instruction-Evolution-for

## 一句话总结
本文提出 Tree-of-Evolution (ToE) 框架，将代码指令合成建模为树状多路径并行演化过程，并结合基于 Challenge 与 Diversity 的质量评估函数与 Beam Search 剪枝机制实现优化驱动的数据迭代；仅用 75k 合成样本微调基础模型，即可在多项代码基准上达到或超越训练量达数百万的 SOTA 开源代码大模型 Qwen2.5-Coder-Instruct。

## 研究问题与动机
1. **人工标注成本高**：高质量代码指令数据依赖编程专家人工编写，成本极为昂贵，难以支撑大规模指令微调。
2. **单向合成限制多样性**：现有主流方法（Code Evol-Instruct、OSS-Instruct）仅沿单一轨迹扩展种子数据，未能充分探索指令空间，导致生成数据分布狭窄。
3. **随机性驱动难以保障质量**：既有方法依赖预定义策略或随机采样生成新指令，缺乏系统性质量把控；LIMA 等研究已指出低质数据会直接拖累模型微调性能。

## 核心贡献（创新点）
1. **提出树状多路径演化框架 ToE**，将代码指令合成从单链扩展转为树结构并行探索。与以往单向演化方法不同，ToE 在每层节点同时生成多条分支，显著提升了数据空间的覆盖广度。
2. **引入优化驱动演化机制**，将 Beam Search 与父子质量约束相结合，强制子节点质量不低于父节点。与依赖随机采样的方法不同，该机制将数据合成转化为确定性的质量爬坡过程。
3. **设计双轴质量评估函数 $V(s)$**，联合衡量 Challenge（复杂度）与 Diversity（多样性）。区别于以往仅依赖启发式规则或单一维度的筛选策略，该函数通过 LLM-as-a-Judge 与跨树 embedding 距离实现了可计算的客观评估。
4. **验证了高质量小样本的高效微调路径**，75k
