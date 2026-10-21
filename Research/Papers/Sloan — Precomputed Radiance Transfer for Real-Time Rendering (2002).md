---
type: paper
title: "Precomputed Radiance Transfer for Real-Time Rendering in Dynamic, Low-Frequency Lighting Environments"
authors: ["Peter-Pike Sloan", "Jan Kautz", "John Snyder"]
year: 2002
published: 2002
venue: "ACM Transactions on Graphics 21(3)（SIGGRAPH 2002 proceedings）"
url: "https://www.ppsloan.org/publications/shillum_final23.pdf"
doi: "10.1145/566654.566590"
code: ""
project_page: "https://www.ppsloan.org/publications/"
category: ["rendering", "global-illumination", "precomputed-transfer", "spherical-harmonics", "real-time"]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（思想层：SH 探针 / 辐照度体 / 光照探针是此后所有引擎的标准件；原始的逐顶点 transfer 管线本身已被 DDGI 与 Lumen 式的运行时更新取代）"
user_level: Normal
status: unread
---

# Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)

## TL;DR

**把"光照"与"光的传播"拆开预计算的那篇论文**——此前所有烘焙方案（[[Goral — Modeling the Interaction of Light Between Diffuse Surfaces (1984)|Radiosity]]、Lightmap、辐照度缓存）烘的都是**某套光照下的答案**；PRT 烘的是**"这个物体如何接收任何低频光照"这个算子**。做法三步：① 入射光照用低阶球谐（SH）展开成 $n^2$ 个系数；② 物体表面每点预计算一个 **transfer 向量**（漫反射）或 **transfer 矩阵**（光泽），把"自遮挡 + 自互反射 + 材质"全部烘进系数；③ 运行时**只剩一次内积**：$L'_p=\sum_i (M_p)_i (L_p)_i$（式 8）。

于是**换光照变免费**——不管场景里有多少盏灯、灯有多大，运行时成本恒定。代价写在假设里：**几何必须静态、运动只能刚性、光照必须低频**。25 个系数（4 阶）就够做出带自阴影与自互反射的实时漫反射；129 Hz（2002 年的 Radeon 8500）。

> 一句话对读者：**Lightmap 存"这一版光照长什么样"，PRT 存"这个物体怎么吃光"**——前者改灯要重烘，后者改灯只需重新点积。

![[Research/Files/Pictures/Sloan 2002 — 图1 无阴影辐照度与预计算转移对比.png]]
**读图**：同一个头部模型、同一张环境光。左 = 2001 年 Ramamoorthi-Hanrahan 的无遮挡辐照度（只剩 $\cos$ 卷积，眼眶、鼻底、下颌全无阴影）；右 = 本文的预计算转移（自阴影 + 自互反射都在系数里）。右图在 2002 年的 Radeon 8500 上 **129 Hz**，且换任意低频环境光都不需要重算——这正是"烘算子 vs 烘答案"的可见差别。（图 1，Sloan/Kautz/Snyder 2002）

## Problem

实时渲染要同时吃掉三份复杂度，而 2002 年的技术栈只能吃一份：

1. **BRDF 复杂度**——真实材质的各向异性/光泽反射；
2. **光照积分复杂度**——每个点要对整个半球积分（面光源、环境贴图）；
3. **光传输复杂度**——遮挡、互反射、焦散（光路被中间物体改变）。

当时的三条路线各自只能解决其中一项：

- **多点光源 + 多趟渲染**（accumulation buffer、逐灯 shadow map）：能吃传输（阴影），但**成本 ∝ 光源数与光源尺寸**——大面光源直接爆炸；
- **预滤波环境贴图**（Blinn-Newell、prefiltered env map）：能吃光照积分，但**假设入射辐射无遮挡**——阴影与互反射全部丢失；
- **烘焙**（radiosity / lightmap / irradiance caching）：能吃传输，但**答案与那一套光照绑定**——换灯就重烘。

**本文的取舍**：放弃高频，换取"传输与光照解耦"——**只做低频光照环境，但把阴影、互反射、焦散一并吃进来，且全部可在运行时换灯**。

## Historical Context

