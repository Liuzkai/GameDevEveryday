---
type: paper
title: "Hierarchical Incremental Beta Splatting for Online Reconstruction"
authors: [Linus Franke]
year: 2026
published: "2026-11-05（arXiv v1）；WACV 2027"
venue: "WACV 2027"
url: "https://arxiv.org/abs/2611.08288"
code: ""
project_page: "https://linus-franke.com/hibs/"
category: [gaussian-splatting, online-reconstruction, lod, kernel, merging, confidence]
importance: A-
historical_importance: 2
game_relevance: 3
production_readiness: Prototype
user_level: Normal
status: unread
aliases: [HIBS, Hierarchical Incremental Beta Splatting, Beta Splatting 在线重建, 矩匹配合并]
tags: [gaussian-splatting, lod, kernel, merging, neural-rendering, online]
---

# HIBS（Franke 2026）

## TL;DR

**"能合并"比"能表达"稀缺。这篇的贡献不是更好的基元，是给基元配了一个"可逆的签名"。**

两件事：

**① 一个标量同时是形状和置信度。** beta 核的形状参数 $\beta=4e^b$ 从"软高斯"（$\beta$ 大）连续滑到"平顶实心面片"（$\beta\to1$）。作者把它当**置信度旋钮**用：几何**可验证**的地方 spawn 成实心（$\beta=1$），**不可验证**的地方 spawn 成软探针（$\beta=5$），随观测累积逐步变硬。

**② 闭式合并，因为矩可逆。** 合并四个基元成一个父级，通常只能"平均"——而平均会**把形状糊掉**（图 5 中间那列：变成一团过大的紫色雾）。HIBS 能做到保形，是因为它的三个矩都有闭式、而且**峰度严格单调于 $\beta$**，于是可以从合并后的峰度**反解**出合并后的 $\beta$：

$$\kappa(\beta)=\frac{15(\beta+2.5)}{\beta+3.5}\ \Longrightarrow\ 15-\kappa(\beta)=\frac{15}{\beta+3.5}\ \Longrightarrow\ \beta=\frac{15}{15-\kappa}-3.5$$

> **这是今天真正想记住的一句：一个表示能不能被无损合并，取决于它的参数能不能被一组"可加的矩"唯一确定。** 平均是默认操作，而默认操作只在"被平均的量在合并区域内是常数或线性"时才对。

## Problem

在线重建（边拍边重建）把离线 3DGS 的假设全部反过来了：

- 每个关键帧的基元**只 spawn 一次**，之后只有**几十次**迭代的优化 burst；
- **没有 clone/split 加密**，没有不透明度重置，没有长 schedule 去修初始化；
- 于是 **"spawn 出什么，模型就留下什么"**。

而高斯基元在这个语境下是**结构性不合适**的：高斯**只在中心完全不透明**，其他地方半透。要用高斯表示一个实心表面，必须**堆很多个重叠的高斯**——而这正好需要离线才有的加密与长时间优化。

论文的判词很准：

> 在 spawn-once / optimize-briefly 的约束下，**基元本身**才是瓶颈，而不是优化策略。

第二层问题更隐蔽：**"有多相信这个几何"这件事，在表示里没有位置**。系统只能对所有基元一视同仁地 spawn 成半透明高斯，然后希望短 burst 能摆对。

## Historical Context

```text
2023  3DGS：各向异性高斯 + 可微 splatting（离线，几万次迭代）
  ↓
2024  Deformable Beta Splatting（DBS）：把高斯换成紧支撑 beta 核
        · β 是【离线学出来的紧凑度自由度】，和 densification 一起优化
        · 关键遗产：beta 核的【矩有闭式】
  ↓
2025  On-The-Fly NVS / i3DGS：在线重建，spawn-once + 短 burst
        · 但基元仍是高斯 ⇒ 只能 spawn 半透明的，靠堆叠
  ↓
2026 ★ HIBS：把 β 从"离线紧凑度"改成"在线置信度"
        · 再补一刀：β 可从峰度反解 ⇒ 合并闭式且保形
```

