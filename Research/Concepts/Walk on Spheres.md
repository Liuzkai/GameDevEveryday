---
type: concept
user_level: Normal
aliases: [Walk on Spheres, Walk-on-Spheres, WoS, 球面游走, 球面行走法, 无网格蒙特卡洛 PDE]
prerequisites: ["[[Rendering Equation]]", "[[Linear Transport Theory]]"]
first_introduced: "1956（Muller，Dirichlet 问题的连续 Monte Carlo 方法）；2020 进入几何处理（Sawhney & Crane）"
---

# Walk on Spheres

## Definition

**解椭圆型偏微分方程（Laplace / Poisson）的一种无网格 Monte Carlo 方法：只在你要的那几个点上求值，不建体网格、不装配矩阵、不求全局解。**

它的全部内容是一个等式——**调和函数的期望表示**：

$$u(x)=\mathbb E\big[g(X_\tau)\ \big|\ X_0=x\big]$$

- $X_t$：从 $x$ 出发的**布朗运动**；
- $\tau$：它第一次碰到边界 $\partial\Omega$ 的时刻（**首出时**）；
- $g$：边界上的 Dirichlet 数据。

**$u(x)$ 就是"从 $x$ 出发的布朗粒子，最后落在边界哪一点，那里的 $g$ 是多少"的期望。** 这个等式不是近似，是 Laplace 方程解的概率表示（与"平均场性质"是同一件事的两种说法）。

**算法本体只有四行**：

```
x ← 查询点
repeat until dist(x, ∂Ω) < ε:
    找以 x 为心、完全在 Ω 内的最大球 B(x, R)，R = dist(x, ∂Ω)
    在球面 ∂B 上均匀采一点，x ← 那一点
return g(x)      # 或对 N 条独立游走取平均
```

**没有网格、没有线性系统、没有时间步。** 每步的代价是"算一次到边界的距离"（对三角网格是一个点到三角形集合的最近距离查询），步数取决于几何。

## Core Principle

### 为什么"球面上均匀采一点"就够了

这是整套方法唯一需要记住的数学事实：

> **从球心出发的布朗运动，首次穿出球面的位置，在球面上是均匀分布的。**

也就是说，当球心就是起点时，Poisson 核是**常数** $1/|\partial B|$。**这让"一步"变成"在球面上均匀采一点"——不需要重要性采样、不需要算核、不需要拒绝采样。**

**而这个"均匀"只在球心成立。** 一旦起点偏离球心（$p\neq c$），退出分布就变成非均匀的 Poisson 核：

$$K_B(p,z)=\frac{1}{|\partial B|}\cdot\frac{R^2-\|p-c\|^2}{R\,\|p-z\|^3}\quad(\text{3D 球})$$

**这个偏移核是理解后续一切工作的钥匙**——今天那篇把 WoS 轨迹存成图的论文（[[2026-10-30-Liao — From Walks to Skeletons A Reusable Transport Architecture|Walks to Skeletons]]），它的"挂接重加权"用的就是这个比值 $K_B(p,z)/K_B(q,z)$。**圆心处的均匀性让算法便宜；偏离圆心时的非均匀性让"复用"变贵。**

### 它为什么"无网格"却仍然正确

因为它把"解 PDE"换成了"算期望"，而期望只需要**两件几何信息**：

1. **到边界的距离**（决定球多大）——对三角网格是最近点查询；
2. **边界上的取值** $g$（决定终止时的贡献）。

**它完全不需要"域的内部结构"**。这正是它能处理"体网格生成很贵或很脆"的几何的原因（薄壁、复杂内部结构、CAD 模型、点云隐式曲面）。

### 代价在哪里

| 维度 | 表现 | 原因 |
|---|---|---|
| **每样本成本** | 低（每步一次最近距离查询） | 一步 = 一次查询 + 一次采样 |
| **收敛率** | $O(N^{-1/2})$（和所有 MC 一样） | 方差主导，不是离散化误差 |
| **精度 vs 成本** | 要 10× 精度 = 100× 样本 | 这是 MC 的宿命 |
| **小特征** | **方差爆炸** | 小球 ⇒ 步数暴涨；薄壁 ⇒ 球很小 |
| **光滑大尺度场** | 差 | 每点独立采样，无法摊销 |
| **Neumann 边界** | 需要改造（Walk on Stars） | 反射边界没有"终止"这个事件 |
| **时间依赖** | 需要改造（带时间预算） | 布朗运动的"剩余时间"必须进状态 |

**一句话：WoS 用"每个点独立、无全局结构"换来了"任意几何、任意精度、天然并行"，代价是"没有摊销、没有全局信息"。**

## Prerequisites

- **Laplace 方程与调和函数**：$\Delta u=0$ + 边界条件；"内部值 = 边界值的某种平均"这个直觉是全部基础；
- **布朗运动与首出时**：随机游走、首达时间、马尔可夫性。**马尔可夫性是 WoS 能"分步"的全部理由**（走了多远不影响后面怎么走）；
- **Poisson 核 / 调和测度**：球面上的退出分布；圆心处退化成均匀；
- [[Rendering Equation]]（**结构同构**：都是"随机游走 + 终止事件上的贡献"，只是渲染的终止事件是"打到光源"）；
- [[Linear Transport Theory]]（**同一套语言**：随机游走、边界条件、"把表面问题重写成体积问题"）。

