---
type: paper
title: "Surface Splatting"
authors: ["Matthias Zwicker", "Hanspeter Pfister", "Jeroen van Baar", "Markus Gross"]
year: 2001
published: 2001-08-12
venue: "SIGGRAPH 2001（ETH Zürich + MERL Cambridge）"
url: "https://www.cs.umd.edu/~zwicker/publications/SurfaceSplatting-SIG01.pdf"
code: "无（2001 年；Algorithm 1 伪码 + 完整雅可比推导，可实现）"
project_page: ""
category: ["point-based-rendering", "texture-filtering", "anti-aliasing", "splatting", "transparency"]
importance: A
historical_importance: 5
game_relevance: 3
production_readiness: "Industry Adopted（公式层面：3DGS 的 $\Sigma'=JW\Sigma W^\top J^\top$ 就是本文的核；实现层面当年是 0.7 fps 的 CPU 软件渲染）"
user_level: Normal
status: unread
---

# Surface Splatting

## TL;DR

**"splat" 这个词今天人人都在用，但它的数学是这一篇给的。**

2001 年的问题是：激光扫描能产出**几百万个不规则分布的点**，每个点带高分辨率纹理。你要把它们画出来，就面临一个传统管线里被三角形掩盖掉的问题——**这些点不是像素，它们的"脚印"在屏幕上是各向异性的椭圆，而且会随视角剧烈变化**。用圆形 splat 会糊，用不带低通的椭圆 splat 会走样。

本文做的事，是把 Heckbert 的**纹理重采样框架**从"规则网格"推广到"不规则点采样"，得到**屏幕空间 EWA 滤波**的一个闭式核：

$$\rho_k(\mathbf x)=\frac{1}{|J|}\,G_{\,J V_k J^\top \,+\, I}\!\left(\mathbf x-m(\mathbf u_k)\right)$$

**方括号里那个 $+I$ 是全文的心脏**：$J V_k J^\top$ 是原语自己的（被投影扭曲的）重建核，$I$ 是**像素本身的有限面积**。两者相加，意味着"重建"与"带限"在同一式里完成——**抗锯齿不是后处理，是核的一个加数**。

三个数字说明它当年有多贵、今天有多便宜：

| | 本文（2001，1.1 GHz Athlon，未优化 C） | 3DGS（2023，GPU） |
|---|---|---|
| 42.9 万点 @512² | **0.7 fps** | — |
| 478 万点 @512² | **0.1 fps** | 同量级场景 **>100 fps** |
| 每像素缓冲 | **114 B**（3 层 × 38 B），512² 共 **25.5 MB** | 排序，无多层缓冲 |

**同一条公式，隔了 22 年，由硬件兑现。** 这不是"老论文过时了"，而是"**有些论文要等两代硬件才变成产品**"。

> 一句话对读者：**它回答的是"一个 splat 在屏幕上该覆盖多大、边缘怎么衰"——这个问题在 2026 年 Hexels 那里又出现了一次，只是答案从"推导出的椭圆"变成了"选出来的六边形"。**

![[Research/Files/Pictures/Zwicker 2001 — 图1 点云渲染三例.png]]
**读图**：三张图是同一个算法的三种极端输入——左：**激光扫描的人脸**（42.9 万点，带高分辨率纹理，考验纹理滤波）；中：**带地图纹理的地形**（考验高频纹理在斜视角下的 minification）；右：**半透明的直升机**（考验顺序无关透明）。**注意左图的脸颊和鼻翼**——那里点密度变化剧烈，圆形 splat 会在这两处出现明显的"塑料感"；右图的旋翼是半透明的，说明它的 A-buffer 真的在按层混合而不是按绘制顺序。（图 1，Zwicker et al. 2001）

## Problem

点渲染（point-based rendering）当时已经能画几百万点（QSplat 等），但**没有人系统处理"点上的纹理怎么抗锯齿"**。传统管线里这件事被三角形挡住了：纹理空间→屏幕空间的映射由每个三角形的透视变换给出，各向异性过滤（如 EWA、rip-mapping）有明确的雅可比可用。**点没有三角形，没有参数化，没有全局 UV——雅可比从哪来？**