> **DBS 给了工具，HIBS 给了用途。** 同一个 $\beta$，离线语境下它是"这个基元该多紧凑"（一个被优化的自由度），在线语境下它是"这个基元值不值得被相信"（一个从证据写入的状态）。**参数没变，语义变了——这是"重用"最省的一种形式。**

## Core Idea

**用"形状"表达"置信度"，因为这两件事本来就是同一件。**

一个基元"有多像实心表面"和"我有多确定这里真有表面"，在几何上是一回事：
- 确定 ⇒ 敢用平顶、近不透明的小片（**承诺**在这里）；
- 不确定 ⇒ 只能用软塌塌的、可修正的团（**留后路**）。

所以不需要额外的置信度通道，**β 就是那个通道**。

## Technical Approach

### beta 核

局部坐标 $x=r^2/9$（支撑 $r\le3$，沿用常见的 3σ 范围），不透明度 $o$、形状参数 $b\in\mathbb R$：

$$\alpha=o\,\mathcal B(x;b),\qquad \mathcal B(x;b)=(1-x)^{\beta(b)},\qquad \beta(b)=4e^b$$

- $b=0$（$\beta=4$）：形状**接近高斯**；
- $\beta$ **减小**：顶部变平，趋向实心平台；
- $\beta$ **增大**：更软、衰减更快。

颜色用 DBS 的 spherical-beta 基：$C(v)=c_0+\sum_k a_k\max(0,d_k\!\cdot\!v)^{4e^{b_k}}$。**在线场景下所有 lobe 幅度零初始化**——离线 DBS 用非零初始化再优化掉，但在线只有几十次更新，非零初始化会留下**持久的视角相关色偏**；零初始化没有这个问题，且梯度在零点非零。

### 置信度从哪来：两个独立的几何线索

$$\text{confidence}\quad c=(c_{mvs}\cdot c_z)^\gamma,\qquad \gamma=4$$

- **深度可观测性** $c_z=\mathrm{clamp}\big((B/z)/\theta_0,0,1\big)$，$\theta_0=2°$。$B$ 是最大共视基线，三角化角约 $B/z$。**基线相对于深度太小时，很多深度都能解释这组视图**——这正是"低光度误差不等于几何正确"的机制来源。
- **深度一致性** $c_{mvs}=\mathrm{clamp}\big(1-|i_{mvs}-i_{mono}|/r_{mvs},0,1\big)$，在**逆深度**空间比较引导 MVS 与对齐后的单目先验；匹配代价剖面平坦时置零。

然后直接写进初始化：

$$o_{\text{spawn}}=o_0+(o_1-o_0)c,\qquad \beta_{\text{spawn}}=\beta_{\text{soft}}-(\beta_{\text{soft}}-\beta_{\text{solid}})c$$

$o_0=0.1,\ o_1=0.3,\ \beta_{\text{solid}}=1,\ \beta_{\text{soft}}=5$。**无可靠 MVS 对应时用 0.02 的不透明度下限。**

注意还有一步**尺度补偿**：$\beta$ 变了，半高宽就变了，所以 $s\leftarrow s\sqrt{u_{1/2}(\beta_1)/u_{1/2}(\beta_2)}$，$u_{1/2}(\beta)=1-2^{-1/\beta}$，基准 $\beta_1=4$。**没有这步，改 $\beta$ 就等于偷偷改了大小。**

### 生命周期：置信度会长

每关键帧一次，用 EMA（$\eta=0.3$）跟踪基元主导像素的光度误差均值 $\bar e$ 与方差 $\sigma_e^2$，然后 $b\leftarrow b\pm\Delta b$（$\Delta b=0.4$）：

| 方向 | 条件 |
|---|---|
| **变硬**（$\beta$ 减） | 老（≥10 kf）+ 不透明（$o>0.7$）+ 观测充分（>20 主导像素、≥3 视图）+ 误差低（$\bar e<0.03$） |
| **变软**（回 $\beta=5$） | 观测充分 + 误差高 + **强视角不一致**（$\bar e>0.03$，$\bar e^2/(\bar e^2+\sigma_e^2)<0.5$） |
| **不动** | 误差高但**视角一致** ⇒ 交给 relocation（那是几何错，不是外观问题） |

