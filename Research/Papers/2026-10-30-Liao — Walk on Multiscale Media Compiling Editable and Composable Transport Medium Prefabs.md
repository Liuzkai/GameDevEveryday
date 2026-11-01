---
type: paper
title: "Walk on Multiscale Media: Compiling Editable and Composable Transport Medium Prefabs for Monte Carlo PDE Solvers"
authors:
  - Yunfeng Liao
  - Xiucheng Li
year: 2026
published: 2026-10-24
venue: arXiv preprint (physics.comp-ph; cs.GR)
url: https://arxiv.org/abs/2610.35952
code: ""
project_page: ""
category:
  - rendering
  - physics-simulation
  - geometry-processing
importance: A
historical_importance: 0
game_relevance: 3
production_readiness: Research
user_level: Hard
status: read
---

# Walk on Multiscale Media —— 把"介质"编译成带端口的资产

> 哈尔滨工业大学（深圳），Yunfeng Liao / Xiucheng Li，`2610.35952`，12 页预印本（Preprint 版式）。
> **一句话**：如果一块异质材料的内部结构不变，那么"光/热在里面怎么走"这件事**只需要算一次**——把它编译成一个**带端口的输运预制件（transport prefab）**，之后无论谁来问、问多少次、改了多少次吸收系数，都只是在这个预制件上做**重加权**与**局部更新**，不再重新走细网格。

## TL;DR

- **它继承的是 [[2026-10-30-Liao — From Walks to Skeletons A Reusable Transport Architecture|同作者的 Walks to Skeletons]] 的下一步**：那篇把**游走轨迹**存成图，这篇把**介质本身**存成资产。前者缓存"答案的推导过程"，后者缓存"问题的结构"；
- **核心理论动作只有一个**：证明**多尺度有限元基函数 = 首达概率**（$\Phi_{x\beta}=\phi_\beta(x)$），于是"确定性算子压缩"与"随机输运"变成同一个对象的两种读法；
- **由此得到的直接推论**（全文最该记的一条）：**刚度矩阵的 Schur 补，就是一台马尔可夫链的转移矩阵**——非对角元是"从一个端口走到另一个端口"的概率，对角元是"在这里被吸收"的概率。**消元不是数学技巧，它是"把内部积分掉"这件物理事的矩阵写法**；
- **可编辑性的来源是"存证据不存答案"**：预制件里存的是**Poisson 标记的事件记录**（终点 + 途经节点多重集），而不是首达概率本身。于是换一个吸收场 $\kappa$，只需按 $\prod(1-\kappa(z)/\mu)$ 重加权——**一条采样路径可以回答无穷多个"如果吸收更强会怎样"**；
- **代价与收益都量化了**：端到端在线 **22.4×–28.2×**（12 次查询，4096 路径/查询），精度四项里赢三项；**局部 Schur 更新在编辑面积 <5% 时快 25.9–42.9×，编辑满整个内部时只剩 1.8×**——**本地更新的成本随"改了多少"涨，全量重构的成本是常数，两者必然交叉**。

## Problem

异质介质（多孔材料、复合材料、空间变化的吸收体、导热体）在图形学里到处都是。它们的**细尺度结构**（远小于显示分辨率的通道与隔断）会显著改变宏观输运，所以"想粗算"也得"先细算"。

已有的两条路各缺一半：

| 路线 | 强项 | 缺的一半 |
|---|---|---|
| **WoS（[[Walk on Spheres]]）** | 无网格、点求值、任意几何 | **每次查询都从零走**。同一块介质被问一百次，就走一百次 |
| **MsFEM（多尺度有限元）** | 把细尺度压进粗算子，粗网格求解便宜 | 算子是为**离散系统**设计的，不是为**点查询**设计的；改材料就得重装配 |

而交互场景真正需要的是第三种东西：**同一块介质被反复查询、被空间拼接、被局部编辑**。原文的措辞很准：

> "Ideally, the transport behavior of a complex medium should be compiled once into a reusable representation, **much like geometric assets**, while remaining responsive to changes in the surrounding environment."

