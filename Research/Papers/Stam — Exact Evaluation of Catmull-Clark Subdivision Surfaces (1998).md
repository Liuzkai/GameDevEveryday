---
type: paper
title: "Exact Evaluation of Catmull-Clark Subdivision Surfaces at Arbitrary Parameter Values"
authors: [Jos Stam]
year: 1998
published: "1998-07-24（SIGGRAPH '98 Proceedings of the 25th annual conference, pp. 395–404）"
venue: "SIGGRAPH '98（ACM）；署名：Alias|wavefront, Inc., Seattle"
url: "https://doi.org/10.1145/280814.280945"
code: ""
project_page: "http://www.dgp.toronto.edu/~stam/reality/Research/SubdivEval/index.html （特征结构数据下载页；论文 PDF：dgp.toronto.edu/people/stam/reality/Research/pdf/sig98.pdf）"
category: [subdivision-surfaces, geometric-modeling, surface-evaluation, eigenanalysis, classic]
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（全部 SubD 求值器的理论与实现基础；Alias/Maya 直接采用；OpenSubdiv / 特征自适应 GPU 求值的前置）"
user_level: "Normal（结论层：三层求值 + 成本对照；推导层（特征分析）挂起）"
status: unread
aliases: [Stam 1998, Exact Evaluation, SubdivEval, 精确求值, Catmull-Clark 求值]
tags: [subdivision-surfaces, geometric-modeling, siggraph, classic, surface-evaluation]
---

# Exact Evaluation of Catmull-Clark Subdivision Surfaces at Arbitrary Parameter Values（Stam 1998）

> **入库 2026-10-11（Run 33）。** **Jos Stam**（Alias|wavefront，Seattle；论文脚注邮箱 jstam@aw.sgi.com）。**SIGGRAPH '98，pp. 395–404**——与 [[DeRose — Subdivision Surfaces in Character Animation (1998)|DeRose 1998]]（pp. 85–94）**同一届 SIGGRAPH**：细分曲面的"理论求值"与"生产落地"两个中段节点同日建成（两位作者在 2005/2018 两次共享奥斯卡技术奖）。
> **原文已逐页核对**（作者主页 10 页 PDF，pypdf 全文提取；含实现代码段 EIGENSTRUCT / ProjectPoints / EvalSurf、附录 A/B 与参考文献全表）。
> **一句话定位**：**"Catmull-Clark 曲面不能直接求值"是一个被普遍相信但错误的信念**——本文把它推翻：递归细分可展开为矩阵幂，特征分解后极限曲面 = 特征基函数之和，**任意 (u,v) 直接求值、成本与双三次样条相当**。
> **库内位置**：几何建模域"求值线"的**理论节点**（1978 规则 → **1998 本节点** → 2012 GPU 求值 / OpenSubdiv）；[[Subdivision Surfaces]] 概念笔记谱系中的"Stam 1998（精确求值）记录位"就此兑现。

## TL;DR

**1998 年要解决的问题**：AT&T 贝尔实验室的数学家们证明过细分曲面"理论上可求值"，但整个图形学界**坚信**："Catmull-Clark 曲面无法在任意参数值处直接求值——只能一遍遍细分逼近。"于是渲染、拾取（picking）、纹理映射这些标准操作都缺一条快的路；迭代逼近在奇异点附近"太贵，且给不出精确的高阶导数"。

```text
答案：把"无限递归"展开成"矩阵幂 + 特征分解"

① 一次细分 = 一个线性变换（扩展细分矩阵 A，尺寸 2N+8）
      每细分一次：v_{k+1} = A · v_k     →    v_k = A^k · v_0
② A 非亏损（non-defective）→ 特征分解 A = V Λ V⁻¹
      →  v_k = V · Λ^k · V⁻¹ · v_0
③ 曲面片 = 各层新控制点 × 双三次 B-spline 基的无穷和；
   把 Λ^k 吸进基函数 → 预计算"特征基函数" φ_i（只依赖奇异点价 N）：
       p(u,v) = Σ_i  φ_i(u,v) · λ_i^n · (投影控制点_i)
   （n = 参数点所处的细分层级，由 u,v 直接算出）

结果：
· 任意 (u,v) 直接求值——不需要任何细分步骤
· 导数（任意阶）同法可得——法线/曲率/光线求交的 Newton 迭代能用
· 单次求值成本 ≈ 双三次 B-spline（多一个 log2 和一个整数幂）
· 规则区域：特征基 = 幂基（power basis）——本文明首次指出
· 奇异点邻域：特征基 = "幂基的推广"（一般不是多项式，是分片双三次）
```

