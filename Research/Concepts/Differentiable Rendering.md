---
type: concept
title: "Differentiable Rendering"
user_level: Hard
tags: [rendering, optimization, foundation]
---

# Differentiable Rendering

## Definition

让**渲染结果对场景参数可求导**的渲染方法。即：给定一张目标图像，能反推出"场景参数该往哪个方向改，图像才更像目标"。

```
传统渲染:   场景参数 ──正演──> 图像
可微渲染:   图像误差 ──反传──> 场景参数梯度
```

## Core Principle

把渲染管线里每一个不可导的离散操作（光栅化的覆盖判定、深度测试、可见性跳变）用一个**可导的近似**替换掉，从而让梯度能从像素一路传回几何、材质、光照乃至相机。

常见近似手段：
- 边缘软化（软化 coverage，用 sigmoid 代替硬判定）
- 随机估计（stochastic rasterization，用采样积分代替确定性判定）
- 松弛（把离散决策放松成连续权重）

## Prerequisites

```
Vector Calculus（链式法则、雅可比）
      ↓
Optimization（梯度下降、学习率、loss 设计）
      ↓
Automatic Differentiation（计算图、前向/反向模式）
      ↓
Rasterization Pipeline（顶点→裁剪→光栅化→着色，你需要知道它在哪一步断了梯度）
      ↓
Differentiable Rasterization
      ↓
[[Differentiable Rendering]]
```

注意这条链里，**前四项是数学与基础图形学，不是前沿研究**。它们是可以快速补的。

## Evolution

```
Soft Rasterizer
      ↓
Differentiable Rasterizer（如 Pytorch3D / nvdiffrast）
      ↓
Inverse Rendering（从图像反推几何/材质/光照）
      ↓
应用分支：资产生成 / 外观编辑 / 管线优化（[[LightOpt — Lights Optimization for Real-Time Rendering]]）
```

## Related Concepts

- [[Inverse Rendering]]
- [[Neural Rendering]]
- [[Optimization]]

## Game Applications

- **离线/editor-time 优化**：灯光布局优化（LightOpt）、材质参数拟合、LOD 生成
- 资产生成：从照片重建可渲染资产
- **不是**运行时技术——这一点必须分清

## Important Papers

- [[LightOpt — Lights Optimization for Real-Time Rendering]]
- [[Inverse Rendering for Modeling with Line Primitives]]

## Personal Knowledge

Current Level: **Hard**

（初始化默认推断，待用户校正）

## Learning Gap

1. 自动微分的反向模式（为什么反向比前向快）
2. 光栅化在哪一步不可导、为什么
3. 离散量（比如"灯要不要留"）如何变成连续可优化量

## Next Learning Step

先做**最小桥**，不要一上来就读可微光栅化论文：

1. 理解"渲染是函数，参数是自变量"这一句话
2. 找一个可微渲染的 toy 例子（比如用可微方式拟合一个三角形的颜色/位置）跑通
3. 再读 [[LightOpt — Lights Optimization for Real-Time Rendering]] 的问题定义

完整路线见 [[Learning Path — Differentiable Rendering]]。