不解决的后果是具体的：

- **圆形 splat**（半径取椭圆长轴）：放大区域过度模糊；
- **只按法向定椭圆的 splat**（省略带限步骤）：缩小区域**走样**（摩尔纹）。

原文用一句话概括了这个分野：screen-space EWA "provides a continuous transition between minification and magnification"，其余两种各在其中一端失败。

## Historical Context

```text
1985  Levoy & Whitted：点作为渲染原语（"particles"）—— 想法早就有了
1986  Greene & Heckbert：EWA（椭圆加权平均）滤波器 —— **各向异性纹理过滤的经典解**
1989  Heckbert：纹理映射与图像 warping 的重采样框架 —— **本文的数学骨架**
1997  Grossman & Dally（LDI）/ 1998 Levoy & Hanrahan（Light Field）：图像空间的另一种组织
2000  Pfister 等《Surfels》：点升级成"面元"—— 有形状、有法向、有颜色
2000  Rusinkiewicz & Levoy《QSplat》：层次结构 + 实时画百万点，但**每点是常数色**
2001  ★ 本文：把 EWA 推到不规则点上，**屏幕空间闭式核 + 修改版 A-buffer**
2002  Zwicker 等《EWA Splatting》：硬件友好版本，成为点渲染的标准做法
2007  point-based graphics 成熟，然后**沉寂**（被三角形 + GPU 光栅化压住）
2023  3DGS：**同一个核，换成可微、换成 GPU、换成各向异性 3D 高斯** —— 引爆
2026  Hexels：把支撑域从"推导出的椭圆"换成"设计出的六边形"
```

**它的历史位置**：在它之前，"splat" 是个工程技巧；在它之后，"splat" 是一个**有明确重建核与带限保证的渲染原语**。而 3DGS 之所以能成立，恰恰因为它继承了这套数学并且**只需要改一件事**——把"点 + 固定核"换成"可学的高斯 + 可学的协方差"。

## Previous Work

| 路线 | 重建核 | 带限 | 顺序无关透明 | 主要问题 |
|---|---|---|---|---|
| 圆形 splat（Levoy / QSplat） | 各向同性 | ❌ 或启发式 | 靠 z-buffer | 放大糊 / 缩小走样 |
| 椭圆 splat（按法向） | 各向异性 | ❌（**省略带限**） | 靠 z-buffer | 缩小走样 |
| volume splatting | 3D 核预积分成 2D footprint | 启发式缩放 | — | 各向同性、无纹理 |
| **本文** | **屏幕空间 EWA** | ✅ **$+I$** | ✅ **修改版 A-buffer** | CPU 慢 |
| 3DGS（2023） | 屏幕空间高斯（**本文的核**） | 近似（膨胀 0.3） | **全局排序** | 大场景排序开销 |

**关键分野**：本文把"带限"写进了核，而后续为了性能把它**近似掉**（3DGS 用 $\Sigma'+0.3I$ 的膨胀代替严格的 $+I$）。**代价是数学严格性，收益是能在 GPU 上 tile 化。** 这个交易值得记住——**它和 [[2026-10-27-Kaushik — Hexels Compact Hexagonal Primitives for Scene Rendering|Hexels]] 把"带限"整个省掉是同一方向的更激进版本**。

## Core Idea

**别把点当成"一个像素"，把它当成"一个被投影扭曲了的重建核"；别在核外面加抗锯齿，把像素的面积加进核里。**

| 常规做法 | 本文做法 | 换来什么 |
|---|---|---|
| 点 → 屏幕位置 → 固定大小的圆 | 点 → **雅可比 $J$** → 屏幕空间的**椭圆核** | 各向异性（斜视角、minification）自动正确 |
| 抗锯齿作为后处理（超采样/MSAA） | **$G_{JVJ^\top + I}$**：带限在核里 | 一个 splat 就是一次正确的重采样 |
| 每像素一个 z（画不了透明） | **修改版 A-buffer，每像素 $l$ 层** | 顺序无关透明 + 边缘抗锯齿 |

