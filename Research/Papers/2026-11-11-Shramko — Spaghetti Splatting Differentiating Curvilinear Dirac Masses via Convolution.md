---
type: paper
title: "Spaghetti Splatting: Differentiating Curvilinear Dirac Masses via Convolution"
authors: [George Shramko, Shrisudhan Govindarajan, Rudy Lam, Yizheng Xie, Daniel Rebain, David B. Lindell, Andrea Tagliasacchi]
year: 2026
published: "2026-11-09（arXiv v1）"
venue: "arXiv preprint (cs.CV, cross-list cs.GR)"
url: "https://arxiv.org/abs/2611.12716"
code: "（作者声明将完整开源，v1 未见链接）"
project_page: ""
category: [gaussian-splatting, radiance-field, primitive-representation, hair, curve-rendering, inverse-rendering, volume-rendering]
importance: A-
historical_importance: 3
game_relevance: 4
production_readiness: Research
user_level: Normal
status: unread
aliases: [Spaghetti Splatting, 曲线溅射, 曲线狄拉克质量, Curvilinear Splatting, 意面溅射]
tags: [gaussian-splatting, splatting, hair, curve, volume-rendering, inverse-rendering, sfu]
---

# Spaghetti Splatting（Shramko et al. 2026）

## TL;DR

**一个点，是一条长度为零的曲线。**

3DGS 的做法可以被写成一句配方：**把狄拉克质量（点）和一个低频核（高斯）做卷积**，卷积完的场才是可采样、可求导的。这个配方有两个正交的自由度——**核**和**被卷积的东西（支撑）**。

过去四年所有人都在改**核**（高斯 → Beta → 各向异性 Beta → 学生 t）。这篇改**支撑**：把点换成长度可学的**线段**，再把线段连成**折线**。

- 一个高斯卷积的线段，沿光线的积分有**闭式解**（只用 `erf`），不像三角形要算 Owen's T；
- 每段独立体积渲染 ⇒ **接头处不过计数**（因为密度是可加的测度，不是独立的 alpha）；
- 细分一段线段时，**图像空间完全不变**（子段继承父段密度）；
- 点 = 零长曲线 ⇒ **严格包含 3DGS**，任何"核"上的改进可以免费继承。

结果：**泛化场景只赢 0.03–0.16 dB（= 打平）**，但在**毛发**上赢 2.7 dB / F1@2mm +14.3 点，并且**输出是可直接喂物理解算的长连通股**。

## Problem

生产管线的几何基元是**点、线、三角形**——因为硬件为它们做了优化。但从图像里"学"出这些基元很难：**它们是定义在无穷小位置上的狄拉克质量，梯度没有东西可作用**。

二十年前的两个工作给了同一个解法：**卷积一个低频核**（EWA splatting / Poisson 重建），把狄拉克变成体密度，采样就满足了 Nyquist–Shannon。3DGS 用这个思路把"点 + 各向异性高斯"做成了实时可微渲染。

**于是问题变成：这个配方还能往哪扩？**

作者把它写成一行目标函数，答案就浮出来了：

$$\mathcal L_2=\sum_c\left\|I_c-R_c\Big(\sum_i \delta_i \circledast k_i\Big)\right\|^2$$

$\delta_i$ 是第 $i$ 个基元的狄拉克测度，$k_i$ 是它的核。**两个下标，就是两条推广轴**：

| 轴 | 已走到哪 | 谁做过 |
|---|---|---|
| **核** $k_i$ | 高斯 → Beta（紧支撑） → 各向异性 Beta → 重尾/带符号 | 3DGS / DBS / Student Splatting |
| **支撑** $\delta_i$ | 点 → 三角形 → 凸体 → 线 | Triangle Splatting / 3D Convex / Line Primitives（并发） |

点和三角形已经卷得很凶，**曲线这一格是空的**——而曲线恰好是毛发、草、线缆、纤维的天然表示。

## Historical Context

```text
1991  Bloomenthal & Shoemake：Convolution Surfaces
        · 骨架（曲线/多边形）+ 平滑核的等值面 —— 思想源头，但只用来定义隐式面
  ↓
2001  Zwicker（EWA Splatting）：点 ⊛ 椭圆高斯，屏幕空间闭式，可微
  ↓
2006  Kazhdan（Poisson 重建）：点 ⊛ 核 → 指示函数，同一配方的重建版
  ↓
2023  Kerbl（3DGS）：点 ⊛ 各向异性高斯 + 瓦片光栅化 ⇒ 实时
  ↓
2024-25 改【核】的一整批：DBS（Beta）、Student Splatting（重尾）、Mip-Splatting（抗锯齿滤波）
        改【支撑】的一批：Triangle Splatting、3D Convex Splatting、MeshSplatting
  ↓
2026 ★ 本篇：曲线 ⊛ 核 —— 并且指出"点 = 零长曲线"，所以 3DGS 是它的特例
```

