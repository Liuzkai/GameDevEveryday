---
type: paper
title: "Vision and the Atmosphere"
authors: ["Srinivasa G. Narasimhan", "Shree K. Nayar"]
year: 2002
published: "2002"
venue: "International Journal of Computer Vision 48(3):233–254（Kluwer）；投稿 2001-01-31，修回 2001-10-16"
url: "https://www.cs.cmu.edu/~ILIM/publications/PDFs/NN-IJCV02.pdf"
code: ""
project_page: ""
category: ["Atmospheric Scattering", "Participating Media", "Inverse Rendering", "Physics-based Vision"]
importance: A
historical_importance: 5
game_relevance: 4
production_readiness: "Industry Adopted（两项模型与"空气透视"的思路被此后所有 dehazing 与大气透视实现继承；原论文本身是分析方法，不含实时实现）"
user_level: Normal
status: unread
---

# Vision and the Atmosphere

## TL;DR

**介质不是噪声，是编码。**

在这篇之前，计算机视觉的默认前提是"观察者浸在透明介质里"——一个像素的亮度只取决于场景里的一个点。坏天气打破了这个前提：亮度变成**两个未知量的和**。

这篇 2002 年的 IJCV 论文做了三件事，每一件都在后面二十年被反复继承：

1. **把两个机制写成加法**：衰减（Bouguer 1729 / Allard 1876）让远处变暗，airlight（Koschmieder 1924）让远处变亮。两者方向相反，合起来才是我们看到的雾。
2. **把未知量挪一个位置**：airlight 的推导里有一个未知的比例常数 $k$，他们用**地平线辐亮度 $L_\infty$（可观测）**替换掉它。于是"未知的照明"变成"可测的地平线颜色"。
3. **把颜色写成两条射线的组合**：$E(d)=p\hat D+q\hat A$。同一场景点在不同天气下的颜色**落在同一个平面上**（二色平面），于是两三张不同天气的图就能把两条射线解出来。

> **一句话判据（本页最该记）：如果"光在到达相机之前，在介质里被吸收过或被改过一次方向"，你就已经在这个问题里。剩下的全部差别只是介质是什么、边界是什么形状。**

## Problem

论文开篇的第一句就是靶子：*"Virtually all work in computer vision is based on the premise that the observer is immersed in a transparent medium (air)."*

这个前提带来一个具体而昂贵的后果：**图像亮度与场景距离解耦**。而一旦介质参与，亮度就同时依赖"物体本身"和"路上发生了什么"，两者纠缠在一个观测里。用他们自己的话说：大气调制了从场景点到观察者的信息——**所以它也可以被看作一种视觉信息编码机制**。

于是论文的动机不是"如何把雾去掉"，而是双向的：

> *"we wish to understand not only what bad weather does to vision but also what it can do for vision."*

**"坏天气能为我们做什么"**——这句话是整篇的世界观，也是它区别于所有 dehazing 工程论文的地方。距离越长，介质写得越多 ⇒ **深度可以从一张图里读出来**，只要你认识那套编码。

## Historical Context

```text
1729  Bouguer      衰减的指数律：E(d) = E₀ e^{−βd}
1876  Allard       点光源版：再乘 1/d²（于是"距离"与"衰减"分开）
1924  Koschmieder  airlight：介质自己散射环境光 ⇒ 随距离增亮，E∞(1−e^{−βd})
1952  Middleton    大气光学综述（本文主要二手来源之一）
1975  McCartney    同上；本文的两个机制都从这里转述
1997  Cozman & Krotkov  首次用 McCartney 的模型算深度——但假设"所有点的晴天亮度相同"
2002  ★ 本篇       两项写成加法；二色平面；单图/双图两套解法；MODTRAN 验证
2011  He et al.    暗通道先验（把"先验"这条路做成主流）
2018  Akkaynak     水下版：β^D ≠ β^B，且依赖反射率
2026  NIRJALA      把被丢掉的前向散射项加回来，并用它给反演增益设界
```

**这篇的位置很特殊**：它既是"物理视觉"这条线的起点，也是"图形学的大气散射"与"视觉的 dehazing"两个圈子之间那条被踩烂的桥。今天做 [[Atmospheric Scattering]] 的人读 Bruneton/Hillaire，做去雾的人读 He 2011——**两边共用的那两项公式，出处都在这里。**

## Previous Work（两条当时的路，各自缺什么）

