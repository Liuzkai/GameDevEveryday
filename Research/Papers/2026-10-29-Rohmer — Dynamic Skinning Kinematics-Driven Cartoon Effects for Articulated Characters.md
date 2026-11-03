---
type: paper
title: "Dynamic Skinning: Kinematics-Driven Cartoon Effects for Articulated Characters"
authors:
  - Damien Rohmer
  - Karim Salem
  - Niranjan Kalyanasundaram
  - Victor Zordan
year: 2026
published: 2026-10-29
venue: "IEEE TVCG 32(3): 2684–2694, March 2026（作者接受版，arXiv 2610.40976）"
url: https://arxiv.org/abs/2610.40976
code: ""
project_page: ""
category:
  - animation
  - skinning
  - real-time
  - procedural
importance: A
historical_importance: 0
game_relevance: 5
production_readiness: Prototype
user_level: Normal
status: unread
---

# Dynamic Skinning —— 二次运动不需要解方程，只需要一个核

> IEEE TVCG 2026（LIX/École Polytechnique × Clemson，Zordan 组），10 页，`2610.40976`。**一句话**：角色的"弹性余韵"（抖动、拖尾、波动沿骨架传播）不必解物理方程——**在 LBS 的结果上加一个由关节角速度与一个有限长因果滤波器算出的位移项**就够了，而且**无条件稳定、参数完全解耦、成本与普通蒙皮同阶**。

## TL;DR

- **它是一个加法项，不是一种混法**：$p_i^{DS}(t)=p_i^{LBS}(t)+\sum_j \beta_{ij}\,\psi\!\left(\mathcal F(\Omega_j,t),\,q_j,\,p_i^{LBS}\right)$。**混法一点没改**，只是在混完之后加了位移——所以它能套在 LBS / DQS / 任何蒙皮上；
- **三个因子各管一件事，这是全文真正的贡献**：$\psi$ 管**形状**（往哪弯、弯多少）、$\beta_{ij}$ 管**传播**（哪些顶点受影响）、$\mathcal F$ 管**时间**（什么时候动、动多久）。VS/AS 只有前两个，第三个恒等于"现在"；
- **$\beta$ 一分钱不花**：$\beta_{ij}=\sum_{j_k\in D(j)}\alpha_{ijk}$——**把"这个顶点归谁管"（蒙皮权重）沿子树求和，变成"这个关节动了会影响谁"**。已有的 rig 直接算出，自动光滑，自动跟随层级。且它**不满足重心性**（$\sum_j\beta_{ij}\ge 1$），这是故意的：一个顶点被三根骨影响，就该叠三次弯；
- **时间因子是一个 FIR 核，不是递推**：$\kappa(t)=A e^{-t/\tau}\sin(2\pi f t)$，于是 $f$、$\tau$、$A$ **三个旋钮完全独立**，并且因为"不迭代上一帧的值"，**对任意参数都无条件稳定**。对照：质量-弹簧-阻尼是二阶递推，**频率与阻尼被同一个递推式绑死**，且低阻尼 + 高频必炸；
- **六个参数里没有"时长"**：$\gamma_{bend}=1$、$A=0.3$、$\tau=1\,\mathrm{s}$、$f=1.5\,\mathrm{Hz}$、$W=5\tau$、$c=1\,\mathrm{m/s}$。**"抖多久"不是一个参数，是 $\tau$ 的副产品**——这条对预算表的含义见下文；
- **代价写在因果性上**：只能看过去 $\Rightarrow$ **造不出 anticipation（预备动作）**，只能造 follow-through 与振荡。这不是实现缺陷，是实时系统的硬边界（见 [[Lasseter — Principles of Traditional Animation Applied to 3D Computer Animation (1987)|Lasseter 1987]]）。

![[Research/Files/Pictures/Rohmer 2026 — 图1 三种动态蒙皮效果（长颈鹿·小丑·小火车）.png]]
**读图**（图 1，Rohmer et al. 2026）：同一条公式的三种用法，从左到右恰好是三个因子的三个极端——**长颈鹿**是 $\beta$ 覆盖全身（整条脖子一起荡）、**小丑**是 $\beta$ 被顶点绘制限制在帽尖与手臂末端（局部）、**小火车**是 $\mathcal F$ 里加了延迟（波动以 $c=1\,\mathrm{m/s}$ 沿车节往后传）。**注意三者用的 $\psi$ 是同一个弯折算子**，差别只在 $\beta$ 与 $\mathcal F$。

