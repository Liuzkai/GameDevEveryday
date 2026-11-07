---
type: paper
title: "View-Dependent Refinement of Progressive Meshes"
authors: [Hugues Hoppe]
year: 1997
published: "1997-08-03（SIGGRAPH '97，pp. 189–198）"
venue: "ACM SIGGRAPH 1997 Proceedings（Microsoft Research）"
url: "https://doi.org/10.1145/258734.258849"
code: ""
project_page: "https://hhoppe.com/vdrpm.pdf（作者主页，PDF 直链 200）"
category: [computer-graphics, classical, lod, geometry, scalability, real-time-rendering]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（思想层；具体 PM 结构已被虚拟几何取代）"
user_level: "Easy（机制层）/ Normal（判据与控制器层）"
status: unread
aliases: [Hoppe 1997, View-Dependent Refinement, VDPM, 视相关细化, 选择性细化]
tags: [lod, geometry, scalability, real-time-rendering, budget, classical]
---

# View-Dependent Refinement of Progressive Meshes（Hoppe 1997）

## TL;DR

**LOD 从"离线烘焙的几档"变成"运行时逐顶点的决定"的那一步。** 一句话主张：

> **判据不该是"这个物体多大"，该是"这个顶点在屏幕上会错多少像素"。**

```text
一个渐进网格（PM）= 基网格 M0 + 一串 vsplit 记录
  ↓
选择性细化 = 只应用这串记录的一个【子序列】
  ↓
逐顶点问三件事：
  ① 它在视锥外吗？        → 不细分（包围球 r_v 测试）
  ② 它背对相机吗？        → 不细分（法锥 α_v / Gauss 映射）
  ③ 它的屏幕投影误差 > τ 吗？→ 不细分（偏差空间 D_n̂(σ,τ)）
  ↓
每帧增量地 split / collapse，利用视角连贯性
  ↓
再加一个【控制器】：τ 不是常量，它由"目标面数 m"反推
```

**对库的意义**：[[Level of Detail]] 这个 99 个文件都在引用、却一直没有页面的概念，**中段就是这一篇**。它同时给出了三样东西——判据（屏幕空间误差）、算法（增量前后缀）、控制器（τ 的比例反馈）。**第三样是绝大多数后续工作漏掉的。**

## Problem

1996 年的 PM（*Progressive Meshes*, SIGGRAPH '96）已经能把任意网格压成一串 `vsplit`，但它优化的是**视无关**的 LOD：整条序列只有"更粗/更细"一个自由度。

这在两类场景上直接失效：

1. **包围观察者的环境**（建筑漫游、地形、NURBS 面片）：视锥外与背面的面片占了大半，视无关的序列**仍然要把它们按同一精度画出来**。剔除能省一部分，但"剔除"本身有成本，而且**视锥内的深度差异被浪费了**——近处该细、远处该粗，而视无关的 LOD 给所有区域同一个精度（原文 Figure 12b 就是这个失败的样本）。
2. **分块 + 各块独立 LOD** 的补救方案会引入两个新问题：块与块之间的**裂缝**（crack），以及**块边界不对应曲面特征**（人在分块边界恰好是轮廓时最明显）。

作者的原话把目标写得很干净：**"理想解是高效地确定满足某些感知图像质量的最粗模型。"**

## Historical Context

```text
1976  Clark：层级几何 + 按屏幕占比变细节 + working set（理论，未实现）
        · 判据 = 几何代理量（面积），不是误差
  ↓
1996  Hoppe：Progressive Meshes —— 任意网格 ⇄ 一串 vsplit
        · 解决了【表示】，但只有"更粗/更细"一个自由度（视无关）
  ↓
1996  Xia & Varshney：顶点层级 / merge tree，支持实时选择性细化
        · 用边长建层级，限制每层 ecol 邻域不重叠 ⇒ 只能【逐级渐变】
  ↓
1996  Lindström 等：高度场的实时连续 LOD
        · 定义了垂直偏差空间 D_ẑ(τ)，本文的 σ→0 特例
  ↓
1997 ★ 本文：任意网格的视相关选择性细化
        · 新判据（屏幕空间几何误差）+ 新算法（增量、可摊销）+ 控制器
  ↓
2000s HLOD / 分块流式：把误差从"逐顶点"聚合到"逐簇"
  ↓
2020s 虚拟几何：同一判据，逐簇求值，决策搬到 GPU
```