**"像几何资产一样"**——这句话就是全文的立场。几何资产的定义特征是：建一次、实例化无数次、可以拼、可以换材质。这篇要给"输运"同样的地位。

## Historical Context

把它放回 WoS 的"轴二"（怎么变便宜）上看，位置非常清楚：

```text
1956  Muller            球面跳跃（无状态，每次重走）
2020  Sawhney & Crane   带进几何处理
2022  Sawhney et al.    变系数（异质介质的第一步）
2023  Boundary Value Caching     存"一个值"
2023  Mean Value Caching         存"一个体积均值"
2025  Harmonic Caching           存"一个局部展开"
2026  Walks to Skeletons         存"一整张依赖图"（← 本月 10-30 入库）
2026  ★ WoMM                     存"介质本身"（本篇）
```

**这条轴的每一次上移都在做同一笔交易：把构建成本前置，换后续查询变便宜。** 而本篇是这条轴上第一次**把"被查询的对象"也变成可存的东西**——前五站存的都是"这次计算的产物"，只有它存的是"下次计算的**前提**"。

一句话区分三篇 2026 年的 WoS 工作：

| 论文 | 存什么 | 换什么变了要重算 |
|---|---|---|
| [[2026-09-15-Grid-Free Monte Carlo for Time-Dependent Diffusion\|时间依赖 WoS]] | 不存 | 什么都得重算 |
| [[2026-10-30-Liao — From Walks to Skeletons A Reusable Transport Architecture\|Walks to Skeletons]] | **轨迹图** | 边界值改了：换叶子；几何改了：局部修复 |
| **WoMM（本篇）** | **介质预制件** | **材料/吸收改了：重加权；拼位置变了：重装矩阵** |

## Previous Work

- **均质化与多尺度离散**：经典均质化（Allaire 1992；Bensoussan 1978）用"有效算子"代替"系数平均"；MsFEM（Hou & Wu 1997）把异质系数吃进局部椭圆基函数；HMM（E et al. 2003）耦合微观与宏观；GMsFEM 加富局部空间；LOD / operator-adapted 多分辨（Målqvist 2014；Owhadi 2017）构造能量自适应的基。**这篇借的是"局部算子压缩"这个动作，但要的不是更快的粗网格解，而是可编辑的随机接口**；
- **WoS 家族**：Muller 1956 → Sawhney & Crane 2020 → 变系数（2022，用变换 + null-scattering 估计器）→ Walk on Stars（2023，Neumann）→ 预计算逆 Laplacian / 调和延拓算子（Cîmpean 2026）→ 时空游走（Bao 2026 / Zhou 2026）；
- **可复用随机算子**：Boundary Value Caching（Miller 2023）、Walk on Probes（Huang 2026）、**WoDS**（Jambon 2026，子域首达算子 + 全局界面系统）、Monte Carlo Steklov Operators（Maesumi 2026）。**WoDS 是最近的邻居**——两者都要"装配全局界面系统"，区别是 WoDS 面向**几何编辑**，本篇面向**材料编辑**，且本篇的算子携带"入口相关"的出口分布（$F_\kappa$），因此能在**连续**求解器里进出。

## Core Idea

### 一、把"基函数"读成"概率"

细网格图上的离散能量

$$E_h(u)=\tfrac12\sum_{\{x,y\}}c_{xy}(u_x-u_y)^2+\tfrac12\sum_x\lambda^0_K(x)u_x^2=\tfrac12u^\top H_Ku$$

把边界节点分成 $p$ 个**端口**（用 $T\in\{0,1\}^{n_b\times p}$ 指派），消掉内部节点，得到 **Schur 补**

$$S_K=T^\top H_{bb}T-R^\top H_{ii}^{-1}R=:D_K-J_K .$$

到这里全是标准 MsFEM。**转折在下一句**：给电导图配一个连续时间马尔可夫链（在 $x$ 停留 $\mathrm{Exp}(d^0_x)$，以 $c_{xy}/d^0_x$ 跳到 $y$），再加"被杀率" $\kappa$，则

