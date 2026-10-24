---
type: paper
title: "How to Draw a Graph"
authors: ["William Thomas Tutte"]
year: 1963
published: 1963
venue: "Proceedings of the London Mathematical Society s3-13(1), pp. 743–767"
url: "https://doi.org/10.1112/plms/s3-13.1.743"
doi: "10.1112/plms/s3-13.1.743"
code: ""
project_page: ""
category: ["graph-theory", "geometry-processing", "parameterization", "classic", "foundation"]
importance: A
historical_importance: 5
game_relevance: 4
production_readiness: "Industry Adopted（以"重心/凸边界展开"的形态活在所有 UV 展开器里，尽管多数工具已换成 LSCM/ARBF 一类）"
user_level: Hard
status: unread
---

# Tutte — How to Draw a Graph (1963)

> ⚠️ **本页是"二手重建"，不是精读笔记。** 原文（Proc. LMS 1963）在 OUP 与 Wiley 均为付费墙，无公开 PDF 镜像（2026-10-24 核实，见 §Notes 的核证清单）。以下内容来自三个可独立验证的来源：① Gortler, Gotsman & Thurston 2006 第 1 节对定理的完整复述；② [[Levy — Least Squares Conformal Maps for Automatic Texture Atlas Generation (2002)|Lévy et al. 2002]] 的引言与 §1.1；③ zbMATH 评论（Zbl 0115.40805，由扫描件转写）。**未读原页之前，本页不应被当作一手依据。**

## TL;DR

**把图的边界画成一个凸多边形，让每个内部点待在它邻居的重心上——解一个线性系统，得到的图形必然不自交，且每个面都是凸的。**

1. **这不是"画得好看"的启发式，是一条定理**：对**节点三连通**的平面图，把某个面（peripheral polygon）钉在凸多边形上，其余顶点解"等于邻居重心"的线性方程组，**解存在、唯一，且得到的直线画法无交叉、所有内面为凸**；
2. **在图形学里它变成了 UV 展开的第一块基石**：把网格的边界环钉在一个凸多边形（通常是圆）上，解两次调和方程得到 $(u,v)$，就得到一个**双射**的参数化——**不折叠、不自交，有证明**；
3. **它的代价也同样清楚**：**边界必须是凸的、且必须钉死**。这条约束在 1963 年是免费的（画图不需要"自由边界"），在 2002 年成了主要瓶颈——[[Levy — Least Squares Conformal Maps for Automatic Texture Atlas Generation (2002)|LSCM]] 用"放弃全局单射"换来了自由边界；
4. **它至今仍在被"重新证明"**：Gortler 2006 用离散 1-form 与 Poincaré–Hopf 指标给出更短的证明并推广到多边界与环面；[[2026-10-22-Kaushik — A Tutte-Like Theorem for Injective Flows on Meshes|Kaushik & Aigerman 2026]] 把同一结论搬到"流"上。**一个 1963 年的定理，被三次重新表述，每次都是为了换一个对象或换一套工具。**

## Problem

1960 年代的图论有一个基本问题：**平面图（planar graph）一定能"画"在平面上而边不相交——但怎么**算**出来？**

- Fáry 定理（1948）已经证明"任何平面图都有直线画法"，但那是**存在性**，不给算法；
- Kuratowski 定理（1930）给了"平面性"的判据（不含 $K_5$ 与 $K_{3,3}$ 的细分），同样不构造；
- 而工程上（当时是电路布线、地图绘制）需要的是**一个能算的构造**。

Tutte 的答案极其简洁：**把外边界固定成一个凸多边形，内部点各自站在邻居的重心上**。这是一个线性方程组——$2(n-h)$ 个未知数、$2(n-h)$ 个方程（$h$ 是边界点数）——**解出来就是**。

**它的全部难度在证明**：为什么解一定存在？为什么画出来一定不自交？为什么面一定是凸的？

## Historical Context

