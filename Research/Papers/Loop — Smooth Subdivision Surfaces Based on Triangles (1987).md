---
type: paper
title: "Smooth Subdivision Surfaces Based on Triangles"
authors: [Charles Teorell Loop]
year: 1987
published: "1987-08"
venue: "M.S. Thesis, Department of Mathematics, University of Utah（官方书目页与多源引用交叉核对：犹他大学数学系论文列表页 + Microsoft Research 发布页 + Stam 1998 参考文献 [3] 逐字核对）"
url: "https://www.microsoft.com/en-us/research/publication/smooth-subdivision-surfaces-based-on-triangles/"
code: ""
project_page: ""
category: [subdivision-surfaces, geometric-modeling, triangle-mesh, box-spline, classic]
importance: A
historical_importance: 5
game_relevance: 4
production_readiness: "Industry Adopted（三角细分的事实标准；几何处理 / 有限元 / 地质建模 / 图形学课程的默认三角方案——具体工业采用形态见正文与待补说明）"
user_level: "Normal（结论层：四条规则 + 『平方关系构造』；与 Catmull-Clark 对照着读）"
status: unread
aliases: [Loop 1987, Loop subdivision, Loop 细分, 三角网格细分, Smooth Subdivision Surfaces]
tags: [subdivision-surfaces, geometric-modeling, classic, triangle-mesh]
---

# Smooth Subdivision Surfaces Based on Triangles（Loop 1987）

> **入库 2026-10-12（Run 34）。** **Charles Teorell Loop**，犹他大学数学系硕士论文（1987 年 8 月）。**三角分支的第一个经典锚点**——[[Subdivision Surfaces]] 概念笔记"谱系后继记录位（未注册）：Loop 1987（三角细分）"就此兑现。
> ⚠️ **原文核证状态（重要）**：**原文 PDF 的所有直链在本环境中已不可得**（MSR 站点改版后 loop.pdf 直链已失效/重定向为落地页；Wayback / CiteSeerX / Semantic Scholar / BASE / timetravel / fatcat 等存档服务当日全部限流或不可达——完整尝试清单见文末 Notes）。本笔记的**书目事实**（作者、学位、院系、时间）经**三个独立官方/一手来源交叉核对**；**技术内容**基于（a）[[Stam — Evaluation of Loop Subdivision Surfaces (1998)|Stam 1998（三角版求值）]] 原文附录 B 的**一手转述**（矩阵 S 的分块、特征值、平方关系、原始 α 公式）；（b）Stam 原文引言对 Loop 工作的陈述；（c）多份权威二手的规则表述（互相一致）。**论文原始推导、边界处理、示例细节待原文可得后精读补录**——本笔记的"待补"项均已显式标注。

## TL;DR

**1978 年 Catmull-Clark 与 Doo-Sabin 定义了"四边形网格 → 光滑曲面"的规则；三角形网格（固体建模、有限元、地质建模里的主流形态）没有对应物。** Loop 把**箱样条（box spline）的递归关系推广到任意三角网格**——三角版细分方案自此成立。

```text
Loop 细分规则（任意三角网格，每轮迭代）：

① 分裂（拓扑）：每个三角形 → 分成 4 个小三角形
   为每条边插入一个"边点"（新顶点价恒为 6，规则点）

② 边点（平均规则）：
   新点 = 3/8·(边两端点) + 1/8·(两侧对顶点)     ← 四权重掩码

③ 顶点点（平均规则，含价 N）：
   v' = (1 − Nβ)·v + β·Σ(邻居)
   其中 β = (1/N)( 5/8 − (3/8 + (1/4)cos(2π/N))² )
   等价形式：v' = α·v + β·Σ，α = 3/8 + (3/8 + (1/4)cos(2π/N))²、β = (1−α)/N

④ 先定"验收标准"再反推权重（最关键的设计逻辑，Stam 原文转述）：
   Loop 的权重从"特征值平方关系"构造而来——细分矩阵的次主特征值 = 主特征值的平方
   （μ₃² = μ₂），"这并不意外，因为 Loop 正是从这一关系构造他的方案的"。
```

**结果**：极限曲面**除奇异点（价 N ≠ 6）外处处 C²**（规则区 = 四阶箱样条片）；**奇异点处 C¹**（方案声明；严格化由后续工作补完）。**规则全是局部平均**——无全局求解、天然并行（与 Catmull-Clark 同一工程哲学）。

