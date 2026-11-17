---
type: paper
title: "FOMOS: Feature-Oriented Meshing of SDFs"
authors: ["Stefan Ohrhallinger", "Amal Dev Parakkat"]
year: 2026
published: "2026-11-12"
venue: "arXiv preprint（ACM 期刊模板，16 页）"
url: "https://arxiv.org/abs/2611.17458"
code: "接收后放出（原文摘要末句）"
project_page: ""
category: ["Geometry Processing", "Isosurface Extraction", "Meshing", "SDF", "Medial Axis"]
importance: A-
historical_importance: 2
game_relevance: 4
production_readiness: "Prototype（CGAL 依赖、CPU only、代码未放出）"
user_level: Normal
status: unread
---

# FOMOS: Feature-Oriented Meshing of SDFs

## TL;DR

**它换掉的不是三角形数量，是"三角形数量正比于什么"这个比例关系。**

[[Lorensen 1987 — Marching Cubes A High Resolution 3D Surface Construction Algorithm|Marching Cubes]] 的原话是"三角形数正比于表面积"。FOMOS 把它换成：

$$\text{三角形数} \;\propto\; \frac{1}{\mathrm{lfs}^2},\qquad \mathrm{lfs}(p)=\text{到中轴的距离}$$

`lfs`（local feature size）同时吃到**曲率**与**厚度**：曲面卷得急，它小；两层曲面挨得近，它也小。于是

| | 密度由谁决定 | 平坦大面上 | 薄壁处 |
|---|---|---|---|
| MC 1987 | 网格步长（常数） | 一堆多余三角形 | 采不到 |
| 曲率自适应 | 曲率 | 少（对） | **仍然采不到**（薄壁可以不弯） |
| **FOMOS** | **到中轴的距离** | 少（对） | **自动加密**（对） |

一句话：**"哪里需要密"这个问题的答案，不在曲面上，在曲面之外的那条中轴上。**

## Problem

SDF 是连续表示，但下游（渲染、仿真、DCC）要网格。从 SDF 拿网格有四条老路，每条都有一处结构性缺陷：

| 路线 | 代表 | 缺陷 |
|---|---|---|
| 均匀网格 + 查表 | [[Lorensen 1987 — Marching Cubes A High Resolution 3D Surface Construction Algorithm\|Marching Cubes]] | 密度是常数 ⇒ 三角形数 ∝ 表面积；三角形质量不受控（缺陷率 11.86%） |
| 网格 + 对偶顶点 | Dual Contouring 系 | 尖锐特征好，但仍是均匀采样驱动 |
| 学习模板 | Neural MC / NDC | 泛化有风险；在分布外几何上产生非流形（原文图 1 直接标红） |
| 网格外包裹 + 梯度流 | RFTS / RFTA（Sellán 2023/24） | 拓扑不能在过程中改变；RFTA 三角形质量差、密集 |

**共同点：它们都在"曲面之上"决定采样密度。** 而"这块曲面有多重要"这件事，是**非局部**的——手掌上一块平坦的皮，它的重要性由"它离另一侧的手指有多远"决定，这个信息不在曲面局部。

这就是 `lfs` 的用武之地：它定义为**曲面上一点到中轴的最近距离**，天然把"曲率"（局部）与"厚度"（非局部）合成一个标量。

## Historical Context

这条线其实很老，只是从没被端到端接起来过：

```
1993  Ruppert          lfs 定义（网格生成理论）
1998  Amenta & Bern    ε-sampling：|p,s| < ε·lfs(p) ⇒ 重建保拓扑
2005  Boissonnat-Oudot 受限 Delaunay + 细化（本文的网格化内核）
      ↑ 但这两条要求"能精确求 lfs"——一般形状求不出来，只能对代数曲面
1967  Blum             中轴（medial axis）概念
2001  Amenta et al.    PowerCrust：从点云求中轴（慢、只对极光滑采样）
2012  Ma et al.        收缩球求中轴（要求已知最近点）
2022-26                中轴简化/剪枝一大批（MATFP / VMAS / MATTopo / Structural MAT）
───────────── 2026 ─────────────
      FOMOS：直接从 SDF 求"中轴脚手架" ⇒ 估 lfs ⇒ 喂给 2005 的内核
```

**它的贡献不是任何单个零件，是把 1998 的理论、2005 的内核、1967 的概念，用一条"SDF 直接给中轴"的捷径接通了。**

