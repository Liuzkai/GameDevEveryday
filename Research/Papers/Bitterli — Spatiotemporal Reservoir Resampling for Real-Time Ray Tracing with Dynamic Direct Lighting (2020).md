---
type: paper
title: "Spatiotemporal reservoir resampling for real-time ray tracing with dynamic direct lighting"
authors: [Benedikt Bitterli, Chris Wyman, Matt Pharr, Peter Shirley, Aaron Lefohn, Wojciech Jarosz]
year: 2020
published: 2020-07-08
venue: "ACM Transactions on Graphics (Proceedings of SIGGRAPH 2020), 39(4), Article 148"
url: "https://doi.org/10.1145/3386569.3392481"
code: "https://research.nvidia.com/publication/2020-07_spatiotemporal-reservoir-resampling-real-time-ray-tracing-dynamic-direct"
project_page: "https://benedikt-bitterli.me/restir/"
category: [rendering, ray-tracing, sampling, direct-lighting, real-time]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal
status: read
---

# Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)

> **别名**：ReSTIR DI。作者阵容：Dartmouth（Bitterli/Jarosz）× NVIDIA（Wyman/Pharr/Shirley/Lefohn）——后来成为 RTXDI 的理论与实现源头。

## TL;DR

把 [[Talbot — Importance Resampling for Global Illumination (2005)|Talbot 2005 的 RIS]] 装进 1982 年的加权蓄水池采样（WRS）里做成**流式算法**，再利用蓄水池**可合并**的数学性质把相邻像素与上一帧的候选样本**免费地**纳入自己的重采样池——320 万动态发光三角形、每像素至多 8 条阴影光线、2080 Ti 上 15–50 ms；无偏版比当时最强的 light BVH 等误差快 **6–60×**，有偏版再快 **35–65×**（代价是几何不连续附近的能量损失）。**一句话：在采样概率层做"时空滤波"，而不是在像素颜色层做后处理滤波。**

## Problem

- 数百万动态光源的直接光：每像素逐灯打阴影射线不可行；找到"这个点最重要的灯"取决于可见性 V、BSDF ρ、灯的功率与形状的**乘积**——任何单一因素都不足以排序。
- 实时约束叠加：场景完全动态（不能预计算）、每像素只有几条射线的预算、没有时间构建/更新复杂加速结构。
- 当时两条路都不够：**light BVH**（Moreau 2019）质量好但要维护数据结构（动态场景下维护成本占比过高）；**降噪器**（SVGF 系）只能重建输入样本里已有的特征——**喂给它的样本太差，神仙也滤不出来**（论文图 2 的核心论证）。

## Historical Context

论文自己在 Related Work 里把这条线梳理得很清楚，可读作"数据结构派"的谱系与 ReSTIR 的立场声明：

```text
1994/1996  Ward；Shirley et al. —— 灯按"期望贡献"分重要/不重要（many-light 分类的开端）
    ↓
2005  Estevez & Kulla 2018；Yuksel 2019 —— light hierarchy：亚线性时间从许多灯里采样（离线）
    ↓
2019  Moreau et al. —— light BVH 做进实时（但动态灯要持续重建，成本相对帧预算过高）
    ↓
2020  ReSTIR —— 立场：把"维护数据结构"整个取消，用"流式候选 + 时空复用"替代
```

另一侧的理论根：**1982 Chao 加权蓄水池采样**（单遍流式选样，为磁带机而生）+ **2005 RIS**（[[Resampled Importance Sampling]]）+ **1995 MIS**（Veach-Guibas）。三件旧工具的组合，出现在一个新位置上。论文附录 B 还补了一条数学史：WUS/WIS 等 ratio estimator 本身有偏，**RIS 可以看成"把 ratio estimator 变无偏"的方法**。

## Previous Work