$$(\Phi_\kappa)_{x\beta}=\phi^\kappa_\beta(x)=\mathbb{E}^0_x\!\left[e^{-\int_0^\tau\kappa(X_t)dt}\,\mathbf 1\{Y\in\Gamma_\beta\}\right]$$

**基函数的值 = 从 $x$ 出发、在 $\Gamma_\beta$ 出界的（带杀死的）首达概率。** 而 $J_\kappa=R^\top\Phi_\kappa$ 的元是

$$(J_\kappa)_{\alpha\beta}=\rho_\alpha\,\mathbb{P}_{X_0\sim q_\alpha}\{Y\in\Gamma_\beta\},$$

即**端口间的未归一化输运量**。于是

$$P^{\kappa}_{\alpha\beta}=-(S_\kappa)_{\alpha\beta}/s_\alpha\ (\beta\ne\alpha),\qquad P^{\kappa}_{\alpha\dagger}=(S_\kappa\mathbf 1_p)_\alpha/s_\alpha$$

**是一个合法的马尔可夫转移矩阵**（$s_\alpha=(S_\kappa)_{\alpha\alpha}>0$）。

> **为什么这件事重要**：Schur 补是 $M$-矩阵的 Schur 补，仍是 $M$-矩阵；而"非负、行和 ≤1"的矩阵就是一个次随机矩阵。**所以"消元"与"走一步"在这个结构里是同一件事**——这不是类比，是同一个矩阵的两种解释。工程含义：**你可以用确定性的稀疏线性代数去算一个随机游走的转移核**，也可以用随机采样去估一个确定性的算子。

### 二、预制件里存什么：不是答案，是证据

$$\mathrm{Prefab}_K=(K,\ \mathcal A^0_K,\ T,\ \Phi_K,\ S_K,\ \mathcal E_K,\ F_K)$$

- $\mathcal A^0_K$：固定的细尺度扩散算子（**不参与编辑**）；
- $\Phi_K,S_K$：多尺度算子（可增量更新）；
- $\mathcal E_K$：**事件记录库**——每个内部节点 $x$ 采 $N_x$ 条**无屏蔽**游走，压缩成 $(Y_{x,r},\Pi_{x,r})$：终点 + **以固定速率 $\mu$ 采到的 Poisson 候选事件上所访问节点的多重集**；
- $F_K$：入口相关的出口分布（在稀疏**锚点**上估计，之间三线性插值）。

给定任意屏蔽场 $\kappa$，首达核**不需要重新走**：

$$\hat F_\kappa(x,b)=\frac1{N_x}\sum_{r=1}^{N_x}\mathbf 1\{Y_{x,r}=b\}\prod_{z\in\Pi_{x,r}}\left(1-\frac{\kappa(z)}{\mu}\right) \tag{4}$$

**这就是体积渲染里的 null-collision / Woodcock 跟踪**：把"被吸收"做成"以概率 $1-\kappa/\mu$ 存活"，而路径本身与 $\kappa$ 无关。**同一套记录，可以回答任意 $\kappa\le\mu$ 的问题。**

> 这里是今天最重要的一句设计判据：**存"答案"只有一个答案，存"证据"可以回答很多问题。** 代价是证据更占地方、更噪，而且每次使用都要再加工一次（重加权 / 滤波）。

### 三、编辑为什么是"局部的"：resolvent identity

设编辑前后 $A_1=A_0+V$（$V=\mathrm{diag}(v)$，$v\ge0$ 只加不减）。因为 $\Phi_t=A_t^{-1}R$、$J_t=R^\top\Phi_t$，有

$$\Delta J:=J_0-J_1=\Phi_0^\top V\Phi_1,\qquad (\Delta J)_{\alpha\beta}=\sum_{x\in S}v(x)\,\phi^{pre}_\alpha(x)\,\phi^{post}_\beta(x)$$

其中 $S=\{x:v(x)>0\}$ 是**被编辑的节点集**。**求和只跑编辑区**——这就是"局部"的全部来源。边界项 $D$ 显式变成 $D^{post}=D^{pre}+T^\top V_bT$，于是

