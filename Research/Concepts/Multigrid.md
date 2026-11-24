---
type: concept
title: "Multigrid"
user_level: Normal
aliases: [Multigrid, MG, 多重网格, 多层网格, 几何多重网格, GMG, 代数多重网格, AMG, V-cycle, 粗网格校正, coarse-grid correction, 平滑器, smoother]
prerequisites: ["[[Shewchuk 1994 — An Introduction to the Conjugate Gradient Method Without the Agonizing Pain]]", "[[Mesh Parameterization]]"]
first_introduced: "1962（Fedorenko，俄文，长期未被引用）；1977（Brandt，变成可实践的方法）"
---

# Multigrid

> **一个平滑器只能在一次扫描里把信息传一个格子。**
> 所以误差里"波长很长"的那部分，在细网格上**永远等不到**——不是精度不够，是**传播速度**不够。
> 多重网格的全部内容就是一句话：**换到粗网格上，同一段误差就变短了。**

## Definition

多重网格 = **用一族由细到粗的网格，把误差按"频率"分段，每段交给最擅长处理它的那一层**。

- **细层**：平滑器（Jacobi / Gauss-Seidel / 块 Jacobi / Chebyshev）快速消掉**高频**误差。
- **粗层**：把残差限制下去，在那里同一段**低频**误差变成了相对高频，**同一把刀又能用**。
- **回来**：粗层的解插值上来，修正细层的解。

一个 V-cycle：`pre-smooth → restrict → 递归 → prolongate → post-smooth`。

**它不是"更快的迭代法"，是"把误差的频谱分工"的框架。** 任何迭代法都能塞进它的某一层。

## Core Principle

### ① 为什么平滑器杀不掉低频（这是全部动机）

一次 Jacobi/Gauss-Seidel 扫描，每个点只看它的直接邻居。**信息一次只走一个格子。**

设误差是一个波长 $\lambda$ 的正弦波（以格子数计）。要把它"感知"到，至少需要信息跨越 $\lambda/2$ 个格子 ⇒ 约 $\lambda/2$ 次扫描。**而低频误差的 $\lambda \approx n$。**

于是：

| 误差频段 | 波长 | 平滑器要几次扫描 | 结果 |
|---|---|---|---|
| 高频 | 2–4 格 | 2–5 次 | **几乎免费** |
| 中频 | $\sqrt n$ 格 | $\sqrt n$ 次 | 开始贵 |
| **低频** | $n$ 格 | **$n$ 次** | **在细网格上不可能** |

**这就是"用一把只能切薄片的刀去砍一整块"的定量版本。** 而 Krylov 方法（[[Shewchuk 1994 — An Introduction to the Conjugate Gradient Method Without the Agonizing Pain|CG]]）绕开的办法是"记住方向"，代价是 $\sqrt\kappa$；多重网格绕开的办法是"换一副尺子"，代价是**建层级**。

### ② 为什么换到粗网格就有效

粗网格的格子间距是 $2h$。同一段物理波长 $\lambda$，在粗网格上变成 $\lambda/2$ 个格子——**相对波长减半**。限制-求解-插值一次，等于在粗层上做了一次"相对高频"的消除。

递归下去：$\lambda \to \lambda/2 \to \lambda/4 \to \dots$，直到最粗层只有几个未知数，直接解掉。

> **判据：多重网格不是"多分辨率近似"，是"精确的频谱分解"。** 每一层处理的不是"更粗糙的画面"，是"上一层的平滑器**明确没处理**的那一段误差"。**这两件事经常被混为一谈**（LOD / mipmap 是前者）。

### ③ 为什么总工作量是 $O(n)$

几何级数：$n + n/2 + n/4 + \dots \approx 2n$。一个 V-cycle 的成本 ≈ **两次细网格扫描**。

而**收敛因子与 $n$ 无关**（这是多重网格的定义性质）：无论网格多细，一个 V-cycle 都把残差压一个固定倍数（典型 0.1–0.3）。

