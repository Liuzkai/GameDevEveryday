---
type: concept
user_level: Normal
aliases: ["Position Based Dynamics", "PBD", "XPBD", "位置动力学", "基于位置的动力学", "约束投影求解器"]
prerequisites: ["Particle Systems", "约束求解（惩罚 / 拉格朗日 / 投影）", "显式与隐式积分"]
first_introduced: "2006（Müller / Heidelberger / Hennix / Ratcliff, VRIPHYS；AGEIA）"
---

# Position Based Dynamics

## Definition

**PBD 是一类把"约束求解"放在位置层的仿真方法族**：不算力、不解线性系统，先用显式 Euler 预测位置，再把点推到（近似）满足约束的位置上，最后用位置差反推速度。

$$x\ \xrightarrow{\ \text{预测}\ }\ p\ \xrightarrow{\ \text{投影（迭代）}\ }\ p'\ \xrightarrow{\ \text{反推}\ }\ v=(p'-x)/\Delta t$$

它的定义特征不是"用位置"（Verlet 也用位置），而是三件事同时成立：

1. **约束是一等公民**：一条约束 = 一个函数 $C(x)$ + 一个梯度 + 一个"软硬"参数，不经过能量与力；
2. **求解 = 反复投影**：Gauss-Seidel 逐条推，没有全局矩阵；
3. **状态永远合法**：投影后的位置一定在可行域里（近似），所以"炸"在结构上不成立。

## Core Principle

### 为什么它能成为通用框架：位置是公共货币

力法里每加一种约束，就要给它一套刚度单位与一套力项：距离是 N/m、弯曲是 N·m/rad、体积是 Pa、接触是 N——**它们不能直接相加，也不能在同一个循环里用同一种方式处理**。

PBD 把它们全部翻译成同一个东西：**"位置的一个函数 + 一个梯度"**。

| 约束 | $C(x)$ | 一句话 |
|---|---|---|
| 距离（拉伸） | $\lvert p_1-p_2\rvert-l_0$ | 边长 |
| 弯曲 | $\arccos(n_1\cdot n_2)-\varphi_0$ | 二面角（**与边长无关** ⇒ 与拉伸解耦） |
| 体积（内压） | $\sqrt{\sum (p_{t_1}\times p_{t_2})\cdot p_{t_3}}-k_pV_0$ | 整个网格一个约束 |
| 碰撞 | $(p-q_c)\cdot n_c\ \ge 0$ | 不等式，每帧现生成 |
| 固定 | ——（把 $w=0$） | 不是约束，是让约束推不动它 |

**这就是它赢的原因**：加一种新约束的成本，从"推导一个能量、求二阶导、处理单位"降到"写一个函数和它的梯度"。所以它能同时是布料、流体、毛发、破坏、刚体的框架，而不是又一个布料技巧。

### "软"的两种定义（本页的核心分界）

| | PBD（2006） | XPBD（2016） |
|---|---|---|
| 软硬参数 | $k\in[0,1]$，无量纲 | $\alpha$，单位 m/N（柔度） |
| 物理含义 | **每次迭代删掉多少残差** | **约束本身有多软**（一根刚度 $1/\alpha$ 的弹簧） |
| 迭代数变多 | 变硬（且更"闷"） | 行为不变 |
| 步长变小 | 变硬 | 行为不变 |
| 一句话 | **软 = 没解完** | **软 = 约束允许被违反** |

这个分界可以直接在引擎里验证：**同一件衣服只改迭代数**。形态变了 ⇒ 你在 PBD 语言里；形态不变 ⇒ 在 XPBD 语言里。

### 三本账

PBD 系没有"每步解一个系统"这一项，但有三个乘起来的因子：

$$\text{每帧成本}\ \approx\ \underbrace{N_{\rm constraints}}_{\text{规模}}\times\underbrace{n_{\rm iter}}_{\text{收敛}}\times\underbrace{n_{\rm substep}}_{\text{步长}}$$

**三者的视觉效果是耦合的**——这就是它便宜又难标定的根源：任何一项变动都会改变"看起来多硬"。隐式路线把这三项换成"一次线性求解"，代价是矩阵与内存，收益是刚度不随这三项漂移。

### 拿到一个 PBD 系求解器，先问四件事

1. **刚度是 $k$ 还是 $\alpha$？** 有单位吗？——决定参数能不能跨项目复用。
2. **改迭代数/子步，行为变吗？**——决定"同一资产换机器是否一致"。
3. **能读出约束力吗（有没有 $\lambda$）？**——决定可断关节、应力驱动特效、触觉能不能做。
4. **接触是零柔度吗？**——绝大多数 PBD 系里**是**（$\alpha=0$），于是"软接触/可压缩接触"不在框架内。

