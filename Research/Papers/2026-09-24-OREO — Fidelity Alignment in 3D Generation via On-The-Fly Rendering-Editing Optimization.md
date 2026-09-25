---
type: paper
title: "OREO: Fidelity Alignment in 3D Generation via On-the-fly Rendering-Editing Optimization"
authors: [Zhiyuan Ma, Wenbo Hu, Wang Zhao, Pengfei Wang, Ying Shan, Lei Zhang]
year: 2026
published: "2026-09-24 (arXiv)"
venue: "arXiv preprint (cs.CV)"
url: "https://arxiv.org/abs/2609.29788"
code: ""
project_page: "https://theericma.github.io/oreo/"
category: [generative, 3d-generation, asset, distillation, alignment]
importance: B+
historical_importance: 0
game_relevance: 3
production_readiness: Research
user_level: Normal
status: unread
aliases: [OREO, Render-Edit-Optimize, 3D 生成保真度对齐]
tags: [generative, 3d-generation, asset, distillation]
---

# OREO: Fidelity Alignment in 3D Generation via On-the-fly Rendering-Editing Optimization

## TL;DR

**3D 生成模型的"保真度"不再靠更大的 3D 数据集，而是靠一条自我提升循环：让生成器渲染自己的输出 → 用 2D 编辑模型把它"改好看"→ 把差距蒸馏回生成器。** 腾讯 ARC Lab × 香港理工。

一句话概括循环（Render-Edit-Optimize）：

```text
Rollout（生成器 policy 出 3D）→ Render（渲染源视图 x_src）
  → Reinforced Editing（2D 编辑模型"修"成高保真目标 x_tgt，同时锚住结构）
  → Contrastive Distillation（x_src 当负样本、x_tgt 当正样本，在 latent 空间监督生成器）
  ↻
```

**今天最值钱的是它的消融表**——三种"更省事/更时髦"的监督方式**全部比不训练还差**：

| 监督变体 | CLIP / DINO（vs 不训练基线 0.7613 / 0.7916） |
|---|---|
| **Full OREO（on-policy + latent 对比 + 显式伪目标）** | **0.7834 / 0.8065 ✅** |
| 像素空间 MSE（可微渲染回归） | 0.6617 / 0.6982 ❌ |
| Off-policy rollout（监督固定在旧生成器上） | 0.7279 / 0.7613 ❌ |
| DMD（隐式 score 蒸馏） | **0.5393 / 0.5486 ❌❌** |

## Problem

3D 生成器（Trellis 一类）吃的是 3D 数据，而 3D 数据在规模与外观多样性上远不如十亿级 2D 图像——"**3D generative models may not fully capture the visual detail needed for photorealism**"。怎么把 2D 扩散模型里泡出来的视觉先验，**灌进一个前馈 3D 生成器**（而不是优化单个资产）？

**关键区分（论文自己划的）**：DreamFusion/SDS 一族是"**逐资产**优化"；本文是"**给生成器做后训练**（post-training）"，目标是让以后**每一次前馈生成**都更好。

## Core Idea

### 1. Reinforced Editing（RE）——"改好看"与"不改动"的权衡

直接用 2D 编辑模型不行：要么改得太狠（Qwen-Edit：姿态变形、裁切、背景漂移），要么像 NanoBanana Pro 那样"**跟随参考图太激进，把源视角换成了正面图**"。**监督信号一旦混入视角/结构漂移，就不是"保真度"信号了。**

RE 在 FlowEdit（无反演编辑）基础上加了两个东西：

1. **Source-aware 正则**（保留 source branch）：防止 target-conditioned guidance 造成的**渐进颜色过饱和与结构漂移**（去掉它 ΔCLIP 直接变 **−0.0523**，变负）；
2. **Dynamic noise update**：去掉它编辑效果大幅减弱（ΔDINO 只剩 +0.0089）。

两个超参：编辑比例 **0.75**（0.25 修不动、1.0 毁结构）、编辑步数 **9**。

### 2. Contrastive Distillation —— 把"差距"蒸馏掉

(x_src, x_tgt) 构成天然的负/正对：**不学"x_tgt 长什么样"，只学"从 x_src 到 x_tgt 的方向"**——在 latent 空间用对比目标把保真度差距灌进生成器。这是它区别于"直接回归目标"的核心。

### 3. 结构性选择：on-policy rollout

每轮用**当前生成器**产出样本（on-policy），而不是固定预训练生成器的输出（off-policy）——"**监督必须跟上学生当前访问的状态**"。off-policy 的实验结果（0.7279）证明固定监督会**变陈旧（stale）**，纠正不了学生新走到的状态。

## Why It Works

1. **监督信号的质量 = 上限**：2D 反馈源对比实验（表 1）证明，一个"保真度提升与结构漂移纠缠"的编辑器（NanoBanana：CLIP vs ref 高，但 Mask IoU 只有 **0.6830**）会污染监督。RE 的 Mask IoU **0.9520**——把"改好看"与"别乱动"这两笔账分开了；
2. **对比式 > 回归式**：学"差距的方向"比学"目标的样子"更稳（latent 对比 vs 像素 MSE：0.7834 vs 0.6617）；
3. **on-policy > off-policy**：监督分布与策略分布同步漂移。

## Limitations

- 论文有 `Limitations and Failure Cases`（附录 S5，本次未逐条展开）；
- 依赖一个**足够强的 2D 编辑模型**（用 Qwen-Image-Edit）——编辑器的能力天花板即整个循环的天花板；
- 评测以 CLIP/DINO 相似度 + 用户偏好（OREO 38% 份额）为主，**没有几何精度类指标**（保真度是"像不像参考"，不是"几何对不对"）；
- 训练成本未在正文展开（附录 S4 有计算开销）；**单条消费级产线可用性未验证**。

