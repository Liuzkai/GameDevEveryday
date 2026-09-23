---
type: paper
title: "A Hitchhiker's Guide to Multiple Scattering: Exact Analytic, Monte Carlo and Approximate Solutions in Transport Theory"
authors: [Eugene d'Eon]
year: 2022
published: 2022-11-25
venue: "免费电子书 / 参考手册（self-published，eugenedeon.com；v0.3.2，2022-11；首版 2016-07-24；746 页；三种记号版本：图形学 σt / 光学 μt / 中子输运 Σt）"
url: "http://eugenedeon.com/hitchhikers"
code: "https://github.com/eugenedeon/hitchhikersscatter"
project_page: ""
category: [rendering, transport-theory, multiple-scattering, reference]
importance: S
historical_importance: 5
game_relevance: 4
production_readiness: Research
user_level: Normal
status: unread
---

# A Hitchhiker's Guide to Multiple Scattering（2022 v0.3.2）

## TL;DR

**它不是教程，是地图册。** 746 页免费参考手册：把**线性输运理论中"已经解出的模型问题"集中在一处**——点源 Green 函数、半空间 / 平板（slab）反照率问题、searchlight 问题……配上 **2000+ 条文献指引**、**可跑的 Monte Carlo 参考代码 + Mathematica 推导交叉验证**（GitHub）。作者 Eugene d'Eon（NVIDIA）。

> **本库定位**：**它是库内两条线（微面多次散射 / 参与介质）脚下的"参考层"。** 过去两周入库的每一篇——[[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)|Walter]] → [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)|Heitz 2016]] → [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)|Kulla-Conty]] → [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)|Fdez-Agüera]] → [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media|Dupuy]] → [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media|GPIS]]——处理的都是这本书里**已编号、已给出基准值的"模型问题"**的特例或新解。**从今天起：遇到能量 / 输运问题，先查它，再决定要不要读论文。**

## What It Is（三句话用法）

1. **查"这个问题解过没有"**：按「几何（无限介质 / 半空间 / slab / 球 / 任意形状）× 散射类型（各向同性 / 各向异性 / Rayleigh）× 源（点 / 平面 / searchlight）」编排的 **100+ 章**，几乎穷举经典模型问题，附"Further Reading"逐条文献；
2. **查"精确基准值"**：大量问题附 benchmark 数值 + MC 代码 → **可以拿来校验你自己的实现**（对 PBR 收口清单里那条 furnace test 尤其直接——见下）；
3. **查"近似谱系"**：同一问题往往并列 **exact / singly-scattered / doubly-scattered / 各类扩散近似 / Van de Hulst / Pomraning / "our first~fourth approximation"** —— **这就是你一直在收集的"更准 vs 更便宜"的全谱对照表**。

## Chapter Map（16 部分；★ = 与你现有库内线直接相关）

| # | 部分 | 页 | 内容 | 相关性 |
|---|---|---|---|---|
| I–II | Foundations / Scattering | 9–64 | 线性 Boltzmann 方程；相位函数（3D / Flatland / rod） | 基础层 |
| III | Boundaries | 65–100 | 电介质 / 导体 / **★13 Rough Boundaries（微面理论 + NDF 动物园 + Smith Λ 函数表 + 球面 albedo）** / **★14 Layered Materials** | ★★★ |
| IV–VI | 1D Rod / 2D Flatland / 3D 球对称 | 101–224 | 降维玩具模型 + 点源 Green 函数族（含 **GRT 非经典**） | 方法层 |
| VII | **3D 平面对称** | 225–308 | **★48 半空间反照率问题（本章 = 库内"五条补法"的原文全谱）** / 49-50 Fresnel 边界 / **★56-58 slab / 多层 slab 反照率** | ★★★ |
| VIII | Searchlight | 309–324 | 准直光束入射族 | 应用层 |
| IX | 3D 一般形状 | 325–334 | 任意形状 / 异质介质 / 球形行星 | — |
| X | **Monte Carlo** | 335–372 | MC 指南 / **方差缩减 / zero-variance 理论 / track-length / 双向方法** | ★★ |
| XI | 积分方程 | 373–388 | **Fredholm 第二类（含 Wiener-Hopf）**；Chandrasekhar 伪问题与 H 函数 | 推导层（Hard） |
| XII–XIII | 位移核 / 自由程分布 | 389–460 | 各类非指数自由程分布（Bessel K / Lévy / Mittag-Leffler…） | 推导层 |
| XIV–XV | 随机过程 / **非经典输运** | 461–550 | 点过程 / 更新过程 / 随机飞行；**非 Beer 介质中的透射与多次散射** | ★（前瞻） |

## Verified Content（已下载 PDF 逐页核对）

