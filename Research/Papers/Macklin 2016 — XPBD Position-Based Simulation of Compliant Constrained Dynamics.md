---
type: paper
title: "XPBD: Position-Based Simulation of Compliant Constrained Dynamics"
authors: ["Miles Macklin", "Matthias Müller", "Nuttapong Chentanez"]
year: 2016
published: "2016-10"
venue: "MIG '16（ACM SIGGRAPH Conference on Motion in Games），Burlingame, CA，6 页，DOI 10.1145/2994258.2994272"
url: "https://matthias-research.github.io/pages/publications/XPBD.pdf"
code: ""
project_page: ""
category: ["Physics Simulation", "Constrained Dynamics", "Cloth", "Real-Time", "Classical"]
importance: A
historical_importance: 4
game_relevance: 5
production_readiness: "Industry Adopted（2016 年后新建的 PBD 系求解器基本都带柔度参数）"
user_level: Normal
status: unread
aliases: [XPBD, Extended Position Based Dynamics, 扩展位置动力学, 柔度, compliance]
tags: [classical, physics, cloth, constraint-solver, compliance, real-time]
---

# XPBD — 给"软"一个单位

## TL;DR

**六页。只加了三样东西：每约束一个标量 $\lambda$、一次 $\alpha/\Delta t^2$ 的除法、和分母里的一个 $+\tilde\alpha$。** 结果是 PBD 那个 $[0,1]$ 的"刚度"变成了有单位的**柔度** $\alpha$（m/N），并且**与步长、与迭代数都无关**。

核心公式（式 18）：

$$\Delta\lambda_j=\frac{-C_j(x_i)-\tilde\alpha_j\,\lambda_i^{\,j}}{\nabla C_j\,M^{-1}\nabla C_j^{\mathsf T}+\tilde\alpha_j},\qquad \tilde\alpha_j=\frac{\alpha_j}{\Delta t^{2}},\qquad \Delta x=M^{-1}\nabla C^{\mathsf T}\Delta\lambda$$

**三个变化各自解决一件事：**

| 变化 | 解决什么 |
|---|---|
| 分子里的 $-\tilde\alpha\lambda$ | **约束不再被推到头**——它留下一个与柔度匹配的残余 $C=-\tilde\alpha\lambda$，也就是一根真的弹簧 |
| $\tilde\alpha=\alpha/\Delta t^2$ | 让"这根弹簧多硬"**与步长无关**（量纲上必须如此，见下） |
| 分母里的 $+\tilde\alpha$ | 让 $\alpha=0$ 时**精确退化成 PBD**（$k=1$）⇒ 升级是渐进的，老代码不用重写 |

作者最诚实的一句在 §6.4：

> "We note that **XPBD does not converge faster than PBD**, simply that behavior remains consistent at different iteration counts."

**它不是更快的 PBD，它是"同一个行为"的 PBD。** 买的是**可移植**，不是速度。

## Problem

PBD 的"软"是**没解完**。这在三个场景里同时崩掉（摘要里点名了两个）：

1. **多材质同场**：软体和近刚体放在一起，为了让刚体够硬去加迭代 ⇒ **所有物体一起变硬**，只能全局重调刚度系数。
2. **单资产内部**：一件布料的拉伸与弯曲约束比例，也随迭代数**非线性**漂移（PBD 的 $k'$ 只把它变成线性，没消除对 $\Delta t$ 的依赖）。
3. **资产复用**：参数不可移植 ⇒ "可复用的仿真资产"根本做不出来（原文：*making the creation of reusable simulation assets extremely difficult*）。

外加一条工程诉求：**PBD 没有"约束力"这个概念** ⇒ 触觉设备、可断关节、应力可视化全部无从下手。

## Historical Context

```text
2006  PBD             刚度 = 每次迭代删掉多少残差（无量纲、不可移植、无力）
  ↓
2006  Servin 等       柔度约束公式 + 速度级半隐式积分（"与步长无关"这条路已经有人走）
2007  Goldenthal 等   Fast Projection：全局线性化 + 直接解（PBD 的无柔度版本）
2013  Liu 等 / 2014   Projective Dynamics：局部-全局，刚度基本与迭代数无关
2015  Tournier 等     Stable Constrained Dynamics：加二阶信息避免线性化失稳
2016  ★ XPBD          把 Servin 的柔度搬进 PBD 的 Gauss-Seidel 循环（只加一个标量）
```

**这条线的形状很特别**：XPBD 不是发明，是**把两个已有想法的接口对上**——Servin 的柔度（物理侧）与 PBD 的局部投影（工程侧）。所以它只有六页，而且明确说"我们能和更贵的非线性求解器给出视觉上同等的解"。

## Core Idea

从隐式 Euler 出发，把约束写成能量：

