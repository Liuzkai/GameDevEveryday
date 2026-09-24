---
type: paper
title: "PartLLM: A Unified Multimodal Foundation for 3D Part Segmentation"
authors: [Zhe Zhu, Yiheng Zhang, Peng Li, Zixing Zhao, Honghua Chen, Yaqing Zhang, Le Wan, Zhiyang Dou, Cheng Lin, Yuan Liu, Mingqiang Wei, Wenping Wang]
year: 2026
published: 2026-09-22
venue: "SIGGRAPH Asia 2026 / ACM Transactions on Graphics 45(6), Article 178（DOI 10.1145/3842577）
         —— 太原理工大学 × 腾讯 Visvise × 港科大 × 岭南大学 × MIT × 澳门科技大学 × Texas A&M"
url: "https://arxiv.org/abs/2609.25832"
code: ""
project_page: "https://czvvd.github.io/PartLLMPage/"
category: [3d-understanding, asset-pipeline, multimodal-llm, segmentation, tooling]
importance: A-
historical_importance: 2
game_relevance: 4
production_readiness: Early Production（腾讯 Visvise 产品线方向的学术成果）
user_level: Normal
status: unread
---

# PartLLM: A Unified Multimodal Foundation for 3D Part Segmentation

## TL;DR

**把"3D 分件"从一堆互相独立的任务，统一成一道题："给定形状 + 用户意图，自回归地生成这堆部件。"**

- **第一步（决策，语言层）**：3D-aware MLLM 生成**部件假设序列** —— 每个假设 = 语义标签 + 粗 3D 位置 + 与已生成部件的关系 → 形成**动态开词表**（标签集随形状与意图现生成）；
- **第二步（解算，几何层）**：分解感知解码器把每个点做 **(S+1) 路联合分配**（S 个部件 + 背景），**互斥**、边界一致；
- 同一模型同时支持**文本指定 / 点选交互 / 全形状分解（粒度可控：coarse / fine / 数量）**；323K 形状 → **1.1M 统一训练样本**；
- 成绩：类别无关全形状 **+34.5 mIoU（coarse）/ +32.0（fine）**；交互点击 1 次 **+10.1**、7 次 **+18.5**；**推理还比基线更快**（P=5 时 4.8 s vs PartField ~10 s）。

> **一句话定位**：这是库内"**生成只做决策层**"的第 5 例（DLSS 5 / Magpie / PBR-Latent / ProxyBuild 之后）—— **MLLM 只负责"要分出什么"，稠密边界交给专用解码器**；也说明游戏工业的"**资产理解层**"（分件）正在被大模型收编，且**出品方是腾讯游戏美术工具线（Visvise）**。

## Problem

"3D 分件"在下游管线里是绑定、碰撞、材质、LOD、编辑的**前置步骤**。但现有方法各自只解决一个子问题（原文 Table 1 的系统对照）：

| 家族 | 代表 | 缺口 |
|---|---|---|
| 类别无关分块 | PartField | 只给区域，**不给语义身份** |
| 类别无关 + 可提示 | PartSAM / P3-SAM / SAM3D | 有交互，**不做"整体怎么拆"的决策**（过分割、无语义） |
| 文本定位 | FIND3D / CoSMo3D | 假设"要哪些部件"已经知道 |
| 全形状语义 | 少数工作 | 粒度固定、不能按意图变 |

**核心洞察（原文的立场）**：这些任务**本质是同一道题** ——"在某个意图下推断部件级分解"。差别只在**意图怎么表达**（文本/点击/无）与**期待什么粒度**。

**且这个问题的答案本身不唯一**：部件数量随对象变化，同一个物体在不同**语义视角 / 粒度**下有不同的合法分解。→ 所以 **它不该被建模为判别任务（固定答案），而应是"意图条件的生成问题"**。原文一句很锋利：

> "Modeling it as a deterministic framework usually results in an **averaged and blurry** result, while generative frameworks have the potential to **sharply** segment all parts."

## Core Idea

**Autoregressive Semantic Decomposition（自回归语义分解）**：

```text
输入：点云（形状）+ 用户 prompt
  ↓ ① 3D-aware MLLM（点云编码器 → 3D tokens → LLM）
生成：label=Backrest, bbox=[...] <|PART|>
      label=Seat,     bbox=[...] <|PART|>
      label=Armrest,  bbox=[...] <|PART|>
      <|BG|>
      （语言形式的"部件假设"序列，数量可变）
  ↓ ② 分解感知解码器
每个点 → (S+1) 路联合分配（互斥 softmax）
  ↓
输出：语义化部件掩码（全局一致）
```

三个设计点：

