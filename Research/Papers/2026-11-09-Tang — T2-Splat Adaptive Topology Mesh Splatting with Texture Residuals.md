---
type: paper
title: "T2-Splat: Adaptive Topology Mesh Splatting with Texture Residuals"
authors: [Zhihao Tang, Youjia Zhang, Mingbo Zhao, Wei Yang]
year: 2026
published: "2026-11-04（arXiv v1）；NeurIPS 2026"
venue: "NeurIPS 2026（40th Conference on Neural Information Processing Systems）"
url: "https://arxiv.org/abs/2611.06974"
code: ""
project_page: "https://t2-splat.github.io/"
category: [mesh-splatting, neural-rendering, lod, mesh-simplification, texture, game-engine]
importance: A-
historical_importance: 2
game_relevance: 5
production_readiness: Prototype
user_level: Normal
status: unread
aliases: [T2-Splat, T2 Splat, Adaptive Topology Mesh Splatting, 自适应拓扑网格溅射]
tags: [gaussian-splatting, mesh-splatting, lod, mesh-simplification, texture, neural-rendering]
---

# T2-Splat（Tang et al. 2026）

## TL;DR

**一个三角形同时干两份活，是"过度细分"的唯一原因。**

三角形在网格溅射里既要**撑起几何**，又要**承载外观**（顶点 SH 插值）。这两件事的粒度需求完全不同：平地上不需要密的面，但高频花纹需要。于是系统只能取两者的最大值——**平面区域被花纹的需求拖着一起细分**。

T2-Splat 的做法是把这两份活分给两个东西：

| | 归谁 | 用什么动 |
|---|---|---|
| **几何**（面在哪、有多少） | 网格拓扑 | 边塌陷 + 顶点分裂（[[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes\|渐进网格]]的两个算子） |
| **外观**（每个像素什么颜色） | 每面残差 Mip 纹理 | 冻结拓扑后单独优化 |

结果不是"稍微省一点"，是**面数降到 1/3 而质量更高**：

| Garden 场景（MacBook Air M3，OpenGL 1920×1080） | 面数 | 帧率 | 文件 |
|---|---|---|---|
| MeshSplatting | 7.3 M | 126 FPS | 152 MB |
| **T2-Splat** | **2.7 M** | **236 FPS** | **62 MB** |

Mip-NeRF 360 平均：PSNR **25.49 vs 24.78**（+0.71 dB），LPIPS **0.234 vs 0.310**，顶点数 2 M vs 3 M。

> **这是本库第一次有一篇论文把"外观"从"几何载体"上摘下来，并且证明摘下来之后几何可以自由简化。** 昨天 [[Garland 1997 — Surface Simplification Using Quadric Error Metrics|Garland 1997]] 的 QEM 说"属性没做"，[[Hoppe 1999 — New Quadric Metric for Simplifying Meshes with Appearance Attributes|Hoppe 1999]] 说"属性要挂回几何上量"，今天这篇说"**别挂了，换个地方放**"。

## Problem

3DGS 的高斯是无结构的点基元，渲染依赖**可见性排序 + alpha 混合**，这与标准图形管线的深度缓冲、硬件遮挡剔除、基于网格的资产处理**天然不对齐**——论文开篇直接点了游戏引擎、AR/VR 的部署困难。

于是有一条"网格化"的路线（2DGS / GOF / RaDe-GS / SuGaR / MiLo），但它们的做法是**先优化辐射场、再抽网格**（TSDF 或 Poisson）。两阶段的问题：**抽取阶段与优化阶段解耦，细节在转换时丢**。

更近的一步是**直接优化不透明三角形**（Triangle Splatting / MeshSplatting），端到端、有连接性、能出 GLB。但论文指出它们有 **Topological Rigidity（拓扑刚性）**：

> MeshSplatting 用 Restricted Delaunay Triangulation 建立连接后，**拓扑就冻住了**，不能按场景复杂度自适应。

冻住之后只剩一条路：**一开始就把面撒够**。这就是"过度细分"的工业来源。

## Historical Context