## Problem

角色动画里"活着"的那部分，绝大部分不在主体运动里：起跑时身体的滞后、急停时帽檐甩出去又弹回来、尾巴像波一样一节节传过去。这类**二次运动（secondary motion）** 在传统管线里靠两种办法：

| 路线 | 怎么来 | 死在哪 |
|---|---|---|
| **物理仿真**（CPD / CODY / 肌肉层） | 解"主体运动之外的正交子空间" | 慢；且**必须从初始条件积分** ⇒ 想改第 300 帧的一个细节，得重跑整段 |
| **降阶模型**（reduced / subspace） | 用少量模态近似动力学 | 快一些，但**输入与输出是非线性关系**，艺术家调不动 |
| **手工 deformer**（Maya 的 squash/jiggle） | 美术逐个摆 | 每个资产重做一遍，且没有"沿骨架传播"的概念 |

论文的立论句很直白：**"物理模型给出的是现象的*外观*，而外观完全可以不靠解方程得到。"**（原文用词是 "produce secondary motions that exhibit the appearance of physical phenomena without need for simulation"。）

## Historical Context

这条线的真正起点不是图形学，是 1930 年代迪士尼的动画原则；把它带进 CG 的是 [[Lasseter — Principles of Traditional Animation Applied to 3D Computer Animation (1987)|Lasseter 1987]]。而**把原则变成可执行代码**这条路，走了近四十年：

```text
1930s  Disney 十二原则（squash & stretch / anticipation / follow-through / timing…）
  ↓   原则是"要什么"，不是"怎么算"
1987  Lasseter：把原则搬进 CG —— 立论"不是软件给了角色生命"
  ↓
2006  Cartoon Animation Filter：x*(t) = x(t) − 平滑后的 x''(t)（等价于卷积 −LoG）
  ↓   第一次把"原则"写成"滤波器"；但那是对称核 ⇒ 看得到未来 ⇒ 只能离线
2008  Wiggly Splines：把振荡写成带约束的样条求解 ⇒ 好看，但要解全局系统
  ↓
2021  Velocity Skinning：把滤波降到"每个顶点、只用当前帧的关节角速度" ⇒ 因果，可进顶点着色器
2022  Acceleration Skinning：再加一阶导 ⇒ 有了 follow-through，但仍只是"一次反转"
  ↓
2026 ★ Dynamic Skinning：把"时间"这一维补成一个可调的核，并加上"沿骨架的延迟"     ← 本篇
```

**四十年里发生的不是"越来越物理"，而是"越来越像一个信号处理问题"**：从解方程（1987–2008）→ 查当前帧（2021–2022）→ **对一段很短的历史做卷积（2026）**。

## Previous Work

VS 与 AS 是本篇的直接父代，两者的式子值得抄下来对照（$\Omega_j$ 是关节 $j$ 的角速度，$q_j$ 是它的位置）：

$$p_i^{VS}=\underbrace{\sum_j\alpha_{ij}T_jp_i^0}_{p_i^{LBS}}+\underbrace{\sum_j\beta_{ij}\,\psi(\Omega_j,q_j,p_i^{LBS})}_{\text{瞬时形变}}$$

$$p_i^{AS}=p_i^{LBS}+\sum_j\beta_{ij}\,\psi(\dot\Omega_j,\dots)$$

- **VS 的病**：$\psi$ 只吃**当前** $\Omega_j$，所以"运动停了，形变立刻归零"——**它做不出余韵**。你只能靠输入运动本身的起伏去带出抖动；
- **AS 的病**：换成 $\dot\Omega_j$ 之后确实出现了"加速阶段拖、减速阶段甩"的一次反转，但**只有一次**。要持续的振荡，二阶导本身不提供。

**两者的共同点是：都没有"历史"这个概念。** 本篇补的正是这个。

