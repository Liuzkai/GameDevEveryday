---
type: paper
title: "Evaluation of Loop Subdivision Surfaces"
authors: [Jos Stam]
year: 1998
published: "1998-07（SIGGRAPH '98 CD-ROM Proceedings）"
venue: "SIGGRAPH '98 CD-ROM Proceedings（引用口径经 Google Scholar 与 ACM 论文引用列表双重核对：「Stam J. Evaluation of Loop subdivision surfaces. In: ACM SIGGRAPH '98 CDROM proceedings; 1998」）；署名 Alias|wavefront, Inc.（与 [[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|Stam 1998 四边形版]] 同单位、同月）"
url: "https://www.dgp.toronto.edu/people/stam/reality/Research/pdf/loop.pdf"
code: "lptest.c（作者主页 SubdivEval 页可下载；特征结构数据 lpdata50.dat——至价 N=50）"
project_page: "http://www.dgp.toronto.edu/~stam/reality/Research/SubdivEval/index.html （特征结构数据下载页：lpdata50.dat / lptest.c；同一页面另有 ccdata50.dat / cctest.c 供四边形版对照）"
category: [subdivision-surfaces, surface-evaluation, eigenanalysis, triangle-mesh, classic]
importance: A
historical_importance: 4
game_relevance: 3
production_readiness: "Industry Adopted（思路层面）——Loop 求值器的理论与实现先例；数据文件与 C 程序公开（1998 年的『可直接跑』样本）"
user_level: "Normal（结论层：与四边形版同构的三层结构 + 两个专属细节）"
status: unread
aliases: [Stam 1998 Loop, Evaluation of Loop Subdivision Surfaces, Loop 求值, 三角版求值]
tags: [subdivision-surfaces, surface-evaluation, classic, triangle-mesh]
---

# Evaluation of Loop Subdivision Surfaces（Stam 1998b）

> **入库 2026-10-12（Run 34）。** **Jos Stam**（Alias|wavefront）。**SIGGRAPH '98 CD-ROM Proceedings**——与 [[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|Stam 1998 四边形版]]（SIGGRAPH '98 正刊 pp. 395–404）**同月姊妹篇**："This paper describes a technique to evaluate Loop subdivision surfaces at arbitrary parameter values. **The method is a straightforward extension of our evaluation work for Catmull-Clark surfaces. The same ideas are applied here, with the differences being in the details only.**"（摘要原文）。
> **原文已逐页核对**（作者主页 15 页 PDF：`dgp.toronto.edu/people/stam/reality/Research/pdf/loop.pdf`，858 KB；关键词页图像精读 + pypdf 文本提取交叉；含完整代码段、附录 A/B/C 与参考文献）。
> **库内位置**：几何建模域"求值线"的**三角分支节点**——四边形版（[[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|Stam 1998a]]）给出"怎么算四边形"，本篇给出"**同样的机器怎么算三角形**"；与今日 [[Loop — Smooth Subdivision Surfaces Based on Triangles (1987)|Loop 1987]] 构成"**规则 → 求值**"的三角对偶闭环（四边形弧线是"1978 → 1998"；三角弧线是"1987 → 1998"）。

## TL;DR

**三角版把四边形版的整套机器原样搬过来，只换三个"细节"**（原文自述：different parameter domain / different subdivision rules / different eigenanalysis）：

