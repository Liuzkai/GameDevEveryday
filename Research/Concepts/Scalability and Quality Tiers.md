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