$$
\underbrace{O(n\sqrt\kappa)}_{\text{CG}},\ \kappa\propto h^{-2}\ \Rightarrow\ O(n^{1.5})
\qquad\text{vs}\qquad
\underbrace{O(n)}_{\text{MG}}
$$

**三级阶梯**（二维，$m\approx n$，$\kappa\approx n$）：

| 方法 | 迭代次数 | 每次成本 | 总计 | 凭什么 |
|---|---|---|---|---|
| 最速下降 | $O(\kappa)=O(n)$ | $O(n)$ | $O(n^2)$ | 只沿当前梯度 |
| 共轭梯度 | $O(\sqrt\kappa)=O(\sqrt n)$ | $O(n)$ | $O(n^{1.5})$ | 记住方向，Krylov 子空间 |
| **多重网格** | $O(1)$ | $O(n)$ | $\mathbf{O(n)}$ | **换尺子，按频率分工** |

> **这是数值方法里最漂亮的一个台阶：不是把常数做小，是把指数降下来。**

### ④ 三个变体，区别只在"粗层怎么来"

| 变体 | 粗层怎么构造 | 优点 | 欠下的 |
|---|---|---|---|
| **GMG**（几何多重网格） | 用网格本身：粗化顶点、重连边 | 粗化便宜、插值有几何意义 | 需要网格结构；拓扑变化时要重建 |
| **AMG**（代数多重网格） | 只看矩阵：强连接聚合成粗未知量 | 不需要几何，任意稀疏矩阵可用 | 不利用几何；系数一变 coarsening 可能失效；setup 贵 |
| **MAS**（多重网格加法 Schwarz） | 非重叠块划分 + 就地块求逆 | **每次迭代极便宜**、可全并行 | 加法界松 ⇒ Poisson 型问题上迭代数爆炸 |

### ⑤ 乘法 vs 加法：一句话的分别

- **乘法（MG）**：校正**逐层作用在当前残差上**。上一层没削干净的，下一层继续削。序列相关 ⇒ 界更锐。
- **加法（MAS）**：各层校正**独立算完再相加**。全并行 ⇒ 每次更便宜，但"上一层没削干净的"不会被下一层接着削。

$$
\text{乘法：} e \leftarrow (I - P A_c^{-1} P^\top A)(I - M^{-1}A)\,e
\qquad
\text{加法：} e \leftarrow \Big(I - \sum_l P_l A_l^{-1} P_l^\top A\Big)e
$$

**这不是"哪个更好"，是"耦合能传多远"**：见下。

### ⑥ 决定胜负的那个物理量：耦合长度（screening length）

弹性隐式时间步解

$$
\left(K + \alpha M\right)x = b,\qquad \alpha = \Delta t^{-2}
$$

质量项 $M$ 充当**屏蔽项**——**这正是 screened Poisson 方程**（$\alpha$ 就是屏蔽系数的平方）。

- $\alpha$ 大（短时间步、几乎纯惯性）：远处的影响被屏蔽 ⇒ **耦合短程** ⇒ 局部校正够用 ⇒ **加法赢**（MAS 的便宜占上风）。
- $\alpha\to0$（静态极限）：耦合贯穿整个网格 ⇒ **低频主导** ⇒ **乘法赢**（且差距随 $n$ 放大）。

> **判据：先量"耦合能传多远"，再选求解器。**
> 同一个几何、同一个右端项，只改时间步长，最优解就会换人。**所以"仿真的求解器搬到几何处理"从原理上就不成立。**

### ⑦ 成本结构：演化系统里，setup 是主角

$$
T_{\text{总}} = \underbrace{T_{\text{setup}}}_{\text{建层级}} + N_{\text{解}} \times \Big( \underbrace{T_{\text{update}}}_{\text{更新系数}} + n_{\text{iter}} \times T_{\text{iter}} \Big)
$$

**大部分"多重网格论文"只优化最后一项。** 而在"系数每步都变、要解四万次"的场景里，前三项才是大头——[[2026-11-24-Lyu — ParaBMG A Parallel Block Multigrid Framework for Large-Scale Evolving Sparse Systems on Unstructured Meshes|ParaBMG 2026]] 的全部工作就是压这三项（层级复用 + 就地块逆 + 预备粗解）。

