---
type: moc
title: "Game Development Research — Index"
created: 2026-09-07
---

# Game Development Research — Index

全球游戏研发技术情报系统 + 个人认知知识库。

## 入口

- **今日**：[[2026-09-16]]（昨日：[[2026-09-15]]）
- **本周综合**：[[2026-W37]]
- **本月雷达**：[[2026-09]]
- **认知模型**：[[Personal Knowledge Model]] ⚠️ 推断值，待校正

## Papers（32）

| 论文                                                                            | 分级  | 用户水平        | 与你相关度        |
| ----------------------------------------------------------------------------- | --- | ----------- | ------------ |
| [[Learned Motion Matching (Holden 2020)]]                                     | S（基准） | Normal→Easy | **最高（MM 桥接材料）** |
| [[LightOpt — Lights Optimization for Real-Time Rendering]]                    | S   | Normal      | **最高**       |
| [[DLSS 5 — Generative Neural Rendering]]                                      | S   | Normal      | 高            |
| [[MotionBricks — Scalable Real-Time Motions]]                                 | S   | Hard        | 中（时序锚点）      |
| [[Magpie — Real-Time World Renderer for Interactive Games]]                   | A   | Normal（接口层） | 高（战略性）       |
| [[STyMo — Fast and Controllable Few-Shot Motion Style Transfer]]              | A   | Normal      | 高（方法论+生产）    |
| [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]] | A   | Normal      | 中高（方法论）      |
| [[Lightweight Attention-based Indirect Illumination (AMD)]]                   | A   | Hard        | 中            |
| [[Compact Neural Appearance Models for Efficient Gaussian Splatting]]         | B+  | Normal      | 中高（预算语言）     |
| [[Inverse Rendering for Modeling with Line Primitives]]                       | A   | Hard        | 低（Watchlist） |
| [[UniMate — One Unified Model to Animate Diverse Skeletons]]                  | A   | Hard        | 低（Watchlist） |
| [[LLM-Guided RL for Adaptive NPC Behavior]]                                   | B   | Normal      | 低            |
| [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]    | B   | Hard        | 低            |
| [[GradRig — Differentiable Weights for Skinned Gaussian Splat Deformation]]   | B   | Normal      | 低（Watchlist） |
| [[WorldParticle — Unified World Simulation of Lagrangian Particle Dynamics via Transformer]] | A- | Hard（桥极短） | 中（概念启发） |
| [[FlexMoGen — Flexible Motion Generation from Language and Style References]]                  | A   | Normal      | **高（MM 瓶颈之桥）** |
| [[From Splats to Silicon — Rethinking Computational Efficiency of 3DGS]]                      | B+  | Normal      | 高（预算方法论）     |
| [[Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement]]                 | A-  | Hard        | 中（管线痛点）      |
| [[RelightFormer — Feed-forward Generative Transformer for Multiview Object Relighting]]       | B+  | Hard        | 中（灯光线对照）     |
| [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]]    | A-  | Normal（系统侧） | 中高（系统方法论+新阵营锚点） |
| [[SceneHI — High-Resolution 3D-Consistent Scene Texturing with Controllable Illumination]]   | B+  | Hard（概念可读） | 中（生成式资产烘焙对照）   |
| [[MOONWALK — Intent-Evidence-Action Alignment for Animation VFX Review]]                     | B   | Easy（流程语言） | 中（评审工作流模板）     |
| [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]                         | S（经典） | Normal（研读中） | **高（当前学习焦点）** |
| [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]] | S（经典） | Normal | **高（PBR 环境光半边 / 移动端 GI 根源）** |
| [[2026-09-14-Gaussian Light Transport]]                                                      | A-  | Hard（概念桥短） | 高（GI 显式路线对照）     |
| [[2026-09-14-Learning Realistic Athletic Sprinting Without Demonstrations]]                  | A-  | Hard        | 中（零示范范式）        |
| [[Kajiya — The Rendering Equation (1986)]]                                                    | S（经典） | Normal | **高（渲染谱系最大锚点 / PBR 容器侧）** |
| [[2026-09-15-Grid-Free Monte Carlo for Time-Dependent Diffusion]]                             | A-  | Hard（取两个认知点即可） | 中（方法论对偶样本）     |
| [[2026-09-15-UniMo — Unifying Human and Animal Motion Generation]]                            | B+  | Hard（取一句话即可） | 中低（数据价值 > 方法价值） |
| [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]                   | S（经典） | Normal（研读中） | **最高（GGX 论文：你用的 D 与 G 出自这里）** |
| [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]]                       | A-  | Hard（D/G 部分 Normal 可读） | 中高（Smith 假设 → height-field 极限） |
| [[2026-09-16-ESG — Generating Physically Consistent Dynamic 3D Scenes from Text]]             | B+  | Normal | 中（架构模板 > 技术） |

