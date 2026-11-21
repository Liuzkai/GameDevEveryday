---
type: paper
title: "Fast Contact Force Computation for Nonpenetrating Rigid Bodies"
authors: ["David Baraff"]
year: 1994
published: "1994-07"
venue: "ACM SIGGRAPH 1994（Computer Graphics Proceedings, Annual Conference Series），pp. 23–34"
url: "https://www.cs.cmu.edu/~baraff/papers/sig94.pdf"
code: "论文未给（CMU 内部实现；作者自陈「非数值专家也能写出来」）"
project_page: ""
category: ["Physics", "Contact Dynamics", "Rigid Body Dynamics", "Numerical Methods", "Historical"]
importance: A
historical_importance: 5
game_relevance: 4
production_readiness: "Industry Adopted（枢轴式 LCP 求解是现代引擎接触求解的直系祖先；今天引擎多用 PGS，但它解的是同一个 LCP）"
user_level: Normal
status: unread
---

# Fast Contact Force Computation for Nonpenetrating Rigid Bodies

## TL;DR

**不要把它变成优化问题。**

1994 年的默认做法是：接触力满足一个 LCP，而 LCP 可以写成 QP，于是调用现成的二次规划软件包。Baraff 的反对意见只有一句，但极硬：

> 把**很具体**的问题（求接触力）翻译成**更一般**的问题（数值优化），**翻译过程丢掉了原问题的结构，于是任务变难了。**

他保留的结构是：每个接触点上 $f_i a_i=0$ 这**一条一条**的条件，而不是 $\sum_i f_i a_i=0$ 这**一个**总和。于是算法变成"逐点驱动到零 + 在两组索引之间搬成员"——这就是 **Dantzig 枢轴**：

| 集合 | 含义 | 维持什么 |
|---|---|---|
| $C$（clamped） | 贴住的接触 | $a_i=0$ |
| $NC$（unclamped） | 分离的接触 | $f_i=0$ |

**驱动第 $n$ 个接触**：先假设它不受力，若 $a_n<0$（正在往地里钻），就"加大 $f_n$ 直到 $a_n=0$"，同时按比例调整 $C$ 里所有力以维持它们的 $a_i=0$；一路上用最小步长 $s$ 防止任何 $f_i$ 变负或任何 $a_i$ 变负——**变负就把那个索引在 $C$ 与 $NC$ 之间搬一次（= 枢轴）**。

四个数：
- 无摩擦时，**约为同规模高斯消元求解的 2–3 倍**；
- 比当时最好的 QP 方法**快 5–10 倍**（$n\le150$）；
- 交互式 2.5D 场景在 **SGI R4400 上稳定 20–30 Hz**；
- 静态摩擦：**终止性未证明**（作者明写），而"所有静态摩擦系统都有解"这个猜想**至今未证**。

## Problem

摩擦接触要同时满足三组条件，而它们互相纠缠：

1. 法向：$a_i\ge0$、$f_i\ge0$、$f_i a_i=0$（不穿透 + 只推不拉 + 贴上才有力）；
2. 切向静摩擦：$|f_{F_i}|\le\mu f_{N_i}$、$a_{F_i}f_{F_i}\le0$、$a_{F_i}(\mu f_{N_i}-|f_{F_i}|)=0$；
3. 切向动摩擦：$|f_{F_i}|=\mu f_{N_i}$，方向精确反向于相对滑动速度。

把 1 写成矩阵就是 $a=Af+b$、$a\ge0$、$f\ge0$、$f^\top a=0$——**LCP**；再写成 $\min_f f^\top(Af+b)$ 就是 **QP**。两者等价，但**只有 LCP 记得"每个点各自有一条条件"**。

前人的路是走 QP：Lötstedt 用二次规划，Baraff 自己 1991 年那篇用 Lemke 解 LCP。问题不在正确性，在**可用性**——Baraff 列了四条，全是工程语言：

- 严肃的 LP 实现本来就少，QP 更少；
- 把数值包接进仿真器的接口工作量"有时高到不可接受"；
- 包里有一堆容差/迭代上限**可调参数**，同一问题换一组参数就解不出来；
- 出问题时是黑盒，**没有中间量可看**。

