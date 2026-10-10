---
type: paper
title: "Recursively generated B-spline surfaces on arbitrary topological meshes"
authors: [Edwin Catmull, Jim Clark]
year: 1978
published: "1978-11（Computer-Aided Design, Vol 10 No 6）"
venue: "Computer-Aided Design 10(6): 350–355, November 1978（IPC Business Press；署名：Computer Graphics Laboratory, New York Institute of Technology）"
url: "https://doi.org/10.1016/0010-4485(78)90110-0"
code: ""
project_page: ""
category: [subdivision-surfaces, geometric-modeling, b-spline, asset-pipeline, classic]
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（全部 DCC / 游戏引擎 / 汽车工业型面设计的标准表示；2005 年奥斯卡技术成就奖；Pixar OpenSubdiv 开源实现）"
user_level: "Normal（结论层：三条规则 + 两个开放问题 + 与今日 SubDGuide 的正逆对话）"
status: unread
aliases: [Catmull-Clark, Catmull-Clark subdivision, 细分曲面, 递归细分, Recursively generated B-spline surfaces]
tags: [subdivision-surfaces, geometric-modeling, b-spline, classic, asset-pipeline]
---

# Recursively generated B-spline surfaces on arbitrary topological meshes（Catmull & Clark 1978）

> **入库 2026-10-10（Run 32）。** **Edwin Catmull 与 Jim Clark**，署名 **NYIT 计算机图形实验室**（Computer Graphics Laboratory, New York Institute of Technology）。**Computer-Aided Design 10(6): 350–355, 1978 年 11 月**——注意：发表在 **CAD 期刊**（不是 SIGGRAPH！），后来成为整个计算机图形工业的曲面标准。
> **原文已逐页核对**（6 页扫描件；USTC 课程镜像 PDF；含图 1–图 11 全部图版与完整参考文献）。
> **一句话定位**：**把"递归细分"从矩形网格推广到任意拓扑网格**——双三次 B-spline 的泛化；三条规则（面点 / 边点 / 顶点）从此定义了"稀疏控制笼 → 光滑曲面"的全部后续世界（DCC / 游戏 / 汽车）。
> **库内位置**：**几何建模域第一个经典锚点**（此前库内渲染/动画/AI/物理/PCG 均有谱系，独缺"建模"）；与 [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)|Clark 1976]]（同一作者）相邻；与今日前沿 [[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction|SubDGuide]] 构成**"正问题 ↔ 逆问题"的 48 年对话**（本库主动建立，原文无引用关系）。

## TL;DR

**1978 年要解决的问题**：B-spline 曲面只能在**矩形控制点网格**上定义——但现实中的物体拓扑是任意的（有洞、有任意价顶点）。能否有一个**递归细分**方案，让任意拓扑的控制网格也能生成光滑曲面？

```text
答案：三条规则（任意拓扑均适用）
(A) 面点 = 该面所有旧顶点的平均
(B) 边点 = (旧边中点 + 相邻两个新面点的平均) / 2
(C) 新顶点 = Q/n + 2R/n + S·(n−3)/n
        Q = 相邻面点的平均 · R = 相邻边中点的平均 · S = 旧顶点 · n = 该顶点价
——（矩形情形退化为 q = Q/4 + R/2 + S/4；论文用 splitting matrix H₁ = M⁻¹SM
    证明：n=4 时整个子片恰为标准 bicubic B-spline 片 G₁ = H₁GH₁ᵀ）

结果性质：
· 一次细分后：所有面都是四边形 → 新顶点价恒为 4 → 【奇异点数恒定】
· 非奇异区域：标准 B-spline 片（切线与曲率连续）→ 极限曲面"除奇异点外处处光滑"
· 奇异点（extraordinary points，Coons 建议的命名）：图片显示切线连续，
  【但无解析证明】——作者三处重复强调
```

**三个开放问题（论文自己列的，全部后来被兑现）**：
1. 奇异点连续性的**解析证明**（→ 1998 Peters-Reif 等）；
2. **高价奇异点行为不佳**（Sabin 建议的鞍面 z=xy 实验：中心 8 价顶点处"not well behaved"，作者自认"还没找到最好的规则集"→ 后世持续研究）；
3. "**任意拓扑网格上的近似格式需要一个统一的数学处理**"（结尾呼吁——细分曲面整个理论领域的第一声）。

