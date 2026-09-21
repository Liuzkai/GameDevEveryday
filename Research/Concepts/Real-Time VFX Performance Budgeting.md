---
type: concept
title: "Real-Time VFX Performance Budgeting"
user_level: Easy
tags: [vfx, performance, production, user-domain]
---

# Real-Time VFX Performance Budgeting

> 这是**用户本人的专业领域**，不是从论文学来的知识。
> 本文件的目的不是教你，而是把你的实践抽象成可与外部研究对接的模型。

## Definition

为游戏中的视觉特效分配**可度量的资源上限**，使得在目标硬件与画质档位下，帧时间、内存与带宽同时不越界的一套方法论。

在 NGR 淬炼系统的语境下，它以**技能**为单位，以 **SABC 分级**为纵轴，以**实测指标 × 五档画质**为约束维度。

## Core Principle

```
技能 → SABC 分级（触发频率 × 难度 × 伤害）
     → 分配资源预算（与分级占比倒挂）
     → 五档画质实测校验
     → 不达标则降级或重构
```

**倒挂配比原则**：S/A/B/C 的资源分配约为 **35-40% / 30% / 20% / 10%**，与各级数量占比（约 10% / 13% / 42% / 34%）相反。即：少数高价值技能吃掉大部分预算，大量低价值技能保持极低成本。

## Prerequisites

- [[Real-Time Rendering]]
- [[GPU Architecture]]
- [[Niagara]]
- [[Profiling and Measurement]]

## 五个量化维度（NGR 现行模板）

| 维度 | S | A | B | C |
|---|---|---|---|---|
| Niagara 发射器数 | ≤12 | ≤8 | ≤4 | ≤2 |
| 同屏粒子数 | 3000 | 1200 | 400 | 100 |
| 贴图尺寸 | 2048 | 1024 | 512 | 256 |
| 动态灯光 | ≤3 | ≤2 | ≤1 | 0 |
| VFX 时长 | ≤3s | ≤2s | ≤1s | ≤0.5s |

## 五档画质

`PC_High` / `PC_Low` / `Android_High` / `Android_Mid` / `Android_Low`

## 实测指标

`Particles` / `DrawCalls` / `Primitives` / `OverDraw` / `CPUTime` / `GPUTime`

## Related Concepts

- [[Scalability and Quality Tiers]]
- [[Overdraw]]
- [[Tile-Based Rendering]]
- [[Niagara]]

## Game Applications

- [[AAA Real-Time VFX]]

## External Research Touchpoints

外部研究对这个模型的冲击点：

1. **[[Reeves — Particle Systems (1983)]]** → **你的五个维度里有四个的原始出处**（屏占比 LOD / 发射器层级 / 寿命 / 加法混合免排序）。**含义：这份预算表可以引用 40 余年前的成本法则来论证，而不必只靠实测经验值。** 见 [[预算五维_1978-1983_源头图解]]
2. **[[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]** → **"动态灯光"维度的成本依据**：每盏灯的阴影代价 ≈ +1× 场景渲染（单位是"一遍渲染"，不是"一个图元"）→ **这才解释了为什么它的上限是 0–3 而同屏粒子是 100–3000**
3. **[[LightOpt — Lights Optimization for Real-Time Rendering]]** → 把"动态灯光上限"从经验值变成可推导值
4. **[[DLSS 5 — Generative Neural Rendering]]** → 部分表达力外包给神经层，预算的自变量集合会变（但代价是 -50% 帧率，且只覆盖 PC）。**2026-09-21 补：神经层已确认消耗三个预算维度 —— 功耗（9-19）、帧时间（9-20）、显存/设备缓冲（9-21，Intel Xe2 重实现在 720p 占约 2.3 GiB）**
5. **[[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]]** → "端点监督换连续控制"的思路可迁移：S/A/B/C 是 4 个离散端点，还是 1 个强度滑杆？
6. **[[MotionBricks — Scalable Real-Time Motions]]** → 若技能动作变成运行时生成，VFX 时序锚点必须从"固定帧"改为"语义事件"（**2026-09-21 由 [[2026-09-18-GestureFAR — Streaming Co-Speech Gesture Generation with Flow Autoregression]] 独立佐证第二次**）
7. **🔴 一条硬约束（2026-09-21，二手待核实）** → 二手来源称 **UE 5.8 MegaLights 不支持半透明物体 / 流体 / 云 / 发丝，也不支持前向渲染** → **特效打光不在 MegaLights 覆盖范围内**，仍走 [[Shadow Mapping]] 的成本法则。**"动态灯光变便宜"不能直接推到 VFX 侧。**

## Personal Knowledge

Current Level: **Easy**

## Next Step

你的 Normal 区不在这里，而在：
- 把这些经验上限**形式化**（为什么是 12 个发射器？）
- 与 [[Differentiable Rendering]] 结合，把"上限"变成"优化解"
- 预判神经渲染普及后预算模型的重构