这段话在 2026 年读起来仍然刺人：**它不是反对优化，是反对"因为有个通用求解器就把问题交出去"。**

## Historical Context

```text
1989  Baraff   Analytical methods for dynamic simulation（加速度层 + 事件驱动）
1991  Baraff   Coping with friction —— 摩擦一致性 NP-完全；放弃"只用非冲量力"原则
1993  Baraff   Issues in computing contact forces（Algorithmica）—— 完整版复杂度分析
1994  ★ 本篇    Dantzig 枢轴：不转 QP、保住逐点结构、无摩擦可证终止
1995  Mirtich  impulse-based simulation —— 另一条支线（速度层冲量，游戏引擎直系）
1996  Stewart–Trinkle / 1997 Anitescu–Potra   多面体锥 ⇒ LCP 总有解
2005  Catto    GDC《Iterative Dynamics with Temporal Coherence》—— PGS 进游戏引擎
2010  Anitescu–Tasora  锥互补的 PGS 收敛性证明
2026  Beker et al.      九个转写 + 三笔账（本篇是"精确 + 枢轴"那一格）
```

**本篇在历史上的位置很特殊**：它把"接触力计算"从"调用优化器"变成"**写一个二十行的循环**"。这条路在游戏里没走通（引擎选的是 PGS，不是枢轴），但在**离线/高精度**一侧活了下来（Siconos、Chrono 的 pivoting 分支）。

## Previous Work

- **Lötstedt 1982**：第一个在解析框架里算摩擦力的尝试，用 QP 简化 Coulomb 模型。
- **Baraff 1991**（[[Baraff 1991 — Coping with Friction for Non-penetrating Rigid Body Simulation|库内]]）：动态摩擦用 Lemke；静态摩擦用"类 Gauss–Seidel"迭代，**作者自己承认不可靠**；三维还要近似（⇒ 各向异性）。
- **Cottle–Dantzig**：LCP 的枢轴算法本体。本篇无摩擦的情形**就是** Dantzig 算法（作者明说"不是我们要重新发明轮子"），贡献在于**把它讲成物理**（"这个接触力没有尽到它的本分"）并**扩到摩擦**。

## Core Idea

**把"求解"变成"驱动"，把"模式切换"变成"搬索引"。**

算法的物理直觉只有一句：如果第 $n$ 个接触点在零力下仍然往地里钻，那是**它的力没尽到本分**，加大它；加大它会通过 $A$ 的耦合改变别的点，所以别的点要按比例调整来维持自己的状态；一路加到某个点"撑不住了"（力降到 0，或加速度降到 0），**那一点就换一种状态**——换状态就是枢轴。

于是"组合爆炸"被换成了"**在有限个索引集合之间走**"，而终止性靠一个引理：**在一次 drive-to-zero 内，集合 $C$ 不会重复**（附录 A 定理 3）。有限状态 + 不回头 ⇒ 终止。

## Technical Approach

### ① 无摩擦：可证终止的那个版本

```text
f ← 0, a ← b, C ← NC ← ∅
while ∃d: a_d < 0:
    drive_to_zero(d)

drive_to_zero(d):
    Δf ← fdirection(d)        # 解 A_CC x = −A_Cd，其余 Δf=0，Δf_d=1
    Δa ← A Δf
    (s, j) ← maxstep(f, a, Δf, Δa, d)
    f ← f + sΔf ;  a ← a + sΔa
    if j ∈ C:  C \ {j}, NC ∪ {j};  goto 重算 Δf
    if j ∈ NC: NC \ {j}, C ∪ {j};  goto 重算 Δf
    else:      C ∪ {d};  return        # j = d ⇒ a_d 已到 0
```

三处值得记住的细节：

