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

**另一条反向路线（2026-09-21 补记）**：不是在 RGB 上"生成最终外观"，而是把**物理渲染器改写进生成模型的潜空间** —— [[2026-09-17-PBR-Latent — Physically Based Rendering in the Latent Space]] 给渲染方程加三项修正（带符号辐射 / 平响应 / 遮挡）以直接输出 VAE latent 图。**前者是"生成替代物理"，后者是"物理进入生成的表示"——两条方向相反、可互补的路线。**

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
- [[2026-09-17-PBR-Latent — Physically Based Rendering in the Latent Space]] —— **反方向**：把物理渲染器改写为潜空间渲染器（"物理进入生成"而非"生成替代物理"；其 SDS 应用模式可先于实时侧落地）
- [[2026-10-22-Liu — AnyGameRenderer Entity-Consistent Generative Rendering Across Games|AnyGameRenderer]]（★ 2026-10-24 入库）——**"实体账"成为一等验收项**：输入纯 gameplay 视频（灰盒/图元/语义图/线框均可，不需要 G-buffer 与类别标签），输出同镜头、同实体位置与时序的成品画面；提出**实体级 benchmark**（59 例 / 753 实体、四个指标）与**字幕契约**（"哪个色块 = 哪个实体 + 位置/尺寸/时序/人数严格保留"）。**它给这一族技术补上了此前缺的那把尺子**：不问"像不像"，问"该出现的都出现了吗、在对的帧吗、在对的位置吗、有没有多出来的"。

## 验收标准（2026-10-24 追加）

生成式渲染一旦**生成内容而不只是补细节**，验收就必须从"帧级"下沉到"实体级"。AnyGameRenderer 的四项可以直接借用：

| 指标 | 问题 | 对 VFX/技能的平移 |
|---|---|---|
| Unrendered rate | 该渲染的没渲染？ | 该有的粒子/拖尾漏了吗 |
| Temporal match | 出现/消失的帧对吗？ | 第 12 帧亮、第 40 帧灭，对上了吗 |
| Position match | 位置对吗？ | 挂点/偏移对吗 |
| Hallucination rate | 有没有凭空多出来的？ | 多出来的残影/鬼影 |

**注意它同时公布的"GT 天花板"**：把真值视频喂进同一套度量管线，幻觉率仍有 34.1%——**度量本身有噪声，读数时要把这层扣掉**。

## Personal Knowledge

Current Level: **Hard**

## Learning Gap

1. 一步扩散模型如何在毫秒内跑完（与多步采样的差异）
2. deterministic / causal 推理意味着什么
3. "conditioning" 具体把什么东西喂给模型

## Next Learning Step

不要从扩散模型数学开始。先读 [[DLSS 5 — Generative Neural Rendering]] 的 condition buffer 清单——那是你作为渲染工程师唯一真正需要关心的接口层。
