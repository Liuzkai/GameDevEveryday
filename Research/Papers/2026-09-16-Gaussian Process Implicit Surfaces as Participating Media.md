---
type: paper
title: "Gaussian Process Implicit Surfaces as Participating Media: Realization-Free Rendering from Level-Crossing Statistics"
authors: [Jack Cui, Kehan Xu, Eugene d'Eon, Wojciech Jarosz]
year: 2026
published: "2026-09-13"
venue: "arXiv 2609.14695 (cs.GR)"
url: "https://arxiv.org/abs/2609.14695"
code: ""
project_page: ""
category: [rendering, participating-media, microfacet-theory, stochastic-geometry]
importance: A-
historical_importance: 0
game_relevance: 3
production_readiness: Research
user_level: Hard
status: unread
---

# Gaussian Process Implicit Surfaces as Participating Media

## TL;DR

把「随机隐式曲面（GPIS）」和「参与介质」在**两个方向上**打通：用 Kac–Rice 水平穿越公式直接从一个高斯过程的逐点统计量解析出各向异性辐射传输方程（RTE）参数，**不再需要采样任何具体几何实现**（realization-free）。同一套统计结构顺带推导出全球面 Beckmann 与 GGX 法线分布、解析 masking–shadowing 函数，并证明在 height-field 极限下其局部条件近似**退化为 Smith 的独立性假设**。

一句话：**GGX 和 Smith 从"假设"变成了"可被推导出的极限"。**

## Problem

场景表示自图形学诞生起就有一个二分法：一个物体要么是**硬表面**，要么是**参与介质**。这个二分不只是哲学问题：

- 逆渲染 / 三维重建出来的表面天然带**不确定性**（噪声、缺失视角），但经典表面模型没有地方放"不确定"；
- **滤波与 LOD** 把硬表面变成体积效应——远处的树叶、栅栏、烟雾边缘——既有表面式反射，又有介质式透过，经典 RTE 表示不了这种相关性；
- 于是每一类随机几何各有一套自己的统计量、自己的光传输平均，**彼此不能对话**。

Seyb 等人（SIGGRAPH 2024）首次把 microfacet 表面和参与介质放进同一个框架，但代价是：要渲染就得**采样显式的几何实现**（realization），既要生成 realization，又要做求交，成本高到不实用。

## Historical Context

这条线是 Dartmouth × NVIDIA 四年四连：

```text
Seyb, d'Eon, Bitterli, Jarosz — SIGGRAPH 2024 (TOG 43(4), 112)
"From microfacets to participating media: A unified theory of light transport with stochastic geometry"
→ 建立统一理论：随机隐式曲面可表达硬表面 / 微面 / 介质 / 中间连续体；保留空间相关性
        ↓
Xu, Bitterli, d'Eon, Jarosz — SIGGRAPH Asia 2025
"Practical Gaussian process implicit surfaces with sparse convolutions"
→ 稀疏卷积，让 GPIS 表示变得可算
        ↓
Zhou, Seyb, Zhao — SIGGRAPH 2026
"Adaptive Ray Marching for Rendering Gaussian Process Implicit Surfaces"
→ 在线采样 + 自适应步进，等时间下 MSE 降 46×
        ↓
★ 本文（2026-09）：连 realization 都不采样了
```

注意这里的推进方式：前三步都是**把 realization 采样做得更快**，本文是**换了一条路——直接从统计量解析出 RTE 参数**，realization 这个中间产物被整个删掉。

## Previous Work

- **Seyb et al. 2024**：统一理论的起点，但渲染靠采样 realization，慢；
- **Xu et al. 2025 / Zhou et al. 2026**：把采样加速到可用，但**没有绕开"必须先有 realization"这件事**；
- **Microflake theory（Heitz et al. 2015）**：用方向分布描述介质的微观薄片，能表达各向异性介质，但和"曲面"是两套东西；
- **SGGX（Dupuy et al. 2016）**：SGGX 分布把微面法线分布推广到体积，是本文要"作为特例恢复"的目标之一。

## Core Idea

GPIS 把几何表示为**一个高斯过程的零水平集**：GP 的每一次采样（realization）给出一个可能的曲面。关键是——**渲染要的是所有 realization 上光传输的平均值，而不是某一个 realization**。

既然要的是平均值，那就不该先生成 realization 再平均，而应该**直接从 GP 的统计量算这个平均**。

工具是 **Kac–Rice 水平穿越公式**（随机过程理论里数"零穿越点"的经典结果）。在**局部条件近似**下，作者从 GPIS 的逐点均值和协方差直接推出：

- 消光系数 σ_t
- 散射系数 σ_s
- 各向异性相位函数

即完整的各向异性 RTE。几何一致性由一个**共享的投影面积**保证——它同时耦合 extinction 和 scattering，所以得到的体积表示和原 GPIS 在几何上是自洽的，不会"看着像但物理上不是同一块东西"。

## Technical Approach

1. **正向**：GPIS 逐点统计 →（Kac–Rice + 局部条件近似）→ 各向异性 RTE 参数 → 标准体积渲染器直接渲染。整个谱系连续覆盖：参与介质（左）→ 多孔 / 非高度场几何（中）→ 硬表面（右）。
2. **副产品（对微面理论）**：同一套统计结构推出**全球面 Beckmann 与 GGX 法线分布函数**，支持面内与面外各向异性；可证明恢复 **SGGX / Beckmann / GGX** 作为特例；支持**精确可见法线重要性采样**；并给出解析 masking–shadowing 函数与镜面微面的单次散射表面模型（可扩展到多次散射）。
3. **极限关系**：在 height-field 极限下，证明局部条件近似退化为 **Smith 的独立性假设**。
4. **反向**：刻画与给定 RTE 参数相容的 GPIS 族，给出异构密度场的实用 lift。已有体积资产（OpenVDB 一类）因此可以当作 GPIS 渲染；训练好的辐射场重建能**不经网格提取**直接给出表面几何与着色法线，并给出**几何不确定性的密度式表示**。

## Key Contribution

1. 双向理论：GPIS ↔ 参与介质，而不只是单向近似；
2. **realization-free**：删掉"生成 realization"这一步，效率优于 realization-based 方法，且能塞进标准体积渲染器；
3. 把三个已知分布（SGGX / Beckmann / GGX）**统一为一个更一般框架的特例**；
4. 给出 Smith 独立性假设的**理论来源**（height-field 极限）；
5. 反向 lift 让"不确定性"变成可渲染的密度——这是重建与渲染之间缺失的一环。

## Why It Works

关键在于它换掉了平均的顺序。原来：

```text
采样 realization → 求交 → 渲染 → 多次结果求平均（MC 收敛慢）
```

现在：

```text
GP 统计量 → 解析 RTE 参数 → 一次体积渲染（平均已包含在解析式里）
```

把"对 realization 的蒙特卡洛平均"换成了"对统计量的解析表达"——噪声和 realization 生成开销同时消失。这是求解器设计里反复出现的那一刀：**不要模拟集合再平均，直接算集合的统计量。**

## Limitations

- **局部条件近似**是核心近似（不是精确的 Kac–Rice）；其误差边界在强相关 / 长程相关区域尚不明；
- 论文自己是离线体积渲染器，**没有实时路径**，不要和"能进引擎"混为一谈；
- 反向 lift 面向异构密度场，尚不涉及大规模真实场景；
- 论文 24 页 17 图，理论密度高——**不建议从头读推导**。

## Game Development Relevance

对游戏/实时，短期无落地可能，但有三个不该错过的信号：

1. **"表面 ↔ 介质连续体"正是 VFX 日常在手工伪造的东西**。Niagara 里粒子软化、卡片软边、远处 LOD 把硬几何糊成体积——这些本质上都是"介于表面和介质之间"。Seyb 2024 的原文直接点名了 LOD 与滤波会产生这种中间态。今天没有引擎能正确表达它，所以美术只能靠手感调。
2. **OpenVDB → GPIS 的反向 lift** 是资产侧的一个可能方向：体积资产将来也许能直接当概率表面渲染，省掉网格提取。
3. **几何不确定性 = 密度**：三维重建（含生成式重建）的不确定区域可以直接渲染成"雾"，而不是一个错误的硬表面。这对"生成式资产进入生产管线"的质量评估是一个可想象的接口。

## Unreal Engine Relevance

无直接映射。硬要给的话，Niagara 的参与介质类效果（烟雾/体积雾）在理论上属于本文 RTE 那一侧；但本文没有实时算法，不应作为 UE 侧的技术选项讨论。

## Technology Evolution

```text
微面理论（Torrance-Sparrow 1967 → Blinn 1977 → Cook-Torrance 1981）
        ↓
★ Walter 2007：GGX 分布 + Smith height-correlated masking-shadowing（成为工业默认）
        ↓
随机几何统一（Seyb 2024：微面 ↔ 介质连续体）
        ↓
实用化（Xu 2025 稀疏卷积 → Zhou 2026 自适应步进）
        ↓
★ 本文 2026：从随机几何反推 GGX / Smith，假设降级为极限
```

对个人知识图谱的意义：这条链把 [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] → [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] → 本文串成**微面理论 45 年闭环**。

## Relationships

### Based On

- [[BRDF]]、[[Microfacet Theory]]（本文恢复并推广了其中的 D 与 G）
- Kac–Rice 水平穿越公式（随机过程）
- Chandrasekhar 1960 辐射传输方程
- [[Participating Media]]（目标表示）

### Extends

- Seyb et al. 2024（SIGGRAPH）：从"需要 realization"推进到"realization-free"
- Microflake theory / SGGX：被本文统一为特例

### Related

- [[2026-09-15-Grid-Free Monte Carlo for Time-Dependent Diffusion]] — **同一个 d'Eon × Jarosz 组合，连续两天**。昨天删掉的是"时间步"，今天删掉的是"realization"。这是本周"删一个中间产物"主题的第三、第四个实例。
- [[2026-09-14-Gaussian Light Transport]] — 同周第三个"从经典求解器里删东西"的样本（它删的是采样）
- [[Physically Based Rendering]] — 本文把 PBR 的 D/G 两项从工程近似还原为可推导结果
- [[Gaussian Splatting]] — 同属"表示革命"，但方向相反：GS 是显式基函数复兴，本文是概率表示统一

### Followed By

- （待观察）是否进 SIGGRAPH / TOG；实时化是否有后续工作

## Personal Knowledge State

**Hard**，但有一个例外：**关于 D（法线分布）和 G（masking-shadowing）的部分，你今天就能读，且值得读。**

理由：你正在研读 Cook-Torrance 的 D/G/F 物理来源。本文做了两件直接服务于这件事的事：

1. 从更一般的随机几何**重新推导出 GGX 和 Beckmann**，说明它们不是"拟合出来的形状"，而是某种统计结构的必然结果；
2. **证明了 Smith 独立性假设是 height-field 极限**——也就是说，你在 UE shader 里用的那个 G 项近似，它的前提（高度与斜率不相关）现在有了一个明确的理论位置和失效边界。

建议读法：跳过第 1–4 节的全部推导，只看关于 NDF 恢复与 Smith 极限的那两三个结论 + 图。10 分钟。

## Learning Value

- **认知点 1（可迁移）**：求解器优化的两条路——"把中间步骤做得更快" vs "把中间步骤整个删掉"。本文走了第二条。这与 [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]]、[[From Splats to Silicon — Rethinking Computational Efficiency of 3DGS]] 是同一课：**组件加速 ≠ 端到端加速，但"删除组件"几乎总是端到端加速。**
- **认知点 2（直接有用）**：GGX + Smith 不是经验拟合，而是有推导来源的；记住"height-field 极限"这个定位，日后看到任何"改进 G 项"的论文，可以先问"它放松了 Smith 的哪个假设"。

## Notes

- arXiv 提交日 2026-09-13，进入 9-15 listing（周二），属 24h 窗口。
- 单位：Jack Cui / Kehan Xu / Wojciech Jarosz = Dartmouth College；Eugene d'Eon = NVIDIA（瑞士）。这解释了为什么 d'Eon 连续两天出现——他同时在做无网格 MC 求解器（昨天）和随机几何理论（今天）。
- 关键词（论文自列）：Gaussian process implicit surfaces, stochastic processes, **microfacet theory**, microflake theory, light transport.