1. **$A_{11}$ 永不死**。$A$ 对称半正定、且 $b$ 在 $A$ 的列空间里 ⇒ 每个主子矩阵 $A_{CC}$ 都可解。**秩亏不是障碍，是常态**（原文：*A is often extremely rank-deficient*）。
2. **每次枢轴只改一行一列**，所以增量分解（LUSOL）能把 $O(n^2)$ 的更新做到很便宜。作者用普通高斯消元也能跑——**这是"可实现性"的证据**。
3. **bilateral 约束免费**：把它的 $a_i$ 恒设为 0、允许 $f_i$ 为负，就是"永远留在 $C$ 里"。

### ② 静摩擦：那个诚实的缺口

静摩擦要多两组索引：$C_F$（维持 $a_{F_i}=0$）、$NC^\pm$（顶在 $\pm\mu f_{N_i}$ 上）。逻辑上多了一大堆分支，**数值核心没变**（还是解 $A_{11}x=-v_1$）。

但终止性**证不出来**，原因很具体：静态摩擦引入了侧条件 $\Delta f_{F_i}=\pm\mu\Delta f_{N_i}$，它**破坏**了"加大 $f_d$ 必然加大 $a_d$"这条单调性（定理 2 失效）。于是可能出现**步长为 0 的循环**：某个 $i$ 在 $C$ 与 $NC$ 之间来回弹。

作者的补救是工程式的、而且他明说这是"negative progress"：

> 观察到某个变量在两组之间来回弹，就**把它从两组里都摘掉**——暂时放弃那个接触点的条件，先去把当前这个点弄好，**以后再回来补**。

**代价是失去了"已经解好的点保持解好"这个不变量**，所以不能证明终止。原文写得很直：

> We cannot prove that the algorithm we present for computing static friction forces will always terminate; if we could, that in itself would constitute a proof of the conjecture.

**这条缺口 32 年后仍然开着**：2026 年的教程只能说"速度层没有已知的多项式算法"，存在性只能在"严格可行"假设下证（Acary 2011）。

### ③ 三维：不做近似，但解不唯一

前人做三维摩擦要么用多面体锥（⇒ 各向异性），要么对 $\|f_t\|\le\mu f_N$ 这个**非线性**约束做线性化。本篇的做法是：**把"何时顶到锥面"写成一个关于步长 $s$ 的二次方程**

$$(f_x+s\Delta f_x)^2+(f_y+s\Delta f_y)^2=\big(\mu(f_N+s\Delta f_N)\big)^2$$

解它得到 $s$，于是**约束被精确处理，不需要锥近似**。这是本篇相对 1991 的实质进步。

代价写在 §5.6 最后一句：**当切向加速度非零时，Coulomb 律并不完全确定摩擦力的方向**（只要求"至少部分反向"），所以**换一个切平面基、或换个 $x/y$ 的求解顺序，会得到不同的解**。

> 这是"解不唯一"在 1994 年的第一次显式出场。2026 年的教程把它变成了一句方法论：**解不唯一时，求解器是模型的一部分。**

### ④ 动摩擦：无解被检测成"无穷步"

动摩擦把 $f_F$ 换成 $\mp\mu f_N$，于是 $A$ **失去对称与半正定**。后果是：**可能根本不存在有界的力解**，物理上必须上冲量。

妙处在于**不需要额外代码**：这种情况下 `maxstep` 会返回 $s=\infty$（无界射线），而这正好对应"该上冲量了"。**力与冲量被统一进同一个算法**——这就是 1991 年"换答案空间"的算法化落地。

## Key Contribution

1. **把 Dantzig 枢轴讲成物理**（"这个力没尽本分"），使非数值专家能实现。
2. **证明无摩擦情形的终止性**（$C$ 不重复 + $A_{11}$ 可解 + 步长恒正）。
3. **三维静摩擦不做锥近似**（二次方程定步长），代价是方向不确定 ⇒ 解不唯一。
4. **力与冲量统一**：$s=\infty$ 既是"无解"的检测，也是"该上冲量"的判据。
5. **诚实的缺口**：静态摩擦终止性未证，且明确指出"若能证，就等于证明了那个猜想"。

## Why It Works

