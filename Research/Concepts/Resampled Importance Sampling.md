---
type: concept
title: "Resampled Importance Sampling"
user_level: Normal
aliases: [RIS, Resampled Importance Sampling, Importance Resampling, 重采样重要性采样, 重要性重采样]
prerequisites: ["Monte Carlo 估计器直觉（平均 = 估计）", "重要性采样直觉（pdf 越像 f，方差越小）", "期望与方差的初等概念"]
first_introduced: "1987（统计学 SIR，Rubin）；2005 图形学形式化（Talbot / Cline / Egbert, EGSR）"
---

# Resampled Importance Sampling

## Definition

**当"想要的目标分布 g 抽不动"时，从"抽得动的源分布 p"出发、用权重把差额补回来的采样技术**：

```text
① 从易抽的 p 抽 M 个候选 X₁…X_M
② 计权 w_j = g(X_j)/p(X_j)   （g 可为未归一化的"目标形状"）
③ 按权重比例重采样出 Y —— Y 的分布"近似"g
④ 用校正因子 (1/M)Σw_j 结清"近似"的账 → 无偏估计
```

一句话：**"样本从便宜的地方来，'像不像 g'由权重说了算，'差多少'由校正因子结账。"** 标准重要性采样是 M=1 的特例。

## Core Principle

**三层结构（本库记忆骨架）**：

| 层 | 内容 | 记住这一句 |
|---|---|---|
| 生成层 | p 抽 M 个候选；M 是**分布插值旋钮**（M=1→p，M→∞→g） | "多付候选的钱，买分布更像" |
| 修正层 | 重采样 + 校正因子 (1/M)Σ g/p 保无偏 | "校正因子 = 候选权重的平均"——**reservoir 的 W 的前身** |
| 预算层 | 方差向量被劈成两段：增 N 缩两段、增 M 只缩 e3−e2；等时间准则 **M = TY/TX** | "两段管线按单次成本之比分配预算" |

**无偏条件**：p、g 在 f 非零处严格正；M、N ≥ 1。
**边界**：若 g 不比 p 更接近 f，RIS 不占优；**"g 不包含的方差源（如 visibility），RIS 修不了"**（Talbot 原文的 70%→10% 实验）。

## Prerequisites

- Monte Carlo 估计：Î = 1/N Σ f(xᵢ)/p(xᵢ)，以及"方差 = 噪声"的直觉；
- 重要性采样：为什么 pdf ∝ f 时方差小；
- 会读"两个 pdf 的比值"（概率密度比 = 权重）——RIS 的全部魔法都在这一个比值上。

## Historical Evolution

```text
1987  Rubin：SIR（sampling/importance resampling）——统计学源头（缺失数据插补）
1995  Lafortune & Willems：resampling 减少 BDPT 阴影射线（CG 最早使用）
1996  Shirley et al.：直接光建议；2004 Burke：环境光乘积分布
★ 2005 Talbot / Cline / Egbert：形式化（估计器 + 方差分析 + 参数选择）——"RIS"得名
2020  Bitterli et al. ReSTIR："building on Talbot et al.'s RIS"——reservoir + 时空复用，进入实时渲染
2021  ReSTIR GI（路径空间）；2022 GRIS / ReSTIR PT（补"相关样本"的理论）
2021- RTXDI：工程化（DI/GI/PT 模块）；2020s 进入引擎产品线（随机直接光一族）
2026  读者框架：确认其为"采样 → 复用 → 重建 → 失效"四阶段的第一阶段底座
```

## Important Papers

- [[Talbot — Importance Resampling for Global Illumination (2005)]]——**定义者**（★ 2026-10-14 入库：三步 + 校正因子 + 方差劈分 + 等时间准则；直接光 10–70%）；
- **Bitterli et al. 2020（ReSTIR DI）**——首个大规模应用（reservoir 化 + 时空复用）；**待入库**（下一步的头号候选）；
- **Lin et al. 2022（GRIS）**——把 RIS 的"样本独立"假设放松到"相关样本"（ReSTIR 迭代复用的理论修补）。

## Related Concepts

- [[Reservoir Sampling]]——**容器侧**：RIS 的重采样在实现上由加权 reservoir（单槽流式）完成；"校正因子"演化为 reservoir 里携带的 W；
- [[ReSTIR]]——**实时化下游**（2026-10-15 建立）：流式 RIS + 时空复用 + MIS 无偏化；本篇的三步与校正因子是它的直接数学基座；
- [[Rendering Equation]]——被估计的积分；[[Global Illumination]]——应用域；
- [[Importance Sampling]] 无独立页——其"抽不动"的边界正是 RIS 的起点（本页 Definition 已覆盖）；
- 家族对照：**"一个表示代表全部"（IR 1997 的 VPL 场 / Zinke 2008 的原型路径 / reservoir 的候选集合）**——"用少量保留的统计量代表大规模中间集合"。

## Technologies

- **RTXDI**（NVIDIA）：RIS/ReSTIR 的工程化载体（含 DI/ReGIR/GI/PT 模块）；
- **随机直接光产品线**（MegaLights 一类"每像素采样预算 + 引导 + 重建"系统）：采样阶段的同族框架——对照口径以 [[Modern Real-Time Lighting Framework|读者专题报告]]的版本证据表为准。

## Game Applications

- 数千动态光源的实时直接光（"每像素几条射线"预算下，抽样分布决定噪声）；
- 任何"先生成候选、再对少量入选者做昂贵计算"的管线（阴影射线、GI 连接、光源选择）——**"用便宜候选换贵样本"**。

## Personal Knowledge

- **Normal（推断）**：读者 10-13 报告已把 RIS 的地位与机制写准（"先从易抽取的分布产生候选，再向目标分布重采样"）——概念层已接触；**方差公式推导层未验证**，保持 Normal；
- 与工作背景接口：**"两段预算分配（等时间准则 M=TY/TX）"可直接翻译进特效/渲染管线的预算语言**（先测两段单次成本，再按比例分配数量）。

## Learning Gap

- 方差劈分公式（e1/e2/e3）的推导——可跳过（结论可用）；
- RIS × MIS 的组合方式（硕士论文版有章节，按需）；
- 从 RIS 到 reservoir 的字段级对应（等 ReSTIR 原文入库补齐）。

## Next Step

1. （≈20 分钟）读 [[Talbot — Importance Resampling for Global Illumination (2005)]] 结论层——三行公式 + "等时间准则"一句即可；
2. ✅ ReSTIR DI（2020）已入库（2026-10-15）——对照"校正因子 → W"的演化已完成（见 [[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)|ReSTIR DI]] 的 §4：多源 PDF 的 |Z(y)|/M 偏差与 MIS 修正）；
3. 把"等时间准则"写进个人预算方法论草稿（两段管线各测一次单次成本）。
