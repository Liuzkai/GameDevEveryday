---
type: paper
title: "New Quadric Metric for Simplifying Meshes with Appearance Attributes"
authors: [Hugues Hoppe]
year: 1999
published: "1999-10-24（IEEE Visualization '99）"
venue: "IEEE Visualization 1999（VIS '99），pp. 59–66"
url: "https://hhoppe.com/proj/newqem"
code: ""
project_page: "https://hhoppe.com/proj/newqem"
category: [mesh-simplification, lod, geometry-processing, appearance-attributes, classical]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal
status: read
aliases: [Hoppe 1999, New Quadric Metric, 属性 QEM, Appearance Attribute QEM, wedge quadric]
tags: [mesh-simplification, lod, geometry-processing, attribute, classical, easy-mechanism]
---

# Hoppe 1999 — New Quadric Metric for Simplifying Meshes with Appearance Attributes

## TL;DR

**把属性塞进几何的度量空间里，会同时做坏两件事；这篇论文证明了这个"不可能三角"，并给出了当时唯一可行的解法。**

Garland–Heckbert 1998（下称 GH98）的推广很自然：既然 QEM 是"到平面的距离平方和"，那就把它搬到 $\mathbb R^{3+m}$——"到**超平面**的距离平方和"。Hoppe 说这错了，而且错得很具体：

> 投影点 $\bar p$ 不再是 $p$ 在几何面上的投影。它跑到"**几何上更远、但属性上更近**"的地方去了。于是度量**低估**了真实误差。

Hoppe 的替代方案一句话：**投影在 $\mathbb R^3$ 里做，属性在投影点上插值**。属性变成"定义在几何面上的一个线性泛函 $\hat s_j(p)=g_j^\top p+d_j$"，而不是顶点的私产。

三个后果，都是钱：
1. **存储从 $(4+m)(5+m)/2$ 降到 $11+4m$**（线性于属性数）：m=3 时 **28 → 23**，m=6 时 **55 → 35**，m=8 时 **78 → 43**。
2. **更快**（矩阵稀疏：前 3×3 稠密，后 $m\times m$ 是一个常数倍的单位阵，只占 1 个系数）。
3. **更准**：同样的 rms 误差，**常常只需要不到一半的面数**。

> 还有一句来自作者 20 年后主页的话，比论文本身更像预言：
> **"The present trend is to replace per-vertex attributes by texture atlases."**
> 他自己把这条路指出来了——而 27 年后 [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] 正是这么干的。

## Problem

1997 年 QEM 解决了几何简化：$\sum_p (p^\top v)^2 = v^\top Qv$ ⇒ 每顶点 10 个浮点数、合并就是矩阵加法。但**带属性的网格**怎么办？法线、顶点色、纹理坐标、辐射度解……

GH98 的答案是把维度抬上去。代价有三处，Hoppe 逐条点了：

| | GH98（$\mathbb R^{3+m}$ 超平面） | Hoppe 1999（$\mathbb R^3$ 投影 + 属性泛函） |
|---|---|---|
| 投影点 | 几何上**不一定最近**（可能更远） | 就是几何最近点 |
| 误差性质 | **低估**真实误差 | 几何/属性各自可解释 |
| 存储（m=3 / 6 / 8） | 28 / 55 / 78 | **23 / 35 / 43** |
| 矩阵 | 稠密 | 稀疏（后 $m\times m$ 是 $c\cdot I$） |

**低估这件事有多具体？** 论文给了一个极限行为：对**平面**几何，GH98 在属性权重 $\lambda\to0$ 时**收敛到**本方法（但 $\lambda=0$ 时不行——那时超平面塌成平面，属性项消失）。换句话说，**GH98 在"属性完全不算"和"属性算满"之间没有正确的插值**，只有一端是对的。

## Historical Context

```text
1996  Ronfard–Rossignac：到一组平面的距离，取 max（不可加）
  ↓
1997  Garland–Heckbert：max → sum ⇒ 二次型（几何的答案）
  ↓
1998  Garland–Heckbert：抬到 R^{3+m} 超平面 ⇒ 属性也能进（但度量坏了）
  ↓
1999  Cohen / Olano / Manocha：Appearance-preserving（SIGGRAPH'98）
        · 用【参数对应】而不是几何对应 ⇒ 对纹理映射是正确度量
        · Hoppe 明说：这只对"定义了参数化"的属性正确
  ↓
1999 ★ Hoppe：回到 R³ 投影，属性在投影点上插值 + wedge 处理不连续
        · 这是 vertex attribute（颜色、法线、辐射度）的正确度量
        · 作者自陈：纹理坐标那条路他没走（该用 Cohen 的参数误差）
  ↓
2000s Simplygon / Unreal "Generate LOD" / MeshLab：工业实现全部走这条路
  ↓
2026  T2-Splat：干脆不让属性住在顶点上 —— 每面一张残差纹理
```

