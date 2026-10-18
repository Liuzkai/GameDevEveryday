---
type: paper
title: "Optimally Combining Sampling Techniques for Monte Carlo Rendering"
authors: [Eric Veach, Leonidas J. Guibas]
year: 1995
published: 1995-08-06
venue: "SIGGRAPH '95 Proceedings, pp. 419-428"
url: "https://doi.org/10.1145/218380.218498"
code: ""
project_page: "https://graphics.stanford.edu/papers/combine/"
category: [rendering, sampling, monte-carlo, theory, classic]
importance: A
historical_importance: 5
game_relevance: 4
production_readiness: Industry Adopted
user_level: Normal
status: unread
aliases: [MIS, Multiple Importance Sampling, MIS 1995]
---

# Veach — Optimally Combining Sampling Techniques for Monte Carlo Rendering (1995)

## TL;DR

**Multiple Importance Sampling（MIS）的出生证明**，采样线"三件旧工具"（1982 WRS / 2005 RIS / **1995 MIS**）的最后一件。核心问题：渲染的被积函数是多个无关因子的乘积（BRDF × Le × 几何项），任何单一重要性采样分布在"未覆盖因子"主导的区域都会方差爆炸——Veach 把"如何组合多个采样技术的样本"形式化为一个**反问题**：在全体无偏加权组合（权重函数 Σwᵢ=1）中找方差最小者。答案惊人的简单：**balance heuristic**（ŵᵢ = cᵢpᵢ/Σcⱼpⱼ）与任意替代方案的最优者的方差差距被一个可忽略的小量（"variance gap"）上界；**power heuristic**（β=2）在"某一技术接近完美匹配"的常见情形下更好，成为 30 年来的事实标准。两个常被忽视的次级结果：**用近似 PDF 算权重仍无偏**（§5.4——30 年后成为 ReSTIR DI O(M) MIS 与 RTXDI 的理论许可）与**权重选择与样本数分配的不对称**（§5.2——坏权重可任意放大方差，坏样本配比最多差 n 倍）。

## Problem

- 渲染积分的被积函数几乎总是**病态的**：不连续、奇点、在极小区域有大值。
- 被积函数是**无关因子的乘积**（式 2：f_r · G · L_i）——每个重要性采样分布只能匹配其中一个子集：面积采样匹配 Le 但无视 BRDF；BRDF 采样匹配 f_r 但无视 Le。**当被忽视的因子占主导时（小而亮的光源 / 近镜面），该技术方差爆炸**（图 2a/2b 的互补失败区）。
- 当时的标准做法是**显式分区**（partitioning）：把积分域切开、每区配一个技术——需要手工确定切换边界（Shirley-Wang 1992 甚至为特定 Phong BRDF 推导切换条件），且分区边界处产生硬阈值噪声。
- 论文问的反问题：**给定若干技术，如何加权样本使组合无偏且方差低**——"如何划分域"是它的特例。

## Historical Context

```text
1984  Cook 分布式光追（多效应积分，单技术+分区思想萌芽）
1986  Kajiya 渲染方程（MC 求解的统一对象）
1990  Kirk-Arvo 无偏采样技术（分区中的偏差陷阱）
1994  Veach-Guibas / Lafortune 双向估计器（BDPT 的前夜）
1995  ★本文 MIS（balance/power 启发式 + 定理 1/2；BDPT 作为应用同场发布）
1997  Veach 博士论文（BDPT + MLT 完整体系；MIS 成为其骨架）
2004  pbrt 初版把 MIS 做成基础设施（离线渲染默认配置）
2005  Talbot RIS（在重采样框架内使用 MIS 组合候选池——O(M²) 伏笔）
2020  ReSTIR DI（Bitterli：多源 PDF 的 O(M) MIS 无偏化；§5.4 近似许可兑现）
2021  ReSTIR GI / RTXDI SDK（MIS 权重进实时管线）
2022  GRIS（M=置信权重替代逐对 MIS——统一理论对照面）
2025  MegaLights 进 UE 5.6（家族产品化高峰）
```

## Previous Work

