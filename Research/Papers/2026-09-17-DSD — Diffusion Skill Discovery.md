---
type: paper
title: "DSD: Learning Diverse and Reusable Motor Skills via Diffusion Skill Discovery"
authors: [Sun Woo Kim, Xue Bin Peng]
year: 2026
published: "2026-09-15 (v1)"
venue: "arXiv 2609.17682 (cs.LG; cs.GR) — 未见 venue 标注"
url: "https://arxiv.org/abs/2609.17682"
code: ""
project_page: "https://youtu.be/QhMs67fvuWk (视频)"
category: [animation, physics-based-character, reinforcement-learning, skill-discovery, diffusion]
importance: A-
historical_importance: 2
game_relevance: 4
production_readiness: Research
user_level: Hard
status: unread
aliases: [DSD, Diffusion Skill Discovery, 扩散技能发现]
tags: [animation, physics-based-animation, reinforcement-learning, diffusion, skill-discovery]
---

# DSD: Learning Diverse and Reusable Motor Skills via Diffusion Skill Discovery

## TL;DR

**这是一条和你熟悉的 [[Motion Matching]] 平行、但机制上更"生成式"的路线：不要录制的动作库，要一个学出来的技能库。**

- 传统技能发现（DIAYN/ASE 一路）的目标函数是**潜变量与状态之间的互信息**，其中"状态分布的边缘熵"这一项在高维控制问题里**算不动**，所以前人都用间接近似——结果是技能库不够宽，"能做的动作"堆在初始位置附近（论文 Figure 8 的根轨迹热力图直接给出这个结论）。
- DSD 的做法：**用一个扩散模型通过 score matching 直接估计"策略诱导状态分布"的熵梯度**，把那一项算出来。
- 一个你要带走的抽象：**"动作库"这件事可以从"录制的片段集合"换成"学出来的潜变量流形"**——而两者在工程上用的都是同一套东西：**生成候选 + 用代价函数挑**（见下 "与 Motion Matching 的对偶"）。

作者是 **Xue Bin Peng（SFU，AMP / ASE / CALM 一路的核心人物）+ Sun Woo Kim**。这不是新阵营，是同一阵营里"把技能库做大"的一步。

## Problem

物理角色动画要的不是单个动作，是**可复用的技能库**——一个策略能演走路、跳、踢、格挡，下游任务只负责"挑技能 + 定目标"。

学技能库的主流目标函数（DIAYN 系）：

$$\max \; I(\mathbf{s};\mathbf{z}) = H\big(\text{state}\big) - H\big(\text{state}\mid \mathbf{z}\big)$$

- 第二项（条件熵）好算：鼓励"同一潜变量→同一行为"，用判别器/编码器就能逼近；
- **第一项（边缘状态熵）难算**：它需要对策略诱导的状态分布做密度估计，在高维人形控制里不可处理。

前人的绕法：在**潜空间**里做近似（DIAYN 的判别器），或用**粗粒度估计器**（APS 的粒子估计）。论文对这条路的判词很直接：

> 这些近似可能无法有效促进状态空间的广泛覆盖，导致技能的行为多样性有限、对下游任务的效用降低。

## Core Idea

把扩散模型当**熵梯度的估计器**用：

- 扩散模型（score-based）天然在对数密度梯度 $\nabla_{\mathbf{s}} \log p(\mathbf{s})$ 上做拟合；
- 于是**不需要密度本身**，只需要 score——这正是熵梯度需要的量；
- 用这个估计出来的梯度去驱动技能发现目标，策略就被推向"覆盖更宽的状态空间"，同时条件熵项负责"每个潜变量行为一致"。

两个工程决定很关键：

1. **潜空间取单位超球面** $\mathcal{Z}=\{\mathbf{z}:\lVert\mathbf{z}\rVert=1\}$，条件似然用 **von Mises–Fisher 分布**参数化（避开归一化常数要积分整个高维状态空间的问题）——这是沿 ASE 的归一化技能编码器参数化；
2. **抖动引起的多样性抑制（Jitter-Induced Diversity Suppression, JIDS）**：论文第 7 节专门处理一个陷阱——**扩散 score 很容易被高频抖动撑起来**，让你在"多样性"指标上赢、在观感上输。修法是用二阶 Butterworth 滤波器（截止 1 Hz、30 Hz 采样、窗口 10 帧，实现细节引自 HTML 版）把抖动从 score 里滤掉。消融显示 JIDS 让根关节 jerk 降约 13%、FID 降约 12%。

> **这是本篇最值得记的一条工程经验（与技能发现无关）：用扩散/生成模型当"评分器"时，先问它有没有把噪声当成信号。**

## Technical Approach

### 角色与网络（HTML 版第 9 节）

