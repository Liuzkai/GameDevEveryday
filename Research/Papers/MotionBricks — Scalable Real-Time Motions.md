---
type: paper
title: "MotionBricks: Scalable Real-Time Motions with Modular Latent Generative Model and Smart Primitives"
authors: ["NVIDIA", "ETH Zürich", "Simon Fraser University", "UT Austin"]
year: 2026
published: "2026-07"
venue: "SIGGRAPH 2026 (ACM TOG) / Emerging Technologies"
url: "https://blogs.nvidia.com/blog/siggraph-news-2026/"
code: "https://github.com/ (GR00T Whole-Body Control / MotionBricks)"
project_page: ""
category: ["Animation", "Motion Generation", "Real-Time", "Production"]
importance: "S"
game_relevance: "Critical"
production_readiness: "Prototype"
user_level: "Normal"
status: unread
tags: [animation, motion-generation, real-time, generative, unreal-engine]
---

# MotionBricks: Scalable Real-Time Motions

## TL;DR

一个实时动作基础模型：35 万+ 动作片段训练（约 700 小时、9300 种技能、163 名演员），**15000 FPS 吞吐 / 2ms 延迟（RTX 5090）**，单次前向传播出姿态 token（无迭代扩散采样）。上层提供"Smart Locomotion"（速度+朝向+风格 → 关键帧）和"Smart Objects"（接近与接触物体）两种人类可控原语。同一个模型既驱动 UE5 角色，也驱动真实的宇树 G1 人形机器人。

## Problem

Motion Matching 统治了游戏动画 20 年：一个巨大的 mocap 库，运行时检索+混合。它只能**重放**数据库里有的东西——遇到没录过的过渡就糊。状态机 + 过渡的维护成本随内容量爆炸。

## Core Idea

**把"检索"换成"生成"**，但生成必须快到能塞进引擎控制回路。为此放弃迭代式扩散采样，改用离散 token 单次前向预测。

## Technical Approach

- **下层（Modular Latent Backbone）**：
  - tokenizer 把动作压成离散 token
  - 150M 参数 Transformer，**单次 forward** 预测 pose token
- **上层（Smart Primitives）**：
  - Smart Locomotion：velocity + heading + style → 关键帧
  - Smart Objects：从接近到接触的完整交互动作
  - 用户给少量关键帧/意图，模型补齐中间所有帧
- 已支持 retarget 到 UE5 角色
- Jetson Orin 上单 pass 5ms

## Key Contribution

1. 生成式动作模型首次达到**游戏引擎级延迟**（2ms）
2. 分层设计把"生成能力"和"美术可控性"解耦（Smart Primitives 是关键）
3. 同一模型驱动虚拟角色与真实机器人——动作的物理正确性得到硬件验证

## Game Development Relevance

- 直接威胁 Motion Matching 的地位：更少的录制片段、更少的过渡维护、**能生成没录过的动作**
- 对技能动画制作流程：从"拍 mocap + 编状态机"可能转向"给关键帧意图 + 生成 + 精修"
- 但：运行时开销虽然低（2ms），**集成复杂度高**，且输出的动作仍需走 QA / 物理校验 / 与打击判定对齐

## Unreal Engine Relevance

- 已有 UE5 retarget demo
- 对应模块：Animation Blueprint / Motion Matching (Distance Matching) / Control Rig
- 对技能的潜在影响：技能动作与 VFX 的**时间轴对齐**——如果动作变成运行时生成，VFX 的 Notify 触发点就不能再硬编码在固定帧上

## Limitations

- 目前是 early preview：公开代码只有轻量 G1 demo + 合成训练管线
- 完整版（GR00T Whole-Body Control 集成 + 完整训练管线）尚未发布
- 35 万片段规模的三方复现能力存疑
- 精细空间控制仍有限（与 [[UniMate]] 类似的问题）
- 2ms 是 RTX 5090 上的数字，主机/移动端完全不适用

## Related Concepts

- [[Motion Matching]]
- [[Neural Animation]]
- [[Motion Generation]]
- [[Motion Tokenization]]

## Related Technologies

- [[Real-Time Generative Motion]]

## Related Papers

- [[UniMate — One Unified Model to Animate Diverse Skeletons]]
- [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]]

## Personal Knowledge State

Current Level: **Normal**

Reason:
你理解游戏动画管线与技能动画和 VFX 的耦合（Notify/时间轴），但不一定深入 Motion Matching 的内部（特征、检索、混合）。MotionBricks 的"生成替代检索"这一层需要先理解 motion matching 的瓶颈才成立。

## Learning Path

1. 先把 [[Motion Matching]] 从 Normal 推到 Easy：能说清它为什么需要海量片段、为什么过渡难做
2. 再理解离散 token + 单次前向 vs 扩散迭代的差别（与 [[DLSS 5 — Generative Neural Rendering]] 的 one-step 是同一类思路）
3. 最后评估：NGR 的技能动画是否适合生成式路线（MOBA 系技能对**确定性**和**打击帧精度**要求极高，这是生成式方案的硬伤）

## Notes

2026-09-07 首次收录。对你而言的战略意义在于：**技能 VFX 的时间锚点**。如果动作从"固定片段"变成"运行时生成"，VFX 的时序预算就必须从"对齐固定帧"改为"对齐语义事件"。这一点值得提前想。