$$\hat S^{post}=\hat S^{pre}+\widehat{\Delta J}+T^\top V_bT .$$

**一个必须记的实现细节（也是本文最漂亮的一处技术）**：$\widehat{\Delta J}$ 要估的是**两个概率的乘积**，而两者来自**同一批记录**（$a_r=W^{pre}_r\mathbf 1\{Y\in\Gamma_\alpha\}$，$b_r=W^{post}_r\mathbf 1\{Y\in\Gamma_\beta\}$）。直接乘样本均值是有偏的（同一记录的相关性无法分开）。作者用**二阶 U-统计量**配对**不同**的独立样本：

$$\widehat{\Delta J}_{\alpha\beta}=\sum_{x\in S}\frac{v(x)}{N(N-1)}\Big[A_\alpha(x)B_\beta(x)-\mathbf 1\{\alpha=\beta\}C_\alpha(x)\Big]$$

并证明 $\mathbb E[\widehat{\Delta J}_{\alpha\beta}]=\Delta J_{\alpha\beta}$，$\mathrm{Var}\le\frac1N\sum v(x)^2$（Thm 4.1）。

> **可迁移的判据**：**从同一批样本里估两个量的乘积，必须先扣掉对角线。** 这和 [[2026-10-30-Liu — Texture Consolidation of Generated Multi-View Renderings|Texture Consolidation]] 里"置信度必须是一个显式的场"是同一类错误——**把相关性当成独立性，误差不会报错，只会悄悄偏向一边**。

### 四、两种"不用重走"的编辑：方向与时间

- **方向（steering）**：Doob $h$-变换给出漂移 $b_h=2a_K\nabla\log h$。若取 $h$ 为端口权重 $w$ 的**调和延拓**，则 $L_0h=0$、修正项 $V_h$ 消失，只剩**端点重加权**
$$F^{w}_\kappa(x,b)=\frac{w_{\beta(b)}}{h_w(x)}F_\kappa(x,b).$$
  **"往哪边走"只改出口的权重，不改路径。** 方向控制被压缩成 $p-1$ 个端口权重比——这就是"低维方向盘"；
- **时间（热容）**：对 $m_K\partial_tu-\nabla\cdot(a_K\nabla u)+(\lambda^0_K+v)u=0$ 做 Laplace 变换，**热容变成等效屏蔽** $\kappa_\sigma=\lambda^0_K+v+\sigma m_K$。于是"改热容"与"改吸收"走同一条重加权通道，**同一个 U-统计量直接复用**（$v\to\delta v+\sigma\delta m_K$）。

**这两条合起来说明**：一个预制件真正携带的不是"三个功能"，而是**一个统一的"有效势"$\kappa_\sigma+V_h$**——屏蔽、非调和方向、热容全部进这一项，漂移只进端点因子。

### 五、连续与离散怎么缝合

外层是标准 WoS（在均匀区用球跳），一旦进入预制件的**内触发区** $E_{in,K}$：查所在粗单元 → 用 $\hat F_{\kappa,j}(x,\cdot)$ 采一个**粗单元边界**上的点 → 映射到该单元的端口 → 在**内部骨架** $S_{skel}=\sum_jE_j^\top S_{K_j}E_j$ 上走 → 出外端口时按条件分布 $Q^{K_j}_{\alpha\beta}(b)$ **还原细边界位置** → 回到连续 WoS。

**三级分辨率各管一件事**（连续位置 → 粗端口 / 骨架 → 细出口），而"内触发 / 外出口"分离是为了**防止在人工界面上来回抖**。最后是**自适应**：查询点距预制件 $d(q,K)\le r$ 用细级，否则用粗级，**每次查询决定一次、整批游走共用**。

## Technical Approach