```text
1996  Hoppe：渐进网格（vsplit / ecol）——为"连续 LOD"发明的两个算子
  ↓
1997  Garland & Heckbert：QEM 给 ecol 一个可加的代价（10 个数 + 一次加法）
  ↓
1999  Hoppe：属性怎么进 QEM（几何投影 + wedge）
        · 他自己 20 年后在主页写："the present trend is to replace per-vertex
          attributes by texture atlases" —— 属性离开顶点，这条路他当时就看见了
  ↓
2023  3DGS：可微 splatting，但没有拓扑
  ↓
2025  MeshSplatting：不透明三角形 + RDT 连接 ⇒ 有拓扑，但拓扑冻结
  ↓
2026 ★ T2-Splat：把 1996 的两个算子接回可微管线 + 把属性搬到每面纹理
        · 拓扑按几何复杂度走，外观按纹理分辨率走 —— 两条路各自独立
```

> **这条线绕了 27 年回到原点是假象。** 1996 年的 vsplit/ecol 是为**离线简化**发明的（一次跑完，产物是 LOD 链）；T2-Splat 把它们放进**可微优化循环里**，每 N 步执行一次。同一个算子，两种执行语境——**这才是"接回来"的真实含义**（见 [[Mesh Simplification]] 中"决定频率"那一栏）。

## Core Idea

**把"面该有多少"这个决定，从"外观需要多少细节"里解放出来。**

具体是三条互相支撑的设计：

**① 先有合法拓扑，再谈优化。** Safe RDT 在 RDT 候选面上加一个**约束面选择**：按"到输入三角汤的距离 / 无符号法向一致性 / 三角形质量"排序，沿边前沿贪心生长，保证三条不变量——**无重复面、每条边至多两个邻面、每个顶点的面构成单一连通扇形**。关键是它**只改连接与绕向**，顶点位置、外观、不透明度、优化器状态**全部保留**。开放边界与多连通分量都允许。

**② 拓扑用几何判据改，外观不参与。** 边塌陷用标准 QEM（`K_f = p_f p_fᵀ`，顶点 quadric 累加邻面 + 开放边界的虚拟平面）排优先级；顶点分裂用**透视补偿后的梯度分数**挑位置。两者都是**几何/光度梯度驱动**，不看纹理。

**③ 外观在冻结的拓扑上补。** 每面一张零初始化的残差纹理，mip 金字塔，三线性采样。**零初始化**是关键：Stage I 结束时残差为 0，Stage II 从"与 Stage I 完全一致"的状态开始，不会扰动已优化的几何。

## Technical Approach

### 不透明三角形与 overdraw 归零

沿用 MeshSplatting 的定义：每顶点存位置 $v_i$、不透明度 logit $a_i$、SH 系数 $c_i$。为了让三角形光栅化可微，离散指示函数被换成软窗：

$$I(p)=\Big(\mathrm{ReLU}\big(\phi(p)/\phi(s)\big)\Big)^{\sigma}$$

$\phi$ 是 2D 三角形的有符号距离（内负外正），$s$ 是内心（保证中心为 1、边缘为 0），$\sigma$ 是平滑度。不透明度重参数化为 $\alpha_i=\alpha_{\text{floor}}+(1-\alpha_{\text{floor}})\cdot\mathrm{sigm}(a_i)$。

训练时 $\sigma$ 从 1.0 **线性退火到 0.0001**。收敛后 $C(p,d)=c_{T_n}(p,d)\,I(p)$——**每像素只求值一次，overdraw 归零**。这是它能在 GLB 管线上跑到 236 FPS 的根。

### 边塌陷：QEM + 两项外观惩罚

合并后的 quadric 是 $Q_{uv}=Q_u+Q_v$，最优位置解齐次系统；奇异时回退到 $\{u,\ v,\ (u+v)/2\}$ 三个离散候选里挑最优。

**代价函数里最有信息量的是第一项的归一化**：

$$\mathrm{Cost}(e)=w_q\frac{\bar v_h^{*\top}Q_{uv}\bar v_h^*}{\mathrm{median}(E_q)+\epsilon}+w_c\big\|c_u^{dc}-c_v^{dc}\big\|^2+w_a|\alpha_u-\alpha_v|$$