**两条前驱的差别值得单独记**：Xia-Varshney 的层级是**用边长建的**、且强制"每层变换邻域不重叠"，后果是细节只能**逐级渐变**（类似四叉树）；本文反过来——**让层级由几何优化的 vsplit 序列自由决定，只保留最少的依赖**。代价是层级不平衡（bunny 24 层 vs Xia-Varshney 65 层），收益是同样误差下**面数更少**（因为不受规整细分结构约束）。

## Previous Work

| 路线 | 做法 | 局限 |
|---|---|---|
| 视锥/遮挡剔除 | 整块整块地不进管线 | 可见部分仍可能远超一帧预算（Soda Hall 有视点可见 8 万多面） |
| 分块 LOD 层级 | 每块独立选档 | 块缝裂缝；块边界不对应特征；层级建立本身很难 |
| 高度场四叉树 / 四分三角细分 | 规整细分 ⇒ 连接受限 | 同样精度下需要**更多三角形**（TIN 更省） |
| NURBS 自适应细分 | 参数域规则采样 | 要处理参数步长、裁剪曲线、面片缝合 |
| Xia-Varshney merge tree | 边长建层级，逐级解冻 | 只能逐级渐变；层级深 |

**作者对"动态 TIN 太贵"这一当时共识的正面反驳**正是本文卖点之一：原文引 Lindström 1996 说"在交互速率下动态适配 TIN 被认为代价过高"，然后说"本文演示了高度可适配 TIN 的实时修改"。

## Core Idea

### ① 选择性细化 = vsplit 序列的一个子序列，但要先重定义变换

朴素的"取子序列"不成立——`vsplit` 只有在当前网格满足前置条件时合法。作者试过给旧定义补前置条件，**失败**（约束过强，框架不自洽），于是**从头重定义 vsplit / ecol**：

- 新 `vsplit(v_s → v_t, v_u, f_l, f_r, f_n0..f_n3)`：父顶点 $v_s$ 分裂成两个子顶点 $v_t,v_u$，新建两个面 $f_l,f_r$，依赖四个邻居面；
- **顶点在每次分裂时被重命名**（$\mathcal{V}$ 的规模约为原顶点数的 2 倍）——重命名本身就是依赖关系的载体；
- 合法性只需两条：$v_s$ 活跃 + 四个邻居面活跃（ecol 对偶）。

由此得到"选择性细化网格"$\mathcal{M}^S$ 与**顶点前沿**（vertex front）：基网格的顶点是森林的根，最细网格的顶点是叶，任一选择性细化网格 = 一条横切森林的前沿。

### ② 三个判据，第三个才是新的

```text
qrefine(v_s):
  if outside_view_frustum(v_s) return false     ← 包围球 r_v（后序遍历聚合）
  if oriented_away(v_s)        return false     ← 法锥半角 α_v（Gauss 映射）
  if screen_space_error(v_s) < τ return false   ← 偏差空间 D_n̂(σ_v, τ_v)
  return true
```

**偏差空间** $D_{\hat n}(\sigma,\tau)$ 是本文的技术核心，形状是"半径 $\sigma$ 的球 ∪ 半径 $\tau$ 的'双锥'（bialy）"：

- $\tau$ 沿法向（$\hat n$）方向的分量——**曲面内插误差的主要方向**；
- $\sigma$ 均匀分量——$\tilde N_v$ 弯曲时必需，也**让不连续曲线（曲面边界、材质边界）能被准确逼近**（这类偏差往往切于曲面，纯法向空间表达不了）。

屏幕投影半径有一个极简闭式：$\max(\sigma,\ \tau\,|\hat n\cdot \hat v|)$。**$\hat n \perp \hat v$（轮廓处）时退化为 $\tau$，$\hat n \parallel \hat v$（正对）时退化为 $\sigma$** —— 这就是"轮廓处细化更密"的来源，不是启发式，是几何（Figure 13b）。

实现上只存 4 个标量 $\{r_v,\ \sin^2\alpha_v,\ \sigma_v^2,\ \tau_v^2\}$，三项测试共享子表达式，**单次调用 230 cycles**。

