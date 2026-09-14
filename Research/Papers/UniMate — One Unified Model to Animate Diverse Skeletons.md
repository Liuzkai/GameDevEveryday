---
type: paper
title: "UniMate: One Unified Model to Animate Diverse Skeletons"
authors: ["Linzhan Mou", "Jiahui Lei", "Zhiyang Dou", "Chenyue Cai", "Chaoyue Song", "Adam Finkelstein", "Szymon Rusinkiewicz"]
year: 2026
published: "2026-09-04"
venue: "SIGGRAPH Asia 2026"
url: "https://arxiv.org/abs/2609.05415"
code: ""
project_page: "https://linzhanmou.com/unimate/"
category: ["Animation", "Motion Generation", "Rigging"]
importance: "A"
game_relevance: "High"
production_readiness: "Research"
user_level: "Hard"
status: unread
tags: [animation, skeleton, diffusion-transformer, topology, zero-shot]
---

# UniMate: One Unified Model to Animate Diverse Skeletons

## TL;DR

已有学习型动画生成器都是**拓扑受限**的：要么依赖分类别模板，要么推理时需要 per-skeleton 微调或参考动作。UniMate 是一个统一基础模型：输入一个已绑定的 3D 资产 + 文本提示，直接生成任意骨架的关节动画，**无需测试时优化、无需 per-skeleton 重训**。配套 UniML3D 数据集（13,006 段动作，覆盖双足/四足/鸟类/海洋/昆虫/蛇形/ articulated rigid objects）。

## Problem

自动绑定（auto-rigging）已经能规模化产出 animation-ready 资产，但"给它们配动作"仍是瓶颈。人的"膝盖"能和另一个人的膝盖对应；换成蛇、翅膀或机械连杆，这套对应关系就失效。已有方案要么每个物种单独训，要么测试时先优化新骨架表示。

## Core Idea

**把骨骼拓扑本身编码进注意力机制**，让一个模型理解任意运动学树。

## Technical Approach

拓扑感知扩散 Transformer（topology-aware diffusion transformer），三个机制：
1. **Graph-aware attention bias**：由关节两两关系与测地距离构造
2. **Spectral rotary position embedding**：用图拉普拉斯把 RoPE 推广到任意运动学树
3. **Global topological conditioner**：从 rest-pose 骨架 attention-pool 出来的全局拓扑条件

能力：zero-shot 跨拓扑迁移、in-betweening、expansion、文本引导编辑。

## Key Contribution

- 首个真正意义上的"任意骨架"统一动画基础模型
- UniML3D 数据集（13k 序列，统一 canonicalization + 文本配对）
- 质量、泛化、效率均超 SOTA

## Game Development Relevance

- 对**非人形角色 / 怪物 / 载具 / 机械**的动作生产价值最高——这类资产在开放世界项目里数量庞大但 mocap 数据稀缺
- 跨拓扑迁移意味着：人形 mocap 数据可以迁移给四足怪
- 但游戏技能动作要求**帧级确定性**（打击帧、受击帧、无敌帧），生成式方案在此处仍是硬伤

## Unreal Engine Relevance

- 输出为 skeletal animation，理论上可导出为 anim sequence
- 与 Control Rig / IK Retargeter 的关系：更像是"生成源数据"，下游仍需 retarget + 精修 + 物理校验

## Limitations

- 精细空间控制仍有限
- 扩散模型推理成本——未报告实时性能，与 [[MotionBricks — Scalable Real-Time Motions]] 的 2ms 不在同一量级
- 无游戏引擎集成报告

## Related Concepts

- [[Neural Animation]]
- [[Motion Generation]]
- [[Skeleton Topology]]
- [[Motion Retargeting]]

## Related Technologies

- [[Real-Time Generative Motion]]

## Related Papers

- [[MotionBricks — Scalable Real-Time Motions]]
- [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]]
- [[STyMo — Fast and Controllable Few-Shot Motion Style Transfer]]

## Personal Knowledge State

Current Level: **Hard**

Reason:
图拉普拉斯谱位置编码、扩散 Transformer 这两块都是明确前置缺口。

## Learning Path

桥：
1. Transformer 与注意力（若未掌握）
2. 扩散模型基本原理
3. 图上的位置编码（谱方法）
4. 本文

## Notes

2026-09-07 首次收录。Tier A，但 Hard 且桥较长 → 进 Watchlist，不进今日主动学习队列。
