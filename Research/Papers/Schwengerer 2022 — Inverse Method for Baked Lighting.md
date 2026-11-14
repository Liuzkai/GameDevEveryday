---
type: paper
title: "Inverse Method for Baked Lighting"
authors: ["Mathias Schwengerer"]
year: 2022
published: 2022（TU Wien 学士论文；出版页归档于 2023）
venue: "TU Wien 学士论文（Bachelor's Thesis），Rendering and Modeling Group（Wimmer 组）；项目页 cg.tuwien.ac.at/research/publications/2023/Schwengerer_Mathias-2022-BGL"
url: "https://www.cg.tuwien.ac.at/research/publications/2023/Schwengerer_Mathias-2022-BGL"
code: "未公开（基于组内在研的可微 GI 框架 [Pri23]，即后来的 Lipp et al. 2024）"
category: ["inverse-rendering", "lighting-design", "baked-lighting", "lightmap", "relighting", "game-remaster"]
importance: A-
historical_importance: 0
game_relevance: 5
production_readiness: "Prototype（端到端跑通 Quake III Arena 三张地图；离线城市规模未验证）"
user_level: Easy
status: unread
---

# Inverse Method for Baked Lighting

## TL;DR

**Lipp 2024 那篇 §6.3.4「从 lightmap 反推灯」其实有一整本学士论文做前传——而这本前传比论文本身更贴近"做工具"的视角。**

Lipp et al. 2024（[[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization]]）把「从烘焙光照反推灯参数」写成一个应用案例（OpenArena，42 s）；而 Schwengerer 这篇学士论文是**同一个组、同一个框架，把整个问题从头到尾当工程做了一遍**：解析 Quake III Arena 的 `.bsp` 二进制场景文件 → 从 lightmap 构造辐射场目标 → 撒初始灯阵 → 跑优化 → **用路径追踪把场景重新打光**。

对"旧灯光系统 → 新灯光系统迁移"的读者来说，这篇的价值不在理论（理论全是 Lipp 的），而在三处**论文不会写、只有真做过一遍才知道的东西**：

1. **初始灯阵怎么撒、撒多密**（§3.3 / §5.1）：均匀网格、低强度白光起点，并实测了网格尺寸与步长的影响；
2. **优化方案怎么排期**（§5.2，Table 5.1）：5 种「Adam / L-BFGS × 参数子集 × 阶段顺序」组合的实测对比——**先 L-BFGS 全局粗调、再 Adam 精调（Scheme 3）是速度/精度最优**；
3. **结论里的一句大实话**（§6）：**最终质量由"初始配置"和"优化方案"共同决定，不是优化器一个说了算**——初始灯阵"尽可能小、必要地大"，逐步加灯直到收益可忽略。

一句话对读者：**如果 Lipp 2024 证明了"这件事可行"，这篇就是"真做一遍会踩哪些坑"的第一手记录。**

![[Research/Files/Pictures/Schwengerer 2022 — 图3.1 烘焙光照反推灯参数优化管线.png]]
**读图**：全文唯一一张系统图。左上 Quake III 场景文件拆出两路——`Illumination`（lightmap）经 `Build Target` 变成目标辐射场 $L^*$；`Geometry`（几何）与 `Light Parameters`（初始灯参数）进 `Forward` 光追得到当前辐射场 $L$；$O(L,L^*)$ 是目标函数；`Optimization` 更新参数回到灯。**注意这张图里"旧数据"和"新表示"的分界：lightmap 只在左上角出现一次，之后整个循环里跑的都是显式灯——这正是"迁移"的最小闭环。**（图 3.1，Schwengerer 2022）

## Problem

实时渲染受硬件限制，历史上游戏普遍把光照**烘焙进 lightmap**；代价是**灯的信息丢了**——lightmap 里只有"每个面元被照成什么样"，没有"是哪盏灯照的"。当硬件光追普及、想给老游戏做现代化重打光（动态光、间接光、软阴影）时，**原始灯具数据往往已不可得**（原文特别强调 "especially when the original data is unavailable or lost"）。

问题被明确陈述为：**给定场景几何 + 烘焙光照，重建一组物理灯光参数，使重渲染结果尽量贴近原 lightmap 的观感**，从而解锁现代渲染效果。

## Historical Context

