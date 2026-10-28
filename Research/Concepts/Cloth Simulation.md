---
type: concept
user_level: Normal
aliases: ["Cloth Simulation", "布料仿真", "Cloth", "布料解算", "Garment Simulation"]
prerequisites: ["Particle Systems", "数值积分（显式/隐式）", "约束求解（惩罚/投影/拉格朗日）"]
first_introduced: "1987（Terzopoulos / Platt / Barr, SIGGRAPH）"
---

# Cloth Simulation

## Definition

**布料仿真 = 在一组"几乎不可伸长"的强约束下，求一堆受碰撞约束的粒子轨迹。**

它不是"软体的一种特例"那么简单。布料有三个别的形变体没有的性质：

1. **各向异性到极致**：沿纤维方向极硬（棉布拉伸 1–2% 就到底），弯曲与剪切极软。刚度谱跨三四个数量级。
2. **薄**：厚度可忽略，所以自碰撞是"面与面"，不是"体与体"——这是它比软组织难的地方。
3. **要可控**：美术要"再飘一点/再厚一点"，所以它必须有一组**人能懂的旋钮**（刚度、阻尼、摩擦、最大距离）。

## Core Principle

### 三件事，缺一不可

| 层 | 在算什么 | 关键量 | 典型失败 |
|---|---|---|---|
| **本构** | 拉伸/剪切/弯曲各自多硬 | 条件向量 $C(x)$ + 刚度 $k$ | 橡皮布（拉伸太软）、纸片感（弯曲太硬） |
| **积分** | 时间怎么推进 | 步长 $\Delta t$、每步迭代数 | 抖动/爆炸（步太大）、太慢（步太小） |
| **接触** | 不穿透 + 粘/滑 | 法向约束 + 切向摩擦 $\mu$ | 穿透、贴身穿模、衣领下滑 |

### 成本恒等式

$$\boxed{\text{每帧成本}=\underbrace{\text{步数}}_{\text{由最硬约束决定}}\times\underbrace{\text{每步代价}}_{\text{由规模与迭代数决定}}}$$

这条式子决定了**所有**布料优化的路线，因为两条路只能选一条：

| 路线 | 步数 | 每步代价 | 规模定律 | 代表 |
|---|---|---|---|---|
| **显式** | 多（$\Delta t\lesssim\sqrt{m/k}$） | 便宜（一次力求值） | $O(n)$ 步 × $O(n)$ = **$O(n^2)$** | Breen 1994 / Provot 1995 / Volino 1995 |
| **隐式** | 少（由稳定性而非刚度定） | 贵（解线性系统） | $O(n^{1.5})$（CG 收敛率） | **Baraff-Witkin 1998** |
| **投影（PBD/XPBD）** | 多（子步）但每步极便宜 | 极便宜（无矩阵） | $O(n)$ × 迭代数 | Müller 2007 / Macklin 2016 / Chaos Cloth |
| **学习（神经）** | 1（一次前向） | 中（网络推理） | 与规模弱相关 | HOOD / ContourCraft / Truong 2026 |

**关键判据：先问"你的瓶颈是步数还是每步代价"**。调迭代次数是在动第二项，调子步/刚度是在动第一项——两者不可互换。

### 成本会搬家，不会消失

Baraff-Witkin 1998 的 Table 1 是这条最干净的证据：换成隐式后步数掉到 2–3 步/帧，但**求解（CG）与碰撞检测从 0% 长到 70–85%**。所以任何"我们换了个积分器所以快了 N 倍"的说法，都要追问"那每步贵了多少、贵在哪个格子里"。

### 四种失败模式（按出现频率排序）

| 现象 | 根因 | 先查什么 |
|---|---|---|
| **穿透**（衣服陷进身体） | 接触约束不足 / 碰撞体缺失 / 步太大 | 碰撞体是否覆盖关节内侧；子步数 |
| **橡皮布**（拉伸得离谱） | 拉伸刚度被调低换帧率 | 拉下摆看回弹是否过冲 |
| **抖动/爆炸** | 步长超过稳定界（显式）或约束冲突 | 最大 Δt；同一点是否被两个约束抢 |
| **滑移不真实**（该粘不粘、该滑不滑） | 摩擦是近似（PBD 切向投影）或训练时被平均掉 | 摩擦系数与法向载荷的**序关系**（坡面 vs 平地） |

## Prerequisites

- [[Particle Systems]] — 同一套"粒子 + 力 + 时间步"骨架；布料多的是连接与静止坐标 $(u,v)$。
- **数值积分**：显式 Euler / 中点法 / 后向 Euler 的稳定性差异（库内散见，无专页）。
- **约束求解三选一**：惩罚弹簧（不精确）/ 拉格朗日乘子（破坏正定性）/ 投影（PBD）。
- 三角网格与重心坐标（[[Mesh Parameterization]] 的 $u,v$ 坐标在此复用）。

