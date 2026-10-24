---
type: concept
user_level: Normal
aliases: ["Mesh Parameterization", "UV Unwrap", "UV 展开", "UV 映射", "参数化", "Texture Atlas", "Chart", "UV 岛"]
prerequisites: ["线性代数（稀疏线性系统）", "离散微分几何（cotangent Laplacian）", "复分析（共形映射）"]
first_introduced: "Tutte, Proc. London Math. Soc. 1963（凸边界重心嵌入）；图形学工程起点为 Eck et al. 1995 与 Lévy et al. 2002"
---

# Mesh Parameterization

## Definition

**把一张三维曲面摊到二维平面上，并且尽量"摊得开、不重叠、不变形"的映射。** 工程上它叫 **UV 展开**；它的输出是一张 **texture atlas（纹理图集）**，由若干片 **chart（UV 岛）** 拼成。

形式化地说：给定一个三角网格 $M\subset\mathbb{R}^3$（通常是圆盘拓扑的一片，或切开后成为圆盘），找一个映射 $U:M\to\mathbb{R}^2$，使得

- **单射**（injective）：不同的点映到不同的位置——**不能重叠**；
- **低扭曲**（low distortion）：角度、面积尽量保持；
- **可算**（tractable）：在几秒内对十万级面片求解。

**这三个要求在数学上互相冲突**——这是整个领域的形状。

**它为什么在游戏里无处不在**：贴图、光照贴图、法线/AO 烘焙、贴花、地形 splat、VFX 的 flipbook——**所有"把 2D 图像贴到 3D 表面"的事，都要先有一个 $U$**。你在 DCC 里点的那个 "Unwrap"，解的就是这个问题。

## Core Principle

### ① 三个目标，只能取两个

| 目标 | 含义 | 数学名 | 谁在乎 |
|---|---|---|---|
| **保角** | 局部形状不变形（圆还是圆） | conformal | 法线贴图、细节纹理（拉伸会让细节方向错） |
| **保面积** | 单位面积对应单位面积 | authalic / equiareal | **texel density 均匀**、光照贴图、程序化图案 |
| **单射** | 不重叠、不翻转 | injective / bijective | **光照贴图**（重叠就漏光）、任何烘焙 |

**Lévy 2002 的原文说得很直白**：不可能三者兼得，所以每种方法都在**选两个、放弃一个**，并且**用别的手段把放弃的那个补回来**（后处理、检测、细分、约束优化）。

### ② 最重要的区分：局部单射 ≠ 全局单射

这是最容易在生产里被糊过去的一点，也是今天两篇论文的交汇处：

| 性质 | 含义 | 谁能保证 |
|---|---|---|
| **无翻转**（局部单射） | 每个三角形的定向不变（不"翻面"） | LSCM 可证；多数优化器可保证 |
| **无重叠**（全局单射） | 任意两个不相邻区域不叠在一起 | **只有 Tutte 系（凸边界）或显式的双射方法能证** |

**为什么这个区分要命**：DCC 里的"检查 UV"通常只查**翻转**（逐三角形定向，O(n) 就能查），**不查重叠**（要判断两个岛是否相交，通常是渲染到纹理空间再数覆盖次数）。于是你会得到一份"检查通过"的 UV，然后在烘焙时发现漏光——**因为漏光来自重叠，不来自翻转**。

LSCM 的处置方式很有代表性（图 6）：**用 stencil 把参数域画一遍，被画两次以上的像素就是重叠**，然后把 chart 切开。**保证被换成了检测 + 补救。**

### ③ 三步流水线（2002 年定型，至今没变）

```text
① 分割 Segmentation  —— 决定"在哪里切开"（seam 的位置）
       ↓                目标：切在看不出来的地方（高曲率/遮挡处）
② 参数化 Parameterization —— 每一片 chart 摊平（本文档的主战场）
       ↓                目标：低扭曲 + 单射
③ 装箱 Packing       —— 把若干片塞进一张方形贴图
       ↓                目标：浪费的面积最小
   Texture Atlas
```

**三步的预算互相牵制**：分割得越碎 → 每片越好摊 → 但接缝越多、padding 越多、装箱率越低。**"接缝数量"是一个真实的预算项**，不是美术偏好。

### ④ 扭曲是怎么量的