## Problem

- **应用侧需求**：picking、rendering、texture mapping 都要求"给定 (u,v) → 曲面点（及导数）"的快速精确求值；
- **学界的"墙"**：光滑性证明（Doo-Sabin 1978 / Ball-Storry 1988 / Reif 1995 / Peters-Reif / Halstead-Kass-DeRose 1993）都基于特征分析，但**只用了特征空间的一个子集**（为了证光滑性足够了），"**没有一篇解决'到处都能求值'的问题**"；
- **迭代逼近的两个硬伤**（原文）：奇异点附近"too expensive"，且"does not provide exact higher derivatives"；
- 后果（结论段原话）：**"缺少这样的求值方案，一直是被引为反对在自由曲面建模器中使用细分方案的首要论据"**（"sited as the chief argument against the use of subdivision scheme in free-form surface modelers"）。

## Historical Context

```text
1978  Catmull-Clark 定义规则（递归⇔极限）——"发明的第一天就留下求值问题"
        ↓ 光滑性证明线（都基于特征分析，但只用子集）
1978  Doo-Sabin：奇异点邻域行为分析（离散傅里叶方法首次用于细分）
1987  Loop：三角细分方案
1988  Ball & Storry：递归 B-spline 曲面的切平面连续性条件
1993  Halstead-Kass-DeRose：用特征分析做"公平插值"（(2N+1) 阶细分矩阵）
1995  Reif：奇异点邻域细分算法的统一处理
1997  Zorin 博士论文：一般细分类的 eigenbasis 光滑性证明
        ↓ ★ 1998 本文：把"证明工具"升级成"求值工具"——
          "we have extended a theoretical tool into a very practical one"
1998  DeRose/Kass/Truong（同届 SIGGRAPH）：半锐折痕 + Geri's Game 生产落地
2005/2018  两人与 Catmull 共享奥斯卡技术奖（见文末）
2012  Nießner et al.：特征自适应 GPU 求值（把"直接求值"搬上 GPU）
2010s OpenSubdiv：工业级开源实现（求值器谱系的"事实标准"）
```

**谱系要点**：特征基函数（eigenbasis）概念由 **Warren**（细分曲线理论）提出、**Zorin**（一般细分方案光滑性）推广——但**从没有人给出过某个具体方案特征基的解析表达式**；本文是第一次（"explicit analytical expressions for particular eigenbases have never appeared before"）。

## Previous Work

| # | 引用 | 作用 |
|---|---|---|
| [2] | Catmull & Clark 1978 | 被求值的对象（方案定义） |
| [3] | Doo & Sabin 1978 | 离散傅里叶方法（本文算循环块特征结构的方法来源） |
| [1] | Ball & Storry 1988 | 切平面连续性条件（分析传统） |
| [4] | Halstead-Kass-DeRose 1993 | 特征分析做插值；"文献中常见的细分矩阵"；极限点定义 |
| [5] | Loop 1987 | 三角方案（本文方法的第二实例） |
| [6][7] | Peters-Reif / Reif 1995 | 奇异点邻域的统一分析（光滑性理论前沿） |
| [8] | Stam（同届 CDROM） | **Loop 方案的求值姊妹篇**："Evaluation of Loop Subdivision Surfaces" |
| [9][10] | Warren / Zorin 1997 | eigenbasis 概念的理论先行者 |

> 论文明确说：**方法不限于 Catmull-Clark**——"只要规则区域与某个已知参数表示重合（reif [7]），方法就适用"；Loop 版细节在同届 CDROM 论文 [8]；Doo-Sabin 的扩展细分矩阵一般不可对角化，需用 **Jordan 标准型 + Zorin 的缩放关系**——这是方法的边界条件。

## Core Idea

**"无限过程 → 有限表示"：把递归细分折叠进特征空间。**

