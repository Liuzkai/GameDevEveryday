---
type: concept
title: "Neural Animation"
user_level: Hard
tags: [animation, neural]
---

# Neural Animation

## Definition

用神经网络生成、编辑或压缩角色动画的一类方法统称。它不是单一技术，覆盖从"动作数据压缩"到"文本直接生成动作"的很宽谱系。

## Core Principle

把动画看作**序列数据**，用处理序列的模型（Transformer / 扩散 / VAE / 自回归）来建模。核心张力始终是：

```
可控性  ←→  生成自由度  ←→  推理成本
```

游戏工业的特殊要求把这三个角拉得很紧：需要帧级确定性（打击帧/无敌帧/受击帧），而生成模型天生是概率性的。

## Prerequisites

- [[Motion Matching]]（必须先理解被替代的对象）
- [[Skeletal Animation]]
- 序列模型基础（Transformer / 扩散）

## Evolution

```
Motion Matching（检索）
        ↓
Learned Motion Matching（神经辅助检索与压缩）
        ↓
Motion Diffusion（文本/轨迹条件生成）
        ↓
Motion Tokenization + 单次前向（[[MotionBricks — Scalable Real-Time Motions]]，2ms）
```

## Related Concepts

- [[Motion Generation]]
- [[Motion Matching]]
- [[Motion Retargeting]]
- [[Motion Tokenization]]

## Game Applications

- [[Open World Character Animation]]
- 技能动画原型

## Important Papers

- [[MotionBricks — Scalable Real-Time Motions]]
- [[UniMate — One Unified Model to Animate Diverse Skeletons]]
- [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]]

## Personal Knowledge

Current Level: **Hard**

## Learning Gap

1. [[Motion Matching]] 尚未到 Easy
2. Transformer / 扩散基础

## Next Learning Step

**先把 [[Motion Matching]] 推到 Easy。** 这是最短桥，且它本身对你有独立价值。