## Core Idea

### 一、把"二次运动"拆成三个正交的因子

$$p_i^{DS}(t)=p_i^{LBS}(t)+\sum_{j\in\mathcal J}\beta_{ij}\,\psi\big(\mathcal F(\Omega_j,t),\,q_j,\,p_i^{LBS}\big)$$

| 因子 | 管什么 | 从哪来 | 换掉它会发生什么 |
|---|---|---|---|
| $\psi$（弯折算子） | **形状**：往哪个方向弯、弯多少 | 用户参数 $\gamma_{bend}$ | 换成"缩放"就是 squash & stretch，换成"平移"就是拖尾 |
| $\beta_{ij}$（层级权重） | **传播**：哪些顶点被这根骨带动、带多少 | **已有的蒙皮权重**（离线求一次和） | 换成 $\alpha$ 就退化成"只有归属骨影响自己"，没有沿骨架的连贯 |
| $\mathcal F(\Omega_j,t)$ | **时间**：什么时候开始、振多快、衰减多快 | 一个有限长因果核 | 换成恒等映射就退回 VS；换成 $\dot\Omega$ 就退回 AS |

**这个拆分本身就是结论**：过去所有"卡通形变"的工作都在改 $\psi$（挤压、拉伸、弯曲、噪声），而 $\beta$ 与 $\mathcal F$ 一直没被当成独立的、可设计的东西。

### 二、$\beta$ 是从蒙皮权重"求和"出来的（而且这招是免费的）

刚体蒙皮下，顶点 $i$ 绑在骨 $j_0$ 上，它的速度沿运动链可分解为 $\dot p_{i/j_0}=\sum_{j\in A(j_0)}v_{i/j}$，其中 $v_{i/j}=L_j+\Omega_j\times(p_i-q_j)$ 是**只有关节 $j$ 在动**时顶点获得的速度。把它代进 LBS 的加权和，双重求和可以**合并成一重**：

$$\dot p_i^{LBS}=\sum_{j\in\mathcal J}\beta_{ij}\,v_{i/j},\qquad \beta_{ij}=\sum_{j_k\in D(j)}\alpha_{ijk}$$

即：**$\beta_{ij}$ = 顶点 $i$ 的所有影响骨中，落在关节 $j$ 的子树里的那些权重之和**（$D(j)$ 含 $j$ 自身）。算法只有五行（附录 A）：从每个 $\alpha_{ij_0}>0$ 出发，沿父链一路累加。

三个必须记住的性质：

1. **它不改任何已有数据**——$\alpha$ 是现成的，$\beta$ 离线算一次；
2. **它自动光滑**——$\alpha$ 光滑 $\Rightarrow$ 有限项求和仍光滑，不需要额外做权重平滑；
3. **它不满足重心性**（$\sum_j\beta_{ij}$ 可以远大于 1，且更稠密）——**这正是想要的**：父关节转一下，整条子链上的顶点都该被带弯一点，叠加是物理直觉，不是 bug。

### 三、时间因子：用 FIR 核，不要用递推

论文的做法是：先规定"滤波器对单位阶跃的响应"$h(t)=(1+\kappa(t))U(t)$，其中 $\kappa$ 是一个**临时扰动**（$\kappa(0)=0$、$\kappa(W)\simeq0$、$|\kappa|\le A$）。默认取阻尼正弦：

$$\kappa(t)=A\,e^{-t/\tau}\sin(2\pi f t),\qquad W>5\tau$$

对任意输入 $\Omega_j(t)$，输出是冲激响应 $\dot h=\delta+\dot\kappa U$ 与输入的卷积；**分部积分一次**后得到一个更好实现的形式：

$$\mathcal F^{osc}(\Omega_j,t)=\Omega_j(t)+\int_0^{W}\kappa(u)\,\dot\Omega_j(t-u)\,du$$

也就是：**当前角速度 + 过去 $W$ 秒内角加速度的加权和**。离散化就是每帧对过去 $W/\Delta t$ 个样本做一次加权差分（式 15）。

**为什么不用质量-弹簧-阻尼？** 论文给的理由只有两句，但都硬：