### ③ 算法：该细就强制细，该粗只在合法时粗

```text
for each active vertex v:
    if can split and qrefine(v):        force_vsplit(v)
    elif can collapse and not qrefine(parent(v)):  ecol(parent(v))
```

`force_vsplit` 用一个显式栈把"为了让 $v$ 合法而必须先做的那些分裂"依次做完——**栈的存在是因为合法性有依赖，而依赖不能靠拓扑保证**。

复杂度 $O(|\mathcal{V}_A|+|\mathcal{V}_B|)$（最坏情况），**连续视角变化时 $|\mathcal{V}_B|\approx|\mathcal{V}_A|$，瓶颈退化为一次链表遍历**。

### ④ 控制器：标题写帧率，正文调面数（最容易被漏的一节）

固定 $\tau$ 会让面数在 1 千到 200 万之间摆（Figure 9），于是：

$$\tau_t=\tau_{t-1}\cdot\frac{|F_{t-1}|}{m},\qquad \tau\ \text{不低于}\ 0.5\ \text{像素}$$

作者明确解释为什么不直接调帧率：**"帧率对操作系统'打嗝'更敏感"**，所以应改用**更慢的外层控制器去调 $m$**。摊销（每帧只遍历一部分 $\mathcal{V}$）之后响应会滞后，于是又加一条：$|F|$ 到 $1.2m$ 就不再 split，但把"本可引入的面数"计入下一次 $\tau$ 调整。

> **这是全篇最可迁移的一段。** 判据（$\tau$）只回答"要不要"，配额（$m$）才回答"多少"，两者之间**必须有一个控制器**；而控制器要控的是**稳定的量**，不是**想优化的量**。

## Technical Approach

| 环节 | 做法 | 代价 |
|---|---|---|
| PM 构建 | 离线，几何优化序列 + 一个平衡惩罚项 $c\,(n_{v_t}+n_{v_u})$ | canyon600 **627 分钟**（离线） |
| 加载 + 建层级 + 算 $r_v,\alpha_v$ | 载入后一次性 | < 1 分钟（大头在算 $r_v,\alpha_v$） |
| 每帧细化 | 遍历活跃顶点链表，增量 split/collapse | **14% 帧时间**（摊销后 8%） |
| 判据求值 | 4 个标量 + 共享子表达式 | **230 cycles/次（4% 帧时间）** |
| vsplit / ecol | 常数时间 | 2200 / 4000 cycles |
| **每帧重建三角带** | 贪心，边遍历边拼 | **600 cycles/面，26% 帧时间** |
| 内存 | 未优化 224 B/顶点（可压到约 140 B） | canyon600 **80.6 MB** |

**这张表里最反直觉的一行是三角带。** 决策只要 4% 帧时间，**把决策结果翻译成渲染器能吃的格式要 26%**——因为连接关系变动态了，预计算好的 strip 全作废，只能每帧贪心重建（平均 10–15 面/广义带 @IRIS GL，4.2 面/顺序带 @OpenGL）。

## Results

**削减幅度（同一份 PM，只改 $\tau$）：**

| 模型 | 原始面数 | 视相关后 | $\tau$ | 帧率变化 |
|---|---|---|---|---|
| Grand Canyon（俯视） | 79,202 | **33,119**（$\tau=0$，仅视锥） | 0% | — |
| Grand Canyon（俯视+常规） | 79,202 | **10,013** | 0.33%（600×600 下 2 px） | — |
| 球体 | 19,800 | **1,422** | 0.075% | — |
| 参数曲面（截断 PM） | 25,440 → 10,000 | **1,782** | 0.15% | **14.7 vs 6.8 fps** |
| gameguy | 42,712 | **3,157 / 2,559** | 0.6% | — |
| **bunny** | **69,473** | **10,528** | 0.1% | **6.7 vs 1.9 fps（3.5×）** |

**CPU 账（150 MHz R4400）：** 细化 14% / 渲染 26% / GL 库 19% / OS+图形 21% / **空闲 20%**。飞行漫游平均 7.2 fps，摊销后 8 fps；换 Impact 图形系统后 7.2 → 14.0 fps。

