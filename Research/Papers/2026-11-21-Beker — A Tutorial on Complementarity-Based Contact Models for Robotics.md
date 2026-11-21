---
type: paper
title: "A Tutorial on Complementarity-Based Contact Models for Robotics"
authors: ["Onur Beker", "A. René Geist", "Michael Posa", "Marc Toussaint", "Justin Carpentier", "Georg Martius"]
year: 2026
published: "2026-11-19"
venue: "arXiv preprint（cs.RO）2611.26600 —— 32 页教程，182 条参考文献"
url: "https://arxiv.org/abs/2611.26600"
code: "论文未给（正文逐节点名 Siconos / MuJoCo / Drake / Bullet / ODE / Chrono / Simple / Kamino 的实现）"
project_page: ""
category: ["Physics", "Contact Dynamics", "Rigid Body Dynamics", "Numerical Methods", "Tutorial"]
importance: A
historical_importance: 3
game_relevance: 5
production_readiness: "Research（教程本身不进管线；它描述的 PGS / LCP 求解器就是今天所有引擎的物理后端）"
user_level: Normal
status: unread
---

# A Tutorial on Complementarity-Based Contact Models for Robotics

## TL;DR

**接触不是一个函数，是一组「要么/要么」；把这组「要么/要么」写成代数式的过程有九个版本，每个版本买一样东西，也都欠一笔账。**

一句话结构：接触模型只承担两个现象学职责——**不许穿透**、**不许白白滑动**。这两条都写成互补条件（$0\le a\perp b\ge0$，即「至少一个为零」），于是**分离 / 粘着 / 滑移三种模式不是输入，是解的输出**。剩下的全部工作是把这组条件变成可解的形式：

| 层 | 转写 | 买到 | 欠下 |
|---|---|---|---|
| 精确（5 种，等价） | Signorini–Coulomb / N-CCP / root-finding / C-QCQP / C-QP | 完全等于 Coulomb 定律 | **非凸**，速度层无已知多项式算法 |
| 近似（丢 de Saxcé 项） | L-CCP / QCQP | 凸 | **滑动的物体会浮起来**（$v_n^+=\mu\|v_t^+\|$） |
| 正则（再加 $R$） | reg. L-CCP / reg. QCQP / NLS | 强凸、解唯一、可逆 | **失去静摩擦**（夹在两指间的笔会往下滑）+ 法向变软 |

三句最该记的话：

1. **互补条件的全部内容是「三个条件」**：$a\ge0$、$b\ge0$、$a^\top b=0$。第三种情况（两个都为正）被排除了，这就是"模式自动选择"的代数来源。
2. **精确与松弛的差别只有一项**：de Saxcé 修正 $\Gamma(v^+)=\mu\|v_t^+\|\hat n$。有它，法向条件干净；没它，锥的对偶把切向速度硬塞进法向（dilatancy），滑动的箱子就会升离桌面。
3. **当解不唯一时，求解器是模型的一部分。** 冗余接触有很多力分布、摩擦系统可能有多个运动解——**选哪一个由初始化、扫描顺序、正则项决定**，而论文把这件事明写进正文（§6.1）。

> 它回答的不是"怎么写一个物理引擎"，而是"**你引擎里那个求解器，在物理定律上做了哪一处让步**"。

## Problem

刚体接触的难点不在碰撞检测，在**模式**。两个物体接触时，法向只有两种状态（分离 / 受力），切向只有两种（粘着 / 滑移），合起来每点 4 种；$N$ 个接触点就是 $4^N$ 种组合，而真实解只是其中一种。

传统做法是**枚举**：假设一个模式组合，解线性方程，检查不等式是否成立，不成立就换组合。Baraff 1993 证明这条路的判定问题是 NP-完全的（见 [[Baraff 1991 — Coping with Friction for Non-penetrating Rigid Body Simulation|Baraff 1991]]）；指数时间的算法在 **10–15 个接触点**上就不可用。