```text
1999     Quake III Arena 发售：BSP + lightmap 的工业标准形态
2008     OpenArena：开源复刻，成为学术界的测试床
~2022    TU Wien Wimmer 组在研 view-independent 可微 GI 框架（[Pri23]）
2022  ★  本文：把该框架第一次用到"真实游戏资产"上（Q3A 三张地图）
2024     框架本身发表为 Lipp et al. 2024（TOG），§6.3.4 即本课题的论文版
```

**读法上的注意**：这篇是"上游框架的应用验证"。它不提供新算法，但提供了**算法落地所需的全部周边决策**——数据解析、目标构造、初始配置、方案排期、结果验收。Lipp 2024 的 §6.3.4（OpenArena 两关）与本文（Q3A 三关）是同一批工作的两个出口。

## Previous Work

论文背景章（Ch.2）覆盖：预计算渲染（lightmap / irradiance volume）、实时渲染、逆渲染、优化算法（Adam / L-BFGS）。**方法本身完全取自 [Pri23]**（即 Lipp 2024 的预发表版），论文明确说自己的贡献是"adapts the underlying approach to our specific problem"——把框架适配到"烘焙光照反推"这个具体问题。相关的灯位优化谱系（Pellacini、Lin、Schwarz-Wonka 等）见 [[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization|Lipp 2024]] 的 Previous Work，本文不展开。

## Core Idea

### 四段式管线（§3 开头，图 3.1）

```
① Loading the scene        解析 Q3A .bsp 二进制 → 几何 + lightmap
② Building the target      lightmap → 顶点辐射场目标 L*（面积加权 + 均匀三角形采样，GPU compute shader）
③ Initial configuration    场景包围盒内均匀网格点光，低强度白色
④ Optimization process     forward 光追 → O(L, L*) → 伴随梯度 → 更新灯参数
```

### 目标构造（§3.2）：lightmap → 顶点辐射场

lightmap 是贴图域的数据，框架要的是顶点域的辐射场。论文给了完整的投影推导（式 3.6–3.7）：顶点 $i$ 的关联面积 $A_i=\frac13\sum_{k\in N(\mathbf v_i)}A_k$（邻接三角形面积的三分之一），然后

$$r_i=\sum_k \frac{1}{A_i}\frac{A_k}{n}\sum_{l=1}^{n}r(\mathbf x_l)\,\varphi_i(\mathbf x_l)$$

即**每个三角形上均匀撒 $n$ 个样本，把 lightmap 的值按基函数权重分摊回顶点**。实现上是 GPU compute shader（Ch.4，Algorithm 4.1）。

**这一步对迁移工具是通用件**：不管你旧管线给的是 lightmap、探针还是离线渲染，第一步都是"把旧数据投影成框架认识的目标"，而投影必然带采样误差——论文选择用显式积分公式把它写清楚，而不是藏起来。

### 初始灯阵（§3.3）：一个被低估的自由度

原文的关键句："the optimizer only optimizes existing light sources; **no light sources are added or removed during the optimization process**. Therefore, the initial lighting configuration has a significant impact on the final result."

- 方案：场景包围盒内**均匀网格**的点光，初始**低强度、白色**（图 3.3）；
- 每盏灯可优化的参数：**位置、颜色、强度**；
- 论文明确承认均匀网格的副作用：灯会落在房间外、墙后——"但这些灯不会产生视觉影响，只浪费渲染时间，所以忽略之"。**这是一个工程上务实但值得注意的简化**：对封闭性差的场景未必成立。

### 优化方案 = 初始配置 × 阶段序列（§5 的定义）

"An optimization scheme is defined by an initial lighting configuration and by one or more optimization phases. Each phase consists of an optimization algorithm and its parameters." ——**把"方案"显式建模成一个数据结构**，这个抽象本身就是做工具时该抄的。

实验设置（§5 开头）：三张 Q3A 地图（q3dm1 Arena Gate / q3dm2 House of Pain / q3dm3 Arena of Death）；Adam 上限 **400 迭代**，L-BFGS 理论上限 10000（实际总是提前终止）；**每盏灯约 100 000 条光线**；每个实验跑 3 次取最好。

## Technical Approach

### §5.1 基础方案：步长与网格尺寸的扫描

