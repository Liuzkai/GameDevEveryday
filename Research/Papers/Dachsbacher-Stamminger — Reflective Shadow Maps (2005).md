---
type: paper
title: Reflective Shadow Maps
authors:
  - Carsten Dachsbacher
  - Marc Stamminger
year: 2005
published: 2005-04-03（I3D '05；正文页脚 203–208）
venue: I3D 2005 — Symposium on Interactive 3D Graphics and Games（ACM SIGGRAPH 主办；Washington DC）
url: https://doi.org/10.1145/1053427.1053460
code: ""
project_page: ""
category:
  - real-time-rendering
  - global-illumination
  - indirect-lighting
  - many-lights
  - shadow
  - classical
importance: A（经典）
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted（思想层：'从结构化缓存 gather 固定样本数'的实时 GI 范式；被后续 many-lights / VPL 管线与 2026 神经 GI 直接引用）
user_level: Normal
status: unread
aliases:
  - RSM
  - Reflective Shadow Map
  - 反射阴影贴图
  - pixel light
  - 像素光源
  - Dachsbacher 2005
tags:
  - gi
  - indirect-lighting
  - real-time
  - many-lights
  - rsm
  - real-time-rendering
---

# Reflective Shadow Maps（Dachsbacher & Stamminger 2005）

## TL;DR

**把阴影贴图的每个像素当成一盏"小灯"（pixel light）——单光源下，整个第一个 bounce 的间接光全部来自光源可见的那些表面，所以 shadow map 本身就装着间接光的全部信息。** RSM = 标准 shadow map 的四缓冲扩展（深度 / 世界坐标 / 法线 / 反射通量 Φ），间接光 = 对这些 pixel light 求和；因为几十万像素求和太贵，缩到 **约 400 个重要性采样**（按与受光点的屏幕空间距离降密），再用**屏幕空间插值**从低分辨率（32×32/64×64）把间接光铺满全屏。2005 年的硬件（D3D9 / PS2.0–3.0）上做到 **4.2–27.5 fps**（512×512、112–448 样本）。

> **一句话定位**：[[Keller — Instant Radiosity (1997)]] 的"每盏虚拟灯一遍渲染"被它压成"**一趟 gather 着色器 + 固定样本数**"——**"光照复杂度预算"在实时 GI 里的第一个具体形态**。库内 [[Global Illumination]] 谱系表第 6 行（缓存族）的**第一个具名节点的落库**。

## Problem

- 全 GI 与交互性天然不相容：光追需要集群，辐射度对动态场景近乎不可用（原文开篇即立此背景）；
- 但作者抓住了关键观察：*"for many purposes, global illumination solutions do not need to be precise, but only plausible"*（**很多场景下间接光不需要精确、只需要可信**）→ 只求**一个 bounce** 的近似；
- 直接的问题形式：**单点光源 + 漫反射场景下，怎么用一次阴影贴图渲染的时间尺度，把第一个 bounce 的间接光画出来？**

## Previous Work（原文定位）

| 路线 | 状态 | 本文差异 |
|---|---|---|
| 阴影贴图 / 阴影体（1977–1987） | 交互标准的直接光 | 本文在其上扩一个 bounce |
| **Instant Radiosity**（Keller 1997） | 动态可用，但**每盏虚拟灯一遍渲染，"many rendering passes are required"** | 本文把多趟压成**一趟 gather + 固定样本预算** |
| 预计算 GI（辐射度 / PRT / 光场） | 交互但静态 | 本文不改几何、适合动态场景 |
| 交互光追（Wald 等 2001–2003） | 需集群 | 本文单卡 |
| 影视近似 GI（Tabellion & Lamorlette 2004） | 证明"一个 bounce 在多数场合够用"；直接光存纹理图集再 gather | 本文**不需要**额外图集——光源视野的 shadow map 就是全部信息 |
| Translucent Shadow Maps（**同作者 2003**） | shadow map 像素当次表面光源（透射） | 本文同一思路的**反射版** |

## Core Idea（三个组成部分）

### 1. 数据：四缓冲的 shadow map

每个像素存：深度 $d_p$、**世界坐标 $x_p$**（省像素着色指令，不现场反推）、**法线 $n_p$**、**反射通量 $\Phi_p$**。每个像素 = 一盏"pixel light"：