## Historical Evolution

```text
1987  Terzopoulos / Platt / Barr《Elastically deformable models》
      连续体 PDE + 隐式积分 + 矩形网格；布料被当作"可变形曲面"的一种
1988  Terzopoulos & Fleischer：粘弹性/塑性/断裂（同一框架）
1992  Carignan 等：显式 + 各向异性阻尼；指出"阻尼不能惩罚刚体运动"
1994  Breen / House / Wozny：放弃连续体，用 Kawabata 实测织物参数做粒子系统（静态垂坠）
1995  Provot：mass-spring + **步后位置修正**（"先跑再拉回"这条路的起点）
1995  Volino 等：三角网格 + 中点法 + 碰撞检测/响应（第一批"穿在角色身上"的结果）
1998  **Baraff-Witkin：隐式 + 直接约束 + modified-PCG** → 步长与刚度解耦（[[Baraff-Witkin — Large Steps in Cloth Simulation (1998)]]）
2001  Meyer 等：各向异性应变能（"布"→"织物"）
2002  Bridson 等：鲁棒碰撞处理（连续碰撞、无穿透）
2007  Müller 等 PBD：把"解线性系统"换成"投影约束"——实时侧的分岔口（库外）
2016  Macklin 等 XPBD：补上"刚度—步长"一致性，PBD 才有了物理意义上的刚度（库外）
2010s Houdini Vellum / UE Chaos Cloth：PBD 系的工业化（迭代 + 子步 + 最大距离）
2022  SNUG / NCS：自监督神经布料（无切向接触）
2024  HOOD / ContourCraft：跨拓扑自监督 + 多层穿插（摩擦仍是训练常数）
2026  **Truong 等：把摩擦搬到推理之后**（闭式库仑修正 + 体表行走 + 限应变传播）（[[2026-10-24-Truong — Explicit Frictional Contact Modeling in Neural Garment Simulation]]）
```

**读这条时间线的正确方式**：不要读成"越来越准"，要读成"**每一次都把某个不可调的东西变得可调，同时把另一个可调的东西变得不可调**"——

- 隐式（1998）让**步长**可调（不再被刚度绑），代价是每步要解方程；
- PBD（2007）让**每次迭代**可调（不再解方程），代价是刚度变得依赖步长；
- 学习（2022+）让**每帧成本**可调（一次前向），代价是**μ/刚度这些旋钮被烧进权重**；
- 事后算子（2026）把**μ**找回来，代价是它不再是"解"（穿透率差 30 倍）。

## Important Papers

| 论文 | 年份 | 角色 | 本库 |
|---|---|---|---|
| Terzopoulos / Platt / Barr, Elastically Deformable Models | 1987 | 布料 = 连续体 + 隐式积分的起点 | 未入库 |
| Provot, Deformation Constraints in a Mass-Spring Model | 1995 | "步后修正"这条路的源头 | 未入库（Baraff-Witkin 引用 [13]） |
| **Baraff & Witkin, Large Steps in Cloth Simulation** | **1998** | **步长与刚度解耦；约束精确；成本搬家** | ✅ [[Baraff-Witkin — Large Steps in Cloth Simulation (1998)]] |
| Bridson 等, Robust Treatment of Collisions | 2002 | 无穿透的碰撞处理 | 未入库 |
| Müller 等, Position Based Dynamics | 2007 | 实时侧分岔口（投影取代解方程） | 未入库 |
| Macklin 等, XPBD | 2016 | 刚度—步长一致性 | 未入库 |
| **Truong 等, Explicit Frictional Contact in Neural Garment Simulation** | **2026** | **摩擦搬到推理后：闭式库仑 + 可调 μ** | ✅ [[2026-10-24-Truong — Explicit Frictional Contact Modeling in Neural Garment Simulation]] |

> 库内目前只有两端（1998 与 2026），中间 1995/2002/2007/2016 四个节点是**明确的缺口**——尤其 PBD/XPBD，因为游戏引擎里跑的是它们，不是 1998 那套。

## Related Concepts

- [[Skinning]] — **必须分清的一对**。蒙皮是"网格按骨骼权重变形"（零自由度、纯几何）；布料是"独立自由度 + 碰撞约束"（有自由度、要解算）。**衣服穿模看起来像蒙皮问题，八成是碰撞体问题**。
- [[Fluid Simulation]] — 同一个"稳定性 vs 步长"母题的两个答案：[[Stam — Stable Fluids (1999)]] 用"无条件稳定"取消步长限制（但步长大了会失真），Baraff-Witkin 用"隐式 + 约束"取消刚度限制（但每步变贵）。
- [[Particle Systems]] — 布料是"有连接的粒子系统"；Niagara 的约束模块与布料解算器在数学上是同一类东西。
- [[Neural Physics Simulation]] — 布料是这条线上最成熟的子域（有自监督目标、有明确的接触语义）。
- [[Animation Compression]] — 无关（仅备注：布料的"每帧成本"与动画压缩的"每帧字节"是两条独立的预算线）。

