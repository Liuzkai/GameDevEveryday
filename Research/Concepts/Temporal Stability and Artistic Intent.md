---
type: concept
title: "Temporal Stability and Artistic Intent"
user_level: Normal
tags: [neural, quality]
aliases: [Temporal Stability, Artistic Intent Preservation, 时域稳定性, 艺术意图保留]
---

# Temporal Stability and Artistic Intent

> 这是**同一枚硬币的两面**：神经渲染进入生产所必须同时解决的两个非技术性硬门槛。

## Artistic Intent Preservation

**问题**：生成/重建模型是在大量数据上训练的，天然把结果往"数据的平均"拉。这会让画面偏离美术的既定风格。

**行业反应**：2026 年这个问题被正式写进研究议程与产品参数。DLSS 5 提供了 `Structure Intensity` / `Tone Intensity` 与语义 mask 作为显式控制；SIGGRAPH 2026 的神经渲染论文普遍把它列为三大挑战之一。

**关键判别**：当产业界把用户的批评写进产品架构，说明这个方向不是炒作，而是在解决真实问题。

## Temporal Stability

**问题**：抖动（flicker）、沸腾（boiling）、拖影（ghosting）——时域不一致。静态对比图看不出来，一动就废。

**为什么难**：模型对相邻帧的微小输入变化可能产生非连续的输出变化。

**工程惯例**：用引擎 motion vector 做 reprojection、carried temporal state、一帧进一帧出的因果设计、训练时加时域一致性损失。DLSS 5 明确宣称其推理是 causal 与 deterministic 的，并为帧间稳定做了专门训练。

**光照域的系统性答案（★ 2026-10-14 补）**：时域稳定问题在"光照重建"域的工程化起点是 [[Schied — Spatiotemporal Variance-Guided Filtering (2017)]]——**时域累积 + 方差引导的空间滤波**：历史一致性（depth/法线/ID）决定"敢不敢复用"，方差决定"哪里多滤、哪里保留"。概念要点：**"可信度"是每像素估计出来的量**（方差/历史长度/置信度都是其形态），而不是全局参数。

## Why This Matters for VFX

对做特效的人而言，这两条是**判断一个神经方法能否进管线的先行指标**：

- 不稳定的神经增强 + 高频运动的粒子 = 最坏情况
- 粒子本身就是时域高频内容 → VFX 是神经渲染最容易翻车的场景

## Related Concepts

- [[Neural Rendering]]
- [[Generative Rendering]]
- [[Real-Time VFX Performance Budgeting]]

## Important Papers

- [[DLSS 5 — Generative Neural Rendering]]
- [[Schied — Spatiotemporal Variance-Guided Filtering (2017)]] —— **重建分支的奠基之作**（★ 2026-10-14 入库：1 spp 实时重建；时域累积 + 方差引导 à-trous；降级路径 = 历史不足退回空间估计）

## Personal Knowledge

Current Level: **Normal**

## Next Step

把这两条加进你的 VFX 验收清单：任何引入神经方法的特效，必须单独做"快速镜头运动 + 粒子高频闪烁"的时域稳定性检查。