- 方向性辐射强度：$I_p(\omega) = \Phi_p \max\{0, \langle n_p | \omega \rangle\}$；
- 受光点 $x$ 处的辐照度：$E_p(x,n) = \Phi_p \frac{\max\{0,\langle n_p | x - x_p\rangle\}\max\{0,\langle n | x_p - x\rangle\}}{||x - x_p||^4}$（**注意 1/r⁴**：通量形式 = 1/r² 距离项 × 2 个余弦项）；
- 为什么存**通量**而非 radiosity/radiance：*"we don't have to care about the representative area of the light"*——不用给每个像素光源定义一个"面积"，生成与求值都简单；
- flux buffer 本身"像一张未着色的图像"（通量 × 反射系数），生成成本极低；墙角奇异处沿负法线偏移光源位置即可大幅缓解。

### 2. 求值：重要性采样 + 固定样本预算

- 对每个着色点 $x$：投影回 shadow map → 在 (s,t) 周围按**密度随距离平方下降**采 pixel light（极坐标：半径 $\xi_1 r_{max}$、权重 $\xi_1^2$ 补偿）；
- **全场固定约 400 个样本**（原文："we have to reduce the sum to a restricted number of light sources, e.g. 400"）——预计算一份采样图案全局复用（时间相干 → 减少闪动；空间相干 → 样本不足时产生 banding，可用 Poisson 采样改良）。

### 3. 屏幕空间插值（加速的关键）

- 先按低分辨率（32×32 或 64×64）完整求值间接光；
- 全分辨率 pass 中逐像素检查**周围 4 个低分辨率样本是否可用于插值**（判据：法线相似 + 世界位置接近）；3–4 个可用 → 双线性插值；否则落入"最后一遍完整求值"；
- 用**屏幕四边形网格 + occlusion query** 跳过已完整重构的区域（原文承认这会带来 pipeline stall——Table 1 里 112→224 样本的掉帧有它一份）。

### ⚠️ 最重的近似：不处理间接光的遮挡

> *"Note that we do not consider occlusion for the indirect light sources… This is a severe approximation, and can lead to very wrong results."*

- 原文的态度：多数情况下"间接光效果可见"就够了；**自阴影用 AO 贴图补偿**（Fig 6）；
- 含义：RSM 是"**光照存在性**"的近似，不是"光照正确性"的近似——**与 IR 的对比点**（IR 每盏 VPL 走完整硬件阴影）。

## Key Data（原文 Table 1 / 正文数字）

| 项目 | 数据 |
|---|---|
| RSM 分辨率 | 512×512（与相机分辨率同） |
| 样本数 | 112 / 224 / 448（"400 samples were sufficient in our example scenes"） |
| 低分辨率图 | 32×32 或 64×64 |
| 实测帧率（GeForce Quadro FX4000，P4 2.4GHz） | table 场景 **24.2 fps**（112 样本）→ 14.3（224）；lucy **19.3** → 11.4；droids **15.1** → 7.9 → 4.2（112/224/448） |
| 实现 | Direct3D 9 + HLSL；PS2.0 可跑（分多趟、shader 常量传样本），PS3.0 单循环 + 查表纹理更快 |
| 架构 | 直接光 + 间接光走 **deferred shading**（解耦场景复杂度、避免隐藏表面跑复杂 shader） |

原文自陈的性能边界：*"The application of deferred shading buffers does not completely decouple scene complexity from rendering time."*——自适应细化的密度取决于深度/法线变化，即仍与场景复杂度挂钩。

## Limitations（原文自述）

- **无间接遮挡**（见上，最重的一条）；
- 假设**全漫反射**；扩展到非漫反射需要 material ID buffer + 现场求 BRDF → 样本数需显著升高（原文认为对交互 GPU 太贵）；
- 单光源（spot / parallel）；全向光需 cube map + 改造采样图案（Future Work）；
- 纹理表面需用**滤波版**纹理渲染 flux，否则少量样本下闪烁；
- 面积光源的 penumbra 不被 flux buffer 捕获（但与整体近似相比影响小）。

## Historical Context & Technology Evolution

