---
type: paper
title: "Fluid Control Using the Adjoint Method"
authors: [Antoine McNamara, Adrien Treuille, Zoran Popović, Jos Stam]
year: 2004
published: "2004-08（SIGGRAPH 2004）"
venue: "ACM SIGGRAPH 2004；ACM TOG 23(3):627–636"
url: "https://doi.org/10.1145/1015706.1015780"
code: ""
project_page: "https://grail.cs.washington.edu/projects/smokeControl/"
category: [fluid-simulation, optimal-control, adjoint, animation-authoring, differentiable-simulation]
importance: A
historical_importance: 5
game_relevance: 4
production_readiness: Research
user_level: Normal
status: unread
aliases: [McNamara 2004, Fluid Control Using the Adjoint Method, 伴随法流体控制, 伴随法控流体]
tags: [adjoint, fluid, control, animation, classical, differentiable-simulation]
---

# McNamara 2004 — 图形学第一次用伴随法求梯度

## TL;DR

**2003 年控烟的做法是"每个控制参数跑一遍仿真"；2004 年这篇把成本变成"每个梯度跑两遍仿真"——与参数个数无关。**

它的全部技术内容，是一段**线性对偶**：

$$\text{想算 } g^\top B\ \text{ 其中 } AB=C \qquad\Longleftrightarrow\qquad \text{解 } A^\top s=g\ \text{ 然后算 } s^\top C$$

左边要对**整个矩阵** $B$（每个参数一条状态序列）求解，右边只是**一个**线性系统。**这就是"可微仿真"在图形学的出生方式**——不是因为它更精确，是因为**只有它能扩展到几十万个参数**。

而在做这件事的时候，它**悄悄替后来二十年的整个领域选好了"微分的是哪个对象"**：

- **线性求解器当成精确算子**（投影、扩散的伴随 = 前向本身，"我们没觉得有必要对求解器本身求导"）；
- **快速步进的堆序冻结**（"充分小的扰动不改变遍历顺序，所以我们把顺序当固定，忽略 min-heap"）。

这两句话在 2026 年有了名字：**A1\*×B3**（见 [[2026-11-13-Shu — Same Physics Different Gradients A State-of-the-Art Report on Differentiable Physics Simulation|Shu 2026]] 的表 2）。**这篇论文是那一格的原始样本。**

## Problem

物理动画的四十年把**正演**做快了：给定初始状态和参数，积分器往前推。但动画师要的是**反的**：

> "让这团烟升起来，正好在这几个时刻凑成这只兔子。"

做法（Treuille 2003）是给仿真加一组**高斯风场**作为控制量 $u$，定义一个"离关键帧有多远"的目标 $\phi(u)$，然后做梯度下降。**问题在梯度**：

- $u$ 的维度是"网格 × 时间"，**几十万到上百万**；
- 有限差分要对每个参数各跑一遍**完整仿真**——成本 $O(p)$ 条轨迹；
- 结果就是：**2D 勉强、3D 不可能**。

原文把这条写得很直白：*"previous systems required a derivative simulation for each variable, making control feasible only at relatively coarse scales."*

## Historical Context

```text
最优控制（Lions 1971）          —— 伴随法在数学与 CFD 里已经成熟
        ↓
Giles & Pierce 2000             —— 连续 vs 离散两种伴随的系统综述（本篇"照它的推导写"）
        ↓
Treuille / McNamara / Popović 2003 —— 关键帧控烟：目标有了、梯度是前向差分
        ↓
★ 本篇 2004                      —— 换伴随，成本与参数个数脱钩；第一次控自由表面液体
        ↓
Wojtan 2006                     —— 积分器互换（前后向不是同一个格式）
        ↓
2019–21 DiffTaichi / DiffPD / DiffCloth —— 编译器级与方程级两条支线
        ↓
2026  Shu et al. STAR           —— 回头给这篇贴上 "A1*×B3"
```

**二十二年间，这篇的"参照物选择"一直没有名字，直到有人给它起了名字。**