- **显式分区**（Cook 1984 / 分布式光追传统）：域切分 + 每区一技术——本文 §3.2 指出它只是权重函数的退化特例（除一个 wᵢ 外全为 0）；
- **Kirk-Arvo 1991**：自适应技术在分区下的偏差陷阱——MIS 用固定 cᵢ + 无偏权重绕开；
- **Shirley-Wang 1992**：为特定 BRDF/光源组合推导"何时切换技术"——只适用特定模型、只能二选一；MIS 对一般 BRDF、任意多技术统一处理且免切换；
- **Lafortune-Willems 1993/94 双向路径追踪**：提供了一族高质量采样分布（§4.3 的 k+2 种 p_{k,m}）——MIS 首个旗舰应用的素材库。

## Core Idea

**Multi-sample model（式 11）**：从 pᵢ 采 nᵢ 个样本，估计量 F̂ = Σᵢ (1/nᵢ) Σⱼ wᵢ(Xᵢⱼ)·f(Xᵢⱼ)/pᵢ(Xᵢⱼ)。约束 **Σᵢwᵢ(x)=1（对所有 x）**即无偏——权重可随位置变化，只要逐点和为 1。整个"如何组合"的问题变成在权重函数空间里最小化方差。

**balance heuristic（式 12）**：ŵᵢ(x) = cᵢpᵢ(x)/Σⱼcⱼpⱼ(x)——它让样本值 f·ŵᵢ/pᵢ **与样本来自哪个分布无关**（"各技术的样本账目拉平"）。

**定理 1（provably good）**：任何权重组合的方差 ≤ balance heuristic 的方差 + (1/min nᵢ − 1/N)·F²。这个"variance gap"相对图 2 那种坏分布造成的方差可忽略，且随样本数增大趋于 0——**balance heuristic 不用知道 f 就接近最优**。

**power heuristic（式 14）**：wᵢ = pᵢ^β/Σpⱼ^β。**β=1 退化为 balance，β=∞ 退化为 max**（硬分区）——一个参数统一了连续谱。当某技术接近完美匹配 f 时（常见情形），把大权重推向 1、小权重推向 0 能进一步压方差；**β=2 实践全局最佳**（§5.1），成为 30 年来的默认选择。附录 A 给出最坏界：β=2 时 c = 1.25(1+√n)。

## Technical Approach

**两个模型，两个定理**：

| 模型 | 采样方式 | 结论 |
|---|---|---|
| multi-sample（式 11） | 每技术固定 nᵢ 个独立样本 | 定理 1：balance heuristic 近似最优（gap → 0） |
| one-sample（式 16） | 每样本按 cᵢ 随机选一个技术采一个 | **定理 2：balance heuristic 严格最优**（无 gap） |

- **one-sample 模型就是路径追踪本身**：每个弹射点随机选 diffuse/specular/transmitted 分布——pbrt、UE 材质采样走的都是这条路，定理 2 保证这种"随机选路"在权重选择意义下无懈可击。
- **BDPT 应用（§4.3，图 8/9）**：长度 k 的路径有 k+2 种双向生成技术（光子子路径 m 个顶点 × 眼子路径），每条路径的 PDF 可显式计算 → power heuristic 组合全部技术。图 9(b)（25 spp）在焦散与镜面上碾压同时间标准 PT（图 9(c)，56 spp）。
- **权重计算成本 <5%**（§4.1）——低开销是它被全行业采纳的工程前提。
- **近似权重仍无偏（§5.4）**：只要近似 PDF p'ⱼ **一致使用且与样本来源 i 无关**，权重就可以用近似值算——pᵢ(Xᵢ) 本身仍须精确。这是"教科书不会强调但工程离不开"的条款：**ReSTIR DI 把 O(M²) 的逐对 PDF 求值降为 O(M)，靠的正是这个许可**；RTXDI 的近似 MIS 权重同理。
- **样本数分配是次要问题（§5.2）**：V[F̂] ≤ n·V[F] + (n−1)/N·F²——**均分 cᵢ 相比最优分配最多差 n 倍**，而**坏的权重选择可任意放大方差**。不对称性指明了优化优先级：权重要选对，配比均分即可。

## Key Contribution

1. **把"组合多个采样技术"形式化为无偏估计器家族上的优化问题**——第一次给"多技术渲染"一个数学骨架，而不是配方集；
2. **balance/power 启发式 + 两条定理**——不用知道 f 就 provably good；power(β=2) 成为 30 年事实标准；
3. **分区是特例**：显式分区、Shirley-Wang 切换条件都被收编进权重函数谱系；
4. **BDPT 的诞生**：双向技术族 + MIS 组合 = 1997 博士论文的核心章节（与 MLT 并列）；
5. **近似权重条款（§5.4）**——为 25 年后的实时复用管线预埋了理论接口。