## Prerequisites

- **稀疏线性系统与 SPD**：$A$ 对称正定，二次型是碗。
- **定常迭代 = 平滑器**：Jacobi / Gauss-Seidel 为什么"局部有效"（这也是它们的局限）。
- **残差方程**：$Ae=r$，$e=x^*-x$。**粗网格解的是误差，不是解。** 这一步不理解，后面全是名词。
- **限制与插值算子** $R$（restrict）与 $P$（prolongate），以及 Galerkin 积 $A_c = P^\top A P$。
- [[Shewchuk 1994 — An Introduction to the Conjugate Gradient Method Without the Agonizing Pain|CG]]：多重网格最常见的用法是给 PCG 当预条件器（$M^{-1}=$ 一个 V-cycle）。

## Historical Evolution

```
1962  Fedorenko            首次提出多层网格（俄文，被忽视十余年）
  ↓
1977  Brandt               把多重网格变成可实践的方法
                           + 自适应网格细化（MLAT）：网格跟着解走
  ↓
1980s AMG（Brandt–Ruge–Stüben 一系）
                           不要几何，只要矩阵 ⇒ 打开"非结构/无网格"的门
  ↓
1992  Xu                   space decomposition 统一框架
                           乘法/加法的收敛界从此有出处（ParaBMG 引的就是它）
  ↓
2000  Briggs/Henson/McCormick《A Multigrid Tutorial》
                           教科书定型；V-cycle / FMG / 收敛分析成为标准语言
  ↓
2010s GPU 化               每次迭代变便宜；setup 成为新的大头
  ↓
2022–2026 MAS              非重叠块 + 就地块逆 ⇒ 弹性仿真 SOTA
  ↓
2026  ParaBMG              承认"演化"是一等公民：层级复用 + 块结构一鱼两吃
```

## Important Papers

- [[2026-11-24-Lyu — ParaBMG A Parallel Block Multigrid Framework for Large-Scale Evolving Sparse Systems on Unstructured Meshes|ParaBMG 2026]] —— 本页的当代样本：块划分既是粗化产物又是平滑器单位；40,987 个演化系统；$\alpha$ 扫描给出"什么时候乘法赢"的实测边界。
- [[Shewchuk 1994 — An Introduction to the Conjugate Gradient Method Without the Agonizing Pain|Shewchuk 1994]] —— 不是多重网格文献，但它给出"$\kappa$ 决定的 $\sqrt\kappa$ 台阶"，**这是理解多重网格为什么还要再降一级的坐标系**。
- ⚠️ **未入库（本页最大缺口）**：
  - **Brandt 1977**, *Multi-Level Adaptive Solutions to Boundary-Value Problems*, Math. Comp. 31(138):333–390 —— 这条线的源头。**已试 4 条路径（AMS 403 / Weizmann 404 / fatcat TLS 失败 / 各镜像无直链）**，需换 `scholar.archive.org` 或大学课程镜像。
  - **Fedorenko 1962/1964** —— 更早的源头（俄文），通常只在综述里被转述。
  - **Xu 1992**, *Iterative Methods by Space Decomposition and Subspace Correction*, SIAM Review 34(4):581–613 —— 乘法/加法界的出处，ParaBMG 与 MAS 一系共同的引用锚点。
  - **Briggs, Henson & McCormick 2000**, *A Multigrid Tutorial*（SIAM，第 2 版）—— 教科书；若要动手实现，从这本开始。

## Related Concepts