## Previous Work（三条都不够）

1. **Barzel 1996 / Popović 2000 / Chenney-Forsyth 2000**：刚体控制的各种范式——**给了"控制什么"的语言，没给"大参数空间怎么办"**；
2. **Foster-Metaxas 1997 / Foster-Fedkiw 2001**：直接设流体参数或在指定位置设速度——**局部、手工、不支持高层目标**；
3. **Treuille 2003**：关键帧控烟，范式对了——**但梯度是前向差分，只能做 2D**。

## Core Idea

### 一、线性对偶（把"每个参数一遍"换成"一次求解"）

设状态按 $q_{i+1}=f_i(q_i,u)$ 演化。把所有状态摞成 $Q$、所有步摞成 $F$，则仿真是**一个约束**：

$$Q=F(Q,u)$$

目标 $\phi(Q,u)$ 的全导数是

$$\frac{d\phi}{du}=\frac{\partial\phi}{\partial Q}\frac{dQ}{du}+\frac{\partial\phi}{\partial u}$$

直接算要**整个 $\frac{dQ}{du}$ 矩阵**（每个参数一条完整状态序列）。但约束的微分给出

$$\Big(I-\frac{\partial F}{\partial Q}\Big)\frac{dQ}{du}=\frac{\partial F}{\partial u}$$

**这正是"矩阵版的 $Ab=c$"**，于是可以用对偶：

$$\boxed{\ \frac{d\phi}{du}=R^\top\frac{\partial F}{\partial u}+\frac{\partial\phi}{\partial u}\qquad\text{其中}\qquad\Big(I-\frac{\partial F}{\partial Q}\Big)^{\!\top}R=\Big(\frac{\partial\phi}{\partial Q}\Big)^{\!\top}\ }$$

$R$ 就是**伴随向量**。把它按时间拆开，得到逐时间的递推：

$$r_n=\Big(\frac{\partial\phi}{\partial q_n}\Big)^{\!\top},\qquad r_i=\Big(\frac{\partial f_i}{\partial q_i}\Big)^{\!\top}r_{i+1}+\Big(\frac{\partial\phi}{\partial q_i}\Big)^{\!\top}$$

⇒ **前向按时间正着算、伴随按时间倒着算**，最后 $r_1$ 出来时 $\phi$ 和 $d\phi/du$ 同时到手，**"代价基本上是算两遍 $\phi$"**（原文）。

> **这就是"目标维度 1、参数维度 $p$"时的全部秘密**：伴随把 $O(p)$ 次仿真的账，换成了 $O(1)$ 次仿真 + 一条长度为 $T$ 的倒扫。

### 二、内存：唯一真正的代价，而且它被"粒度"决定

**必须存前向状态**（递推里 $r_i$ 要用 $q_i$）⇒ **内存随时程线性增长**。原文自己写：*"This is the main drawback of the adjoint method."*

它的应对不是断点重算算法，而是**一个更根本的选择：把 $f_i$ 定义在多粗的粒度上？**

- **最细**：每条机器指令一个函数 = 反向模式 AD ⇒ 内存爆炸，"除最小问题外不可行"；
- **本篇的选择**：**一个 $f_i$ = 一个流体操作**（`APPLYCONTROL → ADVECT → DIFFUSE → HEAT → PROJECT → MATCHKEYFRAME`）⇒ 每个操作自己决定"全存 / 存一部分现算" ⇒ 这就是**检查点（checkpointing）**的框架。

> **判据：内存不是伴随法的属性，是"你把反向的记账单位定在哪一层"的属性。** 这一句话在 2026 年被 [[2026-11-13-Shu — Same Physics Different Gradients A State-of-the-Art Report on Differentiable Physics Simulation|Shu 2026]] 正式拆成独立的一轴（"状态恢复"），并明确**它不决定参照物、也不由实现方式决定**。

### 三、逐操作的伴随（"逐个转置"）