## Previous Work

按论文的两分法：

**A. 求中轴**——传统做法是从点云/网格求 Voronoi 顶点（Amenta 2001 / Boissonnat-Cazals 2001），理论上完美（采样趋于无穷时收敛），实践上"produce a clean medial axis from real-world inputs"做不到，于是有一整条剪枝/简化支线（Amenta-Kolluri 2001、Dey-Zhao 2004、MATFP 2022、VMAS 2024、MATTopo 2024、Structural MAT 2026）。另一条是收缩球（Ma 2012 / Jalba 2012 / Dou 2022），但**要求最近点已知**。还有一条是深度学习预测中轴（Lin 2021 / Rebain 2019 / Kong 2026）。

**B. SDF 网格化**——MC 及其改进（消歧、保特征、省内存）、可微等值面（Deep MC / MeshSDF / FlexiCubes）、包裹式（RFTS / RFTA）、自适应采样（McGrids / 幂图提取 / MESC）。

**论文对 A 的关键判断**：这些方法都要求输入是**网格或点云**，而 SDF 没有显式拓扑、在尖锐特征附近梯度敏感——**"从 SDF 求中轴"这条路是空的**。它填的就是这个空。

## Core Idea

### 三步，共享一个便宜的原语

$$\underbrace{\text{足点采样}}_{\text{把格点投到曲面}}\;\to\;\underbrace{\text{中轴脚手架}}_{\text{切球二分}}\;\to\;\underbrace{\varepsilon\text{-采样网格化}}_{\text{受限 Delaunay 细化}}$$

三步都只用一个原语：**SDF 的梯度给出的最近点投影**

$$x = c-\phi(c)\,\nabla\phi(c)$$

这一条式子让"球是否为空"变成一次 SDF 求值（$|\phi(c)| < r$ 就是在球内碰到了曲面），于是**不需要显式几何，也不需要已知最近点**——这正是收缩球方法做不到的。

### 切球二分：为什么不是"从大往小收缩"

经典收缩球要"已知最近点 + 迭代缩小"。这里已知的只有**距离**，最近点要靠梯度投影算，而投影在格点离散下不准。论文的做法反过来：**从一个上界半径开始二分**。

- 上界 = 包围盒对角线的一半（保证球是空的）；
- 每次取中点 $r$，球心 $c = p + r v$（$v$ = 法线方向，取正反两侧）；
- 若 $|\phi(c)| < r$ 且最近点离 $p$ 足够远（$|p,q| \ge d$，$d$ = 单元格对角线）⇒ 球在别处碰到了曲面 ⇒ **缩小上界**；
- 否则 ⇒ 球是空的，或碰到的是 $p$ 自己 ⇒ **放大下界**。

收敛到 $|r - |\phi(c)|| < \mu$ 或区间宽度 $< \mu$，**6–10 次迭代**。取内外两个球里**较小的那个**作为该足点的中轴球。

> 为什么取小的：`lfs` 是"到最近中轴点的距离"，内外两个中轴球哪个更近，哪个就决定了这一点的特征尺寸。薄壁处外侧球会很大（穿出物体），内侧球才是约束。

### 网格化：只换细化判据

内核是 Boissonnat–Oudot 2005（受限 Delaunay + 逐三角形检查）。它原本的判据是"边长小于阈值"或"能精确算 lfs 的代数曲面"。FOMOS 把判据换成

$$\varepsilon_t=\frac{r_t}{\widehat{\mathrm{lfs}}(p_t)}$$

其中 $r_t$ 是受限 Delaunay 三角形对偶 Voronoi 边的半径（表面 Delaunay 球半径），$p_t$ 是该边与零等值面的交点。**$\varepsilon_t > \varepsilon$ 就插入 $p_t$ 并局部重三角化。**

因为只换了判据，Boissonnat–Oudot 的**全部保证被继承**：水密、2-流形、与目标曲面同胚、必然终止（下界取"与单元格尺寸成比例的最小边长"）。

### 理论：这条判据真的成立吗

这是全文最硬的一段（§3.6）。要证的是"只在细化点检查 ε，全曲面都满足 ε"。