**① 决策前移**：原文明说 —— "**the ambiguity of what to segment has already been resolved before mask decoding**"。分割的"元问题"（要分出什么、有几个、叫什么）由语言模型**在掩码解码之前**一次性解决；掩码解码退化为"**把点分配给已经定好的标签集**"。这与库内 [[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction|Mira-Scene]] 的"学习只建立对应、解算交给确定性几何"是**同一种姿势的两个层级**（那边是位置，这边是标签）。

**② 联合分配替代独立二值掩码**：SAM 式"每 query 一个独立掩码"会**互相重叠、边界不一致**；PartLLM 让所有部件 + 背景**在同一个 softmax 里竞争**（"different parts compete within the same decomposition"）→ 互斥 + 对称部件身份一致。**损失就是个动态标签空间上的逐点交叉熵**。

**③ 动态标签集 + 开词表**：部件名由模型**生成**（不是外部标签事后贴上），标签集随形状与意图变化；`<|BG|>`（背景）是**"当前假设集的语境补集"**（不是一个固定类别）—— 因此解码器**不需要任何任务特定的分支逻辑**，所有任务共用同一条 softmax。

**统一任务 = 统一 prompt-response 模板**（原文 Table 2）：

| 任务 | prompt 变体 |
|---|---|
| 全形状 | general / **coarse / fine / number（粒度控制）** |
| 文本指定 | "Please segment the Backrest and Armrest…" |
| 交互 | 首次点击定位 → 多轮 refine（include/exclude point） |

→ 异质标注变成同一条训练信号：**"任务与数据多样性成为一个新的 scaling 轴"**（结论原文）。

## Key Numbers

| 维度 | 结果 |
|---|---|
| 类别无关全形状（Table 3） | 3DCoMPaT200 上比最强基线 **+34.5 mIoU（coarse）/ +32.0（fine）**；PartNeXt / HY3D-Bench / PartObjaverse-Tiny 全面领先 |
| 语义全形状（Table 4） | Mask mIoU 与 **SA-mIoU（语义感知）** 双料第一；两阶段基线语义匹配后**掉 17.5–23.6 分**（说明"先分块再贴标签"路线的语义脆弱性） |
| 交互式（Table 5-6） | 1 次点击 **+10.1 mIoU**；7 次点击后差距拉大到 **+18.5** |
| 数据缩放（Table 12） | 加文本数据：全形状 mask mIoU **56.2 → 62.7**（且启用文本评测）；全部数据源（几何+文本+交互）：**74.2 mIoU** |
| 鲁棒性 | 随机旋转 **78.5 ± 2.2 mIoU**；prompt 表达变化 76.2（默认模板 78.8） |
| 效率（Table 13, H20） | P=5/10/20/30 → **4.8 / 7.1 / 11.7 / 16.3 s**，mIoU 74.2；**比 PartField(~10s/35.0)、PartSAM(~12s/42.2)、P3-SAM(~10s/40.3) 更快且更准** |

## Limitations

- **类别先验错 → 全盘错**：几何上类别歧义时（"路由器"被认成"床"），会沿错误先验**幻觉出整套部件结构**（原文 Fig 14 的代表性 failure）；把类别作为 prompt 附加信息可修正。→ 依赖"对象级上下文"，未来拟引入渲染图像/多视角证据与 CoT；
- 生成式范式的固有代价：推理延迟**随请求部件数增长**（P=30 时 16.3 s）——适合**离线/工具链**，不适合每帧运行时；
- 评测仍以合成/基准数据为主（含真实扫描泛化实验，但规模有限）。

## Game Development Relevance

- **资产管线自动化**：分件是在 **绑定（骨架划分）/ 碰撞（凸包分解）/ 材质（按部件赋材质）/ LOD（部件级简化）/ 编辑（部件级操作）** 之前的公共前置。本文的"**意图条件分件**"意味着同一条资产可以按**不同下游需求**生成不同粒度的分解 —— **"粒度"第一次成为可点单的接口**；
- **对 AI 生成资产尤其重要**：原文专门展示了在**几何生成模型的稠密网格**与**自回归拓扑生成器的低模**上的分解结果 —— AI 造物潮之下，"**理解并结构化 AI 资产**"的中间层需求正在成形（与库内 [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]] 的"结构锚定"母题相接：生成之后，**结构化理解**是下一站）；
- **Part-aware editing 是产品化接口**：预测掩码直接作为"编辑手柄"（换材质 / 改几何），原文 §4.7 给出两个工作流 —— **这就是 TA 工具链里"部件级批量操作"的原生形态**；
- **出品方信号**：**腾讯 Visvise**（腾讯游戏美术工具线）+ 太原理工等，SIGGRAPH Asia 2026 —— 库内"腾讯来源"再度出现（继 LightOpt、WorldCrafter 之后），且这次是**直接面向美术生产工具**的工作。值得放入 Watchlist 跟踪其产品化动作。

