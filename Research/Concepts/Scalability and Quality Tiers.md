---
type: concept
title: "Scalability and Quality Tiers"
user_level: Easy
tags: [performance, production, user-domain]
---

# Scalability and Quality Tiers

## Definition

同一份内容在不同硬件能力下，用**分档参数**产出不同画质/性能平衡的机制。

## Core Principle

分档不是"开关功能"，而是**在多维参数空间里定义若干条可行轨迹**。常见维度：分辨率、阴影质量、后处理、粒子数量、植被密度、LOD bias、动态光数量。

## NGR 现行五档

```
PC_High      基准，特效完整
PC_Low       削减后处理 / 降分辨率 / 降粒子
Android_High 移动端上限，需要注意 TBDR 带宽
Android_Mid  主流机型目标
Android_Low  保底，优先稳定帧率
```

**移动端与 PC 的分档不是同一条曲线。** Android 侧是 TBDR 架构（见 [[Tile-Based Rendering]]），带宽与 tile 内存是首要约束；PC 侧更偏算力与显存。用同一套降级顺序在两端都会踩坑。

## 🔴 一个正在发生的前提变化：档位从"调参数"变成"选管线"（2026-09-20 记录）

**上面的定义隐含一个前提**：所有档位都跑**同一条渲染管线**，差异只在参数（分辨率 / 质量 / 数量 / 频率）。

**2026-09-20 出现了一个把这个前提顶掉的样本** —— Remedy《控制：共振》：

| 事实 | 数据 |
|---|---|
| **所有光追预设都强制内置路径追踪 + 全局光照** | PT 不是独立高档位，而是"开了光追就必然进入的管线" |
| 原生 4K + 中等光追，**关闭全部升频** | RTX 5090 **48 fps**；RTX 5080 **< 30 fps**（TechPowerUp） |
| 原生 4K Ultra，**开完整 PT vs 不开** | 5090 **72 → 39**（**−46%**）；4090 64 → 27；5080 56 → 23（GameGPU） |
| PS5 性能模式 | 1440p/60（**内部渲染约 864p + FSR 升频**） |

**后果**：**中间档消失了。** 玩家面对的不再是"高/中/低"，而是 **"进不进 PT 管线"的二值选择**，以及"用多激进的升频把它拉回来"。同期 TweakTown 的评述把结论写成同一件事：**升频已从"可选的性能加成"变成"基线优化工具"**。

> **对分档设计的三条具体建议**：
> 1. **在 S/A/B/C × 五档的矩阵里，显式区分两类档位差异：「换参数」与「换管线」。** 后者不是"降一档"，而是**一次结构性切换**（GI 方案、阴影方案、材质 Shading Model 数、是否启用神经层）；
> 2. **"每档一套独立的资源上限"这个前提要重新审视** —— 换管线的档位需要**另一套上限**，而不是同一套上限的缩放；
> 3. **移动端多一条独立约束**：神经层的算力**可能要先跟功耗墙谈判**。2026-09-20 记录：Windows Auto SR 走 NPU，但 **NPU 与 CPU/GPU 共享整机功耗预算**，官方明说部分轻薄本会升温 / 掉续航 / 小幅掉帧。**对 Android 三档而言，"要不要给神经层留功耗预算"是前置决策，不是后期优化。**

## Prerequisites

- [[Real-Time Rendering]]
- [[GPU Architecture]]
- [[Tile-Based Rendering]]

## Related Concepts

- [[Real-Time VFX Performance Budgeting]]
- [[Overdraw]]
- [[Niagara]]

## Game Applications

- [[AAA Real-Time VFX]]

## Personal Knowledge

Current Level: **Easy**

## Next Step

你的分档目前是**人工定义**的参数集合。值得探索的方向：把"画质档位"定义为**外观误差阈值**，再由优化自动求解各档参数——这正是 [[LightOpt — Lights Optimization for Real-Time Rendering]] 在灯光维度上做的事。