**最后这条分流是全篇最漂亮的一个判断**：同样是"重建不好"，**视角一致的误差说明几何位置错了**（该搬走），**视角不一致的误差说明外观表达不了**（该软化）。**同一张误差图，两种病因，两个科室。**

### 闭式合并（今天的核心）

每个基元带 $(o,\mu,\Sigma,b)$。三个量都有闭式：

$$Z=\text{mass},\qquad \mathrm{Cov}=\xi(\beta)\Sigma,\quad \xi(\beta)=\frac{4.5}{\beta+2.5},\qquad \kappa(\beta)=\frac{15(\beta+2.5)}{\beta+3.5}$$

**$\Sigma$ 参数化的是支撑椭球，不是散布**——因为峰越尖，质量越集中在中心，所以需要一个 shrink factor 把"支撑"和"散布"分开。

合并（质量权重 $w_i\propto Z_i$，偏移 $\delta_i=\mu_i-\mu^\star$）：

$$\mu^\star=\sum_i w_i\mu_i,\qquad \Sigma_{\text{mix}}=\sum_i w_i\big(\xi(\beta_i)\Sigma_i+\delta_i\delta_i^\top\big)$$
$$\beta^\star=\frac{15}{15-\hat\kappa}-3.5,\qquad \Sigma^\star=\frac{\Sigma_{\text{mix}}}{\xi(\beta^\star)},\qquad o^\star=\frac{\sum_i Z_i}{Z(\beta^\star,\Sigma^\star)}$$

**为什么不用 KL？** 论文的理由很短但很硬：**对紧支撑核，父级支撑之外的任何非零质量都会让 KL 散度无穷**。所以 KL 在这个族上根本不适用——**不是"效果差"，是"没定义"**。

> **这一段的可迁移性极高**：合并策略的选择不是品味问题，是**表示的函数族决定的**。高斯有全支撑 ⇒ KL 可用（且与矩匹配等价，见图 5 左列）；beta 有紧支撑 ⇒ KL 失效 ⇒ 只能走矩匹配，而矩匹配可行的前提是**矩有闭式且可逆**。

### 光栅化边界：一条正确性修复

$$x_{\text{cut}}=\max\big(0,\ 1-(255o)^{-1/\beta(b)}\big)$$

论文明确指出：**标准 3DGS 的高斯截断阈值对"实心、低不透明度"的 beta 核是不够的**，会丢掉仍含非零贡献的 tile。所以这个 beta 专用边界**既是加速，也是修 bug**。

> 这是一个很好的"换了基元就得重算所有下游假设"的例子。高斯截断阈值（比如 3σ）来自高斯尾部的衰减率；**平顶核的尾部衰减完全不同**，直接沿用就是错的。

### 其余工程

- **error-driven relocation**：把"快死的"基元（低于回收不透明度阈值、在层级外、过了 spawn 保护窗口）**传送**到误差热点（按每基元误差归因采样，限制在前 10%），复制目标的位置/形状/颜色，用 3DGS-MCMC 的 render-preserving 调整组合不透明度与尺度，再给一点位置抖动 + 优化器重置，让后续 burst 把副本分开。
- **层级**：**四个投影都低于半像素的邻居**合成一个父级；不活跃的细层 offload 到主机内存。
- **融合光栅器**：FasterGS 式，反向时按基元重算前向量、中间梯度留寄存器、稀疏 Adam 直接作用在优化器状态上 ⇒ **基元梯度从不落地到显存**。

## Key Contribution

1. **把"形状参数"重新语义化为"置信度"**，并证明它可以从几何证据初始化、随观测演化。
2. **给出 beta 基元的闭式合并**（保形、保质量），从而支撑屏幕空间 LOD 层级。
3. **指出紧支撑核上 KL 合并不适用**，并给出替代（矩匹配 + 峰度反解）。
4. **beta 专用的光栅化边界**（正确性修复，不只是优化）。
5. **误差分流**：视角一致 → relocation，视角不一致 → 软化。

## Why It Works

**因为"形状"是一个同时被几何和不确定性需要的量。**

如果只用不透明度表达置信度，遇到一个矛盾：**不透明度低既可能是"不确定"，也可能是"这个表面本来就半透明"**。两个语义挤在一个通道里。加一个 $\beta$ 之后：
- $o$ 管"贡献多强"（辐射度量）
- $\beta$ 管"几何承诺多深"（几何量）