**① 13.3.4 随机表面的散射截面（= Heitz 2016 的"体积化"视角的规范出处）**

$$\sigma(u) = (1+\Lambda(u))\,u, \qquad \sigma(-u) = \Lambda(u)\,u \quad (0<u\le 1)$$

原文要点：高度场散射的截面**没有反向对称性**（$\sigma(\omega)\neq\sigma(-\omega)$）——朝表面走（$u>0$）撞上微面的概率大于背离走；且 **$\sigma(-1)=0$**（垂直向上永不碰撞）。**"粗糙表面的多次散射 = 三维半空间里一次带非标准截面的经典随机游走"**——这正是库内 [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)|Heitz 2016]] 那套构造的教科书式表述。

**② 13.4 形状不变 NDF 与表 13.1（"NDF 动物园"一张表）**

Beckmann / **GGX（Trowbridge-Reitz）** / K0 / Bessel K / E1 / Erfc / Exponential / Gen-Exp / Double GGX……每个给出 slope 函数 $f(x)$ + 均匀 **shadow 函数 $\Lambda(x)$** + 散射截面——**评估任何新 NDF 的最小信息集**。

**③ 13.8 GGX 的等价刻画（Eq. 13.30）**

> GGX/TR 微表面 ≡ **随机不相关 Beckmann 补丁的混合，补丁粗糙度服从 Fréchet 分布**（$D_{TR}(u,\alpha)=\int_0^\infty \ldots D_B(u,\alpha_B)\,d\alpha_B$）。

即：**"长尾"不是 GGX 自己造的，是粗糙度异质性的混合结果。** 同节还给出 GGX 的 Smith Λ（13.33）与散射截面 $\sigma_{TR}(u,\alpha)=\frac{1}{2}\left(\sqrt{\alpha^2-\alpha^2u^2+u^2}+u\right)$（13.34）。

**④ 13.7.1 / 13.8.1 球面 albedo 的闭式近似（对你的 furnace test 直接有用）**

原文定义："**the probability that a single photon arriving from a uniform white sky reflects**"——粗糙电介质 Beckmann / GGX 界面的球面 albedo 给成 **$\eta$ 与 $\alpha$ 的拟合闭式**（平滑界面值 $S=\bar{R}_{FR}(\eta)$ 为基准）。**注意它是近似（拟合），不是精确**——但意味着"**球面 albedo**"这个量有两条工程化路线：**拟合闭式（本书）vs 查表（Kulla-Conty 的 $E_{avg}$）**。

**⑤ 14.2 分层材料——两处与本库归属注对账**

- 粗糙电介质 + Lambert 底的 BRDF 精确形式（14.5）原文注：*"It is **equivalent to the BRDF model of Kelemen and Szirmay-Kalos (2001)**, although it does not seem to be known that this corresponds to this exact layering."* → **库内"Kulla-Conty 公式本体来自 Kelemen 2001"的归属注，在此获得手册级佐证**；
- 同段原文：*"the albedos $a_{spec}(\theta)$ are **not known in closed form** and require **approximate fitting or tabulation** (Kulla and Conty, 2017; Hirvonen et al., 2019)."* → **"方向 albedo 不闭式、只能拟合或查表"是手册盖章的边界**——库内"精确、便宜、有参数，三样不可兼得"的判据在此有了权威出处。

## 为什么它值得现在入库（与库内线的逐条对账）

| 库内工作 | 在本书的坐标 |
|---|---|
| [[Multiple Scattering and Energy Compensation]] 的"五条补法谱系" | **VII.48 一章的内部结构**（exact / singly / doubly / diffusion / 各类改进近似，逐条并列） |
| [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] | **13.3.4 的 $\sigma(u)$ 与 13.8 的 GGX 截面** |
| [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] | **13.8.1 球面 albedo（查表路线）vs 本书拟合闭式** |
| [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media\|Dupuy 2026]] | 本书"半无限微片介质"族问题的**新闭式**（该族 = 13.3.4 起的一整条） |
| [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media\|GPIS]] | 本书 13.1 开篇"microfacet theory 是几何光学近似"命题的**现代随机几何重表述** |
| [[Participating Media]] | IV–IX 全部 + **XV 非经典输运**（非指数自由程 → 毛发 / 植被 / 云这类相关介质） |

## Personal Relevance（可以立刻用的三条）

