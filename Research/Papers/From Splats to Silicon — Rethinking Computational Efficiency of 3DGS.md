---
type: paper
title: "From Splats to Silicon: Rethinking Computational Efficiency of 3DGS"
authors: ["Minnan Pei", "Qiwei Dong", "Yihan Zhou", "Gang Li", "Yuchen Zhu", "Wenju Zhao", "Zhongtian Long", "Siting Wang", "Peisong Wang", "Jian Cheng"]
year: 2026
published: "2026-09-05"
venue: "arXiv preprint (cs.AR)"
url: "https://arxiv.org/abs/2609.06157"
code: ""
category: [gaussian-splatting, systems, hardware, performance]
importance: "B+"
game_relevance: "中高（方法论）"
production_readiness: "Research（测量方法论）"
user_level: "Normal"
status: unread
---

# From Splats to Silicon — Rethinking Computational Efficiency of 3DGS

## TL;DR

一篇 3DGS **系统效率的测量学**论文：不复宣称"我们的方法快 X 倍"，而是建立 workload-centric 框架，把表征研究、GPU 运行时、硬件架构三层的优化放到同一把尺子上量，并用受控 GPU profiling 验证。**方法论与你的"技能 × SABC × 实测指标 × 五档画质"框架同构**——这是今天对你预算工作最有直接价值的一篇。

## Problem

3DGS 效率论文各报各的数字：有的算 Gaussian 数量减少，有的算 kernel 时间，有的算端到端帧率。同一项优化在"上游减少的工作量"和"下游实际省下的时间"之间经常对不上——**中间被数据搬运、同步、缓存、梯度/优化器状态吃掉了**。

## Core Idea

```
表征/算法优化（减少 primitive / 计算量）
        ↓ 未必等价于
GPU 运行时收益（kernel 时间、内存流量）
        ↓ 未必等价于
端到端系统收益（帧时间、功耗、平台约束）
```

三层之间要用 **workload 计数 → stage 时间 → 内存流量** 的可追溯链条连起来，才能判断一项优化是否"到达了下游"。

## Technical Approach

1. 文献分析 + **复现测量** + 对选定实现做受控 GPU profiling
2. 把 workload 计数（Gaussian 选择数、屏幕空间工作量、数据移动量）与 stage 时间、内存流量关联
3. 提炼反复出现的 workload 模式（recurring patterns）

## Key Contribution

三条系统级结论（对任何"宣称加速"的神经渲染论文都适用）：

1. **Workload 减少必须传导到下游执行才算数**——上游剪掉的 splat 若被同步开销吃掉，端到端无收益
2. **优化粒度必须匹配每个 stage 的特性**——一刀切的全局优化常常不如 stage-local
3. **隐性成本显性化**：数据传输、同步、缓存结果、梯度、优化器状态——论文级 benchmark 通常不计这些

## Game Development Relevance

**直接映射到你的工作**：

- 你的预算体系里 Particles / DrawCalls / Primitives / OverDraw / CPUTime / GPUTime 六指标，正是"workload 计数 → stage 时间"的同款思路
- 本文第三条（隐性成本）解释了为什么"发射器数达标但帧时间超标"：数据移动与同步不在你当前六指标里——**值得评估是否需要加第七个维度（如显存流量 / buffer 拷贝次数）**
- 评估任何 GS/神经渲染供应商的"加速 X 倍"宣传时，本文给出验收清单

## Unreal Engine Relevance

无直接映射。价值在**评估方法论**：当 Niagara/渲染组引入任何神经渲染组件时，用本文框架验收性能声明。

## Limitations

- 是分析与测量论文，不提供新 SOTA 方法
- 测量集中在桌面 GPU，移动端（Mali/Adreno）的 TBR 架构下的结论是否平移未验证——恰恰是你五档画质里三档移动端的盲区

## Related Concepts

- [[Gaussian Splatting]]
- [[Scalability and Quality Tiers]] ★ 方法论同构
- [[Tile-Based Rendering]]

## Related Papers

- [[Compact Neural Appearance Models for Efficient Gaussian Splatting]]（被测量的那类"算法层优化"）
- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]（同上，GPU 运行时层）

## Personal Knowledge State

Current Level: Normal

Reason: GS 内部机制你在收口（本周建议标 Easy），本文的系统测量方法论与你的预算框架同构——**阅读门槛低，方法论回报高**。

## Learning Path

无需学习路径。建议直接读 §"workload-centric framework"一章（约 20 分钟），带着一个问题读：**我的六指标体系缺不缺"隐性成本"维度？**

## Notes

GS 工程化文献脉络：[[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]（排序）→ [[Compact Neural Appearance Models for Efficient Gaussian Splatting]]（存储）→ [[GradRig — Differentiable Weights for Skinned Gaussian Splat Deformation]]（形变）→ **本篇（测量学）**。算法优化告一段落，社区开始回头建立评估标准——通常是技术从 Assess 走向 Trial 的前兆。
