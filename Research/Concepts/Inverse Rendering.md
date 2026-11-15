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
