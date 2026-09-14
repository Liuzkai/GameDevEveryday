---
type: paper
title: "Inverse Rendering for Modeling with Line Primitives"
authors: ["Kenji Tojo", "Ariel Shamir", "Nobuyuki Umetani", "Bernd Bickel"]
year: 2026
published: "2026-09-01"
venue: "SIGGRAPH Asia 2026"
url: "https://arxiv.org/abs/2609.00625"
code: ""
project_page: "this https URL (arXiv abs page links)"
category: ["Rendering", "Inverse Rendering", "Hair and Fur", "Geometry"]
importance: "A"
game_relevance: "High"
production_readiness: "Research"
user_level: "Hard"
status: unread
tags: [hair, fur, inverse-rendering, differentiable-rasterization, line-primitives, vfx]
---

# Inverse Rendering for Modeling with Line Primitives

## TL;DR

毛发、皮毛、纤维、织物这类**模糊、各向异性**结构，主流做法是用 3D Gaussian 这类半透明体积基元重建——但体积基元不兼容标准深度测试光栅化、反射建模和物理模拟。本文改用**显式线段**重建：亚像素网格光栅化线段做抗锯齿以重现半透明外观，并提出一个**随机可微分线段光栅化器**来优化顶点位置、属性和离散连接关系。结果是完全显式几何，可直接接进标准图形管线。

## Problem

- 毛发/纤维类结构难捕难画难实时渲染
- 体积基元（3DGS）能捕到模糊边界，但：不兼容 depth-tested rasterization / 反射建模 / 物理模拟 → 进不了游戏管线
- 直接优化海量线段去匹配目标图，梯度难求（尤其离散连接关系）

## Core Idea

用**显式线段**作为基元，配合一个能对顶点位置/属性/**离散连接性**都产生有效梯度的随机可微光栅化器。

## Technical Approach

- 线段在亚像素网格上光栅化 + 抗锯齿 → 半透明外观
- Stochastic differentiable rasterizer for line segments
- 对顶点位置、属性、离散 connectivity 都能出梯度
- 输出完全显式几何 → cross-platform rendering、任意 shading model、物理模拟

## Key Contribution

在"捕得住模糊外观"和"进得了标准管线"之间取到了显式几何这一侧，同时质量对标体积表示。

## Game Development Relevance

对 VFX 的意义：

- **毛发/丝线/纤维类特效的资产来源**：如果能从多视角图直接重建出可用的线段几何，就能绕开手工 groom
- **能进标准管线**是关键分野：这意味着它理论上可以走 UE 的 standard rasterization，而不是一个只能离屏渲染的玩具
- 但当前是**重建**问题（从照片生成资产），不是**实时模拟**问题。对你的实时性能预算暂无直接影响

## Unreal Engine Relevance

- 若产出为显式线段/管状几何，可映射到 Niagara Ribbon / Groom / 或自定义 mesh
- 更现实的路径：作为 DCC 侧资产生成工具，产出 groom 或 ribbon 数据再入引擎

## Limitations

- 逆渲染优化通常慢（分钟~小时级），纯 offline
- 离散连接性优化的稳定性与收敛性未验证于大规模场景
- 与 UE Groom / Niagara Ribbon 的 LOD 与移动端降级路径完全没覆盖

## Related Concepts

- [[Differentiable Rendering]]
- [[Inverse Rendering]]
- [[Hair and Fur Rendering]]

## Related Technologies

- [[Neural Asset Reconstruction]]

## Related Papers

- [[Compact Neural Appearance Models for Efficient Gaussian Splatting]]

## Personal Knowledge State

Current Level: **Hard**

Reason:
需要可微光栅化、随机优化、逆渲染目标函数三块前置。你目前未标记这三块的水平，按默认推断为未掌握。

## Learning Path

见 [[Learning Path — Differentiable Rendering]]。
最短桥：[[Differentiable Rendering]] → 可微光栅化 → 本文。

## Notes

2026-09-07 首次收录。今日列入是因为它守住了"显式几何 vs 体积基元"这条对 VFX 极其重要的分界线。进入 Watchlist 而非主动学习队列——Hard 且桥较长。