**这是把两个正交的语义分到两个参数上。** 而它们恰好都能被同一族核平滑地表达——所以不需要新基元，只需要一个新解释。

**合并能闭式，是因为 $\beta$ 与峰度是双射。** 这一点值得展开：合并时你能算的是**矩**（质量、协方差、峰度），你能要的是**参数**（$o,\mu,\Sigma,\beta$）。如果矩到参数的映射存在且唯一，合并就是"算矩 → 反解"；如果不存在（多对一、或不可解），就只能近似。**HIBS 的 $\beta(\kappa)$ 是显式的，所以它是"算得出来"的那一类。**

## Limitations

**① 标题主打的两项，消融里都不是最大的。** 9 个 otf 场景平均（Full = 25.30）：

| 去掉什么 | PSNR | 差 |
|---|---|---|
| w/o regularizers | 24.98 | **-0.32** ← 最大 |
| w/o error relocation | 25.09 | -0.21 |
| w/o SB lobes（换回 SH） | 25.12 | -0.18 |
| w/o confidence | 25.14 | -0.16 |
| w/o solidification | 25.15 | -0.15 |
| w/o fused rasterizer | 25.24 | -0.06（但 120 s vs 101 s） |

**"置信度 spawn" 与 "solidification" 加起来 0.31，还不如一个正则项。** 这与 [[2026-11-05-Wang — Perception Over Geometry Rethinking Gaussian Allocation in Mesh-based Splatting|P-MGS]] 的"消融表比标题诚实"是同一种现象，值得当成读论文的固定动作。

**② 大场景上真正值钱的是合并，不是在线机制。** SmallCity：full 24.45 → **w/o moment merge 23.78（-0.67）**；CityWalk：25.81 → **23.54（-2.27）**。**这两个数字比上面所有单项消融都大一个量级**——说明"层级怎么建"在城市尺度上是主项，"在线怎么 spawn"是次项。

**③ TUM 上时间反而更慢**（1:44 vs i3DGS 1:24，+24%）。论文自陈 "except time on TUM"。摘要里的 "up to 40% faster" 是 MipNeRF360 那一组的成绩。

**④ Deep Blending 是弱项**：26.49 vs i3DGS **27.16**——**输了 0.67 dB**。换回 SH 能补约 0.3 dB。**spherical-beta 颜色基不是普适更优。**

**⑤ $\beta$ 的"置信度"语义会被优化器污染。** 论文自己写："through gradient descent, $\beta$ is free to be optimized outside the clamped range." 也就是说图 4 那张"置信度场"读到的其实是**置信度写入 + 优化器偏好**的混合。**论文没有量化这个污染有多大**——而这正是"参数复用"的代价：**你借用了这个参数，就不能独占它。**

**⑥ 位姿骨干是 i3DGS 的**（Meuleman et al. 的 online pose estimation），不是自研；单作者，WACV 2027。

**⑦ 只在"在线采集"这个语境下成立。** 离线场景下 β 的置信度语义没有意义（因为没有"证据随帧累积"这个结构），那里 β 仍然只是紧凑度。

## Game Development Relevance

**直接相关性中等，机制相关性很高。**

- **直接**：游戏内实时 capture（玩家扫描场景、AR 场景理解）是它的目标场景，但游戏工业目前不做这件事。
- **机制**：
  1. **"一个参数同时是自由度与置信度"** —— 任何"渐进式构建"的系统（运行时 LOD 生成、流式加载、渐进式资产）都需要"我有多确定这块数据"，而**加一个专用置信度通道比复用一个已有参数贵得多**。
  2. **闭式合并的条件** —— 这与 [[Garland 1997 — Surface Simplification Using Quadric Error Metrics|QEM 可加]]、以及 11-08 那条"可加 ≠ 可比"是同一条线上的第三个例子：**能不能合并，取决于有没有一个可加且可逆的签名**（QEM 是二次型系数，HIBS 是矩）。
  3. **换基元要重算所有下游假设** —— 高斯截断阈值直接搬到 beta 上就是错的。**这条在引擎里天天发生**（换一种粒子、换一种代理体、换一种 LOD 表示，缓存/剔除/排序的假设全部要重推）。

