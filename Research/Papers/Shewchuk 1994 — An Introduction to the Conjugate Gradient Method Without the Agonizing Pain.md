---
type: paper
title: "An Introduction to the Conjugate Gradient Method Without the Agonizing Pain"
authors: ["Jonathan Richard Shewchuk"]
year: 1994
published: "1994-08-04"
venue: "Carnegie Mellon University 技术报告（Edition 1¼）"
url: "https://www.cs.cmu.edu/~quake-papers/painless-conjugate-gradient.pdf"
code: ""
project_page: ""
category: ["Numerical Methods", "Linear Solvers", "Teaching", "Physics Simulation"]
importance: A
historical_importance: 4
game_relevance: 4
production_readiness: "Production Ready（不是方法，是教材：三十年来的标准入门材料；文中所有算法可直接实现）"
user_level: Normal
status: unread
aliases: [Shewchuk 1994, CG without the pain, 共轭梯度无痛入门, 共轭梯度, conjugate gradient, PCG]
tags: [solver, conjugate-gradient, preconditioning, linear-algebra, physics, teaching]
---

# Shewchuk 1994 — 为什么共轭梯度比最速下降快，而且快在一个你没料到的地方

## TL;DR

**最速下降每一步都"重新开始"；共轭梯度每一步都"不撤销前面做过的"。**

两个数说明差距：

| | 收敛因子 | 把误差压到 $\epsilon$ 要多少次迭代 | 时间复杂度 |
|---|---|---|---|
| 最速下降（SD） | $\dfrac{\kappa-1}{\kappa+1}$ | $\tfrac12\kappa\ln(1/\epsilon)$ | $O(m\kappa)$ |
| **共轭梯度（CG）** | $\dfrac{\sqrt\kappa-1}{\sqrt\kappa+1}$ | $\tfrac12\sqrt\kappa\ln(2/\epsilon)$ | $O(m\sqrt\kappa)$ |

$\kappa$ 是条件数。**代价从 $\kappa$ 变成 $\sqrt\kappa$——不是"每次更快"，是"次数变少"。**

而二阶椭圆型问题的离散化给出 $\kappa\approx O(h^{-2})$，在二维里 $h^{-2}\approx n$——于是：

$$
\text{SD} = O(n^2),\qquad \text{CG} = O(n^{1.5}),\qquad \text{多重网格} = O(n)
$$

**这条阶梯是理解"为什么物理引擎里到处是 PCG、而离线求解器用多重网格"的最短路径。** 本文负责中间那一级。

## Problem

解稀疏 SPD 系统 $Ax=b$。三种思路：

- **直接法**（Cholesky / LU）：$O(n^3)$ 或带 fill-in 的稀疏分解，$n$ 一大就爆内存。
- **定常迭代**（Jacobi / Gauss-Seidel / SOR）：每次很便宜，但收敛因子由谱半径决定，**与 $n$ 一起退化**。
- **Krylov 子空间法**（CG）：每次仍然只有一个矩阵-向量乘，但**收敛次数有上界，且上界只跟 $\sqrt\kappa$ 有关**。

问题在于：**为什么"多记一个方向"就能把 $\kappa$ 变成 $\sqrt\kappa$？** 大多数教材不给直觉，只给代数。这篇的全部价值就是给直觉——**66 幅插图，几乎不用连续散文**。

## Historical Context

- **1952 Hestenes & Stiefel**：CG 原始论文（一篇是 NIST 报告，一篇是 J. Research NBS）。当时被认为是"直接法"（$n$ 步精确收敛），因为**没有预条件时它数值上不稳定**，被当成奇技淫巧冷藏了二十年。
- **1970s Reid / Concus-Golub-O'Leary**：认识到"**CG 作为迭代法 + 好的预条件**"才是正确用法，CG 复活。
- **1994 Shewchuk**：把这套直觉写成一本可读的小册子。**它没有提出新方法，它生产的是"能自己推一遍"的人。**
- **2010s–**：GPU 上的 PCG 成了弹性/布料/流体 IPC 的标准内层求解器（[[Contact Dynamics]] 里那张"每步解一次"的表，内层就是它）。

> **这条线值得注意的曲率**：CG 从"直接法"变成"迭代法"，关键不是数学变了，是**判据变了**——从"多少步精确收敛"变成"多少步把残差压到够用"。**这是本库反复出现的同一个动作：换一个判据，同一个算法就有了新生命。**

## Previous Work

本篇的"前作"是**教材**而不是论文：作者的原话是，这个话题的多数教科书"既没有插图也没有直觉，其受害者至今仍能在落满灰尘的图书馆角落里含混自语"。所以它的对手是**表述方式**，不是算法。

