---
type: concept
title: "Gaussian Splatting"
user_level: Easy
tags: [rendering, 3d-representation, radiance-fields]
---

# Gaussian Splatting

## Definition

用大量**显式 3D 高斯椭球**作为场景表示，通过可微光栅化（splatting）实现高质量新视角合成。2023 年提出后迅速成为神经渲染领域最主流的显式表示。

## Core Principle

- 场景 = 一堆带位置、协方差（形状）、不透明度、颜色（通常用球谐表示视角相关颜色）的高斯
- 渲染 = 把高斯投影到屏幕 → 按深度排序 → alpha 混合
- 训练 = 可微光栅化 + 梯度下降 + 自适应密度控制（分裂/克隆/剪枝）

## Prerequisites

- [[Neural Rendering]]
- 协方差与椭球（线性代数）
- [[Differentiable Rendering]]
- Alpha blending / 半透明排序

## Evolution

```
NeRF（隐式 MLP + volume rendering，慢）
        ↓
3D Gaussian Splatting（显式基元 + 可微光栅化，实时）
        ↓
工程化深水区（2026）：
  · 收敛速度 → 结构感知密集化（SADGS）
  · 重建质量 → 视频扩散先验（VidSplat）、物理+光学混合（GauSmoke）
  · 规模 → 光场显示（CoherentRaster）、4D 不确定性（GraphiXS）、
           语义抠资产（LangSplatV2）+ 稀疏体素（fVDB）
  · 渲染管线 → 随机光栅化（取消排序与混合）+ 时域神经去噪 ★ 2026-09-24 入库
        ↓
成为生成式世界模型的原生输出格式
```

**2026 年的关键信号**：SIGGRAPH 2026 Workshop 直接宣布 3DGS 已经成熟为**生成式世界模型的原生输出格式**。"世界模型生成 → 3DGS 表示 → 可交互世界"被行业默认为一条已成立的流水线。

## Related Concepts

- [[Neural Rendering]]
- [[Differentiable Rendering]]
- [[Radiance Fields]]

## Game Applications

- 场景/资产捕获与重建
- VR 照片级写实
- 开放世界重建（[[Open World Reconstruction]]）

## Important Papers

- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]
- [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising]] —— **"排序瓶颈"的第四条路线（取消式）**：随机光栅化 + 时域神经去噪；全管线 2.1×、去噪 +1 ms 常数（与 TileGS 构成同题两端）★ 2026-09-24
- [[Inverse Rendering for Modeling with Line Primitives]]（对比：显式线段 vs 体积基元）

## Personal Knowledge

Current Level: **Easy**（2026-09-11 标 Easy，4 条 Mastery 判据全过）

讲解笔记（图解两轮，含嵌入图）：[[GS 图解 1 — 协方差与椭球：高斯的形状说明书]] · [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制]]

## Learning Gap（2026-09-11 全部闭环）

1. ~~为什么排序是它的性能瓶颈~~ → [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制]]（blending 不交换 × 百万级 × 每像素几十层）
2. ~~它为什么难进标准管线~~ → 同上：G-buffer 单表面假设被"每像素几十层雾"直接违反
3. ~~自适应密度控制在做什么~~ → 同上：训练时的发射器管理（split / clone / prune）

## Mastery Criteria

- [x] 说清为什么 3DGS 比 NeRF 快
- [x] 解释它的排序需求与半透明粒子排序的同构性
- [x] 说清它为什么不能直接进 deferred 管线
- [x] 判断一个给定场景该用 mesh 还是 3DGS

## Next Learning Step

~~→ [[Learning Path — Gaussian Splatting]]~~ ✅ 2026-09-11 完成，标 **Easy**。

后续按约定**停止基础推送**，只推建立在 GS 之上的新研究（动态 GS、GS 角色管线等）。

> **2026-09-24 更新**：首篇"GS 之上的新研究"已入库 —— [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]]（**排序瓶颈的"取消式"解法**：把排序与混合从管线里删掉，用 ~1 ms 神经去噪器偿还噪声债；自由导航 PSNR 29.80 vs ST-TAA 21.48）。**Easy 标记不变** —— 该文不含 GS 基础内容，全部价值在"管线置换 + 去噪器设计"两层。
