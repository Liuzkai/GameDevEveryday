---
type: concept
user_level: Normal
aliases: ["Isosurface Extraction", "等值面提取", "表面提取", "Surface Extraction", "Contouring", "体数据转网格"]
prerequisites: ["Radiometry（标量场的含义）", "Delaunay 三角化（受限 Delaunay）", "隐函数 / 零等值面"]
first_introduced: "1987（Lorensen & Cline, Marching Cubes）"
---

# Isosurface Extraction

## Definition

**给定一个标量场 $f:\mathbb R^3\to\mathbb R$ 和一个阈值 $c$，求集合 $\{x:f(x)=c\}$ 的三角形（或四边形/多面体）表示。**

说人话：**把"每点一个数"的体数据，变成"一堆三角形"**。

它在游戏里出现的位置比想象的多：SDF 烘焙成网格、可破坏地形的碎片、体素世界的表面、体积云/雾的边界、扫描资产入库、医学数据可视化。**它是"体数据"与"光栅化管线"之间唯一的桥。**

## Core Principle

### 三个独立的决定（本页的骨架）

任何一个等值面提取算法，都必须在三件事上各做一个选择。**把这三件事混在一起谈，是这一领域最常见的误读**：

| 决定 | 问题 | 可选答案 | 代价 |
|---|---|---|---|
| **① 拓扑从哪来** | 哪些边被穿、怎么连成三角形 | 查表（256 例）/ 受限 Delaunay 的对偶 / 沿边行进 / 学出来的模板 | 查表最便宜；Delaunay 能保流形；学习有泛化风险 |
| **② 顶点位置从哪来** | 交点落在哪 | 棱上线性插值 / QEF 投影 / 任意位置 / **解析求解** | 决定"精度上限"与"三角形质量" |
| **③ 密度从哪来** | 哪里的三角形该多 | 均匀网格 / 曲率 / **到中轴的距离** / 误差驱动 | 决定"三角形数正比于什么" |

[[Lorensen 1987 — Marching Cubes A High Resolution 3D Surface Construction Algorithm|Marching Cubes 1987]] 的历史贡献，正是**把这三件事拆开**（拓扑 = 14 模式表；位置 = 棱上线性插值；法线 = 体数据梯度）。而它四十年的所有问题，也全部来自这三个选择的组合后果。

### 误差的三个来源（对应上面三个决定）

| 误差 | 来自哪个决定 | 表现 | 谁能修 |
|---|---|---|---|
| **离散化误差** | ② 顶点被钉在格边上 | 曲面被"棱"量化，折痕处阶梯 | 换位置策略（DC 的 QEF / FOMOS 的任意顶点） |
| **拓扑误差** | ① 查表在相邻面之间不自洽 | 孔洞、非流形边、亏格错 | 换拓扑来源（MC33 / Delaunay / 学习） |
| **密度误差** | ③ 均匀密度 | 平坦处冗余、薄处缺失 | 换密度判据（lfs / 误差） |

