---
type: concept
title: "Mesh Simplification"
user_level: "Easy（边塌陷 + QEM 的机制层）/ Normal（代价模型与属性的取舍）"
aliases: [Mesh Simplification, 网格简化, Decimation, 边塌陷, Edge Collapse, 二次误差度量, QEM, LOD 生成]
prerequisites: ["[[Level of Detail]]", "[[Real-Time Rendering]]"]
first_introduced: "1992（Schroeder 等，顶点删除）；1997（Garland & Heckbert，二次误差）"
tags: [lod, geometry, mesh-simplification, budget, asset-pipeline, classical]
---

# Mesh Simplification

## Definition

**在尽量保住"看起来一样"的前提下，把一份网格变成元素更少的一份。**

它在 [[Level of Detail]] 的三个部件里对应**第一个——"表示"（representations）从哪来**。那一页讲"这一帧该用哪一份、用多少"；本页讲"**这些份是怎么造出来的**"。两者共用同一条判据（误差），但决定频率差得很远：

| | 简化（本页） | [[Level of Detail\|LOD]] |
|---|---|---|
| 什么时候做 | **离线一次**（构建期 / 导入期） | **每帧**（运行时） |
| 决定什么 | 造出哪几份表示 | 这一帧用哪一份 |
| 判据 | 世界空间误差 / 面数 | 屏幕空间误差 / 像素 |
| 出错的表现 | 资产不好看（跳变、UV 撕、轮廓塌） | 帧率抖、pop |

> 一句话：**简化是"把 LOD 的货备好"，LOD 是"把货发出去"。** 本库此前 99 个文件讲 LOD，**没有一页讲这些货是怎么造的**——今天补上。

## Core Principle

**简化不是一个算法，是三个独立决定的乘积。** 混着谈就会得出"QEM 好/不好"这种没有信息量的结论。

```text
① 删什么      —— 原子操作：删点 / 聚类 / 塌边 / 塌"配对"
  ×
② 代价怎么算  —— 距离？二次型？屏幕误差？只算几何还是连属性？
  ×
③ 停在哪      —— 要 N 个面？要误差 ≤ ε？要屏幕上 ≤ τ 像素？
```

**三条互相独立**：可以"边塌陷 + 二次代价 + 面数目标"（工业默认），也可以"边塌陷 + 屏幕误差 + 任意时刻停"（[[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes|Hoppe 1997]]），还可以"配对塌陷 + 二次代价 + 无拓扑约束"（[[Garland 1997 — Surface Simplification Using Quadric Error Metrics|Garland–Heckbert 1997]]）。

> **② 决定 ③ 能不能成立。** 只有"可加的标量代价"才能塞进一个堆里，才能**在任意时刻停下来**，才能同时支持"要面数"和"要误差"两种目标。这是 QEM 之所以统治工业界的唯一原因——**不是它最准，是它最容易被打断。**

## 三条实现路线

| 路线 | 原子操作 | 代价 | 强项 | 硬伤 |
|---|---|---|---|---|
| **顶点删除 + 重三角化** | 删一个顶点，局部重新三角化 | 局部几何误差 | 保拓扑、保流形；质量稳定 | 只能用于流形；重三角化本身是个难题；不能合并不连通块 |
| **顶点聚类** | 栅格/八叉树，把一格内的点合成一个 | 栅格尺寸（间接） | 极快、极通用、可处理任意多边形汤 | 质量差；面数只能间接控制；**结果依赖模型相对栅格的位置与朝向** |
| **迭代塌陷**（边 / 配对） | 塌一条边或一对顶点 | 二次型（QEM）/ 屏幕误差 | 质量最好，可任意时刻停；配对塌陷还能合并不连通块 | 可能翻面、可能改拓扑；属性（UV）处理麻烦 |

**第四条路（2020s 才成规模）：簇化 + 每簇误差界**（Nanite 类虚拟几何）——把"塌陷序列"换成"簇层级 + 每簇的误差上界"，把决策从 CPU 搬到 GPU。**它不是本篇三条的替代，是把"什么时候决定"从离线搬到了每帧。**

## 代价模型的演化（本页主线）