## Why It Works

- **样本值拉平机制**：balance heuristic 让"某技术的样本贡献多少"只由样本位置决定、与来源无关——任何技术都补不进别人的洞，但也没有谁会在别人的洞里爆炸；方差被"组合分布 p = Σcᵢpᵢ 与 f 的失配程度"控制，而不是被最差技术控制。
- **鲁棒性哲学**（原文 §1 的宣言）："意义不在于把几个坏技术拼成好技术，而在于把几个潜在的好技术组合起来**保留各自的优势**"——robust estimator = 对广类被积函数都低方差。
- 定理 1 的证明技术（Lagrange 乘子逐点最小化 Σwᵢ²/nᵢ·f²，附录 A）本身就说明了最优解的形状：逐点二次最优 = balance。

## Limitations

- 需要**能对任意样本 x 求值所有 pⱼ(x)**——某些技术（如多候选再选一的直接光策略）PDF 依赖样本外的随机数；实践中退到 §5.4 的近似许可，方差可能上升；
- balance/power 只在"至少一个技术匹配得好"时显著优于彼此；**所有技术都失配时（图 6），各启发式差距很小**——MIS 不能凭空创造好分布；
- cᵢ 固定于采样前：自适应分配样本数会引入偏差（需两阶段采样，Kirk-Arvo 1991）；
- multi-sample 模型要求各技术样本独立存储——对内存受限的实时管线，这正是 ReSTIR 用蓄水池替代全量候选池的动机。

## Game Development Relevance

- **实时管线中的 MIS 无处不在**：Lumen 最终收集、随机直接光、环境光遮蔽的多种射线策略组合、RTXDI 的 DI/GI/ReGIR 多源合并——全部站在 MIS（或其 GRIS 后继）上；
- **§5.2 的不对称性是预算论证的样板**："给可调项均分预算最多损失 n 倍，给机制选错损失无上界"——与 SABC 预算工作的"机制 > 参数"直觉同构；
- **§5.4 是"复用必须自带失效条件"的前置条款**：近似权重免费，但必须一致使用——ReSTIR DI 的 O(M) MIS、RTXDI 的近似 confidence 权重都受它保护；
- one-sample 定理 2 解释了为什么"每弹射随机选 lobe"的路径追踪不需要额外修正。

## Unreal Engine Relevance

- **UE5 Lumen**：最终收集与屏幕/世界空间探针的样本组合是 MIS 思想的工程化；MegaLights（UE 5.6+）的 stochastic light sampling 与 ReSTIR 权重链路直接继承本文谱系；
- **Niagara/材质**：单技术失配的教训适用于粒子光照与高光采样——"小而亮"（emissive 精灵、技能闪光）正是图 2a/2b 的互补失败区；
- 实操映射：Lightmass 离线阶段（pbrt 式 MIS 全家桶）→ 运行时近似（split-sum 把"组合"预计算成查找表——预处理家族的另一例）。

## Technology Evolution

```text
1995 MIS（组合已有技术）
  ↓ 成为基础设施（pbrt 2004 / 离线渲染默认）
2005 Talbot RIS 在重采样内用 MIS（O(M²) 代价）
  ↓ 组合对象从"技术"变成"历史样本"
2020 ReSTIR DI：O(M) MIS 无偏化 + 蓄水池（§5.4 许可兑现）
  ↓ 统一理论
2022 GRIS：M=置信权重替代逐对 MIS；M-cap 成为理论必需
  ↓ 产品化
2025 MegaLights（UE 5.6）
```

## Relationships

### Based On

- [[Kajiya — The Rendering Equation (1986)]] — 被积函数与 MC 求解对象
- Kirk-Arvo 1991（无偏采样与偏差陷阱）— 反面教材
- Lafortune-Willems 双向估计器 — 旗舰应用的分布来源

### Extends

- Cook 1984 分布式光追的"多技术"实践 → 数学化
- Shirley-Wang 1992 的切换条件 → 一般化（任意 BRDF、任意多技术）

### Related

- [[Talbot — Importance Resampling for Global Illumination (2005)]] — RIS 在重采样内调用 MIS；候选池全量 O(M²) 求值的出处
- [[Resampled Importance Sampling]] / [[Reservoir Sampling]] — 采样线三件套的另外两件
- [[Schied — Spatiotemporal Variance-Guided Filtering (2017)]] — 重建端与采样端的互补

### Followed By