- **Lemma 1（Lipschitz）**：$\mathrm{lfs}(p)\le \mathrm{lfs}(q)+|p,q|$——lfs 变化不快于距离。
- **Lemma 2（法线变化）**：$|p,q|\le \rho\,\mathrm{lfs}(p)$、$\rho<1/3$ ⇒ 法线夹角 $\le \rho/(1-3\rho)$。
- **Lemma 3（细化点）**：终止时每个 $t$ 满足 $\delta(p_t)=r_t<\tilde\varepsilon\,\mathrm{lfs}(p_t)$，其中 $\tilde\varepsilon=(1+\alpha)\varepsilon$（$\alpha$ 是 lfs 估计的相对误差）。
- **Lemma 4（三角形内最近顶点）**：$\forall x\in t:\ \min_i|x,v_i|\le R_t\le r_t$——**钝角三角形不需要特殊处理**（这是最容易出漏洞的地方，作者专门证了）。
- **Theorem 1（全局 ε-采样）**：令 $\kappa=\max_t\max_{x\in t}|x,\pi(x)|/r_t$，若 $(1+\kappa)\tilde\varepsilon<1$，则全曲面被 $\varepsilon'=\frac{(1+\kappa)\tilde\varepsilon}{1-(1+\kappa)\tilde\varepsilon}$ 采样。
- **Corollary 1**：无条件 $\kappa\le1$ ⇒ $\varepsilon'\le 2\tilde\varepsilon/(1-2\tilde\varepsilon)$；若曲面在每个三角形上方局部平坦，$\kappa=O(\tilde\varepsilon)$ ⇒ $\varepsilon'\approx\tilde\varepsilon$。

**读法**：$\kappa$ 度量"三角形到曲面的偏离 / 球半径"。它把"离散化误差"显式地折算进了 ε。**这是本文与所有"经验阈值"方法的真正分界线——它知道自己的 ε 会被打折多少。**

## Technical Approach

### 复杂度：从立方到平方

关键不在算法本身，在**所有操作都在表面体素上，不在体积上**：

- 足点只取 $|\phi(g)|<d$ 的格点（即含曲面的那些 cell 的角点）；
- 中轴球只从这些足点长出来；
- 因此时间与空间从 $O(n^3)$ 降到 $O(m)$，$m$ = 输出三角形数。

**"输出敏感"（output-sensitive）是这一节的关键词**：不是"跟网格分辨率走"，是"跟你要多少三角形走"。

### 数字（全部取自原文）

| 项 | 数值 |
|---|---|
| 足点投影迭代 | **2.5–3.5 次**，精度 $\mu=10^{-2}d$，$d=\sqrt3\lvert e\rvert$ |
| 切球二分 | **6–10 次**（覆盖 $16^3$–$1024^3$） |
| 预计算梯度 | 时间**减半**，内存 **×4**（可关） |
| Buddha $128^3$，$\varepsilon{=}0.3$ | MC 输出 **187k** 顶点 vs FOMOS **16k**（约 **11.7×**） |
| 512³ 中轴计算多线程 | 18.99 s → **2.96 s**（64 核，6.41×，−84.4%） |
| Emu $1481{\times}2609{\times}1843$（k=32） | 中轴 0.02 s + 网格 0.88 s = **0.90 s** / 1345 MB |
| Thai $10240^3$（k=32） | 0.21 + 3.50 = **3.71 s** / 14.8 GB |
| Thai $10240^3$（k=1，全密度） | 30,863 + 888 = **31,752 s** / 61.5 GB |

最后一行是全篇最有信息量的数字：**同一个 $10240^3$ 的 TSDF，k 从 1 到 32，总时间从 8.8 小时降到 3.7 秒（约 8,560×），内存从 61.5 GB 降到 14.8 GB——而网格用的是完整 TSDF。** 省掉的只是"中轴点的数量"，不是"场的精度"。

### 质量（对照表，最该背的三行）

| 指标 | MC | NDC | McGrids | **FOMOS** |
|---|---|---|---|---|
| 缺陷三角形占比 | **11.86%** | 0.68% | 17.85% | **0.00%** |
| 最小角 | **0.02°** | 2.23° | 0.006° | **27.1°** |
| 最大角 | 178.18° | 174.13° | 179.39° | **122.6°** |
| 全局拓扑正确率（512³） | 60% | × | 97.54% | **100%** |

（缺陷率 / 角度取自 Table 9；拓扑取自 Table 10，MC 在 32³→512³ 是 68/84/81/68/60——**分辨率升高，MC 的拓扑正确率反而下降**，这是均匀网格在薄结构上"越细越容易连错"的直接证据。）

