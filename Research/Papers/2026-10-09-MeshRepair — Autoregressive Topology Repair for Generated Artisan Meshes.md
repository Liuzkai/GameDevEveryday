---
type: paper
title: "MeshRepair: Autoregressive Topology Repair for Generated Artisan Meshes"
authors: [Xiyu Wang, Yufei Wang, Ruocheng Wu, Zhihao Li, Lanqing Guo, Bihan Wen]
year: 2026
published: "2026-10-09（arXiv v1, 2610.14113；周三公告组 2026-10-14）"
venue: "arXiv Preprint（cs.GR，30 天窗口内首批周末清关件）"
url: "https://arxiv.org/abs/2610.14113"
code: ""
project_page: ""
category: [mesh-processing, topology, generative-assets, autoregressive, prefix-lm, asset-pipeline]
importance: B+
historical_importance: 0
game_relevance: 4
production_readiness: "Research（benchmark 对照修复工具；代码未提）"
user_level: "Normal（结论层 12 分钟：'就地修而非重生成' + 删除期/添加期两段编辑）"
status: unread
aliases: [MeshRepair, Artisan Mesh Repair, 生成网格拓扑修复]
tags: [mesh-processing, topology, generative-assets, asset-pipeline]
---

# MeshRepair: Autoregressive Topology Repair for Generated Artisan Meshes（Wang et al. 2026）

> **入库 2026-10-14（Run 36）。** 周三公告组（cs.GR 首批周末清关件）捕获；提交 2026-10-09。
> **一句话定位**：**AI 生成网格的"出厂质检与返修工位"**——"artisan" 级生成器产出的网格拓扑已接近美术偏好，但仍带**洞 / 游离面 / 重复面**；**从头重生成代价高、且可能整个换掉你想要的拓扑**；本篇把修复建模为**在草稿上原地生成"编辑序列"**（面级 prefix-LM：先删后加），**未触及的面原样通过**。
> **库内位置**：[[Subdivision Surfaces]] 产业线的**前置清理工位**——"生成 → **修复** → SubD/生产"链条中枢；与 [[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction|SubDGuide]]（mesh→SubD 逆问题）构成"**修干净 → 反推 cage**"的上下游；"**就地修正，而非重新生成**"是今日跨库母题（与 PriMoRetarget/GeoFix 同族）。

## TL;DR

**问题**：artisan mesh 生成器的产出"接近但不到"美术级拓扑——典型缺陷：**holes（洞）、stray faces（游离面）、duplicated faces（重复面）**。修复的三条现有路都不对：**重生成**（贵；且"另一抽"可能换掉原本满意的拓扑、或在别处引入新错）；**人工修**（贵）；**通用修复工具**（不懂"artisan 偏好"，quad-equivalent F1 上不去）。

**做法（面级"编辑序列生成器"）**：

```text
输入：草稿网格（draft）——作为前缀（prefix）喂入
模型：面级修正网络，基于 prefix-LM（双向读网格 + 自回归吐编辑序列）

编辑序列 = 删除期 → 添加期
  ① deletion head：自回归逐面决定"删哪些面"
  ② addition head：选顶点构成新面（必要时补充新顶点）
  ——未触及的面原样通过（草稿的顶点与布局被保留）

解码步数：只生成"编辑"，远少于"把每个面重新生成一遍"
```

**结果**：在**脚本注入缺陷**与**两个生成器的留出草稿**上对照修复工具——quad-equivalent face F1 提升、非流形面清零、拓扑更干净。

## Problem

- **"生成 ≠ 生产"的最后一公里**：拓扑质量（面的走向、密度、四边性）决定资产能否进下游（细分、动画、UV、LOD）；生成器"接近美术偏好"意味着**大部分网格已经对，只有局部需要修**——全盘重来是浪费；
- **修复 ≠ 重生成的经济学**：重生成风险 ="a new draw may change the preferred topology entirely or introduce errors elsewhere"（抽卡换的是整张牌）；修复的价值 = **保留已认可的部分**；
- **任务定义（本篇的贡献之一）**：artisan topology correction —— "把草稿的拓扑**就地**修向美术会做的布局"（in place, toward the layout an artist would make）。

## Core Idea

**"修正 = 对草稿生成最小编辑序列"** —— 两个设计选择：

```text
① 先把"错"删掉，再"补"对（deletion phase → addition phase）：
   删除期清出空间，添加期填补——顺序固定，模型不用学"交错混合"的难模式
② 草稿作为前缀、只生成 delta：
   未触及的面 pass through —— 输出天然"贴着草稿"，改动最小化
   （解码步数与"错误量"成正比，而不是与"网格规模"成正比）
```

## Technical Approach（摘要级）

- **面级 prefix-LM**：双向读入网格（面序列），自回归输出编辑序列；**deletion head** 逐面选择删除；**addition head** 选顶点组面（"also adds vertices when needed"）；
- **保留语义**：只有被编辑的部分改变——顶点与布局继承自草稿；
- **评测**：两类对照——（a）**脚本化缺陷**（可控注入洞/游离面/重复面）；（b）**留出草稿**（两个独立 mesh 生成器的产出）；指标含 **quad-equivalent face F1**（"等效四边面"质量——指向 SubD 工作流的硬语义）与**非流形面清除**；
- **效率叙事**：修复一步（编辑序列）vs 重生成全网格——解码步数更少。