## Unreal Engine Relevance

- **不直接可用**：UE 没有在线辐射场重建管线。但 **"四合一 + 半像素判据"的屏幕空间 LOD** 与 Nanite 的簇选择是同一个判据，只是对象不同（基元 vs 簇）。
- **可借鉴**：**"误差按视角一致性分流"** —— 视角一致的误差 ⇒ 几何错；视角不一致的误差 ⇒ 外观错。UE 的 LOD 切换抖动诊断可以用同一套分流：**如果某个 LOD 在所有视角都错，是几何/简化的问题；如果只在某些视角错，是法线/材质的问题。**
- **可借鉴**：**尺度补偿**。$\beta$ 变了半高宽就变，必须补偿。**UE 里任何"改了形状参数"的操作（比如用曲线改粒子大小、用材质改不透明度曲线）都要问一句"半高宽变了吗"。**

## Technology Evolution

见 [[Gaussian Splatting]]（基元族的演化）与 [[Level of Detail]]（屏幕空间判据）。

**本篇的位置**：GS 基元族的**第五个自由度**——在位置 / 协方差 / 不透明度 / 颜色之外，加了一个**形状**（$b$），并且这个形状**同时承载置信度**。

## Relationships

### Based On
- Deformable Beta Splatting（DBS, 2025）—— beta 核与 spherical-beta 颜色基（库外）
- i3DGS（2025）—— 在线位姿估计骨干（库外）
- 3DGS-MCMC —— relocation 的 render-preserving 修正（库外）
- [[Gaussian Splatting]] —— 基元族与光栅化框架

### Extends
- DBS 的 $\beta$：从"离线优化的紧凑度"改为"在线写入的置信度 + 可由矩反解的合并参数"。

### Improves
- 在线重建的**初始化质量**：i3DGS 的 MipNeRF360 (otf) 26.66 → **27.47**（+0.81 dB），时间 2:09 → **1:18**（-40%）。

### Related
- [[Level of Detail]] —— 屏幕空间 LOD，判据是"投影小于半像素"
- [[Garland 1997 — Surface Simplification Using Quadric Error Metrics|Garland 1997]] —— **同一条"可合并性"主线**：QEM 的二次型系数可加，HIBS 的矩可加且可逆（见 [[Level of Detail]] 的「可合并性」一节，2026-11-09 补）
- [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] —— **对照**：T2-Splat 的拓扑合并用加权平均（不可逆）⇒ 只能靠罚项；HIBS 的合并可逆 ⇒ 保形
- [[Radiance Fields]] —— 在线重建的父类

### Contrasts
- **高斯基元**：全支撑 ⇒ KL 可用，但表示实心表面要堆叠。
- **KL 式合并**：对紧支撑核**没有定义**（支撑外质量 ⇒ 无穷散度）。

### Followed By
- 待观察：$\beta$ 的置信度语义与梯度优化能否分离（限制 ⑤）。

## Personal Knowledge State

- **机制层 Easy**：splatting、mipmap、LOD、EMA —— 日常。
- **今天新增（Normal）**：
  1. **可合并性的判据是"矩可加且可逆"**，不是"算法好不好"。这条把 [[Level of Detail]] 的"表示"栏从"有没有 LOD 链"推进到"**这个表示能不能造出 LOD 链**"。
  2. **紧支撑 ⇒ KL 失效**。选择合并策略前先看函数族的支撑。
  3. **误差分流**：视角一致 = 几何错，视角不一致 = 外观错。
- **仍是 Hard**：为什么 $\kappa(\beta)$ 恰好是这个形式（beta 分布的峰度解析式）。这条不影响使用，但影响"能不能推广到别的核"。

## Learning Value

**最高价值的一条：$\beta=\frac{15}{15-\kappa}-3.5$ 这个反解。**

它很短，但它代表了一类设计：**如果你想让一个表示"可合并"，先问"它的参数能不能从一组可加的统计量反解出来"**。能，就闭式；不能，就只能近似（然后你就得在别处付误差）。