| 操作 | 伴随 | 备注 |
|---|---|---|
| `HEAT`（速度 += $h\rho$） | 矩阵转置 $H^\top$ | 线性的，直接转 |
| `GRAVITY` | 恒等 | 力不连续依赖状态 |
| `PROJECT` / `DIFFUSE`（解线性系统） | **与前向相同** | 因为算子对称；**"这要求一个精确的线性求解器"** |
| `ADVECT`（半拉格朗日回溯） | 线性插值导数的转置 | 非线性，但链式可推 |
| `REDISTANCE`（快速步进） | **冻结遍历顺序后逐体素反传** | 见下 |
| `APPLYCONTROL` | 不改伴随态，但**唯一给梯度累加的地方** | $d\phi/du \mathrel{+}= r^\top M_t+\alpha u M_t^\top M_t$ |

### 四、快速步进的伴随：全篇最漂亮的一处"取巧"

重距离化（redistancing）用快速步进法：从界面出发，按距离递增的顺序解一族二次方程，**用一个 min-heap 管遍历顺序**。这在字面上是**不可微**的（顺序是离散的、heap 是离散的）。

原文的解法只有两句话：

> *"The key insight is that **sufficiently small perturbations of the water surface do not in general change the traversal order**. Therefore, when linearizing this operation, **we consider the order fixed** and view each redistanced voxel as a small differentiable function of its neighbors, **ignoring the min-heap**."*

而且它还**自己说了这个假设什么时候破**：

> *"while small perturbations of the level set do not in general change the traversal order, **in some cases they do, an extreme example being topological changes**. In practice, however, we found that our approach to fast marching yielded derivatives no less accurate than those for the other operations."*

**这是"参照物"这个概念在 2004 年的雏形**：它没有对"真实程序"求导（真实程序里顺序会变），它**对"顺序冻结后的那个程序"求导**，并明确声明了这两个对象的差在哪、有多大（"拓扑变化时"）、以及实测上是否可以接受（"不比其他操作更差"）。

## Technical Approach

**目标函数**（两项：贴合关键帧 + 惩罚过度控制）：

$$\phi(u)=\frac12\sum_{t=0}^{n}\Big(\big\|W_t(\gamma(q_t)-\gamma(q_t^*))\big\|^2+\alpha\|M_tu\|^2\Big)$$

- $W_t$：逐时刻的权重矩阵（可只匹配速度、可加权某些空间区域、可整帧忽略）；
- $\alpha$：平滑项权重——**它其实是在问"这段动画该多服从物理"**；
- $\gamma$：**状态预处理**——烟用恒等；**水不能直接比符号距离场**（那不是体积差），所以取

$$\gamma(x)=\frac{2}{\pi}\arctan(x)$$

一个可微的 S 形函数，把正区推到 $+1$、负区推到 $-1$ ⇒ **目标比较的是"正负区域的不一致"，也就是体积差**，而不是两个距离场逐点的差。

> **这一处小设计值得单独记**：**当你手里的量只是"间接"代表你要的东西时，先换一个单调变换把它变成你要的东西，再算误差。** 直接比 SDF 是"看起来对、语义错"的典型。

**控制量**：高斯风场（来自 Treuille 2003）+ **source**（直接加在 $\rho$ 网格上，只对 level set 有用——因为重距离化会把远离界面的改动抹掉，**于是 source 只能影响界面本身**，往里或往外推一点）。烟不允许 source，否则"删掉当前位置的烟、在关键帧处加回去"就平凡可解了。

> **这一条是"给优化器留后门"的反面教材，也是一条通用检查：你允许的每一个控制自由度，都要问一句"它会不会让问题变成作弊"。**

## Key Contribution

1. **把最优控制的伴随法引入图形学**，让梯度成本与参数个数脱钩——从"每参数一条仿真"到"每个梯度两次仿真"；
2. **第一次完整控制自由表面液体**（level set + Navier–Stokes），其中包括对快速步进这种离散算法的伴随；
3. 一套**逐操作伴随**的工程模板（每个仿真算子各自负责自己的转置与状态存储）；
4. 一个**规模证据**：150 万控制参数、600 MB 内存、$45\times50\times36$ 网格、46 个时间步。