> **1999 年就有两条路，而且两条都对——只是适用对象不同。** Hoppe 这篇的关键判断是**"属性是否定义了曲面上的参数化"**：
> - **定义了**（纹理坐标）⇒ 误差该在**参数域**量（Cohen 那条路），否则"UV 挪了一点但视觉没变"会被罚，"UV 没动但纹理被拉伸"反而不罚。
> - **没定义**（颜色、法线、辐射度）⇒ 误差该在**几何对应点上**量（本篇）。

## Core Idea

**先把"对应关系"定下来，再在这个对应上比属性。**

这是全篇的支点。GH98 把位置和属性捆成一个 3+m 维向量，于是"最近"是在联合空间里定义的；Hoppe 说**对应关系只能由几何定义**——因为属性是**定义在表面上的场**，它本身不携带"哪里离哪里近"的信息。

一旦这么看，属性就不是"顶点的第 4、5、6 个坐标"，而是**"这张面上的一条线性函数"**：

$$\hat s_j(p)=g_j^\top p+d_j,\qquad \text{约束 } n^\top g_j=0$$

第二个约束是关键：它保证 $\hat s_j(p)=\hat s_j(\bar p)$——**沿法向走，属性值不变**。也就是说这个泛函**只在曲面内变化**，垂直于曲面方向它没有定义（也不该有）。

$(g_j,d_j)$ 由解一个 4×4 系统得到（三个顶点值 + 法向正交约束）。

属性误差就是：
$$Q^f_{s_j}(v)=(\hat s_j(p)-s_j)^2=(g_j^\top p+d_j-s_j)^2$$

于是每个面的 quadric 是两块的和：
$$Q^f=Q^f_p+\sum_{j=1}^m Q^f_{s_j}$$

**几何项零扩展**（前 3×3 是 $nn^\top$，后 $m\times m$ 全零），**属性项只在第 $j$ 个对角元上放 1**。所以 $A$ 的后 $m\times m$ 子块是一个常数倍的单位阵——**这就是"$11+4m$"的全部秘密**。

## Technical Approach

### Wedge：把"不连续"变成数据结构

网格的属性场常常不连续：折痕（法线跳变）、材质边界（颜色/UV 跳变）、辐射度解（非平行面之间强度不同）。一个几何点因此需要**多个属性值**。

Hoppe 用 **wedge**：一个顶点被切成 $k\ge1$ 个 wedge，每个 wedge 带自己的属性向量 $\bar s_i$，相邻面的"角"被指派到某个 wedge。

$$Q_v(p,\bar s_1,\dots,\bar s_k)=\sum_{i=1}^k Q_{w_i}(p,\bar s_i)$$

$Q_v$ 的维度是 $3+km$（**可变**），但**从不需要显式存储**——只在考察一条边塌陷时从 $Q_{w_i}$ 现场拼出来。最小化它同时给出**位置和全部 wedge 属性**。

塌陷后的 wedge 合并规则很克制：**若 $w_a$ 与 $w_b$ 在塌陷前都延伸到面 $f_1$，就合并**（每个面 0 / 1 / 2 对）。当两个端点各只有一个 wedge 时，这套规则退化为原来的 QEM。

尖锐边（含开放边界）另加一个**边 quadric**，加到该边相邻的 4 个角上（边界边是 2 个）。

> **这是全文最耐用的一段。** 二十多年后所有工业简化器的"UV 接缝 / 硬边 / 材质边界"处理，本质都是这条思路的变体：**先声明"这里不许平均"，再谈代价。** [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] 用的是另一个方向——**不声明，改在代价里收税**。

### Memoryless：忘掉细节才能继续简化

标准 QEM 在**原始网格**上算好每个顶点的 quadric，之后一路相加。Hoppe 试验了另一种做法：**每次都用当前简化后的网格重算**。

反直觉，但有效。原因用一张图说清了：原始网格上，细尺度特征对应**方向各异的小椭球**；把它们相加，会得到一个**紧致的小球**——顶点被锁死，即使结果网格已经是平的。memoryless 让系统"忘掉"已经抹平的细节，于是能继续塌。