**平衡惩罚项（§7）**：$c$ 从 0 增大时，层级变浅（bunny 24 层附近），面数先略降；$c$ 再大就**四叉树化**，结果"markedly"变差（Figure 17）。结论：**在没有几何偏好时，轻微偏向平衡层级是有益的；偏向过头就把自由度还回去了。**

## Key Contribution

1. **判据的单位从"物体"缩到"顶点"**，且判据本身是**误差**而非代理量——这让"总预算"第一次可以被写成"求和 < 预算"（误差可加）。
2. **给出"增量 + 摊销"的可运行算法**：不是每帧从零算一遍该画什么，而是"上一帧的前沿 + 少量 split/collapse"。
3. **给出控制器**：$\tau \leftarrow \tau\,|F|/m$。这是把"感知阈值"接进"帧预算"的那根线。
4. **geomorph**：任意两个选择性细化网格之间可插值（要求中间网格 $M_G$ 的前沿处处不高于两者），用于消 pop。代价是**要一个面集是两者超集的中间网格**。

## Why It Works

**因为"屏幕空间误差"是一个可加、可阈值、可逐顶点求值的量，而"面积/距离"不是。**

- 面积和距离**不能跨物体相加**（一平方米的近处墙面与一平方米的远处墙面，误差不同）；
- 误差可以——所以只有误差能拿来做**场景级总预算**；
- 而一旦可加，"这一帧画多少"就变成一个**优化问题**（在预算内最小化总误差），而不是"给每类物体配一个阈值表"。

第二条原因更工程：**视角连贯性**。相邻两帧的前沿几乎相同，所以"遍历 + 少量改动"的增量做法能摊销到 8%；如果每帧从零求解，代价会高一个量级。

## Limitations

1. **三个"性质"没有形式化证明**。原文脚注直说："这些性质在我们大量实验中成立，但**我们尚未给出形式证明**。"（唯一性 / 合法性闭包 / 与补序列等价）
2. **误差界只是近似**，作者列了三条：远离视口中心时**低估**（Lindström 已指出）、投影用了**平行投影假设**、以及**偏差空间只在顶点处是严格界**（选择性细化后 $v$ 的邻域可能与 PM 序列里的不同）。
3. **内存不划算（尤其高度场）**：224 B/顶点，而规整网格方案每顶点只存一个标量。作者承认"对高度场而言远超规整网格方案"。
4. **参数曲面的最细分辨率是先验固定的**（先密网格 → PM → 截断），不能按需生成更细。
5. **不含动画模型**（作者列为 future work），也不含碰撞检测——**动态拓扑与"每帧重建 strip"这一对矛盾被留在了原地**。
6. **绝对帧率按今天标准不高**：6.7 fps（bunny）——不是算法问题，是 1997 年硬件；但**它说明"能省 3.5 倍"和"能跑 60 帧"是两件事**。

## Game Development Relevance

**5/5 —— LOD 这条线的"判据 + 控制器"两份说明书都在这一篇里。**

1. **判据的选择决定你能做什么**：用距离/屏幕占比 ⇒ 只能逐物体设阈值；用屏幕空间误差 ⇒ 可以做**全局预算**。今天的虚拟几何、GS-LOD 全部走后者。
2. **控制器比判据更容易被忽略**：库内三篇 GS-LOD 论文（[[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]] / [[2026-10-21-Beche — KoalaGS Complexity-Aware Continuous Level of Detail for Gaussian Splatting|KoalaGS]] / [[2026-10-14-GLoD-LVSM — Learning Geometry-Grounded Continuous-LoD Scene Representations for Large View Synthesis Models|GLoD-LVSM]]）都在讲"怎么分配"，**没有一篇给出 $\tau \leftrightarrow m$ 这种闭环**。这是一个可以直接搬进自己项目的缺口。
3. **"翻译费"这一条对引擎侧最实用**：本文的 26% 花在**每帧重建三角带**上——不是算错了，是渲染器只吃 strip。**评估任何"每帧重新决定"的方案时，先问"下游接受的格式是不是静态的"。**
4. **geomorph 的代价是"超集网格"**：消 pop 不是免费的，它要求一个包含两边所有面的中间网格。这解释了为什么工业界更爱用 alpha 淡入淡出（不需要超集）。

## Unreal Engine Relevance