在 11×11 灯阵上扫 Adam 步长（图 5.1/5.2，三张图）：**步长越小结果越好，但 0.02 在 400 迭代内没收敛**——小步长收益与迭代预算挂钩。

### §5.2 复杂方案：五种排期的对比（Table 5.1，图 5.9）

| 方案 | 阶段 1 | 阶段 2 | 阶段 3 |
|---|---|---|---|
| Scheme 1 | Adam 0.1，强度+颜色 | Adam 0.1，位置 | Adam 0.1，全部 |
| Scheme 2 | Adam 0.1，位置 | Adam 0.1，强度+颜色 | Adam 0.1，全部 |
| Scheme 3 | **L-BFGS 1.0，全部** | **Adam 0.1，全部** | — |
| Scheme 4 | Adam 0.1，全部 | Adam 0.1，全部 | Adam 0.1，全部 |
| Scheme 5 | L-BFGS 1.0，全部 | L-BFGS 1.0，全部 | L-BFGS 1.0，全部 |

结果（原文，图 5.9/5.10/5.11）：

- **Scheme 4 目标函数最低**（纯 Adam 反复跑）——但"Adam 从不自行终止，迭代上限直接决定精度与耗时"，且终止慢；
- **Scheme 3 速度/精度最优**（先 L-BFGS 后 Adam）；
- Scheme 5（纯 L-BFGS）在图 5.9 里目标函数明显更高（约 1.8–3.0 区间，各 run 分散）；
- **分参数子集优化（Scheme 1/2）反而最差**——先调颜色再调位置这类"分而治之"在这里不如全参数一起调。

![[Research/Files/Pictures/Schwengerer 2022 — 图5.9 五种优化方案目标函数对比.png]]
**读图**：左图是五种方案三次运行的"终点目标函数 vs 迭代数"散点——Scheme 4（青）与 Scheme 3（黄）聚在最低的 1.0–1.4 区，Scheme 5（浅绿）散在 1.8–3.0；右图是收敛曲线——注意 Scheme 2（红）有长长的平台期：Adam 在小步长下会在一个水准上停很久才再降。**这张图直接回答了"该选什么优化器组合"：不是理论问题，是实测问题。**（图 5.9，Schwengerer 2022）

### 最终验收（§5.2 末 + 图 5.14）

拟合出的灯组被用来做**路径追踪重打光**（图 5.14）："the overall lighting ambience of our results is very close to the original lighting stored in the lightmaps"——并且换上了全局光照、软阴影等原游戏没有的效果。

![[Research/Files/Pictures/Schwengerer 2022 — 图5.14 拟合灯光路径追踪重打光结果.png]]
**读图**：三张地图（q3dm1/2/3）各两个视角，全部用拟合出的灯组 + 路径追踪渲染。注意画面里出现了**原 lightmap 不可能有的内容**：软阴影的半影、墙面的间接光渐变。**这张图是"迁移为什么值得做"的答案：目标不是复刻 lightmap，而是拿到一组可以在新管线里继续生长的灯。**（图 5.14，Schwengerer 2022）

## Key Contribution

1. **把"从烘焙光照反推灯"做成了端到端管线**，输入是真实的商业游戏资产（Q3A `.bsp`），不是合成场景；
2. **lightmap → 顶点辐射场的显式投影公式**（式 3.6/3.7），可 GPU 化；
3. **"优化方案 = 初始配置 × 阶段序列"的显式建模**，并对 5 种排期做了受控对比（Table 5.1）；
4. **实测结论：先 L-BFGS 粗调、再 Adam 精调（Scheme 3）速度/精度最优**；分参数子集优化反而最差；
5. **明确记录初始灯阵是主要自由度**（数量、密度、初值都对结果有显著影响），并给出"逐步加灯直到收益可忽略"的选灯策略雏形。

## Why It Works

**因为问题被正确地"降维"了两次。**

第一次：目标从图像降到辐射场（Lipp 框架的贡献）——lightmap 本来就是视角无关的，不该为它发明相机。

第二次（这篇论文自己的）：**把"重建灯"变成"给现成的网格灯阵调参数"**。均匀网格初始化的聪明之处在于把离散问题（该有几盏灯、放在哪）近似连续化——灯多得足够密时，"哪盏灯负责哪块区域"可以由位置和强度的连续优化自己涌现出来。代价是浪费了灯（房间外、墙后的灯照样参与渲染），换来的是**完全绕开了离散搜索**。