- **$b\in\mathrm{col}(A)$**：这是"$A_{11}$ 永不死"的全部理由。物理上它说的是"外力/惯性力总可以被某些接触力平衡"，数学上它保证 $A_{11}x=-v_1$ 始终有解。
- **单调性**（定理 2）：$A\succ0$ 时，加大 $f_d$ 一定加大 $a_d$ ⇒ 驱动一定前进。**这条在静摩擦里失效，是全部麻烦的根源。**
- **有限状态 + 不回头**：$C$ 在一次驱动内不重复 ⇒ 最多 $2^n$ 次枢轴，但实测 $O(n)$ 次。

## Limitations

1. **最坏情况指数时间**（枢轴类算法的通病，像单纯形法）。作者明说：*worst-case problems resulting in exponential running times can be constructed*。
2. **静态摩擦无终止保证**，只有工程补丁；而补丁会"退掉"已解好的接触。
3. **解不唯一**：冗余接触下力分布不唯一；三维静摩擦方向不唯一（换基就换解）。
4. **只处理点接触**，面接触要靠多点采样；**不做碰撞/冲量响应**（那部分在 1991 那篇）。
5. **1994 年的性能语境**：$n=150$、SGI R4400、20–30 Hz。**这些数字不能直接搬**，但"2–3 倍高斯消元"这个比值是可迁移的量级判断。
6. **没有 GPU 路径**，且枢轴的串行本质与 GPU 不合——这是它在游戏里输给 PGS 的结构性原因。

## Game Development Relevance

**为什么引擎最终没选枢轴，而选了 PGS？** 本篇其实已经把答案写在自己的优点里：

| 需求 | 枢轴（本篇） | PGS（引擎实际选择） |
|---|---|---|
| 精确解 | ✅ 有限步精确（无摩擦） | ❌ 迭代到收敛 |
| 早停可用 | ❌ 中途状态无意义 | ✅ **任何时刻都返回可行力** |
| warm start | ❌ 每次从头 | ✅ 上一步的解直接当初始值 |
| 单帧成本可预测 | ❌ 最坏指数 | ✅ 固定迭代次数 |
| GPU | ❌ 串行 | ✅ Jacobi/着色并行 |

**游戏要的不是"解对"，是"每帧在预算内给出一个不会炸的力"。** 这条判据解释了 1995 年之后整个行业的走向。

但那句"**不要把它交给通用优化器**"仍然是活的：今天引擎里每个求解器都是**手写的、问题特定的**，没有一个调用通用 QP 包。**本篇赢的是这个论点，不是那个算法。**

## Unreal Engine Relevance

- Chaos 的接触求解是 PGS/TGS（迭代、可 warm start、可早停），**不是枢轴**——但**它解的 LCP 与本篇是同一个 LCP**，所以"$C$/$NC$ 的物理直觉"仍然可以用来读求解器的中间状态。
- **"静态摩擦无终止保证"在引擎里的对应物是"迭代上限"**：PGS 也不是解到精确，而是固定 8–16 次迭代停。**两者都在用一个非精确解冒充精确解，只是失败方式不同**（枢轴可能循环，PGS 只会没收敛）。
- 本篇的 **bilateral 约束免费**这一条，正是引擎里"关节约束与接触约束写在同一个系统里"的原始论证。

## Technology Evolution

```text
1989  加速度层 + 事件驱动（Baraff）
  ↓
1991  NP-完全 ⇒ 换答案空间（力或冲量）
  ↓
1994  ★ 枢轴：保住逐点结构，力/冲量统一，静摩擦留缺口
  ↓
1995  Mirtich 冲量法（速度层）—— 与枢轴平行的另一支
  ↓
1996-97 多面体锥 + LCP 存在性证明（Stewart–Trinkle / Anitescu–Potra）
  ↓
2005  Catto 的序列冲量（= PGS）进游戏引擎
  ↓
2010  锥互补 PGS 的收敛性证明（Anitescu–Tasora）
  ↓
2026  九个转写的统一记账（本篇 = 图 5 里 "exact + 枢轴" 那一格）
```

**分岔点在 1995**：一条追求"精确 + 有限步"（枢轴、内点），一条追求"可行 + 可早停 + 可并行"（PGS）。**游戏选了后者，离线仿真选了前者。** 这不是谁对谁错，是**验收标准不同**：一个是"每帧预算内不出错"，一个是"这一步解对"。

