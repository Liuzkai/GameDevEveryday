---
type: paper
title: "UniMo: Unifying Human and Animal Motion Generation"
authors: [Zeyu Zhang, Zhiyuan Zhang, Siheng Wang, Yiran Wang, Danning Li, Ian Reid, Richard Hartley]
year: 2026
published: "2026-09-11"
venue: "SIGGRAPH Asia 2026 (Posters)"
url: "https://arxiv.org/abs/2609.12342"
code: ""
project_page: "https://steve-zeyu-zhang.github.io/UniMo"
category: [animation, motion-generation]
importance: B+
historical_importance: 0
game_relevance: 3
production_readiness: Research
user_level: Hard
status: unread
---

# UniMo: Unifying Human and Animal Motion Generation

## TL;DR

用一个**点云表示**绕过骨骼拓扑差异，把文本驱动的动作生成从"人类专用"扩展到"人 + 动物统一"：把参数化骨架转成无参数的点云（关节变成点集），再用**动态采样**给运动活跃的关节分配更多点。配套发布 UniML3D 数据集：145,907 条动作序列 + 433,388 条 caption，**比现有动物动作数据集大 102 倍**。在 UniML3D + HumanML3D / KIT-ML / AnimalML3D 三个公开基准上 SOTA。

## Problem

文本驱动人体动作生成（text-to-motion）近年进展很快，但扩到动物就卡在两处：

1. **拓扑差异**：人的骨架结构标准，动物千差万别（四足/翅膀/长尾），统一建模困难 → 现状是每个物种训一个模型，又贵又碎；
2. **数据规模**：动物动作数据集又小又脏，标注质量撑不起生成模型。

## Core Idea

**不统一骨架，统一表示**。与其设计一个能表达所有物种的"超级骨架"（拓扑统一，难），不如把骨架降级成点云（表示统一，易）：

- 参数化骨架 → 无参数点集：关节位置/运动变成点的位置/运动，拓扑信息被"点密度 + 空间分布"隐式表达；
- **动态采样**：给运动幅度大的关节分更多点——四足动物的腿、人的手臂自然获得更高表示分辨率，静止部位不浪费容量。

## Key Contribution

- 表示层方案：点云化 + 动态采样（解决拓扑问题，不动模型结构）；
- 数据层方案：UniML3D（145,907 motions / 433,388 captions，动物数据 102× 扩容）——**对社区的价值可能大于方法本身**；
- 跨物种单模型 SOTA。

## Limitations

- Poster track——结果规模与消融深度低于正式 track 论文，细节待扩版；
- 点云表示丢掉了骨骼的**约束结构**（骨长不变、关节层级），生成结果的物理合法性与重定向可用性需要后处理兜底；
- 文本驱动生成 ≠ 游戏运行时控制：无条件/文本条件生成与 Motion Matching 的"目标驱动实时检索"是两个问题类；
- 动物数据虽扩 102×，但对长尾物种（昆虫、鱼类）的覆盖未说明。

## Game Development Relevance

- **对 NGR 类项目的直接价值有限**（游戏要的是可控运行时动作，不是文本生成）；
- 间接价值两条：①UniML3D 若开放，是动物 NPC/坐骑动作的训练数据源；②"表示统一替代结构统一"的取舍思路可迁移——你的 Retargeting 线（[[Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement]]）面对的是同一痛点；
- 与 [[UniMate — One Unified Model to Animate Diverse Skeletons]] 构成方法对照：UniMate 走"模型级统一骨架"，UniMo 走"表示级去骨架"——同目标（一套模型驱动多种拓扑）的两条路线。

## Unreal Engine Relevance

- 无直接映射；若数据/模型放出，可作为离线资产生成器接入内容管线（生成 → Retargeting → 动画库），不进运行时。

## Relationships

### Contrasts

- [[UniMate — One Unified Model to Animate Diverse Skeletons]]（同目标不同路线：结构统一 vs 表示统一）

### Related

- [[Motion Generation]]（Hard，文本/语言条件分支）
- [[FlexMoGen — Flexible Motion Generation from Language and Style References]]（语言+风格条件，人类域）
- [[Skinned Motion Retargeting via Artifact-driven Kinematic Prior Refinement]]（拓扑差异问题的工业侧解法）

## Personal Knowledge State

`Hard`（扩散/transformer 生成细节），但**这篇取一句话即可**："动作生成领域正在用'表示层降级'绕开骨骼拓扑统一问题，并顺手把动物数据扩了两个数量级。" 不必深读。

## Notes

- 作者含 Ian Reid / Richard Hartley（阿德莱德，几何视觉元老级），生成 + 几何的混合班底；
- Watchlist：UniML3D 的开放许可与下载方式放出后值得记录。