## Game Development Relevance

**3/5 —— 不在运行时路径上，但在"AI 资产进入管线的瓶颈"上。**

1. **生成资产的"可用性门槛"就是保真度**：游戏管线要的 AI 生成资产，卡在"看起来对不对"这一关。OREO 的方法论（自我提升循环 + 显式伪目标）是这条赛道当前最干净的一个"监督配方"；
2. **对你已有的生成式资产线（[[Generative Rendering]] / 资产生成）是"后处理级"补充**：库内已有"生成什么"（ProxyBuild/Mira-Scene/PartLLM）与"生成怎么加速"（DLSS 5 系）；OREO 补的是"**生成得不够好看怎么办**"——答案是**用一个更强的 2D 先验给生成器当教练**；
3. **RL 语汇进生成式渲染**：on-policy / off-policy / stale supervision —— 这套词汇以前只在游戏 AI（RL）侧，现在出现在资产生成里。**你的"游戏 AI"与"渲染"两条线在词汇层开始合流**（与本库 9-18 LYRIC 的"冻结慢层当老师"、9-24 WorldCrafter 的"蒸馏"同一趋势）。

## Unreal Engine Relevance

- 无直接引擎集成（研究原型 + 开源项目页 `theericma.github.io/oreo`）；
- 若落地进资产管线，形态会是"**外包前的生成质量提升步骤**"，而非引擎内功能。

## Technology Evolution

```text
2022 DreamFusion/SDS —— 用 2D 先验"逐资产"优化 3D（score distillation）
        ↓
2023-25 前馈 3D 生成器（Trellis 等）—— 快，但保真度被 3D 数据上限锁死
        ↓
2026 ★ OREO —— 把 2D 先验从"逐资产优化"升级为"生成器后训练"：
        显式伪目标（编辑过的渲染）替代隐式 score 梯度；
        on-policy 循环替代静态数据集
        ↓
（同期）WorldCrafter / DLSS 5 系：DMD 用于"蒸馏加速"——
        OREO 的消融进一步划出边界：DMD 当监督信号（这里是 3D 生成器后训练）不行
```

## Relationships

### Based On

- **FlowEdit**（无反演图像编辑）—— RE 的编辑底座；
- **Trellis**（前馈 3D 生成器）—— 被后训练的 backbone。

### Contrasts

- **DreamFusion / ProlificDreamer（SDS 一族）**：同为"2D 先验 → 3D"，但那是逐资产优化；本文是生成器后训练（论文自划边界）；
- **DMD（Distribution Matching Distillation）**：本库此前在 [[2026-09-21-WorldCrafter — Consistent Video World Model with Implicit 3D-aware Memory]] 与 [[DLSS 5 — Generative Neural Rendering]] 里以"**蒸馏加速器**"出现；OREO 的消融里 DMD 当"**监督信号**"是全表最差（0.5393）。**两者不矛盾——它界定的是 DMD 的适用边界：加速可以，当老师不行**（记忆点：同一工具在不同角色下的成败差异）；
- **Photo3D**：同为 Trellis 后训练，但它做离线细节增强且不保结构（0.7380 反而低于基线）——**"不锚结构的增强会伤害生成器"的又一实证**。

### Related

- [[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction]] / [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]] / [[2026-09-22-PartLLM — A Unified Multimodal Foundation for 3D Part Segmentation]] —— "结构锚定"母题的又一个落点：**RE 的 source branch 就是把编辑"锚"在源视图上**；
- [[2026-09-18-GestureFAR — Streaming Co-Speech Gesture Generation with Flow Autoregression]] —— freeze-and-distill 的近亲：都靠"更强的老师"给"学生"供监督；区别在 OREO 的老师是 2D 编辑模型、且学生-老师的差距是**在线生成**的。

## Personal Knowledge State

- **user_level: Normal（接口层可读）**：不需要读训练细节；**四条消融结论本身就是可拿走的判据**（它回答的是"怎么给一个生成模型喂监督"——你做 VFX 预算时不会直接用，但你在评估"AI 资产管线成熟度"时天天用）；
- 与你已建立框架的接口：**"表格里的每个数字都要问它是相对什么基线"**——本表的三个失败变体全部低于"不训练"，这条在评估任何 AI 工具宣传时直接可用。

## Learning Value

1. **四条可直接引用的判据**：
   - 监督信号必须与"学生当前状态"同步（on-policy），否则 stale；
   - 显式伪目标 > 隐式 score 梯度（至少在这个任务上）；
   - 像素空间回归有梯度衰减，**latent 对比更好**；
   - **"保真度提升"若与"结构漂移"纠缠，是脏信号**（Mask IoU 是分离器）。
2. **一个通用评估习惯**：看 AI 论文先找"**降级消融**"——把每个组件换成"更省事的替代"看它掉多少。OREO 用四行表把边界画得很清楚。

## Notes

- arXiv 2609.29788（cs.CV 主分类，9-24 提交；双通道捕获——不在 cs.GR recent 页中，由 API `submittedDate` 窗口查到）；
- 全文已下载核对：表 1（2D 反馈源，含 Mask IoU 0.9520 vs 0.6830）、表 2（RE 组件消融）、表 3（3D 主结果 Trellis 0.7613→0.7834）、表 4（设计选择消融）、编辑比例 0.75 / 步数 9、用户偏好 38%、2,396 张 Conceptual Design Dataset 均出自原文；
- 署名：腾讯 ARC Lab（Wenbo Hu、Wang Zhao、Ying Shan；Zhiyuan Ma 双聘）+ 香港理工大学（Lei Zhang、Pengfei Wang）；项目页 https://theericma.github.io/oreo/ 。
