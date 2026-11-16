---
type: paper
title: "Precomputed Atmospheric Scattering"
authors: ["Eric Bruneton", "Fabrice Neyret"]
year: 2008
published: "2008-06"
venue: "Computer Graphics Forum 27(4), pp.1079–1086（EGSR 2008 Special Issue）"
url: "https://inria.hal.science/inria-00288758"
code: "https://github.com/ebruneton/precomputed_atmospheric_scattering"
project_page: "https://ebruneton.github.io/precomputed_atmospheric_scattering/"
category: ["Rendering", "Atmospheric Scattering", "Precomputed Radiance Transfer", "LUT", "Real-Time"]
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（Unity HDRP Physically Based Sky 的公开文档明确基于此法；十多年里所有「物理天空」的商业实现几乎都从这份代码起步）"
user_level: Normal
status: unread
---

# Precomputed Atmospheric Scattering

> **本篇笔记的特殊说明**：截至 2026-11-16 未能取得开放的全文 PDF（HAL 主机在本环境的 TLS 握手失败，ACM DL 需订阅）。因此本文的**结构与数字全部来自三处可核验的二手来源**，并逐处标注出处：① 论文自身的摘要（HAL 元数据，逐字可核）；② [[Bruneton 2017 — A Qualitative and Quantitative Evaluation of 8 Clear Sky Models|Bruneton 2017]] 第 10 节的算法与复杂度描述（**同一作者**，逐条对应原论文的 Algorithm 4.1）；③ [[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 第 5.2、7 节对本文代码的实测。**没有表号/图号的地方，我不编。**

## TL;DR

**它把"天空"从一次积分变成一次查表，而代价全部预付在一张 4 维网格上。**

三句话：

1. **观测点、视线方向、太阳方向**三个自由度（$n_r \times n_\theta \times n_{\theta_s} \times n_\gamma$）全被枚举进一张 4D LUT——**包括太阳在地平线以下的那些角度**，这是它能画出日落与地球阴影的原因；
2. 多阶散射是**迭代**算的：用第 $n-1$ 阶的散射 LUT 去算第 $n$ 阶，一阶一遍；
3. 于是运行时是 **$O(1)$**（无采样、无积分），而**代价是预计算 $O(n^6)$ 时间 / $O(n^4)$ 内存**，以及一个直到 2020 年才被解决的后遗症——**改一次大气要 250 ms**。

| 指标 | 数值 | 出处 |
|---|---|---|
| 预计算时间复杂度 | $O(n^6)$ | Bruneton 2017 Table 2 |
| 预计算内存 | $O(n^4)$ | 同上 |
| 运行时 | $O(1)$，无采样 | 摘要 + 同上 |
| 与实测的 RMSE | **11.3** $\mathrm{mW/(m^2{\cdot}sr{\cdot}nm)}$（八模型中最准） | Bruneton 2017 Table 2 |
| 渲染一帧（1280×720） | **0.22 ms**（**不含** LUT 更新） | Hillaire 2020 §7 |
| 全量更新一次 LUT | **250 ms**（99% 在多阶迭代） | Hillaire 2020 §7 |

## Problem

2008 年的处境，用摘要里的一句话概括：**要"从地面到太空的任意视角"、"实时"、"含 Rayleigh 与 Mie 的多次散射"三件事同时成立。**

当时没有一条路能同时满足：

- **ray march**（Nishita 1993/1996）能算，但每像素积分，且换太阳角要重算体素；
- **解析拟合**（Preetham 1999）快，但只能地面视角、只有一种大气；
- **Haber 2005** 有全阶散射，但**一个太阳天顶角一张 3D 网格**，太阳一动就要重烘。

**本文的关键判断是：把"太阳天顶角"也变成 LUT 的一个维度。** 这个决定同时解决了两个问题——太阳可以任意动（包括落到地平线以下），且预计算只做一次。

## Core Idea：为什么必须是 4 维

把渲染方程沿视线积分，被积的量（in-scattering）在每一点依赖三个方向/位置的自由度再加上太阳方向：

| 维度 | 含义 | 为什么不能省 |
|---|---|---|
| $n_r$ | 观测点高度 | 大气密度随高度指数变化，不能当均匀 |
| $n_\theta$ | 视线天顶角 | 同一高度、不同仰角，光学厚度差几十倍 |
| $n_{\theta_s}$ | **太阳天顶角** | 日落、地球阴影、晨昏线全在这一维里 |
| $n_\gamma$ | 视线与太阳的夹角 | 相函数（Rayleigh 的 $\cos^2$、Mie 的强前向峰）只依赖这个夹角 |

**四维 ⇒ 预计算 $O(n^6)$（每个格点要对全天球积一次）、内存 $O(n^4)$。** 这就是它贵的结构性原因，不是实现问题。

**而它换来的运行时是 $O(1)$**：查表 + 插值，没有采样、没有迭代。摘要原话：*"evaluate at runtime the light transport equation in constant time, without any sampling, while taking into account the ground for shadows and light shafts."*

## Technical Approach（据 2017 作者自述还原）

**① 透射率用 2D 表**：以"同心球层 × 朝太阳方向的柱面"的交点作为采样点，预存"从该点到大气顶"的透射率。这是 Nishita 1993 就有的结构，本文把它做成了可复用的 LUT（Hillaire 2020 至今仍沿用这个参数化）。

**② 多阶散射 = 迭代**：第 $n$ 阶的散射 LUT，由第 $n-1$ 阶的 LUT 在**全天球**上积分得到（不是 Nishita96 的 8 个共面方向，而是全方向）。这是精度上比 Nishita96 高一个档次的直接原因——[[Bruneton 2017 — A Qualitative and Quantitative Evaluation of 8 Clear Sky Models|Bruneton 2017]] 的对照里，两者 RMSE 是 11.3 vs 18.3。

**③ 迭代的代价就是那个 250 ms**：地球大气 5 阶就够（2017 与 2020 两处独立确认），但**阶数 × 一次全表遍历 = 250 ms**；一旦让美术把大气调稠密，阶数要往几十走，代价线性上涨。**这是"$O(n)$ 更新"的根源。**

**④ 省内存的两个技巧，两个都有代价**：

- **Mie 通道压缩**：单次 Mie 散射的 RGB 三通道被压成"一个标量 + 一个近似重构"（RGBA 里 A 存"无色 Mie"），省下 2/3 的散射 LUT。**代价**：日落的橙色会被洗淡——这是 [[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 拿来当对照组短板打的那一点（Hillaire 2020 图 9 第二列）。
- **太阳光谱取常数**（原文实现里 $S(\lambda)=c$）。**代价**：物理上更不准，但**意外起到了白平衡作用**——[[Bruneton 2017 — A Qualitative and Quantitative Evaluation of 8 Clear Sky Models|Bruneton 2017]] 的感知实验里，被试反而觉得这个版本更真实（见该篇「感知实验」一节）。

**⑤ 光柱（light shafts）用 shadow volume**：山影投进大气，靠几何体挤出而不是每像素采样。这条在 2020 年被否掉了——**LUT 结构不允许沿视线做积分采样**，所以新方法只能 ray march（Hillaire 2020 §7 明说）。

## Historical Context

```text
1993  Nishita      单散射 ray march              能算但每像素积分
1996  Nishita      +二阶，8 方向体素              换太阳角要重烘
2005  Haber        球坐标全阶网格                 一个太阳角一张表
2008  ★ 本文        4D LUT（含太阳角）+ 全阶迭代    O(1) 运行时
2009  Elek         任意大气 + 海洋，15 波长
2013  Yusov        改参数化，压地平线伪影
2016  Hillaire     加时间切片，把 250 ms 摊到多帧
2020  Hillaire     几何级数替迭代 → 0.17 ms/帧    ← 250 ms 的账在这里才被清掉
```

**这十二年的主线不是"怎么算得更准"，而是"怎么让那张表的更新变便宜"。** 2008 把运行时压到 $O(1)$，代价是更新 $O(n)$；2016 用时间切片缓解症状；2020 才算治了根。

## Limitations（继承给后世的四条）

1. **更新贵**：250 ms／次，直接导致"天空只能烘一次"的工作流。→ 2020 解决。
2. **4D 参数化的地平线伪影**：空气透视靠"两个采样相减"实现，分辨率/参数化不够时地平线出现接缝。→ Yusov 2013 改进，2020 用"视锥体积"绕开。
3. **小星球/高曲率失准**：4D 网格的参数化按地球尺度设计，地面曲率一大就糊（[[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 图 10 的"tiny planet"一格是反例）。
4. **单张 RGBA 的 Mie 压缩**：省内存换来日落偏色。→ 2020 分表存解决。

## Game Development Relevance

- **它是"物理天空"这个品类的定义者。** Unity HDRP 的 `Physically Based Sky` 在公开文档里明确写着基于 Bruneton & Neyret 2008；十多年里绝大多数"物理大气"实现（含早期 UE 插件、各类开源天空）都从作者 2017 年开源的那份代码起步。
- **对本库的意义**：它是 [[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 的**对照组**，也是 [[Bruneton 2017 — A Qualitative and Quantitative Evaluation of 8 Clear Sky Models|Bruneton 2017]] 里"最准的那个模型"。**三篇连读，构成"方法—量尺—换代"的完整三角。**
- **一个仍然有效的判据**：如果你的天空是**静态 ToD 或一天只变几次**，2008 这套（+ 时间切片）在 2026 年依然够用且更简单；**只有当美术需要实时拖滑杆时**，2020 的收益才显现。

## Relationships

### Based On

- **Nishita 1993 / 1996**：单散射积分结构与 2D 透射率表的原始形式。
- **Haber 2005**：全阶散射的"在每点对全天球积分"的算法骨架（本文把"一个太阳角"扩成"$n$ 个太阳角"）。

### Extends

- **Nishita 1996**：从"8 个共面方向"扩到"全天球"，从"一个太阳角"扩到"全部太阳角"。

### Related

- [[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]]：**直接换代**——同一问题的下一次回答，把更新成本降了约 1,470 倍。
- [[Bruneton 2017 — A Qualitative and Quantitative Evaluation of 8 Clear Sky Models|Bruneton 2017]]：**同作者的量尺**，本篇在其中排第一（RMSE 11.3）。
- [[Participating Media]]：本篇是它在球壳几何（行星大气）上的完整实现。

### Followed By

- **Elek 2009/2010**（任意大气 + 海洋 + 光谱）→ **Yusov 2013**（参数化）→ **Hillaire 2016**（时间切片）→ **Hillaire 2020**（几何级数）。

## Personal Knowledge State

- **Normal（结构）**：4 维枚举 + 迭代多阶 + $O(1)$ 运行时——这三件事足以在评审里定位它。
- **Hard（实现）**：$\Delta J$ 的全天球积分（2017 论文称之为 Algorithm 4.1）、参数化里的 ad-hoc 常数、以及"两个采样相减"求空气透视的推导。
- **待补**：**全文未取**。下一步优先从作者 2017 年开源的代码仓库反读（`model.h` / 预计算 pass），比找 PDF 更快。

## Visualization

（本篇无原文图——未取全文。相关视觉证据见 [[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 的图 9：第二列就是本方法的输出，可与路径追踪、2020 方法逐格对照。）

## Notes

- **出处逐条**：摘要（含 "constant time, without any sampling"、"light shafts"）来自 HAL 元数据；复杂度 $O(n^6)/O(n^4)/O(1)$、RMSE 11.3、4D 网格的四个维度、以及"对全天球积分"来自 Bruneton 2017 §10；250 ms / 0.22 ms / 5 阶足够 / 透射率参数化沿用 / Mie 压缩技巧 / shadow volume 来自 Hillaire 2020 §5.2、§7。
- **未取全文的原因**：`inria.hal.science` 在本环境的 TLS 握手失败（schannel `SEC_E_INVALID_TOKEN`），`hal.science`、`hal.inria.fr/docs/...` 亦不可达；ACM DL 返回 403。**这属于通道问题，不是论文不存在**——用户在自己的浏览器里应当能直接下载。
- **不要引用本篇的表号/图号**：我没有原文，任何"Table 1 说……"都是编的。