## TL;DR（生产视角）

> 所有 DCC（Maya / Blender / 3ds Max / Modo……）、UE / Unity 的建模管线、汽车工业（**今天的 [[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction|SubDGuide]] 就是 Mercedes-Benz 在做**）的型面设计，全部建立在这 6 页论文的三条规则上。**2005/06 年 Catmull、DeRose、Stam 因此获奥斯卡技术成就奖**；Pixar 的 OpenSubdiv 是它的开源工业实现。

## Problem

**矩形网格 = B-spline 的牢笼。**

- 双三次 B-spline 片由 **4×4 控制点网格**定义（Figure 1）——"standard bicubic B-spline patch on a rectangular control-point mesh"；
- 但真实几何的拓扑是任意的：三角面、任意价顶点、洞；
- 直接放弃矩形结构就失去 B-spline 的全部好性质（连续性、局部性、可求值）；
- **目标**：找到一组**递归规则**——对任意拓扑网格，每次细分生成更细的网格，其极限曲面"在矩形区域内是标准 B-spline、在任意区域也能光滑"。

## Historical Context

```text
1974  【双源头】
  · Catmull 博士论文（Utah）："A subdivision algorithm for computer display of
    curved surfaces"（UTEC-CSc-74-133）——为【渲染】而生的递归细分：
    把曲面片细分到像素大小，让可见性/着色测试变简单（"rendering shaded
    pictures of curved surface patches"）
  · Chaikin 在 seminar 上给出"递归切角法"生成光滑曲线
    （后由 Riesenfeld 整理发表："On Chaikin's Algorithm", 1975）
        ↓ "Motivated by this..."
★ Catmull 发明了任意拓扑的细分规则——【但无法证明曲面良态，他没有实现】
        ↓ "Recently, Clark implemented the method to empirically determine
          if the surface is well behaved and generalized the rule..."
★ Clark 实现了它（经验验证）+ 泛化了新点规则——两人合作发表（本文）
        ↓
1978  Doo & Sabin：分析奇异点邻域行为（同期下一篇文章, pp. 356–360；
      他们此前已给出另一套细分方案 Doo-Sabin，并建议了"extraordinary points"命名）
        ↓
1987  Loop：三角网格细分方案
1998  Stam：极限曲面的精确求值（小波/矩阵指数——不再递归，直接求点）
1998  DeRose/Kass/Truong：SubD 进角色动画（《Geri's Game》——史上首部 SubD 动画短片）
      + 半锐折痕（semi-sharp creases）
2005/06  Catmull / DeRose / Stam 获奥斯卡技术成就奖
2012  Nießner et al.：特征自适应 GPU 求值（SubD 进实时渲染）
        ↓
★ 2026  今日 SubDGuide：反方向——从稠密网格【恢复】控制笼
```

**"金字塔尖的两个开放问题"**——论文的案例图直接把未解问题摆出来：
- **四面体**（最小的非矩形体）：6 个原始 3 价顶点 + 4 个面 → 8 个奇异点（图 3/4）；
- **鞍面 z=xy**（应 Sabin 的建议）：中心 8 边形 → 价格外高。**渲染图显示中心"not well behaved"（图 10）**——作者原话："The saddle demonstrates that the authors have not found the best set of rules."

## Previous Work

论文引用的全部 5 条（原文参考文献逐条核对）：

| # | 引用 | 作用 |
|---|---|---|
| 1 | Catmull 1974（Utah 技术报告） | 递归细分渲染的起点（本文的"正传") |
| 2 | Riesenfeld 1975 "On Chaikin's Algorithm" | 切角法的整理发表（灵感来源之一） |
| 3 | Lane & Riesenfeld 1977（Utah 技术报告） | "total positivity" 曲线曲面设计（同期竞争路线） |
| 4 | Barnhill 1974 "Smooth interpolation over triangles" | 三角片插值（另一条竞争路线） |
| 5 | **Doo & Sabin 1978 "Analysis of the behaviour of recursive division surfaces"（pp. 356–360，同期下一篇）** | 奇异点邻域分析（本文作者致谢他们的预测） |