- **Light hierarchies / BVH**：质量好、亚线性，但 (a) 构建与更新成本在实时帧预算里过重 (b) 采样分布通常不含 BSDF（Moreau 含部分）。ReSTIR 实测：**light BVH 甚至打不过不带复用的 streaming RIS**（图 1/图 11）。
- **VPL / 辐照度缓存**（[[Keller — Instant Radiosity (1997)|Instant Radiosity]] / [[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)|Irradiance Caching]] 一族）：复用"带光的路径"，但每次复用都要重打可见性射线，复用规模受限。
- **降噪/重建**（[[Schied — Spatiotemporal Variance-Guided Filtering (2017)|SVGF]] 2017 等）：与本文**正交且互补**（图 2：ReSTIR 输出 + OptiX 降噪器效果最佳）。
- **MCMC（MLT 族）**：也复用路径，但需要大量样本收敛且有结构化伪影。
- **Talbot RIS 本身**：需要存下全部 M 个候选（内存 O(M)），且多源 PDF 混合时成本 O(M²)——都不能直接上 GPU 实时。

## Core Idea

三步，每步都是旧工具 + 一个关键观察：

1. **流式 RIS**（Alg. 3）：RIS 的 M 个候选不必存下来——边生成边喂给加权蓄水池，恒定内存，输出选中的样本 y 和权重和 wsum。
2. **蓄水池可合并**（Alg. 4）：**关键观察**——两个蓄水池合并，只需把对方的 y 当成一个权重为 wsum 的新候选喂进来，结果在数学上等价于对两条候选流整体做蓄水池采样。**不需要访问任何原始候选**，O(k) 时间。于是"邻居像素的候选池"可以像积木一样拼起来：空间拼 k 个邻居 × n 轮迭代 → 每像素有效候选数 knM；时间上再把上一帧的蓄水池拼进来（Alg. 5）——有效样本数随滤波核大小**指数式增长**，成本近似线性。
3. **多源 PDF 的无偏化**（Alg. 6 + §4）：邻居的目标分布 p̂q′ 各不相同，朴素合并有偏（|Z(y)|/M < 1 时反向 PDF 被系统性低估 → **变暗**）。修法：把 RIS 权重里的 1/M 换成 MIS 平衡启发式 m(x) = pz/Σpi——**O(M) 成本拿到无偏性**（Talbot 原方案要对每个候选评所有 PDF，O(M²)，在"评 PDF = 打射线"的场景下完全不可行）。

## Technical Approach

**完整管线**（Alg. 5）：

```text
每像素 M=32 个候选（按功率重要性采样发光三角形 + alias table；有环境图时 25% 候选来自环境图）
  → RIS 权重 w = p̂(x)/p(x)，p̂ = 无遮挡贡献 ρ·Le·G（不打可见性射线的近似目标分布）
  → 第一轮选中样本立即测可见性，被遮挡的蓄水池整池作废（visibility reuse：不让遮挡样本扩散）
  → 时域复用：motion vector 反投影取上一帧蓄水池，合并
  → 空间复用：n=2 轮 × 每轮 k=5 个随机邻居（30px 半径内低差异序列采样）
  → 有偏版的邻居拒绝启发式：深度差 >10% 或法线夹角 >25° 的邻居拒收
  → shade：f_q(y)·W，有偏版存 N=4 个蓄水池，无偏版 N=1
  → 下一帧复用前：上一帧的 M 截断到当前 M 的 20× 以内
```

**无偏版 vs 有偏版**：
- 无偏（Alg. 6）：合并时对每个来源蓄水池检查 p̂qi(y) > 0（若开了 visibility reuse 还要补一条阴影射线），用 |Z| 归一或 MIS 权重——**额外每邻居约 1 条射线**（k=3、n=1、N=1，参数收窄以对齐成本）。
- 有偏：完全不加校正射线，靠邻居拒绝启发式压偏差；残差表现为**几何不连续附近的变暗**（阴影边界、锐利高光、树冠类复杂几何最明显）。

**时域 M 截断（clamp 20×）**是容易被忽略但极重要的一笔：不加限制时 M 随帧数无界增长，**陈旧样本的权重会盖过新样本**——这是"复用历史"必须同时给出"历史失效条件"的最小范例。

## Key Contribution