- **角度扭曲**：$u$ 与 $v$ 梯度方向的夹角偏离 90° 的程度（LSCM 图 9 左）；
- **面积扭曲**：纹理面积 / 模型面积的比值分布（图 9 右）；
- **stretch metric**（Sander 2001）：把两者合成为一个"拉伸"标量，是工业界最常用的单一指标——**LSCM 论文里那行 "3.5 → 1.52" 就是它**。

## Prerequisites

- **稀疏线性系统**：整套方法的成本几乎全在"解一次 $Ax=b$"上。共轭梯度 + 预条件就够了；今天的实现会再加多重网格；
- **Laplacian 与 cotangent 权重**：调和映射的离散形式。**库内入口**：[[Spherical Harmonics]] 的邻居（同一套"离散算子 + 基函数"的语言），以及 [[2026-10-22-Kaushik — A Tutte-Like Theorem for Injective Flows on Meshes|Kaushik 2026]] §4.1 对"任意正权 Laplacian"的用法；
- **复分析的基本直觉**：共形 = 保角 = 复解析。**只要接受"复数是 2D 旋转+缩放的乘法"，LSCM 的整个推导就只剩一行**；
- **一点点拓扑**：圆盘 / 球面 / 亏格。**"为什么必须先切开"** 就卡在这里（球面不能双射到平面，因为紧致无边 vs 有边）。

## Historical Evolution

```text
1963  Tutte《How to Draw a Graph》
      边界钉在凸多边形上，内部每个点 = 邻居重心；证明平面嵌入是双射
      → 这是"调和 ⇒ 单射"的第一条定理，也是 UV 展开的数学祖师爷
       ↓  但凸边界太苛刻：形状复杂的 chart 会被拧得很厉害
1993  Pinkall & Polthier：cotangent 权重（离散极小曲面），给"调和"一个可算的形式
1995  Eck et al.：把调和映射用于多分辨率建模（u、v 两个独立的实线性系统）
1997  Floater：重心权推广到任意凸组合权（中值坐标等）；边界可共线
       ↓  仍然要求凸边界 → chart 必须切得很碎
2000  MIPS（Hormann & Greiner）：第一次让边界自由，但非线性、慢、可能局部极小
★2002  Lévy et al. LSCM：复数最小二乘 → 一次线性求解 + 自由边界 + 无翻转
       → "一键展开"从此能落地；代价是放弃了全局单射
2001/2003  ABF / ABF++（Sheffer & de Sturler）：直接解角度，质量更好，更慢
2005  ARAP（Liu et al. / Sorkine & Alexa）：目标从"保角"换成"局部刚性"
2016  Campen, Silva & Zorin：simplicial foliations —— 组合意义上的**全局单射**条件
2017  SLIM：把"局部单射"写成可优化的约束（工业级求解器）
2024  TutteNet（Sun, Groueix, Song, Huang, Aigerman）：用 2D 单射形变复合出 3D 单射形变
2026  [[2026-10-22-Kaushik — A Tutte-Like Theorem for Injective Flows on Meshes|Kaushik & Aigerman]]：
      把 Tutte 定理从"映射"推广到"流"——同一个数学结构的第三个对象
```

**这条线的主旋律不是"越来越准"，而是"每放松一条保证，就要在别处把它找回来"**：

```text
Tutte    用"凸边界"换"全局双射"
  ↓ 放松边界
LSCM     用"检测+细分"补回"无重叠"
  ↓ 想要可证的全局单射
Campen/SLIM/TutteNet  用"组合条件 / 约束优化 / 复合构造"把保证找回来
  ↓ 换对象
2026     用"调和性"证明流的单射
```

## Important Papers

