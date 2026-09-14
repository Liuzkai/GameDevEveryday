---
type: concept
title: "Tile-Based Rendering (TBDR)"
user_level: Normal
tags: [gpu, mobile, architecture]
---

# Tile-Based Rendering (TBDR)

## Definition

移动端 GPU 的主流架构：不直接往整帧 framebuffer 上画，而是把屏幕切成小 tile，逐 tile 把几何与着色做完，最后一次性写回外部内存。

## Core Principle

**带宽是移动端的第一约束，不是算力。**

```
Immediate Mode (PC):  每次 draw → 直接读写外部显存
Tile-Based (Mobile):  几何分桶到 tile → tile 内全在片上 SRAM 完成 → 一次性写回
```

推论（对 VFX 尤其重要）：

- **Overdraw 在 TBDR 上不像 PC 那样直接烧带宽**（片上混合便宜），但**会烧 tile 内存与着色算力**
- 一旦 tile 内存放不下（大量 render target、MSAA、半透明层），会 tile flush 到内存，性能断崖
- 透明/半透明 pass 会强制打断 HSR（Hidden Surface Removal）→ 半透明粒子多是移动端大忌

## Prerequisites

- [[GPU Architecture]]
- [[Real-Time Rendering]]

## Related Concepts

- [[Overdraw]]
- [[Scalability and Quality Tiers]]
- [[Gaussian Splatting]]（tile-local 排序）

## Game Applications

- Android 三档画质优化
- [[AAA Real-Time VFX]] 的移动端降级策略

## Important Papers

- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]

## Personal Knowledge

Current Level: **Normal**

## Mastery Criteria

- [ ] 说清为什么 TBDR 省带宽
- [ ] 解释半透明为什么打断 HSR
- [ ] 说清 tile 内存溢出的后果
- [ ] 判断一个 VFX 在移动端的主要开销是带宽还是算力
- [ ] 对比 PC 与移动端对同一特效的降级顺序差异

## Next Step

把它与你的五档画质体系显式绑定：写出"PC 降级顺序"与"Android 降级顺序"两条不同的规则。