## Historical Evolution

```text
1956  ★ Mervin E. Muller《Some Continuous Monte Carlo Methods for the Dirichlet Problem》
        —— 提出"在球面上跳"这个做法；此后 60 年在数值分析界安静存在
1960s-2000s  零星使用（主要在统计物理与工程），图形学几乎不知道
2020  ★★ Sawhney & Crane《Monte Carlo Geometry Processing》
        —— 把它带进几何处理：强调"无网格 / 点求值 / 渐进 / 可并行"，
           并给出与网格法对比的完整论证（这就是本库 2026-09-15 那篇的前身）
2022  Sawhney et al.：空间变系数（$\nabla\cdot(a\nabla u)=0$）—— 步长要按系数调
2022  Qi et al.《A Bidirectional Formulation for Walk on Spheres》
        —— 反向 Green 输运：可以从"源"侧采，补正向采不到的贡献
2023  ★ Sawhney et al.《Walk on Stars》
        —— Neumann 边界：用"星形域"代替球，让反射边界也能终止
2023  Miller et al.《Boundary Value Caching》——  ★ 第一次"存"：缓存边界上的值与导数
2023  Bakbouk & Peers《Mean Value Caching》——  存体积均值
2025  Zhou et al.《Harmonic Caching》——        存局部调和展开
2025  Miller et al.《Differential Walk on Spheres》—— 可微：几何/边界敏感性
2026  Czekanski et al.《Talking to Neighbors》—— 邻居共享游走（方差缩减）
2026  Huang et al.：Probe-based WoS（固定球面上的中间状态）
2026  Jambon et al.《WoDS》：子域上可复用的首达算子 + 稀疏确定性耦合
2026  Saeed et al.：cage cutting，保留未被切割影响的路径
2026  ★ Liao et al.《From Walks to Skeletons》—— 把轨迹存成可查询/细化/压缩/修复的 DAG
2026  Zhou / d'Eon / Sawhney / Jarosz：**时间依赖**（每步带时间预算，见 2026-09-15 那篇）
```

**两条演化轴，值得分开记**：

```text
轴一：能解什么      Laplace → 变系数 → Neumann（WoSt）→ 时间依赖（热方程）→ 可微
轴二：怎么变便宜    纯采样 → 缓存值（2023）→ 缓存展开（2025）→ 缓存轨迹图（2026）
```

**轴二的方向极其清楚：粒度一路上移（值 → 函数 → 邻域 → 图）。而每一次上移都在做同一个交易——用"构建成本前置"换"后续查询变便宜"。**

## Important Papers

| 论文 | 位置 | 贡献 |
|---|---|---|
| Muller 1956（库外） | 起点 | "在球面上跳"这个方法本身 |
| [[2026-09-15-Grid-Free Monte Carlo for Time-Dependent Diffusion]] | 轴一：加时间 | **给每条游走一个时间预算**，把 WoS 推到瞬态热方程；一次估计多个时刻 |
| [[2026-10-30-Liao — From Walks to Skeletons A Reusable Transport Architecture\|Walks to Skeletons]] | 轴二：加记忆 | **轨迹 → DAG**；改边界值只换叶子（$O(V+E)$），几何编辑局部修复；累计 8.1×（8 次求解） |
| Sawhney & Crane 2020（库外） | 轴一：奠基 | 把这套方法带进几何处理，建立"无网格 MC 几何处理"这个方向 |
| Sawhney et al. 2023《Walk on Stars》（库外） | 轴一：边界条件 | Neumann 边界（星形域） |
| Qi et al. 2022（库外） | 轴一：方向 | 双向形式（反向 Green 输运） |
| Miller et al. 2023 / Bakbouk 2023 / Zhou 2025（库外） | 轴二：前身 | 三类缓存（边界值 / 体积均值 / 局部展开） |

## Related Concepts

- [[Linear Transport Theory]] — **同一个随机游走在两个域里**：渲染里的游走终止于"打到光源"，PDE 里的游走终止于"碰到边界"。**"表面 vs 体积"和"输运 vs 椭圆 PDE"是两对不同的二分，别混**；
- [[Rendering Equation]] — **结构同构的母本**。渲染方程的 Neumann 级数解 = 路径积分；Laplace 解的期望表示 = 布朗游走。**两者都是"随机过程 + 边界/源项"，只是一个在光线空间、一个在空间域**；
- [[2026-10-30-Liao — From Walks to Skeletons A Reusable Transport Architecture|Walks to Skeletons]] — **本页的直接下游**：它证明"轨迹值得存"，并给出存了之后能做的五种局部操作；
- [[ReSTIR]] — **"复用"的另一个域**（像素/光源样本），与 WoS 的轨迹复用共享同一套合法性论证（重要性权重 + 上界）；
- [[Neural Physics Simulation]] — **对照路线**：学一个解算子代替采样。**WoS 的优势是"任意几何、任意精度、无需训练"，劣势是"每点独立、无摊销"——而学习路线的优劣正好相反**；
- [[Differentiable Rendering]] — 同一件事在 PDE 侧的对应：**Differential Walk on Spheres**（2025）让几何/边界可微。