- [[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)]] — O(M) MIS 无偏化；§5.4 近似许可的首次实时兑现
- [[Ouyang — ReSTIR GI Path Resampling for Real-Time Path Tracing (2021)]] — MIS 权重 + Jacobian 进实时 GI
- [[Lin — Generalized Resampled Importance Sampling Foundations of ReSTIR (2022)]] — 用置信权重 M 替代逐对 MIS；统一理论
- Veach 1997 博士论文 — BDPT/MLT 体系（MIS 为骨架）

## Personal Knowledge State

## Learning Value

- **采样线三件旧工具就此集齐**（WRS 1982 / RIS 2005 / MIS 1995）——读者框架"采样 → 时空复用 → 失效"的第一阶段自此有原文锚点；
- 一条主线读法：**本文 §3.3 定理 1 → Talbot §MIS 候选池（O(M²) 痛点）→ Bitterli Alg.4（O(M) 无偏化）→ GRIS M-cap（置信权重）**——"组合权重的 30 年降本史"；
- §5.4 近似许可 + §5.2 不对称性是两枚"读到才看得见"的工程接口，比定理本身更常被引用者兑现。

## Visualization

![[Research/Files/Pictures/Veach 1995 — 图2 两种采样技术各自失败与MIS组合.png]]
**读图**：四种半径/四块粗糙度的对比矩阵。(a) 面积采样在**小光源 + 高粗糙**处好、但镜面板上高光糊掉（BRDF 因子失配）；(b) BRDF 采样在**大光源 + 低粗糙**处好、但小光源打不中（Le 因子失配）；(c) power heuristic（β=2）组合两者，**全矩阵无失败区**——"robust"一词的最小样本。出处：Figure 2，Veach & Guibas 1995（SIGGRAPH '95, pp. 419-428）。

![[Research/Files/Pictures/Veach 1995 — 图5 四种启发式方差曲线对比.png]]
**读图**：σ/μ 随粗糙度的曲线（虚线 = 两种单技术，实线 = 启发式）。四种启发式全程贴着两技术的下包络；极端粗糙度处 **balance (a) 最差**、**power (c, β=2) 全程最低**、max (d) 在交点附近有尖峰——"β=2 成为默认"的量化证据。每格上方三张小图是圈选粗糙度处的 1spp 渲染，可见 cutoff/power/max 消掉了 balance 的"亮点/暗点"噪声。出处：Figure 5，Veach & Guibas 1995。

![[Research/Files/Pictures/Veach 1995 — 图6 final gather场景三技术对比.png]]
**读图**：final gather 简单场景。(a) 光源采样近处好远处噪（1/r² 因子失配）；(b) 方向采样远处好近处噪（立体角失配）；(c) power 组合全程干净；(d) σ/μ 随距离——(c) 是 (a)(b) 的下包络。**所有技术都失配时各启发式差距变小**的注脚场景。出处：Figure 6，Veach & Guibas 1995。

![[Research/Files/Pictures/Veach 1995 — 图9 BDPT组合与标准路径追踪对比.png]]
**读图**：玻璃蛋 + 落地灯 + 聚光灯场景。左 9(b)：power heuristic 组合全部双向技术，25 spp——桌面焦散、镜面反射完整；右 9(c)：同计算量标准路径追踪 56 spp——焦散几乎不可见、全域噪声。**"组合的价值"旗舰证据**：贡献几乎全部来自个别技术（caustics 只被光子子路径命中）的场景，MIS 把它们零成本纳入。出处：Figure 9，Veach & Guibas 1995。

## Notes

- 免费全文：[UCSD CSE168 课程镜像 PDF](https://cseweb.ucsd.edu/~viscomp/classes/cse168/sp26/readings/veach.pdf)；[Stanford 官方页](https://graphics.stanford.edu/papers/combine/)（含单独 JPEG 图与电子版附加图 2(x)/9(x)/9(y)——图 9(x) 的逐技术贡献金字塔比印刷版多一列一行）。
- 4 图均为 PDF 内嵌位图原样提取（PyMuPDF clip 渲染）；图 2 右侧说明段为原文排版一部分。
- power heuristic 的 β=2 建议来自实验而非理论最优——论文只给了 β=2 的最坏界 c=1.25(1+√n)。
- 与读者光照框架文档的对接点：框架"采样阶段"的原文锚 = 本文式 11/12；"复用阶段"的失效条款谱系见 ReSTIR 概念页。