## Problem

- **1978 年后的版图**：Catmull-Clark（四边形）、Doo-Sabin（四边形为主）——**四边形方案统治了"细分"这个词**；
- **而大批应用天然是三角形**："Triangular meshes arise in many applications, such as solid modelling and finite element simulations"（Stam 原文引言）——三角形网格需要自己的"定义 → 光滑曲面"规则；
- **规则区的标准答案已经存在**：三角网格上的光滑曲面 = **三方向四阶箱样条（quartic box spline）**——难的不是"规则区长什么样"，是"**奇异点（任意价）怎么办**"（箱样条的递归关系只定义在规则网格上）。

## Historical Context

```text
1974  双源头（曲线/渲染侧）：
      · Catmull 博士论文（曲面片递归细分到像素）  · Chaikin 切角法（曲线）
1978  ★ 四边形方案元年：
      · Catmull-Clark（任意拓扑规则；奇异点"命名"）
      · Doo-Sabin（另一方案；奇异点邻域行为分析）
1987  ★ Loop：三角网格细分（本论文）——"四边形方案"垄断六年后的三角对偶
1994  Hoppe et al.：折痕/角点扩展（边界与特征规则体系）
1998  · Stam：三角版精确求值（[[Stam — Evaluation of Loop Subdivision Surfaces (1998)|姊妹篇]]）
      · 四边形侧同年完成"求值（Stam）+ 生产化（DeRose）"
2000s Loop 细分进入几何处理教科书与库（半边结构实操的标准例题）
2010s OpenSubdiv 时代 / 实时求值与自适应细分（Nießner et al. 2012 等）
2026  逆问题时代（SubDGuide）——三角与四边形方案共享同一条 48 年弧线
```

**一个背景注记**：论文出自犹他大学数学系——1970-80 年代 B 样条与 CAGD 理论的重镇（犹他茶壶、Böhm/Farin/Riesenfeld 一系的理论中心）；"三角版箱样条方案 + 平方关系构造"的选题与这一传统同源。

## Previous Work

- **Catmull-Clark 1978 / Doo-Sabin 1978**：四边形框架已立；三角对偶缺席；
- **箱样条理论**（de Boor–Höllig–Riemenschneider 等）：三角网格规则区光滑曲面的理论基石（"三方向四阶箱样条"）；
- **曲线细分（Chaikin 1974）**与 **B 样条递归关系**：'递归细化'思想的先声。

## Core Idea

**"用箱样条定规则区，用平均规则推奇异点——权重从'验收标准'反推。"**

```text
① 规则区（价 6）：三方向四阶箱样条 —— 这是"想要的极限行为"
② 奇异点：把箱样条的递归关系改写成"局部加权平均"形式，再要求
   "任意价都可算 + 特征结构健康"——由此得到含价 N 的顶点规则
③ 健康标准 = 平方关系：μ₃² = μ₂
   （次主特征值 = 主特征值平方 ⇒ 极限曲面在奇异点处"行为像三次族"，
     保证切线连续所需的谱性质——Loop 从这一关系反推 β(N) 的解析式）
④ 全部规则 = 分裂（1→4）+ 两类平均（边点 / 顶点点）——纯粹的局部运算
```

**与 Catmull-Clark 的结构对照（记忆骨架）**：

| | Catmull-Clark 1978 | Loop 1987 |
|---|---|---|
| 面 | 四边形（一次细分后全域变四边） | 三角形（分裂后仍是三角形） |
| 规则数 | 面点 / 边点 / 顶点 三条平均 | **边点 / 顶点 两条平均**（分裂自带面点位置=退化情形） |
| 规则区 | 双三次 B-spline 片 | **四阶（quartic）箱样条片** |
| 每面分裂 | 1 → n（产生 n 个四边形） | 1 → 4（恒为 4） |
| 顶点规则权重 | K=4 附近双三次族 | **平方关系反推的 β(N)** |
| 奇异点 | G¹（后补证明） | **C¹（方案声明，后续严格化）** |

## Technical Approach（按可核来源转述）

**边点（odd point，价恒 6）**——标准形式（多源一致）：
```text
e = 3/8·(v1 + v2) + 1/8·(v3 + v4)
v1,v2 = 该边两端点；v3,v4 = 共享该边的两个三角形的对顶点
```

