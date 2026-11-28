---
type: paper
title: "Position Based Dynamics"
authors: ["Matthias Müller", "Bruno Heidelberger", "Marcus Hennix", "John Ratcliff"]
year: 2006
published: "2006（会议版）/ 2007（期刊版）"
venue: "3rd Workshop in Virtual Reality Interactions and Physical Simulation（VRIPHYS 2006），Eurographics；期刊版 JVCIR 18(2):109–118, 2007"
url: "https://matthias-research.github.io/pages/publications/posBasedDyn.pdf"
code: ""
project_page: ""
category: ["Physics Simulation", "Constrained Dynamics", "Cloth", "Real-Time", "Classical"]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（PhysX Cloth / Houdini Vellum / UE Chaos 的解算内核都属于这一族）"
user_level: Normal
status: unread
aliases: [PBD, Position Based Dynamics, 基于位置的动力学, 位置动力学, 约束投影]
tags: [classical, physics, cloth, constraint-solver, real-time, game-engine]
---

# Position Based Dynamics — "软"，是因为没解完

## TL;DR

**这篇论文把仿真里的"求解"从力层搬到了位置层：不算力、不解线性系统，直接把点推到满足约束的位置上，再用位置差反推速度。**

它真正的贡献不是"更快"，是**换了一个可以被人手调的旋钮**：

- 传统路线：刚度 $k$ 是材料属性（N/m），刚度大 ⇒ 系统硬 ⇒ 显式积分炸、隐式要解方程；
- PBD：**刚度 $k\in[0,1]$ 是"每次迭代删掉多少残差"**。$k=1$ 是"一次推到位"，$k=0.01$ 是"每次只推 1%"。

所以 PBD 里的"软"，本质是**欠松弛（under-relaxation）**——**软 = 没解完**。这个定义在游戏里工作了十年，因为游戏本来就用固定步长、固定迭代数：**只要这两件事不变，"没解完"看起来就是一个材料属性**。

作者自己写下了这条边界（§3.3 最后两行）：

> "However, the resulting material stiffness is still dependent on the time step of the simulation. **Real time environments typically use fixed time steps in which case this dependency is not problematic.**"

**这句话是整个 PBD 家族的宪法，也是它九年后被 [[Macklin 2016 — XPBD Position-Based Simulation of Compliant Constrained Dynamics|XPBD 2016]] 修掉的那一条。**

## Problem

2006 年的处境：游戏要布料的**稳定性、可控性与速度**，不要精度。三条硬约束：

| 需求 | 力法为什么难 |
|---|---|
| **不能炸** | 布料的强拉伸约束 ⇒ 刚度谱跨三四个数量级 ⇒ 显式积分的稳定步长 $\Delta t\lesssim\sqrt{m/k}$ 被最硬的约束钉死 |
| **要能直接摆弄** | 美术要把一个顶点钉在骨骼上、要保证它不穿进碰撞体——力法只能"加一个很硬的弹簧"，然后祈祷 |
| **碰撞要能管住** | 力法把穿透当"力不够大"，穿透是**软约束的副产品**；而项目要的是"绝不穿" |

第三条是根本分歧。力法回答"该用多大力把它推出来"，PBD 回答"它就不该在那里"。

## Historical Context

```text
1995  Provot        质量-弹簧 + 【步后位置修正】——只修拉伸过头的弹簧，是"补丁"不是"方法"
  ↓
1999  Desbrun       同样把投影当抛光步骤（projection as a polishing process）
  ↓
2001  Jakobsen      GDC《Advanced Character Physics》：Verlet + 直接改位置
                    ⇒ 已经是 PBD，但只处理距离约束，且没定义"一般约束"怎么进来
  ↓
2005  Müller 等     形状匹配（goal position）：积分方式最接近，但只处理一个全局约束
2005  Clavet 等     粘弹性流体：投影里到处是 Δt ⇒ "not fully position based"，仍条件稳定
  ↓
2006  ★ PBD        一般约束框架 + 动量守恒分析 + 商业化布料解算器
  ↓
2013  PBF           同一套投影搬到流体（密度约束）
2016  XPBD          把 [0,1] 的 k 换成有单位的柔度 α（见 [[Macklin 2016 — XPBD Position-Based Simulation of Compliant Constrained Dynamics]]）
```