分母是**当前所有候选边 QEM 误差的中位数**。这是一个很实用的工程决定：QEM 的量纲随场景尺度变化（一栋楼 vs 一把刀差三个数量级），**不归一化就没法给外观项定权重**。昨天 [[Level of Detail]] 里"世界空间误差不可比"那条判断，这里是它的一个直接工程后果。

后两项是**在代价里给"跨外观不连续"收税**：基础色差、不透明度差。**注意这不是约束，是罚项**——所以它只是"不鼓励"，不是"禁止"。

拓扑合法性用完整 **link condition**：$\mathrm{Lk}_M(u)\cap\mathrm{Lk}_M(v)=\mathrm{Lk}_M(\{u,v\})$，另外拒绝：边界顶点经内部边相连、非法顶点扇形、局部面退化或绕向翻转。同一轮内接受的塌陷**受影响 one-ring 互不相交**（保证可并行）。

属性合并用不透明度加权平均：
$$c_{\text{new}}=\frac{\alpha_u c_u+\alpha_v c_v}{\alpha_u+\alpha_v+\epsilon}$$

> **这一行就是 [[Hoppe 1999 — New Quadric Metric for Simplifying Meshes with Appearance Attributes|Hoppe 1999]] 批评的"拍权重"**。它靠上面的 $w_c$ 罚项兜底。**这是本篇最值得盯的一处妥协**：几何判据是 1996/1997 年的，外观判据是手调的。

### 顶点分裂：透视补偿的梯度分数

$$\bar g_i^c=\frac{1}{N_i}\sum_{k=1}^{N_i}\big\|\nabla_{v_i}\mathcal{L}\big\|\cdot d_{i,k}$$

梯度模长乘**相机空间深度** $d_i$——远处的顶点梯度天然小，不补偿就永远轮不到它。新顶点插在**最长邻边中点**，原三角形 $(i,j,k)$ 拆成 $(i,v_{\text{new}},k)$ 与 $(v_{\text{new}},j,k)$，属性线性插值。

### 优化器状态缝合（这篇最容易被忽略、也最值得抄的一段）

改拓扑的常见翻车方式是：几何变了、Adam 的一阶/二阶矩还是旧的 ⇒ **优化动量指向一个已经不存在的形状**。

- **塌陷**：合并顶点按 incident-area 加权继承两端点的 $m$ 与 $v$。
- **分裂**：保留的父顶点与插入的子顶点**共享二阶矩**，一阶矩加**相反扰动** $\Delta m=10^{-3}\sqrt{v_k}\odot z,\ z\sim\mathcal N(0,I)$——意思是"从这里开始，你们两个往相反方向探索"。

**这是把"拓扑编辑"与"优化器状态"当成一个整体来维护。** 库内此前的 LOD / 简化论文全部在离线语境，没有这个问题。

### 残差纹理

每面一张 $R\times R\times 3$（4×4 或 8×8），**零初始化**。像素在面内的局部坐标直接取重心坐标：$u=w_j,\ v=w_k$。为了让方形纹理域被充分利用且边界插值稳定，用 **mirror mapping**：$u+v>1$ 时反射为 $T_f(1-v,1-u)$。

mip 由平均池化生成，层级 $\lambda$ 从屏幕空间导数估计：
$$\lambda=\log_2\Big(\max\Big(\sqrt{(\partial u/\partial x)^2+(\partial v/\partial x)^2},\ \sqrt{(\partial u/\partial y)^2+(\partial v/\partial y)^2}\Big)+\epsilon\Big)$$
再在相邻两层间三线性插值。**这等价于一个动态低通滤波器**：远处自动降频，近处保留高频。

### 两阶段

Stage I（30,000 步）优化位置 / 不透明度 / SH，损失 $\mathcal L=\mathcal L_{\text{3DGS}}+\beta_o\mathcal L_o+\beta_z\mathcal L_z+\beta_n\mathcal L_n+\beta_d\mathcal L_d$（不透明度正则 + 顶点深度对齐 + 法向一致性 + Depth Anything v2 单目深度先验）。
Stage II（5,000 步）**冻结拓扑、位置、不透明度**，只优化残差纹理与 SH；纹理损失含 TV（平滑相邻纹素）与 L2（约束残差幅度，让它"只补高频"）。