```text
1992  Schroeder：局部几何误差（删点后洞的形变）
  ↓
1993  Rossignac–Borrel：栅格尺寸当误差界（间接、不可控）
  ↓
1996  Ronfard–Rossignac：到【一组平面的距离】，取 max（不可加）
  ↓
1997  ★ Garland–Heckbert：改取 sum ⇒ 二次型 ⇒ 10 个数 + 一次加法
        · 可加、可堆、可任意截断 —— 工业标准的起点
        · 代价：同一平面最多被数 3 次；平面无限延伸 ⇒ 只在局部有效；不可撤销
  ↓
1997  Hoppe：把同一串塌陷记录当"选择性细化"的素材，判据换成【屏幕空间像素误差】
        · 这一步把"世界距离"换成"像素"，于是可以跨物体相加（见 [[Level of Detail]]）
  ↓
1998  Garland–Heckbert：把属性抬进 R^{3+m} 超平面 ⇒ 属性也能进代价
        · 【分岔】投影点不再是几何最近点，跑到"几何更远、属性更近"处 ⇒ 误差低估
  ↓
1999  Hoppe：打回 R³ 投影 + wedge ⇒ 属性第一次被【正确】地放进 QEM
        · 存储从 (4+m)(5+m)/2 降到 11+4m；不连续处成为数据结构问题
  ↓
2013  Thiery：SQEM —— 未知量加一维（球的半径）
  ↓
2026  DC-SQEM：4D 二次型 + 对偶轮廓 ⇒ 从点云直接提中轴（同一套机器的换维用法）
  ↓
2026  T2-Splat：把属性【拿出】顶点，搬到每面残差纹理
        · 拓扑按几何复杂度自由改，外观按纹理分辨率独立保证
        · 面数降到 1/3 而质量更高 —— 见 [[Decoupling Geometry and Appearance]]
```

> **这条线的方向：误差的单位从"世界距离"走向"像素"，从"几何"走向"几何 + 属性"，而"可加性"始终是那条暗线。**

## 三件必须一起处理、却常常被省略的事

**这三件事决定了"论文里的简化器"和"能进管线的简化器"之间的距离。**

**① 边界。** 开放边界（地形、布片、破损扫描、薄片零件）在纯几何误差下会被"吃掉"——因为边界处的平面集本来就少，塌陷代价低。1997 年的补丁（给边界边加**垂直约束平面**并大罚因子加权）今天还在被抄。**症状：简化后地形边缘往里缩一圈。**

**② 属性（UV / 法线 / 顶点色 / 蒙皮权重）。** 几何保住了、UV 撕了，是自动 LOD 最常见的翻车方式。

**问题出在哪，1999 年就说清了**（[[Hoppe 1999 — New Quadric Metric for Simplifying Meshes with Appearance Attributes|Hoppe 1999]]）：

| 做法 | 后果 |
|---|---|
| 把位置和属性捆成一个 $(3+m)$ 维向量，量**联合空间里的距离** | **度量坏**：投影点跑到"几何更远、属性更近"处，误差被**低估** |
| 分项求和（几何误差 + 属性误差） | **几何坏**：属性梯度高处，顶点被推向曲率中心——**用几何换属性**（原文 Fig 7 的 1D 例证） |
| 几何投影在 $\mathbb R^3$，属性在投影点上插值 | ✅ 度量正确（**顶点属性的标准答案**） |
| 纹理坐标 | ❌ 该量**参数误差**（Cohen 1998），本框架没覆盖 |

**而"不连续处"（折痕 / UV 缝 / 材质边界）有三种处理方式，性质完全不同：**