## Why It Works（数字）

| 例子 | 控制参数 | 网格 / 步数 | 内存 | 耗时 |
|---|---|---|---|---|
| 黏土落成十字 / 环 / 人形 | 30 万+ | $50\times23\times50$ / 10 步 | ~200 MB | — |
| 烟聚成兔子 / 犰狳 | 10 万 | $50^3$ / 单关键帧 | ~600 MB | **约 2 小时**（兔子） |
| 烟的"出拳" | 60 万+ | 9 万体素 / 20 步 | ~550 MB | — |
| **水 / 烟的奔跑人形**（封面图） | **150 万** | $45\times50\times36$ / 46 步 | **600 MB** | **约 2 天**（水人） |

- **2D 时可以"给每个仿真变量都配一个控制变量"**（原文）——这在 2003 年的前向差分下是不可想象的；
- 优化器用**有限内存拟牛顿（L-BFGS 类）**，与 Treuille 2003 一致；
- 运行时长"不是紧的"——作者自陈收敛判据设得过严，黏土那组"早就在收敛前找到了好结果"。

## Limitations（三处冷水）

**冷水一：水比烟难，而原因是"离散集合在变"——这正是今天叫参照物的那件事。**

原文 §8 写得很清楚：水的 `PROJECT` 与 `DIFFUSE` **也是不连续的**，因为线性系统**只在液体内部求解**；表面稍微一动，内部单元集合就变，于是解的是**另一个（尽管相似的）线性系统**。

后果是量化的：*"With smoke simulations, the optimizations could be tuned to an almost arbitrary precision, while **water simulations often could only reach a certain level of detail**."*

⇒ **同一套伴随代码、同一个参照物选择，在"离散结构会变"的问题上精度就掉一档。** 这是 [[2026-11-13-Shu — Same Physics Different Gradients A State-of-the-Art Report on Differentiable Physics Simulation|Shu 2026]] 把接触单独列成一根正交轴的历史依据之一。

**冷水二：那个"精确线性求解器"的假设，是整篇最脆弱的一环，而它只用一句话带过。**

`PROJECT` / `DIFFUSE` 的伴随等于前向，前提是**算子对称**，而且**要求线性求解器足够精确**。原文紧接着说：*"Another approach would be to take the adjoint derivatives through the linear solver itself. However, we did not find this necessary."*

**这就是 A1\* 的原始出处**：它把"求解到容差的迭代"替换成"精确线性算子"来求导。2026 年的报告专门为这一格发明了一个带星号的标签，理由写得很硬：**"这个理想化程序与有限容差程序之间的差，不由求解器容差单独决定——系统的条件数与残差对参数的敏感度也进来。"** 二十年后的读者能看出这一点，2004 年的作者只能给一句"实测没必要"。

**冷水三：验证是"看起来对"，不是"导数对"。**

全篇的结果都是**渲染图与关键帧的贴合**（黏土落成人形、烟聚成兔子、水滴从水花里升起）。**没有一处把伴随梯度与有限差分做对照**——除了快速步进那句"不比其他操作更差"（而这本身也是一个相对判断，没有绝对基准）。

⇒ **它的说服力来自"这套东西真的能做出以前做不出的动画"，而不是"这个梯度是对的"。** 这个传统一直延续到今天：[[2026-11-13-Shu — Same Physics Different Gradients A State-of-the-Art Report on Differentiable Physics Simulation|Shu 2026]] 的清单第 7 项专门点名这件事——*"一个可微仿真器的论文，如果没有和不使用梯度的方法在同一任务同一预算上比过，就还没有证明梯度有用。"*

## Game Development Relevance

**中高，且位置很明确：离线授权（authoring）工具，不是运行时。**