## Key Contribution

1. **"从 SDF 直接求中轴脚手架"**——不需要显式拓扑、不需要已知最近点，用梯度投影 + 切球二分，6–10 次迭代一个球。
2. **把 lfs 从"理论定义"变成"可算的量"**，从而让 1998 的 ε-sampling 第一次能用在任意 SDF 上。
3. **网格化的复杂度从 $O(n^3)$ 降到 $O(m)$**（输出敏感），并给出 $10240^3$ 这种规模的实测。
4. **无条件的水密 / 2-流形 / 同胚**，且**缺陷三角形 0%**——这两条是它相对 RFTS/RFTA/NDC 的真正优势。

## Why It Works

因为它把"哪里要密"这个**几何问题**，换成了一个**距离问题**。

曲率是二阶量，噪声敏感、且对薄结构不敏感（薄壁可以不弯）。`lfs` 是一阶量（到中轴的距离），对两者都敏感，而且**光滑**（Lipschitz 常数 1）——光滑这件事至关重要：它保证"相邻两点的采样密度不会突变"，而这正是 Delaunay 细化能收敛、能保拓扑的前提。

> **判据必须是光滑的，否则细化会在判据的跳变处来回震荡。** 这条经验对任何自适应细分都成立（LOD 的误差判据、纹理的 mip 选择、粒子的密度控制）。

## Limitations

### 论文自陈的四条

1. **不适合极低分辨率输入**（32³ 时比 MC 差）。
2. **尖锐特征仍有锯齿**：Delaunay 属性导致折痕处产生阶梯，**需要后处理**（提到 EdgeSharpener / Attene 2003）——**但论文没有跑这一步**。
3. **薄特征仍要足够高的网格分辨率**（图 16：不足采样时薄结构断开或消失）。
4. **未来工作**：用三二次/三三次插值加速切球、支持神经 SDF、GPU 并行。

### 读原文才发现的三条

5. 🔴 **对 MC 的"精度优势"不成立。** 摘要说自己在"reconstructed result accuracy"上有 distinct strengths，但那是相对 NDC / RFTS / RFTA / MESC 说的。**对 MC，它在多数精度指标上仍略输**：

   | 512³ | MC | FOMOS（ε=0.1） |
   |---|---|---|
   | Chamfer（Table 3） | **0.000017** | 0.000023 |
   | Mean distance（Table 4） | **0.00217** | 0.00226 |
   | F-Score（Table 7） | **99.615** | 98.970 |
   | Normal consistency（Table 8） | **0.99824** | 0.99481 |

   **它赢的是三角形质量、拓扑、规模与复杂度，不是几何误差。** 这个区分对整个领域都重要：**"网格更好"和"离曲面更近"是两个不同的评价轴**，而 MC 在后者上四十年没输过。

6. 🔴 **正文与自己的表打架。** Table 1 里 FOMOS 512³ 是 **CD $6.1\times10^{-4}$ / RMSE $0.00082$**，正文却写"lowest CD ($7.5\times10^{-4}$) and RMSE (0.00104)… improving upon Structural MAT by approximately 20.2% and 14.8%"。而 $7.5\times10^{-4}$ **大于** Structural MAT 的 $7.29\times10^{-4}$（按正文的数字它并不 lowest）；按表算的改进是 **16.3%**，不是 20.2%。**数字对不上，且方向相反。**

7. **"单参数"是宣传。** 实际有三个旋钮：$\varepsilon$、输入网格分辨率、以及大输入下的 $k$（每 $k$ 个 cell 取一个中轴点）。三者都影响时间/内存/质量，论文的 Table 2/11 就是在扫这三者。

8. **中轴精度的验证对象是理想光滑网格**：与 PowerCrust 对比时，模型被 Loop 细分到 level 6（≈1M 三角形，平均边长 ≈ 单元格直径）。**真实资产不是 Loop 光滑的**，折痕与 UV 接缝会让"中轴脚手架"退化。

9. **基线全是研究代码**：没有 OpenVDB / libigl / DC-QEF 这类生产级网格化器。而 CGAL 的三维 Delaunay 在每次插入/剪枝时**全局重建**，这个成本被埋在总时间里，论文没有单独报。

## Game Development Relevance