**第二价值**：**消融表又一次比标题诚实**——最大贡献是正则项（-0.32），标题主打的两项合计 -0.31。**当作固定动作：先读消融表的行标题，再读摘要。**

## Visualization

![[Research/Files/Pictures/Franke 2026 — 图2 HIBS 在线重建管线.png]]

**读图**：五段流水线——**input**（无序 RGB 流 + 在线位姿）→ **confidence spawn**（观测性 + 一致性 ⇒ 初始化 $o$ 与 $\beta$）→ **error relocate**（回收死基元到热点）→ **optimize**（30 次 burst + $\beta$ solidity 控制器）→ **hierarchy**（四个合一）。右上角的色条是**今天的钥匙**：$\beta=5$ 软（浅绿）到 $\beta=1$ 实（深紫），**同一个核在不同 $\beta$ 下的形状差异就是"承诺程度"的可见形式**。注意最下面那条"once per keyframe"的括线——**整条流水线每个关键帧只跑一次**，这是在线场景的硬约束。
（图 2，Franke 2026, WACV 2027）

![[Research/Files/Pictures/Franke 2026 — 图5 矩匹配合并与 KL 合并的对比.png]]

**读图（今天第二该看的一张）**：三列都是"四个合一"。
- **左（Gaussians）**：KL 与矩匹配**等价**（高斯族里两者给出同一结果），合并后是一个干净的小团。
- **中（beta, KL-style）**：把 beta 当高斯合并（$\Sigma$ 直接当协方差、$\beta$ 取平均）⇒ 得到**一团过大的紫色雾**：footprint 胀了、形状糊了、颜色因为重复叠加而过饱和。**这就是"平均"在紧支撑族上的失败。**
- **右（beta, ours）**：$\xi(\beta)$ 收缩 + $\kappa^{-1}$ 解码 ⇒ **footprint 保住、形状保住**（色相偏移是 beta 族本身的色基效应，不是合并误差）。

**中间与右边的差别只有两个符号：$\xi(\beta)$ 和 $\kappa^{-1}$。** 但它们决定"合并四次之后还认不认得出原来是什么"。
（图 5，Franke 2026, WACV 2027）

## Notes

- **主结果（Table 1，pose-free）**：TUM 25.94/0.868/0.175/1:44（i3DGS 25.36/0.858/0.188/**1:24**）；**MipNeRF360(otf) 27.47/0.852/0.204/1:18**（i3DGS 26.66/0.835/0.224/2:09）；StaticHikes 22.49/0.710/0.259/2:02（i3DGS 22.17/0.699/0.269/2:53）。
- **3DGS 基准（Table 2，GLOMAP 顺序建图，7k 步）**：MipNeRF360 **25.19** / T&T **23.00** / DB 26.49；i3DGS 24.75 / 22.08 / **27.16**。**时间全线最快（1:17 / 1:18 / 1:22 vs 1:59 / 1:25 / 2:03）**——**除了 T&T 输给 i3DGS 的 1:25。**
- **大场景（Table 3）**：SmallCity 24.45/0.808/0.285/**3:47**（Octree-GS 24.18 但 ~1h）；CityWalk **25.81**/0.857/0.239/**42:43**（i3DGS 22.03，**+3.78 dB**）。
- **合并消融（大场景）**：w/o moment merge ⇒ SmallCity **-0.67**、CityWalk **-2.27**。**这是全篇最大的单项贡献，比所有在线机制加起来都大。**
- **参数**：$\gamma=4$，$o_0=0.1$，$o_1=0.3$，$\beta_{\text{solid}}=1$，$\beta_{\text{soft}}=5$，$\theta_0=2°$，$\Delta b=0.4$，EMA $\eta=0.3$，$\lambda=0.15$，$\lambda_R=10^{-2}$。**除 $\gamma=4$ 外全是整数值，看得出是手选的。**
- **未报数字 / 待核**：① $\beta$ 的"置信度语义"被梯度优化污染多少（限制 ⑤）；② 层级重建的周期与代价；③ 与 DBS 在**离线**语境下的正面比较（本篇只在在线协议下比）。
- 2026-11-09 建页。与 [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] 同批：**两篇都在问"合并怎么做"，一篇靠罚项，一篇靠矩反解。**