## Prerequisites

- [[Particle Systems]] —— 同一套"粒子 + 力 + 时间步"骨架；PBD 换掉的是"力"这一层。
- **约束求解三选一**：惩罚（不精确）/ 拉格朗日乘子（破坏正定性、要解系统）/ 投影（PBD）。三者的分工见 [[Contact Dynamics]] 的"转写光谱"。
- **显式 vs 隐式积分**：PBD 的第 ② 步是显式 Euler，第 ⑤ 步是"把位置改动记进速度"——所以它"不像显式也不像隐式"（原文自陈）。

## Historical Evolution

```text
1995  Provot            质量-弹簧 + 步后位置修正（只修过冲的弹簧）
2001  Jakobsen          Verlet + 直接改位置（GDC，游戏侧的第一份流行文本；只做距离约束）
2006  ★ PBD             一般约束框架 + 动量守恒分析 + AGEIA 的商业布料解算器
2013  PBF               同一套投影做流体：密度约束 + 人工压力 + 涡量
2016  ★ XPBD            柔度 α（有单位、与 Δt/迭代数无关）+ 约束力 λ
2019  Small Steps       多子步 × 每步 1 迭代：把"没解完"压到极小
2020  XPBD 刚体          同一框架做关节、摩擦、接触（刚体终于进来）
2010s Houdini Vellum / UE Chaos Cloth / PhysX Cloth —— 工业化：迭代 + 子步 + 最大距离
```

**读法**：这条线二十年只在修一件事——**"软"由谁定义**。答案换了三次（迭代规程 → 约束柔度 → 子步精度），而**"位置永远合法"这条底层承诺一次都没动**。

## Important Papers

| 论文 | 年份 | 角色 | 本库 |
|---|---|---|---|
| Jakobsen, Advanced Character Physics | 2001 | 游戏侧源头（Verlet + 位置修正，只做距离约束） | 未入库（GDC 讲义） |
| **Müller 等, Position Based Dynamics** | **2006** | **一般约束框架；动量守恒；"软 = 没解完"** | ✅ [[Müller 2007 — Position Based Dynamics]] |
| Macklin & Müller, Position Based Fluids | 2013 | 密度约束 + 人工压力；流体侧分岔 | 未入库 |
| **Macklin 等, XPBD** | **2016** | **柔度 $\alpha$；约束力 $\lambda$；$\alpha=0$ 退化成 PBD** | ✅ [[Macklin 2016 — XPBD Position-Based Simulation of Compliant Constrained Dynamics]] |
| Macklin 等, Small Steps in Physics Simulation | 2019 | 多子步单迭代；把人工柔度压下去 | 未入库 |
| Müller 等, Detailed Rigid Body Simulation with XPBD | 2020 | 刚体 + 关节 + 摩擦进同一框架 | 未入库 |

> 库内现在是**两端 + 中间两格**（2006 / 2016），前后各缺一格（2001 的流行源头、2019 的工程转折）。**PBF 缺页**意味着"流体也走这条路"在本库只有一句转述。

## Related Concepts

- [[Cloth Simulation]] —— PBD 的第一个应用场，也是它商业化的入口；那一页的"成本恒等式"与本页的"三本账"是同一个式子的两种写法。
- [[Contact Dynamics]] —— **必须分清的一对**：接触那页讲"力怎么定义"（互补条件、Delassus 矩阵、转写光谱），本页讲"约束怎么被满足"。PBD 把接触降级成"又一条不等式约束"，于是**没有 LCP、没有唯一解保证、没有真库仑摩擦**——换来的是一个循环解决所有约束。
- [[Multigrid]] —— 一对镜像：多网格把"每步代价"降到 $O(n)$（迭代数不算成本），PBD 把"每步代价"降到一个循环（但迭代数**直接就是**刚度）。**两者都在回答"迭代数算不算钱"，答案相反。**
- [[Baraff-Witkin — Large Steps in Cloth Simulation (1998)]] / [[Shewchuk 1994 — An Introduction to the Conjugate Gradient Method Without the Agonizing Pain]] —— 隐式路线的两端：刚度是材料属性、成本在解方程。**本页是它们的反面。**
- [[Fluid Simulation]] —— PBF 是本页在流体侧的实例（密度约束 + 人工压力），库内只有转述。

## Technologies