## Concepts（23）

**基础锚点**
- [[Real-Time Rendering]]
- [[Global Illumination]]
- [[Rendering Equation]] ★ 2026-09-15 入库（谱系最大锚点补齐）
- [[Microfacet Theory]] ★ 2026-09-16 入库（当前学习目标节点：D·G·F）
- [[Participating Media]] ★ 2026-09-16 入库（VFX/OverDraw Easy 域 ↔ 理论的最大缺口）

**你的领域（Easy）**
- [[Real-Time VFX Performance Budgeting]]
- [[Niagara]]
- [[Scalability and Quality Tiers]]
- [[Gaussian Splatting]] ✅ 2026-09-11 标 Easy

**Normal 学习区**
- [[Motion Matching]] ★ 关键瓶颈
- [[BRDF]] ★ 主动研读中（2026-09-14 信号）
- [[Physically Based Rendering]] ★ 主动研读中（与 BRDF 同线）
- [[Tile-Based Rendering]]
- [[GPU-Driven Rendering]]
- [[Neural Upscaling and Frame Generation]]（实为 Technology）
- [[Temporal Stability and Artistic Intent]]

**Hard 待建桥**
- [[Neural Rendering]]
- [[Generative Rendering]]
- [[Differentiable Rendering]] ★ 第二个瓶颈
- [[Neural Global Illumination]]
- [[Neural Animation]]
- [[Motion Generation]]
- [[Inverse Rendering]]
- [[Neural Physics Simulation]]（★ 全库桥最短的 Hard）
- [[Physics-based Character Animation]]（桥：排在 Motion Matching 之后）
- [[World Models for Games]]（Normal 读法：只学接口）

## Technologies（4）

- [[Neural Upscaling and Frame Generation]]（Adopt / Normal）
- [[Real-Time Global Illumination]]（Adopt / Normal）
- [[Real-Time Generative Motion]]（Assess / Hard）
- [[Arm Neural Graphics]]（Assess / Normal）

## Applications（2）

- [[AAA Real-Time VFX]] — 你的领域，含外部研究映射表与 4 个 Open Question
- [[Open World Character Animation]]

## Learning Paths（3）

- [[Learning Path — Gaussian Splatting]] — ✅ 完成（2026-09-11 标 Easy），基础推送已停
    - 产出笔记：[[GS 图解 1 — 协方差与椭球：高斯的形状说明书]] · [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制]]
- [[Learning Path — Differentiable Rendering]] — 目标是读懂 LightOpt
- [[Learning Path — Neural Rendering]] — 从你已懂的 TAA 出发

## 目录结构

```
Research/
    Papers/       论文笔记（一论文一文件，同一研究的不同版本合并）
    Concepts/     概念（"这个知识是什么"）
    Technologies/ 技术（"如何真正可用"）
    Applications/ 应用（游戏工业落点）
    Engines/      引擎与生产
    Daily/        每日研究
    Weekly/       每周综合
    Monthly/      每月总结
    Learning/     学习路径
    Radar/        技术雷达
```

## 使用约定

1. **原子化**：一个文件 = 一个稳定知识实体
2. **链接优先于复制**：发现新知识先查是否已有对应笔记
3. **frontmatter 的 `user_level` 是机器判断的唯一来源**，普通 tag 只是给人看的
4. **Easy 不重复推送**，除非该领域出现重大突破
5. **Hard 不强行推送**，先找 Normal 桥

---

最后更新：2026-09-16（Run #8：+3 Papers 含经典 Walter 2007「GGX 论文」、+2 Concepts（Microfacet Theory / Participating Media）、+1 图解（Microfacet D·G·F）、Daily、Index/BRDF/PBR/Arm/Real-Time GI 五处编辑）
