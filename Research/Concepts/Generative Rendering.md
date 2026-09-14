---
type: concept
title: "Generative Rendering"
user_level: Hard
tags: [rendering, neural, generative]
---

# Generative Rendering

## Definition

渲染管线的最终画面**由生成模型产生**，而不是由渲染器计算产生，也不是对参考输出的重建。

它是 [[Neural Rendering]] 的一次**范式跃迁**，不是一个新算法。

## Core Principle

三代技术的本质区别：

| 代 | 目标 | 上限 |
|---|---|---|
| 传统渲染 | 计算画面 | 算力能模拟多少 |
| 神经重建（DLSS 2-4） | 近似一个更贵的参考输出 | **渲染器能表达什么** |
| 生成式渲染（DLSS 5） | 生成最终外观 | 模型学到的真实世界先验 |

关键句：**生成式渲染的上限不再是"渲染器能表达什么"，而是"模型知道真实世界长什么样"。**

## Prerequisites

- [[Neural Rendering]]
- [[Real-Time Rendering]]
- [[Temporal Stability]]
- [[Artistic Intent Preservation]]
- 扩散模型 / 一步生成模型基础

## Evolution

```
DLSS 1（每张卡单独训的 super resolution）
   ↓
DLSS 2（通用时域超分 + motion vector）
   ↓
DLSS 3（帧生成）
   ↓
DLSS 4（多帧生成 + 光线重建）
   ↓
[[DLSS 5 — Generative Neural Rendering]]（3D-guided 生成式渲染）
```

## Related Concepts

- [[Neural Rendering]]
- [[Temporal Stability]]
- [[Artistic Intent Preservation]]

## Game Applications

- 高端 PC 画质增强（RTX 50）
- 中长期：可能改变 VFX 的写实度来源（见 [[AAA Real-Time VFX]]）

## Important Papers

- [[DLSS 5 — Generative Neural Rendering]]

## Personal Knowledge

Current Level: **Hard**

## Learning Gap

1. 一步扩散模型如何在毫秒内跑完（与多步采样的差异）
2. deterministic / causal 推理意味着什么
3. "conditioning" 具体把什么东西喂给模型

## Next Learning Step

不要从扩散模型数学开始。先读 [[DLSS 5 — Generative Neural Rendering]] 的 condition buffer 清单——那是你作为渲染工程师唯一真正需要关心的接口层。