**顺带一个一手归属**：Hoppe 指出 Lindstrom–Turk 的"四面体体积平方和"度量，其实就是**以面面积平方为权重的 QEM**，只是定义在已简化的邻域 $\mathcal F_{i+1}$ 上。原文括号里写了一句 "This observation does not appear in their paper [15]。"——**这条等价关系是 Hoppe 补的，不是 LT 提的。**

### Volume preservation：修"属性逼着几何缩水"

这是全篇最值钱的一节，因为它暴露了一个**结构性问题**而不是参数问题。

实验发现：新度量有时会在**属性梯度高**的地方让模型**收缩**——合并出的顶点被推向曲率中心。

论文用 1D 折线说清了机制。原始折线有五个点，属性 $s$ 沿它走 $1\to0\to1\to0$。

| 塌陷方式 | 几何误差 | 属性误差 |
|---|---|---|
| 上右：$v=(2,1,a)$，位置不动 | $Q_p=0$ ✅ | $Q_s=(a+1)^2+2(a-1)^2>0$ ❌ |
| 下右：$v=(1,0,0)$，**往曲率中心移** | $Q_p=3(1)^2>0$ ❌ | $Q_s=0$ ✅ |

**为什么往曲率中心移能消掉属性误差？** 因为顶点移到那里之后，它在原三段折线上的**投影落到了线段内部**，于是属性可以**内插**而不是**外推**。而外推正是 $Q_s>0$ 的来源。

> **这不是 bug，是"加权求和"的必然。** 只要几何项和属性项被加进同一个标量、权重有限，就存在一个**汇率**：优化器会找到"几何稍微错一点、属性完全对"的那个解。**Hoppe 用体积约束把这个汇率按住**（等价于禁止"用体积换属性"），但汇率本身没有消失——它只是被限制在"不改变体积"的那一维里。

修法很轻：体积保持是一个**线性**约束
$$g_{\text{VOL}}^\top p+d_{\text{VOL}}=0,\qquad g_{\text{VOL}}=\sum_{f\in\mathcal F_{i+1}}\tfrac13\,\mathrm{area}(f)\,n_f$$
于是最小化 $Q_v$ 变成解一个带**一个拉格朗日乘子**的系统。

### 数值细节（都是实战经验）

- **病态**：邻域零高斯曲率（平面 / 柱面）时系统会病态。此时设 $p=(p_1+p_2)/2$，只解属性部分。
- **属性不会造成奇异**：因为 $A$ 的对角 $m\times m$ 子块都是单位阵的倍数——**属性维永远满秩**。这条很漂亮：几何可以退化，属性不会。
- **越界**：颜色有 $[0,1]$ 界，优化出来的顶点属性可能越界。做法是**截断 + 重新求值**，而不是解带约束的二次规划。法线同理（归一化）。**明确说了这是工程妥协。**
- **性能**：450 MHz Pentium II 上测试台只有 **150–400 faces/sec**（自陈"未为速度设计"）；但基于 GH98 / LT 的数据，**优化实现预期 ~10,000 faces/sec**。

### 结果

| 模型 | 指标 | GH98 | **本篇（全开）** | 慢方法 Hoppe 96 PM |
|---|---|---|---|---|
| mandrill（79,202 → 1,000 面，顶点色） | color rms | 0.086 | **0.054** | 0.056 |
| head（135,133 → 1,500 面） | geometry rms | 0.00135 | **0.00099** | 0.00095 |
| head | color rms | 0.035 | **0.027** | 0.027 |

**结论是"更准且更快"**：精度追平了一个**慢得多的全局优化方法**（Hoppe 96 的 PM，需要反复重投影采样点），而速度快得多。原文的表述是 **"often requiring less than half as many faces for the same rms error"**。

消融的排序也很干净（按贡献递减）：**新度量 > memoryless > 体积保持**。

## Key Contribution

1. **指出 GH98 的度量错误**：$\mathbb R^{3+m}$ 的投影点不等于几何最近点 ⇒ 误差低估。
2. **给出正确形式**：几何投影在 $\mathbb R^3$，属性是投影点上的线性泛函（$n^\top g_j=0$ 保证沿法向不变）。
3. **把存储从二次降到线性**（$11+4m$），且矩阵稀疏 ⇒ 可直接进堆。
4. **wedge 数据结构**：让"属性不连续"成为一等公民，且 $Q_v$ 从不显式存储。
5. **两个增强**：memoryless（忘掉细节）与体积保持（按住汇率），并给出 memoryless 失效机制的解释。
6. **一条边界判断**：顶点属性用几何对应是对的，**纹理坐标要用参数对应**（Cohen 1998），quadric 能否近似"留作未来工作"。