- **Houdini Vellum**（约束 + 迭代 + 子步；布料/毛发/颗粒共用一套求解器）
- **UE Chaos Cloth**（stiffness / damping / 迭代 / 子步；PBD 语言的典型界面）
- **PhysX Cloth**（PBD 的商业起点，作者单位 AGEIA 即 PhysX 前身）
- **刚体侧**：XPBD 刚体（2020）与 TGS（2019）把同一套"投影 + 柔度"推广到关节与接触

## Game Applications

- **角色服装 / 旗帜 / 绳网**：默认解算器。成本主要在**碰撞体数量与自碰撞**，不在布料分辨率。
- **可破坏 / 可断结构**：需要 $\lambda$（约束力）才能判"断不断"；纯 PBD 只能用"超长阈值"这种几何代理。
- **毛发 / 颗粒**：同一套约束（距离 + 弯曲），只是约束图不同。
- **跨平台一致性**：若项目要在 PC / 主机 / 移动间保持手感，**必须把"迭代 + 子步 + 刚度"当成一个整体来标定**，不能只搬 stiffness 数值。

## Personal Knowledge

`Normal`。入口在手上，机制今天才补：

- **已建立（实操）**：Vellum / Chaos 的三件套（约束、迭代、子步）；"加迭代会更硬"这条经验。
- **今日新增（机制）**：$s$ 的分母 = Newton 步 × 逆质量 × 动量守恒；$k'$ 只修迭代不修步长；多约束下 $k$ 改的是**不动点**；XPBD 的 $\tilde\alpha=\alpha/\Delta t^2$ 是量纲逼出来的，且 $\alpha=0$ 退化成 PBD。
- **未建立**：没写过求解器；$\alpha$ 与真实材料（N/m、Pa）怎么标定没有一手数据；质量比极端（1000:1）时的收敛行为没测过。

## Learning Gap

1. **Small Steps（2019）未入库**：本库多处写"XPBD + 小步长"，但"多子步单迭代为什么比少子步多迭代好"没有一手材料。
2. **PBF（2013）未入库**：流体走这条路的证据链是断的（[[Fluid Simulation]] 页只提了 Stam 与 SPH 两条线）。
3. **刚体 XPBD（2020）未入库**：本页说"刚体后来进来了"，但关节/摩擦怎么进柔度框架，库里没有原文。
4. **"$\alpha$ 怎么标定"零篇**：从杨氏模量到每约束柔度的换算（尤其弯曲约束）在工程上靠试，缺一份可引用的流程。
5. **质量比与收敛**：$\nabla CM^{-1}\nabla C^{\mathsf T}$ 在极端质量比下的条件数、"大质量比场景变慢"这条在接触侧有定性说法（[[Contact Dynamics]]），在 PBD 侧没有实测。

## Next Step

- **最小实验（30 分钟）**：同一块布，只改迭代数（20 / 40 / 80 / 160）截四张图；再只改子步（1 / 2 / 4）截三张。**目标：把"哪个旋钮在动收敛、哪个在动步长"变成手感。**
- **阅读顺序**：先 [[Müller 2007 — Position Based Dynamics]] 的 §3.1–3.3（算法 + 投影 + 刚度三节，10 页里约 3 页）；再 [[Macklin 2016 — XPBD Position-Based Simulation of Compliant Constrained Dynamics|XPBD]] 的式 (18) 与图 2/图 6（6 页，约 20 分钟）。
- **与已有工作的挂钩**：把本页的"三本账"接进 [[Cloth Simulation]] 的成本恒等式；把"$k$ 还是 $\alpha$"接进 [[Budget Units]] 的"可调性"一栏。

## Notes

- **建立于 2026-11-28**，由当日两篇经典（PBD 2006 / XPBD 2016）双向开域。此前库内 `XPBD` 出现在 16 个文件里、`Position Based Dynamics` 只出现在 2 个文件里（[[Cloth Simulation]] 与一篇前沿），**没有一页承载它**——而它是游戏里布料/毛发/软体/破坏的默认解算器。
- **命名**：本页用方法族的名字（Position Based Dynamics）而不是"约束投影"，因为库里已经有 [[Contact Dynamics]] 承担"投影作为一种转写"的角色；本页承担"这一族怎么运作、怎么选参数"。
- **术语**：compliance（柔度，m/N）与 stiffness（刚度，N/m）互为倒数；XPBD 里 **$\alpha$ 是柔度**，而 PBD 的 $k$ 两者都不是——它是一个无量纲的松弛因子。**这三者混用是本领域最常见的误读。**