作者在 Related Work 里对 Jakobsen 的定位很准：**"他已经是对的，但没有给出一般约束的写法，也没有处理动量守恒"**。这篇论文补的正是这两块——**一个通用的约束接口，和一条"为什么这样推不会凭空造力"的证明**。

## Core Idea

五个动作，其中三个是新的：

```text
① 外力        v ← v + Δt·g
② 预测        p ← x + Δt·v            ← 只是显式 Euler 的一步，谁都会
③ 生成碰撞约束  C(p) = (p − q_c)·n_c ≥ 0   ← 每帧现生成，和固定约束一起解
④ 投影（迭代）  p ← p + Δp(C)          ← 【核心】不是解方程，是逐个约束推位置
⑤ 反推速度     v ← (p − x)/Δt          ← 位置改了，速度跟着改（这就是"隐式"的那一半）
```

第 ④ 步的公式是本篇的全部技术内容：

$$\Delta p_i=-s\,w_i\nabla_{p_i}C,\qquad s=\frac{C(p_1,\dots,p_n)}{\sum_j w_j\big|\nabla_{p_j}C\big|^2},\qquad w_i=\frac1{m_i}$$

**三个因子，三个来源，缺一个就错：**

| 因子 | 从哪来 | 去掉会怎样 |
|---|---|---|
| $\nabla C$ 的方向 | **动量守恒**：内部约束对刚体运动不变 ⇒ $\nabla C\perp$ 平移/旋转模式 ⇒ 沿梯度推不产生"鬼力"（ghost force） | 物体会被自己的约束推着走、凭空转动 |
| $\sum_j w_j\lvert\nabla C\rvert^2$ 的分母 | **对单条约束做一次 Newton 步**：$C(p+\Delta p)\approx C+\nabla C\cdot\Delta p=0$ | 推过头或推不到位，迭代不收敛 |
| $w_i$ 的加权 | 逆质量：无限质量的点不动（$w=0$），轻的点让得多 | 钉住的顶点会被拉走、重物被轻物推飞 |

**动量守恒那一条值得单独记住**，因为它是"为什么可以随便推"的唯一理由：投影只沿 $\nabla C$ 走，而内部约束的梯度与刚体模式正交，于是 $\sum m_i\Delta p_i=0$ 且 $\sum r_i\times m_i\Delta p_i=0$ 自动成立。**这不是事后补的修正项，是"选对方向"的副产品。**

## Technical Approach

### 刚度 $k$ 到底是什么

最朴素的做法是把 $\Delta p$ 乘上 $k$。作者立刻指出它的病：$n_s$ 次迭代后残余误差是 $\Delta p(1-k)^{n_s}$ ⇒ **$k$ 的效果对迭代数是非线性的**。于是给了一个修正：

$$k'=1-(1-k)^{1/n_s}$$

代入后误差变成 $\Delta p(1-k)$——**与 $n_s$ 无关，与 $k$ 线性**。

**但这一步只修好了"迭代"，没修"步长"**，作者紧接着写下那句"material stiffness is still dependent on the time step"。这两句话在论文里只占三行，却是 2016 年整篇 XPBD 的靶子。

### 碰撞：不是另一套系统，是另一批约束

- **连续性检测**：对射线 $x_i\to p_i$ 求进入点 $q_c$ 与法向 $n_c$，加不等式约束 $C(p)=(p-q_c)\cdot n_c\ge0$；
- **静态兜底**：射线整段在体内（连续检测已经失败）时，取最近表面点 $q_s$；
- **摩擦/恢复**：不在投影里做，放在第 ⑤ 步之后动速度（沿法向阻尼 + 反射）。

**代价写在明处**：碰撞约束在求解循环**外面**生成，解算过程中集合不变 ⇒ 可能漏碰撞（"artifacts are negligible" 是经验陈述，不是保证）。

### 四个工程细节，都能直接搬