| | 二阶递推（弹簧-阻尼） | 本篇的 FIR 核 |
|---|---|---|
| 频率与阻尼 | **耦合**（同一个递推式的两个系数） | **独立**（$f$ 与 $\tau$ 分别是正弦频率与指数衰减） |
| 稳定性 | 低阻尼 + 高频会发散，要额外做隐式/钳制 | **无条件稳定**——不迭代上一帧的输出，没有反馈回路 |
| 参数含义 | 刚度/阻尼，艺术家不认 | 振幅/衰减/频率，艺术家认 |

### 四、延迟：让"波动"沿骨架走

$\mathcal F^{delay}(x,t)=x(t-d_{ij}/c)$，其中 $d_{ij}$ 是"顶点到它最主要的骨 $j_{max}$ 的距离 + 沿骨架从 $j_{max}$ 上溯到 $j$ 的累计骨长"，$c$ 是波速（默认 $1\,\mathrm{m/s}$）。**$d_{ij}$ 在 rest pose 里一次算完**，运行时只查表。

把它套在振荡上 ⇒ **延迟振荡**；把它套在 LBS 的局部变换上（每个关节取不同时刻的 $T^{local}$）⇒ **dLBS（延迟蒙皮）**，小火车就是这么来的：**车头动了，车厢要过一会儿才动**。

![[Research/Files/Pictures/Rohmer 2026 — 图2 LBS 与时间滤波振荡的对照.png]]
**读图**（图 2，Rohmer et al. 2026）：左 = 标准 LBS，顶点位置由 $\Omega_j$ 唯一决定，**角速度一归零，形状立刻回正**；右 = 同一个 $\Omega_j$ 先过 $\mathcal F$，于是关节已经停了、形状还在往回弹（右侧那条红蓝曲线就是 $\mathcal F(\Omega_j,t)$ 与 $\Omega_j$ 的差别）。**这张图是"瞬时 vs 有记忆"的最小对照**，两边的几何完全一样。

![[Research/Files/Pictures/Rohmer 2026 — 图3 顶点到关节的骨架距离.png]]
**读图**（图 3，Rohmer et al. 2026）：$d_{ij}$ 的两段——红色从 $p_i$ 先垂到主骨（$p_{i\perp j_m}$），再沿骨链一路走到 $q_j$。**这条"沿骨架量的距离"而不是欧氏距离，是波动能绕过关节继续传的原因**：手贴在腰上时，欧氏距离很近，但沿骨架要走完整条手臂。

## Technical Approach

**每帧的完整流程**（附录 E 的伪代码，几乎就是全部）：

```text
每帧：
  1. 对每个关节 j：算 Ω_j，压进历史缓冲 LΩ_j
  2. 对每个顶点 i：p ← LBS(i)
  3. 对每个顶点 i、每根影响骨 j：
       Ω_filtered ← LΩ_j[m] + Σ_{n<W/Δt} κ(nΔt)·(LΩ_j[m-n] − LΩ_j[m-n-1])
       p += β_ij · ψ_bend(Ω_filtered, p − q_j)
  4. 输出 p
```

**加速版（式 24–25）**：如果运动是"一串脉冲"（突然启动/急停/受击），只保留**贡献最大的那一次冲击** $\hat t$：

$$\mathcal F(\Omega_j,t)\simeq\Omega_j(t)+\kappa(t-\hat t)\,\dot\Omega(\hat t),\qquad \hat t\leftarrow t \text{ 当 } \Omega(t)>e^{-(t-\hat t)/\tau}\Omega(\hat t)$$

**把 $W/\Delta t$ 次乘加压成 1 次**，复杂度回落到与 VS 相同。论文诚实写了它的失效条件：**连续变化的运动它记不住**（只有脉冲序列才近似得好）。

![[Research/Files/Pictures/Rohmer 2026 — 图8 全卷积与脉冲近似的对照.png]]
**读图**（图 8，Rohmer et al. 2026）：蓝 = 阶跃输入的角速度，绿 = 全卷积结果，红虚线 = 脉冲近似。**三次阶跃里两条曲线几乎重合**——这就是"近似成立"的样子；论文没画的是连续缓变输入下两者分岔的情形，而那正是它的边界。

