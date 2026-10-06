---
type: paper
title: "Hierarchical Geometric Models for Visible Surface Algorithms"
authors: [James H. Clark]
year: 1976
published: "1976-10-01 (CACM 19(10): 547–554; SIGGRAPH '76 同题摘要 p.267, 1976-07-14)"
venue: "Communications of the ACM, Vol. 19, No. 10（SIGGRAPH '76 会议版为 1 页摘要）"
url: "https://doi.org/10.1145/360349.360354"
code: ""
project_page: ""
category: [computer-graphics, classical, lod, visibility, scalability, engine-architecture]
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（思想层；原文系统当时未实现）"
user_level: "Easy（机制层）/ Normal（成本模型层）"
status: unread
aliases: [Clark 1976, Hierarchical Geometric Models, 层级几何模型, LOD 起源]
tags: [lod, scalability, visibility, rendering, engine-architecture, budget]
---

# Hierarchical Geometric Models for Visible Surface Algorithms（Clark 1976）

## TL;DR

**LOD（层级细节）与"分档"思想的起点论文。** 一句话主张：**几何结构不该只用来摆放物体——它应该被用来加速图像生成的每一步。**

```text
把场景组织成树（几何层级）
  ├─ 每个节点带包围体（bounding volumes）
  ├─ 每个物体可在多个细节层级上表示（different levels of detail）
  └─ 于是：
       · 视景体裁剪 = 对"可分辨部分"的【对数搜索】
       · 细节量随【屏幕占比】与【相机/物体运动】而变  ← LOD 的核心定义句
       · "工作集"（graphical working set）= 内存里该驻留多少结构
       · 递归下降可见面算法 —— 计算时间随【可见复杂度】增长（近线性），
         而不是随【物体空间复杂度】超线性增长
```

**对库的意义**：本库五维预算、五档画质、以及反复出现的"降档 = 换表示层级"判据，**共同的祖先这一篇**。原文摘要的目标句（可引用）：*"…designing a visibility algorithm in which the computation time grows linearly with the visible complexity of the scene."*

> ✅ **核验状态（2026-09-30 结清）**：**原文扫描件已获取并逐段核对**——OSU Pressbooks 镜像（`ohiostate.pressbooks.pub/app/uploads/sites/45/2017/09/clark-vis-surface.pdf`，8 页全文，与 Wayne Carlson 历史档案同源扫描件的站内镜像；Wayback 此前 429 的同一个文件，无需经过 Wayback）。核验方式：全文文本抽取 + 关键段落逐条核对。**9-29 挂账的三项复核清单已全部完成**（见笔记末尾 Notes）；本笔记以下内容为一手材料。

## Problem

1976 年"画复杂场景"的核心矛盾：**场景数据库的组织方式跟不上复杂度的增长**。原文摘要开篇：

> "The geometric structure inherent in the definition of the shapes of three-dimensional objects and environments is used **not just to define their relative motion and placement**, but also to **assist in solving many other problems** of systems for producing pictures by computer."

即：当时几何数据（层级结构）只被当作"摆放与运动的分组"，没有被当作**加速结构的来源**。而复杂度在涨、可见的像素数（分辨率）却被硬件锁死——**两者之间缺一个"按可见性归一的表示"**。

## Historical Context

```text
1974  Catmull ①曲面细分显示；Clark 的 Utah 博士论文（B-spline 曲面）
1975  Newell 等：利用过程模型（procedure models）做图像合成——"细节按需生成"的先声
1976 ★ 本文（CACM 10 月）：层级几何模型 + LOD + working set + 递归下降
1976  （同年）Warnock 分治可见面（已发表）；Gouraud / Phong 着色出现
1978  Williams 深度图阴影（[[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]）
1982  Clark 创办 Silicon Graphics（背景信息，非论文内容）
```

原文参考文献里有一条关键线索：**Denning 1968《The working set model for program behavior》**——本文的 "graphical working set" 直接借自虚拟内存的 working set 概念（摘要明说："in conjunction with a storage hierarchy of the sort used in virtual memory computing systems"）。

## Core Idea — 五个"显著改进"（摘要逐条）