```text
1984  Goral 等：Radiosity——"视角无关 ⇒ 可预计算"（烘答案，换灯重解）
  ↓
1986  Kajiya 渲染方程：所有方案成为它的特解（线性性自此是公开事实）
  ↓
1986  Greger 等：Irradiance Volume（把辐照度存进 3D 网格——本文 neighborhood-transfer 的直系前身）
  ↓
1988  Ward 等：辐照度缓存（把"答案"按误差容限缓存起来，按需计算）
  ↓
1994  Nimeroff 等：steerable function basis（天光基，固定视角）
1995  Dobashi 等：SH 基 + 表面转移向量场（但光照变化只允许"既有光源的方向强度"）
1997  Teo 等：PCA 压缩基（非无穷远光源）
2000  Debevec 等：反射场（固定视角）；Heidrich 等：预计算可见性的微几何着色
2001  Malzbender 等：多项式纹理贴图（PTM，2D 微几何的传输多项式）
2001  ★ Ramamoorthi & Hanrahan：SH 辐照度环境贴图——"无遮挡漫反射 = 9 个系数"
  ↓
★ 2002  本篇 PRT：把"可见性 V"塞进同一个 SH 框架，把 9 系数推广到 25 系数 + 矩阵
  ↓
2003  CPCA（Sloan/Hall/Hart/Snyder）：transfer 场聚类主成分压缩——本文 Future Work 第一条兑现
2003  Bi-Scale Radiance Transfer（Sloan/Liu/Shum）：把传输拆成本地细节 + 全局低频
2005  LDPRT（Local, Deformable PRT）：局部可变形——本文"只能刚性运动"的缺口被补
2006-2010s  SH 探针 / 光照探针成为引擎与主机标准件（UE 的 Indirect Lighting Cache、Volumetric Lightmap 皆此族）
  ↓
2019  DDGI：不再预计算"转移"，改为运行时用光追**增量更新辐照度场**——预计算线在此转向
2020s  Lumen / RTXGI：表面缓存 + 探针场 + 时域摊销；PRT 的"预计算转移"退居离线/静态烘焙
```

**两个容易被讲错的地方**：

- PRT **不是** Ramamoorthi-Hanrahan 的替代，而是它的**推广**：后者是 PRT 的 `DU`（unshadowed diffuse）档，且 $M=H_N$ 有解析式、无需存储、9 系数够用——本文原文明确引 [34] 并承认这一点；
- PRT **不是**"第一个用 SH 做传输的"（Dobashi 1995、Westin 1992 都在前），它是**第一个把"任意低频光照 × 自传输"做成可实时换灯的完整系统**——novelty 在"解耦 + 运行时成本恒定"，不在 SH 本身。

## Previous Work

- **场景重光照（relighting）**：Nimeroff 1994 的可操纵基（球面单项式，与 SH 差一个线性变换）、Teo 1997 的 PCA 降基、Debevec 2000 的反射场——共同短板是**固定视角**；本文把 transfer 存在 3D 表面上，因此视角可变；
- **软阴影**：shadow map 家族（Williams 1978 起）假设点光源/局部光源；面光源靠多趟分解（成本 ∝ 光源尺寸）；Soler-Sillion 1998 的 FFT 卷积软阴影成本与光源尺寸无关，但**只能处理预分割的物体簇**，复杂网格自阴影困难；
- **微几何着色**：horizon map、Heidrich 2000 的预计算可见性、Malzbender 2001 的 PTM——都是"2D 微几何纹理"，作用域局部；本文在**3D 物体表面**上直接算自传输，因此能吃到更全局的效应；
- **缓存族**：Ward 1988 缓存标量辐照度、Jensen 1996 光子图。本文原话最准确：**"我们把缓存思想用到实时渲染上，但缓存的是以 SH 光照基参数化的 transfer 函数，而不是标量辐照度。"**——这句话就是 PRT 与 [[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)|辐照度缓存]] 的分界线。

## Core Idea

**核心是一次"线性性套利"：光传输对入射光是线性的，所以"算子"可以独立于"输入"被预计算和压缩。**

```text
传统烘焙：  几何 + 材质 + 光照  →  [求解器]  →  一张答案（radiance / irradiance）
PRT：       几何 + 材质         →  [求解器]  →  一个算子 M（SH 系数场）
            算子 M  ⊙  光照 L   →  [点积]    →  答案（任意低频光照）
```

