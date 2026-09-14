---
type: concept
title: "Motion Generation"
user_level: Hard
tags: [animation, generative]
---

# Motion Generation

## Definition

不检索已有片段，而是**合成**新的动作序列。与 [[Motion Matching]] 的根本区别在于：能否产生数据库中不存在的动作。

## Core Principle

条件生成：`P(动作 | 条件)`，条件可以是文本、轨迹、关键帧、音频、物体位置等。

三种主流实现路线：

| 路线 | 代表 | 特点 |
|---|---|---|
| 扩散（迭代去噪） | 多数 2023-2025 工作 | 质量高，慢 |
| 自回归（流式） | [[MotionBricks — Scalable Real-Time Motions]] 类 | 可实时，长程一致性需设计 |
| 离散 token + 单次前向 | MotionBricks | 最快，2ms 级 |

## Prerequisites

- [[Motion Matching]]
- [[Neural Animation]]
- 生成模型基础

## Evolution

```
Motion Matching（检索）
        ↓
Motion Diffusion（文本到动作）
        ↓
可控性增强：风格控制 / 关键帧约束 / 物体交互
        ↓
实时化：token 化 + 单次前向（2026）
```

2026 年的关键变化：**"实时"这条线被打通了**（2ms / 15000 FPS），生成式动画第一次有资格被当作运行时系统而不是内容工具讨论。

## Related Concepts

- [[Neural Animation]]
- [[Motion Tokenization]]
- [[Motion Retargeting]]

## Game Applications

- [[Open World Character Animation]]

## Important Papers

- [[MotionBricks — Scalable Real-Time Motions]]
- [[UniMate — One Unified Model to Animate Diverse Skeletons]]
- [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]]

## Personal Knowledge

Current Level: **Hard**

## Next Learning Step

先 [[Motion Matching]] → Easy，再回到这里。
