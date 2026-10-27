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
| 块级 状态 + 注意力（2026） | [[2026-10-26-Liu — PASPA Past as State Present as Attention\|PASPA]] | 最久（分钟级）；历史压状态、块内注意力 |

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
        ↓
检索 × 生成的混合形态（2026-10-01）：[[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph|运动缝合]]——
"结构交给搜索（聚类图最短路）、细节交给生成（Transformer 过渡姿势）"
        ↓
长时程：历史不进上下文，进状态（2026-10-27）：[[2026-10-26-Liu — PASPA Past as State Present as Attention|PASPA]]——
"过去压成定长状态（BGDN 块级递归）、当下用块内注意力"；10 s → 60 s 的 FGD 退化被压平
```

**长时程为什么不能靠"把历史塞进上下文"**：PASPA 拿 BEAT2 做了自相关实测——**风格特征超过 100 帧仍然相关**（要记住），**局部运动特征不到 20 帧就衰减**（不必记住）。一条序列里同时存在两个时间尺度差 5 倍以上的量，**统一上下文窗口对前者不够长、对后者纯浪费**。这正好是 [[Motion Tokenization]] 的判据来源：**token 化的单位必须匹配你要记住的那个时间尺度**——逐帧 token 记不住 100 帧的风格，而状态可以（状态是常数大小，与历史长度无关）。

2026 年的关键变化：**"实时"这条线被打通了**（2ms / 15000 FPS），生成式动画第一次有资格被当作运行时系统而不是内容工具讨论。同年下半年重心转向**"能跑多久"**：PASPA 在 600 s 输入上仍是 56.0 s（无 CFG）/ 61.1 s 的线性成本，峰值显存 0.36–0.48 GB——**长时程的成本曲线回到了线性，这是"状态"相对"全注意力"的本质收益**。

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
- [[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph]] —— **检索 × 生成混合形态**：聚类图最短路定"结构 + 长度"，Transformer 生成过渡姿势（2026-10-01 入库）
- [[2026-10-26-Liu — PASPA Past as State Present as Attention|PASPA]] —— **长时程问题的当前最优解**（NeurIPS 2026）：BGDN（块级 Gated DeltaNet）+ BMHSA（块对角注意力），过去进状态、当下用注意力；BEAT2 FGD 3.71 / SHOW 0.00111（−43.9%）；**消融里最有价值的一行：State+State 7.652 比完全不给历史 7.065 还差**——历史给错方式，负收益（2026-10-27 入库）

## Personal Knowledge

Current Level: **Hard**

## Next Learning Step

先 [[Motion Matching]] → Easy，再回到这里。
