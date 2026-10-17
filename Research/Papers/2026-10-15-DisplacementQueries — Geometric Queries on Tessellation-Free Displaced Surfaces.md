---
type: paper
title: "Geometric Queries on Tessellation-Free Displaced Surfaces"
authors: [Bing Xu, Yang Zhou, Brian Budge]
year: 2026
published: 2026-10-15
venue: "arXiv 2610.21584 (cs.GR)"
url: "https://arxiv.org/abs/2610.21584"
code: ""
project_page: ""
category: [rendering, geometry, ray-tracing, displacement, sampling]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: Research
user_level: Normal
status: unread
aliases: [DisplacementQueries]
---

# 2026-10-15-DisplacementQueries — Geometric Queries on Tessellation-Free Displaced Surfaces

## TL;DR

位移贴图的**"无细分"路线补全查询能力**：此前无细分方法（Ogaki 2023 / Thonat 2021-23）只会"射线求交"这一种查询；本文给位移曲面装上**一阶纹理空间层级**（每个节点 = 局部仿射高度平面 + 保守残差界 r），让**表面采样（面积估计）、最近点查询（surface enclosure）、specular 光路搜索（斜率界）**全部免细分可用。代价账：56 个位移曲面 vs FCPW 预细分，查询内存几何均值省 **≥55.4×**（20/56 曲面在 5000 万三角形预算内连 10⁻⁴ 距离误差都达不到）；但预细分在达标的 36 个曲面上查询快 **7.48×**——**内存换速度的取舍被量化了**。一阶层级本身比标量 min-max 快 **3.06×**（消融）。

## Problem

- 位移贴图本该省内存（细节存纹理），但主流用法要**预细分**成稠密网格才能参与渲染/物理——内存优势被细分化链吃掉（Nanite displacement 也是"受控细分"路线）；
- 已有无细分方案只支持**射线求交**；更广的几何查询需要三样标量 min-max 给不出的东西：**surface enclosure**（最近点/距离）、**面积估计**（表面采样）、**法线/斜率界**（specular manifold 搜索）。

## Historical Context

```text
1978  Blinn 位移贴图思想（细节进纹理）
2001  Cook et al. 微多边形/REYES（细分管线正统）
2021  Thonat et al. 无细分射线求交（min-max 高度区间）
2023  Ogaki 2023 无细分 BVH 求交
2026  ★本文：一阶（仿射）层级——求交之外的查询族补齐
      （同期游戏侧：Nanite displacement = 预细分/运行时细分管线）
```

## Previous Work

- **预细分 + BVH**（FCPW、Embreed 等）：查询最快，内存与构建时间爆炸；
- **min-max 无细分求交**（Thonat 2021/2023、Ogaki 2023）：只记录 [h_min, h_max]，丢掉了"高度如何随位置变化"——求交够用，采样/最近点/specular 搜索不够；
- **SMS**（specular manifold sampling, Zeltner 2020）：specular 路径搜索的离线基线——本文用斜率界做 NEE 式搜索与它对比。

## Core Idea

**每个层级节点存"局部平面 + 残差"**（图 2）：

- 仿射高度平面 Ĥ(Δu,Δv) = h_c + g_u·Δu + g_v·Δv（中心高度 + 两方向斜率）；
- 保守残差 r ⊇ 区域内真实高度与平面的偏差 → **平面 ± r 构成 surface enclosure**（求交与距离搜索的包络）；
- 平面斜率 g_u, g_v → **面积估计**（采样的 PDF 分母）+ **斜率界**（specular 光路搜索的可行域）。

min-max 只是一个"零阶盒子"；本文升到"一阶斜面 + 残差带"——空间高度变化信息被保留。

## Technical Approach

- 层级构建：纹理空间自顶向下拟合平面 + 计算残差界（构建期一次）；节点额外存平均发光亮度（发光体采样用）；
- 按查询定制遍历：求交沿用 enclosure 剪枝；最近点查询用 enclosure 做距离下界剪枝；采样用面积权重自顶向下选节点；specular NEE 用斜率界 + 距离条件剪枝（接收方感知权重 R(x) 承自 Conty Estevez & Kulla 2018 的基于面积的发光体采样传统）；
- 三个示范应用（图 1）：**位移水面焦散**（直接光追位移曲面）、**位移发光体采样**（火山/珊瑚——发射分布由位移曲面上的 screened Poisson 解给出）、**表面 PDE 求解器**（Projected Walk on Spheres 用最近点查询）。

## Key Contribution

1. **表示层**：一阶（仿射+残差）高度层级——"位移曲面作为通用图形基元"的查询底座；
2. **算法层**：四类查询（求交/最近点/采样/specular 搜索）的遍历-剪枝-采样配对算法；
3. **实证层**：内存-速度取舍的量化账本（55.4× vs 7.48×；3.06× 消融）。