## Key Contribution

1. **任务形式化**：artisan 网格的"就地拓扑修复"（面向"只差局部"的生成资产）；
2. **编辑序列式修复**：删除期 + 添加期的两段自回归 + 双 head（面级删除 / 顶点级添加）——"**输出是人能理解的编辑列表**"（可审计、可观感）；
3. **保草稿性**：untouched faces pass through——修复不"重画"已认可的部分；
4. **评测协议**：脚本缺陷 + 留出草稿双基准；quad-equivalent F1 / 非流形清零。

## Why It Works

- **"局部问题用局部手段"**：生成资产的问题是局部的（几个洞、几片游离面）——**编辑序列的复杂度跟着"错多少"走，而不是跟着"网格多大"走**（与"预算跟着缺口走"同构——10-13 的 [[2026-10-09-Efficient Confidence-Aware Densification for 3DGS|Confidence-Aware Densification]] 是同一判据的 GS 版本）；
- **先删后加的课程学习感**：两段式把"删"和"补"解耦——比混合编辑更可学（对比：自由编辑序列的搜索空间大得多）；
- **prefix 结构 = 保真约束**：未编辑部分"物理上"无法被改坏——**把'别乱动'写进了架构**（而非损失函数）。

## Limitations（摘要级，未读全文）

- benchmark 为自设协议（脚本缺陷 + 两个生成器的草稿）——**跨生成器/跨风格的泛化待更多第三方验证**；
- "artisan 风格"的监督来源（训练数据构成）与代码放出情况未在摘要说明；
- 面级编辑对大范围拓扑崩坏（系统性错误）的适用边界未知（"局部修"与"重生成"的切换阈值是工程问题）。

## Game Development Relevance

- **AI 资产生态的"返修工位"**：生成式建模进入管线后，**"质检 + 局部修复"是绕不开的中间工序**——本篇给出"修而不是重画"的工作流骨架；对熟悉 DCC retopo 的 TA：**"编辑序列"正是一种可审计的自动 retopo 输出**；
- **quad 语义 = 下游通证**："quad-equivalent face F1"提醒：**四边面率不是审美偏好，是 SubD/UV/动画的下游硬约束**——修复器的目标函数直接编码了生产语义；
- **判据**：*"这份生成资产是'重抽'还是'返修'？"*——返修的前提是"大部分已对 + 错误可定位"（对照 10-13 的 Make It Flow："注入 vs 重建"的同构选择）。

## Unreal Engine Relevance

- 无直接集成；接口层参考：**导入期校验/修复**（Interchange/Datasmith 管线中"自动修复"类功能的语义正是本篇任务）；与 UE Modeling Tools 的"编辑动作可撤销"工作流同构（编辑序列 ↔ 事务性编辑）；SubD 资产进引擎前的"拓扑净化"步骤有实际价值。

## Technology Evolution

```text
生成资产生态的工具链位置（本库视角）：
  生成：artisan mesh generators（2610.13947 "Artisan" 一线）
   ▼
  ★ 修复：MeshRepair（本篇，2026-10-14 入库）——"就地编辑序列"派
   ▼
  逆问题：[[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction|SubDGuide]]（mesh → SubD cage）
   ▼
  生产：[[Subdivision Surfaces]]（cage → 极限曲面 → 引擎）
  ——四段链条在 2025-2026 内全部出现 2026 态节点；"生成资产要过生产门"是共同叙事
```

## Relationships

### Based On

- **Artisan mesh 生成一线**（生成器产出为输入）；**autoregressive mesh generation / prefix-LM** 的表示法（"draft 作为 prefix"沿用生成模型自己的语汇）。

### Related

- [[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction|SubDGuide]]——"修干净（本篇）→ 反推 cage（SubDGuide）"的上下游；两者都在解"生成的稠密/dirty 资产怎么进生产"；
- [[Subdivision Surfaces]]——"quad-equivalent F1"的下游语义（四边面 → SubD 工作流）；
- [[2026-10-09-Efficient Confidence-Aware Densification for 3DGS|Confidence-Aware Densification]]——"预算/工作量跟着缺口走"的跨域同判据（GS 密度端）；
- [[2026-10-08-PriMoRetarget — Humanoid Motion Transfer Through Prioritized Motion Retargeting|PriMoRetarget]]——"就地修正 vs 重算"的动画端样本。

### Followed By

- 待观察：代码/数据放出；"修复 + 逆问题"端到端的可能（修完直接进 SubD 重建）；生成器侧是否内嵌"修"作为后处理标准件。

## Personal Knowledge State

- Normal（结论层 12 分钟："就地修"立场 + 两段编辑序列 + quad F1 语义）。

## Learning Value

- **两句可背**：① "**修复的输出应该是一份编辑列表，而不是一张新网格**"（可审计、保已认可部分、成本∝错误量）；② "**quad 率是下游硬约束，不是审美**"；
- **判据**：*"这份资产（或数据）是'重生成'还是'局部编辑'？"*——先问"大部分是否已对"，再选工具。

## Notes

- 周三公告组（10-14）捕获；摘要级记录（未读全文）；提交 2026-10-09；作者团队含 Bihan Wen（图像/图形领域，同日另见其参与的 2610.14254）。