| 场景 | 用哪一条 |
|---|---|
| **SDF 资产烘焙成网格**（破坏碎片、扫描资产、体纹理） | 三角形质量直接可用（0% 缺陷），不需要再 remesh |
| **交互式雕刻 / 体素编辑的即时预览** | $k$ 子采样中轴：$10240^3$ 在 **3.7 秒**内出网格（k=32） |
| **体积仿真（FEM/CFD）的内域网格** | ε-采样保证四面体质量；论文用它跑了 FEM 演示 |
| **LOD** | **$\varepsilon$ 天生是一个"特征相对"的分辨率旋钮**（见下） |

**$\varepsilon$ 这件事值得单独说。** [[Level of Detail]] 那一页的结论是"判据必须相对某个量，不能是绝对量"。FOMOS 的 $\varepsilon$ 是**相对于局部特征尺寸**的：$\varepsilon=0.8$ 时三角形数掉一个数量级，但"每个特征上有几个三角形"不变。**这是一个可以直接抄进 LOD 系统的设计**——同一份 SDF，一份 $\varepsilon$ 序列就是一条 LOD 链，而且档与档之间的画质差异是**可预期**的（不是"高模/低模"那种黑箱）。

## Unreal Engine Relevance

- **没有直接对应系统**。UE 的 Geometry Script 没有 SDF→网格；Nanite 走的是网格→簇，不是 SDF。
- **可对接的位置**：① 自定义 `UGeometryScriptLibrary` 函数（CPU，CGAL 依赖重，只能做离线烘焙）；② 把中轴计算搬进 compute shader（论文说"应该能并行得很好"，**未实现**）。
- **更现实的用法是反向验证**：用 FOMOS 把 UE 的 Mesh Distance Field 重新网格化，与原始网格比 CD——**这是检查 SDF 烘焙质量最直接的一条路**（本库此前没有工具做这件事）。

## Technology Evolution

| 年 | 工作 | 决定密度的量 | 顶点能去哪 |
|---|---|---|---|
| 1987 | MC | 网格步长 | 棱上 |
| 2002 | Dual Contouring | 网格步长 | 单元格内（QEF） |
| 2023 | RFTS | 初始网格的拓扑 | 曲面附近（包裹） |
| 2024 | RFTA | 同 RFTS + 弧线 | 同上 |
| 2026 | **FOMOS** | **到中轴的距离** | **任意（Delaunay 顶点）** |
| 2026 | [[2026-11-17-Della Pietra — TetraCoder Adaptive Tetrahedral Encoding for Neural Fields\|TetraCoder]] | 四面体的 ReLU 单元数 | 场本身分片仿射（不需要"顶点"） |

**两条出路**：让顶点自由（FOMOS），或者让场可解析（TetraCoder）。**前者仍在"采样"范式里，后者跳出去了。**

## Relationships

### Based On

- [[Lorensen 1987 — Marching Cubes A High Resolution 3D Surface Construction Algorithm|Marching Cubes 1987]]（作为对手与基线，也是"密度=常数"这个前提的来源）
- Boissonnat & Oudot 2005（受限 Delaunay 网格化内核）、Chew 1993（其源头）
- Amenta & Bern 1998（ε-sampling 理论）、Ruppert 1993（lfs 定义）、Blum 1967（中轴）

### Extends

- [[Mesh Simplification]] 的 QEM 思路：**它把"该删哪些三角形"换成了"该在哪生成三角形"**——同一个精度预算，一个在事后删，一个在事前分配。

### Related

- [[Sparse Volumetric Representation]]：TSDF 是其 $10240^3$ 实验的输入形态。
- [[Level of Detail]]：$\varepsilon$ 是一条天然的、特征相对的 LOD 链。
- [[2026-11-17-Della Pietra — TetraCoder Adaptive Tetrahedral Encoding for Neural Fields|TetraCoder]]：同一天的另一条路——不问"在哪采样"，问"场能不能直接解"。

### Followed By

- 未知（2026-11 的 preprint）。作者列的 future work：三二次插值加速、神经 SDF、GPU。

## Personal Knowledge State

**Normal。** 理论段（Theorem 1 / $\kappa$ 的折算）建议只记结论——**"ε 会被 $\kappa$ 打折"**，不必推。真正要记的是三条判据（见 Learning Value）。

## Learning Value