1. **蓄水池可合并性**引入渲染：把"像素间的样本复用"从图像空间的后处理问题，变成采样概率空间的**流合并问题**——这是后续整个 ReSTIR 家族（DI/GI/PT/ReGIR）与 RTXDI 的技术基座。
2. **多源 PDF RIS 的 O(M) 无偏化**（MIS 平衡启发式进 RIS 权重），解决了 Talbot 方案在"评 PDF = 打射线"场景下 O(M²) 的不可行性。
3. **"过滤 PDF 而不是过滤颜色"**——把降噪从渲染完成后的后处理搬进渲染器核心（结论原话，见 Why It Works）。

## Why It Works

- **合并等价性**：蓄水池状态 (y, wsum, M) 足以完整代表"整条候选流被加权抽出一个样本"这个事件——所以合并操作不需要历史候选。这是全文的数学支点（Alg. 4 的正确性证明）。
- **相关性就是免费的样本**：相邻像素的 p̂（几何 × BSDF × 灯光）高度相关，邻居选中的好样本对我也大概率是好样本——**一个邻域共享一次候选生成**，有效样本数 knM 以近零成本放大。
- **无偏性来自权重的期望校正**：E[W] = (1/p(y))·Σm(xi)，只要 Σm = 1 就无偏——MIS 平衡启发式天然满足。图 7 的 1D 实验把三种选择（不修正=有偏变暗 / 1/|Z|=无偏但噪声爆炸 / MIS=又稳又无偏）画在同一张图里。
- **结论层的自我定位**（原文）："we have moved denoising into the core of the renderer and **filter PDFs rather than colors**"——在采样概率层复用保持无偏的可能性，在颜色层复用则只能降噪。

## Limitations

- 有偏版在几何不连续附近系统性变暗（能量损失）；误差-时间曲线上约 1 s 渲染时间后，无偏版反超（偏差项开始主导）。
- 无偏版每邻居多一条阴影射线，实时预算下参数被迫收窄（k=3/n=1/N=1）。
- 目标分布 p̂ 是"无遮挡贡献"——visibility 不在 p̂ 里，可见性噪声**不会因 M→∞ 消失**（只能靠 visibility reuse 缓解）——与 Talbot 2005 的"g 修不了它不包含的东西"一脉相承。
- 邻居拒绝启发式（深度 10%/法线 25°）是手调阈值；BSDF/遮挡剧变的邻域仍会产生偏差或浪费。
- 时域复用在快速运动/大面积 disocclusion 下历史失效——本文用 M 截断压权重，完整的失效处理留给了后续工作（RTXDI 的 confidence、MegaLights 的探索机制）。

## Game Development Relevance

- **MegaLights（UE 5.6+）的直接前驱**：stochastic direct lighting 的"每像素光照预算 + 选灯 + 可见性 + 重建"形态就是 ReSTIR DI 的产品化路线；读者框架文档中 ReSTIR 行的"复用储备池统计量并重采样样本、重新评价目标、处理来源域和历史有效性"全部对应本文机制。
- **RTXDI**：NVIDIA 把本文算法 + 后续 ReSTIR GI/GRIS 打包成 SDK；读者已固定的 RTXDI 3.1.0 提交里 `Reservoir.hlsli` 的结构几乎逐字段对应本文 Alg. 2。
- **动态灯预算问题的范式转换**：传统 tiled/cluster shading 是"减少影响每像素的灯数"（有偏、限区域），ReSTIR 是"聚合全部灯光贡献"（可无偏）——**"限灯数"与"算总账"是两种预算哲学**，前者是本库"动态灯光 ≤3/≤2/≤1/0"预算线的世界，后者是 MegaLights 的世界。
- 等时对比数字：2080 Ti，Bistro-day 23.6 ms（有偏，RMAE 0.39 vs Talbot 0.70）；Zero Day 11,000 动态灯 15 ms；Amusement Park 340 万动态发光三角 50 ms（Titan RTX）。

## Unreal Engine Relevance

