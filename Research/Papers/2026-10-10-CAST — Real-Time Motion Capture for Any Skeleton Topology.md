---
type: paper
title: "CAST: Real-Time Motion Capture for Any Skeleton Topology"
authors: [Pu Pang, Zhongfei Qing, Ling-Hao Chen, Songnan Li, Hua Huang, Bo Hu]
year: 2026
published: "2026-10-10（arXiv v1, 2610.14856；周三公告组 2026-10-14）"
venue: "arXiv Preprint（cs.CV，39 页；代码开源）"
url: "https://arxiv.org/abs/2610.14856"
code: "https://github.com/fankewen/CAST"
project_page: ""
category: [motion-capture, monocular-video, skeleton-topology, retargeting, real-time, animation-pipeline]
importance: A-
historical_importance: 0
game_relevance: 5
production_readiness: "Prototype（代码开源；CAST-B 因果变体 A100 单卡 51+ FPS，含图像编码与动作预测；TopoCap 协议 + Mixamo 基准 + 用户研究）"
user_level: "Normal（结论层 15 分钟：'可观测性分析 + 可靠性门控' + 动捕/重定向边界合并）"
status: unread
aliases: [CAST, Topology-Aware Motion Capture, 任意骨架拓扑动捕]
tags: [motion-capture, monocular, retargeting, real-time, animation]
---

# CAST: Real-Time Motion Capture for Any Skeleton Topology（Pang et al. 2026）

> **入库 2026-10-14（Run 36）。** 周三公告组（cs.CV 周末清关件）捕获；提交 2026-10-10；39 页 + 代码开源。
> **一句话定位**：**"目标骨架和视频里的人长得不一样，也要能捕"**——单目视频 → 动作，但**目标骨架拓扑可以 ≠ 视频主体的骨架**（不同关节结构、甚至没见过的结构）。此前方法的通病：**关节旋转误差沿运动链传播、跨骨架耦合**；CAST 用"**可观测性分析**"把旋转分量分成"能被骨骼方向约束的"与"不能被约束的"，**能测的用几何修正（可靠性门控、父到子逐级对齐），测不到的交给学习预测器**。
> **库内位置**：**"动捕 ⇄ 重定向"边界合并的第一节点**——[[Motion Retargeting]] 原本是"捕到之后再把动作搬过去"，CAST 把"目标形态"前置成捕捉条件本身；与 [[2026-10-08-PriMoRetarget — Humanoid Motion Transfer Through Prioritized Motion Retargeting|PriMoRetarget]]（优化式重定向）构成"**捕捉侧 vs 转换侧**"对照；其"可靠性门控"是今日"可信度三连"（SVGF 方差 / CAST 门控 / 读者报告"重要度 vs 可信度"）的动画端样本。

## TL;DR

**问题**：单目动捕从"人形捕给人形"扩展到"**任意目标骨架**"（四足、异形、不同比例/拓扑的虚拟角色）——两类误差源叠加：① 旋转误差**跨关节耦合**，沿运动学层级**逐级传播放大**；② 目标骨架的某些自由度在观测里根本**不可见/不可约束**（比如仅凭骨骼方向推不出的自旋）。

**做法（三个组件）**：

```text
① 可观测性分析（observability analysis）：
   刻画"哪些旋转分量能被出射骨骼方向（outgoing bone directions）约束"——
   把"可解 / 不可解"写成显式判据（不是拍脑袋分堆）
② GRACE 修正模块（几何侧）：
   以"预测的全局骨骼方向"为引导，按父→子顺序做【可靠性门控的对齐】——
   修复合成朝向里"可观测"的分量
③ 学习预测器（学习侧）：
   承接"不可观测"的分量（门控放行给它）——两模块联合训练，
   监督来自运动数据本身（motion-derived supervision；图像编码器冻结）
```

**结果**：TopoCap 评测协议下跨目标拓扑（**含未见结构**）的动作恢复提升；Mixamo 整理的跨骨架对照基准 + 用户偏好研究；**CAST-B 因果变体在单张 A100 上 51+ FPS**（含图像编码与动作预测全流程）；代码已开源。

## Problem

- **"目标骨架"这个变量此前被当作常量**：动捕系统通常假定"被捕捉者与目标角色同拓扑"（人形→人形）；跨拓扑的情形留给重定向管线——但**捕捉阶段的误差会污染重定向的输入**；
- **旋转误差的层级传播**：关节旋转误差不是独立噪声——沿运动链**耦合**（父关节的错会改变子关节的观测含义）；
- **不可观测自由度**：视觉观测只能约束"骨骼指向"能推出来的旋转分量；**绕骨轴的扭转等内容"看视频推不出来"**——此前方法把两类分量混在一起回归，误差互相拖累。

## Core Idea

**"把'修正'和'预测'按可观测性分工"**：

```text
可观测分量（能被骨骼方向约束）→ 几何修正（GRACE：可靠性门控 + 父到子逐级对齐）
不可观测分量（观测给不出信息） → 学习预测器（先验知识的地盘）

——"能测的用几何（确定、可解释），测不到的交给学习（先验）"
```

**工程要点**：修正按**父→子顺序**逐级进行（误差不跨级传播）；门控信号是"可靠性"（对齐结果可信才采纳）——**不是全盘修正、也不是全盘学习**。

## Technical Approach（摘要级）