**三者不能互相抵消。** 常见错误是"把网格加密"去修拓扑错误——MC 在 32³→512³ 的拓扑正确率是 68/84/81/68/**60**（[[2026-11-17-Ohrhallinger — FOMOS Feature-Oriented Meshing of SDFs|FOMOS]] 表 10）：**分辨率升高，拓扑反而更差**，因为薄结构被切得更碎、连接更容易接错。

### 那条决定一切的比例关系

$$\text{MC：三角形数}\;\propto\;\text{表面积}\qquad\text{vs}\qquad\text{特征自适应：三角形数}\;\propto\;\frac{1}{\mathrm{lfs}^2}$$

左边是"均匀采样"的必然结果——**每个单位面积上放同样多的三角形**，不管那块面积是平的还是折的。右边是"按特征尺寸采样"——**每个"特征"上放同样多的三角形**。

这一个比例关系的替换，是 1987 与 2026 之间真正的分水岭。**"更少的三角形"本身不是目的，改变这个比例才是。**

## Prerequisites

- **隐函数与零等值面**：$f(x)=0$ 的集合是一个曲面；SDF 的 $f$ 还额外满足 $|\nabla f|=1$。
- **Delaunay 三角化 / 受限 Delaunay**：三维 Delaunay 的对偶是 Voronoi；"受限"指只保留那些对偶 Voronoi 边与曲面相交的三角形——这是保拓扑网格化的核心工具。
- **插值的阶与仿射性**（读 TetraCoder 才需要）：三线性插值**不是**分片仿射，四面体内的重心插值**是**。这个区别决定"能不能解析求解等值面"。
- **中轴（medial axis）与局部特征尺寸 lfs**（读 FOMOS 才需要）：lfs$(p)$ = $p$ 到中轴的最近距离。

## Historical Evolution

```
1967  Blum            中轴（medial axis）概念——为 lfs 埋下定义
1975  Keppel          轮廓三角化（层间连通性靠猜）
1977  Fuchs           最优轮廓拼接（仍是"先有轮廓"）
───────────── 1987 断层 ─────────────
1987  Lorensen/Cline  Marching Cubes：8 顶点状态 → 256 例 → 14 模式 + 棱上插值
1995  Chernyaev       MC33：14 → 33 例，消歧
2002  Ju et al.       Dual Contouring：顶点可离开棱（QEF），保尖锐特征
2018+ 可微等值面      Deep MC / MeshSDF / FlexiCubes：把提取嵌进优化回路
2021  Chen & Zhang    Neural Marching Cubes：把模板学出来
2022  Chen et al.     Neural Dual Contouring
2023  Sellán          RFTS：网格包裹 + 梯度流（拓扑不能变）
2024  Sellán          RFTA：能处理非零亏格
2026  Ohrhallinger    FOMOS：密度 ∝ 1/lfs²，输出敏感，水密 2-流形
2026  Della Pietra    TetraCoder：让场分片仿射 ⇒ 提取变解析
```

**四个世代，一条主线**：

| 世代 | 代表 | 换掉了哪个决定 | 遗留问题 |
|---|---|---|---|
| 一（1987–2000s） | MC / MC33 / DC | 拓扑表、位置 | 密度仍是均匀的 |
| 二（2021–2024） | Neural MC / NDC / RFTS / RFTA | 拓扑来源（学 / 包裹） | 泛化、拓扑限制、三角形质量 |
| 三（2026a） | [[2026-11-17-Ohrhallinger — FOMOS Feature-Oriented Meshing of SDFs\|FOMOS]] | **密度**（∝ 1/lfs²） | 仍在"采样"范式里；尖锐特征要后处理 |
| 四（2026b） | [[2026-11-17-Della Pietra — TetraCoder Adaptive Tetrahedral Encoding for Neural Fields\|TetraCoder]] | **场本身**（分片仿射） | 只对受限函数类成立；提取成本跟激活胞数走 |

## Important Papers

| 论文 | 年 | 在本页的位置 |
|---|---|---|
| [[Lorensen 1987 — Marching Cubes A High Resolution 3D Surface Construction Algorithm\|Marching Cubes]] | 1987 | **定义者**。三个决定的原始拆分；"三角形 ∝ 表面积"这句原话是此后四十年的靶子 |
| [[2026-11-17-Ohrhallinger — FOMOS Feature-Oriented Meshing of SDFs\|FOMOS]] | 2026 | **第三代**。把密度换成 1/lfs²，复杂度从 $O(n^3)$ 降到输出敏感；水密 / 2-流形 / 同胚 |
| [[2026-11-17-Della Pietra — TetraCoder Adaptive Tetrahedral Encoding for Neural Fields\|TetraCoder]] | 2026 | **第四代**。重心插值 ⇒ 场分片仿射 ⇒ 等值面变线性方程 |
| Dual Contouring（Ju et al.） | 2002 | **位置那一半的自由度**。QEF 让顶点离开棱。⚠️ 本库只有一篇变体（DC-SQEM），**无原文** |
| MC33（Chernyaev） | 1995 | **歧义的解**。⚠️ 本库无原文 |

## Related Concepts

- [[Sparse Volumetric Representation]]：等值面提取的**输入端**形态（TSDF / NanoVDB / 稀疏块）。稀疏化直接决定"能提取多大的东西"。
- [[Mesh Simplification]]：**同一枚硬币的另一面**——简化是"事后删自由度"，网格化是"事前分配自由度"。两者的评价指标不同（简化看误差-面数曲线，网格化看三角形质量与拓扑）。
- [[Level of Detail]]：等值面提取的 $\varepsilon$ 是一条天然的 LOD 链（特征相对的分辨率旋钮）。
- [[Radiance Fields]]：神经场是新一代的输入端；**"场属于哪个函数类"决定了它能不能被解析提取**。
- [[Microgeometry]]：同一个"编译边界"问题在另一种尺度上的版本——能编译的是局部的，不能编译的是非局部的。
- [[Subdivision Surfaces]]：与它互补的"离散 → 连续 → 更细的离散"路径。

## Technologies

- **Marching Cubes 及其变体**（surface nets / dual contouring / marching tetrahedra）——引擎与 DCC 里的默认实现。
- **OpenVDB / NanoVDB** 的 `volumeToMesh`（基于 MC 与 DC 的混合）。
- **CGAL 3D Mesh Generation**（受限 Delaunay 内核，FOMOS 用的就是它）。
- **Geometry Script / Houdini VDB**——DCC 侧的入口。
- 神经场侧的解析提取（TetraCoder 一系）——尚未产品化。

## Game Applications

| 用途 | 为什么必须用它 |
|---|---|
| **可破坏地形 / 碎裂** | 破坏后是"体数据"，要变回可渲染的网格 |
| **体素世界表面** | Minecraft 系全部依赖它（或其变体） |
| **SDF 资产 → 网格** | 虚拟几何的 fallback、离线烘焙、跨引擎搬运 |
| **体积模拟的内域网格** | FEM/CFD 要四面体，要水密、要高质量三角形 |
| **扫描 / 医学 / 摄影测量资产入库** | 输入天然是体数据 |
| **交互雕刻的预览** | 每改一笔就要重提取一次，成本直接决定手感 |

## Personal Knowledge

**Normal。** 三条：
1. MC 的机制（8 位索引 / 14 模式 / 线性插值 / 中心差分）是 Easy——本科内容，也是写 shader 的常识。
2. **"三个决定"这个拆法是 Normal 的核心**：它让你在读到任何新方法时能立刻定位"它换了哪一个"。
3. **"三角形 ∝ 表面积 vs ∝ 1/lfs²"这条判据是 Normal 的上限**——它是判断"这个自适应值不值"的唯一入口。

## Learning Gap

1. **Dual Contouring 的 QEF 没有原文。** 本库只有 DC-SQEM 一篇变体（2026-11-05）。**"顶点位置"那一半的自由度（QEF 最小化什么、奇异时怎么退化）在本库是空的**——而这正是 90% 生产用 DC 实现的核心。
2. **歧义判据完全空缺。** MC33 的 33 例怎么分类、asymptotic decider 判的是什么，本库零覆盖。**"为什么 MC 会在相邻面上打架"这个问题，目前只能靠推理，没有原文支撑。**
3. **没有一条"三角形数 vs 画质"的实测曲线。** FOMOS 给了 CD / F-Score / 三角形质量的**点**，但没有给"同一资产、不同三角形预算下，渲染出来差多少"。**这意味着"自适应网格化值不值"这个问题在本库仍是未答。**
4. **GPU 侧空白。** [[GPU-Driven Rendering]] 那一页讲的是网格进 GPU 之后的事，**"网格怎么在 GPU 上生成"（compute 版 MC / DC）本库一页没有**。
5. **解析提取的成本模型不清楚**：TetraCoder 的提取时间跟"ReLU 激活胞数"走，这个量与分辨率、与场景复杂度是什么关系，原文没给。

## Next Step

**做一次三个点的实测**（一个下午能做完）：拿一个真实 SDF 资产（UE 的 Mesh Distance Field，或一个 NanoVDB），分别在

1. **均匀 MC**（$\varepsilon$ 固定），
2. **特征自适应**（按 FOMOS 的规则手工实现一个简化版，或直接用 OpenVDB 的 adaptive meshing），
3. **两者三角形数对齐后**

量三件事：**三角形数 / 提取时间 / 渲染帧时间 + 一张并排截图**。目标不是复现论文，是拿到本库第一条"三角形预算 → 画质"的真实曲线，把 Learning Gap 第 3 条填掉。

## Notes

- 建页依据（2026-11-17）：全库 `grep "Marching Cubes"` = **0 命中**——**这是本库最反常的一个空洞**：一个所有引擎都实现过、所有 SDF 论文都拿它当基线的算法，在 221 篇论文、67 个概念里一次都没被提到。而 2026-11-17 一天之内三篇论文（MC 1987 / FOMOS / TetraCoder）全部落在它上面。
- 与 [[Mesh Simplification]] 的分工：那一页讲"给定网格怎么减"，本页讲"给定场怎么生成"。两者共享"自由度分配"的语言，但方向相反。