## Key Contribution

1. **证明"拓扑适应"与"外观精化"可以解耦**，且解耦后两边都变好：拓扑用几何判据自由改，外观用纹理分辨率独立保证。
2. **把渐进网格的两个算子接进可微渲染**，并解决了一个此前没人写清楚的工程问题——**优化器状态的缝合**。
3. **面预算成为可调旋钮**：通过控制"拓扑操作的相对速率"调节质量/面数权衡。

## Why It Works

**因为"面"和"纹理"的分辨率上限不一样。**

- 面的分辨率上限受**几何复杂度**约束。平面上再多面也不带来几何信息，只带来冗余。
- 纹理的分辨率上限受**纹素密度**约束，与面数无关（只要面还在纹理覆盖范围内）。

旧方案让两者共用同一个分辨率（顶点密度），于是只能取 max。拆开之后各取所需，**总预算自然下降**——这不是优化，是**去掉了一个本来就不该存在的约束**。

一个佐证在消融表里：面数从 baseline 的 5.3 M 一路降到 2.5 M（-53%），PSNR 反而从 24.66 涨到 25.49（+0.83 dB）。**"面少了质量反而高"只在"面本来就不是瓶颈"时成立**——这正是过度细分的定义。

## Limitations

**① 几何精度不是强项。** DTU 上 Chamfer mean **0.68**，论文自陈"joint second-best"，而 **Gaussian Wrapping 是 0.49**（明显更好）。15 个场景里只有 3 个最优（63 / 83 / 105）。**它的目标是渲染质量，不是表面重建精度。**

**② 训练时间涨了 54%**（9 场景平均 54 → 83 分钟），其中 Stage II 的 8×8 纹理占大头（72 → 83）。这还没算 MeshSplatting 自己的训练时间——论文**没有给训练时间的横向对比**，只给了推理 FPS。**"快"说的是渲染，不是训练。**

**③ 属性合并仍是加权平均。** 式 (10) 是不透明度加权，跨 UV 接缝 / 材质边界靠式 (8) 的 $w_c$ 罚项"不鼓励"。**罚项有限，就存在一个"卖一点点属性换很多面数"的汇率。** 这是与 [[Hoppe 1999 — New Quadric Metric for Simplifying Meshes with Appearance Attributes|Hoppe 1999]] 的 wedge 结构最直接的分歧：**Hoppe 用数据结构把不连续处变成"不可平均"，T2-Splat 用损失函数把不连续处变成"平均起来贵"。**

**④ 62 MB 这个数字对不上纹理预算。** 按 8×8×3 逐面估算，2.7 M 面需要约 **518 MB** 未压缩纹素，与 62 MB 差一个数量级。可能的解释：6 GB 级的是压缩格式、或只有部分面启用纹理、或 62 MB 不含纹理。**论文没说清楚，这是一处必须自己核的地方**（见"未报数字"）。

**⑤ 每面纹理天然不连续。** mirror mapping 与 TV loss 都是**补丁**：相邻两面各自采样自己的纹理，理论上可以在共享边上取到不同值。面数越低，这个缝越容易看见。

**⑥ 单目深度先验作为监督。** Depth Anything v2 的尺度/偏移归一化之后进损失——**先验的偏差会以"几何偏差"的形式固化**，且这部分不在 QEM 的视野里。

## Game Development Relevance

**这是本库目前最接近"能进游戏管线"的一篇 GS 系论文**，理由不是质量，是**产物形态**：

- 直接导出 **PLY / GLB**，不透明三角形，**overdraw 归零** ⇒ 深度预pass、硬件遮挡剔除、阴影贴图全部可用；
- 2.7 M 面 / 62 MB 在开放世界资产的可接受区间（对比 Nanite 簇的粒度）；
- 面预算可调 ⇒ **同一份资产能出多个档次**，这是 LOD 链的原料（[[Level of Detail]]）。

