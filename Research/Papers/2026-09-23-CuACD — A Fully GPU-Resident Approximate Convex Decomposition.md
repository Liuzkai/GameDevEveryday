---
type: paper
title: "CuACD: A Fully GPU-Resident Approximate Convex Decomposition"
authors: [Ruoxi Shi, Xinyue Wei, Fanbo Xiang, Zexiang Xu, Hao Su]
year: 2026
published: "2026-09-23 (arXiv) / SIGGRAPH Asia 2026 Conference Papers"
venue: "SIGGRAPH Asia 2026 Conference Papers (ACM TOG)"
url: "https://arxiv.org/abs/2609.28731"
code: "https://github.com/eliphatfs/cuacd"
project_page: ""
category: [gpu-computing, geometry-processing, collision, physics, tools-pipeline]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: Prototype
user_level: Normal
status: unread
aliases: [CuACD, GPU convex decomposition, 全 GPU 凸分解]
tags: [gpu-computing, collision, physics, pipeline, asset-processing]
---

# CuACD: A Fully GPU-Resident Approximate Convex Decomposition

## TL;DR

**做碰撞代理的"凸分解"（ACD）第一次整个搬进 GPU 常驻，单网格从十几秒掉到 0.2 秒级（~80–100×），且质量更好。** 你资产管线里"overnight bake"的那一步，现在是交互式的。

| 数据集 | CoACD | VisACD（前 SOTA） | **CuACD** |
|---|---|---|---|
| V-HACD 平均耗时 | 18.03 s | 9.60 s | **0.23 s** |
| PartNet-Mobility | 12.82 s | 7.42 s | **0.16 s** |
| Objaverse 子集 | 25.93 s | 15.92 s | **0.25 s** |
| 质量（concavity ↓ / 块数 ↓） | 0.0495 / 40.5 | 0.0604 / 37.5 | **0.0488 / 33.6** |

（RTX 4090 单卡；质量三列以 V-HACD 集为例。论文图 1 的"困难档"gallery（τ=0.03）全图 17.71 s。）

**方法一句话**：不再"把一个算法拆成很多小 kernel"，而是**把 warp（32 线程 SIMD）当作算法设计单元**，并给 GPU 装一个**设备端堆分配器**——让 ACD 的各个阶段融合成"永不离开 GPU"的 warp-resident kernel。

## Problem

**ACD（近似凸分解）是游戏物理与机器人仿真的标准预处理**：物理引擎的碰撞查询只对**凸形状**有近常数时间的算法（GJK 等），所以"**现代游戏里的每个角色、道具、环境资产在运行时可用之前，几乎都必须先被分解为一组小凸块**"。

但现有方法（V-HACD / CoACD / NavACD / VisACD 系）都有一个昂贵骨架：**对候选切割平面的搜索 × 凹度目标函数的评估**，内循环要反复构建两半的凸包并测量与原始表面的偏差——单网格十几秒到几十秒。

由此产生的**管线痛点（原文直接点名游戏）**：

> "…forces game pipelines into offline asset processing… **artists iterate on hundreds of mesh assets per sprint, convex decomposition is typically delegated to offline asset processing rather than integrated into interactive authoring, slowing iteration and encouraging workarounds such as hand-authored proxy geometry.**"

翻译成你的语言：**美术一个 sprint 迭代几百个网格，凸分解只能离线跑 → 迭代变慢 → 大家开始手搓代理几何。** 这就是本文要消灭的对象。

## Historical Context

```text
1984 Chazelle —— 精确凸分解是 NP-hard
2009 HACD —— 聚类式近似
2016 V-HACD —— 搜索切割平面范式的确立（至今最常见的资产管线工具）
2022 CoACD —— 更好的搜索 + 凹度目标（当前工业基线）
2024 NavACD —— 导航/变体改进
2026 VisACD —— 第一次 GPU 加速：把"射线可见性查询"卸到 CUDA/OptiX（~2×）★ 但切割与凸包仍在 CPU
2026 ★ CuACD（本文）—— 第一个"全 GPU 常驻"ACD：切割、凸包、凹度、搜索全部上 GPU
```

## Core Idea

### 1. 为什么 GPU 化 ACD "远不是机械移植"

两个根本障碍（原文分析）：