| 环节 | 做法 | 关键数字/参数 |
|---|---|---|
| 细网格 → 粗端口 | 电导图 + 端口指派 $T$，Schur 补 $S_K=D_K-J_K$ | 端口数 $p$：8 太粗，26/56 才有明显增益 |
| 算子估计 | 只在**邻接边界的内部节点**采样即可装配 $\hat S_K$（不需要基函数梯度/法向导数） | 64 路径/节点：基函数误差 **12.08%**，**输运误差 0.330%**，构建 1.45 s；256：6.10% / **0.126%** / 5.57 s；1024：3.03% / 0.083% / 23.09 s |
| 入口相关出口 | 稀疏锚点 + 三线性权重插值 $F_\kappa(x,\cdot)=\sum_a\omega_a(x)F_\kappa(x_a,\cdot)$ | 锚点间距 1/2/4 网格步 |
| 事件库 | Poisson 标记（速率 $\mu$），记录 $(Y,\Pi)$ | 64/128/256 条/节点（**注意：误差不随库增大单调下降**，见 Limitation） |
| 编辑更新 | resolvent identity + 二阶 U-统计量 | 成本 $O(|S|[N(1+M)+p^2])$，**不产生新轨迹** |
| 方向 | 调和 Doob $h$（端口权重延拓）⇒ 端点重加权 | $w_\beta=\exp(\theta\,\mathbf d\cdot(\mathbf p_\beta-\mathbf p_K))$，只需 $p-1$ 个比值 |
| 时间 | Laplace 域：$\kappa_\sigma=\lambda^0+v+\sigma m_K$ | 同一事件库，零新轨迹 |
| 求解 | 连续 WoS + 内部骨架 + 细出口还原 | 内触发/外出口分离防抖 |

## Key Contribution

1. **把 MsFEM 的基函数解释成首达概率**，从而把"确定性算子压缩"和"随机输运"接到同一根轴上——于是**消元 = 走一步**，Schur 补 = 转移矩阵；
2. **提出"输运预制件"这个资产形态**：带端口、带入口相关出口分布、带事件记录库；可实例化、可空间拼接（同一介质重叠 = 表示冗余，可任选一个；物理叠加 = 屏蔽场相加 $\lambda=\lambda_1+\lambda_2$）、可嵌入连续求解器；
3. **两条"零新轨迹"的编辑通道**（调和 steering 的端点重加权、热容的等效屏蔽），并证明它们与屏蔽编辑共用一个估计器；
4. **给出编辑成本的正确计量方式**：不是"场景多大"，是"**改了多少**"。

## Why It Works

三条各自独立的理由：

1. **为什么端口能代表内部**：因为 $\Phi_\kappa$ 是**被杀死**的调和延拓，"从端口值回到内部值"这件事由 $H_{ii}^{-1}$ 唯一决定。端口越多，可表达的边界数据越丰富——8 个端口连"光滑边界数据"的出口依赖都保不住（Fig 2a），26 起步才够；
2. **为什么事件记录能重加权**：因为**路径分布与 $\kappa$ 无关**（$\kappa$ 只改"存活概率"）。这是 Feynman–Kac 公式的直接后果，也是体积渲染里 null-collision 能成立的同一个理由；
3. **为什么编辑是局部的**：resolvent identity 把 $\Delta J$ 写成"编辑区上的双重积分"，**非编辑区对 $\Delta J$ 的贡献在代数上被约掉了**（不是被忽略，是被消掉）。

## Limitations

1. **只做椭圆 + Dirichlet**。Neumann、变系数（$a$ 随位置变）、真时间依赖都不在范围内。作者自己给了时间域的延伸设想（Remark 3：用 Walk-on-Heat-Balls 的记录 + 时间戳标记），**但明说需要"更丰富时空记录"，本实现没有**；
2. **计时不含预处理**（"preprocessing is excluded"），且**单线程 CPU**。22–28× 是在线阶段；构建成本是"约一次 vanilla 解"，多次查询才回本——**这一点和 Walks to Skeletons 的"8 次求解才回本 8.1×"是同一类口径问题，引用时必须带条件**；
3. **事件库不是越大越好**：Fig 3 里 64/128/256 的误差**非单调**。作者的解释是"粗转移一旦被解析，在线采样与空间近似就主导了差异"——换句话说，**再多的证据也补不上表示层的损失**。这条比任何加速比都重要；
4. **端口数 / 锚点间距是两个独立的压缩旋钮，失效方式不同**：端口太少伤"光滑边界数据"，锚点太稀伤"从哪里出去"的空间变化。**混着调会误判**；
5. **强屏蔽 + 局部编辑会退化**：编辑区很小时相对误差上升（参与估计的起始节点变少，而要估的量更小）。Table 3 的 24 次强编辑里，全 Hybrid 的增量误差明显大于纯局部算子测试——**因为局部算子更新还要穿过端口压缩、入口插值和随机采样三层近似**；
6. **Doob 条件化（port-guided sampling）试过但收益有限**，作者如实写了"observed improvement was limited"。**这是一条负结果**：条件在"成功出界"上，并不等于改进了"局部输运变化"的采样。