- **声明式**（wedge / 硬边表 / 材质槽）：把汇率设成 0 或 ∞——**禁止**平均。Hoppe 的 wedge、UE / Simplygon 的 seam 保护都是这一类。**最可靠。**
- **罚项式**（色差罚项、TV loss）：**不鼓励**平均。[[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] 的式 (8) 就是这一类。**罚项有限，就一定有汇率。**
- **搬走**：让属性根本不住在要被合并的元素上（每面独立纹理）。**汇率消失，但缝换了个地方。**

> **"QEM 原文明确没做这一层"是 Garland 1997 的自陈**（作者提过 7×7 的想法但嫌复杂）。**这是"论文复现"和"能上项目"之间最宽的沟**——而 1999 年 Hoppe 就把这条沟填上了，工业界却大多选了更保守的"禁止跨边界塌陷"。**可靠性赢了最优性。**

**③ 拓扑该不该保。** 这是一个**产品决定，不是技术决定**：

| 目标 | 要不要保拓扑 | 后果 |
|---|---|---|
| 角色 / 可破坏物 / 需要物理的形状 | **要** | 不能用配对塌陷；不能用聚类；要加翻面检测 |
| 远景资产 / 扫描件 / HLOD / 遮挡体 | **不要** | 可以聚合（把不连通块合成一块），轮廓更忠实，但会出非流形 |

> Garland–Heckbert 1997 明写："在渲染这类应用里，**整体外观比拓扑更重要**"。**这句话在 1997 年是激进的，今天是默认的。**

## Important Papers

| 论文 | 角色 |
|---|---|
| Schroeder 等 1992 *Decimation of Triangle Meshes* | 起点：顶点删除 + 重三角化（库外） |
| Rossignac & Borrel 1993 *Multi-resolution 3D approximations* | 顶点聚类；"快但糙"的参照系（库外） |
| Ronfard & Rossignac 1996 | 代价模型的直接前身：到一组平面的距离，取 **max**（库外） |
| [[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes\|Hoppe 1997]] | 把塌陷序列变成"运行时逐顶点判据 + 控制器"（预算侧） |
| [[Garland 1997 — Surface Simplification Using Quadric Error Metrics\|Garland & Heckbert 1997]] | **代价侧**：max→sum，二次型，10 个数 + 一次加法 |
| [[Hoppe 1999 — New Quadric Metric for Simplifying Meshes with Appearance Attributes\|Hoppe 1999]] | **属性侧**：回到 $\mathbb R^3$ 投影 + wedge；存储 $11+4m$ |
| [[2026-11-05-Li — DC-SQEM Medial Mesh Extraction by Dual Contouring with Spherical Quadric Error Metric\|DC-SQEM 2026]] | 同一套机器的 4 维用法：未知量加一个半径 ⇒ 从点云提中轴 |
| [[2026-11-05-Wang — Perception Over Geometry Rethinking Gaussian Allocation in Mesh-based Splatting\|P-MGS 2026]] | 同一句话换了对象："误差可加 ⇒ 配额可解"，分的是高斯不是面 |
| [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals\|T2-Splat 2026]] | **不补属性，改搬走**：属性住到每面纹理上，拓扑只按几何走 |

## Related Concepts

- **[[Level of Detail]]** —— 上下游关系。本页造表示，那页每帧选表示。**共用"误差可加"这一判据。**
- **[[Decoupling Geometry and Appearance]]** —— 同一个坎的另一面。本页问"代价怎么算"，那页问"**这个属性到底该不该出现在代价里**"——如果属性被搬走，本页第 ② 栏就不存在了。
- **[[Budget Units]]** —— 简化的输出（面数 / 顶点数）是最古老的一种预算单位，也是**最坏的一种**（与屏幕无关）。
- **[[Streaming]]** —— 简化决定"每份多大"，streaming 决定"哪份驻留"。
- **[[Gaussian Splatting]]** —— 2024 年以后出现的新"简化"对象：不是减面，是**减少高斯数**（Budgeted-GS 的"误差 ∝ $N^{-1/2}$"就是 GS 版的率失真曲线）。
- **[[Subdivision Surfaces]]** —— 反方向：简化是"降分辨率"，细分是"升分辨率"，两者共用同一套多分辨率表示的语言。
- **[[Automatic Rigging]]** —— 另一条"用代理体表示形状"的路：本页用更少的面，那边用一串内切球/骨架。

## Technologies

- **自动 LOD 生成**（Simplygon / Unreal "Generate LOD" / MeshLab Quadric Edge Collapse）：塌陷序列 + QEM 代价 + 属性约束。**工业实现的工作量 80% 在属性与边界，20% 在几何。**
- **HLOD / Impostor**：簇化后简化，或干脆换成公告板——**当简化到几百面还不如一张图时，就换表示**。
- **虚拟几何（Nanite 类）**：簇层级 + 每簇误差界；把"离线塌陷序列"换成"运行时按簇选择"。
- **破碎预处理**：碎裂块面数爆炸，QEM 是最省事的降面手段（注意保边界）。

## Game Applications

1. **判据：一个资产需不需要简化，取决于"它在屏幕上最小会占多少像素"。** 占不到 32 px 的东西，与其简化不如换 impostor。
2. **自动 LOD 的三个必查项**（验收用）：① **边界有没有缩**（地形/薄片）；② **UV 有没有撕**（花纹/法线贴图）；③ **轮廓有没有塌**（剪影是否还认得出）。**这三项都不在 QEM 的误差里。**
3. **不要让 QEM 的世界空间误差直接进预算表**：不同尺度的物体不可比（一栋楼与一把刀差 3 个数量级）。要进预算，先换成屏幕空间误差（[[Level of Detail]] 的判据）。
4. **"LOD 之间的过渡"是另一个问题**：本页只造货，不负责切换时的 pop（那是 geomorph / dither fade / 双面混合的活）。

## Personal Knowledge

- **机制层 Easy**：边塌陷、二次误差、自动 LOD 按钮——日常工作，不复述。
- **今天补的三条（Normal）**：
  1. **简化 = 删什么 × 代价怎么算 × 停在哪**，三者独立；**代价的"可加性"决定"能不能随时停"**；
  2. **可加性有三笔账**（重复计数 ≤3 / 只在局部有效 / 不可撤销），**先记代价再记性能**；
  3. **属性与边界才是工业分水岭**，几何误差只是入场券。