§5.2 的结果还揭示了一个不那么显然的事实：**灯光参数之间的耦合强到"分阶段调子集"是负优化**。颜色/强度和位置不是独立的——一盏灯挪远了就得调亮，这正是 Lipp 2024 里 L-BFGS 靠 Hessian 近似捕捉"高度↔强度"耦合而胜出的同一现象。先 L-BFGS 后 Adam 之所以最优，可以理解为：L-BFGS 的二阶信息先把耦合结构理顺，Adam 再在良性地形上细调。

## Limitations

- **灯的数量与初始布局全由人定**；论文的答案是"网格尽可能小、必要地大，逐步加密看收益"（§6）——是策略，不是算法。
- **均匀网格在非凸/多层场景会浪费大量灯**（房间外、墙后），论文选择忽略；对复杂场景这可能不只是浪费，还会引入错误的光。
- **验收标准偏定性**："visually appealing"、"overall ambience very close"——没有系统的感知评估，只有目标函数曲线 + 渲染图对比（这正是 [[Giroux 2023 — Towards a Perceptual Evaluation Framework for Lighting Estimation|Giroux 2023]] 补的那一格）。
- **只用点光**：没有面光、IES、朝向——迁移到现代灯光系统时参数空间要大得多。
- **每张地图的网格尺寸是手调的**（11×11 做扫描，map 1 用 9×9）——没有跨场景的一致规则。
- **文本提取注意**：该 PDF 无 ToUnicode 映射，文本层乱码，本笔记内容全部来自逐页图像识读；个别未亲见页面的细节标「未核」。

## Game Development Relevance

**这是"老游戏翻新"（remaster）场景下最直接的一篇，也是本专题与读者需求重合度最高的一篇。**

| 论文里的角色 | 迁移工具里的对应物 |
|---|---|
| Q3A `.bsp` + lightmap | 旧灯光系统导出的场景 + 烘焙数据 |
| lightmap → 顶点辐射场投影（式 3.7） | 旧数据 → 拟合目标的格式翻译层 |
| 包围盒均匀网格点光 | 新系统初始灯阵（Lipp 用 9×9×2，本文用 9×9 / 11×11） |
| 位置+颜色+强度，Adam/L-BFGS | 新系统可调参数子集与优化器排期 |
| 图 5.14 路径追踪重打光 | 迁移验收：在新管线下重渲对比 |

三条可直接抄的工程决策：

1. **方案排期用 Scheme 3 起步**（L-BFGS 1.0 全局 → Adam 0.1 精调），别分子集调；
2. **迭代预算和步长一起定**——0.02 步长配 400 迭代就是浪费，步长收益曲线（图 5.2）形状可预期；
3. **把"方案"做成可配置的数据结构**（初始配置 + 阶段列表），而不是写死在代码里——这样换场景时只换配置。

## Unreal Engine Relevance

- **UE 侧数据对应**：lightmap → `Lightmap`/`Volumetric Lightmap`；顶点辐射场目标可用 Lightmass 的 indirect SH 或直接采 lightmap texel 投影；
- **管线形态**：本文是"外部进程读场景文件 → 优化 → 回写灯参数"的离线工具链，与 UE 编辑器外挂 Python/C++ 工具的形态一致；
- **警示**：本文只有点光。UE 里 Spot/Rect/Directional 的参数化（锥角、IES、SourceRadius）需要另做梯度——可参照 Lipp 2024 的 Rodrigues 朝向与 IES 处理；
- **一个现实差距**：本文场景是 1999 年的低密度几何。现代场景的顶点数与遮挡复杂度都会放大"均匀网格 + 全参数"的成本，选灯策略（⑥号槽位）仍然是缺口。

## Technology Evolution

```text
"烘焙光照的逆问题"
1999-2010s  lightmap 是终点：烘焙完即出货，灯的信息随工程文件消亡
2016        ILM LightCraft：从片场 HDRI 解算灯阵（影视侧，半自动，需双高度拍摄）
2022     ★  本文：从 lightmap 全自动反推灯（游戏侧，Q3A 实证）
2024        Lipp et al.：框架正式发表，OpenArena 案例即本工作的论文版
2025        Sony Imageworks（Shlyaev）：HDRI → 灯阵 + 单目深度定距，"几小时 → 约一分钟"
            ↓
        趋势：目标数据从"需要专门拍摄"→"场景里现成的烘焙产物"；
        自动化从"半自动标记"→"优化求解"
```