- [[Level of Detail]]、[[Scalability and Quality Tiers]] —— **结构同构但目的不同**：LOD 用不同精度省**渲染**，MG 用不同精度省**迭代**。共同点是"不同尺度用不同表示"，差别是 MG 的每一层处理的是**上一层明确没处理的那段误差**（LOD 没有这个承诺）。
- [[Mesh Simplification]]、[[Subdivision Surfaces]] —— 粗化/细化的几何侧亲戚；MG 的层级构造本质上是一次"保结构的简化"。
- [[Contact Dynamics]] —— 同一个引擎里的另一层：那边解 LCP（非凸、可无解），这边解 SPD（凸、唯一）。**两层用两套数学，别混。**
- [[Profiling and Measurement]] —— $T_{\text{setup}}$ 算不算进时间里，是判断一个求解器数字能不能比的前提。

## Technologies

> 这一层目前没有独立笔记，先以"技术名 + 它在哪"列出。

- **AmgX / BoomerAMG / AMGCL** —— 通用 AMG 的 GPU/CPU 实现；ParaBMG 的主要基线。
- **Surface Multigrid / GravoMG / GravoTet** —— 面向非结构网格几何多重网格（几何感知粗化）。
- **MAS（多重网格加法 Schwarz）** —— 弹性仿真 SOTA；StiffGIPC 的实现是当前参照。
- **cuDSS / PARDISO** —— 直接法，在"只能解一次"的场景里仍是更好的选择。
- **Chebyshev 平滑** —— 比 Jacobi 更强的平滑器，常用于最粗层与谱估计。

## Game Applications

1. **布料 / 软体 / 破碎**（IPC 家族）：每步解 $K+\Delta t^{-2}M$。**时间步越长越需要 MG；步长越短，块 Jacobi 类够用。**
2. **自动蒙皮权重**（QHW）：网格一密，PDE 求解就卡死；MG 把它变成可交互。
3. **网格编辑 / DCC**：拖点即解，交互性 = 解一次的时间。
4. **参数化 / UV**：边界一动就重解，同理。
5. **体素 / SDF 处理**：Poisson 重建类问题（[[Isosurface Extraction]] 的上游）也是同一族方程。

**反例**：如果系统只有几千个未知数、且一帧只解一次，**层级构造的固定开销会吃掉全部收益**。$O(n)$ 的优势只在 $n$ 大、且要解很多次时显形。

## Personal Knowledge

- **Easy**：迭代法为什么"越迭代越慢"（低频残留）；"多分辨率"这个想法在图形学里到处都是。
- **Normal**：平滑器只杀高频的机制（信息一次一格）；粗层上低频变高频；V-cycle 的成本 ≈ 两次细网格扫描；$\alpha$（屏蔽）决定乘法/加法谁赢。
- **Hard**：Xu 1992 的 space decomposition 收敛界；Galerkin 积的块化；AMG 的 coarsening 在系数剧烈变化时为什么失效。

## Learning Gap

1. **缺源头原文**：Brandt 1977 / Fedorenko 1962 / Xu 1992 三篇都未入库。**这使本页的"历史"部分是二手转述**——而本库的标准是"历史关系必须核验"。
2. **缺"层级构造"的实现细节**：GMG 的粗化在非结构网格上到底怎么选点、怎么保证插值质量，目前只有 ParaBMG 的 §3 一个来源。
3. **缺"什么时候不该用"的实测**：$n$ 多小的时候 MG 输给 CG？没有数字。
4. **缺 GPU 侧的数值差异**：$P/R$ 算子的浮点行为、粗层直接解在 fp32 下是否够。

## Next Step

1. **（30 分钟）读 ParaBMG 的 §3 + 图 2/8/11**：三张图分别给"分层的时间尺度""两个机制可叠加""耦合长度决定胜负"。
2. **（20 分钟）手算一次 V-cycle 的工作量**：$n=2^{20}$，算 $\sum n_l$，验证 ≈ $2n$。**这一步能把"$O(n)$"从口号变成数。**
3. **（可选，1 小时）写一个二维 Poisson 的 V-cycle**：最粗层直接解，用 $\sin$ 误差看收敛因子是否与 $n$ 无关。**这是唯一能真正"摸到"多重网格性质的实验。**
4. **（待办）补 Brandt 1977 原文**：换 `scholar.archive.org` / 课程镜像，别再用 AMS。

---

**相关每日记录**：[[2026-11-24]]