互补条件给出另一条路：**不枚举模式，让模式从方程里掉出来**。代价是问题变成非凸、非光滑的互补问题——"没有已知的多项式时间算法（速度层）"，只能靠松弛或特殊结构。

这篇教程的动机很具体：**现有材料要么写给计算力学、要么写给图形学、要么只讲一个模拟器**（SIGGRAPH 2022 的 Andrews–Erleben–Ferguson 课程是最接近的一本，但它以线性互补 + penalty/barrier 为主，锥互补只带一句），而"把锥互补的九个转写、十四个算法、完整推导、统一记号放在一起"的机器人向材料此前不存在。

## Historical Context

这条线比图形学长得多，教程的附录 B 把它一直追到 1699 年。三个关键节点：

```text
1699/1785  Amontons / Coulomb        干摩擦的经验律：切向力反向于滑移、上界 ∝ 法向力
1798/1838  Fourier / Ostrogradsky    单边约束 + 每个约束一个非负乘子 ⇒ 互补的雏形
1872/1895  Jellett / Painlevé        单点摩擦：解可能不唯一（Jellett）、可能不存在（Painlevé）
1899       Mayer                   由 Gauss 最小约束原理导出加速度层互补条件
1917       Delassus                 "哪些约束是活的"——那个二次型就是今天的 G
1963       Fichera                 Signorini 问题的存在唯一性 ⇒ 变分不等式学科起点
1966       Moreau                  加速度解一个凸 QP、接触力解它的对偶 QP（原始/对偶形式出场）
1983-88    Moreau                  测度微分包含：力与冲量统一成"一步内的总冲量"
1991       Baraff                   "必须先用非冲量力"这条原则导致指数时间 ⇒ 换答案空间
1994       Baraff                   不转成通用优化、保住逐点结构 ⇒ Dantzig 枢轴
1996/97    Stewart–Trinkle / Anitescu–Potra   多面体锥 ⇒ LCP，Lemke 有限步可解
1998       Stewart                 步长趋零时收敛到 Moreau 的解
1991/98    de Saxcé–Feng           锥互补 + 修正项：把两条条件合成一条
2004/06    Anitescu                丢修正项 ⇒ 凸松弛（滑移升离的来源）
2011/14    Todorov                 正则化 + NLS：MuJoCo 的可逆形式
2026       ★ 本篇                   九个转写 + 三笔账 + 一个蓝图
```

**这条线有一个反直觉的形状**：1960 年代之前，摩擦的**难**是力学问题（Painlevé 佯谬）；1980 年代之后它变成**计算**问题（NP-完全）；2010 年代之后它变成**记账**问题（每个松弛欠什么）。本篇是第三阶段的总结——它不再问"能不能解"，而问"**你为了能解，改动了哪一条物理**"。

## Previous Work（它站在谁的肩膀上，又刻意不重复谁）

教程 §1.1.1 罕见地把"我和谁重叠、重叠在哪"写清楚了，这一段本身就是选读指南：

| 已有材料 | 覆盖什么 | 本篇的差异 |
|---|---|---|
| Andrews / Erleben / Ferguson，SIGGRAPH 2022 course | 图形学向：碰撞检测细、线性互补 + penalty/barrier、可变形体 | 本篇是机器人向：**锥互补**为主，碰撞检测只给两页，可变形体不覆盖 |
| Acary / Brémond / Huber 2018 | 计算力学向：精确形式 + Siconos 数值对比 | 本篇**多讲凸松弛**（机器人圈的常用做法）与其物理伪影 |
| Acary & Brogliato 专著 / Stewart 2000 | 非光滑动力学的数学理论 | 本篇不做测度论，只做"够用且自洽"的推导 |
| Le Lidec et al. 2023（T-RO） | 各模拟器的形式与算法对比 + 数值实验 | 本篇补**教学**：逐步推导、统一记号、以及已不再常用的历史形式 |
| MuJoCo 文档 / Drake SAP 论文 | 各自的框架内推导 | 本篇把两者放进**同一个**非线性最小二乘转写里 |