在这个意义上它更接近 [[Baraff 1994 — Fast Contact Force Computation for Nonpenetrating Rigid Bodies|Baraff 1994]] 那类"把一件事讲到你能自己推"的文献，而不是"提出新方法"的文献。

## Core Idea

**把解 $Ax=b$ 变成"在一个二次型碗里找最低点"。"**

$$
f(x)=\tfrac12 x^\top A x - b^\top x,\qquad \nabla f = Ax-b,\qquad \nabla f=0 \iff Ax=b
$$

$A$ 对称正定 ⇒ $f$ 是一个**碗**（抛物面），等高线是椭圆。

- **最速下降**：站在碗上，沿最陡的方向（$-\nabla f$）走一步，走到该方向的最低点，再重新看最陡方向。**问题：下一个梯度一定与上一个正交**，于是路径**在两条线之间来回锯**（图 8）。在细长的碗里，每一步只消掉误差的一小部分，剩下的大部分被下一步"还回去"。
- **共轭方向**：如果选一组**$A$-正交**（"共轭"）的搜索方向 $d_i^\top A d_j=0$，那么**沿 $d_i$ 走到的那个最低点，沿 $d_j$ 走时不会被破坏**。$n$ 个方向走完，精确到解。
- **共轭梯度**：不需要预先知道这组方向——**用上一步的残差构造下一个方向**，自动保证 $A$-正交。这就是"免费"的部分：只需要多存一个向量。

> **一句话：SD 每一步都在同一个二维平面里重复劳动，CG 每一步都走进一个从未走过的新维度。**
> $\kappa$ 衡量"碗有多扁"；SD 的收敛速度随扁度线性恶化，CG 只随**扁度的平方根**恶化——因为 CG 在第 $i$ 步已经把所有前 $i$ 个特征方向"处理过了"，剩下来的只有第 $i+1$ 到第 $n$ 个。

## Technical Approach（三个算法，一条主线）

### ① 最速下降

$$
\alpha_i=\frac{r_i^\top r_i}{r_i^\top A r_i},\qquad x_{i+1}=x_i+\alpha_i r_i
$$

步长是"沿这条线走到最低点"的解析解——**所以每一步都是局部最优，而整体收敛很差**。这就是锯齿的来源。

### ② 共轭方向（Gram-Schmidt 版）

给定 $n$ 个线性无关的 $u_i$，用 Gram-Schmidt 造出 $A$-正交的 $d_i$，然后逐个方向精确最小化。**理论上 $n$ 步收敛，实践上要存 $n$ 个向量**——不可用，但它是理解 CG 的桥。

### ③ 共轭梯度（可用的版本）

不存 $n$ 个方向，只存**当前方向、当前残差、上一个残差**：

$$
d_{i+1}=r_{i+1}+\beta_{i+1}d_i,\qquad \beta_{i+1}=\frac{r_{i+1}^\top r_{i+1}}{r_i^\top r_i}
$$

**新方向 = 新残差 + 旧方向的一个分量。** 这个 $\beta$ 恰好让 $d_{i+1}$ 与 $d_i$ 共轭——**这就是全部魔法**。空间 $O(m)$，每次迭代一次矩阵-向量乘。

### ④ 预条件

把 $Ax=b$ 换成 $M^{-1}Ax=M^{-1}b$，$M$ 是"接近 $A$ 但好求逆"的 SPD 矩阵。几何上：**把那个扁长的碗掰成一个更接近球的碗**（图 36），于是 $\kappa$ 变小。

- 对角预条件（Jacobi）：$M=\mathrm{diag}(A)$，最便宜。
- **块 Jacobi**：$M$ 取块对角 ⇒ 块内联合修正强耦合未知量。**这正是 [[2026-11-24-Lyu — ParaBMG A Parallel Block Multigrid Framework for Large-Scale Evolving Sparse Systems on Unstructured Meshes|ParaBMG 2026]] 里那个"块平滑"的位置**——它在本篇是预条件器的一个特例。
- **多重网格**：$M^{-1}$ 用一次 V-cycle 代替 ⇒ 这是 [[Multigrid]] 与 CG 的交汇点，也是"$O(n)$"那一级的入口。

## Key Contribution

**不是新算法，是三个直觉：**

1. **收敛率由"碗的形状"决定**，而形状就是 $\kappa$——这个量可以算、可以改（预条件），所以**"解得多快"在设计阶段就可预测**。
2. **CG 的 $n$ 步精确收敛是"不同特征值个数"的上界**（原文：精确算术下所需迭代数**至多等于不同特征值的个数**）——所以有重复特征值的问题反而更快。**这解释了为什么"$n$ 步"在实践中从来用不完。**
3. **误差要按 $A$-范数衡量**（$\|e\|_A^2 = e^\top A e$），不是欧氏范数。**用错范数会让你以为算法在收敛而其实没有。**

