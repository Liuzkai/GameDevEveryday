---
type: concept
title: "Neural Global Illumination"
user_level: Hard
tags: [gi, neural, real-time]
---

# Neural Global Illumination

## Definition

用神经网络估计全局光照中的某个分量（通常是间接光或多 bounce 辐射度），替代传统路径追踪中昂贵/嘈杂的采样。

## Core Principle

神经 GI 方法分两大阵营：

| 阵营 | 优点 | 缺点 |
|---|---|---|
| **Screen-space** | 快，天然适配管线 | 视锥外的光进不来 |
| **Scene-wide / data-gathering** | 能补回屏幕外光 | 需要"收集数据"，塞不进标准管线 |

2026 年的主流研究方向是**弥合这两者**——见 [[Lightweight Attention-based Indirect Illumination (AMD)]]，其策略是"把预算从模型复杂度挪到输入丰富度"。

## Prerequisites

- [[Global Illumination]]
- [[Reflective Shadow Maps]] / 虚拟点光源（VPL）
- [[Neural Rendering]]
- 实时渲染管线结构（G-buffer、deferred）

## Evolution

```
烘焙光照贴图（静态）
        ↓
Screen-space GI / SSGI
        ↓
Light Propagation Volumes / VXGI
        ↓
实时路径追踪 + 去噪（RTX GI）
        ↓
Neural Radiance Caching（NRC，2021，已在 RTX Remix 中产品化）
        ↓
神经 GI（2026：AMD attention GI、ArtiFixer 的单 pass GI 预测）
```

值得注意：**NRC 已经产品化**（RTX Remix、《半条命 2 RTX》、《传送门 RTX》），说明神经 GI 不是纯未来技术。1080p 下典型开销约 2.6ms。

## Related Concepts

- [[Neural Rendering]]
- [[Global Illumination]]
- [[Reflective Shadow Maps]]

## Game Applications

- Lumen 的替代/增强
- 老游戏重制（RTX Remix 路线）

## Important Papers

- [[Lightweight Attention-based Indirect Illumination (AMD)]]
- [[DLSS 5 — Generative Neural Rendering]]
- [[2026-09-14-Gaussian Light Transport]]（⚠️ 非神经对照样本：显式 13D 高斯基函数 + 残差优化，视角无关、毫秒渲染、低显存——证明"神经"不是实时 GI 的唯一解；2026-09-14 入库）

## Personal Knowledge

Current Level: **Hard**

## Learning Gap

对你而言性价比最高的不是"神经"部分，而是**经典部分**：RSM / VPL / iVPL 这套近似 GI 的思路。它同时是：
- 理解 Lumen 的前置
- 理解所有神经 GI 论文的前置
- Normal 难度，几个小时可补

## Next Learning Step

先补 [[Reflective Shadow Maps]] 与 iVPL，再回来看神经方法。顺序反了会浪费大量时间。