- **美术交付**：给一个目标形状 / 一个关键帧序列，让仿真自己找外力——这是"程序化 VFX 预演"的最早可行版本。今天游戏里做"烟聚成 logo""水花按分镜走"仍然靠手工调，这篇是那条路的起点；
- **参数的离线标定**：同一套伴随机器可以反过来用——固定动画、反解材质（[[Cloth Simulation]] 的 stiffness / damping 拟合就是这个形状）；
- **物理角色控制的前身**：把"控制量"从风场换成关节驱动、把"关键帧"换成"参考动作"，就是 [[Physics-based Character Animation]] 的轨迹优化；
- **不能用于运行时**：内存随时程线性、梯度要两次仿真、优化要几百次迭代。**它在游戏里的正确位置是 DCC 与离线工具链。**

## Unreal Engine Relevance

- **最接近的落点**：Niagara 的流体/烟预演 + 离线烘焙。把"目标形状"当关键帧、把风场当控制量，产出**烘焙好的外力场序列**（不是运行时求解）；
- **Chaos 侧**：布料 / 软体的参数反解同构（目标 = 参考动画，参数 = 材质）；
- **不该做的**：指望在运行时跑伴随。**2004 年的 2 天/条，换到今天的硬件也还是分钟级**——它是工具，不是帧内技术。

## Technology Evolution

```text
Lions 1971            最优控制的伴随法（数学）
        ↓
Giles & Pierce 2000   连续 / 离散两种伴随的系统化
        ↓
Treuille 2003         关键帧控烟（前向差分，2D）
        ↓
★ McNamara 2004       离散伴随进图形学；A1*×B3 的原始样本
        ↓
Wojtan 2006           积分器互换：前后向可以不是同一个格式
        ↓
2019–2021             DiffTaichi（编译器级）/ DiffPD·DiffCloth（方程级）
        ↓
2024                  Newbury 综述：领域自认成型
        ↓
2026                  Shu STAR：给 2004 的选择贴上标签
```

## Relationships

### Based On

- **最优控制理论**（Lions 1971）与 **CFD 里的离散伴随**（Giles & Pierce 2000）——本篇明说"我们的离散伴随推导是照着他们写的"；
- [[Stam — Stable Fluids (1999)]] 的半拉格朗日 + 隐式粘性 + 投影四步——**被逐操作转置的就是这四步**；
- [[Fluid Simulation]] 的四步分解——本篇的 `ADVECT → DIFFUSE → HEAT/GRAVITY → PROJECT` 是它在烟/水两种介质上的具体化。

### Extends

- **Treuille / McNamara / Popović 2003（关键帧控烟）**：换了梯度计算，别的都保留（同一套高斯风场控制、同一个 L-BFGS）。

### Contrasts

- [[2026-11-13-Shu — Same Physics Different Gradients A State-of-the-Art Report on Differentiable Physics Simulation|Shu 2026]] 的 **A2 路线**（方程级伴随，用隐函数定理在收敛根上求导）：**本篇走的是 A1\*（执行计算的理想化版本）**。差别不在实现好坏——**A2 在"前向被截断"的实时设定下，与真导数可以差到几乎正交**（Shu 图 5）。

### Related

- [[Differentiable Rendering]]：**同一套"反向 = 一条伴随输运"的语言**，但那边的不连续是可见性、这边是接触与离散集合；库内 22 处提"伴随"全部在光传输侧，**这一篇是物理侧的第一个锚点**；
- [[Linear Transport Theory]]：伴随算子的母语坐标。

### Followed By

- **Wojtan 2006**（积分器互换——"解耦"最早的样子）；
- DiffPD / DiffCloth / DiffXPBD / Nimble（方程级）；
- **整条"可微仿真"线**，以及 2026 年的 STAR。

## Personal Knowledge State

**Normal（机制 Easy / 参照物层 Normal）。** 这是全库**最浅的一篇可微仿真入口**：

- 不需要懂神经网络；
- 不需要懂渲染；
- 只需要**看懂一段线性对偶**（$Ab=c$ 换成 $A^\top s=g$）——而这个技巧在 [[Differentiable Rendering]] 的"重放"一节里已经出现过同类。