```text
① 规则片（regular patch）：
   四边形：16 控制点 × 双三次 B-spline
   三角版：12 控制点 × 四阶箱样条片（quartic box spline）
     · 基函数由 Lai 的 FORTRAN 程序转换（B-net 方法）：12×15 矩阵 M
       把 15 个 Bernstein 多项式变成 12 个基函数（附录 A 给出全部 12 式）
     · 参数域 = "单位三角形" Ω = {(v,w) | v∈[0,1], w∈[0,1−v]}（重心坐标 u=1−v−w）

② 奇异点邻域（extraordinary vertex，价 N≠6）：
   K̃ = N + 6 个控制点（N=7 时 13 个；示例图 N=7）
   一次细分 → M = K̃ + 6 = N + 12 个控制点
   ——"Notice that we now have enough control vertices to evaluate
      THREE-QUARTERS of the triangular patch"（一次细分后 3/4 直接可求值）
   剩下 1/4（贴奇异点的角）用无限瓦片递归：
   参数域被切成 Ω₁ⁿ/Ω₂ⁿ/Ω₃ⁿ 无限缩小的瓷砖序列（缩放关系连到单位瓦片）

③ 特征分析（把"矩阵幂"消化掉）：
   A 非亏损（N>3）→ A = VΛV⁻¹；特征基函数 Φ 满足缩放关系
     Φ(v/2, w/2) = ΛΦ(v,w)
   曲面 = 特征基之和：s(v,w) = Ĉ₀ᵀ Φ(v,w)
   实现 = 一张表 + 两个函数（与四边形版同构）：
     EIGENSTRUCT = { L[K]（特征值）, iV[K][K]（特征向量逆）, Phi[K][3][12] }
     ProjectPoints：控制点 → 特征空间（一组控制点只算一次）
     EvalSurf：给定 (v,w) 判断瓦片 n/k → 加权求值（每样本一次）
   ⚠️ 价 N=3 有非平凡 Jordan 块（矩阵不可对角化）→ 附录 C 单开。

④ 数据与代码（1998 年"可直接跑"）：
   CDROM 附带 lpdata50.dat（特征结构，至价 50）+ 读取程序
   （作者主页同款文件至今可下载：lpdata50.dat / lptest.c）

结果（§5）：价 5 与价 7 的完整特征基（Fig. 6）；
"最后 6 个特征基函数与价无关"（只依赖 S₂₁ 的特征向量——四边形版是"7 个共享"）；
价 3~9 网格实测：等参线 / 着色 / 高斯曲率图（Fig. 7）。
结论（§6）：下一步是更一般框架——"Catmull-Clark 与 Loop 视为特例"（Reif 的多项式类）。
```

## Problem

- **与四边形版同题**：奇异点邻域"无法高效求值"被学界当作普遍信念——本篇原文照样写："Until recently it was believed that no parametrizations that lend themselves to efficient evaluation existed at the extraordinary points. **This paper disproves this belief.**"（**同一句"信念推翻"在三角版重演一遍**）；
- **三角特有的负担**：参数域从方形的 [0,1]² 换成**单位三角形**；细分矩阵的块结构、特征值的重数与 Fourier 结构全部换一套（附录 B 是完整的重算）；
- **价 3 的病态**：三角网格里价 3（每顶点只有 3 个邻居）会让矩阵出现 Jordan 块——四边形版无此问题（因为 CC 的奇异价最小值 3 时结构不同），**这是"细节不同"里最大的一个**。

## Historical Context

```text
1998 年 7 月，同一作者、同一单位、同一个月：
  · SIGGRAPH '98 正刊：Exact Evaluation of Catmull-Clark（四边形）  ← pp. 395–404
  · SIGGRAPH '98 CDROM：Evaluation of Loop（三角）               ← 本篇
  ——"求值"从四边形推广到三角形的第一步（作者自述"细节不同而已"，
    但为此重算了全套特征结构 + 单开 Jordan 块附件）
1998 年同届 SIGGRAPH 课程生态："Subdivision for Modeling and Animation"
  （Zorin/Schröder 等组织；Stam 讲授"Exact Evaluation of Subdivision Surfaces"、
   DeRose 讲授 Geri's Game / A Bug's Life 的细分实务——课程页在 Caltech 镜像）
后续：作者的"一般框架"抱负 → 细分方案统一处理方向（Reif 多项式类等）
```

## Previous Work

- [[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|Stam 1998a]]（四边形版）——本方法的"母版"（原文让读者先熟悉 [5]=四边形版再读本篇）；
- **Lai 1992**（B-net / 箱样条子程序，NUMERICAL ALGORITHMS 2:33–38）——基函数转换的实际工具（作者用其 FORTRAN 代码算 12×15 矩阵 M）；
- **de Boor–Höllig–Riemenschneider《Box Splines》**（Springer 1993）——规则区理论；
- **Reif 1995**（统一方法）——作者把"下一步的一般化"指向的候选框架。

## Core Idea

**"换坐标系，不换引擎"**：

```text
同一台机器：递归 → 矩阵幂 → 特征分解 → 特征基直接求值
三角版只换三样：
① 参数域：[0,1]²（方形）→ Δ（单位三角形，重心坐标）
② 规则片：双三次 B-spline（16 点）→ 四阶箱样条（12 点）
③ 特征分析：重算（Fourier 变换 → 特征值 μ₁..；S₁₂/S₁₁ 分块；N=3 的 Jordan 特例）
```