它明确划掉的边界也值得记：**事件驱动法**（Glocker–Pfeiffer 那条，精度高但贵，力学圈主流）、**PBD / penalty / barrier**（图形学主流，已有大量教程）、**compliant contact 的弹簧-阻尼点接触与 hydroelastic 压力场**。**划掉不等于否定，是"这些已有好教材"。**

## Core Idea

### ① 两个职责 ⇒ 三个条件

法向：不许穿透。写成**互补**而不是不等式，是因为"接触力"只在真正贴上时才存在：

$$0\le\underbrace{\left(\frac{d}{\Delta t}+v_n^+\right)}_{\text{分离速度}}\perp\underbrace{f_n}_{\text{法向力}}\ge0$$

它同时说了三件事：分离速度非负（不穿透）、法向力非负（只能推不能拉）、两者至少一个为零（**贴上才有力，有力必贴上**）。

切向：不许白白滑动（最大耗散）。同样两条：

$$0\le\left(\mu f_n-\|f_t\|\right)\perp\|v_t^+\|\ge0,\qquad \|v_t^+\|\,\|f_t\|=-f_t^\top v_t^+$$

第一式是"要么粘着（$v_t=0$，摩擦力可以在锥内任意取值）要么滑移（$\mu f_n=\|f_t\|$，摩擦力顶到上界）"；第二式是"滑移时摩擦力精确反向"。

![[Research/Files/Pictures/Beker 2026 — 图6 Signorini-Coulomb 的四个状态.png]]

**读图**：左上是法向的两态（$f_n=0$ 且留空隙 / $f_n>0$ 且步末正好贴面），左下是切向的最大耗散（$f_t$ 与 $v_t^+$ 精确成 180°），右二是粘着（力可以在锥内任意方向、速度为零），右一是滑移（力顶在锥面上、速度非零且反向）。**四个状态里没有任何一个"被指定"，它们全是方程的解**——这就是互补写法相对枚举写法的全部优势。出处：Beker et al. 2026，图 6。

### ② 两条条件 ⇒ 一条锥条件（de Saxcé 修正）

把法向与切向合成一条，代价是速度里要加一项：

$$\mathcal K\ni f\perp\big(v^++\Gamma(v^+)+\gamma\big)\in\mathcal K^*,\qquad \Gamma(v^+)=\mu\|v_t^+\|\,\hat n$$

**这一项为什么必须存在？** 因为锥的对偶 $\mathcal K^*$ 的边界是 $\|y_t\|\le y_n/\mu$。不加修正时，要求 $v^+ +\gamma\in\mathcal K^*$ 就等于要求 $v_n^++\gamma_n\ge\mu\|v_t^+\|$——**滑移必须伴随法向分离**（术语叫 dilatancy）。物理上滑动不需要离开桌面，所以必须把 $\mu\|v_t^+\|$ 这一份从法向里"预支"出去，让法向条件退回 $v_n^++\gamma_n\ge0$。

![[Research/Files/Pictures/Beker 2026 — 图8 de Saxce 修正.png]]

**读图**：绿色是修正后的速度 $v^++(\mu\|v_t^+\|+\gamma_n)\hat n$，橙色箭头就是被"预支"的那一份 $\mu\|v_t^+\|\hat n$。没有它，粉色的对偶锥边界会把绿色的法向分量压到 $\mu\|v_t^+\|$ 以上，滑动的物体会被抬起来。出处：Beker et al. 2026，图 8。

> **这一个箭头，就是下面"滑移升离"伪影的全部来源。** 丢掉 $\Gamma$ 让问题变凸，代价是这一份预支没了，法向只能自己补——于是 $v_n^+=\mu\|v_t^+\|$。

### ③ 九个转写，一张地图