## Why It Works

**因为误差必须住在一个"有对应关系"的空间里。**

QEM 的全部力量来自一个恒等式：$\sum_p (p^\top v)^2 = v^\top Qv$。左边是"到一堆平面的距离平方和"——它之所以有意义，是因为**每个平面都知道自己该被谁量**。抬到 $\mathbb R^{3+m}$ 之后，"平面"变成"超平面"，**它不再对应任何几何实体**：一个点可以离它很远（几何上）却误差很小（属性上）。

Hoppe 的做法是把"该被谁量"这件事**显式化**：先在 $\mathbb R^3$ 里找对应点 $\bar p$，再问"$\bar p$ 处的属性应该是多少"。**对应关系只有一个来源——几何。**

而 wedge 的存在说明：**这个对应关系不必是单值的。** 一个几何点可以对应多个属性值，只要你说清楚"哪些面用哪个"。**这是"属性是场"这个观点的必然推论**：场可以有多层（像多值函数的分支）。

## Limitations

**① 纹理坐标这条他没走。** §8 明写：未来工作应该量**参数误差**（parametric error）而不是几何误差，因为对纹理映射表面，"UV 挪了一点但视觉没变"与"UV 没动但纹理被拉伸"必须被区分开。**这篇的方法对前者会过度惩罚、对后者会漏罚。** 工业界的实际做法是"UV 缝当硬边，不跨缝塌陷"——**回避问题，不是解决问题**。

**② 属性权重仍是手调的。** $\lambda=0.1$（head 的顶点色）、$\lambda=0.02$（表面法线）——**这两个数字没有推导，是实验选的**。而 $\lambda$ 直接决定"几何与属性之间的汇率"（见 Fig 7）。**这是全文最大的软处：度量本身是"正确"的，但两项的配比是拍的。**

**③ 体积保持只按住一维。** 它禁止"用体积换属性"，但不禁止"用平整度换属性"。**汇率被限制，没有被消除。**

**④ 只演示了颜色与法线。** 没有 UV、没有蒙皮权重、没有材质 ID。**"属性"这个词在这里的实际覆盖面比今天窄得多。**

**⑤ 性能数字极弱。** 150–400 faces/sec 是 1999 年的测试台，作者自己说没优化。**这篇的贡献在度量不在实现**——但这也意味着**没有任何人用它跑过百万面资产的端到端时间**。

**⑥ memoryless 的代价没算。** 它要求每次塌陷都重建 quadric（面数级别的重建次数），论文承认"需要更多的 QEM 构造"，但只给了净速率，没给分解。

## Game Development Relevance

**这是今天所有"自动 LOD"按钮下面那一层。**

Simplygon、UE 的 "Generate LOD"、MeshLab 的 Quadric Edge Collapse——只要它处理了**顶点色 / 法线 / 材质边界**（不是只处理几何），底层就是这篇的三件事之一：几何投影上的属性泛函、wedge、或尖锐边 quadric。

**工程上真正该带走的三条：**

1. **"哪些边不许塌"是一个数据结构问题，不是一个参数问题。** 用 wedge / 硬边表把不连续处标出来，比调罚项可靠。**罚项有限，就一定有汇率。**
2. **属性越界要截断重算，不要上约束求解器。** 颜色截到 $[0,1]$、法线重新归一化——1999 年的这个选择今天依然是对的（约束二次规划在每一条候选边上跑不起）。
3. **零曲率区域会病态。** 平面 / 柱面上，$A$ 奇异。退化为"位置取中点、只解属性"是标准处理，**工业实现里经常漏这一步，表现是平面上出现抖动的小三角**。

**验收上的直接对应**（接 [[Mesh Simplification]] 的三查）：这篇解释了"**UV 有没有撕**"这一查为什么最常翻车——因为**几何误差看不见 UV**，而 UV 误差在这个框架里根本没被正确度量（限制 ①）。

## Unreal Engine Relevance

