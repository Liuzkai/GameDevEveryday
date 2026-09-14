---
type: paper
title: Compact Neural Appearance Models for Efficient Gaussian Splatting
authors:
  - Florian Hahlbohm
  - Jorge Condor
  - Linus Franke
  - Martin Eisemann
  - Marcus Magnor
year: 2026
published: 2026-09-04 (arXiv)
venue: arXiv（TU Braunschweig / CGI 团队）
url: https://arxiv.org/abs/2609.05255
code: ""
project_page: 有（见 arXiv 页）
category:
  - gaussian-splatting
  - neural-rendering
  - compression
  - real-time
importance: B+
game_relevance: Medium-High（资产内存/带宽预算角度）
production_readiness: Prototype（含 WebGL 查看器，笔记本与移动端 GPU 可跑）
user_level: Normal
status:
  - read
---

# Compact Neural Appearance Models for Efficient Gaussian Splatting

## TL;DR

3DGS 里每个高斯的**视角相关外观**不再存 3 阶球谐（SH）系数（192 bytes/高斯），改存一个小 latent code（28 bytes/高斯）+ 共享微型 MLP 解码。**单高斯外观内存 192B → 28B（-85%），优化速度快 1.3×，重建质量反而更好。** 同一可微 CUDA 光栅器内公平对比了 SH、球面外观模型与神经外观三条路线。

## Problem

3DGS 的工程瓶颈早就不是"能不能跑"，而是**内存与带宽**：百万级高斯 × 每个高斯的外观系数，决定了显存占用、流式传输成本和移动端可行性。SH 是质量基准但最贵；怎么在保持视角相关效果的前提下把这部分压下来？

## Core Idea

把"每个高斯各自存一份完整外观函数"换成"每个高斯存一个小 code，外观函数全体共享一个 MLP"。本质是把逐点存储换成了**隐式表示**——和你熟悉的纹理图集 vs 过程贴图的取舍同构。

## Key Contribution

1. 统一框架下的三路线公平对比（同一光栅器、同一管线，不是各跑各的 benchmark）
2. 神经外观：192B → 28B/高斯，1.3× 训练加速，质量持平或更好
3. 附赠可移植 WebGL viewer，笔记本和移动 GPU 实测——比较有了工程意义而不只是纸面
4. **一个生产警告**：表达力更强的外观模型会开始"吸收"场景中的非静态内容（把动态物体烘进静态外观），影响几何恢复——做资产管线时这是坑

## Game Development Relevance

对你的价值在**预算语言**：3DGS 进游戏的前提是它能被翻译成"显存 MB / 带宽 GB/s / 解码 ALU 成本"这些你会写的预算项。这篇论文恰好给了第一组可信数字：外观存储降 85%，代价是每个高斯多一次微型 MLP 解码（ALU 换带宽——移动端 TBDR 上通常是划算交易）。

## Unreal Engine Relevance

若未来 GS 作为 UE 资产格式（Niagara/静态网格之外的第三类表示），外观存储格式直接决定 streaming 成本。本文是"GS 资产化"方向的基础数据点。

## Limitations

- 解码 MLP 的逐高斯调用在大规模场景下的实际渲染开销需实测（论文重点在训练与存储）
- 动态场景不适用（静态外观假设）
- 非静态内容被吸收的问题没有根治方案，只有诊断

## Related Concepts

- [[Gaussian Splatting]]
- [[Neural Rendering]]
- [[Tile-Based Rendering]]（移动端带宽约束的背景）

## Related Papers

- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]] — GS 工程化的另一刀（光栅化排序）
- [[GradRig — Differentiable Weights for Skinned Gaussian Splat Deformation]] — GS 角色形变

## Personal Knowledge State

Current Level: **Normal**

Reason: 不需要理解 MLP 训练，核心取舍（存储 vs 解码算力、逐点存储 vs 共享隐式函数）是经典的工程 trade-off，正好在 [[Gaussian Splatting]] 从 Normal 推向 Easy 所需的那几块拼图里（排序问题已由 [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization|TileGS]] 覆盖，本文覆盖外观存储问题）。

## Learning Path

读摘要 + 对比表 + "expressive models absorb non-static content" 这一节（生产坑）。配合 [[Gaussian Splatting]] 的 Mastery Criteria 自查：看完这篇，"GS 的内存/带宽瓶颈在哪、有哪些解法"这一条应该能打勾了。

## Notes