**(a) 每个 kernel 都以一队"掉队者"收尾（stragglers / last-wave underfill）。** 阶段之间的工作量分布又小又不均匀，核函数末尾只剩几根车道有活，整块 GPU 空转等下个 kernel。

**(b) 变长输出迫使 host CPU 回到环里。** 每个阶段的输出大小要等该阶段跑完才知道 → 为了给下一个 kernel 开缓冲区，**CPU 必须停下、读大小、再发射**。

两个成本都 ∝ kernel 边界数量 → 传统"phase-by-phase"拆法把 GPU 收益又吃回去了。

### 2. 解法一：warp 作为算法设计单元

CUDA 的 grid/block/thread 三级里，**只有 warp（32 车道共享一个指令指针）有硬件对应物**。本文把算法直接按 warp 书写：粗粒度上暴露"几百个独立任务、每任务一个 warp"；细粒度上 warp 内部用 vote/shuffle 原语协作。

**一个漂亮的量化**：填满现代 GPU，按**线程**粒度需要 $\sim 10^4$ 个独立线程；按 **warp** 粒度只需要 $\sim 10^2$ 个常驻 warp——而 **ACD 工作队列本身就有 $10^2$–$10^3$ 个活跃部件**，"顺手就够"。

### 3. 解法二：设备端堆分配器（消除 host 往返）

GPU 上跑一个 heap allocator（sticky-slab 池，占 70% 空闲显存）：各阶段的变长缓冲区**在设备端直接分配**。于是整条管线（worklist / lookahead tree / cutting / hull / concavity）**全部驻留显存，host 中途零同步**——输入上传后直到读出结果才见面。

### 4. 三个可复用组件（论文承诺 drop-in）

| 组件 | 做法 | 备注 |
|---|---|---|
| **Warp 协作凸包** | Preparata-Hong $O(n\log n)$ 分治的 warp 版；**整数坐标形式**（Bullet 的 `btConvexHullComputer`）→ 所有谓词是整数行列式，**无浮点容差决策** | 16 个 2-lane 小组，4 轮平衡树合并 |
| **方向极值预筛** | warp 沿 2 级 icosphere 的 40 个面法线做平行扫描，保留 80 个极值见证点，先建内凸包剔除内部点 | **删 ~70% 点**；n=65,536 时 >12×；n<1024 时开销大于收益，自动退回纯分治 |
| **无锁 union-find** | 连通分量拆分：Find 不做路径压缩（只读父链），union 弱化为 best-effort atomicCAS | 强进展无锁；森林高度保持 $O(\log n)$ |
| **LBVH Hausdorff 评估** | 凹度 = 双向 Hausdorff 距离，每个 mesh-hull 对一个 warp | 距离查询用 LBVH |

## Why It Works

1. **两个低效源都被"消除"而不是"优化"**：不是把 straggler 尾巴修短，而是**让阶段之间不再有 kernel 边界**（融合进 warp-resident kernel）；不是把 host 同步加速，而是**让它不存在**（设备端分配）——"取消式优化"在 GPU 工程里的标准样本；
2. **并发粒度上移一个数量级**（线程 → warp），要求的并行度从 $10^4$ 降到 $10^2$，恰好匹配不规则几何算法的固有并行度；
3. **整数谓词**消灭了浮点容差在并行环境下的判定分歧（确定性 + 免去容差调参）。

## Limitations

- 论文自述有 Limitations 与 future work 章节（本次未逐条展开，以 `Limitations and future work` 为关键词可在原文 §6 核对）；
- **协议要求 NVIDIA + CUDA**（RTX 4090 实测环境）——非 NVIDIA 平台（主机/移动/AMD）暂无路径；
- 质量对比中 NavACD 在 PartNet-Mobility 上本来就快（2.96 s），CuACD 的提升主要体现在 V-HACD / Objaverse 这类常规资产上；
- 发布状态：**SIGGRAPH Asia 2026（12 月）+ 代码已开源**——工程界集成仍需自行验证。

## Game Development Relevance

**4/5 —— 直接命中"资产管线预处理"这一环。**