```text
1930  Kuratowski：平面性的判据（不含 K5 / K3,3 细分）
1948  Fáry：平面图必有直线画法（存在性，无算法）
1960  Tutte：前一篇（Proc. LMS (3) 10, 304–320）——图与平面网格的对应
★1963  本文：凸边界 + 重心 ⇒ 可算的直线平面画法（"spring embedding"）
1991  Colin de Verdière：推广到非正曲率高亏格曲面（边为测地线）
1995  Eck et al.：把同一构造搬进图形学（多分辨率网格的调和映射）
1997  Floater：重心权推广到任意凸组合权（中值坐标），边界可共线
2002  Lévy et al. LSCM：放弃凸边界，用"无翻转 + 检测重叠"换自由
2006  Gortler, Gotsman & Thurston：用离散 1-form / Poincaré–Hopf 指标重证并推广
2015  Aigerman & Lipman：Orbifold Tutte（球面与双曲 orbifold 上的版本）
2024  TutteNet：用 2D 单射形变复合构造 3D 单射形变
2026  Kaushik & Aigerman：Tutte 定理的"流"版本
```

**六十年，五次重写**：本文（1963）→ Floater（1997，换权重）→ Gortler（2006，换证明工具）→ Aigerman（2015–2016，换目标域）→ Kaushik（2026，换对象）。**每一次都不是因为它错了，是因为它太对了——对到可以搬到任何地方去。**

## Previous Work

- **Fáry 1948**：存在性证明（用归纳法把图逐步"拉直"），不给出可算的坐标；
- **Steinitz 定理**（1916/1934）：三连通平面图 ↔ 凸多面体。**这是本文"三连通"假设的来源**——三连通保证了嵌入的唯一性（在球面上，三连通平面图的嵌入本质上唯一）；
- **Tutte 1960**（前作）：把图与"平面网格（planar mesh）"对应起来，建立了本文要用的一整套术语（peripheral polygon、planar mesh、nodally 3-connected）。

## Core Idea

**"重心"这个条件，等价于"每根边都是一根弹簧、每个内部点受力平衡"。**

物理直觉：边界钉死，内部点被邻居的弹簧拉着；弹簧系数全相等时，平衡位置就是邻居坐标的算术平均。**弹簧系统一定有唯一平衡点**（能量是凸的），而 Tutte 的贡献是证明**这个平衡点构成的画法不自交**。

数学表述：设内部顶点 $v_i$ 的度为 $d_i$，则

$$x_i=\frac{1}{d_i}\sum_{j\in N(i)}x_j,\qquad y_i=\frac{1}{d_i}\sum_{j\in N(i)}y_j$$

这是两个**解耦的**线性系统（$x$ 与 $y$ 各一个），矩阵是**对角占优的稀疏矩阵**——存在唯一解，且可以用高斯消元或迭代法算。

**关键定理（据 Gortler 2006 的复述）**：对节点三连通的平面图，把某个 peripheral polygon 映到严格凸多边形，上述方程组的解给出一个**直线平面画法**，且**每个面都是凸区域**。

**为什么面会凸**：因为每个内部点严格位于邻居的凸包内部（重心性质），配合"最小角原理"一类的论证，可以排除退化——**这正是"严格凸边界"这个条件不能放松的原因**。

## Technical Approach（据二手来源整理）

1. **前提**：图是 nodally 3-connected（无自环、无重边、无割点；任意两个顶点分开后，其中一片的内部顶点度为 2）——这个条件保证了"平面网格"的唯一性；
2. **固定**：取一个 peripheral polygon（一个面），把它的顶点映到一个**严格凸**多边形上；
3. **求解**：其余顶点解"重心方程"，$x$ 与 $y$ 分开解；
4. **结论**：解存在唯一；得到的画法是直线、无交叉、面为凸；
5. **副产品**：本文还讨论了"平面网格"的判定与对偶性，并重述了"平面 ⟺ 有平面网格 ⟺ 无 Kuratowski 子图"这三者的等价（据 zbMATH 评论）。

**图形学版本（1995 年之后的标准用法）**：

```text
① 找到网格的边界环（圆盘拓扑）
② 把边界环按弧长比例映射到一个圆（凸！）
③ 解 Δu = 0, Δv = 0（cotangent 权重），边界值固定
④ 得到 (u,v) —— 双射、无折叠
```

**第 ③ 步的权重一旦从"均匀"换成 cotangent，就从"图论画图"变成了"离散调和映射"**——这是 Eck 1995 与 Floater 1997 的贡献，也是"Tutte 嵌入"在图形学里的实际形态。

## Key Contribution