![[Research/Files/Pictures/Beker 2026 — 图5 九种转写的三层地图.png]]

**读图**：横轴按数学形式分组（箱约束 QP / 标量互补 / 锥互补 / 锥约束 QP / 无约束），纵轴按物理保真度分层（exact / approximate / regularized）。**实线箭头是保解的等价改写，红色虚线箭头是改变物理的近似**，每条虚线旁标着"买到什么(+)、丢掉什么(−)"：丢 $\Gamma$ ⇒ +凸 / −滑移升离；加 $R$ ⇒ +强凸唯一 / −蠕滑（无静摩擦）。蓝色回头箭头是教程最后给的外层循环。出处：Beker et al. 2026，图 5。

这张图值得单独读三遍，因为它是**整篇的组织原则**：九个形式不是九个竞争方案，而是**同一个问题的九种"记账方式"**，箭头方向就是"你为了让求解器好写，动了哪一步"。

## Technical Approach

### ① 一切都从一个矩阵开始：Delassus

半隐式欧拉离散后，接触点速度与力的关系是线性的：

$$v^+=v_f+Gf\Delta t,\qquad G=JM^{-1}J^\top\in\mathbb R^{3N\times3N}$$

$G$ 叫 **Delassus 矩阵**（1917 年那个二次型），它把质量、惯量、杠杆、接触几何全部编码进去。**后面所有算法都在处理同一个对象**：给定 $G$ 和 $v_f$，求 $f$ 使互补条件成立。

- $G$ **秩亏**是常态（冗余接触多于自由度）⇒ 力不唯一、线性系统奇异 ⇒ 这就是"正则化 $R$"存在的理由。
- $G$ 的**条件数**决定 PGS 收敛多慢：质量比、转动惯量差异、杠杆长度、近乎相关的约束都会让它变坏。

### ② 五个精确转写（都等价，选哪个只影响"哪种算法能用"）

| 转写 | 形式 | 结构上暴露了什么 |
|---|---|---|
| Signorini–Coulomb (39) | 逐点标量互补 + 方向条件 | 最贴近物理，但三条条件混在一起 |
| N-CCP (43) | 一个锥互补，含 $\Gamma$ | 变分不等式的标准形 ⇒ 投影类算法可用 |
| Root-finding (44/45) | SOCFB 残差 $=0$ | 无不等式 ⇒ 牛顿法可直接上 |
| C-QCQP (47) | 外层 $\sigma$ + 内层锥约束 QP | 直通凸优化文献（Acary 2011） |
| C-QP (55) | 法向 QP 与切向 QP 交替 | **把"难在哪"隔离出来**：难在确定哪些点滑、往哪滑 |

C-QP 那一句是全篇最锋利的诊断：**如果"哪些点滑、滑向哪、上界多少"这三件事已经知道，整个问题塌成一个箱约束凸 QP。** 难的从来不是求解，是**这组离散决定**。

### ③ 四个松弛（每个都在改物理）

| 松弛 | 动作 | 数学收益 | 物理代价 |
|---|---|---|---|
| L-CCP (56) / QCQP (57) | 丢掉 $\Gamma$ | 凸 | 滑移时 $v_n^+=\mu\|v_t^+\|$ ⇒ **滑动的物体升离** |
| reg. L-CCP (58) / reg. QCQP (59) | $G\to G+R\succ0$ | 强凸、解唯一、可逆 | 法向变软（穿透 ∝ 力）+ **静摩擦消失** |
| NLS (60) | 消去 $f$，改在加速度空间 | 无约束、强凸、Hessian 尺寸 $n_v\times n_v$ | 继承正则化的全部代价 |
| LCP（附录 A） | 圆锥换多面体锥 | **总有解**，Lemke 有限步精确求解 | **各向异性摩擦**（方向依赖基向量） |