| 论文 | 年份 | 在谱系里的位置 | 保证了什么 | 用户水平 |
|---|---|---|---|---|
| [[Tutte — How to Draw a Graph (1963)]] | 1963 | **源头** | 全局双射（凸边界） | Hard（原页不可得，二手） |
| Floater — Parametrization and Smooth Approximation of Surface Triangulations | 1997 | 权重推广 | 双射（边界可共线） | Normal（未入库） |
| **[[Levy — Least Squares Conformal Maps for Automatic Texture Atlas Generation (2002)]]** | 2002 | **工业转折点** | **无翻转 + 唯一解 + 自由边界** | **Normal（已读结论层）** |
| Sheffer & de Sturler — ABF | 2001 | 角度最优路线 | 角度最优（需后处理保单射） | 未入库 |
| Sander et al. — Texture Mapping Progressive Meshes | 2001 | **度量**（stretch） | —（是尺子，不是方法） | 未入库 |
| Campen, Silva & Zorin — Bijective Maps from Simplicial Foliations | 2016 | **全局单射的回归** | 组合条件下的全局单射 | Hard |
| [[2026-10-22-Kaushik — A Tutte-Like Theorem for Injective Flows on Meshes\|Kaushik & Aigerman — A Tutte-Like Theorem for Injective Flows]] | 2026 | **换对象（映射→流）** | 流的单射（球面拓扑） | Hard（结论层 Normal） |

**一张"保证对照表"**（做技术选型时直接查这张）：

| 方法 | 全局单射 | 无翻转 | 自由边界 | 角度质量 | 成本 |
|---|---|---|---|---|---|
| Tutte 1963 / 重心映射 | ✅ 可证 | ✅ | ❌ 必须凸 | 差 | 一次线性求解 |
| Floater 1997 | ✅ 可证 | ✅ | ❌ 凸/可共线 | 中（权重可选） | 一次线性求解 |
| 调和映射（Eck 1995） | ✅（靠凸边界） | ✅ | ❌ | 中 | 一次线性求解 |
| **LSCM 2002** | ❌ **不保证** | ✅ **可证** | ✅ **只需钉 2 点** | 好（共形） | 一次线性求解 |
| ABF / ABF++ | ❌（需后处理） | 通常 ✅ | ✅ | **最好** | 约束优化，慢 |
| ARAP | ❌（需局部单射约束） | 可约束 | ✅ | 好（低拉伸） | 非线性，中 |
| SLIM / 双射方法 | ✅（可优化到） | ✅ | ✅ | 可调 | 非线性，慢 |
| **2026 流版本** | ✅（**流**的单射，非映射） | — | ✅（任意正权 Laplacian） | 由权重决定 | 一次线性求解 + 可微优化 |

## Related Concepts

- **[[Subdivision Surfaces]]**：同一族"几何处理"工作，且共享同一种叙事结构——**一个 1960/70 年代的数学结果，被工程需求反复重写**（Catmull-Clark 1978 → Stam 1998 求值 → DeRose 1998 生产化）。参数化这条线（Tutte 1963 → LSCM 2002 → 2026 流）是它的平行故事。**两者的分界**：细分解决"曲面怎么表示"，参数化解决"曲面怎么摊平"；
- **[[Skinning]]**：另一个"把一个空间的东西搬到另一个空间"的问题。**结构上同源**（都是"用一组权重把若干变换/位置混合起来"），**但保证完全不同**：蒙皮要的是"混合后仍是刚体"（[[Kavan — Geometric Skinning with Approximate Dual Quaternion Blending (2008)|DQS]] 解决），参数化要的是"映射后仍是单射"。**两篇经典可以并读**：都在问"混合/求解之后，那个好性质还在不在"；
- **[[Physically Based Rendering]]**：UV 的下游消费者。**参数化的失真会以"贴图看起来被拉长"的形式出现在 PBR 材质里**——而美术往往归因于贴图本身；
- **[[2026-10-22-Kaushik — A Tutte-Like Theorem for Injective Flows on Meshes|Kaushik & Aigerman 2026]]**：把"参数化"的数学搬到"流"上。**参数化是"每点去哪"，流是"每点沿什么路径走"**——同一套调和理论的两个用途。

## Technologies

> 这一层目前没有独立笔记，先以"技术名 + 它在哪"列出；后续若某条线加深再建页。

