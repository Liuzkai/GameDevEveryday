---
type: concept
title: "Motion Tokenization"
user_level: Hard
aliases: ["动作 token 化", "运动离散化"]
prerequisites: ["Motion Matching", "Neural Animation"]
first_introduced: "2022（MotionDiffuse / MDM 一类把动作当序列建模）"
tags: [animation, generative, representation]
---

# Motion Tokenization

## Definition

把连续的动作序列（关节旋转、位移、骨骼参数）**切成有限个离散符号**的过程——码本索引、VQ 潜变量、或"每帧一个 token"。切完以后，动作就可以被当成语言来建模：自回归、掩码、单次前向。

**它不是一个算法，是一个"表示决策"**：决定"什么算一个单位"。

## Core Principle

$$\text{动作序列}\ \longrightarrow\ \text{token 序列}\ \longrightarrow\ \text{可用的生成范式}$$

四种切法，代价与能力完全不同：

| 切法 | 一个 token 是什么 | 得到什么能力 | 丢掉什么 |
|---|---|---|---|
| **逐帧连续** | 一帧的关节向量（float） | 扩散/流匹配，质量最高 | 长序列成本、需迭代采样 |
| **逐帧离散（VQ）** | 码本里的一个索引 | 自回归、可掩码、可压缩 | 量化误差、码本容量上限 |
| **片段级（clip token）** | 一小段动作的潜变量 | **单次前向即可出动作**（2 ms 级） | 段内细节、切换处的连续性 |
| **块级（block）** | 一块（K 帧）的潜变量 | 块内全注意力 + 块间自回归 | 块边界需要显式约束 |

**关键判据**：**token 的第一个用途不是压缩，是"让某种生成范式成为可能"。** 想要 2 ms 单次前向（[[MotionBricks — Scalable Real-Time Motions|MotionBricks]]），就必须有片段级离散表示；想要帧级精确控制，就必须保留逐帧连续。

## 三条设计轴

1. **单位粒度**：帧 / 片段 / 块。粒度越粗，单次前向越便宜，可控性越差；
2. **离散 or 连续**：离散换来自回归与压缩，连续换来质量与可微；
3. **谁决定边界**：固定长度 / 语义事件（脚步落地、动作相位）/ 学出来的转移点。**边界不清，切换处一定抖。**

## 边界：什么时候**不该** token 化

**2026-10-27 新增（[[2026-10-26-Liu — PASPA Past as State Present as Attention|PASPA]] 的核心论证）**：

> **"过去"不该被 token 化保留，该被压成一个状态。**

它先量了时间尺度：风格特征的自相关在 100 帧后仍显著，局部运动特征 20 帧内就衰减。于是——**历史越长，token 越没用**：保留 1000 帧 token 的效果不如保留一个固定大小的状态（FGD 5.216 vs 5.710），而且 token 全历史法的成本随时长增长。

**更狠的一条**：把"现在"也压成状态，FGD **7.652**，比**完全不给历史**（7.065）还差。**token 化不是"更省"，它是一种表示选择，选错了比没有更糟。**

## 与检索的关系

| | [[Motion Matching]] | token 化生成 |
|---|---|---|
| 动作从哪来 | 数据库检索 | 模型合成 |
| 单位 | 数据库里的一帧（精确） | 码本索引 / 潜变量（有损） |
| 覆盖范围 | 只覆盖录过的 | 可外推 |
| 成本 | 检索 + 混合，$O(1)$ 级 | 一次前向 / 迭代采样 |

**混合形态是本库记录过的最好答案**（[[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph|运动缝合]]）：**结构交给搜索（聚类图最短路），细节交给生成（Transformer 过渡姿势）**——即"边界用检索定、段内用生成填"。

## Related Concepts

- [[Motion Generation]] —— 父概念；token 化是它的一条实现路线；
- [[Motion Matching]] —— 检索侧的对照；
- [[Neural Animation]] —— 神经化的整体框架；
- [[Animation Compression]] —— **同一件事的另一半**：压缩也要求"选单位"，只是目标从"可生成"换成"可还原"（误差界即档位）；
- [[Budget Units]] —— token 数是"容量"这一种计价对象，**而 PASPA / UniWorld 证明了它常常不是正确的那一种**。

## Important Papers

- [[MotionBricks — Scalable Real-Time Motions]] —— 片段级离散 + 单次前向，2 ms 级；
- [[2026-10-26-Liu — PASPA Past as State Present as Attention|PASPA 2026]] —— **反例**：历史不该 token 化，该压成状态；
- [[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph|运动缝合 2026]] —— 检索 × 生成的混合形态；
- [[UniMate — One Unified Model to Animate Diverse Skeletons]] —— 跨骨架的统一 token 空间。

## Personal Knowledge

**Hard**。但只需拿走一条：**"token 化 = 选单位"，而单位要匹配时间尺度**。

## Next Learning Step

读 [[2026-10-26-Liu — PASPA Past as State Present as Attention|PASPA]] §3.2.1 与表 3（约 15 分钟）——它用一组消融把"token / 状态"的取舍讲清楚了，比任何综述都直接。

## Notes

建立于 2026-10-27：此前本页在 [[Motion Generation]] / [[Motion Matching]] / [[Neural Animation]] / [[MotionBricks — Scalable Real-Time Motions|MotionBricks]] 四处被引用但从未建页（属**命名债/知识缺口混合**），今日由 PASPA 的"状态 vs token"论证提供了实质内容，故补齐。