**"静摩擦消失"的推导值得单独看**（§4.3.1 + §6.2）：正则化后，接触力等于 $R^{-1}\Pi_{\mathcal K}(a_{\text{ref}}-J\ddot q)$，而参考加速度 $a_{\text{ref}}$ 的切向分量在零速度时恒为零。于是 $R\succ0$、$\dot q=0$、$\ddot q=0$ 时，**切向力必然是零**——不管法向压得多紧。

> 原文的例子：**一支笔夹在两根手指之间，会顺着重力往下滑，无论手指压得多用力。** 这不是实现 bug，是"强凸"这件事的数学后果。

### ④ 十四个算法，按"每步要不要解线性系统"分两族

| 族 | 算法 | 每迭代线性求解 | 收敛 | 早停时返回 |
|---|---|---|---|---|
| **无解线性系统** | PGS (1)、PGD (3)、精确局部 GS (7)、全锥 PGS (8)、法向-切向分离 PGS (9) | 无 | 全局但最好线性，越病态越慢 | **可行力**（都在锥内） |
| **每步解一次** | Newton 自然映射 (2/10)、ADMM (4)、内点 (5)、NLS Newton (6)、SOCFB Newton (11) | $3N\times3N$ 或 $n_v\times n_v$ | 局部超线性 / 全局少迭代 | Newton 类**可能跑出锥外** |
| 混合 | 交错箱 QP (12)、proximal (13)、**blueprint (14)** | 内层继承 | 无保证，但不动点精确 | 继承 |

三条工程判据（Table 2 的读法）：

1. **力空间 $3N$ vs 加速度空间 $n_v$**：接触比自由度多时（$3N>n_v$，机器人常态），NLS 的线性系统更小；反之力空间更便宜。**这是"按哪个空间计价"的直接后果。**
2. **warm start 与早停**：实时应用都是"上一步的解当初始值、固定迭代次数停"。PGS 类**天然可 warm start 且早停仍给可行力**；内点法难 warm start；Newton 类中途的迭代值可能不在摩擦锥内——**所以引擎里几乎都是 PGS/TGS。**
3. **要可微就得光滑**：只有内点法（$\tau>0$）和 SOCFB（$\epsilon>0$）返回**对数据光滑依赖**的解。而这两者都是"松弛过的定律"——**可微性与物理保真度在这里是直接冲突的**。

### ⑤ 蓝图（Alg 14）：精确性由外层买回来

这是教程自己的贡献，也是全篇最像"编译原理"的一步：

```text
外层循环：
  r ← v_f + Γ(σ) + γ − ρDz
  内层：用任意一个松弛求解器解 (G+R) 的那三个等价形式，得到 f*
  z ← f*        （proximal 锚点）
  σ ← v_f + G f* Δt   （de Saxcé 修正用的速度）
不动点处：z = f*，σ = v⁺ ⇒ proximal 项消失、Γ 用对了 ⇒ 满足精确条件
```

**它的意义不是"更快"，是把"精确"与"可解"解耦**：内层随便用什么凸求解器都行，外层负责把被丢掉的那两项（$\Gamma$ 和正则化偏置）迭代回来。代价是嵌套循环、无收敛保证。

## Key Contribution

1. **九个转写放进同一套记号**，并给出"实线=等价、虚线=改物理"的图（图 5）。
2. **把每个松弛的物理伪影写成可检验的命题**：滑移升离与 $\mu\|v_t^+\|$ 成正比；静摩擦消失可从逆动力学直接读出。
3. **把"求解器的选择"提升为模型的一部分**（§6.1）：解不唯一时，初始化、扫描顺序、正则项决定你看到哪一个解。
4. **Alg 14 蓝图**：用任意凸求解器 + 外层 proximal/Γ 更新，逼近精确解。
5. **一张模拟器对照表**（Table 3）：每个引擎用哪个转写、哪个算法、什么硬件。

## Why It Works

三条数学事实支撑全部推导：