> 期刊把 Catmull-Clark 与 Doo-Sabin 分析排为**连续两篇**——1978 年 11 月这期 CAD 期刊是细分曲面的"创刊号"。

## Core Idea

**"让规则在矩形上退化为 B-spline、在任意拓扑上继续工作。"**

**证明结构的优雅之处**（论文 §"RECTANGULAR B-SPLINE PATCH SPLITTING"）：不直接给规则，而是**先做矩阵推导**：
1. 双三次 B-spline 片：$S(u,v)=UMGM^{\top}V^{\top}$（M = B-spline 基矩阵，G = 16 控制点）；
2. 对子片（u,v ≤ 1/2）重复同样形式 → 必须满足 $G_1 = H_1 G H_1^{\top}$，其中 $H_1 = M^{-1}SM$（**splitting matrix**）；
3. 展开矩阵乘法得到**具体公式**：
   - 面点 $q_{11} = (p_{11}+p_{12}+p_{21}+p_{22})/4$（四点平均）；
   - 边点 $q_{12} = \frac{(C+D)/2 + (p_{12}+p_{22})}{2}$（相邻两面点平均与边中点的平均）；
   - 顶点 $q_{22} = Q/4 + R/2 + S/4$（矩形时 n=4 的退化）；
4. **再把公式"升格"为三条任意拓扑规则**（A/B/C）——对 n≠4 的顶点，公式形态相同、参数换成局部平均量。

**规则 (C) 的历史原味细节（原文 §"arbitrary topology" 之后）**：
> "The set of rules presented above is somewhat arbitrary. In fact, **initially a different rule was tried for (C)**. The new vertex point was **(C)(alternate) = Q/4 + R/2 + S/4**. The results using that rule were unsatisfactory in that **the surface became too pointy for the tetrahedron**. The pictures made using that rule **motivated us to find a better set of rules**, the best of which was presented above."

——**规则先凭"图像好看"选出来，数学论证跟在后面**；而且论文自己把这个软肋写明白：
> "A better set of rules, indeed, **a better criterion for judging the rules than the qualitative appearance of a picture, is yet to be devised.**"

## Technical Approach（细节核对面）

- **规则汇总**（论文原文措辞）：
  - (A) 新面点 = 定义该面的所有旧点的平均；
  - (B) 新边点 = 旧边中点与两个相邻新面点平均的**平均**；
  - (C) 新顶点 = $Q/n + 2R/n + S(n-3)/n$；$Q$ = 所有相邻面的面点平均，$R$ = 所有相邻边中点平均，$S$ = 旧顶点，$n$ = 该顶点价；
  - 形成新边/新面：面点连到边点、顶点连到边点（全四边形化）；
- **拓扑不变量**："after one iteration all faces are four-sided, hence all new vertices created subsequently will have four incident edges. **Therefore after one iteration the number of extraordinary points on the surface remains constant.**"（奇异点数在第一次细分后即冻结——这保证了"奇异点稀少"的结构）；
- **双二次版本**（附录性质）：$q_{11} = (9p_{11}+3p_{12}+3p_{21}+p_{22})/16$——"A similar algorithm for biquadratic B-splines"；
- **周期案例**：四面体（8 奇异点）；鞍面 z=xy（Sabin 建议；8 价中心）；
- **变体讨论**：与 Lane-Riesenfeld（generalized basis function）与 Barnhill（三角片）的明确划界："Neither of these approaches is the same as the method described in this paper."

## Key Contribution

1. **三条递归细分规则**：第一个"任意拓扑 → 光滑曲面"的实用方案（B-spline 的拓扑泛化）；
2. **极限曲面的结构理论**（在当时能做到的程度上）：非奇异区域 = 标准 B-spline（连续性来自片间共享）；奇异点数恒定；
3. **工程可实施性**：规则极简（纯平均运算）→ 后来统治整个工业；
4. **诚实的开放问题清单**（见上）——成就的一半是"说清了没解决什么"。

