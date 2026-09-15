---
type: paper
title: "The Rendering Equation"
authors: [James T. Kajiya]
year: 1986
published: "1986-08"
venue: "SIGGRAPH 1986, Computer Graphics 20(4), pp. 143–150"
url: "https://doi.org/10.1145/15886.15902"
code: ""
project_page: ""
category: [rendering, global-illumination, classical]
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal
status: unread
---

# The Rendering Equation (Kajiya 1986)

## TL;DR

把"一个表面点朝某个方向出射多少光"写成一个**积分方程**，一举统一了此前割裂的直接光、镜面反射、漫反射互反射（radiosity）各自为政的局面；并给出通用求解策略——**用 Monte Carlo 沿光路随机采样**（即 path tracing 的思想源头）。你正在研读的 Cook-Torrance D/G/F，就是塞进这个方程里 $f_r$ 位置的一个具体实现。

## Historical Context

1986 年之前，真实感渲染是两半的：

- **Whitted 光线追踪（1980）**：完美的镜面反射/折射/阴影，但只跟踪镜面方向，**做不了漫反射互反射**（color bleeding 全无）；
- **Radiosity 辐射度法（1984，Goral 等）**：完美的漫反射互反射，视角无关，但**做不了镜面**，且依赖有限元剖分。

Kajiya 的贡献不是发明了新效果，而是**给所有光传输写了一个共同的数学容器**——两类方法都被证明是这个方程在特定假设下的特解。

## Core Idea

渲染方程（原文以光强的积分形式给出，现代教材的标准写法）：

$$L_o(x, \omega_o) = L_e(x, \omega_o) + \int_{\Omega} f_r(x, \omega_i, \omega_o)\, L_i(x, \omega_i)\, (n \cdot \omega_i)\, d\omega_i$$

逐项读：

- $L_o$：点 $x$ 朝 $\omega_o$ 方向的出射辐射亮度——**这就是屏幕上每个像素最终要算的东西**；
- $L_e$：自发光项（光源本身）；
- $f_r$：[[BRDF]]——材质把入射光 $\omega_i$ 转成出射光 $\omega_o$ 的比例函数，[[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 的 D·G·F 就是填在这里；
- $L_i$：入射辐射亮度——**注意它是未知量本身在别的地方的取值**：$L_i$ 在另一个表面点 $x'$ 上又由同一个方程定义。方程是递归的，这是全部 GI 困难性的根源；
- $n \cdot \omega_i$：Lambert 余弦项，几何投影衰减。

## Technical Approach

Kajiya 指出这是第二类 Fredholm 积分方程，可用 Neumann 级数展开——每一项对应"光弹射 n 次"的贡献：

```text
L = 自发光 + 1 次弹射 + 2 次弹射 + 3 次弹射 + ...
    （直接光）  （1 bounce GI） （2 bounce GI）
```

级数无穷、积分高维，解析解不存在 → **Monte Carlo 采样**：随机生成光路，用统计平均逼近积分。这直接发明了 path tracing，并给出了 importance sampling 的框架（按 BRDF/余弦的重要性采方向）。代价是噪声——方差随样本数按 $1/\sqrt{N}$ 收敛，这条收敛律到今天仍在定义实时渲染的采样预算。

## Why It Works / 历史影响链

```text
Whitted 递归光线追踪（1980，只镜面）
Radiosity（1984，只漫反射互反射）
        ↓ Kajiya 统一（1986）：一个方程，两种特解都被收编
Path Tracing（本文附带发明）
        ↓
Distributed Ray Tracing（Cook 1984）↔ 双向 PT（Lafortune / Veach 1993-97）
        ↓
Metropolis Light Transport（Veach & Guibas 1997）/ Photon Mapping（Jensen 1996）
        ↓
离线渲染全面 path tracing 化（Arnold / RenderMan / Hyperion，2013-2015"电影业 PT 革命"）
        ↓
RTX 实时光线追踪（2018）→ Lumen / 硬件 RT GI（今天）
```

**电影中的每一帧、你引擎里的每一次 Lumen 追踪，都是这个方程的近似求解。**

## Limitations / 边界

- 几何光学框架：不处理波动效应（衍射、干涉、偏振）——对游戏与电影几乎无影响；
- 原文的体渲染/参与介质处理是初步的，体传输方程要等后续工作补全；
- 方程是"描述"不是"解法"：它告诉你正确答案是什么，但高效求解（降噪、采样策略、缓存）是之后 40 年整个渲染界的工作。

## Game Development Relevance

- **渲染谱系的绝对锚点**：你的实时渲染知识（Lumen、probe、烘焙、降噪）全部是"这个方程在实时预算下的妥协方案"——理解了方程结构，就理解了每个引擎特性在妥协什么：
  - 烘焙 Lightmap = 把方程的解离线预计算、牺牲动态性；
  - Light Probe / SH9（[[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]]）= 把方程的漫反射部分压缩到 9 个系数；
  - Lumen = 用表面缓存 + 屏幕追踪做实时近似；
  - DLSS RR / 降噪器 = 对抗 Monte Carlo 的 $1/\sqrt{N}$ 收敛税；
- 对你当前 BRDF/PBR 研读的直接意义：**方程是容器，BRDF 是内容**——$f_r$ 决定"光往哪弹"，余弦项和递归结构决定"弹多远多亮"。

## Unreal Engine Relevance

- UE 整个渲染管线是该方程的实时近似栈：Lumen（GI 项）、MegaLights（直接光 $L_e \times$ 可见性）、Substrate（$f_r$ 的材质表示）、Path Tracer 参照模式（离线真值）；
- 读 UE 渲染源码/文档时，" solving the rendering equation under frame budget" 是理解一切取舍的元语言。

## Relationships

### Based On

- Radiometry（辐射度量学，物理量定义）
- Whitted 光线追踪（1980）
- Radiosity（Goral et al. 1984）
- 积分方程理论与 Monte Carlo 方法（核工业/中子输运的数学传统）

### Generalizes

- Whitted ray tracing（取 $f_r$ 为 δ 函数镜面方向）
- Radiosity（取 $f_r$ 为常数漫反射 + 有限元离散）

### Followed By

- Bidirectional Path Tracing（Veach & Guibas）
- Metropolis Light Transport（1997）
- Photon Mapping（Jensen 1996）
- [[2026-09-15-Grid-Free Monte Carlo for Time-Dependent Diffusion]]（同族 Monte Carlo 思想在扩散 PDE 的延伸）
- 全部现代 GI 研究（[[Global Illumination]]）

## Personal Knowledge State

`Normal`。你已掌握 BRDF/PBR 的"内容侧"，本文补"容器侧"。Mastery 检查表（建议本周内自测）：

1. 能默写方程并说清每一项的物理含义（$L_e$ / $f_r$ / $L_i$ / 余弦）；
2. 能解释 $L_i$ 的递归性为什么是 GI 的根本困难；
3. 能把 Lumen / Lightmass / SH Probe 各自映射为"方程的哪种妥协方案"；
4. 能解释 Monte Carlo 求解的 $1/\sqrt{N}$ 收敛律与实时渲染采样预算的关系；
5. 能说明 Cook-Torrance 的 D·G·F 填在方程哪个位置、改变什么行为。

## Notes

- 引用信息已核实：Computer Graphics (SIGGRAPH '86) 20(4), pp. 143–150, DOI 10.1145/15886.15902；
- Kajiya 后来在微软研究院（Hair 渲染、压平渲染方程的层级细节工作）——同一人横跨理论奠基与产品时代；
- 原文仅 8 页，是现代图形学单页影响力最高的论文之一，值得通读原文。
