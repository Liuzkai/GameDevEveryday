---
type: paper
title: "Grid-Free Monte Carlo for Time-Dependent Diffusion"
authors: [Zihong Zhou, Rohan Sawhney, Eugene d'Eon, Wojciech Jarosz]
year: 2026
published: "2026-09-11"
venue: "arXiv preprint (cs.GR; math.NA)"
url: "https://arxiv.org/abs/2609.12306"
code: ""
project_page: ""
category: [rendering, simulation, monte-carlo]
importance: A-
historical_importance: 0
game_relevance: 3
production_readiness: Research
user_level: Hard
status: unread
---

# Grid-Free Monte Carlo for Time-Dependent Diffusion

## TL;DR

Walk-on-Spheres 谱系（Sawhney 一脉）从**稳态** PDE 推广到**时间依赖**热方程：给每条随机游走一个"时间预算"，每个空间步采样一个 exit time——时间用完就采内部点求初值，没用完就扣掉预算继续走并累积源项/边界贡献。结果：**不需要体网格、不需要时间步进、没有步长选取问题、没有时间离散化偏差**，直接在任意指定时刻输出解，且保留 WoS/WoSt 的并行、渐进、按需求值特性。共享同一条 walk 还能一次估计多个目标时刻。

## Problem

热方程 / 扩散方程的瞬态（transient）求解在复杂几何上一直被两座大山压着：

1. **空间离散化**：体网格生成（volumetric meshing）在复杂几何上又贵又脆；
2. **时间离散化**：时间步进必须顺序执行（第 t+1 步依赖第 t 步），步长选取直接决定稳定性与精度——显式格式有 CFL 限制，隐式格式每步要解线性系统。

Grid-free Monte Carlo（WoS 2020 / WoSt 2023）已经解决了第一座山，但**此前基本只能解稳态问题**（Laplace / Poisson 的均衡解）。很多科学计算与图形问题关心的是"随时间怎么演化"，而不只是终态。

## Core Idea

把时间当作第二条维度编入随机游走本身，而不是在外面包一层时间步循环：

```text
每条 walk 携带一个有限时间预算 T
    ↓
每个空间步：照常走球/走星 + 采样一个 exit time τ
    ↓
若 τ 超出剩余预算 → 在球内采一点，求初值，walk 终止
若 τ 未超出 → 预算减去 τ，继续走，沿途累积源项与边界贡献
```

技术贡献是一套 **kernel sampling 与方差缩减技术**：低偏差、免查表的 exit time 采样器 + 高效拒绝采样器。这是全文最硬的部分——exit time 的分布来自热核（heat kernel），朴素采样要么有偏要么要预计算大表。

## Why It Works（谱系定位）

| | 传统瞬态求解器 | 本文 |
|---|---|---|
| 空间 | 体网格 | 无网格（walk on spheres/stars） |
| 时间 | 顺序步进 + 步长选取 | **无步进，直接估任意 t** |
| 时间偏差 | 有（离散化） | **零** |
| 并行性 | 时间维本质串行 | walk 间完全并行 |
| 输出 | 全场全时刻 | 按需、逐点、渐进 |

妙处在于它把时间步进的"串行瓶颈"转换成了 walk 预算的"随机抽样"——和 Monte Carlo 渲染把"解积分方程"转换成"采路径"是同构操作。

## Limitations

- 目前只覆盖**热方程/纯扩散**：Dirichlet（WoS 推广）与混合 Dirichlet–Neumann（WoSt 推广）；对流、反应项、非线性扩散不在本次范围；
- 方差随时间预算与边界复杂度增长（MC 通病），文中靠采样器设计压低但未消除；
- 不是为实时设计的——单点求值仍是"若干条 walk"的成本，定位是科学计算/离线仿真/几何处理工具；
- 几何输入仍继承 WoS/WoSt 的假设（边界表示、最近点查询效率）。

## Game Development Relevance

- **直接相关度中等，方法论相关度高**：游戏内不会跑热方程求解器，但 VFX/工具链会——热扩散、损伤传播、程序化材质的老化/灼烧、离线烘焙类工具都可能落在"复杂几何上的瞬态扩散"这个问题类上；
- **d'Eon 血统值得注意**：他是次表面散射扩散剖面（quantized diffusion）的作者之一，本文的"时间依赖扩散"与 SSS 的扩散近似在数学上同源——[[Physically Based Rendering]] 里 skin/wax 材质的扩散项背后是同一族热核；
- **对照样本**：与 9-14 的 [[2026-09-14-Gaussian Light Transport]] 构成有趣对偶——GLT **消掉采样**（用残差优化替代 Monte Carlo），本文**消掉时间步**（把时间编入 Monte Carlo）。同期两篇都在"从经典求解器里删一个离散化维度"。

## Unreal Engine Relevance

- 无直接引擎映射；潜在落点在编辑器/DCC 侧的几何处理与仿真工具（Houdini 类管线中 WoS 已有实际应用先例）；
- 对 Niagara/chaos 无影响。

## Technology Evolution

Monte Carlo 求解 PDE 谱系：

```text
Monte Carlo 解积分方程（Kajiya 1986 渲染方程，见 [[Kajiya — The Rendering Equation (1986)]]）
        ↓
Walk-on-Spheres（稳态椭圆 PDE，图形学化：Sawhney & Crane 2020）
        ↓
Walk-on-Stars（混合边界，Sawhney et al. 2023）
        ↓
★ Grid-Free Monte Carlo for Time-Dependent Diffusion（时间维，2026）
```

这条线的总纲：**每扩展一次，就再删掉一种网格/离散化依赖**。

## Relationships

### Based On

- Walk on Spheres（Sawhney & Crane 2020）
- Walk on Stars（Sawhney, Miller 等 2023）
- 热核理论 / Feynman–Kac 公式（扩散过程与 PDE 的概率表示）

### Contrasts

- [[2026-09-14-Gaussian Light Transport]]（同周对偶：一个去采样，一个去时间步）
- 传统瞬态 FEM/FVM 求解器（网格 + 步进的双重离散化）

### Related

- [[Neural Physics Simulation]]（替代路线：学习求解器）
- [[Physically Based Rendering]]（d'Eon 扩散剖面 SSS 的数学同源）

## Personal Knowledge State

`Hard`：Feynman–Kac、热核采样不在你的现有前置里。**读法建议：只取两个认知——①"时间可以编成 walk 的预算"这个 trick；②"去网格化"谱系的最新一站。** 推导细节跳过。

## Notes

- 作者阵容是 WoS 谱系核心：Sawhney（WoS/WoSt 一作）+ d'Eon（SSS 扩散）+ Jarosz（Dartmouth，体渲染/密度估计老将），可信度高；
- arXiv 阶段、未见 venue 接收标注，后续跟进是否进 SIGGRAPH/TOG。