- **UE 的 "Generate LOD" 与 Simplygon 的硬边处理**：UE 用**顶点色 alpha 或材质槽边界**标记"不许焊接"的位置，Simplygon 用 **material boundary / UV seam 保护**。**两者都是 wedge 思路的工程变体**——只不过工业实现通常直接"禁止跨边界塌陷"，比 Hoppe 的"多属性向量同时优化"更保守、更简单、更不容易出错。
- **顶点色 LOD**：UE 的顶点色常用于风摆、AO、遮罩。**这些量的简化误差在几何空间里完全不可见**，正是本篇要解决的问题。实务建议：**顶点色通道若承载功能性数据（不只是颜色），应视为"不可平均"，按硬边处理，而不是交给 QEM 的 $w_c$ 项。**
- **与 [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] 的对照**：UE 的管线本来就是"网格 + 材质（纹理）分离"，**它二十年前就站在 T2-Splat 那一侧**。这篇的价值是告诉你：**如果你不得不用顶点属性（移动端、无纹理资产、程序化材质），代价在哪里。**

## Technology Evolution

见 [[Decoupling Geometry and Appearance]] 与 [[Mesh Simplification]] 的代价模型时间线。

**本篇的位置**：属性进入 QEM 的**第一个正确形式**，也是"属性必须挂在几何对应上"这条原则的出处。

## Relationships

### Based On
- [[Garland 1997 — Surface Simplification Using Quadric Error Metrics|Garland & Heckbert 1997]] —— 几何 QEM（$\sum(p^\top v)^2=v^\top Qv$）
- Garland & Heckbert 1998 —— 被本篇修正的那一步（$\mathbb R^{3+m}$ 超平面）
- Ronfard & Rossignac 1996 —— "到一组平面距离"的直接前身

### Extends
- QEM 的**属性维度**：从 $m=0$ 扩到任意 $m$，且存储线性。
- [[Hoppe 1997 — View-Dependent Refinement of Progressive Meshes|Hoppe 1997]] 的**属性处理**：那篇把判据换到屏幕空间，但对属性只做了"解耦优化"（几何与属性分开最小化），本篇把两者放回同一个可加度量。

### Improves
- GH98 的误差低估问题；存储从 $(4+m)(5+m)/2$ 降到 $11+4m$。

### Related
- [[Mesh Simplification]] —— 本篇是"属性"这一栏的标准答案
- [[Level of Detail]] —— 本篇产出的是一串可截断的塌陷序列，LOD 链的原料
- [[Mesh Parameterization]] —— 限制 ① 的另一半：纹理坐标该用参数误差
- [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] —— **正面对手**：把属性搬出顶点

### Contrasts
- **Cohen / Olano / Manocha 1998（Appearance-preserving）**：用**参数对应**而非几何对应 ⇒ 对纹理映射是正确度量。**两篇不是竞争，是分工**：属性是否定义了参数化，决定了该用哪一套。
- **Lindstrom & Turk 1999**：四面体体积平方和 ≈ 面面积平方加权的 QEM（**这条等价关系由本篇指出**），且他们先提了 memoryless。

### Followed By
- 所有工业简化器的属性处理（Simplygon / UE / MeshLab）
- 2000s 的 texture atlas 趋势——作者本人在主页点明
- 2026 年 [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] 把这条趋势做到极致

## Personal Knowledge State

- **机制层 Easy**：QEM、边塌陷、拉格朗日乘子、wedge 概念——都在日常范围内（[[Garland 1997 — Surface Simplification Using Quadric Error Metrics|Garland 1997]] 昨天刚过）。
- **今天补的三条（Normal）**：
  1. **属性误差必须住在"有对应关系"的空间里**——$\mathbb R^{3+m}$ 的超平面不对应任何几何实体。
  2. **加权求和 = 一个汇率**（Fig 7）：只要两项在一个标量里且权重有限，就存在"用一个换另一个"的解。体积约束是按住汇率，不是取消它。
  3. **不连续要声明，不能靠罚**——wedge 是数据结构，$w_c$ 是价格。
- **仍是 Hard**：纹理坐标的参数误差（限制 ①）。这需要 [[Mesh Parameterization]] 的框架，本轮未展开。

## Learning Value

**最高价值的一条：Fig 7 那张一维折线图。**

它是"**多目标压成一个标量会怎样**"的**最小可复现实验**：五条线段、两个塌陷候选、两个误差分量，就能看到"几何主动变差去换属性"。**这张图的规模小到可以手算**，值得单独记住——以后凡是看到"我们把 A 误差和 B 误差加权求和"，就该想起这张图。

**第二价值**：**$11+4m$ 这个数字的来历**。不是"优化了存储"，是因为**属性的 $m\times m$ 子块必然是单位阵的倍数**（属性泛函彼此正交）。**"正交 ⇒ 稀疏 ⇒ 可加"这条链，是二次型方法能统治工业界的完整理由。**