1. **furnace test 有了对照物（30 分钟实测题升级）**：手册给出**球面 albedo 的闭式近似 + 大量 benchmark 值**——你的两项检查（① 粗糙端是否变暗 ② 光滑白电介质掠射边缘是否偏亮）现在**可以对着手册的数值看**，不再是"凭感觉"。**这是 9-20 以来那条唯一未闭环实测的对照基线。**
2. **"先解出来的人是谁"的溯源工具**：库内已经历两次归属更正（Kelemen 2001 / Crow 1977）——这类问题**历史长、重名多**；手册的 2000+ 条 Further Reading 是最好的溯源索引。
3. **第 XV 部分（非经典输运）是前瞻方向**：非指数自由程 / 随机介质的输运——**与毛发（细长散射体）、植被、云这类"非经典"介质相关**，是 [[Hair Rendering]] 与 [[Participating Media]] 的共同延伸。

## Limitations / Caveats

- **不是教程**（作者明说 "no attempt to provide theory and derivations"，只给解与文献指引）；
- **三种记号版本**（图形学 / 光学 / 中子输运）——**下载选 Graphics notation（$\sigma_t$）那版**；
- **体积大**（746 页 / 29 MB）→ 按章查，不通读；
- 2022 版托管在 Google Drive；**2016 版有 eugenedeon.com 直链**（hitchhikers_v0.1.3.pdf，18 MB）可用作备份；
- 配套代码：`eugenedeon/hitchhikersscatter`（C++ MC + Mathematica notebooks，每个 worksheet 另存 PDF）——**"解与 MC 互相验证"本身是本书的方法论卖点**。

## Technology Evolution（这本书在谱系上的位置）

```text
1940s-60s  输运理论成熟（中子输运 / 天体物理辐射传输）
              ↓  Chandrasekhar 1960（H 函数）；Case & Zweifel 1967（奇异本征函数）
1960s-80s  基准解汇编传统（Ganapol 1993/2008 等）
              ↓
2016       ★ d'Eon《Hitchhiker's Guide》首版（把三个学科的解收进一套记号）
              ↓  图形学侧同路：Jakob 2010（microflake）→ Heitz 2016（Smith 输运）
              ↓
2021-22    v0.3.x：新增随机介质结果、Draine / Cornette-Shanks 相位函数采样、勘误
              ↓
2026       库内一整条线（Walter→Heitz→Kulla-Conty→Fdez-Agüera→Dupuy→GPIS）
            = 这本书所收"模型问题"的现代工程化与再推导
```

## Relationships

### Based On

- Chandrasekhar / Case / Ganapol / van de Hulst 等经典（书内逐章引用）——**本库不重复建笔记**。

### Combines

- **辐射传输 + 中子输运 + 计算机图形学**：三个学科各自解过同一批问题（点源 Green 函数、反照率问题），本书把它们收进**一套记号**（且出版三种记号版本）。

### Related

- [[Multiple Scattering and Energy Compensation]]、[[Participating Media]]、[[Rendering Equation]]（**表面形式的渲染方程 ↔ 体积形式的线性 Boltzmann 方程**，是同一枚硬币）、[[Microfacet Theory]]、[[Hair Rendering]]。

## Personal Knowledge State

`user_level: Normal` —— **结论层 / 检索层 = Normal；推导层（Fredholm / Wiener-Hopf / H 函数）= Hard，不必现在动**（沿用库内"分层读法"模板：Hard 材料先问"它的结论层是不是 Normal 的"）。

**一句话检验**：能说出"**我库里的能量补偿问题在这本手册的哪一章、属于哪一族模型问题**"即算到位。

## Learning Value

- **结构性价值 > 阅读价值**：它让库内散落的 7 篇左右笔记第一次有了共同的**坐标原点**（"你的线在这本 746 页里的哪一页"）；
- **与 Kulla-Conty 的"两本补能量手册"定位区分**：Kulla-Conty = **工程化文章**（4KB 表 + 证明 + 引擎落地）；本书 = **参考手册**（问题普查 + 基准值 + 溯源）。

## Next Step（可选，均为低成本）

1. **把 13.3.4 / 13.8.1 / VII.48 三节写入 [[Multiple Scattering and Energy Compensation]] 的检索入口**（下次做 furnace test 时先查 13.7.1/13.8.1 的球面 albedo 值）；
2. 备一份 Graphics notation PDF 到本地阅读器（按章查用）；
3. （观察项）第 XV 部分"非经典输运"何时进入图形学工程实践（毛发的相关介质建模是最近的候选）。

## Visualization

![[Hitchhikers Guide 地图_十六部分与你的PBR线图解.html]]

## Notes

- 引用格式：E. d'Eon, *A Hitchhiker's Guide to Multiple Scattering*, v0.3.2（2022-11），eugenedeon.com/hitchhikers；
- 本书被库内多篇笔记反复点名（"未入库，优先级高"）、被 Dupuy 2026 与 Kulla-Conty 原文引用——**今日正式结案**；
- ⚠️ 版本注意：数字与公式核对自 **2022 v0.3.2 Graphics notation**（746 页）；行文引用时标版本号，避免与 2016 版混淆。
