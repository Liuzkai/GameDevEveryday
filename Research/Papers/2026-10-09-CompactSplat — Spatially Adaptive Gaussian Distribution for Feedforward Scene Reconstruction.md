---
type: paper
title: "CompactSplat: Spatially Adaptive Gaussian Distribution for Feedforward Scene Reconstruction"
authors: [Bing He, Jingnan Gao, Yunuo Chen, Zhengxue Cheng, Rong Xie, Li Song, Wenjun Zhang]
year: 2026
published: "2026-10-09（arXiv v1, 2610.13177）"
venue: "NeurIPS 2026（第 40 届神经信息处理系统大会；单位经 PDF 首页核对：Shanghai Jiao Tong University；未公告先见：API 通道 2026-10-12 捕获）"
url: "https://arxiv.org/abs/2610.13177"
code: ""
project_page: ""
category: [gaussian-splatting, feedforward-reconstruction, primitive-allocation, sparse-view, memory-budget]
importance: B+
historical_importance: 0
game_relevance: 3
production_readiness: "Research→Prototype（NeurIPS 2026 接收；推理时稀疏度控制已演示；代码未提及）"
user_level: "Normal（结论层：内容自适应分配 + 推理时稀疏度旋钮）"
status: unread
aliases: [CompactSplat, 空间自适应高斯分布, 内容感知分配]
tags: [gaussian-splatting, feedforward, allocation, memory-budget]
---

# CompactSplat: Spatially Adaptive Gaussian Distribution for Feedforward Scene Reconstruction（He et al. 2026）

> **入库 2026-10-12（Run 34）。** **上海交通大学**（Bing He / Jingnan Gao / Yunuo Chen / Zhengxue Cheng / Rong Xie / Li Song / Wenjun Zhang——单位经 PDF 首页逐字核对）。**NeurIPS 2026**。经 **API"未公告先见"通道**捕获。
> **一句话定位**：**前馈式 3DGS 重建的"内容感知分配"**——不再"每像素一到多个高斯"（均匀分配 = 平坦区浪费 + 复杂区分不够），而是**按区域复杂度分配**：复杂/纹理富集区放密的小高斯，平坦区用"放大的、尺度调制的高斯"铺设；并**在推理时提供动态稀疏度控制**（视觉保真 ↔ 内存占用的连续取舍）。
> **库内位置**：GS"分配三连"之生成端样本（SplatQ = 保留端分配 / **本篇 = 生成端分配** / H2-Ray = 执行端均衡）；**"档位旋钮"家族新增"推理时稀疏度"**（[[2026-09-25-ControlGS — Conditioning Neural Gaussians for Downstream-Processing-Aware XR Rendering|ControlGS]] 的运行时条件化 / [[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]] 的单参数 c 扫）。

## TL;DR

**病灶（原文）**："前馈方法通常给**每个像素**分配一到多个高斯"——uniform allocation 的两个代价：
1. **冗余**：平坦区（天空、墙面）疯狂浪费 primitives；"wasting primitives in homogeneous areas"；
2. **结构损失**：没有利用平面表面的几何先验（"fails to exploit the inherent geometric priors of planar surfaces"）→ 结构建模次优。

```text
CompactSplat = 内容感知分配（content-aware allocation）

· texture-guided spatial partitioning（纹理引导的空间划分）
  + hierarchical sampling（层级采样）
· 复杂区：密集小高斯（concentrating dense Gaussians in complex regions）
· 平坦区：放大的、尺度调制的高斯"铺砖"（expanded, scale-modulated primitives）
· 推理时：dynamic sparsity control ——
  在不重训的情况下扫"稀疏度"，得到 视觉保真 ↔ 内存占用 的连续取舍曲线
· 结果：高保真重建 + 大幅更少的 primitives（"substantially fewer primitives"）
```

## Problem

- **稀疏视角、无逐场景优化的前馈重建**（feedforward，泛化式 3DGS）：要同时恢复**准确几何**与**细纹理**；
- **均匀分配是错的先验**：图像像素均匀 ≠ 信息均匀——像素级 1:N 分配把"采样密度"绑在"像素分辨率"上，而不是绑在**内容复杂度**上；
- 前馈式与优化式（per-scene optimization）不同：**推理时算力/内存约束直接落在网络输出上**——输出多少 primitives、如何分布，是前馈路线的核心设计变量。

## Core Idea

**"把 primitives 当预算、把复杂度当需求"——分配跟着内容走**：

```text
优化式 3DGS：自适应的是"优化过程"（致密化/剪枝）
前馈式 3DGS：自适应必须发生在"一次前向的输出"里 —— 本篇做的事

分配律：
  复杂/纹理富集区 → 密 + 小（多个小高斯捕捉高频）
  平坦区       → 疏 + 大（放大高斯"铺砖"，用尺度换数量）
  —— "用尺度换数量"（scale-modulated）是平坦区省预算的关键机制
```

**推理时可调**：dynamic sparsity control 在不重训的前提下，让同一模型输出不同稀疏度的表示——"seamless trade-offs between visual fidelity and memory footprint across **diverse deployment constraints**"（部署约束 = 档位语言的直接表达）。

## Technical Approach（按摘要与 PDF 首页可核部分）

