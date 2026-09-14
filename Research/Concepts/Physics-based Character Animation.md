---
type: concept
user_level: Hard
---

# Physics-based Character Animation

## Definition

用物理仿真（刚体动力学 + 接触）驱动的角色动画范式：动画不是播放出来的，而是控制策略在物理世界里"演"出来的。与运动学方法（[[Motion Matching]] 检索、[[Motion Generation]] 生成）的根本区别：动作天然满足动力学约束（平衡、接触、碰撞），不需要后处理修穿模。

## Core Principle

```text
参考动作数据
      ↓
模仿学习目标（DeepMimic 式 RL）
      ↓
物理仿真环境中的控制策略
      ↓
动力学可行的动作输出
```

关键难点从来不在"效果好不好"，而在**训练成本**与**控制策略的泛化**。

## Prerequisites

- [[Motion Matching]]（运动学对照系，先理解检索式为什么不需要物理）
- 刚体动力学基础（Easy 侧工程直觉可覆盖）
- RL 基础词汇（当前缺口）

## Evolution

- DeepMimic（2018，模仿学习范式奠基）
- ASE / AMP 系列（对抗式运动先验，NVIDIA/Jungdam Won 线）
- [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]]（2026，训练成本压到秒级）

## Related Concepts

- [[Neural Animation]]（神经运动学路线，平行阵营）
- [[Motion Generation]]
- [[Neural Physics Simulation]]（更一般的学习式物理，反向问题）

## Game Applications

- 物理可信的 NPC 交互动画（推拉、受击、搬运）
- Ragdoll 之上的主动式物理角色
- 体育/格斗类游戏的接触密集场景

## Important Papers

- [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]]
- [[2026-09-14-Learning Realistic Athletic Sprinting Without Demonstrations]]（零示范路线：肌肉驱动 + 任务奖励，同周与 InstantMimic 双双跨过训练成本门槛）

## Personal Knowledge

Current Level: Hard

## Learning Gap

缺 RL 基础词汇与策略学习直觉；工程侧（物理仿真本身）不缺。

## Next Learning Step

不急。桥顺序：先把 [[Motion Matching]] 推到 Easy（理解运动学阵营的极限），再看物理阵营解决的是哪类运动学解决不了的问题。当前只需记住一句话：**物理路线用训练成本换动力学正确性，InstantMimic 把这个成本打掉了几个数量级。**