**顶点点（even point，价 N）**——Stam 1998 附录 B 的一手形式（S 矩阵分块）：
```text
新点 = a_N·v + b_N·Σ(邻居)，其中
  a_N = 1 − α(N)      （自身权重）
  b_N = α(N)/N        （每个邻居的权重）
  α(N) = 5/8 − (3 + 2cos(2π/N))² / 64
```
等价写回"β 形式"：β = α(N)/N = (1/N)(5/8 − (3/8 + (1/4)cos(2π/N))²)；N=6 时：α=3/8+1/4=5/8→自身权重 5/8、邻居各 1/16（标准掩码）。

**特征结构（Stam 转述的构造证据）**：
```text
细分矩阵 S 的特征值：μ₁ = 1，μ₂ = 5/8 − α(N)，μ₃..μ_{N+1} = f(k) = 3/8 + (1/4)cos(2πk/N)
"注意 μ₃² = μ₂。这并不意外，因为 Loop 正是从这一关系构造他的方案的。"
```

## Key Contribution

- **三角网格的任意拓扑细分方案**——与 Catmull-Clark/Doo-Sabin 并列的第三根支柱（且是三角域唯一的主流方案）；
- **"从谱性质反推权重"的构造范式**（μ₃²=μ₂）——本库收下的又一条"先定验收标准、再定实现"的样本（对照：[[DeRose — Subdivision Surfaces in Character Animation (1998)|DeRose 1998]] 的"连续谱折痕 = 可证锚点 + 插值"）；
- **规则极简**：分裂 + 两条平均——与四边形方案共享"局部、稳定、可并行"的工程基因。

## Why It Works

- **规则区的正确性直接继承箱样条理论**（四阶箱样条 = 三方向网格上的自然光滑基）；
- **奇异点的"合理性"由谱关系保证**：细分矩阵的特征值决定细分格式的收敛行为——把"想要的极限行为"翻译成"特征值约束"再解出权重，是细分理论的标准工作法（Loop 是这一工作法的早期样本）；
- **一切局部化**：没有全局求解、没有参数化依赖——同一套规则对任意拓扑一视同仁。

## Limitations

- **奇异点处仅 C¹**（切线连续）——法线/曲率在奇异点邻域不连续；四边形方案（CC 在奇异点同样 C¹）与本方案是同级别妥协；
- **价 3 的特殊性**：细分矩阵含 Jordan 块（非可对角化）——[[Stam — Evaluation of Loop Subdivision Surfaces (1998)|Stam 1998（三角版）]] 为此单列附录 C；
- **极限曲面不插值控制点**（逼近型方案）——设计特征需要折痕/边界规则辅助（Hoppe 1994 等后续工作）；
- **纹理坐标等标量场**同样需要细分管线（对照 DeRose 的 (x,y,z,s,t) 五维细分）；
- ⚠️ 以上为"方案层面"的公共知识；**论文自身的论证细节（奇异点分析、边界处理的原始写法）未读到原文，待补**。

## Game Development Relevance

- **三角形是游戏的母语**：引擎网格、DCC 导出、碰撞代理几乎全在三角域——Loop 是"三角网格也有'笼子语义'"的规则证明；**"四边形建模 + 三角运行"的转换语义**（DCC 里 CC 主导、运行时三角化）与三角域细分的关系是资产管线的既有共识；
- **三角形与四边形方案的"分工"**：四边形（CC）在建模工作流里赢（边流/环形结构友好）；三角形（Loop）在三角原生域（有限元、几何处理、部分扫描/重构管线）赢——**"表示决定方案"的样本**；
- **对分档工作的老线索延续**："细分级别 = 资产精度档"的建模域原型（[[Subdivision Surfaces]] 既有）在三角分支同样成立。

## Unreal Engine Relevance

- 引擎侧不直接跑三角细分（细分发生在资产阶段或作为导入语义）；**与四边形同样的分工**：几何细化在 DCC/资产侧，运行时吃烘焙结果（Nanite 时代的既有格局）；
- 教学向：UE 的"Nanite 三角簇 / 传统 LOD"对照时，Loop 与 CC 是**"光滑语义的两种来源"**——都回答"cage 想说什么"。

## Technology Evolution