## Key Contribution

1. **把"时间"提升为一个独立的、可设计的因子**（$\mathcal F$），并证明用 FIR 核比用二阶递推更适合"给艺术家调"；
2. **$\beta$（层级权重）这个变换**——一行求和，把"归属"变成"影响"，且完全复用已有 rig；
3. **dLBS**：把延迟加到**变换链**上而不是形变项上，于是"波动"这个效果可以脱离"弯曲"独立存在。

## Why It Works

- **线性 + 无反馈 ⇒ 可预测**。式 (8) 对 $\Omega$ 是线性的，$\mathcal F$ 是线性时不变（LTI）的，整条链没有把输出喂回输入。**"艺术家调大 A，抖动就变大，且不会突然炸"这件事，是 LTI + FIR 给的，不是调参技巧给的**；
- **形变只依赖相对量**（$\Omega_j$、$p-q_j$），所以**与角色尺度、朝向、骨架命名全部无关**——换一个模型不用重调；
- **$\beta$ 让"传播"免费**：不需要跑一遍扩散、不需要解泊松方程，蒙皮权重里本来就编码了"哪块皮跟着哪根骨"。

## Limitations

- **没有 anticipation**。$\mathcal F$ 只看过去，所以"起手前先往后缩一下"这一类**在定义上要求未来**的效果做不出来。这是本篇与 2006 年 Cartoon Animation Filter 的分水岭：CAF 的核是对称 LoG（中心在 $t$，用到 $t$ 之后若干帧），所以它能造预备动作——**代价是必须拿到整条曲线**。**"实时"与"预备动作"在这里是一条硬边界，不是工程细节**（此判断为我的推断，论文未讨论）；
- **没有碰撞、没有体积、没有动量**。它是"外观"，不是"物理"：手穿进身体、帽子穿过头，它一概不管（[[Cloth Simulation]] 那一套才管）；
- **dLBS 会让接触失效**。整条变换链都被延迟，意味着角色整体落后于它自己的根——**跑步时脚会离开地面、站立时会滑**。火车没有这个问题，人有（此点为我的推断，论文只展示了小火车与角色旋转的例子，未讨论足部接触）；
- **脉冲近似丢历史**（已述）；
- **Table 1 的口径要小心**（见下），且**本文没有新的用户研究**——"艺术家评估"引用的是前作 VS/AS 的结论（原文：*consistent with extensive artist evaluation in our previous efforts*）。

**Table 1 的正确读法**（这是全篇最容易误读的一张表）：

| | 每面片成本 | 最大模型 | 折算到该模型的每帧时间 |
|---|---|---|---|
| 本篇 | **0.00003 ms/面片** | **1.5 M** | **≈45 ms** |
| Wu & Umetani 2023 | 0.004 ms/面片 | 4.2 K | ≈17 ms |
| CODY 2023 | 0.05 ms/面片 | 0.3 M | ≈15 000 ms |

**"快三个数量级"是每面片的说法，不是每帧的说法。** 乘回去看：在它自己报告的最大模型上，本篇 ≈45 ms/帧（22 fps）；而 Wu 的方法虽然每面片慢 133 倍，但只跑 4.2K 面片，**每帧反而更快**。两列未必取自同一个模型（表题写的是"归一化成本"与"所用最大模型规模"两个独立事实），所以这只能当量级核算，不能当严格对照——**但它足以说明"归一化会藏住总量"**（见 [[Budget Units]] 今日补的第 ⑩ 条）。

## Game Development Relevance

- **风格化/二次元角色是直接受益者**：帽穗、发梢、飘带、兽耳、裙摆的"余韵"，现在可以用 6 个参数在**顶点着色器里**做，不需要给每根飘带挂骨链（Kawaii Physics / Dynamic Bone 那套的替代品）；
- **写实向也有位置**：受击时的肌肉/脂肪颤动、重甲角色的质量感、载具悬挂的二次弹跳——**只要你要的是"看起来有质量"，不是"真的守恒"**；
- **成本量级**：它没有新增 draw call、没有新增 pass、没有额外的骨骼；新增的是"每帧每关节一次滤波"（可 hoist 到 CPU）+ "每顶点每影响骨一次旋转"。**这是懒加载式预算：不开就不花钱，开了按顶点数线性涨**；
- **一条反向用法**：把 $\mathcal F$ 的 $\tau$ 调到极小（$\tau\to0$），它就退化成 VS；**这意味着"要不要余韵"是一个可以在运行时逐角色、逐技能切的开关**，很适合做 SABC 分档里的"表现等级"。