## Game Development Relevance

**先划清边界**：这篇解的是**椭圆 PDE**（稳态扩散/热传导），不是辐射输运。所以它**不是**"给体积雾做实时 GI"的方案。真正的可迁移点是三条：

1. **"区域 = 带端口的算子"这个资产形态**。游戏里所有"建一次、查无数次、还要能局部改"的东西都是同一个结构：Lightmap / Light Probe / 预计算可见性 / 烘焙好的雾。**它们现在的问题都是"改一处要重烘一片"**；这篇给的答案是：**把区域切成带端口的块，编辑只在块内的编辑节点上付账**；引用处 2016→2026 的差别只在于"块内那层用什么估"；
2. **"编辑成本 ∝ 编辑面积"是一个可用的成本模型**。Table 4 那张表（0.8% → 36.2×，100% → 1.8×）本质是一条**交叉曲线**：局部法在 $\lesssim5\%$ 时碾压，超过 30% 就只剩 3×，满编辑 1.8×。**这决定了工具该在什么时候自动切回全量重建**——这是一个可以直接写进编辑器里的阈值；
3. **"存证据不存答案"对 VFX 的直接含义**：**如果你想支持"美术在编辑器里拖一个吸收滑块"，就不要烘"结果"，要烘"还能再算一次的中间量"**。代价是资产更大、运行时更噪。

## Unreal Engine Relevance

- **最接近的现有形态是 Volumetric Fog 的密度/散射预计算与 Lightmass 的 photon/irradiance 缓存**——它们都是"区域预计算"，但**都没有"端口"这个概念**，所以块与块之间只能靠边界条件硬凑；
- **可试的最小实验（不改渲染器）**：拿一块静态雾体积，把它当成"带 8 个面的盒子"，用这篇的端口语言问三个问题——① 我在这个盒子上定义了几个自由度？② 换一个吸收系数，我的预计算还有效吗？③ 如果美术把雾的体积挪了 20 cm，我要重算什么？**这三个问题会立刻暴露现有烘焙方案的"重算面积"**；
- **Niagara 侧**：如果粒子系统里有"稳态场"（风场、温度场、浓度场）需要反复查询，**"编译成带端口的场块"比"每次重跑仿真"更接近正确做法**；但注意本篇是 CPU 单线程的预印本，**离 GPU 化还有距离**。

## Technology Evolution

```text
"异质介质里的输运，怎么算第二次"
1996  光子图（Jensen）              存"证据"（光子 + 方向），换 BRDF 不用重走
2002  PRT（Sloan）                  存"算子"，换灯不用重算
2019  DDGI（Majercik）              存"探针"，换视角不用重算
2023  Boundary Value Caching        存"边界值"，换查询点不用重走
2025  Harmonic Caching              存"局部展开"，换查询点不用重走
2026  ★ WoMM                        存"介质"，换材料不用重走   ← 本篇
        ↓
      被存的对象：答案 → 算子 → 探针 → 值 → 展开 → **介质**
        ↓
      "存的东西越靠近问题端，可复用的范围越大，构建成本越前置"
```

## Relationships

### Based On