| 项 | 设置 |
|---|---|
| 角色 | 34-DOF 人形（无道具）+ 37-DOF 人形（剑盾，右手腕多一个 3-DOF 关节控剑） |
| 状态 | 骨盆线/角速度（局部系）、骨盆高度、6D 旋转表示的局部关节旋转、关节速度、关键关节 3D 位置 |
| 动作 | PD 目标关节旋转（3D 指数映射） |
| 低层策略 | 三层 MLP，隐藏层 1024 / 末层 512，输出高斯动作分布 |
| 价值函数 / 判别器 / 技能编码器 | 同量级 MLP；编码器输出单位向量 |
| **扩散模型** | 两层 Transformer，4 个注意力头，256 隐藏单元，**约 3M 参数**，时间步 $M=50$ |
| RL | PPO + GAE，Adam |
| 运动先验 | AMP 判别器（**可替换**：论文明确说可换 C·ASE 或 SMP） |

数据集（都不是新的，全是既有公开集）：**Reallusion**（角斗士剑盾，187 段约 30 分钟，与 ASE 直接对比）、**LaFAN1**（约 160 分钟日常与表现性动作）、**MimicKit**（27 段共 2.5 分钟，含后空翻这类高动态动作）。

### 对比基线

全部是 **ASE 换技能发现目标**的变体，控制变量做得干净：ASE（DIAYN）、ASE-APS（粒子估计）、ASE-DADS（技能条件动力学）、ASE-METRA（度量感知表示），另与 ExDM 做消融对比。

### 结果（表格数值来自 HTML 版，已核对表 1）

**运动质量与多样性**（FID ↓ / Div-R ↑）：

| 数据集 | ASE | **DSD** |
|---|---|---|
| Reallusion | 2.47 ± 0.37 | **1.64 ± 0.10** |
| LaFAN1 | 3.08 ± 1.21 | **1.88 ± 0.04** |
| MimicKit | 0.61 ± 0.23 | **0.29 ± 0.25** |

三个数据集上 **FID 最低、Div-R 最高**。

**分层控制**（高层策略在冻结低层之上学）：6 个任务里 DSD 拿 3 个第一。最有说服力的是**后空翻**：5 个随机种子中 **DSD 有 4 个学成了**，ASE 是 **0 个**。

**零样本控制**（不训练任何策略，只用预生成的轨迹池 + 适应度函数挑潜变量）：

$$\mathbf{z}^\* = \mathbf{z}^{i^\*},\quad i^\*=\arg\max_i F(\tau^i,\mathbf{g})$$

10 个任务里 **8 个取得最高值**；增益最大的正是 **Jump（0.81 vs ASE 0.31）与 Duck（0.94 vs 0.37）**——**这类"稀有但明确"的行为，恰好最吃技能库的宽度**。

## Why It Works

1. **把"算不动的那一项"换了个可算的等价物**：熵要密度，梯度只要 score，而扩散模型正好是 score 的高质量估计器；
2. **稀有行为不再被均值行为淹没**：覆盖更宽 → 后空翻/下蹲这类不在数据主模态里的行为，才可能被"检索到"，这也是零样本任务增益最大的原因；
3. **条件熵项兜住了可用性**：只求覆盖会得到一堆噪声，条件熵保证"同一个潜变量反复用是同一种行为"（FME 指标 0.60 vs 纯边缘模型 1.08）。

## Limitations（论文自己列的四条，值得原样记住）

1. **潜空间没有语义组织**：不按技能间的语义关系排布，因此**无法直接插值或组合**——"跳+踢"这种组合你指不出来；
2. **零样本控制依赖离线轨迹池里是否恰好录到了合适的行为**：性能上限由池子决定，而不是由策略决定；
3. **只有边缘熵不够**：论文自己的核心论点——"广泛的状态覆盖**不足**以构成有用的技能库"；
4. **去掉 AMP 后运动不自然**（虽然下游回报不低）——**任务回报不能反映运动自然性**，这是所有物理动画工作的通用陷阱。

另外两条本轮未给出的数据，必须诚实标注：**论文正文没给推理延迟、没给训练墙钟时间/GPU 型号**（细节在附录 A）。因此**任何"能不能进实时管线"的判断现在都不能做**。

## Game Development Relevance

**4/5：路线相关度高，但离实时还有明确距离。**

- 这是**离线资产生产**路线：学出的技能库 → 挑潜变量 → 生成动作序列 → 烘焙/蒸馏后才能进游戏。它替代的是**动捕 + 手工状态机**，不是运行时动画系统。
- 真正与你工作相关的两点：**技能库的覆盖率决定了"任意目标动作"的可达性**（零样本那一列就是它的直接度量）；**训练成本被压到单机可承受的量级**（小型网络 + 公开数据集 + PPO），这是 2026 年物理动画的共性变化——继续与 [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]]、[[2026-09-14-Learning Realistic Athletic Sprinting Without Demonstrations]] 同向。

## Unreal Engine Relevance