1. **第一个可算的平面直线画法**：把"存在性"变成"解一个线性系统"，这一步把图论结论变成了工程算法；
2. **"调和 ⇒ 单射"的第一个实例**：重心条件就是离散调和条件（均匀权重下的 Laplacian = 0）。**这条蕴含关系是此后六十年所有"可证单射"工作的祖先**；
3. **凸边界的必要性被固定下来**：它既是保证的来源，也是此后所有改进要突破的对象。

## Why It Works

**因为"每个点都在邻居的凸包里"这件事，本身就排除了"折"的可能。**

直观论证：假设画法自交，则存在一条边穿过了另一条边。沿着这条边看它两侧的面，可以构造出一个"最内侧"的角，而这个角必须由某个顶点张开——但该顶点在它邻居的凸包里，张不出这样的角。**这是一个"最小反例"式的论证**（Tutte 原文的证明结构，据 Gortler 2006 的转述，用的是"最小角"与"环绕数"）。

**换成今天的语言**：重心条件 ⇒ 离散调和 ⇒ 有离散最大值原理 ⇒ 每个内部点的值被边界值夹住 ⇒ 不可能出现"折返"。**2026 那篇用的是同一条最大值原理**（Lemma 3 的证明里明确写了 "discrete maximum principle"）——**同一个工具，六十年后仍在用**。

## Limitations

1. **边界必须凸且固定**。这一条在图形学里是致命的：形状复杂的 chart 会被拧得很厉害，于是只能切碎（chart 变多、接缝变多）；
2. **只对圆盘拓扑直接成立**。球面/高亏格要先切开，或用 orbifold 版本（Aigerman 2015）；
3. **均匀权重 = 差质量**。Tutte 原文的权重是 $1/d_i$（均匀），这会让参数化严重扭曲；工程上必须换 cotangent 或中值坐标（Floater 1997）；
4. **三连通是硬前提**。网格有割点/边界退化时不适用；
5. **原文不可得**（对今天的读者而言）：这是本页最大的限制——**本页的所有技术细节都是二手的**。

## Game Development Relevance

**这是"为什么你的 UV 展开器有时候只能给你一个圆"的根源。**

- 所有"凸边界展开"（很多 DCC 里的 "Planar/Cylindrical/Spherical projection" 之外的 "Unwrap" 选项之一）都是这个定理的直接实现：**边界钉在圆上、内部解调和**；
- 它给出的保证是**全局双射**——**这正是光照贴图需要的那条保证**。今天多数工具为了质量放弃了它（换 LSCM/ARAP），于是**重叠变成"要自己查"的事**；
- **一个实用的判断**：如果你的 UV 展开结果"看起来被拧成一团"，那多半是凸边界的锅——**换工具（LSCM/ARAP）或手动加接缝**，不要试图用 Relax 修好。

## Unreal Engine Relevance

- UE 的 Lightmap UV 生成器**倾向于**走"保守、可证"的路线（因为它的第一目标是"不重叠"而不是"好看"）——**但具体算法未核实**，不要假设它就是 Tutte 嵌入；
- **可观察的现象**：把一个复杂静态网格的 Lightmap UV 生成出来，如果看到某个岛**明显被压成一个圆形/椭圆**，那就是凸边界展开的特征——**它保证了不重叠，但牺牲了 texel 均匀**。

## Technology Evolution

```text
1963  Tutte：凸边界 + 重心 ⇒ 双射（本页）
1995  Eck：搬进图形学（多分辨率）
1997  Floater：换权重（任意凸组合）
2002  LSCM：放弃凸边界与全局单射，换质量与自由（[[Levy — Least Squares Conformal Maps for Automatic Texture Atlas Generation (2002)|Lévy 2002]]）
2006  Gortler：换证明工具（1-form + 指标定理），推广到多边界/环面
2015  Aigerman：换目标域（orbifold）
2016  Campen：换条件（组合通量规则 ⇒ 全局单射）
2024  TutteNet：换构造（2D 单射复合 ⇒ 3D 单射）
2026  Kaushik & Aigerman：换对象（映射 ⇒ 流）
```

## Relationships

### Based On

- **Steinitz 定理**（三连通平面图 ↔ 凸多面体）：提供了"三连通"这个前提的几何含义；
- **Tutte 1960**（前作，Proc. LMS (3) 10, 304–320）：建立了 peripheral polygon / planar mesh / nodally 3-connected 这套术语。