## Unreal Engine Relevance

**落点很清楚：AnimNode 出骨骼变换 → 一个 Custom 顶点工厂/材质 WPO 加位移项。** 但有一个必须改的实现细节：

- **不要照抄算法 2 的循环嵌套**。伪代码把卷积写在"顶点 × 关节"的两层循环里，复杂度是 $O(V\cdot J\cdot W/\Delta t)$——$1.5\,\mathrm{M}$ 面片、每关节 150 个历史样本时完全不可行；
- **正确做法是把滤波 hoist 到关节级**：每帧在 CPU（或一个极小的 compute pass）里对 $J$ 个关节各算一次 $\mathcal F(\Omega_j)$，**结果只有 $J$ 个向量**，塞进一个常量/结构化缓冲；顶点着色器只做 $\sum_j\beta_{ij}\psi(\cdot)$。这样复杂度是 $O(J\cdot W/\Delta t + V\cdot k)$，$k$ 是每顶点影响骨数（通常 4）。**我怀疑论文的 GPU 实现就是这么做的，但正文没写**——表 1 的量级只有这样才能成立；
- **$\beta$ 离线预计算**：从 SkinWeight 导出后跑一次子树求和，结果与 $\alpha$ 同尺寸，作为一张额外的权重贴图/缓冲；
- **与 AnimDynamics / Kawaii Physics 的分工**：那些是**反馈式弹簧骨**（会炸、要 clamp、要 substep），本篇是**前馈 FIR**（不会炸、不用 substep）。**"要稳定的余韵"用本篇，"要真的受力和碰撞"还是得回弹簧骨或 Chaos**；
- **坑**：$\beta$ 不归一 ⇒ 多骨叠加会放大形变，需要一个全局强度上限；且它对"关节数"敏感（骨越密，同一段形变被叠加的次数越多），**换 LOD 骨骼时要重标定**。

## Technology Evolution

```text
"二次运动"的四代实现，以及每一代交换掉了什么
1987  Lasseter        原则清单（squash/stretch、anticipation、follow-through、timing）
                       ↑ 要什么
2006  CAF             整条轨迹 ⊗ (−LoG)           ← 对称核：看得到未来 ⇒ 能造 anticipation，但只能离线
2008  Wiggly Splines  带振荡约束的样条求解          ← 全局解：好看，但改一帧要重解
2021  Velocity Skinning  每顶点、当前帧的 Ω          ← 因果：能进 GPU，但没有记忆
2022  Acceleration Skinning  加 Ω̇                  ← 一次反转，不是振荡
2026 ★ Dynamic Skinning  因果 FIR 核 + 沿骨架延迟    ← 有记忆，且仍然因果
        ↓
      可见的规律：**每往下走一步，"要看多远的过去/未来"就缩短一截**
      1987 要整个表演 / 2006 要整条曲线 / 2008 要整段解 / 2021 只要这一帧 / 2026 要最近 5 秒
        ↓
      被交换掉的同一样东西：**anticipation**（它只能来自"未来"）
```

## Relationships

### Based On

- **Velocity Skinning**（Rohmer et al., CGF/Eurographics 2021，库外）与 **Acceleration Skinning**（Kalyanasundaram et al., Graphics Interface 2022，库外）——同一组作者的前两级。$\beta$ 与 $\psi$ 的框架直接继承，本篇只补 $\mathcal F$。**两篇都未入库，故不给数字**；
- [[Skinning]]——它整体架在 LBS 之上，**不改混法，只加项**。

### Extends

- **Acceleration Skinning（2022，库外）**：把"一次反转"扩展成"任意长度的振荡"，方法是把 $\dot\Omega$ 换成一段 $\dot\Omega$ 的加权积分。