## Technical Approach

### 1) 从"重采样"到"核"

点集给出的是加权和形式的重建：

$$\tilde c(\mathbf x)=\sum_{k\in N} w_k\, r'_k(\mathbf x)$$

关键是把每个点的基函数 $r_k$（定义在局部切平面上）**warpen 到屏幕**。设 2D→2D 映射 $m$、雅可比 $J$，则

$$r'_k(\mathbf x)=r_k(J^{-1}\mathbf x)=\frac{1}{|J|}\,G_{J V_k J^\top}(\mathbf x-m(\mathbf u_k))$$

再乘上"像素是有限面积"这一条——**与单位高斯卷积**：

$$\boxed{\ \rho_k(\mathbf x)=\frac{1}{|J|}\,G_{\,J V_k J^\top + I}\!\left(\mathbf x-m(\mathbf u_k)\right)\ }$$

于是输出的连续函数是 $\tilde c(\mathbf x)=\sum_k w_k\,\rho_k(\mathbf x)$（式 7）。**这一步没有引入任何近似**：$+I$ 就是低通。

反过来写成源空间形式（式 8）：$\tilde c(\mathbf x)=\sum_k w_k G_{V_k+J^{-1}J^{-1\top}}(m^{-1}(\mathbf x)-\mathbf u_k)$——这就是经典 EWA，只不过核的形状由**逆雅可比**决定。**两种写法是同一个东西的两面，本文选了屏幕空间那一面，因为它可以在每个点投影后直接求值，不需要纹理参数化。**

### 2) 雅可比怎么来（图 5 那件事）

$J=s_{vp}\cdot J_{pr}\cdot s_{mv}$，三段分别是视口映射、透视投影、视图变换。

- 视图变换与视口映射都被**限制成"只有旋转 + 均匀缩放"**——这样它们对雅可比的贡献只是标量，各向异性完全由投影产生；
- 透视投影那一段，做法是：把屏幕的两个基向量 $\mathbf x_0,\mathbf x_1$ **沿"相机中心→$P_k$"的视线投回切平面**，得到 $\tilde{\mathbf x}_0,\tilde{\mathbf x}_1$，再令 $\mathbf u_0=\tilde{\mathbf x}_0/\|\tilde{\mathbf x}_0\|$、$\mathbf u_1=\mathbf u_0\times\mathbf n_k$：

$$J_{pr}^{-1}=\begin{pmatrix}\|\tilde{\mathbf x}_0\| & \tilde{\mathbf x}_1\cdot\mathbf u_0\\ 0 & \tilde{\mathbf x}_1\cdot\mathbf u_1\end{pmatrix}$$

**注意这是"局部切平面近似"**——原文明确承认它和传统管线的"逐三角形透视校正"有同样的不一致性（核在三角形边缘外会采样到别的三角形），并说误差小到看不见。**这句话在 2001 年是对的，在 2023 年变成了 3DGS 的一个真实缺陷（大高斯跨越深度不连续处的"拖尾"）。**

### 3) 一个 splat 到底覆盖多少个像素

核的指数里是二次型 $\beta(\mathbf x)=\frac12\mathbf x^\top(I+J^{-1}J^{-1\top})\mathbf x$，取 $\beta<c$ 作为支撑，**典型 $c=1$**。$c$ 大则质量好、代价高。**这就是"一个 splat 的像素预算"——一个可以直接调的旋钮，而不是一个由形状决定的常数。**

### 4) A-buffer：每像素 3 层，114 字节

为了顺序无关透明与边缘抗锯齿，本文用修改版 A-buffer：每像素固定 $l$ 层，每层 38 字节（RGBA + 法向 + 权重和 + 相机空间 z），3 层共 **114 B/像素**；512² 就是 **25.5 MB**。