三个技术支点：

1. **基展开把积分变成内积**：SH 正交归一 ⇒ $\int a(s)b(s)\,ds=\sum_i a_i b_i$（原文式 5）。入射光与 transfer 各自投影一次，运行时只剩点积；
2. **可见性可以进基**：阴影不过是 $M=H_N\cdot V$——$V$ 是 0/1 的可见性函数，投影后就是一组带符号的系数。这一步把"几何遮挡"从运行时挪到了预计算；
3. **旋转不变性消除时域走样**：SH 是旋转不变的（式 4，类比傅里叶的平移不变），因此在旋转的采样点上取光照**不会产生 aliasing / 抖动**——这是本文明确选择 SH 而非球面小波的理由（球面小波能量更集中，但换基会带来时间上的"shading wobble"）。

## Technical Approach

### 1) 三种漫反射 transfer（由简到繁，逐层加物理）

| 档 | transfer 函数 | 吃进了什么 | 需要的系数数 |
|---|---|---|---|
| `DU` 无遮挡 | $M_p(s)=H_{N_p}(s)=\max(N_p\cdot s,0)$ | 只有余弦项 | **9 系数（$l\le2$）**够用，且 $M$ 有解析式、**根本不用存** |
| `DS` 有阴影 | $M_p(s)=H_{N_p}(s)\,V_p(s)$ | 自遮挡（软阴影） | **25 系数（$l\le4$）**——$V$ 会在局部造出高频（自阴影"针孔"） |
| `DI` 含互反射 | $M$ 由传输模拟数值投影得到 | 自阴影 + 自互反射 | 同上（本文实现取 25 系数，原文称 quartic band / $n=5$） |

互反射档的形式值得单独看一眼：$T_{DI}=T_{DS}+\rho_p\int L_p(s)H_{N_p}(s)V_p(s)\,ds$，其中 $L_p$ 是**物体自身上一次反弹的出射辐亮度**。这与 [[Goral — Modeling the Interaction of Light Between Diffuse Surfaces (1984)|Goral 1984]] 的 $B=E+\rho H$ 是同一个能量账，只是**逐顶点、逐 SH 系数地记账**。

### 2) 光泽：向量升级为矩阵

漫反射的输出是标量（视角无关），光泽的输出是"整个球面上的转移后辐射亮度"，因此每点需要一个**矩阵**：