## Technologies

- **PBD 系（Chaos Cloth / Vellum）**：迭代投影 + 子步；参数是"迭代次数 / 子步 / 最大距离 / 摩擦"，不是"解线性系统"。**优点**：实时、稳定、旋钮直观；**缺点**：刚度依赖步长、摩擦是近似切向投影。
- **隐式系（离线渲染 / 影视）**：每步解稀疏系统，刚度不敏感；**优点**：大步、准；**缺点**：单帧几十秒量级，进不了实时。
- **神经系（HOOD / ContourCraft + 事后算子）**：一次前向出结果；**优点**：与网格规模弱相关；**缺点**：需要训练数据/自监督目标，且**可调参数容易丢**（除非像 Truong 2026 那样显式补回）。

## Game Applications

- **角色服装**：多层（内衣/外衣/披风）+ 自碰撞 + 与身体的滑动。成本主要在**碰撞体数量**与**自碰撞**，不在布料分辨率。
- **场景布料**（旗帜、帘子、绳网）：可只用"最大距离 + 少量迭代"，因为它们不需要真实褶皱。
- **VFX 布料**（破碎的披风、被撕开的帐篷）：与破坏系统耦合，需要"拓扑变化"，而 PBD 系的拓扑是固定的——这类通常走 Houdini 离线 + 顶点动画。
- **性能预算口径**：一台角色 1–2 ms（60 fps 的 6–12%）；多层服装按"每层一次解算"线性叠加。**这是"数量 × 单价"结构，与 [[Overdraw]] 的"层数 × 屏占比 × 单价"同构**。

## Personal Knowledge

`Normal`。已建立的连接：

- **成本结构**：每帧成本 = 步数 × 每步代价 —— 这条已经在 [[Real-Time VFX Performance Budgeting]] 的"粒子/发射器"账里用过，布料是同一个式子的另一个实例。
- **"成本搬家"直觉**：1998 年把成本从"步数"搬到"每步"，2026 年把它从"每步迭代"搬到"一次修正"——**你调 Niagara 的 substep / iteration 时其实在同一个平面上移动**。
- **未建立**：没做过布料解算实测；没有 PBD/XPBD 的一手材料（库内缺口）。

## Learning Gap

1. **PBD/XPBD 的机制**（库内缺口，且是游戏引擎实际在跑的）：为什么"投影"能替代"解方程"，$\alpha$ 柔度参数如何把刚度变成"与步长无关"。
2. **自碰撞的工程做法**：空间哈希 + 厚度 + 层级剔除；为什么自碰撞比布-体碰撞贵一个量级。
3. **拓扑变化的处理**（撕裂/破碎）：为什么 PBD 系难做，工业上怎么绕。

## Next Step

- **最小实验（30 分钟）**：UE 里同一件衣服，只改三个参数各一次——`迭代次数`、`子步/解算频率`、`摩擦系数`——记录"视觉变化"与"帧时间变化"。目标是把"步数 vs 每步代价"这条式子变成手感。
- **阅读顺序（按需）**：先 [[Baraff-Witkin — Large Steps in Cloth Simulation (1998)]] 的 §5.1（约束即质量）与 Table 1；再补 PBD 2007（库外）的"投影"一节；XPBD 只在需要"刚度为什么不随步长变"时读。
- **与已有工作的挂钩**：把布料的"层数 × 单价"接进 [[Overdraw]] 式的账本语言；把"摩擦是旋钮还是权重"接进 [[Budget Units]] 的"可调性"一栏。

## Notes

- **建立于 2026-10-28**，由当日经典（Baraff-Witkin 1998）与前沿（Truong 2026）双向开域。此前库内"Cloth"只在 [[DeRose — Subdivision Surfaces in Character Animation (1998)]] 的布料能量项与 [[2026-10-07-Simulation Methods for Multiphysics Phenomena in Visual Computing]] 的框架分类里被提及，**没有独立页**。
- **未入库的历史节点**（1995 Provot / 2002 Bridson / 2007 PBD / 2016 XPBD）在此仅作"时间线占位"，**没有读过原文，不写具体数字**；日后若入库需回来替换为一手结论。
- **术语**：cloth（布）与 garment（服装）在文献里常混用；差别在于 garment 强调"穿在人身上 + 与身体接触"，所以摩擦/接触是它的核心问题，而 cloth 可以只是"一块布"。
