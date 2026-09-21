---
type: concept
title: "Niagara"
user_level: Easy
tags: [ue, vfx, user-domain]
---

# Niagara

> 用户本人的专业领域。本文件仅为知识图谱锚点，不做教学。

## Definition

Unreal Engine 的 VFX 系统。以 Emitter / System / Module 为组织单位，支持 CPU 与 GPU 两种模拟路径，通过 Simulation Stage 与 Data Interface 实现复杂的多阶段模拟（含 Niagara Fluids 的欧拉网格流体）。

## Core Principle

- **CPU 粒子**：灵活、可精确碰撞、可与游戏逻辑交互；上限约数千~1 万
- **GPU 粒子**：Compute Shader 驱动，可到百万级；逻辑受限、回读困难、有 1-2 帧延迟
- **混合方案**：CPU 端计算低分辨率场（如 32³ 矢量场）写入 3D 纹理，GPU 端采样驱动

经验数据（10000 粒子）：CPU ≈ 1.2ms / GPU ≈ 0.3ms / 混合 ≈ 0.8ms。

## Prerequisites

- [[Real-Time Rendering]]
- [[GPU Architecture]]
- Compute Shader

## Related Concepts

- [[Particle Systems]] —— **本系统的理论源头（1983）**：一帧五步循环 = Emitter 生命周期；"粒子本身是粒子系统" = Emitter Stage 内 Spawn；属性"均值 + 方差" = Random Range / Curve
- [[Real-Time VFX Performance Budgeting]]
- [[Scalability and Quality Tiers]]
- [[Overdraw]]
- [[Tile-Based Rendering]]
- [[Shadow Mapping]] —— **对照**：粒子的"无阴影问题"来自点光源假设（发光不反射），实体几何必须处理阴影 → **两者成本结构完全不同，预算不可混算**

## Research Touchpoints

- **[[Reeves — Particle Systems (1983)]]** → **本系统的历史锚点**。三条可迁移律：① 屏占比 × 密度是比绝对上限更几何的分档变量 ② 发射器数是"树节点数" ③ **随机数只在 Spawn 阶段消费 → 换来确定性与可复现**
- **[[LightOpt — Lights Optimization for Real-Time Rendering]]** → 动态灯光上限可推导化
- **[[DLSS 5 — Generative Neural Rendering]]** → Niagara 输出的 motion vector / depth / albedo 会成为生成模型的 condition，其质量影响神经层稳定性
- **[[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]]** → "端点监督换连续控制"可迁移为 User Parameter 强度滑杆

## Personal Knowledge

Current Level: **Easy**
