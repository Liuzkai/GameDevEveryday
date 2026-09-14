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

## Related Concepts

- [[Differentiable Rendering]]
- [[Gaussian Splatting]]
- [[Neural Rendering]]

## Game Applications

- 从实拍生成可渲染资产
- 材质扫描

## Important Papers

- [[Inverse Rendering for Modeling with Line Primitives]]
- [[LightOpt — Lights Optimization for Real-Time Rendering]]

## Personal Knowledge

Current Level: **Hard**

## Next Step

属于 Watchlist 方向。除非你开始负责"从实拍生成资产"类工作，否则优先级低于 [[Differentiable Rendering]] 本身。