```text
1978  Williams：shadow map（"光源视角"这条账本的源头）———— [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]
1986  Kajiya：渲染方程（"一个 bounce"的完整表述）———— [[Kajiya — The Rendering Equation (1986)]]
1990  Haeberli & Akeley：accumulation buffer（IR 的"累加"硬件） [记名]
1991  Heidmann：Real Shadows – Real Time（IR 用的硬件阴影法） [记名]
1997  Keller：Instant Radiosity——VPL 起源（光路顶点 = 虚拟点光源；每盏灯一遍渲染）
        ↓  [本文原文："With Instant Radiosity, dynamic objects and lights are possible,
             but many rendering passes are required."]
2003  Dachsbacher & Stamminger：Translucent Shadow Maps（shadow map 像素当光源，透射版）
2004  Tabellion & Lamorlette：影视近似 GI——"一个 bounce 够用"的实践依据
2005 ★ 本文：Reflective Shadow Maps——反射版 + 固定样本 gather + 屏幕空间插值
2006  Dachsbacher & Stamminger：Splatting Indirect Illumination（RSM 的 VPL 泼溅续作） [记名]
2008  Ritschel et al.：Imperfect Shadow Maps（粗糙点云代理为大量 VPL 生成阴影） [记名]
2010s 实时 GI 进入 Lumen / DDGI / VXGI 时代（缓存族与探针族合流）
2021  Neural Radiance Caching：缓存族接上神经（RTX Remix 产品化）
2026  AMD attention GI：**把 RSM 当输入**（"加一路 RSM 渲染让轻量模型看到屏幕外几何"）——缓存族成为神经 GI 的输入格式
2026  MegaLights：每像素固定光照采样预算——"固定样本预算"原则的当代形态（见下）
```

## Relationships

### Based On

- [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] —— "光源视角渲染一遍"这条账本的地基；RSM 只是在这张账本上多记了三个通道；
- **Translucent Shadow Maps**（同作者 2003，记名）——"shadow map 像素当光源"的透射版原型，本文是反射版的直接前作；
- **Tabellion & Lamorlette 2004**（记名，Sony Pictures Imageworks）——"一个 bounce 够用"的影视生产实践；本文省掉了它的纹理图集。

### Contrasts（与 IR 的对照——两篇合读才完整）

| | [[Keller — Instant Radiosity (1997)]] | **本文（RSM 2005）** |
|---|---|---|
| 光源集来源 | 光路顶点（**quasi-random walk，随机采样**） | 光源视野像素（**一次光栅化，结构化全集**） |
| 光源集规模/形态 | M 盏 VPL（几百） | 40 万+ pixel light（**按需求采样 ~400**） |
| 遮挡处理 | **有**（每盏 VPL 一遍硬件阴影） | **无**（最重的近似；AO 补偿） |
| 成本单位 | 每盏灯一遍渲染（pass 数 ∝ 灯数） | **每像素固定样本数**（与灯数脱钩） |
| 输出 | "几秒"静帧 → 滑动窗口可实时 | 交互帧率（4.2–27.5 fps @2005 硬件） |

### Followed By

- **Splatting Indirect Illumination**（Dachsbacher & Stamminger, I3D 2006，记名）——给 RSM 的间接光做 VPL 泼溅；
- **Imperfect Shadow Maps**（Ritschel et al. 2008，记名）——用点云代理给海量 VPL 做阴影；
- [[Lightweight Attention-based Indirect Illumination (AMD)]]（2026）——**神经 GI 把 RSM 直接当输入通道**（"加一路 RSM 渲染，让轻量模型看到屏幕外几何"）——**VPL/RSM 这条线的"神经续命"**；
- [[2026-09-14-Gaussian Light Transport]] —— 同题域的显式基函数路线（对照样本）。

### Related

- [[Global Illumination]] —— 谱系表第 6 行（缓存族）的具名节点之一 ✅ **2026-10-03 落库**；
- [[Neural Global Illumination]] —— 桥的前置材料（本笔记 + [[Keller — Instant Radiosity (1997)]] 合读 = "缓存族的两种记账"）；
- **"成本关于哪个变量线性"**（库内贯穿母题）：RSM 的三段对照——1978 每盏灯 +1× 场景 → 1997 每盏 VPL 一遍 → 2005 每像素 ~400 样本。**成本单位从"光源数"改写成"像素×固定样本"，这是 MegaLights（"每像素固定光照采样预算"）在 2026 年的同款把戏的 2005 年原型。**

## Why It Works（本库读法）

1. **信息视角的转换**：不问"怎么算间接光"，问"**间接光的信息已经存在于哪个数据结构里**"——单光源下答案惊人地小：就是光源的那张 shadow map。**这是一次"先找信息在哪、再算"的胜利**（与"体检三问"的"不存？"同族，只是这里是"不用存，因为已经在阴影贴图里"）；
2. **固定样本预算**：400 样本与场景中"表面数"无关，只与屏幕像素有关——**成本函数被主动重写**（与 Clark 1976"成本 ∝ 可见复杂度"同向）；
3. **两级近似分工明确**：几何可见性由 shadow map（精确）负责、间接光的低频变化由屏幕空间插值（近似）负责——**高频/低频各归其位**。

