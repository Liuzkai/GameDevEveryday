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

### 第三格：不学，只求导（2026-11-13 补）

上面那张两栏表会让人以为世界只有"显式"与"学习"两格。**还有第三格：求解器照旧显式，但让它可微。**

本页此前把这件事当成下游工具（[[2026-10-06-PhysLDM — Latent Diffusion for High-Fidelity Deformable Simulation|PhysLDM]] 的"可微逆问题 40 秒级"就是这么写的）。但它其实是一条独立的路线，判据完全不同：

> **问一句"它替代了求解器，还是在求解器外面套了一层微分？"**
> 前者改的是**结果从哪来**（学一个映射，省掉解方程）；
> 后者改的是**结果能往哪去**（梯度能回流到参数，于是能反解）。

两个问题不能用同一个判据回答：前者看**误差与泛化**，后者看**梯度质量与内存**。混在一起谈，就会出现"这个神经求解器可微吗"这种没有信息量的问题——**可微是求解器的属性，不是网络的属性**（显式求解器也可以可微，见 [[McNamara 2004 — Fluid Control Using the Adjoint Method|McNamara 2004]]）。

### 混沌判据的另一面（2026-11-13 补）

本页从 PhysLDM 收下的"混沌判据"（扰动被放大 ⇒ 确定性回归在数学上就是错的）在**求导**这一侧有个对偶结论，来自 [[2026-11-13-Shu — Same Physics Different Gradients A State-of-the-Art Report on Differentiable Physics Simulation|Shu 2026]]：

- 预测侧：混沌区不能回归**一个点**，要建模**分布**；
- 求导侧：混沌区不能拿"**方程的根**"当参照物（根对初值敏感，扰动一放大参照物自己就跑了），只能退到"**我实际算过的那条轨迹**"。

**同一条物理事实，在两个问题上各砍掉一个默认假设。** 差别在于：预测侧退到分布是有代价的（生成式建模），求导侧退到"算过的轨迹"是**免费的**——因为那条轨迹本来就在显存里。

### 第四格：不学过程，只学终局（2026-11-14 补）

前两格问的是"谁在推进时间"，第三格问的是"要不要微分"。还有第四格：**时间这个自变量可以被整个拿掉**。

[[2026-11-14-Luo — PIM A Physical Intuition Model for Low-Latency Prediction of Object Interaction Outcomes|PIM 2026]] 的做法是直接学 $\{\text{初态}\}\mapsto\{\text{平衡态}\}$——一次前向出所有顶点的**终态位置**，没有中间帧。它的许可证写在定义里：*"States involving persistent motion, including dynamic equilibria, are outside the scope of this work."*

> **判据：这个问题的答案里，时间是不是一个自变量？**
> 是 ⇒ 必须 rollout（或蒸馏 rollout）；
> 不是 ⇒ 可以直接预测不动点，但要接受"没有中间状态"。

**代价的形状与前两格不同**：不是误差变大，而是**输出少了一个维度**。所以 PIM 那张对比图里，它的中间时刻那一格写的是 **N/A**——**不是没算，是方法的定义域里没有那一格。**

**第四格顺带给出了一条读表判据（比结论本身更值钱）**：PIM 同时报告 **9692×**（Deformable 单样本延迟）与 **1.48×**（Flat 批吞吐）。同一个模型、同一篇论文，两个数字差三个数量级，原因是分母不同：

| 口径 | 分母在量什么 | 适合谁 |
|---|---|---|
| **单样本延迟** | 这个仿真器在这个算例上有多贵 | **游戏**（一帧一个样本 + 16 ms 硬期限） |
| **批吞吐** | 每秒能出多少结果 | 离线数据生成 |

**仿真的成本线性于（物体数 × 接触数 × 步数），网络的成本是常数。** 所以"比仿真器快 N 倍"这句话，N 测的是**仿真器**，不是模型。**看到巨大的加速比，先数调用次数（1 次 vs 250 次），再看分母是延迟还是吞吐。**

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
- [[2026-10-08-RiCo — Neural Simulation of Rigid-Body Interactions via Local Contact Reasoning|RiCo]]（2026，CUHK-Shenzhen × DexForce — **刚体接触**：跨物体推理限定在"稀疏局部接触邻域"（点级）、物体内 point transformer 汇总、锚点 + Kabsch 恢复刚体变换；**"接触是局部的"这一物理事实写进网络结构**）
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
- [[2026-10-08-RiCo — Neural Simulation of Rigid-Body Interactions via Local Contact Reasoning|RiCo]]（★ 2026-10-10 入库：**"结构对齐物理"的接触推理**——跨物体信息交换的边界 = 物理作用边界；接触保真 ΔPTR 11.0% / ΔMPD 2.22 mm（基线 ~43% / ~270 mm）；**"指标看不见的价值"（符号距离消融）**）
- [[WorldParticle — Unified World Simulation of Lagrangian Particle Dynamics via Transformer]]

## 一条新判据（2026-10-07，来自 PhysLDM）

**"这个系统的预测目标是一个点，还是一个分布？"** —— 先做扰动实验判断系统是否处于混沌区（微小扰动被放大）：若是，**确定性回归在数学上就是错的**（收敛到非物理平均值），应改用生成式建模。此判据与"分工判据"（新旧工具各接管哪段）并列，收入本库选型工具箱。

## 再一条判据（2026-10-10，来自 RiCo）

**"这个输入/正则的价值由哪个指标度量？"** —— RiCo 的符号距离消融：去掉"符号"（只留纯距离）后**轨迹 RMSE 几乎不变、接触保真度翻倍变差**（ΔPTR 11.0% → 23.6%）。教训：一个设计元素的好坏必须由**它对应的指标**评估——否则会像本文之前的基线一样"在错误指标上看起来不错"。与 PhysLDM 的"混沌判据"并列收入选型工具箱。

## Personal Knowledge

Current Level: **Hard**（入口已扩：**机制层（"神经替代哪一步、保留什么接口"）现在有 [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]] 一篇可直接读**；训练/推导层仍缺）

## Learning Gap

- **概念层（可读）**：分工判据——"显式部分 vs 学习部分"在 Neuroll 里有一个具体答案（**替代积分步骤、保留经典 I/O 语义与显式物理状态**）；你的粒子机制 Easy 可支撑这一层阅读；
- **未确认层（按 2026-10-05 PKM 校正记录）**：attention、学习式仿真的训练细节与实现——**不再断言"只缺一个 attention 词汇"**（该表述已由 PKM 校正替代）；训练目标、误差累积、泛化条件都是实际缺口，按具体问题再建桥。

## Next Learning Step

1. **先读 [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]] 的机制层**（Figure 2 训练管线 + Table 1 数字）——25 分钟，用自己的话回答"它替代了解算器的哪一步、保留了什么"；
2. [[WorldParticle — Unified World Simulation of Lagrangian Particle Dynamics via Transformer|WorldParticle]] 的 Figure 1 + 项目页视频仍可作为"压缩相互作用图 = 粒子 LOD"的类比入口；
3. 训练/推导侧（扩散 / flow matching / 图网络训练）沿 [[Neural Rendering]] / 具体论文的问题缺口补，不在本概念层展开。
