---
type: paper
title: "Learned Motion Matching"
authors: ["Daniel Holden", "Oussama Kanoun", "Maksym Perepichka", "Tiberiu Popa"]
year: 2020
published: "2020-07"
venue: "SIGGRAPH 2020 / ACM Trans. Graph. 39(4)"
url: "https://theorangeduck.com/page/learned-motion-matching"
code: "https://github.com/orangeduck/Motion-Matching"
category: [animation, motion-matching, neural, foundation]
importance: "S（基准文献）"
game_relevance: "最高（[[Motion Matching]] 的 learned 变体，UE5 Motion Matching 的直系学术源头）"
production_readiness: "Industry Adopted（2020 年即在 AAA 生产中做了用户研究）"
user_level: "Normal → Easy 桥接材料"
status: read
---

# Learned Motion Matching (Holden 2020)

## TL;DR

把 Motion Matching 拆成三个阶段——**Projection（最近邻检索）、Stepping（索引步进）、Decompression（查表取姿态）**——用三个小网络（Projector / Stepper / Decompressor）逐一替换，扔掉特征库 X 和动画库 Y，内存从随数据线性增长变成恒定（Bear 场景 995.6 MB → 7.1 MB），行为几乎不变。

## Problem

MM 的内存（和部分运行时开销）随数据量线性增长，表现力与内存预算永远打架；而纯生成式模型（PFNN/MANN）难控制、难调试、训练久、质量常不如数据本身。

## Core Idea

![[LMM_三网络替换_MM_三阶段对照图.html]]

```
MM:  每 N 帧在 X 里搜最近邻 → 索引步进 → 查 Y 取姿态 → 混合
LMM: 每 N 帧过 Projector    → Stepper 推 Δ → Decompressor 解码 → 混合
```

- **Decompressor D**（+训练期的 Compressor C）：(x, z) → y。z ∈ R³² 是 autoencoder 发现的"特征向量里缺的那部分信息"。损失用 FK 角色空间误差 + 速度损失，纯 MSE 会抖。
- **Stepper S**：(x, z) → Δ，自回归训练（窗口 2N 帧），替代索引递增。
- **Projector P**：仿真最近邻。训练技巧是**加噪回归**：x̂ = x + nσ·n（nσ ~ U(0,1)），回归目标为最近邻条目的 (x*, z*)——因此对任意玩家输入鲁棒。

## Key Contribution

- 三个网络可独立训练/调试/开关；Decompressor 单独用就是通用动画压缩器（DMM 折中档）
- **刻意不泛化**（§7.2）：Projector 把用户输入"投影"回训练数据流形，系统状态永远贴近见过的数据——这是它质量稳定、可控的根本原因，也是它与生成式模型的哲学分界线
- Projector 的两个隐藏角色（§7.1）：把非法输入映回数据分布 + 定期"重置"递归，防止 dying-out / 漂移

## Game Development Relevance

- UE5 的 Motion Matching / Pose Search 是这条线的工业后代；读懂本文 = 读懂 UE5 动画检索系统的学术原型
- 动画侧的"预算语言"：MM 内存线性 vs LMM 恒定，与你的 SABC 预算思维同构——**约束单位从"数据量"变成"网络大小"**
- Table 3 完整给出 7 个场景的 内存/帧耗时/训练时长 三列表，是评估"检索 vs 学习"取舍的原始数据

## Limitations

- Projector 精度主导最终质量，也是最大网络；加深层数提质但突破帧预算（§8）
- 两阶段训练，非端到端；不插值生成新动画（库里没有的还是没有）
- 训练需过夜（CPU），而 MM 无需训练——作者的对策：迭代期用 MM，定型后切 LMM 当后处理

## Related Concepts

- [[Motion Matching]] ★ 本论文是它的 learned 变体
- [[Neural Animation]]
- [[Motion Generation]]

## Related Papers

- [[FlexMoGen — Flexible Motion Generation from Language and Style References]]（Holden 2026，生成式一侧的最新位置）
- [[MotionBricks — Scalable Real-Time Motions]]（同一问题的 2026 生成式回答）

## Personal Knowledge State

Current Level: Normal → Easy 桥接材料

Reason: 六个 Mastery Criteria 中四条（搜什么/内存来自哪/库大为何好/何时不如生成式）可由本文 §3+§4+Table 3+§7 直接回答。

## Notes

原文解读记录见 2026-09-10 会话；关键数字：z ∈ R³²、每 N≈10 帧检索一次、Decompressor 平均关节位置误差 1.4 cm、AABB 双层加速结构（16/64 帧组）。