- **DCC 的 Unwrap / AutoUV**：3ds Max / Maya / Blender / Substance Painter 各有一套；LSCM、ABF、投影展开是常见内核。**具体实现未核实，不要假设是 LSCM**；
- **Lightmap UV 生成器**：UE 的 Lightmap UV 通道、Unity 的 Generate Lightmap UVs——它们的目标函数和 albedo UV **不同**（首要目标是"不重叠 + 均匀 texel"，不是"保角"）；
- **Texture Atlas 打包器**：装箱算法（矩形 / 轮廓 / 地平线）。**Lévy 2002 的地平线装箱率 0.37–0.60 是个可对照的基线**；
- **Flipbook / SubUV**：网格化 atlas 的特例——**它不需要参数化**（帧是规则排布），但"方形帧装非方形内容"的浪费账是一样的；
- **Flow map**：不是参数化，但共享同一套"曲面上的 2D 场"语言。见 [[2026-10-22-Kaushik — A Tutte-Like Theorem for Injective Flows on Meshes|Kaushik 2026]]。

## Game Applications

- **静态网格 albedo UV**：常规展开。允许极小重叠（罕见），但**接缝处的 mip 会露馅**——接缝要 padding；
- **光照贴图 UV（Lightmap）**：**重叠是致命的**。两个面共享同一个 texel ⇒ 一个面的间接光被涂到另一个面上 ⇒ **漏光 / 阴影串面**。这条是"为什么 Lightmap UV 要单独生成、单独检查"的全部理由；
- **法线/AO/曲率烘焙**：重叠会让射线打到错误的岛，得到"别人的 AO"；
- **地形 splat map**：本质是把地形摊到 UV 上，规则网格，不需要复杂参数化；
- **角色贴图与 texel density**：不同部件如果 texel density 不一致，**皮肤的颗粒感会忽然变粗**——这是"面积扭曲"在美术眼里的样子；
- **VFX**：贴花（decal）沿曲面滑动、技能特效沿武器流动——**前者需要参数化，后者需要流场**。

## Personal Knowledge

**现状：Normal 入口，未系统读。** 你在材质与贴图上是强项，但**UV 展开在你这里是黑盒**：会用、会检查"看起来对不对"，但没有"它在优化什么、它保证了什么"的框架。这个概念页补的就是这一层。

**已有的相关底子**（可直接调用）：

- 线性代数与稀疏求解——不陌生；
- [[Physically Based Rendering]] 与 [[BRDF]]——知道"贴图被采样"这件事在着色器里怎么发生；
- 边界情况：你**没有**离散微分几何的训练（cotangent 权重、Laplacian 的几何含义），这是主要缺口。

## Learning Gap

| 缺口 | 为什么重要 | 补法 |
|---|---|---|
| **cotangent Laplacian 的几何含义** | 调和映射的全部成本与质量都由它决定；2026 那篇的"任意正权 Laplacian"直接建立在这个概念上 | 先接受"它是最小化 Dirichlet 能量的离散算子"，不必推系数 |
| **离散 Poincaré–Hopf 指标** | 读懂 2026 那篇 Lemma 3 的门槛 | **可以不补**——结论层够用；要补就读 Gortler 2006 §2–3（PDF 可直读） |
| **局部单射 vs 全局单射的判定** | 这是唯一一条**会影响你日常验收**的缺口 | 见下方 Next Step 的 10 分钟动作 |
| **各 DCC/引擎的具体算法** | 决定"这个工具在什么情况下会坑你" | 未知，需要按工具查文档（本页不臆测） |

## Next Step

**三个动作，按成本从低到高：**

1. **（10 分钟，零成本）查一次重叠**：拿你最近做的一个带 Lightmap 的静态网格，在 UE 里看它的 Lightmap UV 通道有没有岛叠在一起（最直观的做法：把 UV1 导进 Substance 或 Blender，看有没有面片重合）。**如果叠了，而你的烘焙没出问题——那只是运气（两个面恰好同色）**；
2. **（20 分钟）读本篇的图 6 + 图 7**：图 6 是"无翻转但重叠"的物证，图 7 是"装箱浪费"的物证。**这两张图各对应一个你能立刻检查的资产问题**；
3. **（30 分钟）读 [[Levy — Least Squares Conformal Maps for Automatic Texture Atlas Generation (2002)]] 的结论层**：三步流水线 + 四条性质 + 那条边界。**读完你应该能回答："我的展开器保证了什么、没保证什么、"**。

**升级判据（Normal → Easy）**：能对着一个具体资产的 UV，说出"它为什么在这里切开、这里的拉伸是共形换来的还是面积换来的、这块能不能用于光照贴图"。**能说出这三句，这个概念就到 Easy 了。**

---

**相关每日记录**：[[2026-10-24]]
