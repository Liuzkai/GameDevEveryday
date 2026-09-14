---
type: paper
title: "Gaussian Light Transport"
authors: [Patrick Attimont, Kartic Subr, Cyril Soler]
year: 2026
published: "2026-09-10"
venue: "SIGGRAPH Asia 2026 (Conference Track)"
url: "https://arxiv.org/abs/2609.11430"
code: ""
project_page: "https://patrick-attimont.com/projects/gaussian-light-transport/"
category: [rendering, global-illumination]
importance: A
historical_importance: 0
game_relevance: 4
production_readiness: Research
user_level: Hard
status: unread
---

# Gaussian Light Transport

## TL;DR

把光传输方程的解直接表示为 **13 维高斯混合模型**（位置 3 + 方向 2 + 法线 2 + 材质属性 6），不走 Monte Carlo / Neumann 级数，而是**直接最小化渲染方程的残差**来拟合参数。结果：视角无关的 GI 解，渲染毫秒级，显存需求只有神经渲染方法的一个零头。**非神经路线对 Neural Radiance Caching 的正面挑战。**

## Problem

实时 GI 的两条主流路线都有结构性代价：

- Monte Carlo 路径追踪 + 降噪：要 RT 硬件、要去噪器、视角相关（每帧重算）；
- 神经 GI（NRC 等）：权重 + 世界空间结构的显存开销大，训练/查询都要 Tensor Core。

**一个被冷落的老思路**：把光传输的解当作一个函数直接拟合（基函数回归），而不是采样估计。难点一直在高维——位置 × 方向 × 法线 × 材质，朴素表示会爆炸。

## Core Idea

把 GI 解写成高维高斯的线性组合：

$$L(x, \omega, n, m) \approx \sum_i w_i \cdot G_i(x, \omega, n, m)$$

两个关键决策：

1. **把场景属性（法线、材质）塞进高斯的定义域**（13D）。反直觉但有效：高斯自动"粘"在相似材质/朝向的区域上，所需基函数数量**大幅减少**，求值反而更快；
2. **不采样，直接优化**：以渲染方程的残差为目标函数直接拟合参数（类似 Galerkin / 最小二乘思路），绕开 Neumann 级数逐 bounce 展开。

配套工程：高效 culling 策略（高维高斯求值时剔除贡献可忽略的项），让优化与渲染都可行。

## Why It Works（与神经路线的对照）

| | 神经 GI（NRC 系） | Gaussian Light Transport |
|---|---|---|
| 表示 | 网络权重（隐式） | 高斯参数（显式基函数） |
| 求解 | 训练数据拟合 | 渲染方程残差直接优化 |
| 视角 | 随帧查询 | **视角无关**（解本身存在） |
| 显存 | 权重 + 结构 | "一个零头"（原文声称） |
| 硬件 | Tensor Core | 通用计算 |

这正是 [[Gaussian Splatting]] 哲学在 GI 域的翻版：**显式基函数混合 + 可微/残差优化**，对抗神经网络的隐式黑箱。你已掌握 GS，这篇的概念门槛因此很低。

## Limitations

- 静态/准静态场景假设（视角无关 = 换几何/材质要重优化）；
- 13D 高斯混合的容量上限：高频细节（锐利焦散、复杂 glossy 交互）需要多少基函数，论文规模内未充分回答；
- 优化过程本身不是实时的（渲染实时、拟合离线）——定位更接近"新一代烘焙"而非 Lumen 替代；
- 与动态灯光（[[MegaLights]] 类场景）不兼容：灯光变了要重新拟合。

## Game Development Relevance

- **烘焙路线的现代化候选**：视角无关 + 毫秒渲染 + 低显存，命中开放世界静态大场景的 GI 预算痛点；
- 移动端三档（你的 Android_High/Mid/Low）天然只烘焙——若该路线成熟，移动端的 GI 质量上限会被抬高；
- 对照样本：本周库内神经 GI（AMD attention GI 45ms 未优化）vs 本文非神经 GI（毫秒级）——"神经"不是 GI 问题的唯一解，显式基函数路线正在回归。

## Unreal Engine Relevance

- 映射位置：Lightmass / GPULightmass 的潜在替代求解器；
- 与 Lumen 互补而非竞争（Lumen 管动态，本路线管静态高质量）；
- 短期无直接集成路径（Research 阶段）。

## Technology Evolution

GI 求解范式谱系上的新分支：

```text
辐射度法（基函数/有限元的鼻祖，1984）
        ↓
烘焙 Lightmap / Irradiance Volume
        ↓
实时 PT + 降噪（视角相关回归）
        ↓
神经 GI / NRC（隐式表示）
        ↓
★ Gaussian Light Transport（显式基函数 + 残差优化，2026）
```

讽刺的是它比神经路线更接近 1984 年辐射度法的精神——绕了一圈回来，但这次的基函数是高维、自适应、可优化的。

## Relationships

### Based On

- 渲染方程 / 辐射度法（Galerkin 传统）
- 高斯混合表示（与 [[Gaussian Splatting]] 同族不同域）

### Contrasts

- [[Lightweight Attention-based Indirect Illumination (AMD)]]（神经 GI，本方向的对照阵营）
- Neural Radiance Caching（隐式表示的对照）

### Related

- [[Real-Time Global Illumination]]
- [[Neural Global Illumination]]（平行路线）

## Personal Knowledge State

`Hard` 标签是因为数学侧（高维基函数回归 + 渲染方程残差优化）；但概念侧你已具备全部前置（GS 的基函数直觉 + GI 的工程直觉）。**读法建议：只读表示选择与 culling 策略，跳过 13D 高斯的推导细节。**

## Notes

- Inria / 爱丁堡系作者（Soler、Subr 均为 GI 采样理论老将），SIGGRAPH Asia 2026 正式 track，可信度高；
- Watchlist：项目页代码/数据放出后值得复看显存数字。