```text
三角细分线（本库新开记录）：
  1978  四边形方案（CC / Doo-Sabin）
  1987  ★ Loop：三角方案（本论文；平方关系构造；四阶箱样条规则区）
  1994  Hoppe：折痕/角点扩展（特征规则体系）
  1998  ★ Stam（三角版）：精确求值（同届姊妹篇；"细节不同而已"）
  2000s 教科书化（半边结构标准例题）→ 库与工具链
  2010s 实时求值/自适应细分（与 CC 共享的 GPU 求值时代）
  2026  逆问题时代（SubDGuide：从稠密网格找 cage——两方案共享）
```

## Relationships

### Based On

- **箱样条理论**（三方向四阶箱样条；de Boor–Höllig–Riemenschneider）——规则区的理论基石；
- **曲线细分 / B 样条递归关系**（Chaikin 1974 思想链）；
- **Catmull-Clark / Doo-Sabin 1978**——"任意拓扑规则"的问题设定先例（三角版的对偶问题）。

### Extends

- **细分方案家族**：从四边形（1978 两个方案）扩展到三角形；
- **"谱关系构造"工作法**：把细分格式设计变成特征值约束问题（μ₃²=μ₂）。

### Related

- [[Catmull-Clark — Recursively Generated B-Spline Surfaces (1978)]]——**四边形对偶**（同年入库的"48 年弧线起点"）；两条规则的对照表见正文；
- [[Stam — Evaluation of Loop Subdivision Surfaces (1998)|Stam 1998（三角版）]]——**求值姊妹篇**（"同样的方法，细节不同而已"）；本笔记的**一手转述来源**；
- [[Subdivision Surfaces]]——概念载体（本笔记兑现"Loop 1987 记录位"）。

### Followed By

- **Hoppe et al. 1994**（折痕/角点/边界的特征规则扩展）；
- **[[Stam — Evaluation of Loop Subdivision Surfaces (1998)]]**——精确求值；
- **几何处理生态**（半边结构实现、重网格、地质建模应用——引自 Wikipedia 观测到的跨域采用）；
- **2026 逆问题回声**：[[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction|SubDGuide]]——三角与四边形方案共享"资产要 cage 语义"的下半场。

## Personal Knowledge State

- Normal（结论层 ≈20 分钟：四条规则 + 与 CC 的对照表 + 平方关系构造；推导免读）。
- **读法建议**：先读本文"TL;DR + 对照表"，再翻 [[Stam — Evaluation of Loop Subdivision Surfaces (1998)|Stam 三角版]] 的 §2（12 控制点规则片）——"规则 + 算法"两页纸的直觉。

## Learning Value

- **"先定验收标准、再反推实现"**（μ₃²=μ₂ 构造）——与用户分档工作的"先定判据、再定参数"完全同构；
- **"表示决定方案"**：四边形赢建模、三角形赢三角域——评估任何"平滑/细化"方案时先问"我的表示域是什么"；
- **对照记忆**：CC 三条规则 vs Loop 两条规则 + 四阶箱样条——"规则数量少 ≠ 能力弱"。

## Visualization

![[细分曲面_三角分支_Loop 1987 与 Stam 1998 图解]]

## Notes

**原文核证尝试清单（2026-10-12，全部失败——记录以便回溯）**：
1. MSR 直链 `microsoft.com/.../2016/02/loop.pdf` → 站点改版后返回落地页 HTML（同类问题此前见于 Pixar 站点改版）；
2. `research.microsoft.com/~cloop/thesis.pdf`（Wikipedia 历史外链）→ 重定向 HTML；
3. MSR people 页（cloop）→ 无 PDF 链接（JS 页面）；
4. 犹他大学数学系论文列表页 → 仅书目（作者/学位/年份 ✅ 核对通过）；
5. Wayback Machine（4 种格式 × 2 目标）→ 持续 429（**"不硬刚"原则**：单站点 >4 次即停）；
6. CiteSeerX / Semantic Scholar / BASE / timetravel Memento / fatcat / scholar.archive.org / archive.org 全文检索 → 429 或不可达（本日环境**多站限流并发**——method 观察点）；
7. 常见课程镜像（伯克利 / CMU / UIUC / IITD / UCSD 等 6 处）→ 404 或仅有次要材料；
8. charlesloop.com → 522（站点下线）。

**待补（原文可得后）**：论文内的奇异点分析细节 / 边界规则原始表述 / 示例与图表 / 正式页码引用。

**入库信息**：经 API"未公告先见"窗口的同期经典线产出；与 [[Stam — Evaluation of Loop Subdivision Surfaces (1998)|Stam 三角版求值]] 同日入库（**三角对偶日**）。