| 细节 | 做法 | 为什么值钱 |
|---|---|---|
| **阻尼** | 先算整体 $v_{cm}$ 与 $\omega$，只阻尼"相对刚体运动"的偏差 $\Delta v_i$ | $k_{\rm damping}=1$ 时物体表现得像刚体；**阻尼不拖慢整体运动**——这是"阻尼不能惩罚刚体模式"的第一份可执行写法 |
| **弯曲** | 二面角约束 $C=\arccos(n_1\cdot n_2)-\varphi_0$ | **与边长无关** ⇒ 拉伸刚度与弯曲刚度可以独立调（图 3 的六格就是这件事的证据） |
| **固定点** | 把 $w_i$ 设为 0 | 比"加一个超大质量的点"干净；约束里自动不参与 |
| **内压** | 全局体积约束 $C=\sqrt{\sum_i (p_{t_1}\times p_{t_2})\cdot p_{t_3}}-k_{\rm pressure}V_0$ | 一个约束管住整个网格——"全局约束"这条路在 PBD 里是免费的 |

## Key Contribution

1. **一般约束接口**：任意 $C(x)$、等式/不等式、任意基数——距离只是它的一个特例（式 10/11 与 Jakobsen 的公式重合，作者明说"as a special case"）。
2. **动量守恒的证明**：给出"沿梯度推"这条规则的物理合法性，并引入"鬼力"这个概念。
3. **无条件稳定**：不是数值技巧，是**构造性的**——第 ⑤ 步只把点移到"已经合法的位置"，从不外推到未来。原文：唯一的失稳来源是解算器本身，而**它的稳定性不依赖步长，只依赖约束函数的形状**。
4. **产品级实现**：AGEIA 的布料解算器（Rocket 环境），带自碰撞、双向刚体耦合、撕裂、内压。

## Why It Works

三句话，层层递进：

1. **投影从不外推。** 显式积分敢往前走是因为它相信泰勒展开；PBD 只走到"被约束允许的最近位置"。所以"炸"这个失效模式在结构上不存在。
2. **修正方向由约束自己给。** 沿 $\nabla C$ 走既是 Newton 步（收敛），又是正交于刚体模式的方向（守恒）——**一个选择同时买到两件事**。
3. **"软"被重新定义成"没解完"。** 这是它便宜的原因（不需要组装矩阵、不需要共轭梯度），也是它不可移植的原因（$k$ 的含义随 $n_s$ 与 $\Delta t$ 漂移）。

> **一句话**：PBD 把"解方程"换成了"反复推"，于是**材料参数退化成了求解器参数**。

## Limitations

**作者自陈（三条）：**

1. **刚度依赖步长**："the resulting material stiffness is still dependent on the time step"——在固定步长的实时环境里"不成问题"，但这句话把问题推给了使用者的纪律。
2. **不是清晰的隐式或显式**："The integration does not fall clearly into the category of implicit or explicit schemes." 一次迭代像显式，多次迭代像隐式。
3. **刚体不在本文范围内**：Future Work 明写"a topic we have not treated"——**PBD 2007 是粒子 + 约束，不是刚体求解器**（刚体版是 2020 年那篇 XPBD 刚体）。

**读出来的（四条）：**

4. **多约束下 $k$ 改的是不动点，不只是收敛速度。** 单约束时迭代 $k<1$ 仍收敛到 $C=0$（只是慢）；多约束时每个约束推一部分、下一个约束又推回去，平衡点是一个 **$k$ 决定的折中**——这就是"刚度"的真实来源，也是它不可移植的根源。
5. **Gauss-Seidel 的顺序是隐式输入。** 原文承认"过约束时会震荡，如果顺序不固定"；而顺序同时决定收敛速度（压力波一次传多远）。
6. **解算过程中碰撞集合冻结** ⇒ 漏碰撞；补法是加子步，代价是重算碰撞。
7. **没有约束力这个概念。** 没有 $\lambda$ 就没有"这根绳子受了多大力"，于是可断关节、触觉反馈、应力可视化全部做不了——这是 XPBD 补的第二件事。