## Relationships

### Based On

- **Cottle–Dantzig 的 LCP 枢轴算法**：无摩擦情形就是它，本篇的贡献是物理讲法与摩擦扩展。
- **[[Baraff 1991 — Coping with Friction for Non-penetrating Rigid Body Simulation|Baraff 1991]]**：本篇的"力或冲量"统一，直接继承 1991 年"放弃 principle of constraints"的结论。

### Extends

- **Baraff 1991**：把"期望多项式时间"的 Lemke 换成"实测 $O(n)$ 次枢轴"的专用算法；三维摩擦从近似变成精确。

### Improves

- **Lötstedt 1982 / Baraff 1991 的静态摩擦**：那两版的静态摩擦"不可靠"，本篇给了一个可用的（虽然终止性未证）。

### Related

- **[[2026-11-21-Beker — A Tutorial on Complementarity-Based Contact Models for Robotics|Beker 2026]]**：把本篇所在的"精确 + 枢轴"一格，放回九个转写的全景里。
- **[[Contact Dynamics]]**：概念页。

## Personal Knowledge State

- **Easy**：$f_ia_i=0$ 的物理含义、为什么"贴上才有力"。
- **Normal（今天补的）**：
  1. **索引集合的物理读法**（$C$ = 贴住、$NC$ = 分离），以及"枢轴 = 换状态"；
  2. **"不要交给通用优化器"的论证**——丢的是"逐点结构"；
  3. **$s=\infty$ 是"该上冲量"的检测**，力与冲量在同一个算法里；
  4. **静摩擦的缺口**是单调性失效，不是实现 bug。
- **仍是 Hard**：附录 A 的定理 2–5 证明（主子矩阵正定 + $C$ 不重复的技术细节）。

## Learning Value

**对写求解器的人**：这是一份"如何把问题结构用在算法里"的范本。三个动作可以直接迁移——**把约束分组（$C$/$NC$）、把切换写成枢轴、把无解检测成一个极值（$s=\infty$）**。

**对用引擎的人**：读它最大的收益是**知道"精确解"长什么样**。当你在引擎里看到穿透、抖动、脚滑，你能判断这是"没收敛"还是"物理定律被改了"——**这两种故障的修法完全不同**。

## Visualization

本篇的两张图是 1994 年的时序仿真截图（blockfeeder / double-action jack），承载的信息是"能跑"，不是"怎么算"，因此不提取。**核心机制用下面这张状态机读更快：**

```text
              a_i < 0（往地里钻）
                     ↓
   ┌─────────── drive_to_zero(i) ───────────┐
   │  解 A_CC x = −A_Cd  ⇒  Δf, Δa          │
   │  取最小步长 s（三条上界）：             │
   │    f_j 要变负 → 枢轴 j: C → NC          │
   │    a_j 要变负 → 枢轴 j: NC → C          │
   │    a_i 到 0   → 完成，i 进 C            │
   └─────────────────────────────────────────┘
        无界（s = ∞）⇒ 不存在有界力 ⇒ 上冲量
```

## Notes

- 原文 PDF 已归档：`Research/Files/PDF/Baraff 1994 - Fast Contact Force Computation for Nonpenetrating Rigid Bodies.pdf`（12 页，CMU 电子重印，一次 200）。
- **一个容易读错的点**：本篇标题里的 "nonpenetrating" 不是"不穿透的物体"，是"**不许穿透**"这个约束；它处理的是**resting contact**（静置接触），碰撞响应留给 1991 那篇与后续工作。
- **数字口径**：2–3 倍高斯消元是"同一个 $n$"下的比值；5–10 倍对 QP 是"$n\le150$"下的实测。**两者不能混着引用。**
- 与 2026 教程的关系不是"被取代"：教程把它列为 5 个精确转写之外的**算法**（Table 2 最后一行 "Pivoting (App. A) → LCP → 每枢轴一次秩一更新 → 有限步 → 直接法 → 顺序 → 不适合 GPU"）。