$$U(x)=\tfrac12 C(x)^{\mathsf T}\alpha^{-1}C(x),\qquad M\ddot x=-\nabla U^{\mathsf T}$$

$\alpha$ 是**柔度**（compliance）= 刚度的倒数。引入拉格朗日乘子后，把 $\Delta t$ 折进柔度里：$\tilde\alpha=\alpha/\Delta t^2$，得到离散系统

$$M(x^{n+1}-\tilde x)-\nabla C^{\mathsf T}\lambda=0,\qquad C(x^{n+1})+\tilde\alpha\lambda=0$$

第二式就是"软"的定义：**约束不被满足到零，而是满足到"留一个与力成正比的残余"**。把它代回去：

$$\lambda=-\frac{C}{\tilde\alpha}=-\frac{C\,\Delta t^{2}}{\alpha}\quad\Longrightarrow\quad f=\frac{\lambda}{\Delta t^{2}}=-\frac{C}{\alpha}=-k\,C$$

**一根刚度为 $k=1/\alpha$ 的弹簧。** 这就是全篇最重要的一句话：**XPBD 的约束不是"投影到流形上"，是"一根弹簧的隐式解"**。

### 为什么偏偏是 $\Delta t^2$

不是拟合出来的，是**量纲逼出来的**：

| 量 | 单位 | 说明 |
|---|---|---|
| $\alpha$（柔度） | m/N = s²/kg | 位移 / 力 |
| $\nabla C\,M^{-1}\nabla C^{\mathsf T}$ | 1/kg | 梯度无量纲、$M^{-1}$ 是 1/kg |
| $\tilde\alpha$ | 1/kg | **只有 $\alpha/\Delta t^2$ 能和它相加** |
| $\lambda$ | N·s² = kg·m | 由 $\Delta x=M^{-1}\nabla C^{\mathsf T}\Delta\lambda$ 反推 |
| $\lambda/\Delta t^2$ | N | 力 |

**"分母里能加的那一项，决定了分子该除几次 $\Delta t$。"** 同一个除法的第二个作用是极限：$\alpha\to0$ 时 $\tilde\alpha\to0$，式 (18) 精确变成 PBD 的 $s=C/\nabla CM^{-1}\nabla C^{\mathsf T}$。

### 两个近似，都不改解

为了不组装 $K=\partial g/\partial x$（要约束的 Hessian），作者做两件事：

1. **$K\approx M$**：丢掉几何刚度与约束 Hessian ⇒ 局部误差 $O(\Delta t^2)$，**但不改不动点**（准 Newton）。这是"免费升级"的技术前提。
2. **假设 $g(x_i,\lambda_i)=0$**：第一次迭代时按构造成立（$x_0=\tilde x,\lambda_0=0$），梯度变化慢时保持很小，梯度恒定时为零。

于是"每步解一个大系统"退化成"每条约束一次标量除法"——**这正是 PBD 的形状，只是分母多了 $\tilde\alpha$、分子多了 $-\tilde\alpha\lambda$。**

## Technical Approach

- **算法**：与 PBD 逐行相同，只在三处插入——初始化 $\lambda_0=0$（第 4 行）、按式 (18) 算 $\Delta\lambda$（第 7 行）、更新 $\lambda$（第 9 行）。原文：*"identical to the original PBD algorithm with the addition of lines 4, 7, and 9."*
- **阻尼**：Rayleigh 耗散势 $D=\tfrac12\dot C^{\mathsf T}\beta\dot C$，$\tilde\beta=\Delta t^2\beta$，单约束更新里多一项 $\gamma_j\nabla C_j(x_i-x^n)$，$\gamma_j=\tilde\alpha_j\tilde\beta_j/\Delta t$。**注意 $\beta$ 是"阻尼刚度"不是逆量**。
- **接触**：完全照 PBD——**零柔度**（$\alpha=0$），所以接触约束不需要存 $\lambda$。
- **材料接口**：线弹性各向同性 FEM 可以直接写成约束——把应变张量 $\epsilon=(\epsilon_x,\epsilon_y,\epsilon_{xy})$ 当约束向量，柔度矩阵就是刚度矩阵的逆（Lamé 参数），**泊松效应由 $\alpha$ 的非对角项自动带出来**。这一点比 Strain-Based Dynamics 强：后者要手工解耦各方向应变。

## Key Contribution

1. **把"刚度"从求解器手里拿回来**：$\alpha$ 有单位、与 $\Delta t$ 和迭代数无关、可以逐约束不同。
2. **约束力第一次成为可观测量**：$\lambda$ 是总乘子，$f=\lambda/\Delta t^2$ ⇒ 触觉、可断关节、应力可视化。
3. **增量式升级**：$\alpha=0$ 精确退化成 PBD，每约束多一个标量、每步多约 2% 时间（表 1）。
4. **与非线性 Newton 求解器对齐的验证**（这是它比"又一个 PBD 变体"更硬的地方）。

