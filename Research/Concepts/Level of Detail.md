---
type: concept
title: "Level of Detail"
user_level: "Easy（机制层）/ Normal（分配判据层）"
aliases: [LOD, Level-of-Detail, 细节层级, 多级细节, 细节层次]
prerequisites: ["[[Real-Time Rendering]]", "[[GPU-Driven Rendering]]"]
first_introduced: "1976（Clark，CACM 19(10)）"
tags: [lod, scalability, rendering, budget, performance, geometry]
---

# Level of Detail

## Definition

**同一份内容准备多份表示，再按"这一帧真正需要多少"决定用哪一份、用多少。**

三个部件必须分开看，混在一起谈就会得出错误结论：

| 部件 | 回答的问题 | 出问题时的症状 |
|---|---|---|
| **表示**（representations） | 多份从哪来？离线烘焙 / 运行时生成 | 内存爆、构建慢、跳变 |
| **判据**（criterion） | 按什么分配？距离 / 屏幕占比 / 屏幕空间误差 | 该细的地方粗、该粗的地方细 |
| **调节**（regulation） | 分配结果怎么稳定？ | 帧率随视角剧烈波动、pop |

> 本库此前 99 个文件提到 LOD，但没有一页讲它——因为有 3 篇 GS 的 LOD 论文（[[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]] / [[2026-10-21-Beche — KoalaGS Complexity-Aware Continuous Level of Detail for Gaussian Splatting|KoalaGS]] / [[2026-10-14-GLoD-LVSM — Learning Geometry-Grounded Continuous-LoD Scene Representations for Large View Synthesis Models|GLoD-LVSM]]）和一篇起源论文（[[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)|Clark 1976]]），中间那一段一直是空的。今天由 [[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes|Hoppe 1997]] 补上。

## Core Principle

**LOD 不是"降档"。** 这两件事经常被写成同一件事，但它们的决定频率差三个数量级：

| | 档位（[[Scalability and Quality Tiers]]） | LOD |
|---|---|---|
| 决定频率 | 一次（进关卡 / 改设置 / 换机型） | **每帧**（每物体 / 每顶点 / 每样本） |
| 决定对象 | 全局参数集 | 单个物体、顶点、簇、乃至单次采样 |
| 分配依据 | 硬件能力 + 目标帧率 | **这一帧它会错多少像素** |
| 失控后果 | 整体掉帧 | 局部跳变、抖动、pop |

一句话区分：**档位决定"这一局跑在哪条曲线上"，LOD 决定"这一帧把配额花在画面的哪一块"。**

## 三条实现路线

| 路线 | 表示形态 | 判据 | 主要代价 | 代表 |
|---|---|---|---|---|
| **离散 LOD 链** | K 份烘焙好的网格 / mip 层级 | 距离、屏幕占比、固定阈值表 | 跳变；K 是硬上限；切换要过渡 | 传统游戏 LOD、纹理 mip、HLOD |
| **连续选择性细化** | 一个渐进网格（PM）+ 一串 vsplit | **屏幕空间几何误差 τ** | 每帧决策开销（14% 帧时间）+ **必须自建控制器** | [[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes\|Hoppe 1997]] |
| **虚拟几何 / 采样时生成** | 簇层级；或根本不存在，采样时才产生 | 每簇误差界；**采样率本身** | 需要新硬件路径与新管线 | Nanite 类虚拟几何；纹理上采样（[[2026-11-05-Fujieda — Patch-Based Generalized Neural Texture Upscaling\|Fujieda 2026]]） |

三条路线不是替代关系，**今天同一个引擎里三条并存**：几何走虚拟几何、纹理走 mip、角色走离散 LOD 链。

> ⚠️ **这张表的第一列（"表示"）在本页是被预设的，不是被解释的。** "K 份烘焙好的网格"从哪来、由谁保证"这 K 份都保住了轮廓和 UV"——那是 [[Mesh Simplification]] 的问题（2026-11-08 建页）。**本页只管"每一帧挑哪份"，那一页管"这些份怎么造出来"。**

## 分配依据的四十年演化（本页主线）