## Technologies

- **无网格几何处理**（Monte Carlo Geometry Processing）：在扫描/CAD/隐式曲面上直接解 PDE，不生成体网格；
- **交互式 PDE 求解**（骨架化之后）：改边界值 = 换叶子、改几何 = 局部修复；
- **可微 PDE**：几何优化、反向设计（形状优化、材质反演）；
- **与渲染的合流点**：WoS 的"点求值 + 渐进"与实时渲染的"渐进式采样"是同一种计算模型（随时可停、随时可加）。

## Game Applications

**直接用的场景很少，间接的判据很多。**

- **烘焙的替代路线**：光照烘焙（Lightmass / DDGI 的离线预计算）本质上是"在几何上解一个椭圆型方程"。**WoS 给出"不建体网格也能解"的一条路**——对那些"体素化很贵/很脆"的资产（薄壁、镂空、雕刻细节）有实际意义。
- **声学 / 热传导 / 扩散**在玩法上的应用（声音传播、温度场、烟雾扩散的稳态近似）——**这些在游戏里通常是"每帧重算一个粗糙网格"，而 WoS 可以只在你关心的几个点上求值**。
- **最重要的其实是那条判据**：**"我要的是全局解，还是几个点上的解？"** 如果是后者，那么"建网格 + 解全局"里绝大部分计算是浪费的。**这个问题在做任何"预计算"之前都该问一次。**
- **不适用**：需要每帧全场更新的场景（风场、流体），因为 WoS 没有摊销，全场求值等价于全场重算。

## Personal Knowledge

`Normal`。**核心等式一个**：$u(x)=\mathbb E[g(X_\tau)]$。**核心事实一个**：从球心出发的布朗运动**均匀**穿出球面——这就是"跳一步"能这么便宜的全部原因。

**一个容易混淆的点**：WoS 不是"离散化 PDE 再随机求解"，它**根本没有离散化 PDE**——它离散化的是"解的概率表示"。**所以它没有离散化误差（只有统计误差），也没有网格质量这个概念。** 这是它和 FEM/FVM 最根本的区别。

**另一个容易记错的点**：它的收敛率 $O(N^{-1/2})$ **不是缺点**，是 MC 的通用性质。**真正的缺点是"每点独立 ⇒ 没有摊销"**——这正是 2023–2026 那一串缓存工作的动机，也是 [[2026-10-30-Liao — From Walks to Skeletons A Reusable Transport Architecture|Walks to Skeletons]] 的全部意义。

## Learning Gap

**已具备**：渲染方程的路径积分直觉（随机游走 + 终止事件上的贡献）、MC 的 $N^{-1/2}$、重要性采样。

**缺口（三处）**：

1. **Poisson 核的偏移形式** $K_B(p,z)$ —— 圆心处的均匀性我熟，但"偏离圆心时怎么变"没有直觉。**这是读懂挂接重加权的必经之路**；
2. **Doob $h$-变换** —— "条件在某个终点上的布朗桥"这个构造。名字听过，推导没做过；
3. **首出时的期望步数与几何的关系** —— 为什么"薄壁/小特征"会让方差爆炸，需要算一次才信。

## Next Step

1. **手算一次**：在单位球（$\Omega=B(0,1)$）里，取 $g$ 为常数 1 与 $g=\cos\theta$ 两种边界数据，验证 $u(0)$ 的估计；然后**把起点从球心挪到 $0.5$ 处**，看估计怎么偏——这一步会直观暴露"均匀性只在球心成立"；
2. **读 [[2026-10-30-Liao — From Walks to Skeletons A Reusable Transport Architecture|Walks to Skeletons]] 的 §3.1**（两页），把 $\ell=K_B(p,z)/r_s(z)$ 那个约分推一遍；
3. **可选**：用 Python 在二维多边形域上实现一个 30 行的 WoS（最近边距离 + 圆上采点），跑出 $O(N^{-1/2})$ 曲线。**这是最低成本的"信"**。

## Notes

- **与渲染的关系要写准**：WoS 与路径追踪**结构同构但不等价**——路径追踪的"步"由 BSDF 决定方向（重要性采样），WoS 的"步"由几何决定（最大内切球）。**前者是"跟着材质走"，后者是"跟着边界走"**；
- **一个常见误解**：WoS 只能解 Laplace。**不对**——加源项就是 Poisson（球内多采几个点算 Green 函数加权），变系数、Neumann、时间依赖都有对应扩展，代价是每一步更贵；
- **这条线在 2026 年的状态**：从"能不能解"转向"能不能交互"。**判据已经从"精度"变成了"改一次输入要多久"**——这正是今天那篇论文的位置。