## Learning Value

**最高（★ 它给的是"成本随维度"的直觉，不是算法）**：读完能回答三个问题——

1. **为什么伴随法只在"目标标量、参数高维"时赢？**（反过来就是前向模式的主场）
2. **内存的账记在哪一层？**（粒度 = 你选的记账单位）
3. **"把离散决定冻结"算不算作弊？**（不算，但要声明它什么时候破——本篇给了"拓扑变化"这个答案）

## Visualization

![[Research/Files/Pictures/McNamara 2004 — 图1 用水与烟做出的奔跑人形.png]]

**读图**：封面图。**同一个动捕序列**，同一套伴随代码，只把介质换成水（上）与烟（下）——**150 万个控制变量、600 MB、46 个时间步、约 2 天**。两种介质各自的动态特征都保住了（烟在飘、水成珠），因为控制力只被允许"推"，不允许"改形状"。**这张图在 2004 年的意义是：它证明了"高维控制"不是理论上的可行性，是能出片的。**（图 1，McNamara et al. 2004）

![[Research/Files/Pictures/McNamara 2004 — 图2 前向状态必须存储供反向使用.png]]

**读图**：伴随法全部代价的来源。$q_0\to q_1\to\cdots\to q_n$ 前向算完后**必须留着**（虚线向下指的箭头），因为反向的 $r_i$ 递推要用到 $q_i$；然后 $r_n\to\cdots\to r_1$ 倒着扫。**"内存随时程线性"这句话就是这张图。** 而本篇的应对不是算法，是**把 $f_i$ 定义在"一个流体操作"这个粒度上**，让每层自己决定存多少。（图 2，McNamara et al. 2004）

![[Research/Files/Pictures/McNamara 2004 — 图3 快速步进的体素遍历顺序与信息流.png]]

**读图**：快速步进法的遍历顺序（红曲线是界面，数字是体素被处理的次序，蓝箭头是信息流）。**注意它是有向的、由界面往外扩散的**——这既是前向的计算顺序，也是反向要**逆着走**的顺序：伴随从最后处理的体素开始，往回走到界面。而整张图最关键的隐含前提是那句"**充分小的扰动不改变这个顺序**"——**顺序一变，这张图的转置就整张失效。**（图 3，McNamara et al. 2004）

![[Research/Files/Pictures/McNamara 2004 — 图4 同一初始状态的黏土落成三种形状.png]]

**读图**：同一初始状态（一滴黏土落到平面），只换关键帧目标，优化出**十字 / 圆环 / 人形**三种静止构型（30 万+ 控制参数、$50\times23\times50$、10 步、~200 MB）。**这张图是"目标函数决定一切"的最干净展示**——仿真器、控制参数化、优化器全没变。**同时也暴露了那个冷水：作者发现"落地之后再施加控制"效果好得多，因为撞击是个高度非线性的冲击，线性化在那里是坏近似。**（图 4，McNamara et al. 2004）

## Notes

- **"两遍 $\phi$"这个说法值得记牢**：伴随法的成本是 $2\times$ 目标函数，**与 $p$ 无关**。所以判断一篇"可微 XX"论文值不值，第一问是"你的 $p$ 有多大"——$p$ 小的时候它反而是亏的。
- **一处可以立刻搬的工程习惯**：$\gamma(x)=\frac{2}{\pi}\arctan(x)$ 把 SDF 变成"正负区域指示"。**凡是"用手里的量近似代表另一个量"的目标函数，都该先检查这个变换。**
- **与前一天的接力**：昨天 [[Blinn 1978 — Simulation of Wrinkled Surfaces|Blinn 1978]] 写下"bump 贴图不能平均"、48 年后还有人欠着；今天这篇写下"水只能做到某个细节层次"、22 年后这个坑被正式命名为"参照物"。**同一个模式：**作者当场写下的那句边界，比论文的方法寿命长。