```text
1976  Clark：按【屏幕占比】——物体在屏幕上占多少面积，就给多少细节
        · 判据是"几何代理量"（面积），与最终误差无直接关系
        · 附带两条今天仍在用的：中心加权（foveation 先声）、运动自适应
  ↓
1996  Hoppe：Progressive Meshes —— 一个网格 = 一串 vsplit（【表示】成形）
  ↓
1997  ★ 分岔成两半（同年、同问题、互补）
        · Garland–Heckbert：误差 = Σ(点到平面)² = vᵀQv，Q 可加
          ⇒ 决定【先塌哪一条边】（排序 / 造表示）
        · Hoppe：误差 = 屏幕上会错多少像素
          ⇒ 决定【停在哪一档】（判据 / 每帧分配）
  ↓
1997  Hoppe：按【屏幕空间几何误差 τ】——偏差空间投影到屏幕，超过 τ 才细分
        · 判据从"它多大"变成"这一像素会错多少"
        · 关键性质：误差可加、可阈值、可逐顶点求值（230 cycles/次）
  ↓
2000s HLOD：把误差按【簇】聚合——一个簇一个误差界，不再逐物体
  ↓
2020s 虚拟几何：每【簇】误差界 + GPU 侧剔除，CPU 不再参与逐个决策
  ↓
2026  纹理上采样：按【单次采样】决定——连"这一层 mip 存不存在"都不预设，
        采样时现算。判据变成"这一次采样会看到多少 texel"
```

> **这条线的方向是单向的：判据的单位越来越小、越来越贴近消费点。** 从"物体"（1976）到"顶点"（1997）到"簇"（2000s）到"单次采样"（2026）。每一次缩小都换来更准的分配，也换来更高的决策频率与更重的控制问题。

## τ 不是常量，是控制变量（1997 的工程核心，最容易被漏掉的一节）

Hoppe 的标题写的是 "frame rate"，实现里调的却是**面数**：

$$\tau_t=\tau_{t-1}\cdot\frac{|F_{t-1}|}{m}$$

其中 $m$ 是目标面数。作者给的理由是原话级的：**帧率对操作系统抖动太敏感，直接调它不稳，应该用一个更慢的外层控制器去调 $m$**。另一条硬约束：$\tau$ 不许低于 0.5 像素。

**这条对今天所有"按需求分配"的系统都成立：**

1. **判据（τ）给出的是"要不要"**，配额（$m$）给出的是"多少"——两者之间必须有一个控制器；
2. **没有控制器，配额会随视角剧烈波动**（Hoppe 图 9：固定 τ 时面数在 1 千到 200 万之间摆）；
3. **控制器要控"稳定的量"，不是"想优化的量"**——想优化帧率，就控面数。

> 这条判据今天在三个地方同时出现：Hoppe 控面数、REACT 用 defiance buffer 给可靠度做迟滞（[[2026-11-05-de Clermont Gallerande — REACT Real-Time Completion of Skeletal Motion Signals|REACT]]）、以及纹理上采样那篇**完全没做控制器**（全文无运行时性能数据）。**第三种不是遗漏，是这类工作最常见的形态。**

## 判据的质量：误差 vs 代理量

同一个"该多细"的问题，用代理量（面积/距离/面数）和用误差（像素）差在哪：

| | 代理量（面积、距离） | 误差（屏幕空间像素） |
|---|---|---|
| 依赖视角方向 | 否 | **是**（偏差空间在主轴上投影才大） |
| 能否逐顶点求值 | 能 | 能（Hoppe 230 cycles） |
| 能否跨物体相加 | 否 | **是**——这是做成全局预算的前提 |
| 会不会在轮廓处失手 | 会（轮廓处面积小但误差大） | **不会**（Hoppe 图 13：球体细化集中在轮廓） |

**"能不能相加"是分水岭。** 只有误差可以跨物体求和，所以只有误差能拿来做"总预算"；面积和距离只能逐物体设阈值。

> **2026-11-08 修正（重要）："可加"与"可比"是两件事，本页此前把它们混成了一句。**
>
> 读 [[Garland 1997 — Surface Simplification Using Quadric Error Metrics|Garland–Heckbert 1997]] 之后必须补这一刀：QEM 的 $v^\top Qv$ 是**世界空间平方距离和**，它**可加**（同一物体的历史平面能一路累加进一个 4×4 矩阵），但**跨物体不可比**——一栋楼允许 2 cm 误差与一把刀允许 2 cm 误差，在画面上差了两个数量级。
>
> 所以严格的分层是：
>
> | 层 | 量 | 可加 | 可比 |
> |---|---|---|---|
> | 造表示（离线） | 世界空间平方距离（QEM） | ✅ 同一物体内 | ❌ 跨物体 / 跨距离 |
> | 发配额（每帧） | 屏幕空间像素误差（τ） | ✅ | ✅ |
> | 粗分配（Clark 式） | 屏幕占比 / 距离 | ❌ | ⚠️ 只按经验 |
>
> **"可加"是能塞进堆里、能在任意时刻停下来的条件；"可比"是能做总预算的条件。** 前者 1997 年就有了，后者要等到把误差投影到屏幕。**这也解释了为什么自动 LOD 的 QEM 误差不能直接抄进预算表——它缺的是"投影"这一步，不是"加法"。**

