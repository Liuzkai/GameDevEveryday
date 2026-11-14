---
type: paper
title: "Example-Based Model Synthesis"
authors:
  - Paul Merrell
year: 2007
published: 2007-04-30
venue: "I3D '07: Proceedings of the 2007 Symposium on Interactive 3D Graphics and Games, pp. 105–112（ACM）"
url: "https://doi.org/10.1145/1230100.1230119"
code: "作者主页提供 C++ 实现与示例集（paulmerrell.org）"
project_page: "https://paulmerrell.org/?p=143"
category:
  - procedural-generation
  - pcg
  - modeling
  - constraint-solving
  - wave-function-collapse
importance: A
historical_importance: 4
game_relevance: 5
production_readiness: Industry Adopted（算法本身；WFC 是其直系后代）
user_level: Normal
status: unread
tags:
  - procedural-generation
  - model-synthesis
  - wave-function-collapse
  - constraint-propagation
  - np-complete
---

# Example-Based Model Synthesis（Merrell, I3D 2007）

## TL;DR

**给一个小例子模型，自动生成一个大的、看起来像它、且"处处合法"的模型。** 三件套：从例子里数出**邻接表**（六向）、用**全局传播**把不可能的选择提前剪掉、把大问题切成**小块 $B$** 反复重填（填不动就整体放弃这块）。

它在 2007 年就写下了 [[Wave Function Collapse]] 的全部机制（**原作者本人页面：WFC"重新实现"了它，两者"几乎一样"**），也写下了它的天花板：附录证明**"能否补完"是 NP-完全**（Planar 3-SAT 归约），所以那个"全局搜索"注定只能剪掉**容易剪的**那部分。

> **一句话定位**：库内 PCG 线缺失的**第八个源头**——形状文法（1975/2001/2003/2006）管"规则怎么写"，本文管"**约束怎么解**"。这也解释了为什么 2016 年之后独立游戏生态的 PCG 主流不是 CGA 而是 WFC。

## Problem

手工建大场景做不完（2006 CGA 的原话是"15 人年"）。程序化建模的既有路线要求**人写规则**，而规则只能覆盖一小类对象——换一类对象就得重写。

本文反过来问：**能不能不写规则，只给一个例子？** 例子是设计师已经会做的东西；如果算法能从例子里"读"出规则，那规则编写这一步就消失了。

代价是三条硬要求（论文开头就摆出来）：
1. 例子必须能**拆成少量可复用的 piece**，摆在格点上；
2. piece 必须**手工构造**——**分解这一步不是算法做的**；
3. 生成结果是**格点上的离散摆法**，不是连续几何。

## Historical Context

- **1976 Wang tiles**：局部可拼贴。难的不是拼，是**造一个能拼的瓦片集**；Wang cube（3D 版）已被用来铺小行星带与体数据；
- **1999–2001 纹理合成**（Efros-Leung 逐像素 / Image Quilting 逐块）：局部决定局部——本文自称"更像这一类"；
- **2001/2003/2006 形状文法**（[[Parish-Müller — Procedural Modeling of Cities (2001)]] / [[Wonka — Instant Architecture (2003)]] / [[Müller — Procedural Modeling of Buildings (2006)]]）：**人写规则**，把设计意图编码成文法。同一时期、完全不同的问题设定；
- **★ 2007 本文**：不写规则，从例子里**数**出约束，然后**解**它；
- **2009** 作者的博士论文；**2010** TVCG 期刊版 *Model Synthesis: A General Procedural Modeling Algorithm*；**2008** *Continuous Model Synthesis*（脱离格点）；
- **2016 Gumin WFC**：重实现，单头文件，爆红。

**这条线的分岔值得记**：形状文法线（2001–2006）把"规则"当资产，PCG 是"解释规则"；本文把"例子"当资产，PCG 是"解约束"。**2016 年之后游戏业选了后者**——因为设计师更愿意给一个例子，而不是写一套文法。

![[Research/Files/Pictures/Merrell 2007 — 图1 例模型到合成模型.png]]
**读图**：左（a）是一个很小的例子模型——几座科林斯柱式的建筑，摆在格点上（注意水面上的格线，那是 piece 的边界）。右（b）是从它合成的城市：几十座建筑、高低错落、**没有一座是例子里原样出现的**。这就是本文承诺的东西：**"database amplification"**——从一个例子里长出量。出处：Merrell 2007, Figure 1。

## Previous Work