- UE 5.6+ **MegaLights**：stochastic direct lighting，历史可见灯 guiding + 随机直接光重建——本文算法的产品化后代（版本证据见读者框架文档）。
- Lumen 的采样/复用层与本文共享"时空复用 + 失效检查"的骨架，但复用的是探针/Card 照明而非蓄池样本。
- 自研管线角度：ReSTIR DI 的参数集（M/k/n/N/拒绝阈值/clamp 倍数）本身就是一个**可分档的预算参数表**——SABC 各档下调 M/N 与邻居半径即可得到质量-成本阶梯。

## Technology Evolution

```text
1982  Chao 加权蓄水池采样（磁带机时代的流式选样）
1987  Rubin SIR（统计学；采样-重要性-重采样）
1995  Veach-Guibas MIS
2005  Talbot RIS（候选池 + 校正因子；内存 O(M)、多源 O(M²)）
2019  Moreau light BVH（实时数据结构派的天花板）
★2020 ReSTIR DI（流式 + 时空合并 + MIS 无偏化；"过滤 PDF 而非颜色"）
  ↓
2021  ReSTIR GI（Bitterli et al.；从直接光到间接光）
2022  GRIS（Wyman/Pantaleoni；generalized REServoir IS——统一理论）
2022  RTXDI SDK（DI/GI/PT + ReGIR）
2025  MegaLights 进 UE 5.6
2026  读者框架文档把这一整条线组织成"采样→时空复用→重建→失效"四阶段
```

## Relationships

### Based On

- [[Talbot — Importance Resampling for Global Illumination (2005)]] — RIS 三步 + 校正因子，本文的直接数学基座（一手引用 "building on Talbot et al.'s [2005] RIS"）
- [[Reservoir Sampling]]（概念）— Chao 1982 流式加权选样 + Efraimidis-Spirakis 2006 加权版
- Multiple Importance Sampling（Veach-Guibas 1995）— 无偏化的权重来源

### Extends

- [[Resampled Importance Sampling]]（概念）— 从"单像素候选池"扩展到"跨像素/跨帧共享候选池"

### Related

- [[Schied — Spatiotemporal Variance-Guided Filtering (2017)]] — 正交互补（采样端 vs 重建端；图 2 合体效果最佳）
- [[Keller — Instant Radiosity (1997)]] / [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]] / [[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)]] — 复用信息的前辈：VPL/像素光/辐照度缓存（复用的是"光照数据"，ReSTIR 复用的是"采样决策"）
- [[2026-10-08-Tabula Rasa — Monte Carlo estimation of unit-variance noise with controlled spatio-temporal correlation|Tabula Rasa]] — 同为"噪声作为接口"的 MC 时代样本
- [[Modern Real-Time Lighting Framework]] — 读者框架文档：本文是其四阶段中"采样 + 时空复用"两阶段的核心引用

### Followed By

- ReSTIR GI (2021) / GRIS (2022) / RTXDI / ReGIR / MegaLights (UE 5.6) — 整个 ReSTIR 家族

## Personal Knowledge State

- **user_level: Normal（结论层）**；推导层（§4.2 的边缘化积分与 |Z(y)|/M 定理）可后续按 Hard 桥接——前置是 [[Resampled Importance Sampling]] 的方差劈分（已 Normal）+ 基础概率（全概率边缘化）。
- 与既有知识咬合：RIS/Reservoir 两概念昨日刚入库，本文是两者"合体后第一篇应用"——对读位已按计划兑现。

## Learning Value

- **Normal → Easy 的收口材料**：读完应能回答——① 蓄水池为什么可合并（状态充分性）；② 为什么朴素跨像素复用会变暗（|Z|/M < 1）；③ clamp 20× 在防什么（陈旧权重）；④ 有偏/无偏的取舍发生在哪条射线上。
- **预算方法论第三个可直接搬走的条款**：时域 M 截断 = **"任何复用都必须带失效条件与权重上限"**（此前库内已有"等时间准则 M=TY/TX"与"误差定预算"两条）。

## Visualization