## Why It Works

1. **因为它是一根弹簧，不是一次投影。** 投影要求 $C=0$；弹簧要求 $C=-\alpha f$。后者才有"多软"这个自由度。
2. **因为柔度是逐约束的。** 提高迭代数只是让每条约束更接近**它自己**的解，不会改变别人 ⇒ 多材质同场不再互相绑。
3. **因为近似只改收敛路径，不改不动点。** $K\approx M$ 让它保持 PBD 的每步成本，却收敛到同一个隐式解。

> **一句话**：PBD 用"没解完"表示软，XPBD 用"约束本身有一个柔度"表示软；**前者是求解器的状态，后者是材料的属性**。

## Limitations

**作者自陈（四条）：**

1. **$\alpha=0$ 时与 PBD（$k=1$）完全一样**，需要同样多的迭代才能变硬——XPBD 没有让刚体更刚，只让它"刚得更稳定"。
2. **低迭代数 ⇒ 人工柔度**：提前停止 = 系统比设定的更软（"artificial compliance"）。
3. **只是隐式 Euler 的近似**（不像 Projective Dynamics 是精确的变分形式），需要更高精度保证的场合应换传统方法。
4. **收敛速度没变**：Gauss-Seidel / Jacobi 仍是线性收敛（图 3b），作者建议换 Chebyshev 之类加速。

**读出来的（三条）：**

5. **接触仍是零柔度**，所以"软接触"（法向可压缩）不在框架内——这正是后来 TGS / 软约束接触要补的。
6. **测量口径全是小规模**：20 粒子链、64×64 布（24k 约束）、单个悬臂梁；**没有多材质大场景的端到端计时**。
7. **没有"每帧毫秒 vs 迭代数"之外的账**：表 1 只给了单步毫秒，碰撞检测、自碰撞、GPU 传输都不在表里。

## Game Development Relevance

**它回答了项目里最常见的一句抱怨："同一件衣服，换台机器/换个帧率，手感就变了。"**

| 现象 | PBD 的解释 | XPBD 之后 |
|---|---|---|
| 降帧率 ⇒ 布料变软/变形 | $\Delta t$ 进了有效刚度 | $\tilde\alpha=\alpha/\Delta t^2$ 把它除掉 |
| 加迭代 ⇒ 全场景一起变硬 | 迭代是全局的求解程度 | 每条约束有自己的 $\alpha$ |
| 资产只能在原引擎用 | 参数无单位 | $\alpha$ 可换算成 $k$（N/m），能与实测材料对上 |
| 想要"可断的绳子" | 没有力 | $f=\lambda/\Delta t^2$ |

**但要记住它没解决什么**：$\alpha=0$ 的极限下你仍然需要很多次迭代才能"看起来刚"。真正把这件事推进的是 2019 年的**小步长**（多子步、每步 1 次迭代）——**那是另一篇，不在本文里**，库内未入库。

## Unreal Engine Relevance

- **判断一件资产的参数是不是"真参数"**：把它换一个帧率跑同一段动画。**形变随帧率变 ⇒ 参数是求解器参数（PBD 语言）；不变 ⇒ 是材料参数（XPBD 语言）**。
- **Chaos Cloth / Vellum 暴露的仍多是 0–1 的 stiffness**——这是 PBD 的遗产。要在项目里做"跨平台一致的布料"，需要**把界面上的 0–1 与求解设置一起当成一个整体来标定**，而不是把它当杨氏模量填。
- **可断约束 / 应力驱动特效**：$\lambda$ 提供了"这根约束现在受多大力"的读数，这是 PBD 时代做不到的接口。

## Technology Evolution

```text
2006  Servin        柔度约束（速度级）——"与步长无关"的第一次
2006  PBD           位置级投影（快、稳、但刚度不可移植）
2016  ★ XPBD        两者合流：位置级 + 柔度（每约束一个标量）
2019  Small Steps   把"没解完"压到极小：多子步 × 每步 1 迭代
2020  XPBD 刚体      同一柔度框架做刚体关节与摩擦
2020s 工业侧        柔度进入 DCC/引擎的"物理材质"面板
```

## Relationships

### Based On

- [[Müller 2007 — Position Based Dynamics]] —— 算法骨架（预测 → 投影 → 反推速度）原封不动，只换约束的"软硬"表达。
- Servin 等 2006 的柔度约束公式（**未入库**）——$\lambda=-\tilde\alpha^{-1}C$ 这一行是它的。

### Extends

- PBD 的 $k'$ 修正：从"把 $k$ 映射成与迭代数线性"升级成"把 $k$ 换成有单位的 $\alpha$"。

### Related

