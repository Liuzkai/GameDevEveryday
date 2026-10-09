---
type: paper
title: "Simulation Methods for Multiphysics Phenomena in Visual Computing"
authors: [Fabian Löschner, Stefan Rhys Jeske, José Antonio Fernández-Fernández, Jan Bender]
year: 2026
published: "2026-10-07（arXiv v1, 2610.09822；Computer Graphics Forum 45(2)，Eurographics 2026 Tutorial）"
venue: "Eurographics 2026 Tutorial / CGF 45(2)"
url: "https://arxiv.org/abs/2610.09822"
code: ""
project_page: ""
category: [physics-simulation, tutorial, pbd, mpm, sph]
importance: B+
historical_importance: 0
game_relevance: 4
production_readiness: "Industry Adopted（所覆盖的经典方法多数已在引擎/DCC 生产采用；本条为教学资料，本身不引入新方法）"
user_level: Normal
status: unread
aliases: [Multiphysics Tutorial, 多物理仿真教程, Bender Tutorial, Visual Computing Simulations]
tags: [physics-simulation, tutorial, pbd, mpm, sph]
---

# Simulation Methods for Multiphysics Phenomena in Visual Computing（Löschner et al. 2026）

> **入库 2026-10-08（Run 30）。** **RWTH Aachen University**（**Jan Bender** 组——PBD / XPBD 方法谱系的权威作者方）。Eurographics 2026 教程。
> **一句话定位**：**物理仿真方法的一站式地图与教科书式推导入门**——从能量法（Newton / VBD / Projective Dynamics）到约束法（PBD/XPBD）、拉格朗日粒子法（SPH / MLS-RKPM）、欧拉与混合法（流体 / MPM），再到多物理耦合与框架选型。**它不提出新方法，它给你"选型时的坐标系"。**

## TL;DR

- **覆盖对象**：刚体、可形变体、流体、颗粒材料，以及**它们之间的相互作用（耦合）**——"multiphysics" 的重心在**耦合策略**上；
- **写法**：每种方法给出**数学框架 + 详细推导 + 材料/耦合如何装进公式**；另附**现成软件框架列表**（"out-of-the-box multiphysics modeling"）；
- **收尾**：physics-based animation 的**新兴趋势**（机器学习类方法，近年热度上升）——正好是库内 [[Neural Physics Simulation]] 线的**经典侧参照**；
- **用法建议（推断）**：**当工具书用，不当读物逐页读**——按需查"某一类材料/交互在 2026 年的标准解法与公式"。

## 内容结构（原文目录）

```text
§2 能量法建模（Energy-Based）
    2.1 基础：优化时间积分 / Newton 法 / 全局导数与装配 / 惩罚约束 / 线搜索 /
        收敛性与非精确性 / 正定投影 / 线性系统求解 / 可微性 / 框架
    2.2 材料与现象：弹性可形变 / 接触势 / 阻尼·摩擦·塑性 / 壳与杆 /
        刚体与多体系统 / 多物理现象 / 流体与耦合
    2.3 Vertex Block Descent
    2.4 Projective Dynamics（含多物理扩展）
§3 约束法（Constraint-Based）
    3.1 方法：Position Based Dynamics（PBD）→ XPBD → 性能改进
    3.2 材料与现象：流体 / 刚体 / 连续材料 / 杆 / 颗粒材料
§4 拉格朗日粒子法（Lagrangian Point-Based）
    4.1 SPH / MLS 与 RKPM
    4.2 流体 / 刚体与摩擦接触 / 弹性与弹塑性 / 多物理系统
§5 欧拉与混合法（Eulerian & Hybrid）
    5.1 欧拉离散基础  5.2 流体  5.3 固体  5.4 多相流体
    5.5 流固耦合      5.6 物质点法（MPM：基础 / 精度稳定性性能 / 多物理 / 耦合）
§6 多物理仿真框架（软件选型）
§7 新兴趋势（含机器学习方法）
§8 结论
```

## 为什么值得入库（对库内知识图谱的作用）

1. **物理域的"地图层"**：库内物理侧此前只有散点——[[Particle Systems]]（Easy，VFX 视角）、[[Neural Physics Simulation]]（Hard，学习式仿真）、以及 [[2026-10-06-PhysLDM — Latent Diffusion for High-Fidelity Deformable Simulation|PhysLDM]] / [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]]（神经节点）。**缺的正是"经典方法全景"**——本文补上（PBD/XPBD、SPH、MPM、Projective Dynamics、VBD 每个都是可独立成节的稳定知识体）；
2. **"耦合"首次有系统章节**：多物理的核心难点（流体-固体、刚体-可形变、颗粒-连续）在 §2–§5 反复出现，§6 给框架对照——**"素材级"的选型信息**；
3. **PBD/XPBD 从源头讲**（Bender 组即该方法的开发主线之一）——**比任何二手教程更接近"官方口径"**；
4. **§7 的机器学习趋势**：把 [[Neural Physics Simulation]] 的神经方法与经典方法放在同一坐标系里——**"神经网络学的是哪个经典方法的输入输出"** 这类问题可以对照着问（与 Neuroll 的"镜像经典积分器 I/O"判据互证）。

## Key Contribution（作为教学资料的价值）

