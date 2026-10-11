---
type: paper
title: "Subdivision Surfaces in Character Animation"
authors: [Tony DeRose, Michael Kass, Tien Truong]
year: 1998
published: "1998-07-24（SIGGRAPH '98 Proceedings of the 25th annual conference, pp. 85–94）"
venue: "SIGGRAPH '98（ACM）；署名：Pixar Animation Studios"
url: "https://doi.org/10.1145/280814.280826"
code: ""
project_page: ""
category: [subdivision-surfaces, character-animation, cloth-simulation, modeling-tools, production-pipeline, classic]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（半锐折痕进入全部 DCC 的 crease 工具；布料/碰撞/标量场三件套成为生产管线常规；Geri's Game = 细分曲面首次大规模进入电影制作）"
user_level: "Normal（结论层 / 生产层——对 TA 最友好的一篇经典）"
status: unread
aliases: [DeRose 1998, Subdivision Surfaces in Character Animation, Geri's Game paper, semi-sharp creases, 半锐折痕]
tags: [subdivision-surfaces, character-animation, cloth, production-pipeline, classic]
---

# Subdivision Surfaces in Character Animation（DeRose, Kass, Truong 1998）

> **入库 2026-10-11（Run 33）。** **Tony DeRose、Michael Kass、Tien Truong**（Pixar Animation Studios）。**SIGGRAPH '98, pp. 85–94**——与 [[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|Stam 1998]]（pp. 395–404）同届：**"理论求值"与"生产落地"两个中段节点同日入库**（三位作者中 DeRose 与 Stam、Catmull 共享 2005/2018 奥斯卡技术奖）。
> **原文已逐页核对**（INRIA 课程镜像 10 页 PDF；pypdf 全文提取；含 §3 折痕 / §4 布料 / §5 渲染 / 附录 A-C 与图 1-12 说明）。
> **一句话定位**：**细分曲面从"另一个建模选项"变成"电影级生产的第一等公民"的那篇论文**——半锐折痕（semi-sharp creases）、布料模拟接口、标量场三件套，全部在《Geri's Game》上兑现。
> **库内位置**：几何建模域"生产化节点"；[[Subdivision Surfaces]] 概念笔记谱系中的"DeRose 1998（角色动画/半锐折痕）记录位"就此兑现；与 1978 规则、1998 求值构成"**规则 → 求值 → 生产**"三件套。

## TL;DR

**1998 年要解决的问题**：细分曲面理论优雅（无接缝、任意拓扑、自动光滑），为什么**高端 CG 生产用不起来**？——Pixar 的答案是三个具体障碍，逐个修：

```text
障碍①（建模）：光滑规则做不出"锐边"——桌面棱、指甲、折痕都需要可控的锐度
    解决：半锐折痕（semi-sharp creases）＝ hybrid subdivision
      · 前 s 步用"无限锐利规则"细分，之后切换回光滑规则
      · 锐度 s 可为 0（光滑）/ 1,2,3…（整数，逐步消化）/ ∞（无限锐）/ 任意小数（线插混合）
      · 精句："sharp at coarse scales, smooth at finer scales"（粗尺度锐、细尺度滑）
      · 副产品：semi-sharp 的法线跨折痕平滑变化 → 沿法线置换不撕裂（无限锐会撕裂）

障碍②（动画）：布料物理——能量泛函写在细分网格上会"暴露控制网格结构"
    解决：三件套能量项（= 织物的 warp/weft 语义直接映射到网格边）
      · 抗拉伸：边上强固定长度弹簧（ks）
      · 抗剪切：对角"弹簧能量之积"（kd）——任一对角可自由折叠，但整体抗歪斜
      · 抗弯曲：沿"虚拟线"（virtual threads）的角度项（kp）——跨奇异点也有确定走向
      · 碰撞：把网格"粗化"（unsubdivide）成层级包围盒树——O(N²) → 可分级的邻近查询

障碍③（渲染）：没有 (s,t) 参数面 → 参数纹理与程序化着色器无处落脚
    解决：标量场（scalar fields）——(s,t) 与几何用同一套细分规则细分（5-space 细分！）
      · 证明：控制点上给 (s,t) 值，(x,y,z,s,t) 一起细分 → 纹理坐标光滑（附录 C）
      · 标量场还能当任意着色器参数（衣服缝线 / 鼻孔加深 / 布料刚度调制）

兑现：全部用在《Geri's Game》（1997，奥斯卡最佳动画短片）——Geri 的头/手/衣物。
```

## Problem

- **NURBS 装订本（trimmed NURBS patchwork）的两个硬伤**（原文列举）：
  1. **Trim 昂贵、易数值误差**（"Trimming is expensive and prone to numerical error"）；
  2. **接缝平滑难维持**——模型一动，片与片之间的接缝要"藏起来"：原文点名 **《玩具总动员》Woody 的脸"花了大量手工努力来藏接缝"**（"considerable manual effort was required to hide the seams in the face of Woody"）；
- **细分曲面两个天然优势**：无需 trim + 平滑性自动保证（"even as the model animates"）；
- **但"用不起来"**：原文承认细分在动画系统里不是新东西（1980 年代中期 Symbolics 可能最早用于动画系统造细节多面体；LightWave 3D 同用途），**但那都是"给多边形模型贴光"式用法**；Pixar 要的是**"以极限曲面本体的方式使用"**（"our system reasons about the limit surface itself"）——为此必须补三个洞（见 TL;DR）。

## Historical Context

```text
1978  Catmull-Clark 规则
1980s Symbolics / LightWave：细分作为"细节多面体工具"（非极限曲面语义）
1993  Halstead-Kass-DeRose：极限点/公平插值（Pixar 内部前作）
1994  Hoppe et al.：piecewise smooth surfaces——无限锐折痕（Loop 方案版）
      ↓ ★ 1998 本文：把"无限锐"泛化成"可控锐度"（半锐折痕）
        并把布料 + 标量场补进生产管线——全部兑现于 ↓
1997  《Geri's Game》完成（1997-11 首映；1998 奥斯卡最佳动画短片）
      · 行业史公认：细分曲面第一次大规模进入电影制作
      · Geri 头部控制网格 = "digitizing a full-scale model sculpted out of clay"
        （黏土雕塑数字化而来——Figure 2）
        ↓
1998  SIGGRAPH：Stam（求值理论）+ 本文（生产）同届发布
2005/2018  Catmull、DeRose、Stam 两获奥斯卡技术奖
2012  OpenSubdiv：半锐折痕与求值器开源标准化
```

## Previous Work

- **Hoppe et al. 1994**（piecewise smooth surfaces）——无限锐折痕的出处：修改锐边邻域的细分规则；本文的起点正是**"把无限锐泛化为任意锐度"**；
- **Halstead-Kass-DeRose 1993**——极限点/法线的计算方法（本文 RenderMan 实现把网格点移到极限位置用的就是它 [8]）；也是 Stam 论文的 [4]；
- **Breen et al. 1994 / Courshesnes et al. 1995**——布料模拟的文献背景（本文明确"不重复讲布料模拟系统本身"）；
- **Catmull-Clark 1978**——对象；**Pixar 内部系统**：Marionette（动画）+ RenderMan（渲染）双双扩展。

## Core Idea

### ① 半锐折痕 = 混合细分（hybrid subdivision）

**不**去推导"权重被锐度参数化的规则"（原文列出两条反对理由：① 折痕破坏**循环重索引对称性**——DFT 类证明失效，只能逐价研究（Schweitzer）；② 规则会爆炸成"动物园"——过顶点的折痕数/构型都要一套规则）；改用**两步制**：

> **"先用一套规则细分有限但任意的次数，再切换到另一套规则、作用于极限。"**
> **光滑性只取决于第二套规则**——所以"前 s 步无限锐 + 后光滑"必然光滑。

- **整数锐度 s**：前 s 次用锐规则，此后光滑；每次细分后子边的锐度 = s−1（锐度"消化"一层）；
- **非整数 s**：σ = s − ⌊s⌋；两个版本线性混合（v = (1−σ)v⌊s⌋ + σv⌈s⌉）；"所有折痕同锐度时 = 两整数极限曲面的线性插值；否则不是简单混合"；
- **锐度沿折痕变化**：附录 B（锐度在细分中向外传播，Chaikin 式 3:1 角切）；
- **边界**：边界边标锐 + **价 2 的边界顶点标为 corner**（模仿端点插值的 B-spline 行为——Figure 6）。

### ② 布料能量泛函（把织物语义写进网格）

三条能量项，全部**对控制点定义**（有限差分、质点-弹簧；避免 FEM 在奇异点附近的正交化特例）：

| 项 | 形式 | 直觉 |
|---|---|---|
| 抗拉伸 | 边上固定长度弹簧 ks | 织物沿 warp/weft 几乎不拉伸——**网格边 = 织线方向** |
| 抗剪切 | **对角弹簧能量之积** kd | 任一对角原长时能量为 0 → 沿任一方向自由折叠、但抗歪斜（单弹簧的"太刚/太软两难"被乘积形式化解） |
| 抗弯曲 | 虚拟线角度项 kp | 虚拟线沿网格线走，过价 n 奇异点时"顺时针偏移 ⌊n/2⌋ 条边"继续——**跨奇异点也有确定方向** |

- 调参实例：《Geri's Game》夹克**局部调制 kp**（肩垫 / 翻领 / 腋下加强区）——"真实夹克里也会加强"。

### ③ 碰撞：粗化层级（coarsening hierarchy）

- 优先 **2D 曲面层级而非 3D 体积层级**（优点：层级固定不需重建 / 静态分配 / 不需重平衡 / 短边不产生深分支）；
- 细分网格**没有全局 (s,t) 平面**可建四叉树 → **反着建**：以控制网格面为叶子，**逐层合并**（unsubdivide）直到单一 superface；合并后立刻把 superface 的边移出候选表（防层级失衡）；
- 预处理建一次；每次迭代自底向上更新包围盒；查询 = 从根递归测包围盒（点 vs 层级）；
- 布料 vs 障碍 / 布料 vs 自身（自碰撞）同构。

### ④ 标量场与 5-space 细分

- **主定理（附录 C）**：把 (s,t) 值赋给控制点，**与 (x,y,z) 用同一套细分规则一起细分**（即"在 (x,y,z,s,t) 五维空间细分"）→ 得到的 (s,t) 场在曲面光滑处光滑——**参数纹理映射对细分曲面成立**；
- 标量场推广为"任意程序化着色器参数"：Geri 夹克缝线（近缝线处场值大）、鼻孔/耳窝加深、布料刚度 kp 的调制都是同一机制；
- 赋值的三种工程方法：手工点值 + Laplacian 平滑插值 / 在渲染图上画强度图 + 最小二乘反求场值。

### ⑤ RenderMan 实现（REYES）

- 按面切成 patch → 包围盒（凸包性质）→ dice 成 micropolygon 网格；不可 dice 就**细分一次变 4 子片**；
- **认出 B-spline 子片**（除奇异点/锐边邻域外都是）有三利：固定 4×4 省内存 / 可独立双向拆分 / 前向差分 dice 算法现成；
- 网格点移向**极限位置**用 [8]（Halstead 1993）。

## Key Contribution

1. **半锐折痕**：把 Hoppe 的"无限锐"泛化为**任意锐度**——"一切平滑变化的锐度"成为建模词汇（今天每个 DCC 的 crease/crease-multiplier 都是它）；且发明了**混合细分**这一"规则切换"机制（光滑性只取决于后者——一个可以复用的系统设计原则）；
2. **"锐度对比"的产品性发现**：无限锐折痕法线不连续（法线置换会**撕裂**）、半锐则平滑过渡——**从"数学正确"到"生产可用"的最后一公里**；
3. **布料三件套**：把织物语义（warp/weft/加强区）映射到网格定义域的能量设计 + 反直觉的对角"能量之积"形式；
4. **粗化层级**：无参数平面时的碰撞数据结构构造法（细分的逆操作建树——**与今天"逆问题"时代精神同构的 1998 样本**）；
5. **5-space 细分定理**：纹理坐标（及一切标量场）与几何同规则细分即光滑——**细分曲面第一次拿到"参数曲面级"的贴图/着色能力**；
6. **完整的生产叙事**：三障碍 → 三方案 → 《Geri's Game》兑现——"研究到电影"的完整证据链（这是产业界引用它作为"细分进生产"标志论文的原因）。

## Why It Works

- **混合证法的聪明处**：需要证明光滑性的只有第二套规则——**"先锐后滑"把"任意锐度"的连续谱缩回两个已解决的问题**（Hoppe 锐规则 + CC 光滑规则）；非整数锐度只是两者的线性混合；
- **能量设计的域对齐**：织物是"网格化"的（经纬），CC 的规则区恰是四边形网格——**能量项直接借用了网格自身的结构**（这在奇异点处才需要修补：虚拟线规则）；
- **层级方向的选择**：建"粗"的层级而非"细"的层级——因为细层级天然趋近无穷（细分是无限的），**只存在"向粗"的有限层级**。

## Limitations

- 布料能量模型是**近似**（非 FEM；"不是本文目标——那需要另一篇论文"）；碰撞用包围盒层级（对薄膜自碰撞够用，非通用固体碰撞方案）；
- 半锐折痕的混合机制本质是"整数锐度 + 插值"——**锐度语义是离散的**（非参数化连续规则）；
- 论文不是全系统论文：三个方向都点到"生产就绪"但细节（如布料完整求解器）在别处；
- 论文自述生成的生产约束（来自背景资料，非论文正文）：布料管线要求动画提前 ~30 帧送入模拟器、"镜头外也不能偷懒"——**模拟接管后的管线节奏变化**是采用物理布料的第一课。

## Game Development Relevance

- **半锐折痕 = 你今天用的 crease 工具的出处**：Maya/Blender/3ds Max/Modo 的 crease（及"crease multiplier"）语义直接来自本文的"整数锐度 + 插值"；**倒角/棱边/硬表面**的"可控锐度"词汇是它定义的；
- **"可控锐度"= 分档词汇的建模版**：光滑 ↔ 锐利之间的**连续谱**（0 → ∞）——与你五维预算里"档位在连续谱上取点"同构；**"extreme cases = darts / creases / corners"** 是锐度的三个结构性锚点；
- **布料管线的第一课**：能量泛函写在"有结构的网格"上（织物语义 ↔ 网格方向）+ 模拟接管的管线节奏——Chaos Cloth / 自研布料都还在这条延长线上；
- **碰撞层级的构造法**（unsubdivide 建树）对今天的启发：**当正向没有全局参数空间时，反着建立层级**——与 SubDGuide（曲面→cage）、粗化思想共享"逆操作给出结构"的母题；
- **标量场**：顶点属性/UV 在细分级联中如何保持光滑——OpenSubdiv 的 face-varying data（面变化通道）就是这条线的现代形态。

## Unreal Engine Relevance

- **建模向**：UE 的建模模式/资产导入若走 SubD 转换链路，折痕语义（硬边、锐度）决定塔形结果；"semi-sharp 可置换不撕裂"直接关系位移贴图工作流；
- **布料向**：Chaos Cloth 的底层是另一族方法（PBD/XPBD），但本文的"能量项分工（拉伸/剪切/弯曲）"仍是调试布料手感的分类语言——**"手感 = 哪些能量项在起作用"**；
- **渲染向**：5-space 细分定理的现代回声 = "UV 与几何同细分"（OpenSubdiv face-varying）；Nanite 时代运行时吃三角化结果，但**资产阶段**的细分/置换/UV 级联正是本文思想的下游。

## Technology Evolution

```text
"细分进生产"的路线图（本文 = 施工图）：
  1994 Hoppe：无限锐折痕（能用，但"要么锐要么滑"）
  → ★ 1998 本文：半锐折痕 + 布料接口 + 标量场 + RenderMan 实现
  → 1997-98 《Geri's Game》兑现（首个大规模 SubD 制作）
  → 2005/2018 奥斯卡（"scientific and practical implementation"）
  → 2012 OpenSubdiv：折痕语义（含 semi-sharp）+ 求值器开源标准化
  → 今天：DCC crease 工具 / 游戏硬表面 / 汽车 A-class 曲面

本库同周对照：
  10-10 Catmull-Clark 1978（规则）→ 10-11 Stam 1998（求值）+ 本文（生产）
  ——"中段三节点"一天补齐：规则 → 算得出来 → 用得起来
```

## Relationships

### Based On

- [[Catmull-Clark — Recursively Generated B-Spline Surfaces (1978)]]——对象方案（"a variant of Catmull-Clark"）；
- **Hoppe et al. 1994（piecewise smooth surfaces）**——无限锐折痕（本文推广的起点）；
- **Halstead-Kass-DeRose 1993**（Pixar 内部前作，未入库）——极限点计算（RenderMan 实现使用）。

### Extends

- **Hoppe 1994 的"无限锐"** → **"任意锐度"**（半锐折痕；混合细分机制）；
- **布料模拟文献** → **"布料 × 细分网格"的接口设计**（能量域对齐 + 碰撞层级）。

### Related

- [[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|Stam 1998]]（同届 SIGGRAPH，同期独立）——**理论 ⇄ 生产配对**：两份同年答卷共同定义了"细分可用"；
- [[Subdivision Surfaces]]——概念载体（兑现"DeRose 1998 记录位"）；
- [[2026-10-08-SubDGuide — A Modeler-Inspired Agentic Workflow for Mesh-to-SubD Reconstruction|SubDGuide]]——48 年后的逆问题侧（cage 语义正是本文"折痕/边界/特征"词汇的延续）。

### Followed By

- **OpenSubdiv（2012+）**——半锐折痕与求值器开源实现；
- **全部 DCC 的 crease 工具**；**Chaos/自研布料**（能量项分类语言）；
- 本库谱系：[[Subdivision Surfaces]] 时间轴"1998 生产化"节点。

## Personal Knowledge State

- **user_level: Normal（结论层 / 生产层）**——**库内对 TA 最友好的一篇经典**：三障碍三方案，无重数学；
- **读法建议（≈30 分钟）**：§1 Motivation（Woody 接缝段——TA 痛点共鸣）→ §3 半锐折痕（混合细分思想 + Figure 7 立方体 0/1/2/3/∞ 对照；机制细节可跳）→ §4.1 能量三件套（表格级即可）→ §4.2 粗化层级思想 → §5.1 5-space 细分结论段 → §6 结论；
- **与用户的关系**：**"你每天用的 crease 与布料调试语言的出处"**——建模域"日用工具"的第二件套（与 [[Catmull-Clark — Recursively Generated B-Spline Surfaces (1978)|1978 规则原文]]、[[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|Stam 求值]] 三件套齐）。

## Learning Value

1. **"连续谱的离散化"技法**：把"任意锐度"这一连续目标，实现为"整数锐度（可用既有规则证明）+ 线性插值"——**面对连续参数需求，先找它的"可证锚点"，再在锚点间插值**；
2. **"障碍清单式论文"的结构价值**：三个障碍 → 三个方案 → 一个作品兑现——**研究论文的"生产论证"完整范式**（对做工具方案汇报的人可直接抄结构）；
3. **"法线连续性"的产品课**：数学上"锐"与"半锐"都合法，**产品上只有半锐不撕裂**——"正确性"与"可用性"的差距常常在一个具体的下游操作（置换）里；
4. **域对齐再+1**：织物语义（经纬）↔ 网格边方向——与 RiCo 的"接触域对齐"、PCAsplat 的"监督域对齐"同族（**"让数据的结构长得像问题的结构"**，1998 年就有最干净的一例）。

## Visualization

![[细分曲面_求值与生产化_Stam 1998 与 DeRose 1998 图解.html]]

## Notes

- **核对记录**：INRIA 课程镜像 10 页 PDF 全文提取（pypdf）；页码/DOI（pp. 85–94，DOI 10.1145/280814.280826）经 dblp / ACM 记录核实；《Geri's Game》信息（1997-11 首映、1998 奥斯卡最佳动画短片、黏土雕塑数字化的头部控制网格）经多来源交叉（Wikiwand / 制作资料 / 中文渠道）；
- **图版细节**：Figure 5 = Geri 的手（指甲与皮肤之间的**无限锐折痕**）；Figure 7 = 单位立方体锐度 0/1/2/3/∞ 对照；Figure 10 = Lanteri 风格雕塑——**带变量锐度折痕后控制面数 840 → 627**（"减少控制网格尺寸"的量化样本）；
- **"生产就绪"的判词**：结论段"use of subdivision surfaces allows our model builders to arrange control points in a way that is natural... without concern for maintaining a regular gridded structure"——**"局部细化"（local refinement）被点名为此前 NURBS 做不到、SubD 做到的核心工作流解放**；
- **彩蛋**：论文把"极限曲面本体"当第一等公民（"our system reasons about the limit surface itself"）——与 2026 年 SubDGuide"永不生成顶点、只做决策"的两权分离，是同一种"尊重表示语义"的相隔 28 年的回声。
