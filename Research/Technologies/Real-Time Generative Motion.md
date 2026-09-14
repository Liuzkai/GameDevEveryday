---
type: technology
title: "Real-Time Generative Motion"
user_level: Hard
tags: [animation, runtime, generative]
---

# Real-Time Generative Motion

## Overview

把动作**生成**放进引擎控制回路（而不是离线产出动画资产）的技术。2026 年因 [[MotionBricks — Scalable Real-Time Motions]] 达到 2ms 延迟而第一次成为"运行时系统"级别的候选。

## Architecture

典型分层（以 MotionBricks 为例）：

```
上层：Smart Primitives（人类可控接口）
      · Smart Locomotion：速度 + 朝向 + 风格 → 关键帧
      · Smart Objects：接近 → 接触 → 离开 的完整动作
              ↓
下层：Modular Latent Backbone
      · Tokenizer：动作 → 离散 token
      · Transformer（150M）：单次前向预测 pose token（无迭代扩散采样）
      · Decoder：token → 关节运动
              ↓
输出：Skeletal Animation（可 retarget 到 UE5）
```

## Algorithm

关键取舍：**放弃迭代式扩散采样，改用离散 token + 单次前向**。这是"能实时"的唯一原因。

## Performance

| 指标 | MotionBricks 报告值 |
|---|---|
| 吞吐 | 15,000 FPS |
| 延迟 | 2 ms（RTX 5090） |
| Jetson Orin 单 pass | 5 ms |
| 训练数据 | 350,000+ 片段 / ~700 小时 / 9,300 技能 / 163 演员 |

注意：吞吐与延迟是两个不同指标，不能直接比较。

## Memory

未公开。但 tokenizer + 150M 参数 transformer 的权重规模对现代 GPU 不构成问题；真正的内存压力来自**动作库本身**在 motion matching 时代才是大头——生成式反而在这一点上有优势。

## Hardware

- RTX 5090：2ms
- Jetson Orin：5ms
- **主机与移动端：无数据，基本可判定不可用**

## Production Challenges

1. **帧级确定性**：技能动画需要精确的打击帧/受击帧/无敌帧。生成模型是概率性的，这是硬伤。
2. **物理校验**：生成的动作仍需走碰撞、IK、与地面/物体的接触修正
3. **QA**：无法逐帧审查，需要新的自动化测试方法
4. **与 VFX 的时序耦合**：如果动作是运行时生成的，VFX 的 Notify 触发点不能硬编码在固定帧

## Game Engine Integration

- 已有 UE5 retarget demo
- 现实路径：先做**离线生成 + 人工精修 + 导出 anim sequence**，再谈运行时生成

## Unreal Engine Possibilities

- Animation Blueprint / Motion Matching 的替代或补充
- Control Rig 之后接入生成层
- **与 Niagara 的耦合点**：技能动作的语义事件（"挥砍命中"）作为 VFX 的触发锚点，而不是帧号

## Related Concepts

- [[Motion Matching]]
- [[Neural Animation]]
- [[Motion Generation]]

## Related Papers

- [[MotionBricks — Scalable Real-Time Motions]]
- [[UniMate — One Unified Model to Animate Diverse Skeletons]]
- [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]]

## Personal Knowledge

Current Level: **Hard**

## Learning Gap

1. [[Motion Matching]] 未到 Easy（**最大阻塞**）
2. Transformer / token 化的基本原理

## Next Step

**不要先学这项技术。** 先把 [[Motion Matching]] 推到 Easy，这是唯一的前置瓶颈。之后回来看 MotionBricks，判断"生成 vs 检索"在你的场景下谁赢。