## 可合并性：LOD 链的第三个前提（2026-11-09 补）

本节标题里的"表示从哪来"这一栏，一直隐含一个假设：**更粗的那一级能造出来**。今天才看清这是个**可检验的条件**，不是天经地义。

LOD 链实际上要三个前提，前两个已被反复讨论，第三个此前从未被单独拎出来：

| 前提 | 问什么 | 谁给的 | 缺了会怎样 |
|---|---|---|---|
| **① 误差可加** | 能不能排优先级、能不能随时停 | [[Garland 1997 — Surface Simplification Using Quadric Error Metrics\|Garland–Heckbert 1997]]（二次型系数） | 只能"跑完为止"，不能按预算截断 |
| **② 误差可比** | 能不能做跨物体的总预算 | [[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes\|Hoppe 1997]]（投到屏幕像素） | 只能逐物体设阈值 |
| **③ 合并可逆** | **能不能从合并前的统计量反解出合并后的参数** | [[2026-11-09-Franke — Hierarchical Incremental Beta Splatting for Online Reconstruction\|HIBS 2026]]（$\kappa\leftrightarrow\beta$ 双射） | 只能"平均"，而平均会把形状糊掉 |

**③ 的判据很干脆：合并时能算的是"矩"（质量、协方差、峰度），能要的是"参数"（位置、形状、不透明度）。如果"矩 → 参数"存在且唯一，合并就是"算矩 → 反解"，无损；如果不存在（多对一、或不可解），就只能近似，误差无处安放。**

三个例子，正好覆盖三种情况：

| 表示 | 合并时能算什么 | 参数能反解吗 | 结果 |
|---|---|---|---|
| 网格（QEM） | 二次型系数（可加） | 位置 ✅（解 3×3）／**属性 ❌**（平均不可逆） | 几何保得住、**属性只能拍权重** |
| 高斯 | 质量 / 协方差 | ✅ 全可解（且 KL 与矩匹配等价） | 合并干净 |
| **beta 核** | 质量 / 协方差 / **峰度** | ✅ **$\beta=\frac{15}{15-\kappa}-3.5$** | 合并保形（图 5 右列） |

> **反例最值钱**：[[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] 的属性合并不满足 ③（式 10 是不透明度加权平均），所以它加**色差罚项**去调小汇率——但那是"价格"，不是"能力"。它真正绕开 ③ 的办法是**让属性不住在要被合并的元素上**（每面纹理，拓扑塌了纹理重新长）。**这不是解决不可逆，是取消合并。**
>
> **所以"解耦"与"可合并"是同一个问题的两条出口**：要么别合并（把属性搬走），要么让合并可逆（给表示配一个可逆签名）。**两条都不走，就只剩平均 + 罚项。**

## Important Papers

| 论文 | 角色 |
|---|---|
| [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)\|Clark 1976]] | 起源：层级 + 按屏幕占比变细节 + working set |
| Hoppe 1996 *Progressive Meshes* | 表示：一个网格 = 一串 vsplit，连续分辨率 |
| [[Garland 1997 — Surface Simplification Using Quadric Error Metrics\|Garland 1997]] | **造表示**：二次误差（世界空间平方距离）+ 塌陷序列；**"误差可加"的最早一版** |
| [[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes\|Hoppe 1997]] | **判据**：屏幕空间几何误差 + 增量细化 + 控制器 |
| [[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD\|Budgeted-GS 2026]] | GS 线：误差 ∝ N^(−1/2)，把 LOD 写成预算曲线 |
| [[2026-10-21-Beche — KoalaGS Complexity-Aware Continuous Level of Detail for Gaussian Splatting\|KoalaGS 2026]] | GS 线：复杂度感知的连续 LOD |
| [[2026-11-05-Fujieda — Patch-Based Generalized Neural Texture Upscaling\|Fujieda 2026]] | 纹理线：把"层级存不存在"也交给采样时刻 |
| [[2026-11-09-Franke — Hierarchical Incremental Beta Splatting for Online Reconstruction\|HIBS 2026]] | **合并能力**：闭式矩匹配 + 峰度反解 ⇒ 父级保形；紧支撑核上 KL 不适用 |
| [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals\|T2-Splat 2026]] | **合并的绕道**：属性搬出顶点，拓扑自由塌陷 |

