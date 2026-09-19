---
type: paper
title: "LYRIC: Language-Driven Physics-Based Character Control for Contact-Rich Whole-Body Object Interaction"
authors: [Zeyu Han, Zichong Meng, Julian Tanke, Minami Matsumoto, Sergey Bashkirov, Yingruo Fan, Selim Engin, Dongseok Shim, Takashi Shibuya, Yuki Mitsufuji, Huaizu Jiang]
year: 2026
published: "2026-09-17 (v1)"
venue: "arXiv 2609.19688 (cs.GR) — 未见 venue 标注"
url: "https://arxiv.org/abs/2609.19688"
code: ""
project_page: ""
category: [animation, physics-based-character, character-control, flow-matching, contact-rich-interaction, language-conditioned]
importance: A-
historical_importance: 2
game_relevance: 4
production_readiness: Research
user_level: Hard（可只取一条架构）
status: unread
aliases: [LYRIC, Language-Driven Physics-Based Character Control, 语言驱动物理角色控制]
tags: [animation, physics-based-animation, character-control, flow-matching, object-interaction]
---

# LYRIC: Language-Driven Physics-Based Character Control for Contact-Rich Whole-Body Object Interaction

## TL;DR

**这条线的第四篇：前面解决了"学得多快"（[[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]]）、"不用示范"（[[2026-09-14-Learning Realistic Athletic Sprinting Without Demonstrations]]）、"学得多宽"（[[2026-09-17-DSD — Diffusion Skill Discovery]]）；LYRIC 解决的是"**和物体接触**"。**

- 目标：一句自由语言 + 一个稀疏的**末端物体目标**，让仿真角色完成**全身接触密集的操作**（扛、抱、拖、推、顶上物体……）；
- **架构上唯一值得你带走的一条**：把控制器**因式分解**成 **任务级 planner**（预测短时域的物体轨迹 + 人形根轨迹）与 **动作生成器**（在闭环里解全身运动与接触）；
- **训练上唯一值得你带走的一条**：**BC 之后冻结 planner，用 planner 的预测当作"稳定的过程监督"来 on-policy 微调动作生成器** —— 也就是说，**慢的那一层不去追快的，而是给快的那一层当老师**；
- 数据：OMOMO。tracker 成功率 **64.3% vs InterMimic 复现版 53.2%**；统一策略在全量 OMOMO 上 **76.5%**；held-out split 上 **90.3% vs 最强配对运动学 planner 基线 74.2%**（数值来自摘要，未见表格核对）。

## Problem

物理角色动画的两个老大难，在这篇里被合到一起：

1. **参考数据不完美**：动捕数据里的手-物接触通常有穿透/浮空。**直接照抄参考 = 学到一个物理上不成立的策略**；
2. **全身接触密集任务的搜索空间极大**：物体有 6-DoF，接触拓扑会切换，**一维"跟随全身运动学参考"的范式在这里失效**（你没法预定"全身该怎么动"）；
3. **稀疏终端目标 + 语言** → 中间过程完全欠约束。

## Core Idea

### 1. 因子分解：慢层规划 / 快层执行

```text
语言指令 + 末端物体目标
        ↓
【任务级 planner】预测短时域：物体轨迹 + 人形根轨迹        ← 慢、粗、只管"去哪"
        ↓
【动作生成器】闭环解算：全身运动 + 接触                     ← 快、细、只管"怎么动"
```

**为什么这个切法重要**：如果你把"全身运动学参考"换成"根轨迹 + 物体轨迹"，那么**中间的自由度就是设计者主动留出来的**，而不是被参考数据锁死的。原文的表述是：为了引导交互进展而**不规定全身运动学参考**，才做了这个分解。

### 2. 用"放宽参考"从脏数据里掏出干净专家

从**不完美动捕**里拿到可靠专家轨迹的办法：训练单个 **tracking policy**，用

- **几何条件化的交互奖励**（geometry-conditioned interaction rewards）——让接触在几何上说得通；
- **在手-物接触附近放宽参考跟踪**（relaxed reference tracking near hand-object contact）——**接触处不硬跟，别处照跟**。

**"在你知道参考不可信的地方放宽约束"** —— 这是本篇最可迁移的一句工程判据，和渲染侧"在你知道近似会错的地方承认误差"是同一类思维。

### 3. 冻结慢层，用它给快层当监督

BC 之后：

- **冻结 planner**；
- 用 planner 的预测作为 **"stable supervision for intermediate task progression"** 来 on-policy 微调动作生成器。

**这一步在概念上很干净**：稀疏终端目标本身不提供中间过程信号；planner 提供了**一个自产的过程信号**，而且是**冻结的**（不随策略漂移 → 不会出现"老师跟着学生一起跑偏"）。

### 4. 白送的泛化

**不重新训练**即可支持 **test-time object waypoint guidance** —— 因为你把"物体轨迹"放在了架构的接口层，改它不需要改网络。

## Key Contribution