## Learning Gap

- **"属性误差"还是没有统一语言——但 2026-11-09 有了"为什么不能统一"的答案。** [[Hoppe 1999 — New Quadric Metric for Simplifying Meshes with Appearance Attributes|Hoppe 1999]] 给出的不是一个大一统公式，而是一条**分类判据**：**属性是否定义了曲面上的参数化**——
  - **定义了**（纹理坐标）⇒ 误差该在**参数域**量（Cohen 1998），因为"UV 挪了一点但视觉没变"不该被罚，"UV 没动但纹理被拉伸"必须被罚；
  - **没定义**（颜色 / 法线 / 辐射度 / 蒙皮权重）⇒ 误差该在**几何对应点**上量。

  **所以"四者合成一个标量"这个目标本身可能就是错的**——它们住在不同的空间里。工业实现"拍权重"，是因为**拍权重是唯一能在不同空间之间硬接的办法**。**这条从"缺答案"降级为"目标可疑"。**
- **"解耦的代价"没有被量化。** 顶点属性 vs 每面纹理，在什么面数下显存开始反转？[[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] 报的 62 MB **对不上**按 8×8×3 逐面估算的 ~518 MB。**没有换算公式，就没有"该不该解耦"的算术答案。**
- **"简化到什么程度"与"看的时候有多远"之间的映射**（即 $E_i \leftrightarrow$ 像素误差）在 QEM 一侧从未建立；Hoppe 1997 建立了另一侧（偏差空间 → 屏幕）。**两套语言至今没有正式翻译。**
- **GS 的"简化"与网格的"简化"是不是同一件事**：Budgeted-GS 的 $N^{-1/2}$ 与 QEM 的"平方距离和"形状不同，**两者的误差是否可以互相换算，没人做过。**

## Next Step

1. 拿一个自己的资产，**用自动 LOD 生成第 2 级，然后只查三件事**：边界缩了多少、UV 缝有没有裂、剪影还认不认得出。**这三个数比"面数降了多少"有用。**
2. 读 [[Garland 1997 — Surface Simplification Using Quadric Error Metrics|Garland 1997]] 的 §5（误差二次型的推导）与 §8（作者自陈的三条弱点）——**八页，一小时，是这条线的全部地基。**
3. 对照读 [[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes|Hoppe 1997]] 的 §5：**同一个"塌陷序列"，一边用来离线排顺序，一边用来每帧筛子序列。** 看懂这一层，LOD 与简化的边界就彻底清楚了。
4. **读 [[Hoppe 1999 — New Quadric Metric for Simplifying Meshes with Appearance Attributes|Hoppe 1999]] 的 Fig 7（一维折线那张）**——它是"加权求和 = 一个汇率"的最小可复现实验，五条线段、两个候选、两个误差分量，**规模小到可以手算**。看完之后再看 [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] 的式 (8)/(10)：**同一件事，一个用约束按住汇率，一个用罚项把汇率调小。**

## Notes

- **建页依据**：全库 373 处提到 LOD / Level of Detail，**0 处提到 quadric**，只有 1 处顺带提到 decimation（Kavan 2008 的对比里）。[[Level of Detail]] 的三行"表示"栏（离散 LOD 链 / 连续细化 / 虚拟几何）**全都预设了"表示已经存在"**——本页补的就是这个预设。
- **与 [[Level of Detail]] 的边界**：那一页的判据是"**这一帧它会错多少像素**"；本页的判据是"**这份表示比原来差多少**"。**合并会让"决定频率"这个关键区分消失**，故不合并。
- 2026-11-08 建页，同日由 [[Garland 1997 — Surface Simplification Using Quadric Error Metrics|Garland & Heckbert 1997]]（代价模型）与 [[2026-11-05-Li — DC-SQEM Medial Mesh Extraction by Dual Contouring with Spherical Quadric Error Metric|DC-SQEM]]（同一机器的换维用法）两个方向支撑。
- **2026-11-09 修订**：属性那一节重写。原版只有一句"两条处理思路"（塞进误差 / 当约束）——**这两条其实是"罚"与"声明"的分类，而 1999 年 Hoppe 已经把三种失败方式逐条说清了**（联合空间错、求和导致用几何换属性、不连续处必须多值）。同日补上第三条路（搬走，T2-Splat）与时间线两个节点。**Learning Gap 第 1 条从"缺答案"降级为"目标可疑"——四类属性住在不同空间，合成一个标量可能本来就是错的。**