**历史位置很特别**：它不是"又一个基元"，而是**第一次把整族方法写成"核 × 支撑"的笛卡尔积**，然后指着空白的那一格说"这里没人"。这种"把领域重写成一张表"的论文，价值往往不在它自己那一格，而在这张表后来被别人拿去填。

## Core Idea

**支撑与核解耦——而支撑的形状决定这个表示能表示什么。**

3DGS 的一个高斯基元可以写成

$$G(x)=\big[\delta(\cdot-\mu)\circledast \mathcal N_\Sigma\big](x)$$

这个写法把**支撑**（$\mu$，一个点）和**核**（$\Sigma$，决定它如何在空间中弥散）**分开了**。一旦分开，"换支撑"就变成一次局部的替换：

- 线段：$m(x)=\int_0^1\delta\big(x-x(u)\big)\,du$，$x(u)=a+uv$，$v=b-a$
- 基元：$P(x)=[m\circledast\mathcal N_\Sigma](x)$
- 当 $a=b$（长度为零）⇒ 退化成 3DGS 的高斯

**这解释了为什么"曲线"是这一格里最值得填的**：曲线是点的一维延拓，退化关系最干净（严格超集），而且核的所有改进都能继承。

## Technical Approach

### 闭式解：为什么线段比三角形好算

密度 $\sigma(x)=\rho\,[m\circledast\mathcal N_\Sigma](x)$，沿光线积出光学厚度：

$$\tau(r)=\rho\int_0^1\!\!\int_{-\infty}^{\infty}\mathcal N_\Sigma\big(r(t)-x(u)\big)\,dt\,du$$

内层是"高斯沿直线的积分"，本身是 $u$ 的一维高斯 $f(u)=f(\bar u)e^{-\kappa^2(u-\bar u)^2}$，其中

$$\kappa^2=\|v\|^2_{Q_\perp}=v^\top Q_\perp v,\qquad \bar u=\frac{v^\top Q_\perp(o-a)}{\kappa^2},\qquad Q=\tfrac12\Sigma^{-1},\quad Q_\perp=Q-\frac{Qdd^\top Q}{d^\top Qd}$$

外层就是对 $u\in[0,1]$ 积一个一维高斯 ⇒ **`erf`**：

$$\tau(r)=\rho\,f(\bar u)\frac{\sqrt\pi}{2\kappa}\Big[\mathrm{erf}(\kappa\bar u)+\mathrm{erf}\big(\kappa(1-\bar u)\big)\Big]$$

> **一处值得单独记的技术不对称**：多边形的高斯卷积要算 **Owen's T**（二维正态在凸多边形上的积分，需要专门表格/近似）；线段只要 `erf`（一维正态的原函数，硬件里有现成实现）。**所以"曲线比三角形更容易可微"不是工程巧合，是维数带来的**——把二维积分降成一维，特殊函数就从 bivariate 掉到 univariate。这条判据可以推广：**选基元时先问"它沿光线的积分要几维特殊函数"**。

### 三大性质的来源：密度是测度，不是不透明度

- **接头不过计数**：每段独立算 $\tau_k$，沿光线按 $\alpha_k=1-e^{-\tau_k}$ 前到后合成。关键在于 $\sigma$ 是**密度**（可加），两段在接头处重叠时 $\int(\sigma_1+\sigma_2)=\int\sigma_1+\int\sigma_2$，所以不会像"两个独立 alpha 叠加"那样算两遍。
- **细分不变**：把一段切成两半，两个子段各自继承父段密度 ⇒ **图像空间完全不变**（附录给了证明）。这条让"分裂"成为一个**无偏的重采样操作**，而不是一次有损的扰动。
- **抗锯齿免费**：Mip-Splatting 的 3D + 2D 滤波器在 EWA 线性化下**只是加到投影核的协方差上**，闭式解照用，**每像素一次采样**。对照：并发的 Line Primitives 不支持空间卷积，只能 4× MSAA。

### 效率：每段线性化一次投影

逐段算精确的射线相关量太慢。做法是 **EWA 在段中点算一次，然后沿段按逆深度缩放**——这让核可以在屏幕空间变化而保持 3D 形状一致，也让**非高斯的紧支撑核（Beta）**同样可用。

### 优化