| 路线 | 做法 | 缺什么 |
|---|---|---|
| **对比度复原**（图像处理侧） | 直方图均衡 / 对比拉伸 / 已知距离的 Oakley & Satherley 1998 | 坏天气的效应**强烈依赖深度**，全局对比操作没有这个变量 |
| **深度线索**（视觉侧） | Cozman & Krotkov 1997 用 McCartney 的模型 | 假设"所有点的晴天亮度相同"——场景点各有反射率与照明，这个假设实践上很难满足 |

**本篇的分界**：它不假设"晴天亮度相同"，而是**把那个未知量参数化**（$\hat D$ 的方向 + 一个标量 $p$），并从**多张不同天气的图**里把它解出来。**代价从"假设"转移到了"多拍几张"。**

## Core Idea

### ① 两个机制，方向相反

**衰减**（Bouguer / Allard）：一束光穿过薄层 $dx$ 的相对损失 $dE/E=-\beta\,dx$，积分得

$$E(d,\lambda)=E_0(\lambda)e^{-\beta(\lambda)d}\quad\text{（准直光束）}\qquad E(d,\lambda)=\frac{I_0(\lambda)e^{-\beta(\lambda)d}}{d^2}\quad\text{（点光源）}$$

**airlight**：视线上的每一小段体积 $dV=d\omega\,x^2dx$ 都朝观察者散射环境光，再被介质衰减一次：

$$dL(x,\lambda)=\frac{dI\,e^{-\beta x}}{d\omega\,x^2}=k\beta(\lambda)e^{-\beta(\lambda)x}dx\ \Longrightarrow\ L(d,\lambda)=L_\infty(\lambda)\left(1-e^{-\beta(\lambda)d}\right)$$

**注意这一步做了什么**：常数 $k$（含照明方向、强度、光谱与相函数形状的全部未知）在积分后被**地平线辐亮度 $L_\infty=k$ 替换掉了**。原文的措辞是 *"the above expression no longer includes the unknown factor $k$. Instead, we have the airlight radiance $L_\infty(\lambda)$ at the horizon, **which is an observable**."*

> **这是全篇方法论上最值钱的一步：不是把模型做得更准，是把未知量换成一个能测的量。** 后面所有的"从图里解出深度/颜色"都建立在这一步上。

### ② 单张图出深度（夜间点光源 / 白天雾景）

**夜间的点光源**：airlight 可忽略，只剩衰减。两张不同天气的图的比值