## Relationships

### Based On

- **[[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization|Lipp et al. 2024]]**（文中引用为 [Pri23]，预发表版）——全部算法基础：view-independent 辐射场目标、解析伴随光照追踪、Adam/L-BFGS。本文是它的**第一个完整应用**。

### Related

- **[[LightOpt — Lights Optimization for Real-Time Rendering]]** —— 实时侧对应物；
- **[[Giroux 2023 — Towards a Perceptual Evaluation Framework for Lighting Estimation]]** —— 本文缺的"怎么判断像不像"那一格，Giroux 给出了系统的负面答案（标准 IQA 指标大多不预测人类判断）；
- **[[Global Illumination]]** —— lightmap 属预计算光传输一支；
- **[[Mesh Parameterization]]** —— lightmap 贴图域 ↔ 顶点域的投影误差与 atlas 接缝问题相通。

### Followed By

- Lipp et al. 2024 §6.3.4 的 OpenArena 案例（同组直接后继）；
- 待观察：离散选灯（数量/类型）与连续拟合的混合整数方案，两篇文章都把它列为未来工作。

## Personal Knowledge State

`Easy`。**这是一篇"工程记录"型论文，没有需要硬啃的数学**——式 3.6/3.7 是标准有限元面积加权，Adam/L-BFGS 是常识。真正的信息量在三处实测：步长扫描、五种方案对比、初始网格的影响。**读它的正确姿势是当"实验报告"读，不是当"方法论文"读。**

## Learning Value

- **对工具设计**：把"优化方案"建模成"初始配置 × 阶段序列"的数据结构，是值得抄的抽象；
- **对实验方法**：步长 × 迭代预算必须一起报——单独报一个会误导（0.02 步长的"差"是预算不足造成的）；
- **对直觉矫正**：分而治之（先颜色后位置）在强耦合参数上是负优化——**耦合参数就该一起调，调不动先怀疑优化器不怀疑分解**；
- **一句话**：初始条件不是细节，是一半的答案（"the quality of a reconstructed lighting setup is not determined solely by the optimization scheme applied but also by the initial configuration"）。

## Visualization

![[Research/Files/Pictures/Schwengerer 2022 — 图3.1 烘焙光照反推灯参数优化管线.png]]
（图 3.1：四段式管线——lightmap 建目标、几何+灯参数 forward、目标函数、优化回写）

![[Research/Files/Pictures/Schwengerer 2022 — 图5.9 五种优化方案目标函数对比.png]]
（图 5.9：五种排期的终点分布与收敛曲线）

![[Research/Files/Pictures/Schwengerer 2022 — 图5.14 拟合灯光路径追踪重打光结果.png]]
（图 5.14：拟合灯组在新管线下的路径追踪重打光）

## Notes

- **版本与归属**：TU Wien 学士论文，2022 年完成，项目页归档于 2023 年（URL 含 `2023/`）。作者 Mathias Schwengerer，Wimmer 组。正文引用的框架记为 [Pri23]，即后来的 Lipp et al. 2024（TOG, DOI 10.1145/3662180）。
- **阅读方式声明**：该 PDF 内嵌字体无 ToUnicode 映射，文本提取为乱码；本笔记全部内容通过逐页渲染 + 图像识读获得。重点核读页：Ch.3（方法，p27–31）、Ch.5（评估，p41–51）、Ch.6（结论，p53）。Ch.4（BSP 实现细节）只浏览缩略图，**其中关于 `.bsp` 解析的具体描述未逐字核**。
- **数字清单（均已亲见原文）**：Adam 迭代上限 400；L-BFGS 理论上限 10000；每灯约 100 000 条光线；每实验 3 次运行；基础扫描用 11×11 灯阵；map 1 复杂方案用 9×9；Adam 步长 0.1，L-BFGS 步长 1.0；三张地图 q3dm1/q3dm2/q3dm3。图 5.9 中各方案目标函数读数（约 1.0–3.0）为图像判读，**精确值未核**。
