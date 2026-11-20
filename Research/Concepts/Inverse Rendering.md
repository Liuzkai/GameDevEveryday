---
type: concept
title: "Inverse Rendering"
user_level: Hard
tags: [rendering, optimization]
---

# Inverse Rendering

## Definition

渲染的逆问题：从观测到的图像反推场景的**几何、材质、光照**。

```
正向：几何 + 材质 + 光照 → 图像
逆向：图像 → 几何 + 材质 + 光照
```

## Core Principle

逆向问题是病态的（ill-posed）：同一张图可以由无穷多种几何/材质/光照组合产生。必须靠**先验 + 正则 + 多视角约束**来约束解空间。

实现方式上，现代逆渲染基本都建立在 [[Differentiable Rendering]] 之上：定义前向渲染、定义外观 loss、反传梯度到场景参数。

## Prerequisites

- [[Differentiable Rendering]]
- [[Real-Time Rendering]]
- 优化理论

## Evolution

```
Shape from Shading
        ↓
多视角立体视觉 / SfM
        ↓
可微渲染驱动的逆渲染（材质+光照联合估计）
        ↓
神经逆渲染（NeRF / 3DGS + 材质分解）
        ↓
可进管线的显式基元重建（[[Inverse Rendering for Modeling with Line Primitives]]）
```

**另一条支线：反解的对象不是"表面"，是"灯"（2026-11-15 补）**

```
1993  Schoeneman《Painting with Light》 —— 用户画目标，约束最小二乘解**灯强度**（离线、线性）
2016  Gkaravelis                        —— 解**灯数**（候选网格 + NNLS + 聚类 + 最优 cut）
2024  Lipp                              —— 解**位置/强度/方向**（伴随光线追踪，目标搬到曲面辐射场）
2026  LightOpt                          —— 解**全部 + 个数**（可微、实时游戏、ToD、~10 min/场景）
```

**两条支线的差别只有一句话**：**几何/材质反解问"这东西是什么"，灯反解问"我该怎么照它"。** 前者的约束来自物理与多视角一致性，后者的约束来自**美术意图**——所以后者的目标函数往往是人画的（1993/2016）或人选的（LightOpt 选"复现原灯架的辐照度"）。

> 这条支线的工程价值比几何反解**近得多**：一个项目可以不做摄影测量，但不能不布光。

**第四条支线：未知量不是表面、也不是灯，是"中间那层东西"（2026-11-20 补）**

雾、水、烟、组织——**介质挡在相机与场景之间时，它也成了一个待解的量**。于是逆渲染的未知量从三类（几何 / 材质 / 光照）变成四类，而第四类有一个别的类别都没有的性质：

> **它最容易把前三个吃掉。**

一个足够灵活的介质模型可以完美拟合每一张观测，同时把场景纹理吸进自己身上——[[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting|NIRJALA 2026]] 的原话是 *"scene and water are not identifiable from the images alone"*。这不是"先验不够强"，是**结构性的**：

- **加法项与乘法项的混合不可分**。观测是 $I=J\,T^D+B+J\,T^S$；直射与前向**都乘在 $J$ 上**，所以图像只约束它们的**和**，不约束分配。
- **"分离"只能靠行为差异**：一个随距离衰减、一个随距离饱和。这是**函数形状**的信息，不是数据量的信息。
- **所以更多的图不解决问题**。这与几何反解"多拍几张就好"的直觉相反。

**两层处理，顺序不能反：**

1. **先让反演不发散**（数值层）。去介质 = 除以透射率，增益 $e^{+\beta r}$ 随距离指数增长；加一条随距离饱和的光路可给它一个与距离无关的下界 $1/q\le1/\kappa$。而**同一个 $q$ 也是前向梯度**（$\partial I/\partial J=q$）——**"病态"与"远处学不动"是同一个量**。
2. **再钉住可辨识性**（结构层）。用一个**表达力不足**的旁支模型双向约束灵活模型（正向：它能否从辐亮度解释观测；反向：用反解能否落回同一个辐亮度）。**关键在"太小"**——每通道只有标量的模型表达不了场景结构，所以藏在介质里的场景会变成残差。

> **但它只是锚，不是解**：NIRJALA 明写——*"PRATIBIMBA 不使分解唯一，它只缩小优化器能落进去的那组分解。"* 而且锚的参数**不跨站点**（引他人实测：介质网络换站点掉约 9 dB）。

**判据（可直接用于评审任何"去雾 / 去水 / 去烟"方案）**：① $1/q$ 的最大值是多少，会不会随距离发散？② 各分量能不能被数据分开，还是只能靠先验？③ 那个用来钉住的模型，是不是真的"表达力不足"，还是只是"另一种灵活"？——**三个都答不上来，它在薄雾上会好看，在浓介质上会崩。**

（介质侧的正向与工程细节见 [[Participating Media]] 的"第七条路"，域级展开见 [[Underwater Rendering]]。）

## Related Concepts

- [[Differentiable Rendering]]
- [[Gaussian Splatting]]
- [[Neural Rendering]]

## Game Applications

- 从实拍生成可渲染资产
- 材质扫描

## Important Papers

**表面侧**

- [[Inverse Rendering for Modeling with Line Primitives]]

**灯侧（2026-11-15 补，四篇一条线）**

- [[LightOpt — Lights Optimization for Real-Time Rendering|LightOpt 2026]] — 实时游戏场景 + 可微 + 增删并 + ToD
- [[Gkaravelis 2016 — Inverse Lighting Design using a Coverage Optimization Strategy|Gkaravelis 2016]] — 解灯数（NNLS + 聚类 + cut）
- [[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization|Lipp 2024]] — 伴随法 + 视角无关目标
- **Schoeneman et al. 1993《Painting with Light》** — 这条线的源头（⚠️ **本库尚未入库**：原文 PDF 三个来源均被拦——Yale graphics 返回 JS 挑战页、Cornell 目录 404、CiteSeerX 429。**只核到摘要与参考文献，未核正文与图**）

## Personal Knowledge

Current Level: **Hard**

## Next Step

属于 Watchlist 方向。除非你开始负责"从实拍生成资产"类工作，否则优先级低于 [[Differentiable Rendering]] 本身。