**"3/4 可求值 + 1/4 无限瓦片"**是三角版最漂亮的几何直觉：一次细分之后，奇异点附近只剩四个子三角形中的一个还与奇异点相邻——其余 3/4 全是规则片、直接求值；那个 1/4 再细分一次又只剩它的 1/4……**无限缩小的瓷砖序列 = 参数域上一张递归"地毯"**。

## Technical Approach（逐页核对要点）

- **矩阵链条**：C₁ = A·C₀（K̃×K̃）→ C̃₁ = Ã·C₀（M×M）→ 无限序列 C̃ₙ = Ã·A^(n−1)·C₀；"picking matrices" P_k（12×M）从 C̃ₙ 挑出三个规则片的控制点；
- **瓦片映射** t_{n,k}（三个线性映射：2ⁿ 缩放 + 三种翻转/平移）——EvalSurf 里的 if/else 分派就是它；
- **特征基**：Φ(v,w)|Ω = Λ^(n−1)·(P_k·Ã·V)ᵀ·b(t_{n,k}(v,w))；每块特征基 = 单位瓦片上的四阶样条限制 + 缩放关系；
- **代码细节（两个高级修法）**：
  - 求导：把 EvalBasis 换成导数版，结果乘 2^(n·p)（p=求导阶）——且 **k==1 的瓦片带负号翻转**（`P = k==1 ? pow(-2,n*p)*P : pow(2,n*p)*P`）；
  - N=3（Jordan）：幂有解析式（矩阵 [[1/16,1],[0,1/16]]ⁿ 的闭式），EvalSurf 末尾补一项 `(n−1)·pow(L[i],n−2)·EvalBasis(...)·Cp[i+1]`——**Jordan 块只影响修正项**；
- **数据规模**：lpdata50.dat 覆盖价到 **50**；结构 = {L[K], iV[K][K], Phi[K][3][12]}（K=N+6）；
- **数值细节（附录 B）**：特征值 μ₁=1、μ₂=5/8−α(N)、μ₃..μ_{N+1}=f(k)=3/8+(1/4)cos(2πk/N)（成对重根，N 偶数时有一个单重）；**μ₃²=μ₂**——"这说明不了什么，因为 **Loop 正是从这一关系构造他的方案**"；S₁₂ 特征值 {1/8,1/8,1/8,1/16,1/16}；实值化用 cos/sin 组合（复特征向量两两合并）。

## Key Contribution

- **Loop 曲面的"任意参数值直接求值"**——三角细分从此和四边形一样，可以用在渲染/拾取/纹理映射/光求交管线上；
- **"同引擎换坐标系"的证明**：同一套特征基机器对两种细分方案通用——为作者"统一框架"的下一步（Reif 类）铺路；
- **"3/4 + 瓷砖"的构造**：把"无限递归"压缩到"一个角上的无限瓦片序列"——工程上直接变成 EvalSurf 里 10 行 if/else；
- **1998 年的"可复现资产"姿态**：数据文件 + C 程序随文发布（CDROM），作者主页 28 年后仍可下载——**求值器的"一张表 + 两个函数"模板首次三角化**。

## Why It Works

- 与四边形版同理：细分 = 线性变换的迭代 → 特征空间里"幂"变"数乘"；但三角版多了两处结构性细节：
  1. **3/4 的免费午餐来自"分裂 1→4"**——四边形版一次细分后仍有 1/4 邻域需递归（结构类似），三角版把同样的思想映射到三角网格；
  2. **价 3 的 Jordan 块**：特征值 1/16 的重根（重数 2）使矩阵不可对角化——作者用解析幂公式绕开，**代价是 EvalSurf 多一个修正项**（先例：能算对就不重设计）。

## Limitations

- 依赖 **N>3 非亏损**的结构（N=3 走特例路径）；
- 边界/折痕未涉及（与四边形版相同的研究边界；工业要用需叠 Hoppe 1994 的特征规则）；
- 1998 年的实现数据（lpdata50.dat）是为 CDROM 时代准备的——现代 GPU 求值（Nießner 2012 / OpenSubdiv）走的是另一条并行化路线；
- 成本对照：原文只与四边形版同比（"细节不同"），绝对值细节（每样本几个乘加）需按代码段自行估算。

## Game Development Relevance

- **三角求值器的存在证明**：任何"三角网格 → 光滑极限曲面"的资产语义，1998 年起就有了精确、可微、成本受控的求值路径；
- **对照价值**：游戏运行时不起细分（资产阶段化），但**几何处理工具链**（重网格、法线/曲率估计、表面采样）直接受益——用户 Houdini/工具链工作的底层原理层材料；
- **"一张表 + 两个函数"再次出现**：与 [[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|四边形版]]、"[[Karis — Real Shading in Unreal Engine 4 (2013)|Karis]] 的 EnvBRDF LUT"、"[[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)|Kulla-Conty]] 4KB 表"同族——**"把无穷/复杂预先折叠进一张有限的表"在本库第二次以细分形态出现**（第一次是四边形版）。