**工程上真正该抄的是两处，而不是方法本身**：
1. **QEM 误差用中位数归一化**再和外观项加权——任何"把不同量纲的项加在一起"的代价函数都该这么做。
2. **优化器状态缝合**——凡是在优化中途改拓扑/改元素数量的系统（碎裂预处理、动态 LOD、运行时合批）都会遇到，而绝大多数实现直接丢弃 Adam 状态，代价是收敛变慢或抖。

## Unreal Engine Relevance

- **Nanite 的对照物**：Nanite 用"簇层级 + 每簇误差界"把简化决策搬到运行时；T2-Splat 是"簇内拓扑自适应 + 每簇一张纹理"的离线版。**两者的共同点是把外观从顶点密度里摘出来**（Nanite 用虚拟纹理）。
- **顶点色 / 顶点 SH 的老问题**：UE 的材质系统本来就是"网格 + 材质"分离的——**游戏引擎二十年前就做完了这件事**。T2-Splat 的价值在于说明：**为什么 GS 系方法此前做不到**（因为外观被绑在顶点属性上，而顶点就是几何）。
- **可借鉴的实验**：拿一个 UE 资产，把"自动 LOD 第 2 级"的 UV 撕开情况与 T2-Splat 的 mirror mapping 对比——两者都是"每面独立参数化"的思路，但 UE 走的是"保留原 UV 岛"，T2-Splat 走的是"每面自带一张"。

## Technology Evolution

见 [[Decoupling Geometry and Appearance]]。本篇是这条线上的**第二个节点**（第一个是 Hoppe 1999 的"概念解耦"），并且是**第一个把存储也解耦的**。

## Relationships

### Based On
- [[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes|Hoppe 1997]] —— ecol / vsplit 两个算子（论文标注 [11] 引的是 Hoppe 1996 *Progressive Meshes*）
- [[Garland 1997 — Surface Simplification Using Quadric Error Metrics|Garland & Heckbert 1997]] —— QEM 代价
- MeshSplatting（Held et al. 2025）—— 不透明三角形定义 + Safe RDT 的母体（库外）
- [[Gaussian Splatting]] —— 残差纹理的灵感来自 GS 系纹理化工作（GSTex / BBsplat / Texture-GS / Textured Gaussians）

### Extends
- **MeshSplatting 的"拓扑刚性"**：同一套网格溅射框架，加上可逆的拓扑编辑。
- [[Mesh Simplification]] 的**离线语境**：把简化算子从"构建期一次"搬进"优化循环内多次"。

### Improves
- 网格溅射的**面数-质量权衡**：Mip-NeRF 360 +0.71 dB / 顶点数 3 M → 2 M。

### Related
- [[Hoppe 1999 — New Quadric Metric for Simplifying Meshes with Appearance Attributes|Hoppe 1999]] —— **正面对手**：属性该进 QEM（Hoppe）还是该搬走（本篇）
- [[Level of Detail]] —— 面预算可调 ⇒ LOD 链的原料
- [[Overdraw]] —— 不透明三角形 + overdraw 归零，是"能进管线"的技术前提

### Contrasts
- **2DGS / GOF / RaDe-GS / SuGaR / MiLo**：先辐射场后抽网格（两阶段），细节在转换时丢；本篇是端到端。
- **Triangle Splatting**：有颜色但**没有连接性**（`Connect ✗`），本篇有。

### Followed By
- 待观察：属性合并换成 wedge / 硬约束之后，能否在 UV 接缝上不再依赖手调权重。

## Personal Knowledge State

- **机制层 Easy**：边塌陷、QEM、mipmap、重心坐标、Adam 矩——全是日常。
- **今天新增（Normal）**：
  1. **"一个元素干两份活"是可预测的过度细分来源**——判断方法是问"这个元素的分辨率由谁决定？如果有两个答案，就有一份浪费"。
  2. **代价函数里不同量纲的项必须先归一化**（这里是 QEM 除以候选中位数）。
  3. **优化器状态是拓扑编辑的一部分**，不是外挂。
- **仍是 Hard**：`user_level: Normal` 的原因在"为什么纹理能替代顶点属性而 UV 接缝不会崩"——这需要 [[Mesh Parameterization]] 与 [[Subdivision Surfaces]] 的支撑，本轮未展开。