### Extends

- **Fáry 1948（存在性）→ 本文（可算性）**：把"一定画得出来"变成"这样算就画得出来"。

### Related

- **[[Mesh Parameterization]]**：本文是这个概念页所有方法的共同祖先；
- **[[Levy — Least Squares Conformal Maps for Automatic Texture Atlas Generation (2002)|Lévy et al. 2002]]**：**它明确把"若把所有边界点钉在凸多边形上则不会重叠"写进 limitations**——那句话就是本文定理的现代引用；
- **Gortler, Gotsman & Thurston 2006**：库内**最可读的 Tutte 定理二手来源**（PDF 可直读），本文的技术细节主要依据它。

### Followed By

- **[[2026-10-22-Kaushik — A Tutte-Like Theorem for Injective Flows on Meshes|Kaushik & Aigerman 2026]]**：标题里就写着 "Tutte-Like"；
- **TutteNet（2024）**、**Campen 2016**、**Aigerman 2015/2016**：都在这条线上。

## Personal Knowledge State

**Hard，且资料受限。** 结论层（"凸边界 + 重心 ⇒ 双射"）是 Normal，**20 分钟能懂**；但**证明层完全不可达**（原文不可得），所以这一页在你的知识库里应当被当作**"地标"而非"读物"**——知道它在那儿、知道它说了什么、知道谁在引用它，就够了。

## Learning Value

- **它是"用线性系统换几何保证"这个套路的起点**。你每天用的展开器、每一步的"解一次稀疏系统"，源头都在这；
- **它是"约束的代价"的最好教材**：凸边界换来双射，自由边界换来质量——**没有免费的正确**；
- **它示范了"一个定理如何活六十年"**：不是被记住，是被**反复重新表述**（换权重、换证明、换目标域、换对象）。**判断一篇论文是否经典，看它被"重写"了几次，而不是被"引用"了几次。**

## Visualization

**本页不含论文原图**——原文 PDF 不可得（见 §Notes）。若需要图形直觉，读以下两处（都在库外可直读的来源里）：

- **Gortler, Gotsman & Thurston 2006 §1**：对"凸边界 + 重心 ⇒ 无交叉"的复述，含示意图；
- **[[Levy — Least Squares Conformal Maps for Automatic Texture Atlas Generation (2002)|Lévy 2002]] 图 6-A**：（反例方向）同一个 mesh 在自由边界下的样子——**把它想象成"边界被压进一个圆"就是 Tutte 的结果**。

**待补**：若日后拿到原文，应优先补三张图——① 凸边界嵌入的示意；② 面为凸的证明示意；③ Tutte 的原例（他画的是一个立方体图）。

## Notes

**核证清单（2026-10-24）**

| 项 | 状态 | 依据 |
|---|---|---|
| 标题 / 期刊 / 卷期 / 页码 | ✅ 三源一致 | zbMATH Zbl 0115.40805；OUP 期刊目录页；Lévy 2002 参考文献 [30]（Lévy 把年份误写为 1960 的"Convex representation of graphs"，**这是 LSCM 原文的一处引用错误**，注意不要跟着抄） |
| DOI | ✅ | 10.1112/plms/s3-13.1.743（OUP 目录页） |
| 定理陈述（三连通 + 凸边界 + 重心 ⇒ 无交叉、面凸、解唯一） | ✅ 两源一致 | Gortler 2006 §1；zbMATH 评论 |
| 证明结构（最小角 / 环绕数） | ⚠️ **未核实** | Gortler 2006 提到 Tutte 的证明思路，但本页未读到原文证明，**表述可能不精确** |
| 与 Steinitz 的关系 | ⚠️ 推断 | 由"三连通"假设反推；未在二手来源中看到明确陈述 |
| 原文 PDF | ❌ 不可得 | OUP `academic.oup.com/plms/article-pdf/...` 返回 403；Wiley `londmathsoc.onlinelibrary.wiley.com` 同；无公开镜像（JHU 课程页 404，CiteSeerX 429） |

**给未来的自己**：这一页的价值在"位置"，不在"内容"。若哪天需要写参数化方向的正式材料，**必须走图书馆拿原文**，不要引用本页的技术细节。