![[Research/Files/Pictures/Bitterli 2020 — 图1 千级至百万级动态灯三方法等时对比.png]]
**读图**：两组场景、三种方法等时间对比。左：beeple 的 Zero Day，11,000 个动态发光三角；右：Amusement Park，340 万个。每张图从左到右 = Moreau 2019 light BVH / ReSTIR 无偏版 / ReSTIR 有偏版——等时间下 light BVH 误差肉眼可见，ReSTIR 有偏版最干净。零日场景 15 ms、游乐园 50 ms（1080p，每像素 ≤8 条阴影射线）。出处：图 1，Bitterli et al. 2020。

![[Research/Files/Pictures/Bitterli 2020 — 图3 RIS时空复用积木抽象图.png]]
**读图**：全文机制的一张抽象图。左：Talbot 式 RIS——随机候选流进一个"选择器"，输出选中样本；中：把 RIS 看成积木块，相邻像素各自的候选流并联进"空间 RIS"；右：再串联"时域 RIS"（上一帧的样本回收），有效候选数 M 随空间/时间滤波核大小增长。出处：图 3，Bitterli et al. 2020。

![[Research/Files/Pictures/Bitterli 2020 — 图2 低样本输入质量决定去噪上限.png]]
**读图**：去噪器不能凭空造特征。同一场景（340 万动态灯）等时间下：上排无降噪、下排接 OptiX 降噪器；列 = light BVH / 无偏 ReSTIR / 有偏 ReSTIR。结论：**喂给降噪器的样本质量直接决定重建上限**——采样端（本文）与重建端（SVGF）是互补而非竞争。出处：图 2，Bitterli et al. 2020。

![[Research/Files/Pictures/Bitterli 2020 — 图5 空间复用迭代质量.png]]
**读图**：空间复用的迭代放大效应。M=32 候选起步，每轮收集 k=5 个邻居蓄水池；从左到右迭代 1/2/4 次——成本几乎不增，质量急剧上升（每像素有效候选 ≈ kⁿM）。复用收益最终会饱和（邻域候选耗尽）。出处：图 5，Bitterli et al. 2020。

![[Research/Files/Pictures/Bitterli 2020 — 图7 多源PDF偏差与MIS修正1D示例.png]]
**读图**：1D 玩具实验，一眼看懂偏差来源与三种修法。目标分布 p̂=2−2x，两个候选源：p₁=常数、p₂=阶跃函数。黑粗线 = 样本真实分布的 1/f（真值），彩色细线 = RIS 权重的估计。(a) 传统 RIS 权重：p₂ 为零的右半区**系统性低估**（有偏=变暗）；(b) 按 1/|Z| 修正：无偏了；(c) p₂ 近零（10⁻⁴ 而非 0）时 (b) 的权重**噪声爆炸**；(d) MIS 平衡启发式：**既无偏又稳**——这就是 Alg. 6 选它的原因。出处：图 7，Bitterli et al. 2020。

![[Research/Files/Pictures/Bitterli 2020 — 图8 四场景等时质量对比表.png]]
**读图**：四场景（Bistro day/Burger Restaurant/Subway/Zero Day）等时间三方对比：Talbot 流式 RIS / ReSTIR 无偏 / ReSTIR 有偏（附收敛参考）。RMAE 数字直接给出复用的收益：Subway 场景 Talbot 基线 1.16 → 无偏 0.45 → 有偏 0.25（时间几乎相同）——**时空复用把等误差成本压低 1–2 个数量级**。出处：图 8，Bitterli et al. 2020。

## Notes

- 实现框架：Falcor（开源）；supplemental 里有 Talbot 式多源 RIS 与本文 MIS 版的详细对比。
- 与读者框架的逐项对读：四阶段中①采样 = 本文候选生成 + RIS；②时空复用 = Alg. 4/5（含 M clamp 与邻居拒绝）；③重建 = 交给降噪器（图 2）；④失效 = visibility reuse 淘汰 + M clamp + 邻居拒绝启发式（本文未做 disocclusion 的完整方案，产品化后由 RTXDI confidence/MegaLights 探索补齐）。
- 命名彩蛋：ReSTIR = **Re**servoir-based **S**patio**T**emporal **I**mportance **R**esampling。
- 环境图候选占 25% 的混合策略、alias table 加速——工程细节与 [[Reservoir Sampling]] 概念页的"RTXDI 只读"计划互链。