$$(L'_p)_i=\sum_j M_{pij}\,(L_p)_j \tag{9}$$

- 矩阵把 $n^2$ 个光照系数变成 $n^2$ 个"转移后辐射亮度"的系数；再与 BRDF 核（圆对称 Phong 瓣）做 SH 卷积（式 6：卷积 = 系数逐项相乘），最后在**视线相关的反射方向 R** 上求值；
- 允许**非方阵**（如 25×9）：低频光照 → 更高频的转移后辐射亮度（光泽越强，需要的输出阶数越高）；本文用了 25×25 与 9×25。

### 3) 预计算：把 SH 基函数当"无穷远光源"逐个打光

- 在每点生成 **10k–30k 条准随机方向**；用**二十面体 1→2 二分**（球面上 1→4 细分得不到等面积三角形）细分 6–8 层 → 512–2048 个等面积层级 bin；
- **第一趟（shadow pass）**：沿法线半球打阴影线，给每个方向打"是否被自身遮挡"的 bit，bin 也继承该 bit；
- **后续趟（interreflection pass）**：只遍历被遮挡的 bin，射线打到自身另一点 q 就取 q 上一趟的出射辐射亮度继续累加，**直到该趟总能量低于阈值**；
- 镜面面片不记录 transfer，射线在其上直接反射继续传播 → 路径形如 $L[S]^*D[S]^*p$，于是**焦散也能进 transfer**（图 9 的环状焦散照地）。

### 4) 运行时：三步 + 一次点积

1. 在物体附近一个或数个采样点**动态采样入射辐射亮度**（立方图 6 面 × 16×16，2× 超采样后 box 降采样），投影到 SH；
2. 把光照系数**旋转**到物体坐标系（每物体一次，不是每点一次），必要时按 ICP 预计算的点集做混合，得到物体上的光照场；
3. 每顶点/每像素做**内积**（漫反射）或**矩阵乘**（光泽）。

硬件细节（2002 年）：DX8.1（ps1.4）像素着色器 **8 条指令**完成点积；因为系数是带符号且常超出 $[-1,1]$，必须用支持 $[-8,8]$ 的寄存器范围；RGB 三通道色渗需要**三趟**。光泽的矩阵乘在 CPU 上做（当时的 shader 装不下），这是它帧率低的唯一原因。

## Key Contribution

1. **"烘算子而不是烘答案"**——把光照与传输彻底解耦，运行时成本**与光源数量、光源尺寸无关**（对比多趟渲染的成本 ∝ 灯数）；
2. **把可见性纳入同一个线性基**：软阴影、互反射、焦散都变成"预先算好的系数"，而不是运行时的额外 pass；
3. **给出三档 transfer 的完整价目**（DU/DS/DI），并量化了每档需要的阶数（9 vs 25）——这是最早的"质量-预算"刻度表之一；
4. **neighborhood transfer**：把 transfer 存到物体周围的 3D 空间里，任意接收者（事先未知）都能接收它的软阴影/焦散/反射——**Greger 1986 的 irradiance volume** 的"可换灯 + 可光泽"推广；
5. **验证了旋转不变性对时域稳定性的价值**——为"为什么引擎选 SH 而不选球面小波"提供了第一条工程论据。

## Why It Works

- **线性性 + 正交基**：传输算子对入射光是线性的，SH 正交归一 ⇒ 积分内积化。这不是近似，是恒等式（在带限假设下）；
- **低频假设是"预算的来源"**：$n$ 阶只需 $n^2$ 个系数（2 阶 9 个、4 阶 25 个），因为漫反射物体本身就是入射光的**低通滤波器**（余弦核），而 $V$ 只在局部抬高频率——所以"有阴影"比"无阴影"贵 2 阶，这是物理决定的，不是实现偷懒；
- **成本从"每帧每灯"搬到"一次预计算"**：预计算是 $O(\text{顶点数}\times\text{方向数}\times\text{反弹次数})$，运行期是 $O(\text{顶点数}\times n^2)$——两个数量级完全不同的账本；
- **旋转不变性**：SH 系数在旋转下按阶线性变换，因此"物体转、光不转"只需转少量光照系数（避免每帧转整片 transfer 场），且不产生采样走样。

## 成本三本账（本文给出的可迁移预算语言）

| 账本 | 2002 年实测 | 今天的等价物 |
|---|---|---|
| **预计算** | 50k 顶点 / 1.1 h（head）；100k / 2.4 h（tyra）；150k / 4.4 h（teapot）；32³ 体积 / 15 min（cloud） | 离线烘焙机时；PRT 的预计算量与"顶点数 × 方向数 × 反弹数"成正比 |
| **存储** | 每顶点 $n^2$ 个带符号系数（25 阶 ≈ 100 B/顶点；50k 顶点 ≈ 5 MB——**本行为按系数数的推算，非原文实测**）；光泽是 $n^2\times n^2$（25×25 = 625 系数/顶点） | 这就是后来 CPCA 压缩（2003）要解决的问题——原文把"transfer 场压缩"明确列为**未解决** |
| **运行时** | 漫反射 129 Hz（50k 顶点）；光泽矩阵乘在 CPU 上 → 1.7–3.6 Hz（全动态）/ 16 Hz（定光）/ 125 Hz（定视角）；体积 40 Hz | "质量档 = SH 阶数"；"帧率档 = 矩阵乘放 CPU 还是 GPU" |

> 对做预算的人，这张表最有用的一句是：**PRT 把成本从"每帧"挪到了"每次资产变更"**——只要几何/材质不变，光照怎么改都不花钱；一旦几何变了，整本账重来。这正是 [[Goral — Modeling the Interaction of Light Between Diffuse Surfaces (1984)|Goral 1984]] 那张"三档失效价目表"（改几何 / 改材质 / 改灯光）在 2002 年的精确化：**灯光档被彻底免费化，几何档被彻底锁死**。

## Limitations

- **材质被烘进互反射**：含互反射的两档（漫反射 `DI` / 光泽 `GI`）里，反照率与光泽度必须在预计算时定死，运行时不可改（`DS`/`GS` 档例外——不含互反射，允许运行时改材质）；
- **只能刚性运动**：物体可整体平移/旋转，不能形变、不能部件间相对运动（角色动画因此出局；本文 Future Work 提出"把 transfer 按形变参数化"，2005 年 LDPRT 才补上）；
- **凸包入侵即错**：遮挡物或光源进入物体凸包内部会产生错误（transfer 假设外部环境是"无穷远"的）；
- **互反射依赖"光照在物体上近似均匀"**：`DI` 档必须假设 $L_q\approx L_p$，否则互反射项不准（`DS` 档不受此限——它只依赖 p 点的入射光）；
- **低频硬约束**：高频光照（小光源、锐利阴影）会被 SH 带限抹平，产生 **ringing/Gibbs**（简单模型上明显，复杂网格上被掩盖）；本文用 windowing 缓解；
- **transfer 场未压缩**：原文自认这是"important but unaddressed"——这正是 2003 年 CPCA 的题目。

## Game Development Relevance

- **这是所有"光照探针"的法理起点**。今天引擎里的 SH 探针、Volumetric Lightmap、Indirect Lighting Cache，本质都是 PRT 的**退化版**：只保留"辐照度"（0 阶/2 阶 SH），丢掉"带可见性的 transfer"——因为存储与预计算太贵，而静态场景的可见性变化少。理解了 PRT，就理解了"为什么探针只能给低频、为什么漏光"；
- **给"档位"一个物理刻度**：`DU` 9 系数 / `DS` 25 系数 / `GI` 25×25——**阴影的代价是"多两个阶"、光泽的代价是"平方"**。这与 [[Scalability and Quality Tiers]] 的档位语言直接同构；
- **给"动态灯光预算"一条历史注脚**：PRT 证明了"如果几何静态，光可以随便加"；反过来，一旦几何/材质要动，就要回到逐帧方案（DDGI/Lumen）。**"能不能预计算"不是技术选择，而是"几何动不动"的产品选择**；
- **VFX 侧不适用但可借语言**：粒子/特效是高频、半透明、逐帧变化的，PRT 的低频假设全破；但"把不变量烘进资产、把变量留到运行时"这条原则，与 Niagara 的"预烘焙曲线/预积分贴图 vs 运行时参数"完全同构（参见 [[Real-Time VFX Performance Budgeting]]）。

## Unreal Engine Relevance

- **Lightmass 存的是"答案"**：Lightmap 是某一套光照下的 radiance，改灯要重烘——这正是 PRT 要解决的痛点，也是它没被引擎采用的原因（引擎要几何/材质可变）；
- **Volumetric Lightmap / ILC 存的是"SH 辐照度"**：这是 PRT `DU` 档的工程化后代（9 系数、无可见性 transfer），所以"移动物体在静态场景里拿到间接光"能用，"动态物体自遮挡"不能；
- **Lumen 走的是反方向**：不预计算传输，而是每帧用表面缓存 + 探针场重新算——**用算力换自由度**。PRT 与 Lumen 是同一权衡轴的两端，中间是 DDGI；
- **UE 里能直接摸到 PRT 思想的地方**：材质里的 "Precomputed AO / Static Switch"、Lightmass 的 "Ambient Occlusion" 烘焙、以及 `Indirect Lighting Cache` 的 SH 采样——都是"把几何项烘进资产"的同一族；
- 若要在 UE 里复现 PRT 实验：Niagara 或材质里手动做 SH9 点积 + 自制的逐顶点 transfer 贴图即可（**仅教学价值**，生产价值已被 Lumen 取代）。

## Technology Evolution

```text
问题：间接光要"全场景积分"，实时算不动
  ↓ 解法一：把答案烘起来（1984 Radiosity / 1988 辐照度缓存 / 1990s Lightmap）
      → 视角自由了，光照被锁死
  ↓ 解法二：把算子烘起来（2002 PRT）
      → 光照自由了，几何被锁死
  ↓ 解法三：两者都烘，按需更新（2019 DDGI 探针场 / 2020s RTXGI）
      → 几何可动、光照可动，代价是噪声与时间滞后
  ↓ 解法四：不烘，每帧重算（Lumen 表面缓存 / 硬件光追）
      → 自由度最大，代价是算力与稳定性
  ↓ 解法五：让网络学"间接光长什么样"（Neural GI，见 [[Neural Global Illumination]]）
```

**每一代都在换"哪个变量被冻结"**：视角 → 光照 → 几何 → 全不冻结（但引入噪声/延迟）。这是 [[Global Illumination]] 谱系表的第 4 行，也是理解 Lumen 为何必须做"时域摊销"的历史前提。

## Relationships

### Based On

- [[Rendering Equation]] —— 线性性是全部推导的前提；PRT 是它在"低频 + 静态几何"约束下的可计算特解；
- [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]] —— SH 辐照度的解析解；本文的 `DU` 档就是它（$M=H_N$，9 系数，无需存储）。**PRT 相对它的增量只有两项：可见性 $V$ 与互反射项**；
- [[Spherical Harmonics]] —— 投影/重建/卷积/乘积/旋转五条性质，本文第 3 节逐条用到（式 1–7）。