- **直接对应**：`r.ScreenSize` 系列 LOD 阈值——**用的正是"屏幕占比"这一支（Clark 1976），不是屏幕空间误差**。这就是"离散 LOD 链"路线在今天的形态。
- **Nanite 用的是本文这一支**：每簇误差界 + GPU 侧判定，把"逐顶点"换成"逐簇"以求并行度；**但它同样需要控制器**（Nanite 的 `Max Pixels Per Edge` 与分辨率缩放本质上就是 $\tau$）。
- **没有对应物的部分**：geomorph。UE 的 LOD 过渡靠 dithering/淡入，不需要超集网格——**这正是"超集"代价被绕开的地方**。
- **可迁移的一条**：Niagara 的粒子距离剔除是"代理量判据"；若改成"屏幕空间尺寸误差"，就能和总预算挂钩（见 [[Level of Detail]] 的 Next Step）。

## Technology Evolution

```text
1976  Clark：层级 + 屏幕占比 + working set（判据 = 代理量）
  ↓
1996  Hoppe：Progressive Meshes（表示问题解决，视无关）
  ↓
1997  ★ 本文：屏幕空间误差判据 + 增量算法 + 控制器（视相关）
        · 但三角带要每帧重建：26% 帧时间花在"翻译"上
  ↓
2000s HLOD：误差聚合到簇，逐物体决策变逐簇决策
        · 裂缝问题用"共享边界"解决，而不是用"单一全局网格"
  ↓
2010s 硬件细分 / GPU 侧剔除：决策开始离开 CPU
  ↓
2020s 虚拟几何（Nanite 类）：逐簇误差界 + 纯 GPU 路径
        · 把本文的"动态拓扑 + 重建 strip"矛盾彻底绕开：不解 strip，直接簇化
  ↓
2026  GS-LOD：判据换成"每 splat 的协方差投影"，可加性问题重新出现
```

## Relationships

### Based On

- **[[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)|Clark 1976]]**：层级 + "细节随观察条件变"的定义句。"可见复杂度固定上限"是本文控制器的思想前身。
- **Hoppe 1996 *Progressive Meshes***（库外）：表示层；本文只改"怎么用它"，不改表示。

### Extends

- **Xia & Varshney 1996**（库外）：同样是 vsplit/ecol 层级，但本文**放弃"每层邻域不重叠"的约束**，换取更浅的层级与更少的面数。

### Improves

- **Lindström 1996**（库外）：把高度场的垂直偏差空间 $D_{\hat z}(\tau)$ 推广到任意曲面 $D_{\hat n}(\sigma,\tau)$——**$\sigma=0,\ \hat n=\hat z$ 时严格退化为前者**（原文明确说了这一点）。

### Related

- **[[Level of Detail]]**（今日新建，第 58 个概念）：本文是它的"判据 + 控制器"两节的唯一来源。
- **[[Scalability and Quality Tiers]]**：同一套 Clark 血统，但决定频率差三个数量级（档位一次 vs 每帧）。
- **[[GPU-Driven Rendering]]**：本文的"逐顶点判定"在 2020s 被搬到 GPU 上变成"逐簇判定"，**判据没变，执行者变了**。
- **[[Streaming]]**：本文的 working set 一侧（该驻留多少）没有展开，是 [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)|Clark 1976]] 第 4 条改进的延续。

### Followed By

- 虚拟几何（Nanite 类）：同一判据、逐簇、纯 GPU。
- [[2026-10-21-Beche — KoalaGS Complexity-Aware Continuous Level of Detail for Gaussian Splatting|KoalaGS 2026]] / [[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS 2026]]：把 LOD 搬到高斯——**判据从"顶点偏差"变成"协方差在屏幕上的投影"，但"误差可加"这一条需要重新证明**。

## Personal Knowledge State

- **机制层 Easy**：PM / vsplit / 选择性细化 / geomorph——概念上不新。
- **判据层 Normal（值得动手）**：① **"误差可加"是全局预算的前提**——这条把 Clark 的"面积"与 Hoppe 的"像素"彻底分开；② **控制器**：$\tau \leftarrow \tau\,|F|/m$，以及"控稳定的量、不控想优化的量"；③ **翻译费**：26% vs 14%，下游格式的静态性决定这笔钱。
- **Hard（不必读）**：偏差空间 $D_{\hat n}(\sigma,\tau)$ 的完整推导与 $\sigma/\tau$ 比值的预计算（残差向量集上取比值、再取最小包围）——**只需要知道"它是一个沿法向拉长的椭球状偏差域、投影半径是 max(σ, τ|n·v|)"这一句**。