$$\mathcal L=\mathcal L_{\text{recon}}+\lambda_1\mathcal L_{\text{MCMC}}+\lambda_2\mathcal L_{\text{connect}}$$

- **致密化**：MCMC 重定位（不透明度低于阈值就搬），搬到目标后**长段对半劈、短段克隆**；
- **折线合并**：端点距离小于阈值的折线合并，**`L_connect` 主动把附近端点拉近**（否则合并无源可合）；
- 合并只在重定位停止之后跑（训练最后 5k 迭代）。

## Key Contribution

1. **把整族方法重写成"核 × 支撑"的笛卡尔积**，并指出曲线那一格是空的。
2. **给出曲线狄拉克的可微、收敛、高效实现**：折线逼近（段数增大时收敛到连续曲线的卷积）、闭式 `erf` 积分、每段 EWA 线性化。
3. **一个严格超集**：点 = 零长曲线，所以泛化场景不掉点（实测打平），同时在股状几何上刷新 SOTA。
4. **承诺完整开源**（用于精确对比）。

## Why It Works

### 因为"过计数"从来不是几何问题，是合成方式问题

所有基于点的溅射都怕"同一块空间被多个基元覆盖，alpha 叠加算两遍"。**换成密度（测度）之后这个问题自动消失**：密度在重叠处相加，光学厚度取积分，不透明度只在最后取一次 $1-e^{-\tau}$。

> 这条对做 VFX 的人有直接用处：**粒子系统里"重叠处过亮"的根因也是 alpha 叠加**。想解决它，要么改成"密度累加 + 最后一次转换"，要么接受叠加。**本篇给的是前者的一个可微样本。**

### 因为退化关系干净，所以"换支撑"不牺牲"换核"的自由

任何核的改进（Beta、各向异性、重尾）都能直接搬过来——因为核只作用在 $m$ 上，与 $m$ 的形状无关。**这是"解耦支撑与核"最大的工程收益：两条改进路线不再互相打架。**

### 因为长连通股是"表示的直接产物"，不是"后处理拼出来的"

点云要先做连通性重建（聚类、骨架化）才能拿去做物理解算；折线基元**天生就是连通的**——合并与吸引损失只是让它更长，不需要从零重建拓扑结构。

## Limitations

### 五处冷水（数据自己说的）

1. **泛化场景只是打平，不是赢。** Mip-NeRF 360：Beta 核 28.78 vs DBS 28.71（**+0.07**）；Shelly：高斯核 37.38 vs DBS 37.22（+0.16）；Fuzzy：35.03 vs 35.00（**+0.03**）。摘要里的 "state-of-the-art" 全部落在噪声量级上。**它真正的卖点是"不掉点"（超集性质），不是"更好"。**
2. **两项"结构"机制在 PSNR 上是负收益。** 消融表（三数据集平均）：完整模型 **33.54**；去掉折线合并 **33.59**；去掉吸引损失 **33.57**。**也就是说：让股连起来，要付 0.03–0.05 dB。** 作者写的是 "no effect on reconstruction metrics"——严格说不是 no effect，是**轻微负效果**；真正的收益在**平均连通分量大小 1.00 → 3.50**，而这个指标 PSNR 完全看不见。
3. **帧率不是"持平"。** Mip-NeRF 360：3DGS-MCMC 198 FPS → 高斯核 177 / Beta 核 129。Shelly 464 → 452 / 284。**只有高斯核在 Shelly 上算持平**，Beta 核普遍掉 30–40%。
4. **"可直接做物理仿真"是一个承诺，不是一次实验。** 全文没有任何布料/毛发解算结果、没有时间步、没有稳定性数据。摘要那句 *"directly amenable to physics simulation"* 的支撑只有"股是长且连通的"这一几何事实。**这是本文最需要后续验证的一条，也是游戏侧最关心的那条。**
5. **细纹理要靠"更多短段"堆出来。** 作者自陈：*"high-frequency color requires high-frequency geometry, reproducing fine texture currently requires many short segments."* **这正是 [[Decoupling Geometry and Appearance]] 那页说的耦合账单**——外观还是骑在几何上，只是把"面"换成了"段"。

### 论文自陈的边界

- 没有探索"核在段两端之间插值"（作者说这能推广到锥、三角、梯形，进而到曲面）。
- 真毛发重建仍难，接学习式毛发先验是未来工作。
- 拓扑的自动恢复"需要进一步评估"（*broader claims about automatically recovered topology require further evaluation*）——很克制的一句。
- 用的是 3DGS-MCMC 的种群控制 + DBS 的 Beta 核 + Mip-Splatting 的抗锯齿 + Spherical Voronoi 的外观，**四个部件全部来自别人**，本篇的净新增只有"支撑"这一格。

