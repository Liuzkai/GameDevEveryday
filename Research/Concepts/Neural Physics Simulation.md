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
- [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]]（2026，Meta × NVIDIA — **神经时间积分器**：镜像经典积分解算器的输入输出，模拟器在环监督；角色级（发丝）实时）
- [[2026-10-06-PhysLDM — Latent Diffusion for High-Fidelity Deformable Simulation|PhysLDM]]（2026，NTU — **整轨迹分布建模**：时空 VAE（78× 压缩）+ 潜扩散一次性预测；**"混沌判据"**：混沌动力学上确定性回归收敛到非物理平均值，应改为建模分布）
- 毛发神经仿真三代（同域对照）：GroomGen（2023）→ Quaffure（2025，准静态）→ Neuralocks（2026）→ **Neuroll（2026）**

## Related Concepts

- [[World Models for Games]]
- [[Generative Rendering]]

## Game Applications

- 远期：统一 VFX 求解器（一个模型 + 每特效一个 condition，替代"每现象一套求解器"）
- 中期：离线替代仿真（LOD 远处特效用学习模型近似）
- **2026-10-06 现状更新**：**首个"角色级"实时原型出现**（[[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]]：发丝仿真 3000 股 0.460 ms/帧、消费级硬件、密度无关线性扩展）——从"纯概念储备"变为"有可指向的原型路径"；VFX 粒子侧仍无同级样本。

## Important Papers

- [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling]]（★ 2026-10-06 入库：**"镜像经典 I/O"的神经积分器**——替代数值积分、保留经典接口语义；模拟器在环 + 随机视界；strand-space 消融 = "表示即先验"最干净量化）
- [[2026-10-06-PhysLDM — Latent Diffusion for High-Fidelity Deformable Simulation]]（★ 2026-10-07 入库：**可形变体 + 分布建模**——时空 VAE + 潜扩散；**"混沌判据"**：扰动放大实验证明回归产生"non-physical averages"，扩散才建模分布；可微逆问题 40 秒级）
- [[WorldParticle — Unified World Simulation of Lagrangian Particle Dynamics via Transformer]]

## 一条新判据（2026-10-07，来自 PhysLDM）

**"这个系统的预测目标是一个点，还是一个分布？"** —— 先做扰动实验判断系统是否处于混沌区（微小扰动被放大）：若是，**确定性回归在数学上就是错的**（收敛到非物理平均值），应改用生成式建模。此判据与"分工判据"（新旧工具各接管哪段）并列，收入本库选型工具箱。

## Personal Knowledge

Current Level: **Hard**（入口已扩：**机制层（"神经替代哪一步、保留什么接口"）现在有 [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]] 一篇可直接读**；训练/推导层仍缺）

## Learning Gap

- **概念层（可读）**：分工判据——"显式部分 vs 学习部分"在 Neuroll 里有一个具体答案（**替代积分步骤、保留经典 I/O 语义与显式物理状态**）；你的粒子机制 Easy 可支撑这一层阅读；
- **未确认层（按 2026-10-05 PKM 校正记录）**：attention、学习式仿真的训练细节与实现——**不再断言"只缺一个 attention 词汇"**（该表述已由 PKM 校正替代）；训练目标、误差累积、泛化条件都是实际缺口，按具体问题再建桥。

## Next Learning Step

1. **先读 [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]] 的机制层**（Figure 2 训练管线 + Table 1 数字）——25 分钟，用自己的话回答"它替代了解算器的哪一步、保留了什么"；
2. [[WorldParticle — Unified World Simulation of Lagrangian Particle Dynamics via Transformer|WorldParticle]] 的 Figure 1 + 项目页视频仍可作为"压缩相互作用图 = 粒子 LOD"的类比入口；
3. 训练/推导侧（扩散 / flow matching / 图网络训练）沿 [[Neural Rendering]] / 具体论文的问题缺口补，不在本概念层展开。