1. **矩阵化**：一次细分 = 线性映射（扩展细分矩阵 $A$）。对奇异点片的 $2N+8$ 个控制点（N = 价，Figure 3 的编排顺序），$A$ 具块结构：核心块 = 文献中经典的非奇异点子矩阵，外围块 = 标准 B-spline 中点节点插入规则；
2. **特征分解**：$A = V \Lambda V^{-1}$；$\Lambda^k$ 让"第 k 层控制点"瞬间可得——$v_k = V\Lambda^k V^{-1}v_0$，其中 $V^{-1}v_0$ 是**控制点在特征空间中的投影**（每个片只算一次）；
3. **特征基**：曲面片是"各层新控制点 × 双三次 B-spline 基"的**无穷和**——把 $\Lambda^k$ 的幂**吸进基函数**并预计算：
   $$ \varphi_i(u,v) = \sum_{k}\ \lambda_i^{\,k}\ \tilde{b}_i^{(k)}(u,v) $$
   得到**只依赖价 N 的特征基函数**；极限曲面就是它们的加权和（权重 = 投影控制点分量）；
4. **求值 = 选层 + 求和**：给定 $(u,v)$，用 $n = \lfloor\min(-\log_2 u, -\log_2 v)\rfloor$ **直接算出它落在哪个细分层级**（不需要细分！），然后 $\sum_i \lambda_i^{\,n}\varphi_i(u,v)\,c_i$——这就是全部。

**两个漂亮的结构性质**（原文核出）：
- **缩放的优雅**：第 $k$ 层与第 $k+1$ 层的基函数只差"乘一个特征值"——细分在特征空间里就是**按特征值缩放**（这正是光滑性证明的工作方式，本文把它变成计算方式）；
- **七个共享函数**：任意价的特征基里，**最后七个函数恒相同**——它们正是"外层 7 个控制点"对应的双三次 B-spline 张量基（外圈控制点的影响与价无关——Figure 4 的对照实验）；$N=4$（规则区）时全部特征基 = **幂基**（本文首次指出；此时特征向量矩阵 = 幂基到 B-spline 基的换基矩阵）。

## Technical Approach

- **前提**：网格先做两次细分，把奇异点隔离——此后每个面是四边形、至多含一个奇异点（$N$ = 该点价）；求值目标是这一片的 $2N+8$ 个控制点；
- **层的划分（Ω 瓷砖）**：单位参数域被分成无穷层瓷砖（每层 4× 缩小的三个规则象限 + 中心）；每个瓷砖上曲面恰是一条**双三次 B-spline 片**（三个象限就是三条 B-spline——"四分之三的片直接可参数化"）；$n$ 层选域由 log2 直接给出（$u$ 或 $v$ 为 0 时用一个接近机器精度的极小值兜底，避免 log2 溢出）；
- **特征结构的解析计算**（附录 A）：循环块用**离散傅里叶变换**（Doo-Sabin [3] 首创于细分语境）；其余用小型线性系统求解——原文强调**不用数值特征求解器**（"这些数值例程不总是返回正确的特征结构——有时返回复数特征值"），逐价计算到机器精度（LINPACK `dgesv`）；
- **工程形态**（§5）：特征结构**只算一次**，预计算到 NMAX=500 并存成文件；运行时两个函数：
  ```c
  typedef struct { double L[K]; double iV[K][K]; double x[K][3][16]; } EIGENSTRUCT;  /* K = 2N+8 */
  ProjectPoints(Cp, C, N)   /* 控制点 → 特征空间投影（每片/每次网格更新只算一次） */
  EvalSurf(P, u, v, Cp, N)  /* 求值：选域 + Σ pow(L[i],n)·EvalSpline(x[i][k],u,v)·Cp[i] */
  ```
- **导数**：同构——把 `EvalSpline` 换成"双三次多项式的 p 阶导数"，结果再乘 $2^{np}$（仿射变换导出的因子）；**任意阶导数**因此可得。

## Key Contribution

1. **推翻一个普遍信念**：CC 曲面**可以**在任意参数值处精确求值（非迭代）；
2. **把光滑性证明的工具升级为求值工具**：使用**整个**特征空间（此前的证明只用子集）；首次给出具体细分方案特征基的解析表达式；
3. **成本打平双三次样条**：单次求值 ≈ bicubic spline 成本（多一个 log2 + 一次整数幂）；
4. **打通"参数曲面算法库"到细分曲面**：原文——"让大量为参数曲面开发的算法可以迁移到 CC 曲面"；
5. **首次给出奇异点邻域的高分辨率曲率图**：并指出奇异点处（高斯）曲率**无穷大**——解释了 [4]（Halstead 1993）中能量泛函的发散现象；
6. **可复用的实现资产**：特征结构数据表公开下载（价 3 起算到 500）。

## Why It Works

