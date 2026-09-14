---
type: paper
title: "LightOpt: Lights Optimization for Real-time Rendering"
authors: ["Tuo Chen (Tsinghua)", "Luyan Cao (LIGHTSPEED)", "Kui Wu (LIGHTSPEED)", "Shimin Hu (Tsinghua)"]
year: 2026
published: "2026-07-20"
venue: "SIGGRAPH 2026 Conference Paper Track (47:1-47:10)"
url: "https://s2026.conference-schedule.org/presentation?id=papers_372&sess=sess134"
code: ""
project_page: ""
category: ["Rendering", "Real-Time Rendering", "Optimization", "Production"]
importance: "S"
game_relevance: "Critical"
production_readiness: "Prototype"
user_level: "Normal"
status: unread
tags: [real-time-rendering, lighting, optimization, differentiable, mobile, tencent]
---

# LightOpt: Lights Optimization for Real-time Rendering

## TL;DR

一个**可微优化框架**，自动减少实时游戏场景中的灯光数量，同时保持视觉外观不变。通过优化并重构实时光源布局，降低光照开销、减少光源重叠、提升跨平台性能。SIGGRAPH 2026，清华 + 腾讯光子工作室（LIGHTSPEED）。

## Problem

实时场景里的灯光是被"摆"出来的，不是被"解"出来的。美术凭经验布光 → 光源数量膨胀、互相重叠、冗余光源大量存在。每一盏实时光都在 forward/deferred 里按像素付代价，在移动端更是直接决定能不能跑。已有的灯光剔除/合并多为启发式规则，缺乏一个"保持外观不变"的优化目标。

## Core Idea

把"灯光布局"当作一个**可优化变量**：定义外观保真度目标，对光源参数（位置、强度、范围、甚至数量）做可微优化，让优化器自己找出"删掉/合并哪些灯，观感不变"。

## Technical Approach

（官方描述层面）Differentiable optimization framework：
- 对实时光源做优化与重构（restructure）
- 目标函数 = 外观保持
- 结果：更少的光源数、更低的重叠度

## Key Contribution

- 把灯光数量从"美术手工决策"变成"可自动求解的优化问题"
- 直接面向**跨平台**（含移动端）性能
- 工业界（腾讯光子）与学界（清华）合作，问题定义来自真实项目

## Game Development Relevance

**这是本日对你的个人相关性最高的一条。**

你正在为 NGR 淬炼系统设计 SABC 分级的性能预算，其中"动态灯光"是五个量化维度之一，上限为 S≤3 / A≤2 / B≤1 / C=0。LightOpt 正好回答了这个预算项背后的真问题：

1. **这些上限是拍出来的，还是算出来的？** LightOpt 提供了一条从"保持外观"反推"最少需要几盏灯"的路径——你的上限可以从经验值升级为可推导值。
2. **预算之外的冗余**：即使某技能只用了 3 盏灯（S 档达标），这 3 盏灯本身可能仍有合并空间。
3. **跨五档画质的分档依据**：同一技能在 Android_Low 上该砍到几盏，可以用"外观误差阈值"来定，而不是拍脑袋砍。

## Unreal Engine Relevance

- UE 侧对应：Lights / Lumen / MegaLights 的 light count budget、Clustered Deferred Shading 的 per-cluster light culling
- 潜在落地形态：Editor 工具——选一组动态光 → 一键求解"最少保留几盏"
- 与你的淬炼流水线结合方式：把 LightOpt 式优化做成**资产入库前校验项**

## Limitations

- SIGGRAPH 论文，未在引擎中产品化；无公开代码（截至 2026-09-07）
- 可微优化通常是**离线/editor-time**的，不是运行时——所以它优化的是"配置"，不是"每帧"
- 外观保持的度量标准（用什么 loss）决定成败，人眼感知与 L2 差异未必一致
- 对**动态移动/变色/闪烁**的技能光（正好是 VFX 里最常见的）可能不适用或需特殊处理

## Related Concepts

- [[Real-Time Rendering]]
- [[Differentiable Rendering]]
- [[Real-Time VFX Performance Budgeting]]
- [[Scalability and Quality Tiers]]

## Related Technologies

- [[GPU-Driven Rendering]]

## Related Papers

- [[DLSS 5 — Generative Neural Rendering]]
- [[Lightweight Attention-based Indirect Illumination]]

## Personal Knowledge State

Current Level: **Normal**

Reason:
你对"动态灯光数量"已有明确的工程直觉和量化上限（S≤3 等），缺的是把它从经验值变成优化解的数学工具链：可微优化怎么对离散的"灯的数量"求导、外观 loss 怎么定义、为什么它需要离线跑。

## Learning Path

1. 补 [[Differentiable Rendering]] 的最小前置：可微渲染 = 渲染结果对场景参数可求导
2. 理解"离散决策（灯要不要留）如何变成连续可优化量"——通常是 relaxation + sparsity loss
3. 拿你的 S/A/B/C 动态灯上限做一次思想实验：如果外观误差阈值可设，四档上限会变成多少？
4. Mastery 判据：能说清"为什么这个优化必须离线做，不能运行时做"

## Notes

2026-09-07 首次收录。今日 Tier S。理由不是学术影响力，而是**它与你手上正在做的预算维度精确对齐**。建议优先精读，即使只有摘要。