### Extends

- [[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)]] —— 从"缓存标量辐照度"升级为"缓存以 SH 光照基参数化的 transfer 函数"（原文 Related Work 原话）；
- [[Goral — Modeling the Interaction of Light Between Diffuse Surfaces (1984)]] —— 把 $B=E+\rho H$ 的全局矩阵解，改写成逐顶点、逐 SH 系数的算子；"改灯光只换右端项"在这里被推到极致（右端项可以每帧换）。

### Related

- [[Keller — Instant Radiosity (1997)]] / [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]] —— 缓存族"光侧"：存的是"光在哪里"，PRT 存的是"表面怎么吃光"；三者构成缓存族的三个坐标轴（光侧 / 接收侧 / 算子侧）；
- [[Majercik — Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields (2019)]] —— 同一目标（探针场间接光）的"运行时更新"路线；DDGI 的探针里存的正是 SH 系数，但**没有 transfer**；
- [[Split-Sum Approximation]] —— 同为"把积分拆成可预计算的两半"，但拆的是 BRDF × 环境光，不涉及可见性；
- [[Shadow Mapping]] —— 阴影的"逐帧路线"对照：PRT 把阴影烘进系数，shadow map 每帧重算；前者静态几何、后者任意几何。

### Followed By

- **CPCA（Sloan, Hall, Hart, Snyder 2003, ACM TOG 22(3)）**——transfer 场聚类 + 主成分压缩，直接兑现本文"压缩未解决"的缺口（本库记名，未入库）；
- **Bi-Scale Radiance Transfer（Sloan, Liu, Shum 2003）**——传输拆成本地细节 + 全局低频（本库记名）；
- **LDPRT（Local, Deformable PRT, 2005）**——把局部 transfer 绑到可变形模型上，兑现"角色动画"缺口（本库记名）；
- **SH 探针 / 辐照度体（2006–2010s 引擎标准件）**——UE 的 ILC / Volumetric Lightmap、Unity 的 Light Probes 皆此族；
- [[Majercik — Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields (2019)|DDGI]] —— 用"运行时更新"取代"预计算转移"，宣告 PRT 路线在动态场景中退场。

