---
type: concept
title: "Particle Systems"
user_level: Easy
aliases: [Particle System, 粒子系统, Particle Simulation for VFX, Fuzzy Object Modeling]
prerequisites: [Real-Time Rendering, GPU Architecture]
first_introduced: "1983（William T. Reeves, ACM TOG 2(2)）"
tags: [vfx, particles, simulation, rendering, user-domain, classic]
---

# Particle Systems

> 用户本人的专业领域。**本文件是本库此前结构性缺失的锚点** —— [[Real-Time VFX Performance Budgeting]] 与 [[Niagara]] 都是 Easy，但整库没有任何一篇粒子系统的历史与理论源头。
> **本文件不做教学**（Easy Policy）：只做**历史定位**、**成本法则**与**与今天引擎的对应关系**。

## Definition

一种用**大量简单图元（粒子）的集合**来表示一类"模糊物体"的表示法。每个粒子在生成时被赋予独立属性（位置、速度、尺寸、颜色、透明度、形状、寿命），随后按动力学演化、按寿命或贡献度消亡。

**它表示的是一类物体，不是一种效果**：火、烟、云、水、雾、爆炸、雨雪、毛发/草（结构化变体）都属于这个类别。

**判定是否属于粒子系统的关键**：物体**没有明确边界**、**是非确定性的**（由随机过程生成与控制）、**是随时间的动态系统**。

## Core Principle

### 三处与曲面表示的根本不同（Reeves 1983 原文）

1. **体积而非边界** —— 物体由定义其体积的粒子云表示，而非定义边界的图元集合；
2. **非静态** —— 粒子"出生 / 移动改变形态 / 死亡"；
3. **非确定性** —— 形状不被完全指定，由**随机过程**生成与改变。

### 一帧的五步循环 —— 今天是所有粒子引擎的骨架

```text
(1) 生成新粒子            (2) 赋独立属性        (3) 熄灭火亡粒子
(4) 按动力学移动          (5) 渲染存活粒子
```

**关键设计（1983 年就在，且从未变过）**：**五步骨架固定，每步的算子是任意的**。原文明确允许"把粒子运动绑到偏微分方程的解上"。**这就是今天 Niagara 的 Simulation Stage / Data Interface 的设计空间。**

### 三个成本法则（本概念最有实用价值的部分）

| 法则 | 内容 | 出处 |
|---|---|---|
| **① 数量随屏占比** | 生成率可写成 `(每单位屏幕面积粒子数) × ScreenArea`，**渲染时间随之线性变化** | Reeves 1983 §2.1 |
| **② 单位成本极低** | 一个粒子比一个多边形简单得多 → **同等算力能处理更多图元** | Reeves 1983 §1 |
| **③ 顺序无关（有条件）** | 点光源假设 + 加法混合 → **无需排序、无隐藏面、无阴影** | Reeves 1983 §2.5 |

> **⚠️ 法则 ③ 的边界必须记住**：它的成立**完全依赖加法混合**。一旦改用 Alpha 混合（半透明叠加），**排序立刻成为必须**，成本与管线复杂度都变 —— 本库 [[GS 半透明顺序依赖]]、[[GS 排序成本三乘数_TileGS]] 讨论的就是这个问题在另一种图元上的复现。

## Prerequisites

- [[Real-Time Rendering]] —— 帧缓冲、混合、光栅化
- [[GPU Architecture]] —— 今天的粒子在计算着色器上跑
- **概率与随机过程的基本概念**（"均值 + 方差"级别的即可）

## Historical Evolution

```text
1982  前人零散尝试（Wilson 的烟囱烟、Smith & Blinn 的星系、Norton 的分形粒子、Blinn 的粒子层光照）
        ↓ 都缺"方法论 + 动力学 + 随机控制"中的至少一项
★ 1983  Reeves —— Particle Systems：方法论化 + 生产验证（星际迷航 II 火墙，75 万粒子）
        ↓
1985  Reeves & Blau —— 结构化粒子系统：整条轨迹当静态形状 → 毛发/毛皮/草
        ↓                                    （[[Hair Rendering]] 资产表示的最早祖先）
1987  Reynolds —— Boids：粒子之上加"外部状态交互" → 群体行为（另一分枝）
        ↓
1990  Sims —— 数据并行计算做粒子动画（GPU 粒子的概念前身）
        ↓
2006+ GPGPU / 计算着色器 → 粒子数从千级跨到百万级
        ↓
2016+ 引擎级系统：UE Niagara / Unity VFX Graph / Houdini 侧离线管线
        ↓
→ 产业侧出现新问题：粒子不再受"算力"限制，而受"预算"限制
        ↓
2026  [[Real-Time VFX Performance Budgeting]] —— 用 SABC 分级 × 五档画质把粒子数变成可管理的预算
```

## Important Papers