## Unreal Engine Relevance

- 同四边形版（引擎吃烘焙结果）；本篇额外对应：**三角面数守恒的场景**（Nanite 簇/碰撞网格）切线连续性依赖奇异点处理——"奇异点处只有 C¹"在硬表面法线插值上的表现是资产侧常识（细分语义 → 烘焙法线）。

## Technology Evolution

```text
求值线（三角分支）：
  1987 Loop 规则（箱样条递归的推广）
  1998 ★ Stam 三角版求值（本篇；"细节不同而已"）——与四边形版同月双发
  2000s 数据/代码散入教学与实现（lpdata50.dat 至今可下载）
  2010s GPU 求值时代（Nießner 2012 / OpenSubdiv 生态；三角与四边形并行）
  2026 逆问题时代（SubDGuide——"求值"之上再加"反推 cage"）
```

## Relationships

### Based On

- [[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|Stam 1998a]]——方法与代码结构母版（原文要求先熟悉它）；
- [[Loop — Smooth Subdivision Surfaces Based on Triangles (1987)|Loop 1987]]——被求值的对象（规则与 μ₃²=μ₂ 构造的原始出处）;
- **Lai 1992**（B-net 转换工具）/ **de Boor–Höllig–Riemenschneider 1993**（箱样条理论）。

### Extends

- **特征基求值法**：从四边形扩展到三角形（本库"同引擎换坐标系"样本）；
- **作者的统一化路线**：结论段明确"下一步 = Catmull-Clark 与 Loop 视为特例的一般框架"。

### Related

- [[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|Stam 1998a]]——姊妹篇（同月、同单位；四边形版 pp.395–404 正刊 / 本篇 CDROM）；
- [[DeRose — Subdivision Surfaces in Character Animation (1998)]]——同届 SIGGRAPH 的"生产化"节点（三篇共同构成 1998 年 7 月的"细分月"）；
- [[Subdivision Surfaces]]——概念载体（本笔记兑现三角分支的"求值"段）。

### Followed By

- **作者的一般框架方向**（Reif 多项式类候选）→ 细分统一处理研究线；
- **GPU/现代求值**：Nießner et al. 2012（特征自适应 GPU 渲染；四边形侧为主，三角侧同理）；
- **教学与实现**：本 PDF + lpdata50.dat + lptest.c 构成"1998 年可复现包"，至今是"细分求值实现史"的一手材料。

## Personal Knowledge State

- Normal（结论层 ≈20 分钟：① 12 点四阶箱样条规则片；② 3/4 + 无限瓦片；③ 一张表 + 两个函数；④ N=3 特例——按"与四边形版同构"读，重点是**差异**：参数域 / 规则片 / Jordan 块）。

## Learning Value

- **两句背下来**：① "**同样的机器，换坐标系——细节不同而已**"（评估任何"跨域移植"的成熟度时对照）；② "**一次分裂后 3/4 是规则区**"（三角版的"免费午餐"来源）；
- **"3/4 + 瓷砖"的工程翻译**：遇到"无限递归"问题，先找"离奇异源最远的 3/4"——主体直接算、角部单开递归（与"分块 + 局部处理"家族同族）。

## Visualization

![[细分曲面_三角分支_Loop 1987 与 Stam 1998 图解]]

## Notes

- **原文核证**：15 页 PDF 全文精读（关键词页转 PNG 图像读 + 文本提取）；参考文献 [1]–[5] 逐条核对（[3]=Loop 1987 的**书目一手来源**、[5]=四边形版 pp.395–404）；
- **下载资产**（作者主页，已验证可达）：`lpdata50.dat` / `lptest.c`（三角版）；`ccdata50.dat` / `cctest.c`（四边形版）——**本库首份"1998 年求值器"实测候选材料**（可选动作：跑 lptest.c 打印一张表）；
- **与四边形版的清单对照**（速记）：16 点/双三次 ↔ 12 点/四阶箱样条；7 个共享特征基 ↔ 6 个共享；K=2N+8 ↔ K=N+6；方形域 ↔ 三角域；无 Jordan 特例 ↔ N=3 Jordan（附录 C）。