## Learning Value

**最高价值的一条**：**"解耦"不是架构美学，是可测量的预算收益**。2.7 M vs 7.3 M 面、236 vs 126 FPS、62 vs 152 MB——三个数字都指向同一件事：**此前有一份容量被用在"外观要求"而不是"几何要求"上**。

**第二价值**：这篇把 [[Mesh Simplification]] 里"属性是工业分水岭"那句话**从判断变成了可操作方案**——虽然它的方案（纹理）与 Hoppe 的方案（wedge + 投影误差）是对立的两条路。

## Visualization

![[Research/Files/Pictures/Tang 2026 — 图2 T2-Splat 两阶段管线.png]]

**读图**：左半是 Stage I——点云 → 三角汤 → 加密剪枝 → **Safe RDT** 拿连接，左下角是 ecol/vsplit 这对可逆算子（图中用双向箭头，强调"能反悔"）；右半是 Stage II——渲染出的像素 $C(P)$ 由**视图相关的基色 $C_{\text{base}}$（顶点 SH）** 加 **空间残差纹理 $C_{\text{res}}$（每面一张、按重心坐标采样）** 两部分组成，两片绿色网格就是同一面的纹理在两个尺度上的采样。**看图顺序：先看左下的双向箭头（拓扑可变），再看右上的加号（外观可加）——这两件事互不影响，就是全篇。**
（图 2，Tang et al. 2026, NeurIPS 2026）

![[Research/Files/Pictures/Tang 2026 — 图1 与 MeshSplatting 的对比.png]]

**读图**：上排是 shelf 场景的**文字细节**——MeshSplatting 与 MiLo 的标签已经糊掉，本篇仍然可读（PSNR 25.68 vs 24.60 vs 22.63）。下排 garden 是**面数账**：本篇 25.68 dB / **2.7 M 面**，GOF 用 **92 M** 面、MiLo 用 **15 M** 面。**这张图真正想说的是"同样的质量，面数差 5~34 倍"——而面数就是显存、带宽、光栅化时间。**
（图 1，Tang et al. 2026, NeurIPS 2026）

## Notes

- **消融（9 场景平均，Mip-NeRF 360）**：baseline 24.66 / 5.3 M / 54 min → +Safe RDT 24.81 / 3.0 M / 43 min → +DTE 25.12 / 2.5 M / 57 min → +8×8 纹理 **25.49 / 2.5 M / 83 min**。三条全开 +0.83 dB、面数 -53%、时间 +54%。
  - **注意 (b) 的时间比 (a) 还短**（43 vs 54 min）：Safe RDT 砍掉冗余面之后，后续优化也变快了。**面数下降的收益是双向的——推理和训练都省。**
- **压缩实验（Table 4, Garden）**：100% → 25.68 / 2.7 M；50% → 25.38 / 1.4 M；**10% → 23.93 / 0.3 M**。对照 MeshSplatting 100% 是 24.70 / 7.3 M ⇒ **本篇用 1/24 的面数只差 0.77 dB。**
- **DTU Chamfer**：本篇 0.68 / Gaussian Wrapping 0.49 / MiLo 0.68 / RaDe-GS 0.68 / 2DGS 0.80 / MeshSplatting 0.89。**并列第二，不是第一。**
- **T&T**：PSNR 21.12（GOF 22.99 / MiLo 23.29 更高），但 LPIPS **0.213**、SSIM **0.795** 是网格类方法里最好的。**PSNR 与感知指标在这里分叉**——PSNR 偏向"平滑正确"，LPIPS/SSIM 偏向"结构对"。
- **未报数字 / 待核**：① 62 MB 是否含纹理（按 8×8×3 逐面估算差一个数量级）；② 与 MeshSplatting 的训练时间横向对比；③ 纹理总显存随面数的增长率（这决定它在开放世界里能不能用）。
- 2026-11-09 建页。同日与 [[Hoppe 1999 — New Quadric Metric for Simplifying Meshes with Appearance Attributes|Hoppe 1999]] 对照阅读——**同一件事的两个方向：一个把属性塞回几何量，一个把属性从几何拿出来。**
