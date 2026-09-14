---
type: application
title: "AAA Real-Time VFX"
user_level: Easy
tags: [vfx, production, user-domain]
---

# AAA Real-Time VFX

> 用户本人的工作领域。本文件是知识图谱的**应用侧锚点**：把外部研究映射到具体的美术/工程决策。

## Overview

在严格的帧预算内，为开放世界/多人项目产出角色技能、环境、UI 的视觉特效。核心矛盾永远是：**观感要求无限，帧预算有限**。

## 约束结构（NGR 淬炼语境）

```
技能
 ├─ SABC 分级（触发频率 × 难度 × 伤害）
 ├─ 五个预算维度（发射器 / 粒子 / 贴图 / 动态灯 / 时长）
 ├─ 五档画质（PC_High … Android_Low）
 └─ 六个实测指标（Particles / DrawCalls / Primitives / OverDraw / CPU / GPU）
```

## 关键工程约束

1. **Overdraw 是半透明特效的第一杀手**——移动端尤其（打断 HSR）
2. **DrawCall** 在 GPU-Driven 架构下已部分缓解，但仍是移动端硬约束
3. **动态灯光** 数量在 forward/clustered 下按像素付费
4. **技能密度**：多人同屏多角色同时放技能 → 预算必须按**最坏并发**而非单个技能计算

## 外部研究映射

| 研究 | 影响维度 | 时间尺度 |
|---|---|---|
| [[LightOpt — Lights Optimization for Real-Time Rendering]] | 动态灯光上限 → 可推导 | 中期（工具化后） |
| [[DLSS 5 — Generative Neural Rendering]] | 表达力外包，预算自变量集合变化 | 长期 / 仅 PC |
| [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]] | 分档从离散端点 → 强度滑杆 | 中期（方法论迁移） |
| [[MotionBricks — Scalable Real-Time Motions]] | VFX 时序锚点从帧 → 语义事件 | 长期 |
| [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]] | tile-local 思维与移动端排序 | 弱 |
| [[LLM-Guided RL for Adaptive NPC Behavior]] | 低频智能 + 高频执行的结构可迁移 | 弱 |

## Open Questions（值得你自己回答）

1. S/A/B/C 的动态灯上限（3/2/1/0）是经验值还是推导值？能否用 LightOpt 思路重算？
2. 如果 PC_High 档默认开启神经渲染，粒子预算该上调还是下调？
3. 分档是"4 个离散端点"还是"1 个强度滑杆"？后者能省多少维护成本？
4. 最坏并发场景（5v5 团战）下，当前的单技能预算是线性叠加还是需要协同降级？

## Related Concepts

- [[Real-Time VFX Performance Budgeting]]
- [[Niagara]]
- [[Scalability and Quality Tiers]]
- [[Overdraw]]
- [[Tile-Based Rendering]]

## Personal Knowledge

Current Level: **Easy**