## Learning Value

1. **判据的形状决定系统的天花板**：代理量判据只能做逐物体阈值，误差判据才能做总预算。**换判据比换算法值钱。**
2. **"决定"和"执行决定"是两笔账**：230 cycles vs 600 cycles/面。**做性能方案时，先量后者。**
3. **控制器要控稳定的量**：想优化帧率就控面数，想优化画质就控误差——**别直接控那个抖的量**。
4. **"放弃约束"也是一种优化**：去掉"每层邻域不重叠"让层级从 65 层降到 24 层，代价是要写一个更复杂的 `force_vsplit` 栈。**约束越少、依赖管理越重**——这是所有增量系统的共同形态。

## Visualization

**Figure 11 —— 一个网格 = 一条连续的 LOD 序列（1 → 1,000 → 10,000 → 79,202 面）**

![[Research/Files/Pictures/Hoppe 1997 — 图11 渐进网格序列（1至79202面）.png]]

> **读图**：从左到右是同一个 PM 的四个位置。注意最左是**1 个面**——基网格可以粗到这种程度，因为"细化"是逐条记录应用的，不是逐档切换的。这就是"连续"的含义：**没有 3 档，只有一串可以停在任意位置的序列**。（Hoppe 1997，Figure 11）

**Figure 12 —— 同一份 PM，只改 $\tau$：79,202 面 vs 10,013 面**

![[Research/Files/Pictures/Hoppe 1997 — 图12 视相关细化（视锥+误差容限+三角带）.png]]

> **读图**：上排橙色线是**视锥**——锥外的地形整片塌到最粗（(a) 里锥外几乎是平面）。上排右是锥内按 $\tau=0$ 画（33,119 面），中排左是 $\tau=0.33\%$（600×600 下 2 px，**10,013 面**）。中排右的纹理图与最右的三角带图对照看：**面数只剩 1/8，视觉差异要靠放大才看得出来**——而三角带是每帧重建的（764 条），**这一步吃掉 26% 帧时间**。（Hoppe 1997，Figure 12）

**Figure 16 —— 兔子：69,473 → 10,528 面，6.7 fps vs 1.9 fps**

![[Research/Files/Pictures/Hoppe 1997 — 图16 兔子模型 69473面降至10528面.png]]

> **读图**：最右是俯视 + 视锥（橙线），锥外的背侧整片粗化，(b) 是正视图 10,528 面。**同一视点，全画 1.9 fps，选择性细化 6.7 fps——3.5×。** 但注意：6.7 fps 本身仍不"交互"，这说明**省下的倍数与最终能否 60 帧是两件事**（1997 年的 SGI Indigo2，150 MHz）。（Hoppe 1997，Figure 16）

## Notes

- **出版信息**：ACM SIGGRAPH 1997 Proceedings, pp. **189–198**（作者主页 hhoppe.com 与 SIGGRAPH History Archive 双向核对；注意不要与同年的 *Progressive Simplicial Complexes*（pp. 217–224）混淆）。
- **原文获取**：`https://hhoppe.com/vdrpm.pdf`（4.53 MB / 10 页），**作者主页第 6 次一次成功**。同日在 `cs.princeton.edu` 的三个镜像（课程归档 / ~funk / http）全部 502/000，CiteseerX 与 Wayback 均 429——**"经典线优先作者主页"这条已连续 6 次成立**。
- **口径提醒**：Table 2 的百分比是**同一台机器上的帧时间占比**（150 MHz R4400 + 128 MB），不是"算法开销占多少 FLOPs"；其中 CPU 空闲 20%，说明**当时瓶颈在图形侧**。引用"14%"时要连"占用的是帧时间、且总计仍有 20% 空闲"一起说。
- **一处原文表述需注意**：摘要写 "less than 15% of total frame time"，正文 Table 2 写 14%，摊销后写 8%——三者不矛盾（不同配置），但引用时最好指明是哪一个。