- **texture-guided spatial partitioning**：按纹理信息把场景划分（哪里需要密）；
- **hierarchical sampling**：层级采样的粒子分布；
- **两种 primitive 形态**：dense small（复杂区）/ expanded scale-modulated（平坦区）；
- **推理时稀疏度控制**：同一网络的输出可按目标稀疏度调节（连续取舍）；
- **评测**：摘要级结论——"high-fidelity reconstruction with substantially fewer primitives"；Figure 1 给出 "Inference-time Controllable Primitive Number" 曲线（保真度-数量曲线）；
- 注：comments 标注关键词含 "2DGS"（方法可能构建在 2D Gaussian / 平面先验之上——正式 PDF 细节放出后补）。

## Key Contribution

- **前馈式分配机制**：把"内容复杂度"变成分配依据（对照优化式的致密化/剪枝——**同一目标在前馈架构里的等价动作**）；
- **推理时稀疏度旋钮**：部署约束（内存/带宽）与保真度的连续权衡在不重训下可得——**"档位 = 推理时参数"的第 N 个样本**；
- "尺度换数量"的平坦区策略：与"误差 ∝ N^(-1/2)"（[[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]] 引 Zador 律）互补——**后者说"多粗能省多少"，本篇说"省下来的份额该给谁"**。

## Why It Works

- 平坦区的大尺度高斯 = **少量 primitive 覆盖大面积**（面积 ∝ 尺度²，一个顶多个）；复杂区小高斯 = 频率匹配（重建高频需要空间采样密度）；
- 与从图像到高斯的**前馈回归**天然契合：网络输出"每个 primitive 的位置/尺度"时，纹理丰富度是可从图像特征直接读出的信号。

## Limitations（以现有信息可判的部分）

- 复杂区/平坦区的划分粒度与失败模式（细薄结构、半透明）未在摘要展开；
- 稀疏度控制的具体机制（一次前向多输出？重采样？）待 PDF；
- 与下游任务（分割、编辑）的兼容性未提及；
- 动态场景未涉及（static scene reconstruction）。

## Game Development Relevance

- **GS 资产的"预算曲线"形态再 +1**：可交付"保真度-内存"连续曲线的资产（引擎按设备档位取点）——与用户的档位体系语言一致；
- **"分配跟着内容走"的直接借鉴**：特效/资产的资源分配不应按"均匀网格"而应按"复杂度/可见性"——与 [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)|Clark 1976]] 的"可见复杂度"同一条历史主线（成本 ∝ 内容，不 ∝ 全域）。

## Unreal Engine Relevance

- 未涉及引擎集成；长期可想象：GS 内容生产管线中的"按目标设备的向量数/内存预算直接生成"（资产侧一次性前馈，替代逐设备重优化）。

## Technology Evolution

```text
前馈式 3DGS 分配线：
  2023-24 通用化 3DGS：每像素 1:N 高斯（uniform）
  → 2026-10 ★ CompactSplat：内容自适应分配 + 推理时稀疏度旋钮
  对照（优化式侧）：
  · 3DGS 原版：致密化 + 剪枝（启发式）
  · Budgeted-GS：LOD 因子树 + 单参数预算
  · SplatQ：区域配额 + 联合选择（QUBO）
  ——"分配"问题在两条路线（优化式 / 前馈式）并行演化，2026-10 首次同月同框
```

## Relationships

### Based On

- 前馈/泛化式 3DGS 重建（feedforward generalizable reconstruction；comments 关键词含 2DGS——平面先验家族）。

### Extends

- **像素级均匀分配范式**：从"分辨率驱动"扩展为"内容复杂度驱动"；
- **GS 的档位语言**：把"运行时 LOD"（Budgeted-GS）前移为"**生成时即按预算分配 + 推理时旋钮**"。

### Related

- [[2026-10-09-SplatQ — Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection|SplatQ]]（同日入库）——**保留端分配**的姊妹样本；本篇是**生成端分配**；
- [[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]]——连续预算曲线 + 单参数扫（运行时段）；本篇 = 前馈段；
- [[2026-10-04-SteadySplats — Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering|SteadySplats]]——"档位 = 采样数"的另一种旋钮。

### Followed By

- 待观察：代码/项目页；与 4D/动态场景的扩展；引擎侧按预算生成的采用。

## Personal Knowledge State

- Normal（结论层 20 分钟：两区两策略 + 推理旋钮；"用尺度换数量"是最好记的一句）。

## Learning Value

- **"平坦区用尺度换数量"**：预算压缩的一条具体技术（对 VFX/资产同理：远场/平坦信息用低密度大单元覆盖）——可写进分档方法的候选工具箱；
- **"分配依据 = 内容复杂度，不是网格均匀度"**：与 [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)|Clark 1976]]（可见复杂度）/ [[2026-10-09-SplatQ — Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection|SplatQ]]（区域配额）同日三度出现——**"内容感知分配"正在成为 2026-10 的高频词**。

## Visualization

![[分配三连_SplatQ、CompactSplat、H2-Ray 图解]]

## Notes

- 单位核对：PDF 首页（Shanghai Jiao Tong University，全体作者同学术单位）；
- **未公告先见记录**：API 通道 2026-10-12 捕获（提交 10-09，listing 不可见）。