## Unreal Engine Relevance

- 不直接映射某个 UE 模块；但对应**资产导入/Pipeline 前置工具**的位置（"分件 → 骨骼/碰撞/材质/LOD"）；
- **"粒度可控分解"与 UE 侧 LOD/HISM 的分级思想同构**：部件首先是一个**语义单位**，其次才是几何单位。若生产线上出现"按部件批量处理资产"的需求（Nanite 分件剔除、PCG 资产替换、程序化材质按件分发），本工作的"意图条件 + 粒度接口"就是现成的**能力定义**。

## Technology Evolution

```text
2017-2019  PointNet / PartNet 时代：固定词表逐点分类（闭世界）
      ↓
2023-2025  2D 基础模型上抬（SAM 蒸馏 / 多视融合）；可提示分割（点选/文本）
      ↓
2025-2026  ● 类别无关分块成熟（PartField）；交互分割成熟（P3-SAM / PartSAM）
           ● 共同天花板：语义身份与"整体决策"缺席
      ↓
2026  ★ PartLLM：把分件统一为"意图条件生成"，与语言模型对齐
       —— 部件名生成、粒度可点单、任务/数据统一为一个 scaling 轴
      ↓
候选方向：多模态证据（渲染图）消歧；CoT 稳定高层决策；部件级编辑/生成闭环
```

## Relationships

### Based On

- **MLLM 的空间感知能力路线（Bai et al. 2025 等）** —— "用语言接口表达部件假设"的前提
- **SAM 式可提示分割范式** —— 被统一与被修正（独立掩码 → 联合分配）
- **点云编码器（Wu et al. 2024）** —— 3D tokens 的来源

### Related

- [[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction]] —— **"结构锚定"母题的 AI 资产侧同位体**：Mira-Scene 建立"像素↔规范坐标"的对应，PartLLM 建立"点↔部件标签"的对应 —— **都在"生成"与"确定性解算"之间插一层显式对应**；
- [[Procedural Content Generation]] —— 反向连接：PCG 从"规则生成结构"，PartLLM 从"结构反推语义"（一个生成、一个理解，未来可能在"生成即可编辑资产"的闭环上汇合）；
- **"生成只做决策层"目录（第 5 例）**：[[DLSS 5 — Generative Neural Rendering|DLSS 5]]（生成只给像素最终值）/ [[Magpie — Real-Time World Renderer for Interactive Games|Magpie]]（生成只重画白模）/ [[2026-09-17-PBR-Latent — Physically Based Rendering in the Latent Space|PBR-Latent]]（生成只做潜空间解码）/ [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]]（生成只做角色推断）/ **PartLLM（生成只决定"分出什么"）**；
- **动作生成线（[[Motion Generation]]）** —— 方法论同构：**答案不唯一（one-to-many）时，把判别式回归换成条件生成** —— 动作与分割两线今天在同一判据下会合。

### Contrasts

- 与"确定性分割 + 后处理贴标签"的两阶段路线形成对照：**那类方法语义匹配后掉 17.5–23.6 分**（Table 4）—— "决策与解算分离"不是工程细节，是**精度结构**。

## Personal Knowledge State

- **user_level: Normal**。前置（3D 表示、分割基础概念、LLM 接口直觉）都在你的舒适区边缘；本笔记只取三层：**任务统一的动机 / 联合分配机制 / 粒度接口**，不深入 MLLM 训练细节；
- **两条一句话检验**：
  1. 能说出"**分件是意图条件的生成问题（答案不唯一），不是判别问题**"；
  2. 能说出"**先决定'分出什么'，再解算'边界在哪'** —— 决策前移到语言层，掩码只做分配"。

## Learning Value

- **对生产工具认知的直接更新**：分件从"预处理脚本"升级为"**可点单粒度的语义接口**"，且已由腾讯游戏工具线推动到 SIGGRAPH Asia；
- **方法论可迁移**：**"答案不唯一 → 条件生成"** 这条判据今天同时在动作线（已有）与资产理解线（本篇）得到确认；
- Watchlist 价值：跟踪 Visvise 是否把 PartLLM 产品化（部件级编辑/批量材质/碰撞生成），以及"粒度接口"是否成为资产规范的字段。

## Visualization

（本篇不配图解 —— 核心机制用上方流程文本已足够清晰；如后续需要演示"联合分配 vs 独立掩码"，可再补一张。）

## Notes

- 索引页/标题中的"PartLLM"为方法名；论文同时给出项目页（含 demo 视频）；
- 库内去重说明：9-22 分组中另有 **MoSAT / VISTA**（动作生成线，已记名）；本篇与之无重叠；
- **同日双通道捕获**：本篇于 9-22 提交，listing 属 9-23 分组（9-24 放出）—— 与 [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]] 同批入库。
