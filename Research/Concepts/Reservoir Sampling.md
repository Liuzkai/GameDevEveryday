---
type: concept
title: "Reservoir Sampling"
user_level: Normal
aliases: [Reservoir Sampling, Reservoir, 蓄水池采样, 水塘采样, WRS, Weighted Reservoir Sampling, 加权蓄水池采样]
prerequisites: ["概率正比于权重的抽样直觉", "加权平均/权重和概念", "（可选）流式算法概念（单遍、常数内存）"]
first_introduced: "1982/1985（统计学：Vitter / Chao 的加权蓄水池）；2020（图形学：ReSTIR 引入，RTXDI 工程化）"
---

# Reservoir Sampling

## Definition

**一个"常数内存、单遍流式"的加权抽样容器**：面对一个（可能无限长的）加权样本流，它始终保存**一个被选中的样本 + 少量统计量**，使得——流处理结束时，**每个样本被保留的概率正比于它的权重**；且多个这样的容器可以"合并"（组合两个流入、产出等价于合并后的一个流出）。

一句话：**"一个槽位代表整条候选流。"** —— 流式场景下的"用 O(1) 内存做加权抽样"。

## Core Principle

**加权蓄水池（WRS）的核心循环（Chao 1982；ReSTIR 采用的版本）**：

```text
状态：保留样本 y、累计权重和 w_sum
新候选 x（权重 w(x)）到来：
    w_sum' = w_sum + w(x)
    以概率 w(x)/w_sum' 替换 y ← x（否则丢弃 x）
```

**三条关键性质**：

| 性质 | 内容 | 为什么重要 |
|---|---|---|
| **正比于权重** | 结束时 y 被选中的概率 ∝ w(y)（相对全体流） | "按重要性抽一个代表"——**重采样**的流式实现 |
| **对归一化免疫** | 只需**相对权重**，w_sum 本身携带总量信息 | 目标分布 g 可以**未归一化**（与 RIS/重采样的"g 可未归一化"一脉相承） |
| **单槽可合并** | 两个 reservoir 可合并为一个（按 w_sum 竞争） | 时空复用（跨帧/跨像素合并）的机制基础 |

**在 RIS/ReSTIR 里的位置**：

- **对 RIS**：Talbot 2005 的"抽 M 个候选、按权重选一个"——WRS 是它的**单遍 O(1) 内存实现**（候选流式过一遍，不必先物化 M 个）；Talbot 的"校正因子 (1/M)Σw"在 reservoir 里演化为**随样本一起携带的权重统计量 W**（使 f(y)·W 成为无偏估计的"贡献权重"）；
- **对 ReSTIR**：reservoir 的完整形态 = "样本引用 + 权重统计量 + 候选计数 + W"，并支持**时间复用**（reprojection + 目标重评价）与**空间复用**（邻域 reservoir 合并）——读者报告的准确定位：*"reservoir 的意义是：用少量保留样本与统计量，代表处理过的候选集合"*（不是"最终像素的平均颜色"，也不是"完整灯光概率表"）。

## Prerequisites

- "以概率 ∝ 权重"的抽样语义（加权抽一个 = 反复加权抽 N 个的流式版）；
- 加权平均、权重和；期望的线性；
- （可选）流式/在线算法的意识：单遍、常数内存、可合并；

## Historical Evolution

```text
1962 ± 统计学："蓄水池抽样"（等权版：从 N 个里等概率保留 k 个）
1970s-80s 加权化：Vitter 1985 / Chao 1982——加权蓄水池（WRS）
2020  Bitterli et al.（ReSTIR DI）引入图形学：把直接光样本的"候选 → 保留 → 时空复用"装进 reservoir
2021  ReSTIR GI（路径空间复用）；RTXDI 工程化（库内 Reservoir.hlsli 等真实代码形态）
2022  GRIS：把"样本相关"下的复用理论补齐（W 的合法性条件）
2026  ★ 本概念入库（读者报告的代码地图：TemporalResampling / SpatialResampling / Reservoir.hlsli 的初读定位）
```

## Important Papers

- **Bitterli et al. 2020（ReSTIR DI）**——reservoir 进图形学的定义性应用；**待入库**；
- **Ouyang et al. 2021（ReSTIR GI）/ Lin et al. 2022（GRIS）**——扩展与理论；
- [[Talbot — Importance Resampling for Global Illumination (2005)]]——reservoir 里"权重统计量"的思想前身（校正因子）；
- [[Modern Real-Time Lighting Framework]]——读者报告：含 reservoir 的代码阅读地图（固定提交）+ "保留样本与统计量"的准确定位。

## Related Concepts

- [[Resampled Importance Sampling]]——**理论侧**：reservoir 是 RIS 重采样的容器化/流式化；
- "**一个表示代表全部**"家族：IR 1997（一组 VPL 场景灯光场）/ Zinke 2008（一条原型路径代表全部路径）/ GALA 2026（一套基代表全部身份）/ **reservoir（一个槽位代表全部候选）**——"用少量保留统计量代表大规模中间集合"的又一样本；
- [[Gaussian Splatting]]（其压缩/预算线）——工程上的"统计量携带"对照（如 SteadySplats 的方差统计量）。

## Technologies

- **RTXDI 3.1.0**：DI / ReGIR / GI / PT 的完整 reservoir 实现（读者报告已给出固定提交：`RTXDI-Library @ f12037f…`，含 `Reservoir.hlsli` / `TemporalResampling.hlsli` / `SpatialResampling.hlsli`——**可直接只读核验**）；
- 引擎内随机直接光系统（同族框架，口径见读者报告）。

## Game Applications

- 实时直接光/间接光的"每像素少量样本"场景（数千灯选择 + 复用）；
- 任何"海量候选、只留一个代表"的实时系统（光源选择、探针选择、候选路径）——**"单槽 + 统计量 + 可合并"是可迁移的容器设计**。

## Personal Knowledge

- **Normal（推断）**：读者报告已把 reservoir 的语义（"样本 + 统计量"，不是平均色/概率表）与代码入口写准——概念层已接触；字段级细节（W 的归一化公式、bias correction 策略）**待 ReSTIR/RTXDI 源码精读验证**；
- 与工作背景接口：reservoir 的"单槽可合并"（跨帧/跨像素）与特效系统的"预算池可合并"同构——**合并语义（怎么把两份统计量并成一份）是这类系统的核心设计问题**。

## Learning Gap

- W 的精确构造（目标权重 / 归一化 / bias correction 的不同路径）——需读 ReSTIR 原文或 RTXDI 文档（读者报告已给入口）；
- 时空复用的失效条件（灯 ID 消失、表面不匹配、历史长度）——同上；
- "相关样本"下无偏性的理论（GRIS）。

## Next Step

1. （≈15 分钟）沿读者报告的代码地图，只读 RTXDI `Reservoir.hlsli` 的 `RTXDI_StreamSample` / `RTXDI_UpdateReservoir` 两个函数——**看 WRS 循环的工程形态**；
2. （观察项）ReSTIR DI（2020）入库时做"概念 ↔ 原文"对读；
3. 一个跨域小练习：用 30 行伪代码写"单槽加权蓄水池 + 两流合并"，自测三条性质（可选）。