- [[Cloth Simulation]] —— 本页是那一页解法表里"投影（PBD/XPBD）"行的理论依据；也是它 Learning Gap 第 1 条（$\alpha$ 为什么让刚度与步长无关）的答案。
- [[Contact Dynamics]] —— 对照点很干净：**接触那页的三档转写（精确/松弛/正则）在 PBD 系里被压成一档**——不等式约束 + 零柔度 + 同一个 Gauss-Seidel。代价是没有真库仑摩擦、没有唯一解保证。
- [[Multigrid]] —— 同一句判据的两个应用：**迭代数不是成本**（多网格用 $T=T_{\rm setup}+N(T_{\rm update}+n_{\rm iter}T_{\rm iter})$ 说，XPBD 用"行为不随迭代变"说）。

### Contrasts

- [[Baraff-Witkin — Large Steps in Cloth Simulation (1998)]] —— 同样是"让刚度成为材料属性"，但那条路每一步解一个稀疏系统（成本 $\propto$ 共轭梯度迭代），这条路每一步做 $M$ 次标量除法（成本 $\propto$ 约束数）。

### Followed By

- Small Steps in Physics Simulation（Macklin 等，SCA 2019）—— 多子步 + 单迭代，把 XPBD 的"低迭代人工柔度"压下去。**未入库。**
- Detailed Rigid Body Simulation with XPBD（Müller 等，SCA 2020）—— 刚体版。**未入库。**

## Personal Knowledge State

`Normal`。这是"引擎里天天见、从没算过"的典型：

- **已建立**：stiffness 是 0–1、加迭代会变硬、换帧率会变手感——三条都是实操经验。
- **今日新增**：$\tilde\alpha=\alpha/\Delta t^2$ 的**量纲理由**（分母能加的那一项决定分子除几次 $\Delta t$）；$f=\lambda/\Delta t^2=-kC$（XPBD 的约束就是一根弹簧）；$\alpha=0$ 退化成 PBD。
- **未建立**：没算过"$\alpha$ 与实际材料（N/m）怎么标定"；没做过"换帧率不变形"的对照实验。

## Learning Value

**最高，而且成本最低。** 六页，公式只有一条半，但它是"把工程旋钮翻译成物理量"这件事在图形学里最干净的一次示范：

> **判据**：任何一个"0–1 的滑块"，问它三件事——**有单位吗？改别的设置会不会变？换个引擎还能用吗？** 三个都否 ⇒ 它是求解器参数，不是材料参数。

## Visualization

![[Research/Files/Pictures/Macklin 2016 — 图2 简谐振子 PBD 周期随迭代改变.png]]

**读图**（图 2，Macklin 等 2016）：一个距离约束做的简谐振子（$\alpha=0.001$、$m=1$、静止长度 1、初始位移 1.5）。蓝色是解析解，**PBD 的 1 / 5 / 10 次迭代给出三条不同的曲线**——周期和阻尼都随迭代数变；XPBD 与解析解几乎重合。**同一个 $k$，三种物理。**

![[Research/Files/Pictures/Macklin 2016 — 图3 约束力误差与残差收敛.png]]

**读图**（图 3，Macklin 等 2016）：(a) 20 粒子链（$\alpha=10^{-8}$）顶端约束力随帧变化，50 / 100 / 1000 次迭代的最大相对误差 **6% / 2% / 0.5%**——这是"PBD 没有力、XPBD 有力"的定量证据；(b) 残差随迭代下降——**Gauss-Seidel 与 Jacobi 都是线性收敛，XPBD 不比 Newton 快**，这一格就是"它不是更快的 PBD"的图。

![[Research/Files/Pictures/Macklin 2016 — 图6 悬挂布料在 20-40-80-160 次迭代下的对比.png]]

**读图**（图 6，Macklin 等 2016）：64×64 网格、**24k 条距离约束**、人为把刚度压到 $k=0.01$。**上排 PBD**：20 → 160 次迭代，布料从"软塌的深 V"逐步变成"硬挺的平板"；**下排 XPBD**：同样的四个迭代数，形态几乎不变。**这张图是"软 = 没解完"最直观的证据。**

## Notes

- **标题里的 "Compliant" 是关键词**：不是"遵守约束"，是**"可柔顺的"**——约束允许被违反，违反量正比于力。
- **出版信息**：MIG '16（Motion in Games），2016-10-10~12，Burlingame；6 页；DOI 10.1145/2994258.2994272；作者全部在 NVIDIA。
- **数字口径**：表 1 是**单步**仿真时间（ms），不含渲染、不含碰撞检测；64×64 布 = 24k 约束是本文最大规模。
- **别把"小步长"算在它头上**：XPBD 2016 并未推荐 substepping；"多子步 + 单迭代"是 2019 年 Small Steps 的结论。本库多处提到"XPBD + 小步长"，引用时要分清年份。
- 原文 PDF 与 6 张原图已入 `Research/Files/`。