- [[Walk on Spheres]] — **同一个随机游走的连续形式**。本篇的"内部骨架"是离散的，外层仍是球跳；两套表示通过 $F_\kappa$ 与 $Q_{\alpha\beta}$ 对接；
- [[Linear Transport Theory]] — $S_\kappa$ 的马尔可夫解释来自"输运方程 ↔ 椭圆 PDE"的同一性：**这里的"端口"就是体积版的"边界面元"**；
- [[2026-10-30-Liao — From Walks to Skeletons A Reusable Transport Architecture|Walks to Skeletons]] — 同作者前作，提供"轨迹可存成图"的全部论证与 Doob 条件化工具。

### Extends

- **MsFEM / 均质化**（Hou & Wu 1997 等）：把"为粗网格解设计的算子"改造成"为点查询与随机接口设计的算子"；
- **WoDS**（Jambon 2026）：从"子域首达算子 + 全局界面系统"走到"**入口相关**的出口分布"，从而能进出**连续**求解器（不只是离散界面耦合）。

### Related

- [[2026-09-15-Grid-Free Monte Carlo for Time-Dependent Diffusion]] — **对照**：那篇用"时间预算"扩展 WoS 的**方程域**，本篇用"预制件"扩展 WoS 的**复用域**。两者正交，理论上可以叠加（作者在 Remark 3 里也这么想）；
- [[2026-10-30-Böttcher — Total Deformation-based MPM for Perfectly Elastic Solids|Total-Deformation MPM]] — **同一周的两个"把状态移出求解器"的样本**：那篇把 $F$ 移出积分（每次从位置重算），这篇把"介质响应"移出求解（每次从事件库重加权）。**共同点：让"历史"不再是必需的**；
- [[2026-10-30-Liu — Texture Consolidation of Generated Multi-View Renderings|Texture Consolidation]] — **"估计两个相关量的乘积"的同题**：那边用逐 texel 置信度场，这边用二阶 U-统计量。**两者都在修同一个病：把相关性当成独立性**；
- [[ReSTIR]] — **"复用"的另一个域**。可对照的是**失效方式**：ReSTIR 的失效是"蓄水池里的样本过期"，WoMM 的失效是"编辑区太小、参与估计的节点太少"。**都是"复用池里没有足够的证据"**；
- [[Budget Units]] — 本篇引入一个新的计价对象：**编辑面积**（不是场景规模、不是像素数）。

### Followed By

- （推测，未验证）**GPU 化**：事件库是"每个节点 N 条记录"的规整结构，天然适合并行重加权；真正难的是 Schur 补的增量更新（目前是 $O(p^2)$ 的稠密小矩阵，可以并行但不好 SIMD）；
- （推测，未验证）**与 [[Streaming]] 的交汇**：预制件是"可实例化的块"，正好是流式系统的天然单位——**块级预制件 + 块级流送，可能是"开放世界里的材质级 GI"的形态**。

## Personal Knowledge State

- **Normal（本页）**：Schur 补 = 次随机矩阵这一条**现在就能用**；resolvent identity 与 U-统计量需要一点矩阵微积分，但结论可以只记"从同一批样本估乘积要扣对角线"；
- **Hard（细节）**：Doob $h$-变换的 Girsanov 似然比推导（§3.2 后半）、$Q^{K_j}_{\alpha\beta}$ 的条件出口分布（式 19）；
- **关联你的已有认知**：你对"光栅化 overdraw 是层数×屏占比"很熟；**这篇的"成本 = 编辑面积 × 每节点记录数"是同一类账**——只是把"像素"换成了"编辑节点"。

## Learning Value

1. **最值钱的一条**：**"消元 = 积分掉内部 = 走一步"**。这句话把线性代数与随机过程焊在一起，从此看任何"Schur 补 / 块消元 / 边界元"都会多一个概率读法；
2. **第二条**：**存储对象的选择决定可问问题的数量**。事件记录 vs 首达概率，就是"证据 vs 答案"；
3. **第三条**：**误差在聚合后会下降一个量级**（12.08% → 0.330%）。这条对任何"我只关心总量"的估计都成立，**是对"必须把每个分量算准"这个直觉的直接反驳**。