## Why It Works

**因为"共轭"这个约束，把每一步的搜索子空间从 1 维扩大成了 $i$ 维。**

第 $i$ 步的 CG 解，等价于在**Krylov 子空间** $\mathcal{K}_i(A,r_0)=\mathrm{span}\{r_0,Ar_0,\dots,A^{i-1}r_0\}$ 上做最优（在 $A$-范数下）。而 SD 只在 $\mathrm{span}\{r_{i-1}\}$ 上最优。

> **这就是"分工"：SD 把全部力气花在一个方向上（然后被下一个方向破坏），CG 把力气分配到一个逐步增长的子空间上（且互不破坏）。**

而 Krylov 子空间之所以有效，是因为 $A$ 的特征值分布：**如果特征值聚成 $k$ 簇，CG 大约 $k$ 步就收敛**——因为多项式 $p(A)$ 只要在这 $k$ 个簇上为零即可。**预条件的作用就是把特征值往几个簇里赶。**

## Limitations（作者自己列的，比多数论文都诚实）

1. **$O(\sqrt\kappa)$ 的界很松**。真实收敛常常远快于界（特征值成簇时）。**所以"按界估迭代数"会高估。**
2. **浮点会破坏共轭性**。有限精度下残差逐渐失去正交性，$n$ 步收敛不成立；实践上靠"残差足够小就停"来兜底。
3. **只对 SPD**。非对称 / 不定要用 GMRES / BiCGStab / MINRES，那套理论完全不同。
4. **预条件是全部难点**。本文给的是原理，具体 $M$ 的选择（ILU / 块 / 多重网格 / AMG）才是工程战场。
5. **非线性 CG 是另一回事**（Fletcher-Reeves / Polak-Ribière），本文只作引子；它没有"共轭"保证，靠周期性重启兜底。

## Game Development Relevance

**这是"物理引擎最里面那层"的入门券，而且它的结论直接是工程判据：**

1. **判断"要不要上更好的求解器"**：如果你的 $\kappa$ 只由对角缩放决定（比如每个节点质量差几个数量级），**Jacobi 预条件几乎免费且有效**；如果 $\kappa$ 来自网格尺寸（$\propto h^{-2}$），**只有多重网格能救**——因为那是"全局低频"，任何局部预条件都无能为力。
2. **"迭代次数"是可以预算的**：$\tfrac12\sqrt\kappa\ln(2/\epsilon)$——把 $\kappa$ 估出来，迭代数就定了，帧预算就能算。**这是本库"预算"语言里少有的"可以先算再跑"的一项。**
3. **它解释了为什么布料/软体每帧都在喊"迭代不够"**：固定迭代预算 = 固定 $\epsilon$，而 $\kappa$ 随网格加密变差。**加密网格不加迭代，等于悄悄降低精度。**
4. **它也解释了为什么"Jacobi 迭代 20 次"在简单场景够用、在复杂场景崩**：$\kappa$ 变了，界没变，但界是上界。

## Unreal Engine Relevance

- **Chaos 的 PBD/XPBD 不是 CG**（它是约束投影，不是 Krylov 法），但**"迭代次数 vs 收敛"的账是同一本**。
- **Chaos Flesh / 有限元类**：内层是 PCG，预条件器选择直接决定能不能实时。
- **可迁移的心态**：**"我调了 20 次迭代"和"我把残差压到 $10^{-4}$"是两句完全不同的话**——前者在说预算，后者在说精度。**引擎里经常混用。**

## Technology Evolution

```
1952 Hestenes & Stiefel    CG（被当直接法，冷藏）
  ↓
1970s                      + 预条件 ⇒ 变成实用迭代法
  ↓
1980s–1990s                与多重网格合流（MG 作预条件器）
  ↓
1994 Shewchuk              直觉定型（本篇）
  ↓
2000s–2010s                GPU 上的 PCG：稀疏矩阵-向量乘成为主角
  ↓
2020s–2026                 MAS / 块 Jacobi（弹性 SOTA）→ ParaBMG（几何处理）
```

## Relationships

### Based On

- [[Multigrid]] —— 反向依赖：多重网格在这里是**预条件器**的一种；在那页是**主体**。两页互为对方的"另一个视角"。
- [[GPU Architecture]] —— $O(m\sqrt\kappa)$ 里的 $m$（非零元）决定了每次迭代要搬多少字节，而**在 GPU 上这个搬运就是下限**。