$$R'=\ln\frac{E'_1}{E'_2}=-(\beta_1-\beta_2)d$$

**与光源强度、与相机增益都无关**——它只是"两种天气下的光学厚度之差"（DOT）。于是两个光源的 DOT 之比就是它们的深度之比，所有光源深度可解到**一个尺度**。

**白天的雾景**（单张图）：以地平线亮度为参考，

$$S=\frac{E_\infty-E'(d)}{E_\infty},\qquad S'=\ln S=-\beta d$$

**深度到尺度可解，缺的那个尺度就是 $\beta$**。条件很明确：**至少要看得到一小块地平线**（原文：取图像最亮的区域即可）。

> **这两条合起来是本库最干净的一次"未知量计数"练习**：单图给"深度 × β"，双图把 β 消掉。**信息量不够时，你得到的不是一个错答案，是一个带未知尺度的对答案。**

### ③ 二色模型：颜色落在两条射线上

把衰减与 airlight 合起来写成向量形式：

$$\mathbf E(d)=D(d)+A(d)=p\,\hat{\mathbf D}+q\,\hat{\mathbf A},\qquad p=E_\infty r\,\frac{e^{-\beta d}}{d^2},\quad q=E_\infty\left(1-e^{-\beta d}\right)$$

$\hat{\mathbf D}$ 是"直射光颜色"的单位向量，$\hat{\mathbf A}$ 是"airlight 颜色"的单位向量，$p,q$ 是两个标量幅度。**这是一个线性模型：任何场景点的颜色都是这两条固定射线的非负组合。**

于是**同一场景点在不同天气下的颜色落在一个平面上**（二色平面，图 10a）。这个平面只由 $\hat{\mathbf D},\hat{\mathbf A}$ 决定，**与天气无关** ——所以两张或三张不同天气的图就够了。

**实验验证**：三张不同雾/霾条件下的图、约 **50 万像素**，逐像素拟合平面，平均角误差 **0.25°（雾）/ 0.31°（霾）**（图 10b 的表）。同时用 **MODTRAN 4.0** 在 $d=0.2$ km、离地 5° 的几何下验证了"$\beta$ 在可见光内几乎不随波长变"这个关键假设（图 9）。

### ④ 为什么可以丢掉前向散射，以及这条理由后来怎么被推翻

这是本页与今天另外两篇论文的接点，原文写得很明确：

> *"In deriving Allard's law, we have assumed that all scattered flux is removed from the incident energy. … We have ignored the flux scattered in the forward direction (towards the observer) by each particle. **Fortunately, this component is small in vision applications since the solid angles subtended by the source and the observer with respect to each other are small** (see Middleton (1949))."*

**理由本身是对的**：对**单张照片**，光源与观察者互相张开的立体角很小，前向散射进入那条视线的能量占比确实低。同节还明确说多重散射的**模糊效应不做建模**。

**但这条理由有一个隐含前提：你只做"一张图的复原"。** 一旦进入**多视角重建**（3DGS / NeRF），前提就变了——你要在几十个视角上一致地反演同一个场景，而每个远处的像素都要被一个 $e^{+\beta r}$ 放大。2026 年的 [[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting|NIRJALA]] 把这一项加回来，理由不是"物理更真"（作者自己说那一项是逐像素的建模选择、不建模模糊），而是**它让反演增益有界**。

> **一句话：这一项在"读一张图"里可以丢，在"解一个场景"里不能丢。** 差别不在物理，在你要解多少个未知量。

## Technical Approach（三套解法，按需要多少张图分）

| 场景 | 输入 | 解出什么 | 关键假设 |
|---|---|---|---|
| 夜间点光源 | 两张不同天气 | 各光源**相对深度**（到一尺度） | airlight 可忽略；视线不过分倾斜（同一 β） |
| 白天雾景 | 单张 | 深度（到 $\beta$ 尺度） | 可见地平线；$\beta$ 沿水平路径恒定 |
| 白天彩色 | 两三张不同天气 | $\hat D,\hat A$ + 深度分割 + 晴天颜色 | 二色模型；跨天气照明**色相不变**（强度可变） |

**三条共同的假设**（原文都写了，值得单独列）：

1. **均匀介质**：$\beta$ 在水平路径上恒定 ⇒ 观察者在地面附近、不关心天空。
2. **照明色相跨天气不变**：只有强度变 ⇒ $\hat D,\hat A$ 是常数方向 ⇒ 才能有"同一个二色平面"。
3. **单次散射主导**：多次散射造成的模糊不建模（见上）。

## Key Contribution

1. **把两个散射机制统一成一个可解的加法模型**，并显式给出"未知量在哪、怎么消"。
2. **单图出深度**（以地平线为参考）与**双图消未知**（DOT 比值）这两条干净的解法。
3. **二色模型 + 二色平面**：把一个逐像素的欠定问题，变成一个**几何约束**（所有颜色共面）。这是那个年代"用几何换信息"的典型手法。
4. **MODTRAN + 50 万像素的实测验证**：把"$\beta$ 在可见光内近似不变"从假设变成有据。
5. **一个视角**：坏天气不是干扰，是**深度信息的载体**。

## Why It Works

**因为它在"未知量的数量"上做文章，而不是在"模型的精度"上。**

三种手法，本质是同一件事的三次应用：

| 手法 | 做了什么 | 换来什么 |
|---|---|---|
| **换未知量** | airlight 里的 $k$ → 可观测的 $L_\infty$ | 一个不可测的照明常数变成图上能取到的颜色 |
| **消未知量** | 两张不同天气的比值 → DOT 之差 | $\beta$ 被消掉，只剩深度比 |
| **加约束** | 二色平面（所有颜色共面） | 逐像素的 2 未知量变成"两条固定方向 + 2 个幅度" |

**这三条今天仍然在用**：任何"从图像反推物理量"的工作，第一件事都该是**数未知量、看哪个能被观测替换、哪个能被比值消掉、哪个能被几何约束住**。这比"换一个更强的网络"便宜两个数量级。

## Limitations

原文自己列的，加上今天能看出来的：

1. **均匀介质**：$\beta$ 恒定。真实的雾有高度剖面、水下有浊度梯度。今天的 [[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting|NIRJALA]] 在非均匀浊度上 δ<1.25 从 0.987 掉到 0.586——**二十四年后同一个假设仍在付账**。
2. **单图深度缺尺度**：$S'=-\beta d$ 里 $\beta$ 未知。要绝对深度必须加一张图或加先验。
3. **需要看见地平线**：单图法取"图像最亮的区域"当地平线。城市峡谷、室内、水下都没有这个参考。
4. **照明色相跨天气不变**：这是"二色平面存在"的前提。日出/日落时色温剧烈变化，这条破了。
5. **不建模多次散射的模糊**：原文明确。**这也是 [[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting|NIRJALA]] 的前向项被批评的地方**（逐像素、不搬光）——**同一个欠账，隔了二十四年还在。**
6. **不做实时**：全篇是分析方法（平面拟合、约束求解），没有 GPU 实现，也没有时间预算。**把它变成每帧几十微秒的东西，是后来二十年工程侧的工作。**

## Game Development Relevance

**这篇对游戏的价值有两层，一层是老的，一层是新的。**

**老的那一层：空气透视（aerial perspective）与高度雾。** 每一个引擎的 `Exponential Height Fog` 都是这篇的式 (11) 的一个特例——$L_\infty(1-e^{-\beta d})$ 加上一个逐通道的 $\beta$。**它同时是三种东西**：一个物理正确的现象、一个最便宜的空间深度线索、一个美术可以直接调的构图工具。**理解了"远山发蓝 = 视线穿过的那段介质自己发的光"，就理解了为什么雾色不能随便选**——它由环境照明（天空色）决定，不由美术喜好决定。

（这条线的游戏侧展开见 [[Atmospheric Scattering]] 的 Game Applications 一节：ToD 系统、高度雾、飞行/太空游戏。）

**新的那一层：它是"介质反演"这条线的源头，而介质反演正在变成资产管线的问题。** 3DGS 扫描已经进外景管线；水下/雾天/沙尘的扫描一旦出现，就会撞上这篇 2002 年定义的欠定性。**现成工具不会告警，它只会给你一个"颜色随距离漂移"的模型。**

## Unreal Engine Relevance

- **`Exponential Height Fog`**：式 (11) 的工程化——`FogDensity` 对应 $\beta$、`FogInscatteringColor` 对应 $L_\infty$、`HeightFalloff` 把 $\beta$ 变成高度的函数（**这一步就打破了"均匀介质"假设，是有意的近似**）。
- **`SkyAtmosphere` 的 aerial perspective**：[[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 的 AP 体积做的就是这篇的积分，只是把 $\beta$ 换成"随高度变化的 Rayleigh + Mie"两族。
- **不能映射的部分**：单图/双图反解、二色平面——这些是分析工具，不是运行时系统。

## Technology Evolution

```text
1729–1924  三个定律（衰减 / 点光源 / airlight）
1975       McCartney 手册：两个机制的现代转述
1997       首次用它们算深度（但假设晴天亮度相同）
2002  ★    两项加法 + 二色平面 + 单图/双图解法（本篇）
2011       暗通道先验：把"物理约束"换成"图像统计先验"，工程上大成功
2016–18    深度学习复原：物理模型退化为"网络的一个先验"
2018–19    Akkaynak & Treibitz：水下版，β 分离且依赖反射率
2023       SeaThru-NeRF：两项进体渲染
2026       NIRJALA：三项 + 有界增益（把 2002 丢掉的那一项加回来）
```

**一条值得注意的摆动**：2002 年是**物理约束**（少未知量），2011 年是**统计先验**（暗通道），2018 年之后又回到**物理约束**（水下模型必须分离 $\beta^D/\beta^B$ 才准）。**这不是钟摆，是每次"先验"路线饱和之后，人们回去数未知量。**

## Relationships

### Based On

- **McCartney 1975 / Middleton 1952**：两个机制的二手来源（原文自陈）。
- **Bouguer 1729 / Allard 1876 / Koschmieder 1924**：三个原始定律。

### Extends

- [[Participating Media]]：给它补了"最简边界形状（平板）+ 最简相函数（各向同性）+ 可观测的边界条件（地平线）"这一格。
- [[Atmospheric Scattering]]：**这是"逆"的那一半**——那一页讲"介质怎么写进画面"，这一页讲"怎么从画面里读出来"。

### Related

- [[Inverse Rendering]]：介质是几何/材质/光照之外的**第四类未知量**；本篇是最早把它显式参数化并计数的工作之一。
- [[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting|NIRJALA 2026]]：**同一条线的 24 年后**——它加回来的正是本篇以"立体角小"为由丢掉的那一项。
- [[Subsurface Scattering]]：同一族近似的另一个域（那边是"进入介质再出来"，这边是"穿过介质"）。

### Contrasts

- **He et al. 2011 暗通道**：不建模物理，用图像统计先验。**在单图上更强、在物理上更弱**——它无法告诉你"水有多深"，只能告诉你"这里看起来像有雾"。
- **今天的端到端复原网络**：把 $\beta$、$L_\infty$、$J$ 全部交给网络。**代价是失去可解释的中间量**，而本篇全部的力气都花在"让中间量可观测"上。

## Personal Knowledge State

- **Easy**：远山发蓝、雾天对比度下降、水下偏青——做场景的人天天见；`Exponential Height Fog` 的三个参数也熟。
- **Normal（今天建立）**：**两项加法 + 二色平面**，以及那三步"换/消/约束未知量"的手法。能对着一张大雾照片说出"我现在缺的是 $\beta$ 还是 $L_\infty$"。
- **Hard**：式 (30)–(32) 里 $p=E_\infty r e^{-\beta d}/d^2$ 的推导（$r$ 是反射率与天空孔径的合成，原文说"其确切性质不重要，只要不依赖天气"）；以及"两个不同 $\beta$ 的二色平面如何求交"那套约束求解。

## Learning Value

**它教的不是"怎么去雾"，是"怎么数未知量"。**

三步手法（换 / 消 / 约束）在今天的工程里依然直接可用，而且比换一个更大的网络便宜得多。**检验方式**：拿任何一个"从图里反推物理量"的方案，数一遍未知量，问三个问题——**哪个能被一个可观测的量替换？哪个能被两张图的比值消掉？哪个能被一个几何约束压住？** 三个都答不上来的，通常就是"再训一个网络"。

**第二个价值是那句"坏天气能为我们做什么"。** 一个把干扰重新描述成信息的视角，往往比一个更好的去干扰算法走得更远。**这也是今天 [[2026-11-20-Ghorbani — Probing Physics in Motion Diffusion Models Against Their Own Estimate|Ghorbani 2026]] 在做的事的镜像**：那边是把"模型知道得更多"重新描述成"读出更容易"，从而把结论推翻。

## Visualization

![[Research/Files/Pictures/Narasimhan 2002 — 图4 空气光锥.png]]

**读图**：观察者到物体之间的那个**锥体**就是 airlight 的来源——$dV=d\omega\,x^2dx$ 在距离 $x$ 处，朝观察者散射环境光（阳光 / 天空漫射 / 地面反射三路都算），再被介质衰减一次。**公式里那个 $1/d^2$ 与 $e^{-\beta d}$ 的乘积，就是"越远越暗"；而 $1-e^{-\beta d}$ 是"越远越亮"——两个方向相反的项在同一张图上。**（图 4，Narasimhan & Nayar 2002）

![[Research/Files/Pictures/Narasimhan 2002 — 图8 二色大气散射模型.png]]

**读图**：RGB 空间里的两条射线——$\hat{\mathbf D}$（直射光颜色方向）与 $\hat{\mathbf A}$（airlight 颜色方向）。任意场景点的观测颜色 $\mathbf E$ 是 $p\hat{\mathbf D}+q\hat{\mathbf A}$，即两条射线张成的平面上的一个点。**$p$ 随距离指数衰减、$q$ 随距离饱和到 1——所以"距离"在这张图上是沿一条曲线滑动的参数，不是第三个轴。**（图 8，同上）

![[Research/Files/Pictures/Narasimhan 2002 — 图10 二色平面几何.png]]

**读图**：左（a）同一场景点在三张不同雾/霾图里的颜色 $\mathbf E_1,\mathbf E_2$ 都落在同一个"二色平面"上——**平面只由两条射线定，与天气无关，所以这是跨天气的不变量**；右（b）实测：约 50 万像素、逐像素平面拟合，平均角误差 **0.25°（雾）/ 0.31°（霾）**。**这张小表是全篇最硬的证据——它把"二色平面"从一个假设变成一个实测成立的几何事实。**（图 10，同上）

## Notes

- 原文 PDF 已归档：`Research/Files/PDF/Narasimhan 2002 - Vision and the Atmosphere.pdf`（22 页，2.0 MB；来源 CMU ILIM，`NN-IJCV02.pdf`）。
- 该文的两位作者后来都成了这条线的关键人物：Nayar 组的 ILIM 实验室持续做散射介质成像（水下、雾、组织），Akkaynak & Treibitz 的 Sea-thru 系列直接建立在本篇的两项结构上。
- 一个有意思的措辞：原文说 $L_\infty$ 是 "an observable"，而不是"一个需要标定的参数"。**这个用词差别就是整篇的方法论**——把不可测的东西换成可测的，问题就降一维。
- 原文对"为什么照片里的雾看起来是白的"给了一个不显眼但重要的解释：airlight 的颜色由**环境照明的光谱**决定（通常就是天空色），所以雾色不是常数，它随天气与太阳位置变。**这就是为什么引擎里的雾色必须跟天空走，而不是跟美术的心情走。**
- 全篇没有 GPU 实现，也没有时间预算。**这一点常被误读成"理论论文"，其实它是"分析论文"**——它的目标读者是"想从图里量出东西"的人，不是"想每帧画一遍雾"的人。
