---
type: concept
title: "Neural Rendering"
user_level: Hard
tags: [rendering, neural, foundation]
---

# Neural Rendering

## Definition

用神经网络**替代或增强**传统渲染管线中某个可计算环节的一类方法。它不是某一种算法，而是一个**替换策略**：找出管线里最贵或最不准的那一段（采样、积分、着色、重建、合成），用学到的函数顶掉。

## Core Principle

渲染 = 对光传输做积分。传统方法用**采样 + 解析近似**来估这个积分；神经渲染用**训练出来的先验**来估。

交换关系是固定的：

```
用 确定性与可控性 换 速度与表达力
```

这也是为什么它一进游戏领域，争论立刻集中在三件事上（见 [[DLSS 5 — Generative Neural Rendering]]）：
- **Artistic intent preservation**（别把美术的意图平均化）
- **Temporal stability**（别抖、别沸腾、别拖影）
- **Real-time 4K**（别超帧预算）

## Prerequisites

- [[Real-Time Rendering]]
- [[Global Illumination]]
- [[Signal Processing for Graphics]] (采样、重建、滤波)
- 神经网络基础（MLP / 卷积 / 注意力至少知道是什么）
- [[Differentiable Rendering]]（若要理解训练侧）

## Evolution

```
Screen-space 神经后处理（去噪、超分）
        ↓
Neural Upscaling / Frame Generation（DLSS / FSR / XeSS）
        ↓
Neural GI / Neural Shading（NRC、AMD attention GI）
        ↓
3D 表示的神经渲染（NeRF → 3DGS）
        ↓
Generative Rendering（DLSS 5：生成最终外观而非重建参考）
```

关键分叉点：**"重建一个参考" vs "生成最终外观"**。2026 年这个分叉被 DLSS 5 正式跨过。

## Related Concepts

- [[Generative Rendering]]
- [[Differentiable Rendering]]
- [[Gaussian Splatting]]
- [[Neural Global Illumination]]
- [[Temporal Stability]]
- [[Artistic Intent Preservation]]

## Game Applications

- [[AAA Real-Time VFX]]（神经层外包部分表达力）
- 超分与帧生成
- 实时 GI

## Important Papers

- [[DLSS 5 — Generative Neural Rendering]]
- [[Lightweight Attention-based Indirect Illumination (AMD)]]
- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]

## Personal Knowledge

Current Level: **Hard**

（初始化默认推断，待用户校正）

## Learning Gap

1. 缺"采样/重建/滤波"这一层信号处理直觉 —— 这是理解所有超分与去噪的前提
2. 缺神经网络训练侧的词汇（loss、latent、diffusion step）
3. 缺对"帧预算内跑神经网络"的工程感受（tensor core、FP8、算子融合）

## Next Learning Step

不要直接读 neural rendering 论文。先补：
1. TAA / 时域累积为什么需要 motion vector（你大概率已 Easy，作为锚点）
2. 从 TAA 出发理解"超分就是带历史信息的重建"
3. 再进 [[Generative Rendering]]

完整路线见 [[Learning Path — Neural Rendering]]。
