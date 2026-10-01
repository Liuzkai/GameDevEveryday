---
type: concept
title: "Motion Matching"
user_level: Normal
tags: [animation, runtime, foundation]
---

# Motion Matching

## Definition

游戏角色动画的一种**运行时检索与混合**方案：维护一个大型 mocap 片段库，每一帧（或每隔若干帧）根据角色当前状态 + 玩家输入，在库里搜索最匹配的片段并平滑过渡过去。

它是过去 20 年 AAA 动作游戏的主流方案，也是理解 2026 年这波"生成式动画"冲击的**必要前置**。

## Core Principle

```
角色当前姿态/轨迹/速度  ──>  特征向量
玩家期望轨迹            ──>  特征向量
                    ↓
        在 mocap 库里做最近邻搜索
                    ↓
        选中最匹配片段 → 混合过渡 → 播放
```

关键洞察：**它只能重放数据库里有的东西。**

## Prerequisites

- [[Skeletal Animation]]（骨骼、关节、局部/世界变换）
- [[Animation Blending]]
- 特征工程与最近邻搜索（基础）

## Evolution

```
固定状态机（State Machine）/ Move Trees（工业手工图：手工捕获 + 手工混合 + 手工连接）
        ↓
运动图（图形学学术界：自动连接点 + 自动过渡 + 图搜索）★ 2026-10-01 补锚：[[Kovar — Motion Graphs (2002)]]
        ↓
Motion Matching（检索 + 混合）
        ↓
Learned Motion Matching（神经网络辅助检索）
        ↓
Motion Generation（扩散 / 自回归）
        ↓
Real-Time Generative Motion（[[MotionBricks — Scalable Real-Time Motions]]，2ms 延迟）
        ↓
过渡子问题的"图回归"（2026）：[[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph|变长缝合]] 用聚类转移图估计过渡长度——"结构交给搜索、细节交给生成" ★ 2026-10-01
```

## Related Concepts

- [[Neural Animation]]
- [[Motion Generation]]
- [[Motion Retargeting]]
- [[Motion Tokenization]]

## Game Applications

- [[Open World Character Animation]]
- 技能动画
- Locomotion

## Important Papers

- [[Kovar — Motion Graphs (2002)]] ★ **图时代开山（运动合成叙事的第 0 章）**：相似度矩阵局部最小值 = 天然拼接点；固定 ≈1/3 s 混合窗；SCC 剪枝 + 分支定界搜索——**"过渡问题"的最上游形态**（2026-10-01 入库）
- [[Learned Motion Matching (Holden 2020)]] ★ learned 变体基准文献，已解读（2026-09-10）
- [[MotionBricks — Scalable Real-Time Motions]]
- [[UniMate — One Unified Model to Animate Diverse Skeletons]]
- [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]]
- [[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph]] ★ **过渡长度的自适应化**：图路径决定过渡长度（0.3 s 沿用 Kovar 值）；14.5 ms 单次缝合（2026-10-01 入库）

## 过渡问题的三分（2026-10-01 沉淀）

"库里没有的过渡怎么办"现在可拆成三个**可分别回答**的子问题：

| 子问题 | 一代表述 | 现行工具 |
|---|---|---|
| **在哪切** | Kovar 2002：相似度局部最小值 | PoseSearch 的 continuing pose 匹配 / 缝合的簇图 |
| **切多久** | Kovar 2002：固定 ≈1/3 s 窗 | 多为**配置项**（Blend Time）；2026 新解：图路径长度（NMS） |
| **怎么接** | 线性 + slerp 混合 | 混合（低差异）↔ 神经生成（高差异）；对齐落位可由网络预测 |

## Personal Knowledge

Current Level: **Normal**

（初始化默认推断：你理解动画管线与技能/VFX 时序耦合，但未必深入 motion matching 内部）

## Learning Gap

1. 特征向量具体包含什么（轨迹点、速度、朝向、骨骼位置）
2. 为什么过渡难做（库里没有的过渡只能硬混合）→ **2026-10-01 更新**：过渡已可拆为三个子问题（在哪切 / 切多久 / 怎么接，见上表）——2026 前沿给出了"过渡长度自适应"的首个结构化解法（图路径长度）
3. 与状态机相比的真实收益与代价

## Mastery Criteria

能回答以下全部问题后，可标记为 Easy：

- [ ] 说清 motion matching 每帧在搜索什么
- [ ] 解释为什么片段库越大越容易出好效果，以及代价是什么
- [ ] 解释"库里没有的过渡"会怎样失败
- [ ] 说清它对内存的占用来自哪里
- [ ] 判断一个给定项目该不该上 motion matching
- [ ] 说清生成式方案在哪些场景下仍不如它（**帧级确定性**）

## Next Learning Step

先把本概念推到 Easy，再读 [[MotionBricks — Scalable Real-Time Motions]]——否则你无法判断"生成替代检索"到底解决了什么。