## Why It Works

- **"退化验证"策略**：任意拓扑规则在 n=4 时**精确退化为**已知正确的 B-spline 子片——**新方法以旧方法为特例**（数学推广的标准范式）；
- **纯平均 = 局部性 + 稳定性**：三条规则都只依赖局部邻居（1-ring）——细分是局部操作、天然并行、天然稳定（无需求解全局方程）；
- **四边形化**：一次细分后拓扑统一为四边形 → 后续迭代行为可控（奇异点数恒定）；
- **审美 → 数学的两阶段**：先用几何直觉/图像筛选规则，再补连续性证明——**"先有能跑的，再有说得清的"**。

## Limitations（1978 年作者视角 + 后世修正）

- **① 奇异点连续性无证明**（作者反复自我标注："no analytical proof of continuity is given"）——后来由 Peters-Reif（1998）等建立理论；
- **② 高价奇异点行为不佳**（鞍面实验暴露；"还没找到最好的规则集"）——至今是研究点；
- **③ 规则本身"somewhat arbitrary"**（作者原词）——缺少"规则好坏的评判标准"（"yet to be devised"）；
- 当年实现限制：论文的渲染图来自自研系统（图 3/6/10 为 1978 年硬件渲染的扫描照片）。

## Game Development Relevance

- **这是游戏资产管线的地基**：角色/道具的"高模平滑外观"、硬表面倒角、可细分层级（cage → 细分）全是 Catmull-Clark 语义；**UE / Unity 及全部 DCC 的 SubD 支持是它的直接实现**；
- **"控制笼"是美术与管线的接口**：LOD 链、细分级别、型面调整都从 cage 出发——理解"cage 里每条边流的含义"是资产质量的根本（今日 [[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction|SubDGuide]] Figure 1 的教训："自动重网格磨平特征"正因为不懂"边流编码设计"）；
- **与用户的直接关系**：TA 本行（Houdini / Maya 的 subdivision、UE 的 SubD 导入链）；**"48 年后反向问题出现"**（见 §Technology Evolution）；
- 工业旁证：**汽车工业（Mercedes-Benz）今天仍在为"找回 cage"立项**——SubD 的语义价值经过了 48 年检验。

## Unreal Engine Relevance

- UE 支持 **Catmull-Clark 细分**（导入时的 Subdivision Surface 选项 / Geometry Script 的细分能力）；Nanite 时代的用法是"细分后烘焙为高密度网格"（细分发生在资产阶段，运行时吃烘焙结果）；
- **与分档的呼应（历史层）**：[[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)|Clark 1976]] 里"细节按需生成（top-down splitting，Catmull 式曲面片细分）"正是 LOD 与细分同源的证据——**本篇的细分与 [[Scalability and Quality Tiers|分档]] 的"层级细节"思想同根**（都是"由粗到细、按需生成"）。

## Technology Evolution

```text
【细分曲面 48 年弧线】
1974  Catmull 博士论文（细分为渲染）+ Chaikin 切角法（灵感）
1978  ★ 本文：任意拓扑规则（正问题：cage → 曲面）
      + Doo-Sabin 分析（同期）
1987  Loop（三角细分）· 1998 Stam 精确求值 / DeRose 角色动画（工业化）
2005  奥斯卡技术成就奖 · 2010s OpenSubdiv（GPU 求值标准化）
      ——全 DCC + 游戏 + 汽车工业采用
2026  ★ SubDGuide：逆问题（稠密网格 → cage）
      ——"有曲面怎么找回设计结构"：48 年后同一问题的反方向

【本库的另一条呼应线】
Clark 1976（LOD：可见复杂度上限）→ 本文 1978（细分：细节按需生成）
    ——同一作者相邻节点；LOD 的"top-down splitting"引用 Catmull 式细分
```

> **"正问题 ↔ 逆问题"是技术史里罕见的弧线**：多数谱系是"更强/更快"（同方向），细分曲面的故事是"**同一个问题反过来再走一遍**"——而逆向之所以可能，恰因 48 年的资产积累（稠密网格遍地）。