## Game Development Relevance

**今天引擎里那块"布料"，旋钮的语义就是这篇定的。** 三个滑块的真实含义：

| 界面上的名字 | 实际在动什么 | 后果 |
|---|---|---|
| Stiffness（0–1） | 每次迭代删掉的残差比例 | **改迭代数或子步，视觉会跟着变**（它不是一个材料属性） |
| Solver Iterations | 收敛程度 | 迭代越多 ⇒ 越硬 **且越"闷"**（数值阻尼变大，见 XPBD 图 6） |
| Subdivisions / Substep | 每帧切几个 $\Delta t$ | 子步变多 ⇒ 等效变硬（$\Delta t$ 变小） |

**这就是"同一个资产换台机器就变样"的机制**：不是资产坏了，是**资产里存的不是一个材料参数**。反过来，这也解释了为什么 PBD 系的参数"好调"——它调的是**收敛程度**，而人对"硬/软"的感知本来就与"解得多彻底"同构。

## Unreal Engine Relevance

- **Chaos Cloth** 属于这一族：约束 + 迭代 + 子步，参数是"每个约束类型的 stiffness + damping + 迭代/子步"，不是杨氏模量。
- **能直接用的判据**：拿到一件"手感不对"的衣服，先分清是**材料账**（约束本身太软）还是**求解账**（迭代/子步不够）。分法与本库 [[Cloth Simulation]] 里那条一致——**"这个穿/这个软，随迭代数变吗"**：变 ⇒ 求解账；不变 ⇒ 几何/材料账。
- **Niagara 的约束模块**在数学上是同一个东西（位置投影），只是没有 FTL 的迭代调度。

## Technology Evolution

```text
1995  Provot            步后修正（补丁）
2001  Jakobsen          Verlet + 改位置（只做距离约束）
2006  ★ PBD             一般约束 + 动量守恒 + 商业布料解算器（AGEIA）
2013  PBF               同一套投影做流体（密度约束 + 人工压力）
2016  XPBD              k → 柔度 α（有单位、与 Δt/迭代数无关）
2019  Small Steps       宁可多子步、每步少迭代（把"没解完"进一步压小）
2020  XPBD 刚体          同一框架做刚体（关节、摩擦、接触）
2010s Houdini Vellum / UE Chaos Cloth / PhysX Cloth —— 工业化
```

**读法**：这条线一直在修同一个东西——**"软"到底该由谁定义**。PBD 说由迭代规程定义（便宜但不可移植）；XPBD 说由约束的柔度定义（贵一个标量但可移植）；Small Steps 说由子步数定义（更贵但更准）。**三次改动都在同一个旋钮上换单位。**

## Relationships

### Based On

- [[Baraff-Witkin — Large Steps in Cloth Simulation (1998)]] —— 同一个约束函数 $C(x)$ 的语言（约束即质量、$\nabla C$ 即方向），但那条路走"解线性系统"，这条路走"投影"。**两篇是同一道题的两个答案，成本结构正好相反**。

### Extends

- Provot 1995 / Desbrun 1999 的"步后投影"：从"修过冲的补丁"升级成"整个积分器"。

### Related

- [[Cloth Simulation]] —— 本页在那一页的解法表里是第三行（投影路线）；那一页的 Learning Gap 第 1 条（PBD/XPBD 机制）由本篇与 XPBD 结清。
- [[Contact Dynamics]] —— 接触在本篇里**不是独立系统**：碰撞就是每帧现生成的不等式约束，进同一个 Gauss-Seidel 循环。**好处是没有 LCP、没有 Delassus 矩阵；代价是没有真库仑摩擦、没有收敛保证、顺序相关。**
- [[Multigrid]] —— 一对镜像：多网格把"每步代价"换成 $O(n)$，PBD 把"每步代价"换成"一个循环"。两者都在回答"迭代数算不算成本"。

### Contrasts

- [[Baraff-Witkin — Large Steps in Cloth Simulation (1998)]] / [[Shewchuk 1994 — An Introduction to the Conjugate Gradient Method Without the Agonizing Pain]] —— 隐式路线：**刚度是材料属性、成本在解方程**。PBD 是它的反面：**成本在迭代、刚度不是材料属性**。