| # | 原文表述（摘要） | 今天的对应物 |
|---|---|---|
| 1 | "The range of complexity of an environment is **greatly increased** while the **visible complexity** of any given scene is kept within a **fixed upper limit**." | **分档哲学的第一原理**：场景总量可以涨，单帧可见上限固定——你五档矩阵的祖先表述 |
| 2 | "A meaningful way is provided to **vary the amount of detail** presented in a scene （according to the **screen area** occupied by the objects… and according to **camera and object motions**）" | **LOD 的定义句**：细节随观察条件变——分辨率占比 + 运动（运动项即"时间连贯性"的最早形态） |
| 3 | "**Clipping** becomes a very fast **logarithmic search** for the resolvable parts of the environment within the field of view" | 层级剔除（hierarchical culling）：树 + 包围体 → 视锥裁剪加速 |
| 4 | "Frame to frame coherence and clipping define a graphical **'working set'**, or fraction of the total structure that should be present in **primary store** for immediate access" | **流式加载 / 驻留预算的祖先**（World Partition、虚拟纹理、Mega Geometry 2.0 streaming 的史前版本） |
| 5 | "A **recursive descent** visible surface algorithm in which the computation time potentially grows **linearly with the visible complexity** of the scene" | "成本函数关于什么线性"思维——本库从 [[Shadow Mapping]] 到 Mega Geometry 反复使用的同一问法 |

### 结构（一手核对，2026-09-30）

- **树状层级**：整个环境本身是一个 "object"，表示为**根树（rooted tree）**；弧有两类——**变换（transformations）**与**指向更精细结构的指针（恒等变换）**；
- **节点的"充分性"定义（原文）**：每个非终端节点在"**其在屏幕上覆盖不超过某个小面积**"时即构成该物体的 *sufficient* 描述；覆盖超过 *critical maximum area* 时，用其子节点（更精细版本）替换；终端节点 = 多边形或曲面片。人体验证例：远到 3–4 个光栅单位 → 单个体块；约 16 个光栅单位 → 四肢/头/躯干一组体块；近到指尖 → 若干曲面片；
- **最小包围信息**：裁剪所需的最小信息 = **包围球的中心与半径**（原文："The minimum necessary information is the center and radius of a bounding sphere."）；
- **裁剪 = 被面积测试截断的对数搜索**：下降一层的判据是**面积测试**，纳入/剔除的判据是**视锥边界测试**（"Clipping therefore resembles a logarithmic search that is truncated by the area (resolvability) test."）；
- **遮挡剔除的两个体（术语细节）**：每个 object 定义 **occluded volume A**（A 被遮 → 整个 object 被遮；所有 object 都存在）与 **occluding volume B**（B 遮挡某物 → 该物必被此 object 遮挡；开圆柱 / 透明物体可能不存在）；
- **递归下降算法的具体形态**：每层对所有 object 按其包围体排序；遮挡测试可整体消去子树；若两包围体**在三个维度上都重叠**（可能相交），其子代在下一层递归中"当作有相同父节点"处理；递归到终端节点 → 原语的快速排序；**理想条件下计算时间随可见复杂度线性**。原文还建议把"裁剪"与"递归下降"两段下降**合并**（减少被遮挡对象的面积测试、减小 working set）。**并行性备注（1976）**：单处理器时合并为一个算法；多处理器时按流水线与否决定合并或分离——GPU 时代的预演。

### 一手核对新增的四条细节（2026-09-30）

1. **中心加权细节（foveation 的 1976 先声）**：分辨率上限不必均匀——"**允许物体覆盖的最大面积，越靠近视场边缘可以放得越大**"，原文自比"相机的中心加权测光"；
2. **运动自适应细节**：运动物体"**细节量与其速度成反比**"（依据：人眼扫视抑制 + 相机运动模糊）；"**整幅画面在相机运动时都可以用更少的细节**"；
3. **排序改进的精确账（原文自算）**：理想无重叠二叉层级中，朴素做法是 m log₂m（m = 2ⁿ 个终端节点）；利用结构逐层测试只需 **p(2ⁿ−1) ≈ pm** 次——**排序时间从 m log₂m 降到线性**；
4. **数据库构建管线（1976 就给了）**：更粗的高层描述用 *bottom-up pruning*（由最精细版本向下剪枝）；更细的低层描述用 *top-down splitting*（Catmull 式曲面片细分）；何时生成（显示时 vs 离线）是"**传统的 time/space tradeoff**"——LOD 自动生成与烘焙策略的祖先。

## Why It Works（为什么这是"第一原理"级论文）

