---
type: concept
title: "Neural Physics Simulation"
user_level: Hard
tags: [physics, neural-simulator, particles]
---

# Neural Physics Simulation

## Definition

用学习模型替代（或加速）传统数值求解器的仿真路线。核心问题不是"能不能学"，而是**学什么、不学什么**：把已知的物理结构（外力、守恒律、边界条件）留给显式计算，只把难以解析建模的部分（粒子间相互作用、本构关系）交给网络。

## Core Principle

```
显式部分（已知、便宜、确定）     学习部分（难建模、贵）
─────────────────────────     ─────────────────────
重力/外力推进                   粒子间相互作用
边界约束                       本构关系（材料响应）
守恒结构                       长程耦合
```

这个分工和 [[World Models for Games]]（引擎管规则/模型管呈现）、[[DLSS 5 — Generative Neural Rendering|DLSS 5]]（渲染器管结构/生成模型管外观）是同一哲学的三个实例——**2026 年行业级的"分层收敛"**。

## Prerequisites

- [[Real-Time VFX Performance Budgeting]]（Easy — 你已懂粒子的成本结构，这是最好的入口）
- 拉格朗日粒子表示（Easy — Niagara 日常工作）
- GNN 类仿真器（GNS 等，Normal — 粒子=节点、相互作用=边）
- Attention / token merging（Hard — 本概念的真正缺口）

## Evolution

- Deep Lagrangian Networks（2020，早期端到端尝试）
- GNS — Graph Network Simulators（2020，确立"粒子=图节点"范式）
- TIE / Neural Operators（2021-2022，算子学习分支）
- [[WorldParticle — Unified World Simulation of Lagrangian Particle Dynamics via Transformer|WorldParticle]]（2026，SIGGRAPH Asia — 首次单架构统一六类动力学）

## Related Concepts

- [[World Models for Games]]
- [[Generative Rendering]]

## Game Applications

- 远期：统一 VFX 求解器（一个模型 + 每特效一个 condition，替代"每现象一套求解器"）
- 中期：离线替代仿真（LOD 远处特效用学习模型近似）
- 现在：无可落地路径，纯概念储备

## Important Papers

- [[WorldParticle — Unified World Simulation of Lagrangian Particle Dynamics via Transformer]]

## Personal Knowledge

Current Level: **Hard**

## Learning Gap

唯一的实质缺口是 Transformer 注意力机制词汇。物理侧你已全部具备。

## Next Learning Step

**不要读 attention 教材。** 直接读 [[WorldParticle — Unified World Simulation of Lagrangian Particle Dynamics via Transformer|WorldParticle]] 的 Figure 1 + 项目页视频，把 super-token merging 类比为你熟悉的东西：**粒子 LOD / 分簇**——"先压缩相互作用图，再在压缩域里算贵的部分"。这个类比成立之后，本概念可从 Hard 升 Normal。