- 新 fragment 进来时，按 z 与已有层比较，**能合并就合并，不能就插到合适位置**；满了就挤掉最远的一层（原文给了具体的合并/挤出策略）；
- **所有点 splat 完之后再着色**（deferred shading）——因为法向是滤波过的，必须等权重和稳定。

**这是"顺序无关"在 2001 年的答案：花内存，每像素存几层。** 2023 年 3DGS 的答案是反的：**花时间，全局按深度排序，每像素只留一个**。**同一个问题，两代人给了两个相反的答案，因为一个是内存贵、一个是排序便宜。**

## Key Contribution

1. **屏幕空间 EWA 的严格形式**，把 Heckbert 的重采样框架推广到不规则点采样，**不需要全局纹理参数化**；
2. **$+I$ 带限项**：让"抗锯齿"成为核的性质，而不是后处理；
3. **最优的纹理预滤波**：把规则图像纹理采样到不规则点上（还给了"最优算法"与"更快的视角无关 EWA 预滤波"两个版本）；
4. **修改版 A-buffer**：顺序无关透明 + 边缘 AA，代价是每像素固定层数。

## Why It Works

**因为"一个点在屏幕上覆盖多大"这件事，本来就有唯一正确答案，而它由投影的雅可比决定。**

只要接受"点是局部平面上一个各向同性的核"这个前提，投影就唯一地把它变成一个椭圆，椭圆的形状由 $J$ 给出，椭圆的"胖瘦下限"由像素面积 $I$ 给出。**两件事都是几何事实，没有可调参数**（除了截断半径 $c$）。这就是为什么这个核在 22 年后原封不动地出现在 3DGS 里——**它不是某个设计的产物，它是几何的产物**。

而对面的选择（[[2026-10-27-Kaushik — Hexels Compact Hexagonal Primitives for Scene Rendering|Hexels]] 的六边形）恰恰说明：**一旦你允许多花一点参数（一阶颜色场），"形状"就从"被几何决定"变成"可以优化"**。**这是两条路的真正分界。**

## Limitations

- **当年不实时**：1.1 GHz Athlon 上 42.9 万点 @512² 只有 **0.7 fps**，478 万点 **0.1 fps**。作者自己写的是 "unoptimized C implementation"。
- **A-buffer 的内存随分辨率线性涨**：512² 就要 25.5 MB，1080p 三层就是 **~114 MB**——在 2001 年是奢侈，在今天是带宽灾难。
- **切平面近似**：跨越深度不连续的大 splat 会采样到错误的深度/法向。原文认为误差可忽略，**但在今天的高动态范围、大高斯场景里这成了可见的拖尾**。
- **只处理不透明的几何 + 固定层数透明**：超过 $l$ 层的重叠会退化成近似。
- **没有 minification 的层次结构**：本文明确说 "there was no hierarchical data structure used"。**几十万点可以，几千万点不行。**

## Game Development Relevance

- **它是"粒子/sprite 该多大"这个问题的严格答案**。Niagara 的 sprite 目前是"屏幕空间尺寸 × 一个固定比例"，本质上是**圆形 splat**那一档——放大糊、缩小闪。**本文给的是"按视角算椭圆"**，代价是要为每个粒子算一次雅可比。
- **`+I` 这个加数可以直接用**：很多人调 sprite 的 soft particle / mip bias 时在凭经验凑一个"最小尺寸"，**那个最小尺寸的物理含义就是 $I$**——它是像素面积，不是一个美术参数。
- **不适用**：2001 年的 A-buffer 方案在今天的 GPU 上不划算（带宽），而 3DGS 的排序方案在超大粒子数下也不划算。**今天真正的答案是"按 tile 分桶 + 每 tile 排序"**，那是这两条路的中点。

## Unreal Engine Relevance

- **Niagara sprite 的 mip / 尺寸选择**：对应本文的"预滤波"一节——如果粒子用的是图集（sub-UV），**预滤波必须按子图的 footprint 做**，否则会出现图集串色。这是工程上最常见的坑之一。
- **材质里的 `PixelDepth` / 屏幕导数**：本文的雅可比在 HLSL 里就是 `ddx/ddy` 的解析版本。**用解析雅可比比用 `ddx` 稳**（`ddx` 在 2×2 quad 边界上会跳）。
- **不适用**：A-buffer 那套在 UE 里对应 OIT（顺序无关透明），UE 已有 per-pixel linked list 方案，不需要本文的三层固定缓冲。