1. **判据的自变量决定了算法的上限。** 曲率（二阶、局部）与 lfs（一阶、非局部）的差别，就是"薄壁能不能被采到"的差别。**问一个自适应算法好不好，先问它的判据吃到了哪些信息。**
2. **"输出敏感"是可以设计出来的**：把所有操作限制在表面体素上，复杂度就从 $O(n^3)$ 变成 $O(m)$。这与 [[Sparse Volumetric Representation]] 的"稀疏度与跳数是一对直接交换"是同一条原理的两面。
3. **"网格更好"和"离曲面更近"是两个轴。** MC 在后一个轴上四十年没输——**这是评审任何"新网格化方法"时最该先分开的一件事**。
4. **一个可抄的工程手法**：中轴可以**稀疏采样**（每 $k$ 个 cell 一个），而网格仍用完整场——**估密度的量可以比被密化的对象便宜得多**。这条在 VFX 里就是"用低分辨率的速度场驱动高分辨率的粒子"。

## Visualization

### 图 1 — 六种方法在同一 $128^3$ 网格上的对照

![[Research/Files/Pictures/Ohrhallinger 2026 — 图1 六种SDF网格化方法的对照.png]]

**读图**：从左到右 MC / NDC / RFTS / RFTA / MESC / FOMOS（最后两列是 $\varepsilon=0.3$ 与 $0.1$）。红色框标出各方法的失效处：MC 的三角形又碎又乱，NDC 在分布外几何上出现非流形，RFTS 丢细节，RFTA 密集且三角形差，MESC 慢且差。**FOMOS 的两列三角形明显更"均匀"——不是更密，是大小更一致**，因为密度跟着 lfs 走，而 lfs 在相邻区域变化不超过距离（Lipschitz）。
（出处：Ohrhallinger & Parakkat 2026, Fig. 1；原文标注该组实验的跨度：1.05 s–2272 s、119 MB–42563 MB、缺陷三角形 0.0–12.6%）

### 图 2 — ε-采样的细化过程

![[Research/Files/Pictures/Ohrhallinger 2026 — 图2 采样细化的五步演化.png]]

**读图**：最左是曲面（Groundtruth），右边四格是从"几个三角形"逐步细化到满足 $\varepsilon=0.2$ 的过程。**注意它不是在均匀加密**：耳朵、腿部这些曲率大或厚度小的地方先长出三角形，背部大片平坦区域到最后仍然很稀。**这就是"三角形数 ∝ 1/lfs²"的可视化。**
（出处：Ohrhallinger & Parakkat 2026, Fig. 2）

### 图 5 — 中轴脚手架与 PowerCrust 的对照

![[Research/Files/Pictures/Ohrhallinger 2026 — 图5 手模型的内中轴与PowerCrust对照.png]]

**读图**：左一是被 Loop 细分到 6 级的手（≈2M 顶点，只作为中轴的"真值"参照），中间两格分别是 PowerCrust 的中轴与 FOMOS 的中轴，最右是两者的重合。**五指之间的空隙是最难的地方**——那里 lfs 极小，"薄"与"弯"同时发生。重合图显示两者在手掌与指尖一致，差异主要在指缝深处。
（出处：Ohrhallinger & Parakkat 2026, Fig. 5）

### 图 11 — 三角形质量分布

![[Research/Files/Pictures/Ohrhallinger 2026 — 图11 三角形质量分布对照.png]]

**读图**：三张图分别是形状比 $Q_{shape}$、最小角、长宽比 $Q_{aspect}$ 的分布（纵轴是对数计数的绝对三角形数）。**看最左那张的 MC 曲线（红色）在低质量端翘起来**——那 11.86% 的缺陷三角形全在这里；FOMOS（青色）是唯一一条整体靠右的曲线。右边那张的横轴到 10 就截断了（$\ge10$ 全归到 10）。
（出处：Ohrhallinger & Parakkat 2026, Fig. 11）

## Notes

- arXiv `2611.17458v1`，提交 2026-11-12，16 页。ACM 期刊模板（含 CCS Concepts 与作者联系方式），未见接收信息。
- 全部数字取自原文 Table 1–11 与正文；**Table 1 的正文/表格矛盾已在 Limitations 第 6 条标出**，笔记里用的是表格值。
- 中轴精度分析（§3.7）用了四个对象：椭球（解析真值）、四个模型的分辨率收敛、与 PowerCrust 的 Hausdorff、Pieta 的 MAT-曲面距离。**这些证据都指向"smooth 曲面"，不覆盖尖锐特征**。