- **互补 = 投影不动点**：$\mathcal K\ni x\perp F(x)\in\mathcal K^*$ 等价于 $x=\Pi_{\mathcal K}(x-\rho F(x))$（对任意 $\rho>0$）。这一条把"互补问题"翻译成"找不动点"，于是 PGS/PGD/ADMM 全部合法。
- **Moreau 分解**：$x=\Pi_{\mathcal K}(x)+\Pi_{\mathcal K^\circ}(x)$，且 $\nabla\frac12\|\Pi_{\mathcal K}(x)\|^2=\Pi_{\mathcal K}(x)$。第二节那条"用投影把约束搬进目标"的推导全靠它。
- **proximal 项在不动点消失**：$\frac{\rho}{2}\|f-z\|_D^2$ 在 $z=f^*$ 时为零，所以外层循环的极限点满足原问题——**这就是"用松弛解精确"的合法性来源**。

## Limitations

1. **它是教程，不是新方法。** 没有任何新算法、没有基准测试、没有运行时间。所有成本论述都是"每迭代要解多大的系统"这种结构性陈述。
2. **只覆盖锥互补这一族。** PBD/XPBD、penalty、barrier、compliant contact 全部划在界外——**而游戏引擎里 XPBD 用得很多**，所以这本教程不能当作"游戏物理全景"读。
3. **各向同性 Coulomb、无滚动摩擦、无粘附、无各向异性摩擦**（后者只在 LCP 那一节作为代价被提到）。
4. **刚体、点接触。** 可变形体、面接触、软体都不在范围（虽然引用了 Bertails-Descoubes 2011 的纤维精确摩擦作为延伸）。
5. **历史附录是二次文献整理**（作者自陈"supplemented by the authors' own derivations"），个别年代归属需回原文核（例如 Signorini 1933/1959 两个年份并列，说明作者也没完全定）。
6. **"没有已知多项式算法"是速度层的说法**；加速度层的 NP-完全是 Baraff 1991/1993 的结果，两者不是同一件事——**引用时别混**。

## Game Development Relevance

**这一篇是游戏物理的"源码注释"。** Table 3 直接点名：

| 引擎 | 转写 | 算法 | 硬件 |
|---|---|---|---|
| PhysX / Isaac | LCP (85) | PGS、**TGS** | CPU、GPU |
| Bullet | C-QP (55) / LCP | PGS、pivoting | CPU |
| ODE / DART | boxed LCP | PGS、pivoting | CPU |
| MuJoCo / MJX | reg. QCQP（对偶）/ NLS（原始） | Newton、CG、PGS | CPU / GPU |
| Drake（SAP） | NLS | Newton + 精确线搜索 | CPU |
| Chrono | L-CCP / QCQP | PGS、PGD、ADMM | CPU |
| Siconos | 精确族 | 非光滑 GS、Alart–Curnier、SOCFB Newton | CPU |
| RaiSim | N-CCP | 逐接触精确解 + GS 扫描 | CPU |
| Kamino | C-QCQP | proximal ADMM（蓝图 14） | GPU 批处理 |

原文有一句直球：**"PGS 仍然是游戏引擎的主力（workhorse）"**，引的是 Erin Catto 的 GDC 2005《Iterative Dynamics with Temporal Coherence》与 Erleben 的教材。

**四个能直接对上号的引擎症状**（详见 [[Contact Dynamics]] 的 §Game Applications）：

| 症状 | 来源 | 判据 |
|---|---|---|
| 角色站斜坡缓慢下滑 | 正则化丢掉静摩擦 | 把求解器正则/迭代容差调小看是否改善（但会更抖） |
| 箱子在斜面上滑动时"浮起" | 丢了 de Saxcé 项（凸松弛） | $\mu$ 大、滑速快时最明显，粘着时消失 |
| 方块塔抖动 / 缓慢穿透 | Baumgarte-ERP 的 $\alpha$ + 单步求解 | $\alpha$ 越大恢复越快也越抖；$\alpha=0.5$ 是"每步消一半" |
| 大质量比堆叠收敛慢 | $G$ 的条件数 | 换 TGS/子步（把 $\Delta t$ 变小）比换算法更有效 |