1. **它把"表示选择"从资产属性变成运行时决策**：同一物体的多个表示是**同时存在**的，用哪个由观察条件实时决定——这就是今天 LOD/虚拟几何/流式系统的全部结构；
2. **它把"复杂度"分成两个概念**：物体空间复杂度（数据库总量）vs **可见复杂度**（本帧真正要被处理的量）。**整个实时渲染三百年史（夸张，但方向如此）就是不断把"成本 ∝ 物体空间复杂度"改写成"成本 ∝ 可见复杂度"**；
3. **它引入"驻留 = 预算"的思维**：working set 让内存占用成为可讨论的、按帧变化的量——**"资源预算"作为一个运行时概念，最早在这里成型**。

## Limitations

- **原文是"短理论论文"（UCSC 史述原话："a short theoretical paper"），当时未见实现**："Although Clark's methods were apparently not implemented at the time, they foreshadowed subsequent developments"（UCSC 报告）；
- **递归下降剔除隐藏面的实际收效有限**：UCSC 报告评述"would probably cull only a small fraction of the hidden geometry in a typical scene"——遮挡剔除的真正解决要等后续深度缓冲体系；
- 层级结构的**建立与维护成本**（当时的数据库工具尚不存在）——这条限制今天依然以"资产必须按层级组织好"的形式存在（Nanite 的 cluster 构建、Mega Geometry 的预烘焙同理）。

## Game Development Relevance

**5/5 —— 它就是"分档"与"LOD"这两件事的起点，也是你五维预算体系可以引用的最上游依据。**

1. **"可见复杂度固定上限" = 五档画质的第一原理**：你的每一档都在定义"这一档允许的可见复杂度上限"（粒子数、灯光数、贴图尺寸……），Clark 1976 是这个思想的元表述；
2. **"细节随屏幕占比变化" = 分档参数的原始合法化**：为什么贴图/模型/粒子可以按距离/占比降级而不被骂"偷工减料"——1976 年给出的判据是"**当细节已不可分辨时，多余细节没有价值**"；
3. **working set = 流式系统与显存预算的祖先**：今天 Mega Geometry 2.0 的 "drop detail rather than thrash"（9-28 记录）、World Partition、虚拟纹理——都在回答 Clark 的同一问题"primary store 里该放多少"；
4. **与 1978/1983 两篇构成完整的三点谱系**（更新 [[Scalability and Quality Tiers]] 的"分档的祖先"表）：
   - **Clark 1976**：分档体系本身（层级 + 可见上限 + working set）→ *"为什么要分档"*
   - **[[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]**：单维成本法则（灯光/贴图）→ *"每一维的成本函数"*
   - **[[Reeves — Particle Systems (1983)]]**：粒子成本法则 → *"粒子维度的成本函数"*

## Unreal Engine Relevance

- **今日引擎里的直接化身**：Levels of Detail（模型 LOD 系统）/ HLOD / Nanite（虚拟几何＝连续 LOD + 按分辨率取细节）/ World Partition（working set）/ 纹理流送（Mip = 细节层级随屏幕占比）；
- **"分档=按观察条件选表示"在 UE 的落地**：`r.ScreenSize` LOD 阈值、Nanite 的 cluster 选择、Scalability 系统的质量组——**同一个原理的 4 个工程出口**；
- 与 [[Niagara]] 的遥远关联：粒子系统的"距离剔除 / 重要性分级"同理。

## Technology Evolution

```text
1976 ★ Clark：层级几何 + LOD + working set + 递归下降（理论）
  ↓ 飞行模拟器率先落地（LOD 的工业首用）
1990s 多分辨率网格（multiresolution / progressive meshes：Hoppe 1996 等）
2000s HLOD / 分块流式（开放世界世代）
2010s 虚拟几何思想成型（GPU 驱动 + 剔除）
2020s Nanite（连续 LOD 的实际工程化）
2026 ★ Mega Geometry 2.0：光追几何的"流式 + drop-detail"（9-28 记录）
       ——回到 Clark 的第 4/1 条：working set + 超预算降细节，而不是抖动
```

## Relationships

### Extends

- **[[Scalability and Quality Tiers]]**：该概念的"第一原理"章节即本文——"可见复杂度固定上限"是五档定义的上游依据。

### Related

- **⟷ [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]]**："细节表示跟着观察尺度走；小到不该用几何时，预算从几何维度迁移到纹理维度"（本库 9-25 归纳的"表示律"）——**这正是 Clark 第 2 条改进在 13 年后的具体化**；两句合起来是"分档阶梯"的 1976 定义 + 1989 实例；
- **⟷ [[Reeves — Particle Systems (1983)]] / [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]**：本库"预算起源三件套"的第三篇（且是时间最早的一篇）；
- **⟷ [[Shadow Mapping]]**："成本函数关于什么线性"——本文问"相对物体空间复杂度"，Williams/Reeves 问"相对什么"；同一思维在不同维度；
- **⟷ [[GPU-Driven Rendering]]**：簇剔除（cluster culling）的思想可以一路回溯到本文的"层级裁剪 = 对数搜索"。