## Personal Knowledge State

- `user_level: Normal`（**结论层 ≈30 分钟**）——不需要跟推导，只需拿走四件事：
  1. **"烘算子 vs 烘答案"**：Lightmap 烘答案（换灯重烘），PRT 烘算子（换灯只点积）；
  2. **三档价目**：无阴影 9 系数 / 有阴影 25 系数 / 光泽 25×25——**阴影 = +2 阶，光泽 = 平方**；
  3. **三条锁**：几何静态、运动刚性、光照低频——每一条都对应今天某个引擎方案的取舍；
  4. **成本搬家**：从"每帧"搬到"每次资产变更"。
- 与你的工作直接相关的判断：**"烘焙还是实时"本质是"几何动不动"的产品决策**，不是渲染器的技术偏好——这句话可以直接用在 NGR 的画质档位讨论里。

## Learning Value

- **Normal → Easy 的验收标准**（能复述即到位）：能对着 [[Global Illumination]] 谱系表解释"为什么第 4 行必然出现在第 3 行之后、第 6 行之前"——Radiosity 把答案烘死，PRT 把算子烘活，缓存族则把"答案"做成可增量更新的；三者是同一权衡轴上的三个点；
- **可做的 15 分钟检验**：打开 UE 的 Volumetric Lightmap 可视化，找一处"移动物体进入静态阴影区"的位置，观察它拿到的间接光**没有**该处的自遮挡——这就是"探针只有辐照度、没有 transfer"的直接后果；
- **不建议**：复现 PRT 管线（工具链已废弃），或推 SH 旋转公式（工程上直接用现成库）。