## Unreal Engine Relevance

- **Chaos 的接触求解走的是 LCP/PGS-TGS 这一支**（PhysX 3 时代是 PGS，UE5 Chaos 用 PGS + 子步进；TGS 是"小步长 + GS"的工程变体，不是另一个转写）。
- **`Contact Offset` / `Rest Offset` 是 $\gamma_n$ 的工程实现**：把"允许的穿透"和"提前多久算接触"做成两个可调参数，本质就是教程 §4.1.1 里的 constraint stabilization——**推导对 $\gamma_n$ 的函数形式不敏感，只要 $d=0$ 时 $\gamma_n=0$**。
- **子步进（Substepping）是在买条件数**：$\Delta t$ 变小 ⇒ $G\Delta t$ 的谱更集中 ⇒ PGS 收敛更快、穿透更少。**这与"换一个更贵的算法"是两条不同的路。**
- **不要在 UE 里期待"精确 Coulomb"**：Chaos 的摩擦是圆锥/多面体近似的工程实现，静摩擦靠"速度阈值 + 摩擦系数"近似。**教程给的是判据：想知道你的引擎在哪一格，看它在滑移时法向速度是否被 $\mu\|v_t\|$ 顶起来。**

## Technology Evolution

```text
1699–1899  经验律 + 佯谬（Amontons / Coulomb / Jellett / Painlevé）
   ↓
1917–1966  凸分析进场（Delassus 的二次型 / Signorini–Fichera 的变分不等式 / Moreau 的原始-对偶 QP）
   ↓
1983–1999  测度微分包含 + 非光滑接触动力学（Moreau / Jean）⇒ 时间步进成为主流
   ↓
1991–1997  复杂度与算法（Baraff 的 NP-完全与枢轴 / Stewart–Trinkle、Anitescu–Potra 的 LCP）
   ↓
1991–2007  锥互补（de Saxcé–Feng 的修正 / Glocker–Studer 的 SORprox）
   ↓
2004–2014  凸松弛与可逆形式（Anitescu / Todorov 的 MuJoCo）
   ↓
2010–2026  规模化与可微（Anitescu–Tasora 的 PGS / RaiSim / Dojo / Simple / Kamino）
   ↓
2026  ★    统一记账（本篇：九个转写、三笔账、一个蓝图）
```

**每一段的驱动力不同**：1900 前是"物理自洽"，1960 年代是"数学严格"，1990 年代是"计算可行"，2010 年代是"实时 + 可微"。**本篇所在的阶段是"审计"**——不问能不能算，问"你算的是不是你以为的那个物理"。

## Relationships

### Based On

- **Moreau 的测度微分包含（1983–1988）与 Jean 的非光滑接触动力学（1999）**：整篇的时间离散框架来自这里，包括"用一步内的总冲量而不是瞬时力"这个决定。
- **de Saxcé–Feng 的 bipotential（1991/1998）**：$\Gamma$ 修正项的出处；没有它就没有"一条锥条件 = 两条物理条件"。
- **[[Baraff 1991 — Coping with Friction for Non-penetrating Rigid Body Simulation|Baraff 1991]] / [[Baraff 1994 — Fast Contact Force Computation for Nonpenetrating Rigid Bodies|Baraff 1994]]**：图形学一侧的起点（复杂度 + 枢轴算法），教程的 §1.1.4 把 LCP 一族归到这条线上。

### Extends

- **Andrews–Erleben–Ferguson（SIGGRAPH 2022 course）**：把那本以线性互补为主的课程，扩成以**锥互补**为主、并把九个转写摆成一张图。
- **Le Lidec et al. 2023（T-RO）**：从"对比各模拟器"扩到"给出完整推导与统一记号"。