| 论文 | 年份 | 角色 |
|---|---|---|
| [[Reeves — Particle Systems (1983)]] | 1983 | **方法论源头**；五步循环、屏占比 LOD、层级结构、随机数检查点 |
| **Reeves & Blau 1985** | 1985 | 结构化粒子系统（整条轨迹当静态形状）→ 草/毛/发。（**未入库**，与 [[Hair Rendering]] 相关）|
| **Reynolds 1987（Boids）** | 1987 | 粒子 → 群体行为（**未入库**；与用户的 UE5Steering 方向相关）|
| **Müller et al. 2003（SPH）** | 2003 | 粒子 + 流体力学 → 有表面的水（**未入库**）|
| **Sims 1990** | 1990 | 数据并行粒子动画（GPU 粒子的概念前身；**未入库**）|
| [[2026-09-09-WorldParticle — Unified World Simulation of Lagrangian Particle Dynamics via Transformer]] | 2026 | **前沿对照**：用单个 Transformer 统一六类粒子动力学 —— **"显式 Predictor 管外力 + 学习 Corrector 管相互作用"** |
| [[LightOpt — Lights Optimization for Real-Time Rendering]] | 2026 | 不直接相关，但同属"把经验上限变成可推导值"这一诉求 |

## Related Concepts

- [[Real-Time VFX Performance Budgeting]] —— **本概念的工程化终点**（屏占比 LOD + 发射器数 + 寿命 → 预算矩阵）
- [[Niagara]] —— 今天的引擎实现
- [[Participating Media]] —— **本概念的"下一步"**：Reeves 1983 §5 明确列出两个做不到的点（**粒子被打亮**、**云的自阴影**），正是体积渲染要解决的
- [[Gaussian Splatting]] —— **另一种"云状图元"**：高斯椭球同样是无序、半透明、按屏占比膨胀的图元。**两者的排序/密度/分档问题同构**，可互相对照
- [[Scalability and Quality Tiers]] —— 粒子数是五档画质最敏感的一个维度
- [[Hair Rendering]] —— 结构化粒子（1985）的现代形态
- [[Neural Physics Simulation]] —— 学习型粒子动力学（对照 WorldParticle）
- [[AAA Real-Time VFX]] —— 应用落点

## Technologies

- **CPU 粒子 / GPU 粒子 / 混合方案**（见 [[Niagara]] 的三条路径与实测数据）
- **Simulation Stage 与数据接口**（多阶段模拟）
- **Niagara Fluids**（Eulerian 网格流体 —— **注意：这是另一条表示路线，不是粒子系统**）
- **顺序无关透明（OIT）** —— 用来消除"法条 ③"的加法混合依赖

## Game Applications

- 技能特效（**用户当前工作**：NGR 淬炼系统的技能 VFX）
- 环境氛围（雨雪雾尘、体积光中的尘埃）
- 破坏与碎裂（与 [[Physics-based Character Animation]] 的接触/碰撞交汇处）
- 群体单位（Boids 分支）

## Personal Knowledge

Current Level: **Easy**

**按本库 9-20 建立的分层模板拆两层**：

| 层 | 内容 | 状态 |
|---|---|---|
| **操作层** | 发射器/模块/渲染器/混合模式/发射率曲线/寿命 | **Easy —— 你的日常工作，不复述、不推送** |
| **成本模型层** | **① 屏占比 × 密度是比绝对上限更好的分档变量 ② 发射器数是"树节点数" ③ 随机数只在 Spawn 消费换确定性 ④ 加法混合是"免排序"的唯一前提** | **Normal —— 值得动手验证的四条** |

## Learning Gap

**不在知识，在形式化**：

- 你的 3000 / 1200 / 400 / 100 是**绝对上限**（统计的），**不是屏占比密度**（几何的）；
- **绝对上限对性能的预测性是统计的**（"通常够用"），**屏占比密度是几何的**（面积变化 → 数量按比例变化，性能可预期）；
- 极端镜头（贴脸、广角、多技能叠加）下，绝对上限会失效。

## Next Step

1. **（30 分钟，可做）** 在 NGR 里算一次"**每千像素粒子数**"，对比现有绝对上限 —— 看两者在你最重的镜头下是否指向同一个量级。**如果差距很大，说明绝对上限没有抓住真正的成本变量。**
2. **（可选）** 检查是否存在发射器嵌套；若有，**把"≤12"重新表述为"树总节点数 ≤12"**；
3. **（可选）** 确认随机数是否只在 Spawn 阶段被消费。若不是，**联机同步与回放复现都会不稳** —— 这是 1983 年就写明的架构不变式；
4. **（判据）** 能说出"加法混合是免排序的唯一前提"，即说明粒子渲染的成本模型已经吃透。

---

相关：[[Reeves — Particle Systems (1983)]] · [[Niagara]] · [[Real-Time VFX Performance Budgeting]] · [[Participating Media]] · [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]