- **可观测性分析**：把"出射骨骼方向"作为观测量，推导"哪些旋转自由度可被其约束"的判据（论文自述为显式刻画）；
- **GRACE**：对"合成朝向"（composed orientations）做可靠性门控的对齐；**按 parent-to-child 顺序**——先修父级、再修子级（避免把子级误差误诊给父级）；
- **联合训练**：动作预测器 + 修正模块端到端（监督来自运动数据本身，无需 3D 真值配对？——摘要口径为 "motion-derived supervision"，具体构造待全文）；**冻结图像编码器**（省算力、稳定特征）；
- **实时化**：CAST-B 因果变体（causal，流式可用）；51+ FPS on A100（含图像编码在内的全流程口径）；
- **评测**：TopoCap 协议（跨拓扑动捕）+ Mixamo 跨骨架基准 + 用户偏好研究。

## Key Contribution

1. **任务扩展**：单目动捕的"目标骨架拓扑 ≠ 主体"设定——**捕捉与重定向的边界第一次被系统性地搬进捕捉器**；
2. **可观测性分析**：把"哪些旋转可解"写成显式判据——**"误差修正"从整体回归变成"按可解性分工"**；
3. **GRACE 门控修正**：可靠性门控 + 父到子顺序——工程上"修正不倒退"；
4. **实时 + 开源**：51+ FPS（A100）+ 代码——可复现的工程基线。

## Why It Works

- **误差的结构性消解**：旋转误差耦合的根源是"层级+观测不完备"——**先按层级顺序修正可观测部分（消掉传播源），再让学习处理剩余**（而不是让一个网络对全部误差做端到端回归）；
- **"可靠性门控"防止负优化**：几何修正的前提是"对齐结果可信"——不可信就放行给学习；**与降噪/复用的"置信度"思想同构**（今日 [[Schied — Spatiotemporal Variance-Guided Filtering (2017)|SVGF]] 的方差引导、读者报告的"可信度轴"）；
- **冻结编码器 + 运动派生监督**：把学习容量集中在"运动侧"，工程上便宜且稳（对照库内 `freeze-and-distill` 配方家族）。

## Limitations（摘要级）

- 单目视频的固有歧义仍在（深度/遮挡/快速运动）；"未知结构"的提升幅度未在摘要量化；
- A100 的 51 FPS ≠ 消费级显卡的实时（工程部署需再测）；
- "motion-derived supervision" 的数据构成与自采数据依赖未在摘要展开。

## Game Development Relevance

- **动画数据管线的最上游**：动捕"一次捕、多骨架用"——同一段参考表演直接落到不同拓扑角色（怪物/NPC/异形），**省掉"捕完再重定向"的中间误差**；对动捕量大的项目，这是"数据入库格式"级的改变；
- **与 [[Motion Retargeting]] 的关系**：三路线（IK 优化 / 数据驱动 / 优先级优化）之外的新形态——"**捕捉即重定向**"（retargeting-in-the-loop）；
- **判据**：*"我的误差要'整体回归'还是'按可解性分工'？"*——能观测的部分用确定几何、观测不到的交给先验，是任何"测量+补全"系统的通用切分（同日对照：MeshRepair 的删除/添加分工、读者报告的采样/复用分工）。

## Unreal Engine Relevance

- 接口层：UE `IK Retargeter`/`Control Rig` 的工作流与本篇目标重合（跨骨架适用性），但 CAST 在**捕捉端**完成拓扑适配——将来若产品化，可减少一段"先捕标准人形、再逐链重定向"的误差链（当前为研究原型，不越级声称集成路径）；
- 实时性口径（A100）与引擎内目标（消费卡 + 帧预算内）之间仍有距离——观察项。

## Technology Evolution

```text
动捕 → 重定向 的关系演化（本库视角）：
  传统：动捕（同拓扑假设）→ 重定向（事后搬动作，误差已入账）
  [Gleicher 90s] → DCC/引擎工具链（HumanIK / Mecanim / IK Rig）
  2024-26 学习式重定向三路线（优化/数据驱动/优先级）——[[Motion Retargeting]] 库内三节点
  ★ 2026 CAST：捕捉器**直接接受任意目标拓扑**——"重定向前移进捕捉循环"
  ——边界合并的动因：单目动捕精度提升 + 可观测性建模 + 实时推理预算
```

## Relationships

### Related

- [[Motion Retargeting]]——概念页节点新增：**"捕捉侧"**（此前三节点全在"转换侧"）；
- [[2026-10-08-PriMoRetarget — Humanoid Motion Transfer Through Prioritized Motion Retargeting|PriMoRetarget]]——"约束保护重要目标"的转换侧样本（CAST 门控是同一哲学的捕捉侧样本）；
- [[Schied — Spatiotemporal Variance-Guided Filtering (2017)]]——"可信度驱动处理"的渲染端先祖（今日三连）；
- [[Neural Animation]] / [[Motion Generation]]——上游数据源（动作数据的"生产端"改善，下游生成/检索受益）。

### Followed By

- 待观察：消费级硬件实时化；与引擎 IK Rig 的端到端对照；代码库可复现性核查。

## Personal Knowledge State

- Normal（结论层 15 分钟：可观测性分析一句话 + 门控分工 + 实时口径；39 页全文未读，摘要级）。

## Learning Value

- **一句可背**：**"能观测的用几何修正，不能观测的交给学习——中间用'可靠性'决定采纳"**；
- **判据**：*"这个误差的哪些分量是'测得到'的？"*——先做可观测性分解，再设计修正器（对动画/物理/渲染的"残差处理"普遍适用）。

## Notes

- 周三公告组（10-14）捕获；摘要级记录；提交 2026-10-10；代码 `github.com/fankewen/CAST` 已放出（未本地核验）。
