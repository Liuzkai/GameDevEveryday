---
type: paper
title: "Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement"
authors: ["Seokhyeon Hong", "Chaelin Kim", "Inseo Jang", "Soojin Choi", "Junyong Noh"]
year: 2026
published: "2026-09-06"
venue: "SIGGRAPH Asia 2026 (Journal Track)"
url: "https://arxiv.org/abs/2609.06517"
code: "https://seokhyeonhong.github.io/projects/kinematic-refinement/"
category: [animation, retargeting, character]
importance: "A-"
game_relevance: "中高"
production_readiness: "Prototype"
user_level: "Hard（[[Neural Animation]] 下游）"
status: unread
---

# Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement

## TL;DR

骨架无关的神经重定向 + **显式穿模检测**：先学一个跨骨架共享的动作先验，再让网络"看着"目标网格上的自穿透，通过 motion-to-vertex Jacobian 把穿模转成运动修正量。显式几何推理管 artifact、学习模块管先验——"分层收敛"框架本周第四次出现。

## Problem

神经重定向已能跨不同骨架迁移动作，但目标侧几何 artifact（自穿透、肢体穿插）始终解决不了。已有几何感知方案要么绑定固定骨架模板，要么要求单个网络同时做变形、检错、纠错——泛化差。

## Core Idea

把"发现 artifact"和"修 artifact"拆开：

```
Transformer 重定向 autoencoder（骨架无关，学运动学先验）
        ↓ 初始动作
摆姿目标网格 → 显式观测自穿透
        ↓
motion-to-vertex Jacobian 把穿模换算成关节修正方向
        ↓
artifact-driven refinement module 输出修正后动作
```

## Technical Approach

1. **运动学先验**：transformer autoencoder 学跨骨架共享 motion embedding，任意 source→target 骨架对可迁移
2. **artifact 驱动修正**：在摆姿后的目标 mesh 上检测自穿透，经 Jacobian 反推运动修正量
3. **几何条件解码**：用 rest pose 的蒙皮权重构造 joint-aligned 几何特征，让解码器"知道"目标体型

## Key Contribution

- 修正信号来自**显式几何观测**而非网络隐式猜测——这是它与已有 geometry-conditioned 方案的本质区别
- 固定骨架与任意骨架两种设定下都提升运动学精度、降低几何 artifact
- 对未见过的目标角色（新体型）仍成立

## Game Development Relevance

- 重定向是动作资产管线的日常痛点（人形 → 怪物/坐骑/Q 版比例），穿模是返工主因之一
- "显式检测器 + 学习先验"的分工模式可直接类比到 VFX：显式规则管约束（预算、穿透），学习模块管先验（风格、质感）
- 与 [[FlexMoGen — Flexible Motion Generation from Language and Style References]] 互补：生成解决"造动作"，重定向解决"搬动作"

## Unreal Engine Relevance

对应 UE 的 IK Retargeter / Control Rig 环节。目前是离线研究原型，短期定位：**重定向 QA 工具**（自动标出穿模帧）比"自动修"更快落地。

## Limitations

- 修正依赖自穿透可观测，非穿透类 artifact（滑步、重心失真）不在框架内
- 实时性未验证，定位为离线管线工具
- 仍需 rest pose mesh 与蒙皮权重，纯骨架资产不可用

## Related Concepts

- [[Neural Animation]]
- [[Motion Matching]]

## Related Papers

- [[FlexMoGen — Flexible Motion Generation from Language and Style References]]
- [[UniMate — One Unified Model to Animate Diverse Skeletons]]（同问题域：骨架无关动画）

## Personal Knowledge State

Current Level: Hard

Reason: 属 [[Neural Animation]] 下游，被 [[Motion Matching]] 瓶颈间接阻塞。但**问题定义你完全能读懂**——穿模检测 + Jacobian 修正都是经典工具。

## Learning Path

不建桥。作为 Watchlist 条目，待 [[Motion Matching]] 升到 Easy 后再回看。

## Notes

"分层收敛"（显式 vs 学习分工）案例累计：Magpie（世界模型）→ WorldParticle（仿真）→ DLSS 5（渲染）→ **本篇（动画）**。第四个子领域确认，见 [[2026-09-10]]。