**不要强行映射**。当前形态下没有 UE 内可落地的路径：没有运行时推理成本数据、没有 ONNX/推理优化工作、低层策略依赖物理仿真环境（UE 的 Chaos 理论上可做仿真环境，但论文未涉及 UE）。

唯一值得记的间接关系：**运动先验是可替换的模块**（AMP ↔ C·ASE ↔ SMP），这意味着**数据侧的工作（你的动捕规范）在整条链上是可复用的插件**，不是一次性投入。

## Technology Evolution

```text
DeepMimic 2018（模仿学习：单个技能）
        ↓
DIAYN / ASE 2022（技能发现：潜变量 + 运动先验，但技能库偏窄）
        ↓
APS / DADS / METRA（换技能发现目标，仍在潜空间里绕）
        ↓
InstantMimic 2026（训练成本 → 秒级，见 [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]]）
        ↓
★ DSD 2026（本文：把边缘状态熵这一项真正算出来 → 技能库变宽）
        ↑
  一个清晰的两阶段规律：先解决"学得多快"，再解决"学得多宽"
```

## Relationships

### Based On

- ASE / DIAYN 技能发现框架（技能潜变量 + 互信息目标 + 运动先验）——本文是给其中"边缘熵"那一项换估计器
- 扩散模型 / score matching（$\nabla\log p$ 估计）+ von Mises–Fisher 分布参数化

### Improves

- **ASE 全系（含 APS / DADS / METRA）**：同一目标函数的估计器替换，控制变量的对比就是它的立项理由

### Related

- [[Physics-based Character Animation]] — 本文属于这条线；与 [[Motion Matching]] 是**对偶**（见下）
- [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]] — 同月，解决"训练成本"的那一半
- [[Neural Animation]] — 平行阵营（神经运动学 vs 物理仿真），都不碰动力学
- [[Neural Physics Simulation]] — 更靠反问题一侧

### 与 [[Motion Matching]] 的对偶（今天最值钱的一条关系）

| | [[Motion Matching]] | DSD 零样本控制 |
|---|---|---|
| 候选集来源 | 动捕数据库（录制的片段） | 策略用随机潜变量**生成**的轨迹池 |
| 查询方式 | 每帧对当前状态做最近邻检索 | 离线对每个候选算适应度 $F(\tau,\mathbf{g})$，取 argmax |
| 代价/适应度 | 拼接代价（轨迹 + 姿态连续性） | 任务奖励的 Sum / Max |
| 覆盖上限 | **由录了什么决定** | 由**学出来的潜流形**决定 |
| 约束 | 物理不可行的拼接靠后处理修 | 天然动力学可行（在仿真里演化出来的） |

> **两者都是"用代价函数在候选集里挑"，差别只在候选集是"录的"还是"生成的"。** 这条对照可以直接搬去评估任何一个"动作检索/生成"方案：**先问它候选集从哪来、上限由谁决定。**

## Personal Knowledge State

- **user_level: Hard**（随 [[Physics-based Character Animation]] 整条线）。阻塞项不在本文，而在整条物理动画线的 RL 词汇。
- **但有一个 Normal 级的进入点**（不需要 RL 基础就能拿走）：**"技能库 = 潜变量流形"这个抽象**，以及它与 MM 的检索对偶。这一条是纯概念层面的，你现有的 [[Motion Matching]]（Normal）认知足够承接。
- 顺带说明：[[Motion Matching]] 的 40 分钟 Action 已在 2026-09-17 降级为 Watchlist 静默项。**本文的 MM 对照关系不构成新的行动项**——它只是把"库的宽度问题"这个概念补上。

## Learning Value

1. **一个可复用的工程判据**：拿生成模型当评分器时，先查它有没有把高频噪声当成信号（JIDS 就是修这个）——与 9-17 的"误差被谁乘掉"是同一类"先看评价指标本身"的思维；
2. **一个可复用的抽象**：动作库 = 候选集 + 代价函数；候选集可以是录的，也可以是生成的；
3. **一个阵营观察**：物理动画在 2026 年同时解决了"学得多快"（InstantMimic/Sprinting）和"学得多宽"（DSD）——**量变正在累积，但实时化数据仍然全线缺失**。

## Visualization

（无。本文信息以表格与对照为主，画图边际收益低。）

## Notes

- 元数据核实：arXiv:2609.17682v1，提交时间 Tue, 15 Sep 2026 18:01:48 UTC，归类 cs.LG 主 + cs.GR 交叉；arXiv 列表出现在 9-17 公告日（graphics 列表页）。
- 表格数值取自 arXiv HTML 版表 1–3，已抽样核对（FID 1.64 / 2.47、Div-R、34/37-DOF 角色、von Mises–Fisher、JIDS 章节名均在原文中出现）。**训练成本、推理延迟、技能簇数量三项未在正文给出**，笔记中未做任何推断。
- 同日落库的经典 [[Karis — Real Shading in Unreal Engine 4 (2013)]] 与 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 与本文无直接关系，仅同日。