### Extends

- [[Profiling and Measurement]] —— 本篇给出"**可以先算的预算**"这一类（$\kappa\to$ 迭代数），与那页"只能事后测"的保真度形成对照。
- [[Contact Dynamics]] —— 那边解的是 LCP（非凸、可无解），这边解的是 SPD（凸、唯一）。**同一个引擎里两层用两套数学。**

### Related

- [[2026-11-24-Lyu — ParaBMG A Parallel Block Multigrid Framework for Large-Scale Evolving Sparse Systems on Unstructured Meshes|ParaBMG 2026]] —— 它测的 "PCG 迭代数 × 每次迭代成本" 就是本篇的两个因子。
- [[Fluid Simulation]]、[[Cloth Simulation]] —— 消费者。
- [[Rendering Equation]] —— 结构上的远亲：都是"把一个全局问题离散成线性系统然后解"，只是一个解的是辐射、一个解的是位移。

### Contrasts

- 直接法（Cholesky / cuDSS）：**买确定性，付 $O(n^{1.5})$ 内存与 fill-in**。CG 买的是"内存只要 $O(m)$"，付的是"收敛依赖 $\kappa$"。**这笔交易在游戏里几乎总是划算的**——因为内存是硬的，时间是软的（可以少迭代几次）。

## Personal Knowledge State

- **Easy**：$Ax=b$ 与二次型最小化的等价；梯度下降的几何图像；$\kappa$ 是什么。
- **Normal**：为什么 SD 会锯齿（连续梯度正交）、CG 的 $\beta$ 在做什么、预条件 = 改变碗的形状。
- **Hard**：Krylov 子空间与多项式逼近的等价（$p(A)r_0$）、$\sqrt\kappa$ 界的推导、为什么多重网格能把 $\kappa$ 变成与 $n$ 无关。

## Learning Value

**这是本库最"可执行"的一篇**：60 页，66 幅图，读完能自己写一个 30 行的 PCG，并且**能估出它要迭代多少次**。

三个具体动作：
1. **手算一次 $\kappa$**：拿一个一维 Laplace 的三对角矩阵（$n=100$），算出 $\kappa$，代入两个公式，看 SD 和 CG 各要多少次。**这个手算比读十页推导有用。**
2. **看锯齿**：图 8 那张图能让你以后再也不想用"梯度下降"解线性系统。
3. **把"迭代次数"翻译成"残差"**：在你的引擎里找到那个 `MaxIterations=20`，问问它对应的 $\epsilon$ 是多少。

## Visualization

![[Research/Files/Pictures/Shewchuk 1994 — 图8 最速下降的锯齿路径.png]]

**读图**：等高线是一个扁长的椭圆（$A$ 的条件数大）。从 $x_{(0)}$ 出发，每一步都沿当前梯度走到该方向的最低点——**于是路径在两族方向之间来回锯，越锯越密，收敛极慢**。锯齿的成因写在原文一句话里：*"each gradient is orthogonal to the previous gradient"*。**这张图是"局部最优 ≠ 全局最优"最干净的物证。**（图 8，Shewchuk 1994）

![[Research/Files/Pictures/Shewchuk 1994 — 图35 一个CG迭代抵多少次最速下降.png]]

**读图**：横轴 $\kappa$（0–1000），纵轴"多少个 SD 迭代才抵得上一个 CG 迭代"。$\kappa=100$ 时约 10 次，$\kappa=1000$ 时约 31 次——**大致就是 $\sqrt\kappa/2$**。**注意这个数是"逐次"的**：CG 不是每步比 SD 强一点，是**每一步相当于 SD 的一整段**。这就是 $O(\kappa)$ 与 $O(\sqrt\kappa)$ 的肉眼形态。（图 35，Shewchuk 1994）

## Notes

原文 PDF（CMU 作者页，开放）已核对：64 页 PDF，Edition 1¼，1994-08-04。**未归档进 `Files/PDF/`——它是公开教学材料，且链接稳定；如需离线副本再说。**

- **术语坑**：本文的 "conjugate" 指 **$A$-正交**，与复共轭无关。中文"共轭"两个字在这里是历史译名，容易误导。
- **符号坑**：本文用 $\kappa$ 表示条件数；[[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting|NIRJALA 2026]] 也用 $\kappa$ 表示"反演增益地板"。**同一个字母，两个毫不相干的量。**
- **与 [[Multigrid]] 的分工**：本篇是"**在同一条网格上把方向选对**"，多重网格是"**在多条网格上把频率分对**"。两者正交，可以叠加（MG 作 CG 的预条件器就是叠加）。

**相关每日记录**：[[2026-11-24]]
