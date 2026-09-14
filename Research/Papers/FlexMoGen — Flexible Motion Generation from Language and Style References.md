---
type: paper
title: "FlexMoGen: Flexible Motion Generation from Language and Style References"
authors: ["Kai Weixian Lan", "Bodie Criswell", "Briana Fedkiw", "Zhan Zhang", "Joseph Teran", "Daniel Holden"]
year: 2026
published: "2026-09-07"
venue: "Pacific Graphics 2026"
url: "https://arxiv.org/abs/2609.08032"
code: ""
category: [animation, motion-generation, diffusion]
importance: "A"
game_relevance: "高"
production_readiness: "Research"
user_level: "Normal（Motion Matching 侧）/ Hard（生成侧）"
status: unread
---

# FlexMoGen — Flexible Motion Generation from Language and Style References

## TL;DR

文本管"做什么"，风格参考片段管"怎么做"。**无监督**变分风格编码器（不需要风格标签）+ 文本到动作 latent diffusion，支持长时序、时变、多风格合成。**末位作者 Daniel Holden 是 Learned Motion Matching 的作者**——这是动作生成线与你 Motion Matching 瓶颈之间的直接桥梁文献。

## Problem

文本 prompt 能定义语义内容（"走路"、"挥拳"），但抓不住细粒度风格：节奏、肢体表达、动态张力。此前方法（含 [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]]）依赖离散风格标签或有监督端点，无法泛化到长序列与多风格混合。

## Core Idea

- 给一个文本 prompt + 一段风格参考 clip，生成既保语义内容又复现目标风格的长动作
- 风格编码器是**变分、无风格监督**的——风格空间自己长出来，不需要人工标注
- 通过轻量 adaptation module 调制 latent diffusion，而非重训整个生成器

## Technical Approach

1. 风格编码器与文本到动作 latent diffusion **联合预训练**于统一架构
2. 高效相对位置编码，支持长序列
3. 同时在风格化与非风格化数据集上训练 → 对未见过的 text-style 组合泛化

## Key Contribution

风格控制路线的**第三次迭代，且监督信号持续减弱**：

| 工作 | 风格监督 | 控制粒度 | 长序列多风格 |
|---|---|---|---|
| Motion Style Slider（Cygames） | 端点监督 | 连续滑杆 | 未验证 |
| [[STyMo — Fast and Controllable Few-Shot Motion Style Transfer]] | 分解式少量监督 | 静态/时间双分量 | 未验证 |
| **FlexMoGen** | **无监督** | 参考 clip 直接给 | **支持** |

## Game Development Relevance

- NPC/敌兵动作变体：同一段"攻击"动作，换参考 clip 即换气质——正是量产动作变体最省数据的路线
- 与 [[MotionBricks — Scalable Real-Time Motions]] 互补：MotionBricks 解决运行时速度（2ms），FlexMoGen 解决控制维度
- Daniel Holden 的 Learned Motion Matching 是 [[Motion Matching]] 从 Normal 走向 Easy 的必读文献；本篇展示同一批人如何把"检索式"思想接到"生成式"上

## Unreal Engine Relevance

离线生成动作资产 → 进 Motion Matching 数据库做检索池扩充。与 UE 5.6+ 的 Motion Matching 节点天然衔接：生成的是**数据**，不是运行时系统。

## Limitations

- 无运行时数据，实时性未验证（对照 MotionBricks 的 2ms）
- 物理合理性（脚步滑动、接触）未作为核心指标
- 风格 clip 需要与目标骨架兼容，跨骨架风格迁移未覆盖（那是 [[Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement]] 的问题域）

## Related Concepts

- [[Motion Matching]] ★ 你的瓶颈，本文是桥
- [[Motion Generation]]
- [[Neural Animation]]

## Related Technologies

- [[Real-Time Generative Motion]]

## Related Papers

- [[STyMo — Fast and Controllable Few-Shot Motion Style Transfer]]
- [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]]
- [[MotionBricks — Scalable Real-Time Motions]]

## Personal Knowledge State

Current Level: Normal（Motion Matching 侧）/ Hard（生成侧）

Reason: 风格控制的工程价值你能直接判断；latent diffusion 内部机制属 [[Motion Generation]]（Hard），但**本文不要求你读懂 diffusion**——读"无监督风格编码"这一个设计决策即可。

## Learning Path

1. 先读 Learned Motion Matching（Holden 2020，[[Learning Path — Neural Rendering]] 之外动画线的基础文献）
2. 再看本文§3 风格编码器：变分假设如何替代风格标签
3. 对照 [[STyMo — Fast and Controllable Few-Shot Motion Style Transfer]] 的"分解"路线，判断哪种更适合你的数据条件

## Notes

风格控制线四天内第三篇（9-7 两篇 + 本篇），趋势判断见 [[2026-09-10]] Daily。