论文把前作分成三堆，并说明自己站在哪：**全局法**（Heeger-Bergen 1995 / Portilla-Simoncelli 2000，匹配大尺度统计）；**局部生长法**（Efros-Leung 1999 / Wei-Levoy 2000 / Image Quilting 2001）；**Wang tiles / Wang cubes**（Cohen 2003 / Sibley 2004）。

它的自我定位是"更像局部生长法"，但**加了一个当时没人做的事**：**在加料之前先做全局搜索找冲突**。原文的理由说得很直接——

> "A local search may miss some conflicts that a global search would detect, since many model pieces have an influence far outside their local neighborhood."

**这句话就是全文的题眼**：有些 piece 的影响范围远超它的邻域。十九年后，[[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share|IW 2026]] 把这句话变成了定理（沿行携带的量、$1/r$ 地板）。**2007 年的工程直觉，2026 年才被证明是结构性必然。**

## Core Idea

### ① 一致性 = 一张由例子数出来的邻接表

模型 $M$ 由"格点 + piece 标号"组成。**$M$ 与例子 $E$ 一致**，当且仅当 $M$ 里任意两个相邻 piece，在 $E$ 里也以**同样的方向**相邻过。检查六个方向（±x, ±y, ±z）。

于是整个例子被压成一张布尔转移表：

$$T_x(k_1,k_2)=\begin{cases}1,&\exists v\ \text{s.t.}\ (v,k_1),(v+(1,0,0),k_2)\in E\\0,&\text{otherwise}\end{cases}$$

$T_y,T_z$ 同理；$T((v_1,k_1),(v_2,k_2))$ 按 $v_1-v_2$ 落在哪个方向上取对应的 $T$ 或它的反向。

**这一步是全部魔法所在**：例子的"风格"不在任何显式规则里，它被**完全编码**在"哪些标号能在哪个方向上相邻"这张表里。表的大小是 $N^2\times 6$，与例子尺寸无关。

### ② 全局传播：算出"还可能是什么"

$C(M)$ = 当前模型下**每个空格仍可能取的标号集合**。算法是一个迭代到不动点的传播（Table 1）：

```
C₀(M) = 所有 (v,k)，除去与已定标号冲突的
u₀ = 已定标号的格点
while uₜ ≠ ∅:
    取一个 v ∈ uₜ
    C_{t+1} = Cₜ − {(v′,k) | 不存在 (v,i)∈Cₜ 使 T((v,i),(v′,k))=1}
    把发生变化的格点加入 u_{t+1}
```

**这就是 WFC 的"约束传播 / 弧相容"那一步**（WFC 的 `propagate` 逐字对应）。区别只在实现：本文用显式工作集，WFC 用栈。

### ③ 切块 + 局部重填：把 NP-难问题做成可跑

$C(M)$ 不够用，原因是附录证明的：**"能否补完一张不完整纹理"是 NP-完全的**（从 Planar 3-SAT 归约；用 piece 搭出 NOT 门与 OR 门，TRUE/FALSE 两种纹样不能相接）。于是：