1. 把"语言 + 稀疏物体目标 + 全身接触密集操作"这个组合做成可用系统；
2. **planner/generator 分解 + 冻结慢层当监督** 这套训练方案；
3. 用"几何奖励 + 接触处放宽参考"从脏动捕里造出可用专家；
4. 架构层面把"物体轨迹"暴露成可运行时干预的接口。

## Limitations

- **接触密集任务的成功率仍不到 100%**（held-out 90.3% 是最好的那个数）——这是仿真里的成功率，**与"能进游戏"之间还隔着实时性与稳定性两关**；
- 未见推理延迟 / 训练成本 / GPU 型号（与 [[2026-09-17-DSD — Diffusion Skill Discovery]] 同一个缺口 —— **最近三篇物理动画论文都不给实时数据**，这本身是一条值得记录的趋势）；
- 语言的理解深度未在摘要中说明（是"指令"还是"意图"？）。
- 未见项目页/代码。

## Game Development Relevance

- **不构成你的行动项**（与 [[Motion Matching]] 同一处置：静默项）。它是 Research 阶段的仿真控制研究；
- **但给出一条设计判据**：如果你的动画系统里既有"规划层"（哪里去、做什么）又有"执行层"（具体动作），**要防止两层互相追**。LYRIC 的方案是"冻结哪一个、用谁监督谁" —— 这在任何分层动画架构（包括状态机 + Motion Matching 的组合）里都是同一个问题；
- **对 VFX 侧的一个类比（仅供参考，非原论文观点）**：接触密集任务的处理方式 —— **"在几何上说得通的地方用物理约束，在参考不可信的地方放宽"** —— 与特效里"告诉美术哪里必须准、哪里可以糊"是同一类预算分配。

## Unreal Engine Relevance

- 概念上映射 **Animation Blueprint（慢层）+ Control Rig / Physics（快层）** 的分层，以及 **Chaos 物理驱动动画**；
- 接触密集的操作（props 交互、拾取放下、搬运）是 NGR 级项目里的真实内容 —— 但本方法是离线学策略，**当前没有 UE 落地路径**。

## Technology Evolution

```text
物理角色动画的四个问题，2026-09 一个月内被四篇各解决一个：

"学得多快"   InstantMimic（9-11）      全 GPU 训练环路，秒级学技能
"不用示范"   Sprinting（9-14）          零示范学奔跑
"学得多宽"   DSD（9-17）               扩散估计熵梯度 → 技能库变宽
"能和物体接触" LYRIC（9-18）★ 今日入库    planner/generator 分解 + 触处放宽参考
```

**一条结构性观察**：这四篇的共同形状是**"把一个大问题切成两个可分别解决的小问题，然后规定谁服从谁"**。
InstantMimic 切"仿真 / 训练"；DSD 切"多样性 / 可复用性"；LYRIC 切"任务规划 / 动作执行"。
**"分层 + 定序" 是 2026 年物理动画的共同解法（也是 Magpie / DLSS 5 / Karis 在渲染侧的同一个解法）。**

## Relationships

### Based On

- **[[2026-09-17-DSD — Diffusion Skill Discovery]] / [[2026-09-14-Learning Realistic Athletic Sprinting Without Demonstrations]] / [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]]**：同一条线的前三篇（"多快 / 不用示范 / 多宽"）；
- **flow matching** 作为动作生成器：与 [[2026-09-17-EMODY Flow — Emotion-Aware Audio-Driven Full-Body Motion Generation|EMODY Flow]] 同族。

### Improves

- **InterMimic**（原文的对照基线，**未入库**）：LYRIC 的 tracker 在 OMOMO 上 64.3% vs InterMimic 复现版 53.2% —— **注意这是"复现版"**，对比公平性需看原文补充材料，摘要里没有交代。

### Related

- [[Physics-based Character Animation]] — 本文是该概念下的第四篇样本；
- [[Motion Matching]] — 维持静默项，**不构成行动项**（同 9-18 的处置）。

## Personal Knowledge State

**Hard，但今天你可以只取一条：**

> **"把慢的那一层冻结，让它给快的那一层当老师。"**

这条抽象**不需要 RL 基础、不需要 flow matching 基础**，而且可以直接映射到分层动画架构的设计讨论里。**其余部分（接触奖励、放宽参考的具体形式）标为 Hard 未读。**

## Learning Value

- 一条可迁移的**训练/架构**判据：**欠约束的中间过程需要一个"自产但冻结"的监督源**，而不是让两层互相追；
- 一条可迁移的**数据**判据：**在你确知数据不可信的那一小块区域放宽约束**，比整体降低约束更划算。

## Visualization

—

## Notes

- 作者机构推断：Northeastern University（Han / Meng / Jiang）+ Sony AI（Shibuya / Mitsufuji 一路）——**未从原文逐条核对，仅按署名习惯判断**；
- 摘要未见显存的实时性数据，**故不进入任何"能否进实时管线"的判断**；
- 与 [[2026-09-17-DSD — Diffusion Skill Discovery]] 一起构成了"物理动画论文集体缺实时数据"的一个小样本 —— 已在今日 Daily 的趋势段记录。