- **细分规则是线性的**（在齐次坐标下）→ 递归 = 矩阵幂 → 幂 = 特征值的幂；
- **特征值全在单位圆内**（除 λ=1）→ 高阶项快速衰减——"离奇异点越近（n 越大）只有少数几项有显著贡献"，无穷和实际是收敛的、可控的；
- **极限点自洽**：当 (u,v) → (0,0)（奇异点本身）时公式退化为 [4] 中定义的极限点——新公式包含旧结论为特例。

## Limitations

- **推导层门槛**：特征分析、离散傅里叶、非亏损性——数学"involved"（原文自认），学习曲线陡（实现层倒是直白："the tedious task... only has to be performed once"）；
- **适用范围**：要求奇异点隔离（先细分两次）；要求扩展细分矩阵可对角化（CC/Loop 可以；**Doo-Sabin 不行**——需 Jordan 标准型）；
- **参数化是人为的**：片内参数化（瓷砖划分）由作者选定——不唯一，连续性只在片内平滑（参数化本身分段）；
- **数值边界**：u 或 v 为 0 需兜底处理；特征结构数值精度要求机器级（所以必须解析预计算，不能现场数值求解）。

## Game Development Relevance

- **一切 SubD 工作流的隐藏底座**：DCC 视口平滑预览、细分预览、displacement、拾取、射线-曲面求交（Newton 迭代要高阶导数）——凡"给定 (u,v) 求点/法线"的地方都在用它（或其等价形式）；
- **"成本 = bicubic"是产业意义的关键句**：它意味着细分曲面可以和参数曲面**同一预算量级**被消费——细分不再只是"建模表示"，而是"可被渲染/物理/工具直接消费的表示"；
- **对分档的启示（与你的五维预算同构）**：这是"**换表示换成本**"的早期经典样本——"递归逼近（贵、近似） → 特征基展开（便宜、精确）"；同一条推理此前出现过在 GS"取消排序"、毛发"外包积分"……**"把无穷过程预计算成有限和"**，是"性能预算"语言的一种原初形态（1978→1998：从"多细分几次"到"一次算对"）；
- **历史对照**：用你的语言读——就是"**把运行时的迭代开销，换成了预处理（特征结构）+ 常数成本求值**"，与 Turbo/烘焙/预计算光照家族同宗。

## Unreal Engine Relevance

- UE 消费 SubD 的方式以**资产阶段**为主（导入/转换/烘焙，Nanite 吃结果）——但"任意 (u,v) 求值"能力正是资产管线里那些操作（细分、置换、重网格、法线重建）能存在的理由；
- 若做引擎侧 SubD 实验（OpenSubdiv 集成、或自写求值器），本文的"预计算特征结构 + 两层调用"是最小可行路径；**"成本 ≈ 双三次样条"** 是评估"能否把某个预处理搬进编辑器/运行时"的第一句量级判断；
- 与 [[Subdivision Surfaces]] 概念笔记"引擎侧"一节合读：Nanite 时代运行时吃三角化结果，但求值器仍在编辑器/导入链路里活跃。

## Technology Evolution

```text
求值问题的 20 年（1978 → 1998）：
  1978 定义（递归 ⇔ 极限）——求值问题诞生
  → 1978-1995 光滑性证明（特征分析，只用子集）
  → ★ 1998 本文：整个特征空间 → 解析特征基 → 任意 (u,v) 直接求值
  → 1998 DeRose（生产）：渲染/求值进 RenderMan（用 [4] 的极限点法）
  → 2005/2018 奥斯卡（求值问题被公认为"让细分可用"的关键一环）

"递归 ⇔ 直接"的两种命运（对照）：
  · 细分曲面：1998 找到"闭合形式"（特征基）——成功
  · 你的每日主题里另一支：3DGS 的"排序"——2024-2026 选择"取消"而非"求解"（另一种赢法）
```

## Relationships

### Based On

- [[Catmull-Clark — Recursively Generated B-Spline Surfaces (1978)]]——被求值的对象（三条规则）；
- **光滑性证明传统**（Doo-Sabin 1978 / Ball-Storry 1988 / Reif 1995 / Peters-Reif / Halstead-Kass-DeRose 1993）——作者原话："基于首先为证明光滑性定理而开发的技术"；
- **Warren / Zorin 的 eigenbasis 概念**（理论先行者）。

### Extends

- **Halstead-Kass-DeRose 1993**——从"用特征分析做插值"到"用整个特征空间做求值"；
- **Reif 1995 的统一分析**——把"统一处理"从证明推进到算法。