- 统一的**数学框架视角**（能量/约束/粒子/欧拉四族，及各自的离散化与求解器）
- 每种方法**可复现的推导** + 材料如何进公式 + 耦合如何做
- **框架选型表**（§6）与**趋势篇**（§7）——工程决策层信息
- ⚠️ 定位注意：这是 tutorial/course notes（教学资料），不是新研究——入库理由是**图谱价值与查阅价值**，不是"前沿突破"。

## Game Development Relevance

**4/5。物理是游戏里"能做/不能做"的硬边界；本文是边界的地图。**

- **选型坐标系（可直接用于判断与沟通）**：面对"布料/绳索/颗粒/流体/破碎"的需求，先问**族**——约束法（PBD/XPBD，游戏实时之友）↔ 能量法（更准更贵，离线/次世代）↔ 粒子法（SPH 类，流体/颗粒）↔ 欧拉/MPM（雪泥烟，体素材质）；
- **与引擎现状的对照（推断）**：UE Chaos（布料/碎裂/刚体约束）、PhysX 的实时路线本质上都是**约束法族**；MPM/SPH 系在离线特效（Houdini）与传统之上——**"实时用约束、离线用能量/欧拉"的分工**可以在本文找到系统依据；
- **对 VFX 工作的直接用处**：你熟悉的 Niagara 粒子是"**渲染侧+Dynamics 侧**"；本文覆盖的是"**连续介质侧的物理**"（布料/流体/颗粒体），是特效里"看起来物理"与"真的物理"的分界资料；
- **耦合章节**：角色-布料-环境相互作用（游戏里最常崩的环节）在 §2.2/§5.5 有系统视角。

## Unreal Engine Relevance

- Chaos Physics 的概念组（PBD 类约束、XPBD 迭代、刚体-可形变交互）与本文 §3 直接同源（推断，未逐项核对版本实现）；
- 可映射的查阅场景：**Chaos Cloth / Chaos Flesh（可形变）/ Destruction 参数理解**；Niagara Fluids（离线烘焙辅助）与 §4/§5 的对照；
- 不建议把本文当"引擎实现文档"——它是方法学地图，引擎侧细节仍需查官方文档（待核实项）。

## Relationships

### Related

- [[Neural Physics Simulation]] —— **经典侧参照**：§7 的 ML 趋势在库内的对应线；"神经方法镜像了哪个经典求解器"可直接在本文里查（与 Neuroll 的 I/O 镜像判据互证）
- [[2026-10-06-PhysLDM — Latent Diffusion for High-Fidelity Deformable Simulation|PhysLDM]] —— 可形变体学习式仿真的前沿节点（本篇覆盖其经典对应物：FEM/Neo-Hookean 族）
- [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]] —— 发丝仿真（rod 类）的学习式节点（本篇 §2.2/§3.2 覆盖 Cosserat rod 类经典方法）
- [[Particle Systems]] —— 渲染/动力学视角的粒子（Reeves 1983）与本文的"物理正确"粒子（SPH/MPM）是两个世界——**对照阅读能看清"游戏粒子"与"仿真粒子"的鸿沟在哪**
- [[Real-Time VFX Performance Budgeting]] —— 约束法的成本结构（迭代次数 × 约束数）可作为"物理预算"的补充维度（推断）

### Followed By

- 库内"物理线地图"的长期挂载点：未来经典（如 PBD 原始论文 Müller 2007、XPBD Macklin 2016）入库时，本篇是其"上下文容器"

## Personal Knowledge State

- **user_level: Normal（地图层）/ Hard（推导层）**——地图层（四族划分 / 何时用谁 / 耦合难点）可直接读；推导层（Newton 装配、SPH 核、MPM 的 P2G/G2P 数学）按需。
- **用法**：**查阅式**——不是"读完"型材料。把它当作物理向的 [[Real-Time Rendering]]（那次是渲染域地图，本篇是仿真域地图，同一种资产）。

## Learning Value

- **一条框架记忆（推断）**："**同一物理现象至少有四族解法**（能量 / 约束 / 粒子 / 欧拉-混合），每族的'贵在哪'不同——能量法贵在求解器、约束法贵在迭代、粒子法贵在邻居查询、欧拉法贵在网格与平流"。**选型先问'我买得起哪一种贵'**；
- **一条耦合判据（从结构推断）**：多物理的失败大多不在单体方法，而在**耦合界面**（流固/刚柔/颗粒-连续）——遇到"仿真崩了"，先怀疑界面而不是求解器本身。

## Notes

- **原文核对（arXiv HTML v1 目录与正文抽取）**：作者与单位（RWTH Aachen；Löschner / Jeske / Fernández-Fernández / Bender）、章节结构（§1–§8）、"多物理耦合"与"框架"章节、§7 机器学习趋势——**均为原文确认**；
- **元数据**：arXiv 2610.09822v1（07 Oct 2026）；CGF Volume 45, Issue 2（Eurographics 2026 Tutorial 卷）；**DOI/页码未获取**（待补）；
- **关联**：本批同日入库的 [[2026-10-06-PDB — Point-Based Deformation Blending for Facial Animation Retargeting|PDB]] / [[2026-10-07-DynaConTalk — Wavelet-Constrained Diffusion for Long-Form and Controllable Holistic Co-Speech 3D Motion|DynaConTalk]] 是"应用侧"，本篇是"基础侧"——**一天内从物理地基到面部/手势应用同时扩充**。