## Visualization

![[Research/Files/Pictures/Sloan 2002 — 图2 自转移运行期总览.png]]
**读图**：上排（漫反射）——环境光被投影成 SH 系数（红正蓝负），乘以表面上的 **transfer 向量场**，得到最终着色；每个 transfer 向量已经把"该点如何被自己遮挡/被自己照亮"编码进去了。下排（光泽）——每点换成 **transfer 矩阵**，把光照系数变成"转移后辐射亮度"的系数，再与环境 BRDF 核卷积、在反射方向 R 上求值。**这张图是"算子"二字最直白的图解**：左边是输入，中间是预计算好的算子，右边是点积出来的结果。（图 2，Sloan/Kautz/Snyder 2002）

![[Research/Files/Pictures/Sloan 2002 — 图10 SH阶数与阴影质量扫描.png]]
**读图**：横轴是 SH 带数 $n$（系数个数 $=n^2$，从 $n=2$ 的 4 个到 $n=26$ 的 676 个），纵轴是光源角半径（0° / 20° / 40°）。三件事一眼可见：① **光源越小越吃阶数**（0° 行到 $n=6$ 才收得住阴影边缘）；② $n=26$ 出现明显 **ringing**（右侧黑框里的振铃），windowing 后缓解（g 列）；③ 40° 大光源用 $n=4$（16 系数）就很接近光追参考（h 列）。**这就是"档位 = 阶数"的原始刻度表**——本文全篇结果图取 $n=5$（25 系数，d 列，带星号）。（图 10，Sloan/Kautz/Snyder 2002）

![[Research/Files/Pictures/Sloan 2002 — 图11 漫反射与光泽的自转移四格.png]]
**读图**：同一尊佛像，从左到右——漫反射无阴影 / 漫反射含互反射 / 光泽无阴影 / 光泽含互反射。注意第 2 格：腹部与手臂下方的**自互反射**（暖色渗入）是 Radiosity 那一类的效果，但这里换灯不需要重解；第 4 格是矩阵 transfer 的代价：预计算 2.5 h、全动态只有 3.6 Hz。（图 11，Sloan/Kautz/Snyder 2002）

> 本文的视觉材料全部为论文原图（图 1/2/10/11），版权归原作者，仅作学习引用。

## Notes

- **与 [[Lightmass in the Rendering Knowledge Map|读者 Lightmass 专题]] 的接口**：专题的四层区分（问题 / 方法 / 系统 / 数据）里，PRT 正好卡在"方法"与"数据"之间——它提出的是**数据形态的变更**（存算子而非存答案），系统层几乎没有对应物（引擎没走这条路）。这是"方法赢了、系统没跟"的典型案例；
- **为什么引擎最终没采用 PRT**：三条锁里"几何静态"与"材质可改"在游戏里都不可接受（角色要动、材质要调）；而它省下的那点运行时，在 GPU 变快后被 Lumen 式的"每帧重算"直接买回来了——**这是"预计算 vs 算力"价格比在 20 年间翻转的标本**；
- **与 [[Goral — Modeling the Interaction of Light Between Diffuse Surfaces (1984)|Goral 1984]] 的对读价值**：两篇相隔 18 年，回答的是同一个问题（"间接光能不能预计算"），答案从"烘答案"变成"烘算子"——中间那一步"视角无关"的洞察是共享的；
- **一处需要留意的史实**：本文的 "neighborhood transfer" 与 Greger 1986 的 irradiance volume 是**推广关系而非独立发明**，原文在 Related Work 与 §8 两处都明确致谢了这条线；写技术史时不要把 neighborhood transfer 说成"首个空间传输场"。