## Technology Evolution

```text
"一个 splat 的核"
1986  EWA：椭圆加权平均 —— 各向异性过滤的经典解（Greene & Heckbert）
1989  Heckbert：重采样框架 —— 规则网格上的严格处理
2001  ★ 本文：**不规则点上的屏幕空间 EWA** + 显式带限 $+I$ + A-buffer
2002  EWA Splatting：硬件友好化，成为点渲染的标准
2007  点渲染沉寂（三角形 + GPU 光栅化赢了十年）
2023  3DGS：**同一个核**，换成可学协方差 + GPU tile 排序 —— 带限退化为 $+0.3I$ 的膨胀
2026  Hexels：**换掉形状本身**（六边形）+ 一阶颜色场 —— 带限彻底省略
        ↑ 三步：核被推导 → 核被近似 → 核被替换
```

**一条主线**：**"正确"在 2001 年是可负担的（0.7 fps 也行，因为离线扫描），"近似"在 2023 年是必须的（要实时），"替换"在 2026 年是划算的（因为近似已经到位，剩下的是形状）。** 这不是技术进步的单向箭头，是**约束在变**。

## Relationships

### Based On

- Heckbert 的重采样框架（1989，库外）—— 本文把它的"规则网格"前提去掉；
- EWA 滤波器（Greene & Heckbert 1986，库外）—— 各向异性过滤的核，本文把它搬到屏幕空间。

### Extends

- [[Particle Systems]]（Reeves 1983）—— 同样是"用一堆小片覆盖屏幕"，但 Reeves 的片是**各向同性、无带限**的；本文给了它一个正确的核；
- 点渲染（QSplat / Surfels，库外）—— 从"能画百万点"到"画对百万点"。

### Related

- [[Gaussian Splatting]] — **直接后代**。3DGS 的 $\Sigma'=JW\Sigma W^\top J^\top$ 与本文的 $JV_kJ^\top$ 是同一个式子，差别只有那个低通项（本文严格推导出 $+I$，3DGS 用 $+0.3I$ 的膨胀近似）。**读懂本文，3DGS 的投影公式就不需要"记"了**；
- [[2026-10-27-Kaushik — Hexels Compact Hexagonal Primitives for Scene Rendering|Hexels]] — **同一问题的第二个答案**：本文的椭圆是**推导**出来的（唯一解），Hexels 的六边形是**选**出来的（优化解）。**从"推导"到"选择"，是多了一个自由度，也多了一笔理论欠账**；
- [[Overdraw]] — 本文的 A-buffer 层数（3 层 / 114 B）是"overdraw 该怎么记账"的一个早期形态：**它不数混合次数，它数"每像素最多能存几层"**；
- [[Real-Time Rendering]] — 2001 这个节点在时间线上属于"点渲染分支"，与"三角形分支"并行发展了 20 年后在 3DGS 处汇合。

### Followed By

- EWA Splatting（Zwicker 2002，库外）—— GPU 友好版本；
- [[Gaussian Splatting]]（Kerbl 2023）—— 22 年后的产品化；
- [[2026-10-27-Kaushik — Hexels Compact Hexagonal Primitives for Scene Rendering|Hexels]]（2026）—— 原语形状成为设计变量。

## Personal Knowledge State

`Normal`。**结论层约 20 分钟**：抓住"核 = 投影扭曲 + 像素面积"和"顺序无关透明靠内存换"两条即可。

值得停下来看的两处：

1. **式 (7) 里那个 $+I$**——它是"抗锯齿不需要后处理"的全部理由；
2. **图 8 的四行对比**——它是"少一个加数会怎样"的完整分类。

## Learning Value