## Visualization

![[Research/Files/Pictures/Hoppe 1999 — 图2 属性误差在几何投影点上量.png]]

**读图**：$(p,s)$ 是带属性 $s$ 的点，$(\bar p,\bar s)$ 是它在三角形所在**平面上的几何投影**及其在该处的属性。两段误差被分开画：竖线长度 $\|p-\bar p\|$ 是**几何误差**，属性差 $\|s-\bar s\|$ 是**属性误差**。**关键在投影是先做几何、再读属性**——GH98 把这两步合成一步（投到 $\mathbb R^{3+m}$ 的超平面上），于是投影点会横向滑到"属性更近"的位置去，那条竖线就变短了。**这张图就是"低估"两个字的图解。**
（图 2，Hoppe 1999, IEEE VIS '99）

![[Research/Files/Pictures/Hoppe 1999 — 图3 新度量与旧度量的简化对比.png]]

**读图**：同一条 mandrill（79,202 面 → 1,000 面），左是原图，中是 GH98，右是本篇。**看嘴部与鼻梁的红蓝交界**：中间那张的颜色边界已经开始"渗"，右边仍锐利；左半边的线框密度分布也不同——**中图在细节处过度铺面、在平缓处反而不够，右图更接近均匀**。定量是 color rms 0.086 → 0.054（-37%）。
（图 3，Hoppe 1999, IEEE VIS '99）

![[Research/Files/Pictures/Hoppe 1999 — 图7 属性误差让几何主动收缩.png]]

**读图（今天最该看的一张）**：一条折线，五个点，每个点标 $(\text{position},\ \text{attribute})$。属性沿折线走 $1\to0\to1\to0$。
- **上右的塌陷** $V=(2,1,a)$：位置**完全没动**（$Q_p=0$），但属性要**外推**左侧线段，$Q_s=(a+1)^2+2(a-1)^2>0$。
- **下右的塌陷** $V=(1,0,0)$：位置**被拉到曲率中心**（$Q_p=3\cdot1^2>0$），但它在原来三段上的**投影全落在线段内部**，属性可以内插 ⇒ $Q_s=0$。

**两个候选的总代价谁低，取决于权重 $\lambda$。** 这个 1D 例子里，"几何正确"与"属性正确"是两个**互斥**的解——而优化器只会挑总代价低的那个。**这就是"汇率"的全部内容。**
（图 7，Hoppe 1999, IEEE VIS '99）

## Notes

- **存储表（Table 1）**：$m=0$: 10 → 10；$m=3$（颜色）: 28 → **23**；$m=6$（+法线）: 55 → **35**；$m=8$（+纹理坐标）: 78 → **43**。一般式 $(4+m)(5+m)/2$ → $11+4m$。
- **权重**：head 顶点色 $\lambda=0.1$；表面法线 $\lambda=0.02$。**都是实验选的，无推导。**
- **mandrill 的极限行为**：对平面几何，GH98 在 $\lambda\to0$ 时收敛到本篇（$\lambda=0$ 时不行）。**这是两套度量关系的解析证据。**
- **表 2 里有个带 'y' 的数字**（0.071，比相邻的 0.068 差）：作者专门解释——平面网格上体积保持"本应无关"，但"收缩效应 + memoryless 重算"会把几何的数值噪声逐步放大，直到面片明显起伏。**加体积保持后消失。** 这条很实用：**数值噪声会被 memoryless 放大**。
- **PDF 来源**：`hhoppe.com/newqem.pdf` 一次 200（2.5 MB / 8 页）。**作者主页第 7 次胜出**（Reyes 1987 / Jensen 1996 / Lasseter 1987 / Merrell 2007 / Baran 2007 / Garland 1997 / 本篇）。
- **作者主页的 Hindsights 值得单记**：① 后续技术报告证明该 quadric 的最小化可**线性时间**完成；② **"当下的趋势是用纹理图集取代顶点属性"**。第 ② 条是 [[Decoupling Geometry and Appearance]] 这条线的一手证据——**提出者本人认为这条路会走到"属性离开顶点"**。
- 2026-11-09 建页。与 [[2026-11-09-Tang — T2-Splat Adaptive Topology Mesh Splatting with Texture Residuals|T2-Splat]] 同批：**一篇教你怎么把属性塞回去，一篇示范怎么把它拿出来。**
