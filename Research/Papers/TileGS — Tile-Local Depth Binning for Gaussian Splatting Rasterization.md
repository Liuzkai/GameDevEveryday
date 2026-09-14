---
type: paper
title: "TileGS: Tile-Local Depth Binning for Gaussian Splatting Rasterization"
authors: ["Wei Tan", "Matias Turkulainen", "Lauri Ilola", "Hamed Rezazadegan Tavakoli", "Juho Kannala"]
year: 2026
published: "2026-09-04"
venue: "arXiv preprint"
url: "https://arxiv.org/abs/2609.03613"
code: ""
project_page: ""
category: ["Rendering", "Gaussian Splatting", "Rasterization", "Performance"]
importance: "B"
game_relevance: "Medium"
production_readiness: "Research"
user_level: "Hard"
status: unread
tags: [gaussian-splatting, rasterization, tile-based, sorting, performance]
---

# TileGS: Tile-Local Depth Binning for Gaussian Splatting Rasterization

## TL;DR

3DGS 光栅化的核心开销之一是**排序**：半透明高斯必须按深度顺序混合。TileGS 提出 tile-local depth binning——在 tile 内做深度分桶，降低全局排序压力。

## Problem

3DGS 的画质强，但要进实时管线，最大的工程障碍就是它的排序与混合模型跟标准深度测试光栅化不兼容。移动端 TBDR（tile-based deferred rendering）架构下这个问题被进一步放大。

## Core Idea

把排序粒度从"全局/跨 tile"降到"tile 内分桶"，让每个 tile 独立完成混合，适配 tile-based 硬件。

## Technical Approach

Tile-local depth binning（细节需读原文）。

## Key Contribution

面向 tile-based GPU 架构（移动端的现实）优化 3DGS 光栅化。

## Game Development Relevance

- 中等。3DGS 在游戏里的定位仍不明（是场景表示？是资产格式？是过场？）
- 但**tile-based 思维**对你有直接价值：Android 三档全是 TBDR，任何"排序/混合"类开销在 TBDR 上都比在 PC 上更贵
- 与你的 OverDraw 预算项同源问题

## Unreal Engine Relevance

暂无直接对应。

## Limitations

- arXiv preprint，未经过同行评审
- 无代码、无与移动端实测数据

## Related Concepts

- [[Gaussian Splatting]]
- [[Tile-Based Rendering]]
- [[Neural Rendering]]

## Related Technologies

- [[Real-Time Gaussian Rendering]]

## Related Papers

- [[Compact Neural Appearance Models for Efficient Gaussian Splatting]]
- [[Inverse Rendering for Modeling with Line Primitives]]

## Personal Knowledge State

Current Level: **Hard**

Reason:
需要先掌握 3DGS 的光栅化与排序模型，以及 TBDR 架构细节。后者你很可能已经 Normal/Easy。

## Learning Path

桥：
1. [[Tile-Based Rendering]]（TBDR 架构，你大概率已 Normal）
2. [[Gaussian Splatting]] 基本原理
3. 本文

## Notes

2026-09-07 首次收录。Tier B。今日保留主要是因为"tile-local"思路与移动端预算相关，而不是论文本身的突破性。