- 选一块 $B$（连续的一小片格点）；
- 把 $B$ 里的标号全拿掉，得 $M'$；
- **只用 $C(M')$ 里的标号**重填 $B$；
- 若填完 $C(M')=\varnothing$，**整块回滚**；否则接受；
- 换一块新的 $B$（**相邻两次的 $B$ 要重叠**），重复；每个格点应被改到若干次。

**这块"整块回滚"很重要**：它是一种**局部回溯**——失败只损失一块，不是整张画布。代价是要选 $B$ 的大小、要保证重叠，这是两个额外的参数（WFC 没有这两个参数，代价是矛盾时只能整张重开）。

### ④ 让它"像"：按例子里的出现次数加权

一致性只保证"合法"，不保证"像"。所以第四件东西是一个**相似度选择概率**：

- 感知距离 $d(P,Q)=|P|-|P\cap Q|$（"不一样的部分有多少"）；
- $L(k)$ = 例子里有多少个位置，其 $\omega$ 邻域与新模型（加上 $(v,k)$ 之后）的距离不超过阈值 $d_0$；
- 选标号的概率 $P(k)=L(k)\big/\sum_i L(i)$。

**这是一个"数出现次数"的统计**——一个局部配置在例子里出现得越多，就越可能被选中。它和 WFC 的"按窗口频率加权"是同一件事的两种算法（一个数邻域匹配次数，一个数精确窗口频率）。

## Technical Approach

三个变体，都只改一个函数：

| 变体 | 改什么 | 怎么改 |
|---|---|---|
| **对称模型** | 加一个对称函数 $S(c_1,c_2)$（"这两个位置对称时，这两个 piece 允许同时出现吗"） | 把转移表换成 $T'=S\cdot T$——一致性与对称性**同时**满足 |
| **带约束模型** | 用 Metropolis 接受 | 第 5 步的接受概率从 1 改成 $\min(1, f(M')/f(M))$，$f$ 是"多符合约束"（例：把城市长成某个符号的形状） |
| **高维（4D）** | 加一个时间维 | 与 3D 完全同构；**"时间上的一致性和空间上的一样重要"**——只有某些 piece 允许在时间上前后相邻 |

这个"高维"变体值得单独看一眼：它意味着 **model synthesis 也是"时变模型生成器"**（论文的配套视频是"有车在动的城市"）。所以 2026 年那些"4D 程序化世界"的工作，在机制上并不新——新的是规模与表示。

![[Research/Files/Pictures/Merrell 2007 — 图2 四块拼柱的一致与不一致.png]]
**读图**：全篇最能说明"一致性是什么"的图。(a) 一个由**四块** piece 组成的例子（0 空 / 1 柱础 / 2 柱身 / 3 柱头），格线画出来了——**piece 是手工拆的，这是本文最大的前提**。(b) 随便拼：柱头浮在空中、柱础悬空、柱身断成两截——**每一块都是合法的 piece，整体却是荒谬的**。(c) 用邻接表约束后：每根柱子都完整、都落地。**"局部合法"和"整体合理"之间隔着的就是那张转移表**。出处：Merrell 2007, Figure 2。

## Key Contribution

1. **把"例子"变成可解的约束系统**：六向邻接表 + 全局传播，例子 → 约束的转换是自动的、无损的（就"哪些相邻"这个层面而言）；
2. **"加料前先查冲突"**（global search before adding）——相对纹理合成的关键差异；
3. **切块 + 整块回滚**：把 NP-难问题变成"多数时候能跑通"的随机修复循环；
4. **NP-完全证明**（附录）：明确宣告这条路不可能有"总是成功"的版本。**这是全文最诚实、也最容易被忽略的一段**；
5. **加权相似度**：合法之外还有"像"，且"像"被定义成一个可数的量（例子里出现多少次）。

## Results

论文没有指标表——只有图像与一句计时口径：

| 结果类别 | 内容 |
|---|---|
| 2D 纹理对比 | 与 Kwatra 2005（graphcut textures）比一张"单一连续路径、无死端"的纹理：本文生成的纹理**仍然只有连续路径**；对照方法出现了断头。**这是全文唯一的定量对比（而且没有数）** |
| 3D 结果 | 城市（不同建筑风格）、城堡、地形、树木、建筑内外部（剖开显示）、带街灯与车灯的夜景城市 |
| 对称 | 平移 / 镜像 / 旋转三种，都从同一个例子出 |
| 约束 | 把城市长成阴阳、轮椅等符号形状 |
| 4D | 带行驶车辆的城市（配套视频） |
| **时间** | **"几分钟到半小时"**：夜景城市与地形几秒（$B$ 可以大到整个模型）；图 1/6/7/10/11 的模型 30 秒到 30 分钟 |

**"$B$ 越大越快"这件事反直觉但重要**：$B$ 大到整个模型时，问题退化成"一次全局求解"，反而常常直接成功（因为约束少、自由度大）；$B$ 小的时候要迭代很多次。**这解释了为什么后来的 WFC 干脆不要 $B$——直接把整张画布当一次求解。**

## Why It Works

**① 转移表把"风格"压成了一个有限的布尔关系。** 例子的尺寸不进算法，只有"哪些相邻对出现过"进算法。所以一个 3×3×3 的小例子能长出无限大的城市——**这就是 database amplification 的机制**。

**② 传播剪掉的是"必然失败"，不是"可能失败"。** 迭代到不动点时，$C(M)$ 里的每个标号都还有**至少一个**邻居能配合——但"每个局部都有救"不等于"全局有解"。这个差别就是 NP-完全性的来源，也是"为什么必须有回滚"的原因。

**③ 整块回滚是局部化的失败代价。** 因为 $B$ 是连续的一小片，"填不动"这个事件被限制在 $B$ 里，模型其余部分不受影响。**这与 WFC 的"整张画布重开"形成对照：更省，但要两个额外参数（$B$ 的大小、重叠量）。**

**④ 加权相似度让"合法"和"像"分开控制。** 论文很清楚地说了：一致性管合法性，$P(k)$ 管像不像。**两件事不混在一个目标函数里**——这一点比很多后来的工作干净。

## Limitations

1. **例子必须手工拆成 piece。** 原文自述这是"一个重要限制"，而且"有些模型很难这样拆"。**分解是人的工作，不是算法的工作**——建筑类（重复元素 + 格点结构）适合，地形/树需要很小心地构造例子。
2. **🔴 NP-完全 ⇒ 全局搜索注定不完备。** 原文："This global search only eliminates labeled vertices that are relatively easy to eliminate." 所以"总有一个解"这件事**不可能被保证**，只能靠回滚重试。这条不是实现问题，是问题的性质。
3. **🔴 过紧的例子会退化成复读机。** 原文："it is possible that the output will be so tightly constrained that model synthesis will simply reproduce the example model over and over." **十九年后 [[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share|IW 2026]] 证明了这句话什么时候必然发生**：例子带一个沿行携带的量时，"只读位置"这条路（唯一能逃出地板的免费路线）给出的就是"例子按绝对坐标平铺"——**一个世界**。所以"复读机"不是例子做得不好，是**这类例子只允许一个世界**。
4. **只有格点。** 非格点的形状要等 2008 的 *Continuous Model Synthesis*。
5. **没有质量指标，没有并行/显存讨论，计时只有一句话。** 2007 年的 I3D 论文普遍如此，但今天回看，**"30 秒到 30 分钟"这个区间跨度是 60×**，说明算法对参数极其敏感而论文没量化。
6. **2D 对比只对了一个方法、一张纹理。** 证据很薄，不能当作"比纹理合成好"的结论。

## Game Development Relevance

- **它是 2016 之后独立游戏 PCG 主流的**真正**源头**。库内此前只有形状文法线（2001/2003/2006），**约束求解线是空的**——今天补上，PCG 的两条腿才齐：**文法线管"规则怎么写"，本文管"约束怎么解"**。
- **"piece 手工拆"这个前提，今天还在**。UE 的 tile / modular kit、Houdini 的模块化资产、Unity 的 tilemap 规则瓦片——**都要求人先把资产拆成可拼接的单元**。本文是这条工作流最早的形式化。
- **对关卡设计的一条判据**：你的 modular kit 的"邻接表"是**显式**的吗？如果是隐式的（"美术知道什么能接什么"），那你就没有 $T$，也就没有可自动化的生成；把它写成表，生成器才有输入。
- **"过紧 ⇒ 复读机"是可诊断的**：如果生成器总是在复读例子，先查例子有没有"沿某条线必须一致"的量（砖缝、栅栏间距、路的接续），而不是加大随机性。
- **对预算的间接意义**：$B$ 的大小是**生成时间 vs 成功率**的旋钮（$B$ 大 → 一次成功率高但每次贵；$B$ 小 → 便宜但迭代多）。这与运行时预算是两本账，但**"块大小是一个跨越多层含义的参数"**这件事，和 World Partition 的 cell 尺寸、PCG 的 tile 尺寸是同一个形状。

## Unreal Engine Relevance

- **UE 的 `PCG` 与 `Tile Map` 都能表达本文的模型**：piece = 静态网格 / tile 资产，邻接表 = 规则集或排除规则。**但 UE 默认不提供"全局传播"**——PCG Graph 是按点独立求值的，所以它天然是 [[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share|IW 2026]] 分类里的"独立随机数生成器"，**出缝是结构性的**。
- 想复现本文的行为，需要"传播 + 回滚"这一层——这在 UE 里要么自己写（一个 `UPCGElement` 里跑 WFC 循环），要么用 Houdini Engine 的 solver。
- **`Level Instance` + `Data Layer`** 是本文"$B$ 块"的一个现成对应物：一块一块地生成、失败就整块不加载。

## Technology Evolution

```text
1976  Wang tiles：局部可拼贴，但瓦片集难造
        ↓
1999-2001 纹理合成：局部决定局部（逐像素 / 逐块）
        ↓
【2007 本文】邻接表 + 全局传播 + 切块回滚 + 加权相似度
        + 附录：能否补完 = NP-完全  ⇒  "总成功"不可能
        + 自述失败模式：过紧的例子会复读
        ↓
2008  Continuous Model Synthesis：脱离格点
2009  博士论文 / 2010 TVCG 期刊版：完整化
        ↓
2016  Gumin WFC：同一算法，单头文件、overlapping/tiling 双模型 → 爆红
        （作者本人页面："WFC reimplements the model synthesis algorithm"，"nearly identical"）
        ↓
2021-23 块求解器：为无限世界铺固定周期背景
        ↓
2026  ★ IW 2026：证明"保留背景"是数学必然，并给出"何时必然复读"的判据
        —— 2007 的两句"自述限制"，十九年后各自变成一条定理
```

## Relationships

### Based On

- 纹理合成（Efros-Leung 1999 / Wei-Levoy 2000 / Image Quilting 2001）——"局部决定局部"的范式；
- Wang tiles / Wang cubes（Cohen 2003 / Culik & Kari 1996）——"局部可拼贴"的数学原型；
- Planar 3-SAT（NP-完全性证明的归约源）。

### Extends

- 纹理合成：从 2D 图像扩到 3D 模型，并**加了全局传播**（纹理合成只看局部邻域）。

### Related

- [[Parish-Müller — Procedural Modeling of Cities (2001)]] / [[Wonka — Instant Architecture (2003)]] / [[Müller — Procedural Modeling of Buildings (2006)]]——**同期、同目标、相反的方法论**：那三篇写规则，本篇数约束。**两条线不冲突，覆盖的需求不同**（前者要"设计师能改的结构"，后者要"设计师给不出规则的风格"）；
- [[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share|IW 2026]]——直接后代 + 理论收口；
- [[Procedural Content Generation]]——本篇是该概念页"约束求解线"的第一个节点（此前只有文法线与噪声线）。

### Followed By

- 2008 *Continuous Model Synthesis*（同作者，脱格点）；
- 2010 TVCG 期刊版；
- **2016 Gumin WFC**（非论文，单头文件实现）——**它才是让这条线进入游戏业的那一步**；
- 2019–2023 块求解器（Boris the Brave 2021 / Kleineberg 2023）：为无限世界做随机访问。

## Personal Knowledge State

Current Level: **Normal**（结论层 ≈25 分钟；NP-完全证明可跳过）

三层，按可用性：

1. **"局部合法 ≠ 整体合理"，而它们之间隔着的就是一张转移表。** 这是本文最可迁移的一句话——任何"局部规则拼装"的系统（modular kit、tile 规则、材质分层、甚至动画片段拼接）都可以问：**我的转移表在哪？是显式的还是人脑里的？**
2. **"总成功"在数学上不可能（NP-完全），所以工程上唯一能做的是"失败的代价要局部化"。** 本文的答案是把失败限制在 $B$ 块里；WFC 的答案是整张重开。**这是一个"失败粒度"的设计选择，不是一个算法优劣问题。**
3. **"过紧 ⇒ 复读"是这类算法的固有失效模式**，不是调参能解决的。判据见 [[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share|IW 2026]]：找"沿某条线必须一致"的量。

## Learning Value

- **对 PCG 认知的一次补全**：库里此前 PCG 的历史是"文法线"（2001→2003→2006→2026 混合线），今天补上"约束线"（2007→2016→2021→2026）。**两条线的分岔点就在 2007**：一个说"把规则写成资产"，一个说"把例子当成约束"；
- **对"要不要上 WFC"的判断**：本文给了 WFC 的三个先天限制（piece 要拆、NP-难、过紧会复读），比任何教程都清楚；
- **对"NP-难怎么办"的工程观**：不是"找近似算法"，是"**让失败便宜**"。这个思路和渲染里的"近似 + 后处理修正"是同构的。

## Visualization

两张原图已嵌（图 1 例子→城市 / 图 2 四块拼柱）。**没有做 HTML**：核心机制是"一张六向邻接表 + 一个传播循环"，Markdown 的伪代码块已经是最清楚的表达；图 2 把人最容易误解的地方（局部合法 vs 整体合理）画完了。按 §25 三问，三条都不满足。

## Notes

- 原文获取：ACM DL 403，**作者个人主页 PDF 直链 200**（`paulmerrell.org/wp-content/uploads/2022/03/model_synthesis.pdf`，3.96 MB）——**再次验证"经典线优先找作者主页"，不要在 ACM 上耗**（这是本库第 4 次同一结论）；
- **归属证据**：作者主页 `?p=143` 明写 "WFC is Based on Model Synthesis … WFC reimplements the model synthesis algorithm … The two algorithms are nearly identical"，并附一份详细对比。这是本文库能拿到的**最直接的一手归属说明**（作者本人、非第三方转述）；
- 页码：pp. 105–112，I3D 2007（Seattle，2007-04-30 ~ 05-02）；
- 与 2006 CGA 的关系：本文引用了 Müller 2006，并把它归为"需要写规则、只能覆盖小类对象"的那一堆——**同期同会，两种哲学**；
- 遗留：`pypdf` 对本文无警告（8 页，文本干净）。