1. **碰撞代理生成从"隔夜批处理"变成"交互式"**：0.2 s/网格意味着**可以在编辑器里实时跑**（改完模型立刻得到碰撞代理），不必再批处理 + 手工代理几何兜底。按你库里的语言：**这是"工具/管线"域的一次档位跃迁（离线档 → 交互档）**；
2. **"取消式优化"第三例**：9-24 的 GS 排序（"取消它"）→ 今天的 kernel 边界与 host 同步（"消掉它"）。**共同句式：先问"这个中间环节能不能不存在"，再问"能不能变便宜"**；
3. **与数据搬运视角（[[From Splats to Silicon — Rethinking Computational Efficiency of 3DGS]]）同构**："变长输出迫使 host round trip"与"显存流量"是同一族成本——**数据/控制在不同存储层级之间来回搬，每一趟都是纯开销**；
4. **与你的物理/VFX 交界**：凸分解是 destruction（破碎）、布料碰撞、载具物理的共同前置——**碎片资产的批量预处理**是它的直接应用面。

## Unreal Engine Relevance

- UE 侧的碰撞生成（Convex Decomposition 工具、Chaos 物理的 Collision）是同类问题的引擎内版本；**CuACD 目前是独立 CUDA 工具**，是否/何时进引擎未知；
- 值得关注的是**模式**：如果"编辑器内交互式凸分解"成立，Chaos 的碰撞代理工作流（尤其大批量静态网格）会直接受益；
- 与 [[Niagara]] 无直接关系（本文不涉及特效）。

## Technology Evolution

```text
（几何预处理管线的时间轴）
2009-2016   HACD → V-HACD：搜索式 ACD 确立，成为资产管线标配
2022        CoACD：更好的凹度目标，工业基线（"隔夜批处理"时代顶点）
2026 VisACD 射线可见性查询上 GPU（~2×）——只动了配件，主干仍是 CPU
2026 ★ CuACD 全 GPU 常驻：search / cutting / hull / concavity 全部合并进 warp-resident kernel
              ↑ 与 GS 排序（Stochastic GS Denoising）、世界模型记忆（WorldCrafter）同期
                 "取消中间层"三部曲：排序 / kernel 边界 / 物化记忆
```

## Relationships

### Related

- **⟷ [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising]]**（"取消式优化"）：一个取消排序，一个取消 kernel 边界与 host 同步——**同一种"先问能不能不存在"的世界观**；
- **⟷ [[From Splats to Silicon — Rethinking Computational Efficiency of 3DGS]]**（数据搬运）：host round-trip 与显存流量同族——**数据/控制在不同存储层级间来回搬，每一趟都是纯开销**；
- **⟷ [[GPU-Driven Rendering]]**（概念）：同一条"把 CPU 从环里拿出去"的路线，一个在渲染管线、一个在几何预处理——**引擎工程与工具工程共享同一种世界观**。

## Personal Knowledge State

- **user_level: Normal**：你天天做资产管线与性能，**"预处理为什么只能离线"的痛感是 Easy 的**；新的是**"warp 为设计单元 + 设备端分配器"这套消除 kernel 边界/ host 回环的 GPU 工程手法**——这是你 GPU 知识里"DrawCall/调度"一侧的自然延伸；
- 与你分档体系的联动点：**它把"预处理工具"插到了时间轴的另一端**——你的五档体系管的是运行时，而这类工具管的是**进入运行时的成本**（内容生产吞吐）。两者共用同一条预算哲学。

## Learning Value

1. **一条可复用判据**：**"两个低效源（尾波欠填充 / 变长输出）都与边界数量成正比 → 消除边界，而不是优化边界内的成本。"** 任何"多 pass GPU 管线"都适用；
2. **一个具体手法**：**把并行度要求的数量级算清楚**（$10^4$ 线程 vs $10^2$ warp），再选粒度——比"直觉上 GPU 快就上 GPU"可靠得多；
3. **一个工程事实**：Bullet 的 `btConvexHullComputer`（游戏物理引擎代码）被直接用作研究系统的整数化凸包底座——**游戏工程积累在反向喂研究**。

## Notes

- arXiv 2609.28731（cs.CG，9-23 提交）；SIGGRAPH Asia 2026 Conference Papers；代码 https://github.com/eliphatfs/cuacd ；
- 全文已下载核对：表 1 全部数字、warp 设计论述、heap allocator、预筛 70%、>12×@65536、无锁 union-find、17.71 s gallery（τ=0.03）、70% VRAM sticky-slab 均出自原文；
- 署名：UC San Diego（Shi / Wei）+ Sudo GmbH（Xiang / Xu / Su）。