### Followed By

- [[Macklin 2016 — XPBD Position-Based Simulation of Compliant Constrained Dynamics]] —— 补上"刚度—步长/迭代"一致性与约束力估计，只加了每约束一个标量。
- PBF（2013）/ Small Steps（2019）/ 刚体 XPBD（2020）—— 均未入库，见 [[Position Based Dynamics]] 的 Learning Gap。

## Personal Knowledge State

`Normal`。入口很软、里面很硬：

- **已建立**：Vellum / Chaos 的"约束 + 迭代 + 子步"三件套每天都在用；"位置永远合法"这条稳定性直觉直接对应特效里的"不炸"。
- **今日新增**：$s$ 的分母为什么长这样（Newton 步 + 动量守恒 + 逆质量）；$k'$ 修的是迭代不是步长；多约束下 $k$ 改的是**不动点**。
- **未建立**：没写过求解器；$\sum w|\nabla C|^2$ 的数值条件（质量比 1000:1 时会怎样）没有一手数据。

## Learning Value

**最高。** 它是本库"物理"这条线上**唯一一个既在一线生产、又有干净数学的节点**，而且它的核心失效（刚度不是材料）可以直接在引擎里复现：

> **30 分钟实验**：同一块布，只改迭代数（20 / 40 / 80 / 160），截四张图。若"硬度"变了 ⇒ 你调的是求解器；这就是 XPBD 存在的理由。

## Visualization

![[Research/Files/Pictures/Müller 2007 — 图2 约束投影的逆质量加权.png]]

**读图**（图 2，Müller 等 2006）：两个点被距离约束 $C=|p_1-p_2|-d$ 拉住，修正量 $\Delta p_i$ 沿连线、按**逆质量**分配——轻的点让得多。这张图是式 (10)(11) 的全部内容，也是"为什么 $w=0$ 的顶点钉得住"的图示。

![[Research/Files/Pictures/Müller 2007 — 图3 弯曲与拉伸刚度可独立设置.png]]

**读图**（图 3，Müller 等 2006）：2×3 的布袋对照。**上排开弯曲、下排关弯曲**；每排从左到右拉伸刚度 $k_{\rm stretch}=1,\ 1/2,\ 1/100$。看到的正是"弯曲项与边长无关"带来的独立性——**拉伸变软时，褶皱的形态（弯曲）不跟着变**。这是"两个旋钮真的是两个旋钮"的实验证据。

![[Research/Files/Pictures/Müller 2007 — 图10 布料被立方体撕开再被球砸穿.png]]

**读图**（图 10，Müller 等 2006）：4264 顶点 / 8262 三角形的布，被附着的立方体撕开、再被抛来的球砸穿，**47 fps**（Pentium 4 3 GHz，含渲染）。撕裂的做法极简：边长超过阈值 ⇒ 过顶点做垂直于边的分割面，把上方三角形归原点、下方归副本。

## Notes

- **出版信息有两版**：VRIPHYS 2006 会议版（Eurographics，10 页，本页数字全部来自此版）与 JVCIR 18(2):109–118 (2007) 期刊版。引用时常看到"2007"，指的是期刊版；**引用页码时要说清是哪一版**。
- **作者单位是 AGEIA**（PhysX 的前身，2008 年被 NVIDIA 收购）——这篇论文从第一天起就是**中间件厂商的生产文档**，不是学术探索。这解释了它为什么把"稳定性/可控性"排在"精度"之前。
- **常见误读**：以为 PBD 是一个"刚体求解器"。原文的 Future Work 明写刚体未做；引擎里"PBD 解刚体"是后来的事（2020 刚体 XPBD）。
- **数字口径**：30 fps / 47 fps / >380 fps 都是"仿真 + 渲染"在 Pentium 4 3 GHz 上的整帧数，**没有拆开仿真与渲染**——所以不能拿来当"求解器有多快"的基准。
- 原文 PDF 与 6 张原图已入 `Research/Files/`。