## Game Development Relevance

- **与动态灯光预算的直接接口**：动态灯光维度（≤3/≤2/≤1/0）的账本是"每盏灯 ≈ +1× 场景渲染"（1978）；RSM 展示的是**另一条腿**——"把光照复杂度换成每像素固定预算"。MegaLights 的"每像素固定采样"正是这条腿在 2026 的工程形态 → **你的五档矩阵在 PC 档可以同时持有两套语言**（灯数上限 / 每像素预算）；
- **Niagara 粒子光**（UE 5.8 MegaLights 已支持）本质是"大量小光源"——RSM 的 pixel light 与 VPL 正是"大量小光源"的经典处理范式，理解它们有助于判断"粒子光给多少预算"；
- E-Day 实测（10-1/10-2 入库）的"数千（DF）/ ≤100（NVIDIA）"两口径，其实都在说同一件事：**这条路线的问题从来不是"灯多不多"，而是"每像素花多少"**。

## Unreal Engine Relevance

- 无内置 RSM 实现（AMD 2026 论文亦注明"RSM 在 UE 中并非常规产物，需要额外渲染 pass"）；
- 但结构上：**"缓存 + 屏幕空间插值"**这个组合是 Lumen 屏幕探针（screen probe gather）哲学的最小样本——若要给移动档做"一个 bounce 的廉价 GI"，RSM 是被验证过的最省形态之一（对 2026 硬件而言成本极低）；
- 与 [[Shadow Mapping]] 的成本模型合读：RSM 的生成成本 = 一次额外的光源视角渲染（多三个 RT）——**正是 1978 账本里"每盏灯 +1×"的同一笔开销，换到了 400 个间接光源**。

## Personal Knowledge State

- `user_level: Normal（结论层）`——读法：**"四缓冲 + 400 样本 + 屏幕插值 + 无遮挡"四件事**，即可拿走；
- 前置：[[Shadow Mapping]]（Easy，账本）→ [[Global Illumination]]（八代谱系表）→ 本篇；
- 与 [[Keller — Instant Radiosity (1997)]] 合读：**两篇 = "缓存族"从随机到结构化的完整起手式**，[[Neural Global Illumination]] 桥的具名前置由此闭合。

## Learning Value

**四条可迁移抽象（不需要读全文）**：

1. **"先问信息已经在谁手里"**：单光源的一个 bounce 信息 = shadow map 本身——**有些"计算"其实是"读取"**（与"重放换存储"（PRB/IR）同族第四例）；
2. **"固定样本预算"的第一次实时表述**：把成本从"光源数 × 场景"重写为"像素 × 常数"——**"预算"这个词在 GI 里的 2005 年形态**（与 Clark 1976、MegaLights 2026 三连）；
3. **"低频交给插值、高频留给精确"**：屏幕空间插值的判据（法线相似 + 位置接近）就是"什么时候可以偷懒"的可操作判据；
4. **近似的诚实记账**：无遮挡是"严重近似"，作者直说"可以非常错"——**明确近似边界比假装精确更有工程价值**（与本库"边界写进笔记"的规范同构）。

## Visualization

![[间接光缓存族_RSM 2005 与 Instant Radiosity 1997 双源图解.html]]

## Notes

- **来源核对**：正文 PDF（UCSC 课程镜像 `users.soe.ucsc.edu/~pang/160/s13/.../p203-dachsbacher.pdf`，pypdf 提取 7 页全文逐节核对；数值均取自原文 Table 1 与正文）。
- **页码核验**：正文页脚 **203–208**（第 7 页为图版，页脚 231）；ACM 注册记录（CrossRef / OpenAlex 一致）作 **203–231**——两说并存，本笔记引用以正文 **203–208** 为准。
- **历史彩蛋**：[[Keller — Instant Radiosity (1997)]] 的致谢里感谢 *"Marc Stamminger, providing access to the Reality Engine 2"*——**8 年后，Stamminger 正是本文（RSM）的共同作者**。硬件提供者与后续作者是同一人，这条线自己把自己接上了。
- 入库日：2026-10-03（Run 25）；[[Global Illumination]] 谱系表第 6 行"RSM"自此从"记名待入库"改为 ✅。