### 我自己的两处待核

1. **接头的连续性到底有多连续？** 作者声明"细分不变"，但那是**同一段被切开**的情形；两条**不同参数的段**在端点相接时，密度在接头处是两段贡献的直接相加，接缝处的等值面会不会出现"腰"？图 5 的 4/8/16/32 段收敛图显示会（4 段时明显有棱），但**收敛所需的段数与原曲线曲率的关系**没有给。
2. **折线合并的阈值**是"预定义距离"，没有给具体数值，也没有消融它对最终股长的影响。

## Game Development Relevance

- **毛发/毛皮**：这是最直接的一格。卡片（hair card）的生产流程是"美术画 alpha 卡片"，股状表示是"解算 + 着色都更对"的形态。**从多视图直接重建出可解算的股**，等于把"扫描 → 资产"这条管线的最后一段自动化了。
- **草、藤蔓、线缆、蛛网、纤维**：凡是"细长且连通"的东西，点表示都要靠密度堆，段表示天生对。
- **一个反例值得记住**：Line Primitives（Tojo 2026）在 Mip-NeRF 360 上只有 **8.66 dB**——因为它把线画成**恒定屏幕宽度**，无法表达深度。**"支撑是线"不够，支撑还必须带"随深度缩放"。** 这是"支撑的形状"和"支撑的度量"必须一起对的最干净反例。

## Unreal Engine Relevance

- UE 的 Groom（股状毛发）与 Niagara Ribbon 都是"曲线/条带"的一等公民，但**它们来自 DCC 或 groom 资产，不来自重建**。这条路线如果成熟，补的是"资产从哪来"。
- 更现实的一条：**把"股"当渲染基元**这件事在 UE 里已经半成品（spline mesh / ribbon），但**没有可微渲染路径**——所以它只能用于"已有股"，不能用于"从图像学股"。
- 若要落地，最小形状是：一个自定义 primitive（每段一个 AABB + 四个端点参数）+ 一个屏幕空间闭式 `erf` 求值的像素着色器。**难点在排序**：段是长条，屏幕空间重叠关系比高斯更难做正确的深度排序（3DGS 的 per-tile 排序假设"基元投影小"，长段违反这个假设）。

## Technology Evolution

```text
1991  Convolution Surfaces（骨架 ⊛ 核 = 隐式面）
  ↓
2001  EWA Splatting（点 ⊛ 椭圆核，屏幕空间闭式，可微）
  ↓
2006  Poisson 重建（同一配方的重建版）
  ↓
2023  3DGS（点 ⊛ 各向异性高斯 + 瓦片光栅化 = 实时）
  ↓
2024-25  核维：Beta / 学生 t / Mip 滤波；支撑维：三角形 / 凸体 / 网格
  ↓
2026 ★ 曲线（本篇）：点 = 零长曲线 ⇒ 严格超集
  （同一时期还有 Gaussian Haircut / GaussianHair：不改支撑，改"对齐到股方向"）
```

## Relationships

### Based On

- 3DGS（Kerbl 2023）—— 核与光栅化骨架；本篇的"零长退化"直接以它为特例。
- [[Zwicker — Surface Splatting (2001)|EWA Splatting]] —— 屏幕空间闭式 + 每段线性化投影的出处。
- Convolution Surfaces（Bloomenthal & Shoemake 1991）—— "骨架 ⊛ 核"的思想源头。
- DBS（Liu 2025，Beta 核）/ 3DGS-MCMC（Kheradmand 2024）/ Mip-Splatting（Yu 2024）/ Spherical Voronoi（Di Sario 2026）—— 四个直接搬用的部件。

### Extends

- [[Gaussian Splatting]] —— **补上"基元"这一维**：此前本库这一页记录的是排序线、分配线、网格化支线，全部在"点"这个支撑上做文章。
- [[Hair Rendering]] —— 给"从图像重建股"提供了第一个不依赖毛发先验的可微路径（对照 Gaussian Haircut 依赖先验）。

### Related

- [[Decoupling Geometry and Appearance]] —— 本篇的 Limitation 5 是那一页"耦合的代价是 max 不是和"的又一实例：细纹理要更多短段。
- [[Level of Detail]] —— 段数就是这一族的 LOD 旋钮（图 5 的 4/8/16/32）。

### Contrasts