- **对 TA**：`blends/pixel` 与"每像素几层"是同一个量的两种记法。本文的 3 层上限说明——**overdraw 在硬件里一直是有上限的，只是那个上限被隐式写在 ROP 里，不写在你的预算表里**。
- **对架构**：这是一篇"**把一个工程技巧追到数学根上**"的论文。它的价值不在算法多快，在于**它证明了这里的每一步都是被迫的，不是随便选的**——而"被迫"的东西最容易被继承 20 年。
- **对评审**：看到"我们的 splat 用了高斯核"时，问一句"**低通项在哪？**"。没有低通项的 splat，在 minification 下一定走样。

## Visualization

![[Research/Files/Pictures/Zwicker 2001 — 图5 屏幕空间雅可比的几何推导.png]]
**读图**：左边方框是屏幕空间（$z=1$），格点是像素；右边斜放的平面是 $P_k$ 处的**切平面**（也就是局部参数化的源空间）。**看两根红色箭头**：$\tilde{\mathbf x}_0,\tilde{\mathbf x}_1$ 是屏幕的两个基向量**沿视线投回切平面**得到的——它们一般**不正交、不等长**，这个"不正交不等长"就是各向异性的全部来源。再把 $\mathbf u_0$ 取成 $\tilde{\mathbf x}_0$ 的方向、$\mathbf u_1$ 由右手系定出，雅可比就退化成一个**上三角矩阵**（式里那个 $J_{pr}^{-1}$）。**这一张图解释了为什么 splat 在斜面上会被拉长**——不是因为斜面"看起来扁"，是因为投回去的两个基向量长度不等。（图 5，Zwicker et al. 2001）

![[Research/Files/Pictures/Zwicker 2001 — 图8 四种splat方式的滤波对比.png]]
**读图**：同一张棋盘格地面，四种画法，从上到下。**第一行（屏幕空间 EWA）**：近处棋盘清晰、远处逐渐收敛成灰——**这是正确的 minification**。**第三行（圆形 splat，半径取椭圆长轴）**：远处糊成一团，近处也糊——**放大时过度模糊**。**第四行（按法向定椭圆、省略带限）**：近处锐利，但**远处出现明显的摩尔纹**——这就是少了那个 $+I$ 的直接后果。**第二行（源空间 EWA）**与第一行接近，差别在斜视角处的轻微过度模糊。**四行摆在一起，就是"核的形状"与"核的带限"这两个变量的完整笛卡尔积。**（图 8，Zwicker et al. 2001）

## Notes

- **证据形态**：一手 PDF，作者主页版（`cs.umd.edu/~zwicker/.../SurfaceSplatting-SIG01.pdf`，5.2 MB，8 页），页眉写 "To appear in the Proceedings of SIGGRAPH 2001"。**所有数字取自 §8 与 Table 2**。
- **一处需要核实的表述**：本文常被引作 "SIGGRAPH 2001"，PDF 页眉是 "To appear in"，**说明这是接收稿而非正式排版稿**；页码与正式 proceedings 可能不同。**引用时用 SIGGRAPH 2001 无误，但别引本文页码**。
- **与 3DGS 的关系要写清楚"同"与"不同"**：同——投影后的协方差都是 $JVJ^\top$ 形式，都由雅可比决定各向异性；不同——(a) 本文的 $V_k$ 是**固定的**基函数方差，3DGS 的 $\Sigma$ 是**可学的**；(b) 本文严格加 $I$，3DGS 用 $+0.3I$ 膨胀近似；(c) 本文用 A-buffer 解决顺序，3DGS 用全局排序。**三条差异没有一条是"改进公式"，全是"换交易"。**
- **一个容易被忽略的细节**：本文对视图变换限制"**不允许非均匀缩放与错切**"，理由是"保持基函数在相机空间的旋转不变性"。**这条限制在 3DGS 里被悄悄去掉了**（$W$ 是任意视图矩阵），代价是核可能被非均匀地拉——但因为 $\Sigma$ 是可学的，训练会补偿掉。**这是"公式没变、前提变了"的典型例子。**
