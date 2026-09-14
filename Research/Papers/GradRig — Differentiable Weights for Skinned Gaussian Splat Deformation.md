---
type: paper
title: "GradRig: Differentiable Weights for Skinned Gaussian Splat Deformation"
authors:
  - Nina Vesseron
  - Élie Michel
year: 2026
published: 2026-09-04 (arXiv)
venue: arXiv（Élie Michel 为 Adobe Research 方向作者）
url: https://arxiv.org/abs/2609.05127
code: ""
project_page: ""
category:
  - gaussian-splatting
  - character-animation
  - skinning
  - deformation
importance: B
game_relevance: Medium（角色渲染方向信号）
production_readiness: Research/Prototype（WebGL 实时演示）
user_level: Normal
status:
  - read
---

# GradRig: Differentiable Weights for Skinned Gaussian Splat Deformation

## TL;DR

骨骼蒙皮是网格的标准做法，但高斯 splat 没有连通性——刚性变换点在拉伸时会产生空洞。GradRig 利用**蒙皮权重的空间梯度**来正确拉伸 splat，得到一条完全 mesh-free 的 GS 骨骼形变管线，保持实时渲染兼容（WebGL 演示），并附自适应重采样方案修补仍有伪影的 splat。

## Problem

如果 GS 要成为角色资产格式，必须能挂骨骼动画。直接对 splat 中心做 LBS 会在关节拉伸处破洞——网格靠三角形连接保持表面，splat 没有这个结构。

## Core Idea

蒙皮权重不只用于混合变换，它的**空间梯度**携带了"表面在这个方向被拉伸多少"的信息。用这个梯度去调整 splat 的协方差（形状），而不只是位置。

## Key Contribution

- 第一条 mesh-free 的 GS 蒙皮形变管线
- 权重梯度在建 rig 时求值，运行时零额外开销
- 自适应重采样兜底：对仍出伪影的 splat 做 split

## Game Development Relevance

短期低（GS 角色离生产很远），但作为**趋势信号**值得记：GS 正在补齐"角色动画"这块拼图（形变 [[GradRig — Differentiable Weights for Skinned Gaussian Splat Deformation|GradRig]] + 外观压缩 [[Compact Neural Appearance Models for Efficient Gaussian Splatting|Compact Neural Appearance]] + 光栅化 [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization|TileGS]]），三块都齐了才谈得上角色管线。

## Related Concepts

- [[Gaussian Splatting]]
- [[Neural Animation]]

## Related Papers

- [[Compact Neural Appearance Models for Efficient Gaussian Splatting]]
- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]
- [[UniMate — One Unified Model to Animate Diverse Skeletons]]（动画来源侧的对应问题）

## Personal Knowledge State

Current Level: Normal（蒙皮与形变是你的知识范围，GS 侧靠 [[Gaussian Splatting]] 已有概念衔接）

## Learning Path

Watchlist 级：知道"GS 蒙皮的坑在协方差不在位置"这一句即可，不必深读。等你需要评估"GS 角色"可行性时再回来。

## Notes