## Personal Knowledge State

- **user_level：机制层 Easy / 成本模型层 Normal**（沿用库内"分层模板"）：
  - **机制层（不复述）**：LOD、包围体、视锥裁剪、流式加载——你的日常工作；
  - **成本模型层（值得动手的 Normal 材料）**：① **"可见复杂度固定上限"作为分档的定义方式**——你的五档是"资源上限表"，Clark 的原始表述是"**单帧可见量的上限**"，两者可以对照一次（"我的上限在限制资源，还是在限制可见量？"）；② **"working set"视角**——显存/内存里"该驻留多少"是**每档的一个可定义量**（当前五档主要在管"画什么"，可以把"驻留什么"补成第六个维度性检查）；③ **第 2 条的"相机与物体运动"**——细节选择不只与静态屏幕占比有关，还与运动相关（本库已多次出现：运动引起的时间稳定性问题）。
- 你的五维预算体系现在可以引用 **1976 + 1978 + 1983 三篇**作为整体依据（分档原理 + 成本法则 × 2）。

## Learning Value

1. **判据（可复用）**：**"成本 ∝ 物体空间复杂度"是所有性能问题的原罪；每次优化都是在把它改写成"成本 ∝ 可见复杂度"。** 评估任何剔除/流式/分档方案，问它的成本函数现在关于什么量在增长；
2. **一条历史观**：**"预算是运行时概念"**——Clark 的 working set 让"资源上限"第一次成为**随帧变化、可讨论**的量；你今天的五档矩阵是这条线的当代形态；
3. **论文写作层面**：一篇 1976 年的"短理论论文"，五个改进点列成清单、其中四条在五十年后全部工程化——**"提出正确的问题集"比"做完所有实现"更长寿**（它当时甚至没有实现）。

## Notes

- 文献信息（ACM DL 核实）：Commun. ACM 19(10): 547–554, Oct. 1976；DOI 10.1145/360349.360354；SIGGRAPH '76 版为 1 页摘要（p.267，DOI 10.1145/563274.563323）；作者当时单位 **University of California, Santa Cruz**；后于 1982 年创办 SGI（背景信息）；
- **原文扫描件获取记录**：9-29 运行时 Wayback 全程 429（8 次尝试，多端点/多快照变体）→ **9-30 换路径结清**：OSU Pressbooks 站内镜像直接可得（`ohiostate.pressbooks.pub/app/uploads/sites/45/2017/09/clark-vis-surface.pdf`，无需经过 Wayback），8 页全文可抽取（含 CACM 547–554 全篇）；扫描件文件名 `clark-vis-surface.pdf`（与 Wayne Carlson 历史档案同源）。**方法记录：老论文"大学站内镜像"路径第 5 次生效**（Kajiya-Kay → Scheuermann → Parish → Clark 前序失败 → 本次 OSU 站内）；
- **核验清单（9-29 提出 → 9-30 完成，三项全结）**：
  - ① **"分辨率上限"论证**：没有具体位数/数字门槛，机制是 **area test**——节点覆盖超过 *critical maximum area* 才下探；原文教具式数字：**500 个多边形 vs 屏幕 20 个光栅单位**（"makes no sense"）、人体例 **3–4 / 16 个光栅单位**分档；
  - ② **层级结构定义**：根树 + 两类弧（变换 / 指向更精细结构的恒等指针）+ 非终端节点 "sufficient" 定义 + 终端 = 多边形/曲面片；最小包围信息 = 包围球（中心 + 半径）；
  - ③ **递归下降算法形态**：逐层排序 + 遮挡测试消去子树 + 三维重叠时子代"共享父节点"下探 + 终端快速排序；计算时间随可见复杂度线性（理想条件下）；与裁剪合并的建议见"结构"节。