### Related

- **[[Cloth Simulation]]**：同一族求解器（PBD/XPBD 是另一条转写），布料的自碰撞用的就是这里的 PGS 思想。
- **[[Neural Physics Simulation]]**：可微物理要穿过的正是这篇 §7.2 列的四个坑。
- **[[Physics-based Character Animation]]**：脚滑（creep）与穿地（penetration）就是这篇 Table 1 的两笔账。
- **[[Baraff 1994 — Fast Contact Force Computation for Nonpenetrating Rigid Bodies|Baraff 1994]]**：同一问题的"另一条哲学"——不转成通用优化，保住逐点结构。

### Followed By

- **可微接触**（Dojo / DiffContact / Simple）：把 §7.2 的四个困难逐个攻。
- **GPU 批处理的 proximal ADMM**（Kamino）：把蓝图 14 搬上 GPU。

## Personal Knowledge State

- **Easy**：碰撞检测、法向不穿透、摩擦锥、Coulomb 上界、Baumgarte/ERP 的直觉——这些是日常工作。
- **Normal（今天补的）**：
  1. **"三个条件"是互补的全部内容**，模式是解的输出；
  2. **de Saxcé 项是一个"预支"**，它把切向的量从法向条件里挪走，丢掉它就是滑移升离；
  3. **正则化的两笔账**：法向变软 + 静摩擦消失（笔的例子）；
  4. **"按哪个空间计价"**：力空间 $3N$ vs 加速度空间 $n_v$。
- **仍是 Hard**：Alg 14 的收敛性证明（proximal point + 非凸外层）、SOCFB 的广义 Jacobian、以及"解不唯一时求解器如何选"的定量刻画。

## Learning Value

**对做引擎/工具的人**：这是一张"你的求解器在物理上做了什么让步"的对照表。读完应该能做到三件事——看到一个新物理引擎，从它的文档/源码判断它落在图 5 的哪一格；看到一个物理 bug（脚滑、浮起、抖动、穿透），先猜是哪一笔账，再去验；评估"换求解器"的收益时，知道换的是算法还是转写。

**对做技术美术/特效的人**：至少记住"静摩擦是买来的，不是免费的"。所有"把摩擦调大一点"的直觉，在正则化求解器里都会先被"切向力上限"截断。

## Visualization

![[Research/Files/Pictures/Beker 2026 — 图9 摩擦锥的多面体近似.png]]

**读图**：把圆锥换成内接多面体（$K=4/6/8$ 个基向量 $\hat t_k$），第 $k$ 个基向量上的系数 $\lambda_k$ 就是摩擦力的分解。滑移方向被限制在"某一个基向量"或"两个基向量之间"，于是 **LCP 总能用 Lemke 有限步解出来**（这就是引擎里 LCP 的来源）。代价写在图右下：**方向被量化了**，斜着推箱子会感觉"卡角"（各向异性摩擦）。出处：Beker et al. 2026，图 9。

## Notes

- 原文 PDF 已归档：`Research/Files/PDF/Beker 2026 - A Tutorial on Complementarity-Based Contact Models for Robotics.pdf`（32 页，816 KB，arXiv 2611.26600v1）。
- 本次只逐字精读了 §1–§6 与附录 B，§7.2（可微仿真）读了四个困难的分类但未逐条读引文；附录 A 的 LCP 推导只读了结论（式 84–85）。
- **术语对照**（写代码/读文档时容易混）：
  - *transcription*（转写）= 同一组物理条件的**数学写法**，不是近似；
  - *relaxation*（松弛）= **改了物理**的写法；
  - *regularization*（正则化）= 在 $G$ 上加正定项，改的是"解唯一性"，代价是"法向变软 + 静摩擦消失"；
  - *constraint stabilization*（约束稳定）= $\gamma_n$ 的设计，**不改物理**，只改穿透的恢复速度。