### Related

- [[DeRose — Subdivision Surfaces in Character Animation (1998)|DeRose 1998]]（同届 SIGGRAPH，同期独立）——**理论 ⇄ 生产配对**：Stam 给"怎么算"，DeRose 给"怎么用"（论文互相独立、无引用关系——1998 年夏天之前彼此未见）；
- [[Subdivision Surfaces]]——概念载体（本笔记兑现其"Stam 1998（精确求值）记录位"）。

### Followed By

- **Loop 方案求值**（同届 CDROM 姊妹篇）——方法推广的第一实例；
- **Nießner et al. 2012**（特征自适应 GPU 渲染）——"直接求值"搬上 GPU 的关键一步；
- **OpenSubdiv（Pixar，2012+）**——工业级开源求值实现；
- 全部 DCC 的 SubD 评估器（Maya / Blender / 3ds Max …）——事实上都沿此谱系的思路。

## Personal Knowledge State

- **user_level: Normal（结论层）**。前置：[[Subdivision Surfaces]] 三条规则（已读层）+ "矩阵 = 线性变换、特征分解 = 换基"的常识级印象（推导不必展开）；
- **读法建议（≈25 分钟，跳过 §3-4 的公式）**：摘要（"disprove the belief"）→ §1 引言（应用动机 + "扩展理论工具为实用工具"）→ §4 末的 Eq.14-16 与"七个共享函数 / 幂基"结论段 → §5 实现（EIGENSTRUCT + ProjectPoints/EvalSurf 代码段——**最值钱的 30 行**）→ §6 曲率图与"无穷曲率"结论 → §7 结论；
- **与用户的关系**：**"你每天打开的细分预览"背后的数学**——TA 本行"日用工具的第二层原理"；与 [[Catmull-Clark — Recursively Generated B-Spline Surfaces (1978)|1978 规则原文]] 合读构成建模域"规则 + 求值"两件套。

## Learning Value

1. **"无限过程的有限表示"范式**——把递归/迭代展开成矩阵幂 → 特征分解 → 闭合形式。判据：*面对一个"只能迭代逼近"的过程，先问"它的单步是不是线性的？如果是，特征空间里它就是缩放"*（与"把积分变成查表""把排序取消"同宗，是**预处理家族**的理论形态）；
2. **"信念反转"的样本**：整个学界相信"不可能"，作者用既有工具（特征分析）换个用法就推翻了——**"更厉害的数学"常常不是答案，"用满已有工具"才是**；
3. **"研究 → 实现"的距离感**：论文给的是数学，落地形态是"预计算表 + 两个函数"——**理论论文的工程接口可以极窄**（一张表 + 30 行代码），这对你评估"论文能不能进管线"是第一句提问；
4. **成本对照的行业语法**："cost comparable to a bi-cubic spline"——**用已存在的量级做锚**，让新方案被工程界快速接住（公文写作的范本）。

## Visualization

![[细分曲面_求值与生产化_Stam 1998 与 DeRose 1998 图解.html]]

## Notes

- **核对记录**：作者主页 PDF（dgp.toronto.edu）10 页全文提取（pypdf；含实现代码段与附录）；页码/DOI 由 ACM DL 记录核实（SIGGRAPH '98, pp. 395–404, DOI 10.1145/280814.280945，1998-07-24 出版）；
- **作者背景**：Jos Stam——U of T 博士（导师 Eugene Fiume），后加入多伦多 Alias（Maya 的公司），其研究进入 Maya；除本文外以"流体模拟 Stable Fluids"闻名（两次不同领域的奥斯卡：细分求值 + 流体技术）；论文致谢含 Milan Novacek（Alias）等；
- **2005 奥斯卡的精确分工**（Academy 记录核实）：2005 年度**技术成就奖（Technical Achievement Award）**——"To Ed Catmull for the original concept, and Tony DeRose and Jos Stam for their scientific and practical implementation of subdivision surfaces as a modeling technique in motion picture production"；**2018 年三人再获 S&E Award**（"pioneering advancement of the underlying science of subdivision surfaces"）；
- **姊妹篇**：Loop 版求值（SIGGRAPH '98 CDROM，"Evaluation of Loop Subdivision Surfaces"）——三角方案的同构推导；
- **数据资产**：特征结构下载页（project_page 栏）——价 3 起、含每价的 L / iV / x 系数表。