### Related

- [[Lasseter — Principles of Traditional Animation Applied to 3D Computer Animation (1987)|Lasseter 1987]]：本篇声称服务的十二原则里，它实际做到了 **squash & stretch / follow-through & overlapping / secondary action / timing** 四条；**anticipation 做不到**（因果），**staging / appeal 不归它管**；
- **Cartoon Animation Filter（Wang et al. 2006，库外）**：同一个"滤波器"想法，但是**离线且非因果**的（对称 LoG，需要未来帧）。**两篇的差别只在"核的长度与对称性"，结果差出一个"能不能实时"**。库里没有这篇，今日只在关系里记名；
- [[2026-10-30-He — HairDeck Animation-Ready Stylized Hair Cards from a Single Image|HairDeck]]（骨链生成）：**同一个问题的两种答案**——HairDeck 生成**真的骨链**（48 条弹簧骨，会受力、会碰撞、要 substep），DS 生成**位移**（不受力、不碰撞、永不炸）。**判据：要不要和世界交互**；
- [[Cloth Simulation]]：布料是"每个顶点独立自由度 + 约束"，DS 是"每个顶点零自由度 + 一个解析项"。**DS 是布料的"看起来像"版本**；
- [[Skeletal Animation]]：本篇的输入全是骨架量（$\Omega_j$、$q_j$、层级），**是"骨架 → 形变"这条链上的一次加法**。

### Followed By

- （待观察）**把 $\mathcal F$ 换成可学的核**：$\kappa$ 现在是人给的（阻尼正弦/扫频/梯形波），完全可以按角色或按动作学出来——这是最自然的下一步；
- （待观察）**把"因果"换成"短前瞻"**：动画系统里下一段 montage 是已知的，**只要提前 0.2 s 知道要做什么，anticipation 就能回来**。本篇没做，但接口上完全留得下（$\mathcal F$ 允许任意核，包括"负时间"的那一半）。

## Personal Knowledge State

**Normal（机制层 ≈20 分钟：式 8 的三个因子 + $\beta$ 的求和 + 核的三参数）**。

- **Easy**：LBS 是什么、蒙皮权重是什么、为什么二次运动让角色"活"——这些你在角色/VFX 工作里天天见；
- **Normal**：$\beta$ 为什么能从 $\alpha$ 求和得到、FIR 与二阶递推在"可调性/稳定性"上的差别、$W=5\tau$ 这个"记忆长度"是怎么定的；
- **Hard**：不需要。这篇没有需要啃的数学，附录 B 的分部积分是唯一一处推导。

## Learning Value

**最高的一条不是技术，是"参数该怎么设计"**：一个效果有六个旋钮，**其中没有"时长"**——这直接回答了本库 10-29 起挂着的问题（"VFX 时长 ≤3s/2s/1s/0.5s 约束的是长度还是形状？"）。答案是：**长度是形状的积分，把它当参数就是把结果当输入**。

## Visualization

四张原图（图 1 / 2 / 3 / 8）已嵌入上文。**未自绘 HTML**：本篇的核心结构是"一个加法式 + 三个因子"，Markdown 的表格与公式已经能表达；唯一的候选（三因子与四代演化的关系图）已在 Technology Evolution 用等宽文本画完。

## Notes

- **证据形态**：arXiv 2610.40976v1 是 IEEE 的作者接受版（10 页，含附录 A–E），页码/图号均按该版本；正式版为 TVCG 32(3): 2684–2694。
- 引用编号按 arXiv 版：VS = [27]，AS = [11]，CODY = [4]，Wu & Umetani = [34]。
- **一处未核**：正文说 GPU 实现是"single pass vertex shader"，但没给 pass 内的数据布局；"每帧 45 ms"是我按表 1 折算的量级估算，不是论文报告值。
- **一处作者没写但很关键**：$\beta$ 的稀疏性。它比 $\alpha$ 更稠密（子树求和会让远端的权重也非零），实际实现里大概率要按阈值裁剪，否则"每顶点 4 根骨"的假设不成立。这属于工程细节，风险不大但要自己补。