## Relationships

### Based On

- **Catmull 1974（Utah 博士论文）**——递归细分思想与渲染动机的直接来源；
- **Chaikin 切角法（1974/1975）**——"递归生成光滑曲线"的启示（作者原文承认）；
- **双三次 B-spline 理论**——矩形情形的数学母体（论文用 splitting matrix 证明泛化关系）。

### Extends

- **B-spline 曲面**——从矩形网格（4×4）到任意拓扑（本文核心贡献的准确表述）。

### Related

- [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)|Clark 1976]]（库内）——**同一作者**（Jim Clark）的前作：LOD / 层级几何；时间线里已含 "1974 Catmull 曲面细分显示"的预告 —— 两篇合读 = "层级细节"与"细分生成"的同源关系；
- [[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction|SubDGuide]]（今日前沿）——**逆问题**（本库主动建立对话）；
- **Doo-Sabin（1978，同期）**——另一套细分方案 + 奇异点分析（未入库，观察位）。

### Followed By

- **Stam 1998 精确求值** / **DeRose et al. 1998 角色动画（半锐折痕）** / **Loop 1987**——（均未入库，作为谱系后继记录）；
- **OpenSubdiv（Pixar，2012+）**——工业实现。
- 未来：[[Subdivision Surfaces]] 概念笔记（本日建立）承载本条的"概念侧"。

## Personal Knowledge State

- **user_level: Normal（结论层）**。前置：DCC 建模经验（用户日常）+ 基本线性代数（矩阵只是证明工具，结论是"三条平均规则"）；
- **读法建议（≈25 分钟，不必读矩阵推导）**：摘要 → §"ARBITRARY TOPOLOGY" 三条规则 → 图 2（三角拓扑细分过程 a→d）→ 图 3 四面体渲染 → §"BIQUADRATIC"（一句话知道有双二次版即可）→ §CONCLUSIONS + 结尾"统一数学处理"呼吁 → 参考文献 5 条（1978 学术圈的完整切片）；
- **与用户的关系**：**TA 本行知识的"宪法"**——你日常在 DCC 里拨动的每个 cage 顶点，其行为都由此文的规则定义。

## Learning Value

1. **"以旧方法为特例"的推广范式**：新规则在 n=4 退化为已验证正确的 B-spline——**判断一个推广是否可靠的第一检查**：它在已知情形下退化成什么？
2. **"审美先行"的诚实样本**：规则先由图像筛选（(C)(alternate) 太尖→换规则），论文自认"评判标准未发明"——**把《评估标准本身是研究问题》写到明面上，是 1978 年的方法勇气**（与今日 GaussianBench"评估协议"、Sensitivity"检查界面"、SubDGuide"验证回滚"的"可验证性"主题隔 48 年共振）；
3. **开放问题的价值**：三个 open problems 分别催生了一个理论领域（连续性理论）、一个长期研究方向（高价奇异点）、一个数学子领域（细分曲面理论）——**论文的影响力也可以由"留下的问题"定义**。

## Visualization

![[细分曲面_1978正问题与2026逆问题图解.html]]

## Notes

- **核对记录**：原文 PDF（USTC 课程镜像；6 页扫描件）逐页阅读——包括图 1–11、规则 (A)(B)(C) 原文措辞、参考文献 5 条、结论段三处"无证明"声明、"too pointy"故事、鞍面实验结论；期刊卷期（10(6):350–355, Nov 1978）、署名（NYIT）由图 1 首行与页脚确认；
- **作者背景**：Catmull 时任 NYIT 计算机图形实验室主任（1974–1979，后随 Lucasfilm/Pixar）；Clark 时任 UC Santa Cruz 助理教授（1974–78；1979 转 Stanford；1982 创办 SGI；1994 创办 Netscape）——**1978 年两人都在 NYIT 的课题上合作**（论文署名同为 NYIT）；两人均为 Utah 1974 届博士（犹他图形学黄金年代）；
- **命名细节**："extraordinary points（奇异点）"是 **Coons 建议的命名**（原文："Following a suggestion by Coons, we refer to these points as extraordinary points"）——术语的出处级留痕。