- **Triangle Splatting / 3D Convex / MeshSplatting**：那些改的是"二维/三维的支撑"，需要 Owen's T 或近似；本篇的线段只要 `erf`。**"更容易积"是选择曲线的一个非显然理由。**
- **Line Primitives（Tojo 2026，并发）**：同为线基元，但恒定屏幕宽度 ⇒ Mip-NeRF 360 上 8.66 dB，无法用于一般场景。**支撑对了、度量错了，一样不能用。**
- **Gaussian Haircut**：不改支撑，改"把高斯对齐到股方向"并加毛发先验。**两条路的差别是"先验 vs 结构"。**

### Followed By

- 待观察：① 有人把"核在段两端之间插值"做出来（锥/梯形/曲面）；② 真的把重建出的股拿去做 PBD 解算并报告稳定性。

## Personal Knowledge State

- **机制层 Normal**：卷积核与支撑解耦、密度可加 ⇒ 不过计数；这两条今天要真正吃透。
- **判据层 Normal（今天新增）**：
  1. **"表示 = 支撑 ⊛ 核"这个二分，比"点/线/三角"的分类更有用**——因为它直接告诉你"还差哪一格"。
  2. **过计数是合成方式的问题，不是几何的问题**（密度 vs alpha）。

## Learning Value

**高。** 它是 [[Gaussian Splatting]] 一页的**新维度**（基元轴），也是 [[Hair Rendering]] 的当代入口。更重要的是它给了一个**可复用的思考模板**：把一个领域的所有方法写成"两个正交自由度的笛卡尔积"，然后找空格。

## Visualization

**图 1｜"核 × 支撑"的笛卡尔积（本文的概念核心，一张图说完全文）。** 三行 = 三种核（高斯 / Beta / 各向异性 Beta），三列 = 三种支撑（点 / 线段 / 折线）。

![[Research/Files/Pictures/Shramko 2026 — 图2 核与狄拉克的两个正交轴.png]]

**读图**：左上角那一格（点 + 高斯）就是 3DGS；左中（点 + Beta）就是 DBS。**整张表里所有格子都是合法的可微基元**，本篇填的是中列和右列。注意第三行（各向异性 Beta）——它说明"核的各向异性"与"支撑的延展"是**两件不同的事**：一个扁长的高斯和一个圆柱状的线段，在投影上可能相似，但前者没有拓扑、后者有。**这张表本身就是一篇论文的选题地图。**（出处：Shramko et al. 2026，图 2）

**图 2｜同一个场景，点测度 vs 曲线测度。** 左 = 3DGS 学到的狄拉克（散点，辐条是碎的），中 = 渲染图（两者视觉相同），右 = 本篇学到的狄拉克（连续的股，沿着自行车辐条与花枝走）。

![[Research/Files/Pictures/Shramko 2026 — 图3 点测度与曲线测度对照.png]]

**读图**：**中间那张渲染图是关键**——两种表示渲染出来几乎一样，**差别只在"表示本身"**。左边那些散点是"用一堆点去逼近一条细线"的必然结果（点没有方向、没有连通），右边是"一条线就是一条线"。**这就是"支撑形状"的全部意义：它不改变这一帧的像素，它改变的是这个表示还能被用来干什么**（解算、编辑、LOD）。（出处：Shramko et al. 2026，图 3）

**图 3｜一条曲线用 4/8/16/32 段逼近。** 段数增加时折线的卷积收敛到连续曲线的卷积，接头处的"棱"消失。

![[Research/Files/Pictures/Shramko 2026 — 图5 折线四档细分收敛.png]]

**读图**：看 4 段那张——接头处有明显的折角与"关节"感；16 段之后基本看不出分段。**这张图是"收敛性"的可视化证据，也是这一族的 LOD 旋钮**（段数 ↔ 曲率）。它同时暴露一个未答的问题：**收敛所需的段数与原曲线曲率的关系**，论文只给了四个档位，没给判据。（出处：Shramko et al. 2026，图 5）

## Notes

- 10 页 8 图 1 表。作者群是 SFU / Wayve Labs / UBC / Toronto 的混合（Tagliasacchi 组，3DGS-MCMC、Spherical Voronoi 的同组）。
- **致谢里有一句可爱的**："We thank Francesco Di Sario and Maria Taktasheva for cooking, as well as eating, the pasta."——teaser 那盘意面是真煮的。
- **写作上值得学**：摘要第二句就把整族方法压成一行公式（$\sum_i\delta_i\circledast k_i$），然后立刻指出"两个下标 = 两条轴"。**用公式做分类学**，比用文字列 related work 有力得多。
- 待核：接头处等值面的"腰"、合并阈值的具体值、真做一次 PBD 解算看股会不会炸。