## Visualization

![[Research/Files/Pictures/Liao 2026 — 图1 异质介质预编译总览.png]]
**读图**：左边是"没有预制件"的世界——一条游走要在细密的异质结构里一格一格地跳，每一格都要查系数；右边是"有预制件"的世界——同一块介质被压成几个带端口（彩色小块）的粗单元，游走**只在端口之间跳**，细结构的影响被"编译"进了转移概率里。**注意中间那条箭头**：它不是"近似"，是"换了一层表示"——**粗单元上的转移核，是把细尺度的所有路径积分掉之后的精确结果**（只差蒙特卡洛噪声）。**这张图想说的是：贵的东西不是"走"，是"在细结构里走"。**（图 1，Liao & Li 2026）

![[Research/Files/Pictures/Liao 2026 — 图4 在线时间与误差对比.png]]
**读图**：横轴是**12 次查询的总在线时间（ms，对数轴）**，纵轴是对有限体积参考解的 RMSE。四条线：Hybrid pre/post（蓝/绿，本篇）与 Official pre/post（橙/红，官方变系数 WoS）。**看橙线**——它的误差随预算增加而下降，但**时间几乎不动**（3000 ms 一条竖线），因为每次查询都要重走整块介质。**看蓝/绿线**——它们**在时间轴上横着走**（100 ms 量级），误差随采样数增加而下降。**两条线的水平距离就是 22–28×**。还要注意一个诚实的细节：**在 Smooth + 未编辑那一格里，蓝点比橙点略高**（RMSE 0.0099 vs 0.0082）——**预制件不是无损的，它输掉了最容易的那一格**。（图 4，Liao & Li 2026）

![[Research/Files/Pictures/Liao 2026 — 图6 三类编辑示意.png]]
**读图**：三行 = 三种编辑，四列 = 编辑输入 / 编辑前参考解 / 编辑后参考解 / **解的变化量**。第一行（SCREENING，加入吸收块）：输入是一个菱形的高吸收区，解的变化量 $\|\Delta u\|/\|u_0\|=21.8\%$。第二行（STEERING，方向偏好）：输入是一组指向右上的箭头，**解的"前后"几乎没变**（因为调和 steering 只改出口分布，不改稳态场的整体形状），变化量 8.0%。第三行（CAPACITY，热容增量）：一个圆形区域，变化量 6.6%。**这张图最重要的读法是看第四列的量级**：三种编辑的"信号"只有 6–22%，而蒙特卡洛的噪声很容易同量级——**所以"能估出来"和"能估准"是两件事**，这也是 Table 3 里强编辑误差偏大的原因。（图 6，Liao & Li 2026）

## Notes

**证据形态**：arXiv 预印本（v1，2026-10-24），12 页，Preprint 版式（未排版会议模板）。全文可读，图 1/4/6 取自原文；图 4 与图 6 为矢量图，按 6× 渲染裁剪（首轮图 4 带进页眉横线、图 6 带进副标题碎片，已各重裁一次）。

**三处口径必须跟着数字走**：
1. **22–28× 是"在线阶段"**，预处理不计时（原文 Table 2 标题明写 "preprocessing is excluded"）；
2. **单线程 CPU**（"All timed methods run serially on the CPU"风格的口径在同类论文里反复出现，本篇实验段同样声明 single-threaded）；
3. **"8 次求解回本 8.1×"是 Walks to Skeletons 的数，不是本篇的**——两篇同作者、同思路，**数字不能互相借用**。

**一处需要作者澄清的矛盾**：§5.1 说"增大事件库误差只轻微变化、且不单调"，但 Fig 8 又说"增加缓存路径降低姿态级误差"。两者不冲突（前者是**全 Hybrid 端到端**，后者是**纯局部算子增量**），**但读者很容易读成同一件事**——引用时务必分清"端到端"与"局部增量"两个层级。

**未能核实**：无代码/项目页（预印本）；32 个参考文献里 WoDS / Monte Carlo Steklov / Walk on Probes 三篇均为 2026 年新作，本库未入，**未做交叉验证**。