## Related Concepts

- **[[Mesh Simplification]]** —— **上游**：本页的"表示"那一栏由它供给（K 份怎么造、属性与边界怎么保）
- **[[Scalability and Quality Tiers]]** —— 全局档位；与本页是"粗/细两个时间尺度"的分工
- **[[Budget Units]]** —— LOD 是"把配额单位从资产换到消费点"的最大一支
- **[[Streaming]]** —— working set 的另一半：LOD 决定"画多少"，streaming 决定"驻留多少"
- **[[GPU-Driven Rendering]]** —— 判据逐簇求值之后，决策权从 CPU 搬到 GPU
- **[[Overdraw]]** —— 同一件事在像素维度的镜像：按覆盖分配，而不是按物体分配
- **[[Gaussian Splatting]]** —— 2026 年 LOD 最活跃的战场（三个节点）

## Technologies

- **虚拟几何（Nanite 类）**：簇层级 + 每簇误差界 + GPU 剔除；把 1997 的逐顶点决策换成逐簇，换来的是"纯 GPU 路径"与"簇大小"这个新旋钮。
- **虚拟纹理 / mip**：纹理侧的经典 LOD；判据是屏幕空间 texel 密度。
- **纹理上采样（2026）**：把 mip 的"预存层级"也去掉，改为采样时前向一次小 MLP。

## Game Applications

1. **关卡预算表应该按"每帧分配"写，不是按"每档总量"写。** 五维表现在管的是上限（≤3000 粒子 / ≤N 盏灯），LOD 的语言是"这一帧往哪里倾斜"。两者可以并存：**上限是硬的，倾斜策略是软的。**
2. **"同屏粒子数"这一维其实已经有 LOD 语义了**——距离剔除、重要性分级就是它的 LOD；只是判据用的是距离而不是像素误差。
3. **判断一个 LOD 系统是否成熟，看它有没有控制器**：只有阈值没有反馈环的，视角一转就会抖。

## Personal Knowledge

- **机制层 Easy**：离散 LOD 链、mip、距离剔除——日常工作，不复述。
- **判据层 Normal**：① **误差 vs 代理量的区别**（"能不能相加"是分水岭）；② **τ 是控制变量不是常量**（这条最值钱，也最容易被标题误导——标题写 frame rate，正文调 face count）；③ **三条路线在今天的引擎里并存**，不是替代关系。

## Learning Gap

- **"误差可加"这条在 GS 与虚拟几何里怎么落地**：Hoppe 的偏差空间是逐顶点的标量，GS 的误差是逐 splat 的协方差投影，两者如何合成一个场景级预算，库内三篇 GS-LOD 论文各给了一部分，但没有一篇给完整的可加性证明。
- **控制器设计**：除了 Hoppe 的乘法式比例控制，还有别的形态吗？（REACT 的迟滞缓冲是一种；纹理上采样那篇没有。）

## Next Step

1. 拿一个自己的场景，**把"同屏粒子数 ≤3000"改写成"每帧像素覆盖上限"**，看排序是否变化（这是 [[Budget Units]] 挂着的动作 ②）；
2. 读 [[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes|Hoppe 1997]] 的 §5（regulation + amortization）——**这是全篇唯一讲"怎么让它不抖"的部分**，也是今天三篇里唯一给出完整控制器的地方。

## Notes

- 建页依据：全库 99 个文件、363 处提到 LOD/Level of Detail，**零个概念页**；三篇 GS-LOD 论文与 Clark 1976 各自只覆盖一段。
- 与 [[Scalability and Quality Tiers]] 的边界：那一页讲"档位怎么划"，本页讲"档内怎么分配"。两者共用 Clark 1976 作为上游依据，但不合并——**合并会让"决定频率"这个关键区分消失**。
- 2026-11-07 建页，同日由 [[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes|Hoppe 1997]] 补齐 1976→2020s 的中段。