## Why It Works

- 残差界是**保守**的（宁可过大不可漏）——保证剪枝正确性；平面拟合吸收一阶变化，残差只剩高阶小量 → 包络紧；
- 面积/斜率是高度函数的**一阶导数性质**，仿射模型天然携带——这是标量区间结构上给不出的信息。

## Limitations

- **查询速度输给预细分**（FCPW 快 7.48×，CPU 对比）：对速度敏感、贴图分辨率可控的场景，预细分仍是正解；
- 构建期要为每个节点拟合平面+残差（资产管线成本）；残差大的高频位移（噪声/锐利细节）包络变松、剪枝退化；
- 作者三人（NVIDIA + 独立研究员），无开源代码/项目页（截至 v1）；演示在 CPU（FCPW 对比口径），GPU 落地路径未给。

## Game Development Relevance

- **引擎位移路线的分叉点**：Nanite/虚幻代表"细分+硬件求交"，本文代表"免细分+软件查询"——后者是位移几何参与**物理查询/光照采样/PDE** 的低成本路径（角色站上位移地形的高度查询、布料与位移面的最近距离）；
- **VFX 直接相关**：位移发光体（熔岩/能量场/酸液湖）在 Niagara/路径追踪管线下需要"按发光强度采样表面"——本文的发射加权层级就是现成配方；
- **预算视角**：位移几何的内存账从"细分后三角形数"变成"贴图分辨率 × 层级节点开销"——SABC 资产预算表可加一行"位移贴图 + 查询层级"口径。

## Unreal Engine Relevance

- Nanite displacement（5.5+）走的是预细分路线；本文结果可作为"何时改走免细分"的对照证据（内存受限平台 / 需要非射线查询时）；
- UE 侧表面 PDE / Walk-on-Spheres 尚无内建能力——研究侧跟进价值大于短期落地。

## Technology Evolution

```text
位移贴图 = 只能"看"（光栅化/求交）
  → 无细分求交（2021-23）：内存省，但仍只会"被射线打到"
  → ★本文（2026）：采样/最近点/specular 搜索——位移曲面成为"可查询基元"
  → （可期）位移几何参与物理/流水线查询的引擎内建
```

## Relationships

### Based On

- min-max 无细分求交（Thonat 2021/2023、Ogaki 2023）——本文的对照组与起点
- FCPW（Sawhney 2021）——预细分查询基线

### Related

- Walk on Spheres 系列 — 表面 PDE 应用端（本库 2026-10-15 批次已记名 Walk-on-Ambient-Patch）
- Conty Estevez & Kulla 2018 — 发光体采样的接收方感知权重出处（pbrt-v4 系）

### Contrasts

- Nanite displacement（UE 5.5+）— 预细分路线代表；本文量化了两条路线的内存/速度账

## Personal Knowledge State

- **Normal（结论层）**：一阶层级三要素（平面/残差/斜率）与三类新查询的对应关系——几何查询接口层的知识，与渲染预算线正交但互补。

## Learning Value

- "表示决定可用查询"是引擎设计的通用命题：min-max（零阶）→ 仿射+残差（一阶）每升一阶，解锁一类查询、多一份内存——与 GLoD-LVSM（锚点数=预算输入）、Budgeted-GS（LoD 因子化）同属"**预算决定表示精度**"母题。

## Visualization

![[Research/Files/Pictures/Xu 2026 — 图1 无细分位移曲面支撑的几何查询全景.png]]

**读图**：三个示范应用同框——整个水下场景无一处细分：水面位移直接光追出焦散（左上）；
火山与珊瑚是位移发光体（采样自层级），珊瑚的发光分布来自位移曲面上的 screened Poisson 解；
右上小图为基网格（base mesh）——所有细节都只在位移贴图里。（图 1，Xu et al. 2026）

![[Research/Files/Pictures/Xu 2026 — 图2 MinMax与一阶仿射高度模型.png]]

**读图**：核心表示对比——左：传统 min-max 只存 [h_min, h_max] 区间（粉色盒子），
"高度如何随 u 变化"全部丢弃；右：本文一阶模型 = 中心高度 h_c + 斜率 g_u=Δh/Δu 的仿射平面
Ĥ（虚线），加保守残差 r（绿色带）包住真实高度（蓝线）——平面给斜率/面积，残差给包络。
（图 2，Xu et al. 2026）

## Notes

- 提交于 2026-10-15（美东周四），属 10-16 周五公告批次；作者署名 NVIDIA ×1 + 独立研究员 ×2，无代码/项目页——**下轮核查代码放出**。
- 关键数字备查：55.4×（内存 GM）/ 7.48×（FCPW 速度 GM）/ 3.06×（一阶 vs min-max 消融）/ 36/56（5000 万三角形预算内达标数）。
