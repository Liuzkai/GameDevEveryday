---
type: paper
title: "Procedural Modeling of Buildings"
authors: [Pascal Müller, Peter Wonka, Simon Haegler, Andreas Ulmer, Luc Van Gool]
year: 2006
published: 2006-07-30
venue: "ACM Transactions on Graphics 25(3): 614–623（SIGGRAPH 2006）"
url: "https://doi.org/10.1145/1141911.1141931"
code: ""
project_page: ""
category: [procedural-generation, architecture, shape-grammar, production-pipeline]
importance: S
historical_importance: 5
game_relevance: 4
production_readiness: Industry Adopted
user_level: Normal
status: unread
---

# Procedural Modeling of Buildings（CGA shape）

## TL;DR

**程序化建筑的奠基论文：CGA shape 形状文法。** 它把"形状文法"从建筑学的人工推导工具，改造成**计算机可执行的产生式系统**，第一次解决了**复杂体块（volumetric mass model）上生成一致立面与屋顶细节**的问题——此前的方法要么只有简单体块（Parish & Müller 2001），要么只能处理简单拆分（Wonka 2003）。它直接演化为 **CityEngine**（后为 Esri），成为游戏/影视/城市规划行业的标准程序化建模工具。

> **对库的意义**：**程序化生成（PCG）域的第一个历史锚点**，与同日入库的 [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]] 构成"源头 ↔ 前沿"闭环。**且原文 Discussion 里 2006 年就写明了 2026 年正在被解决的三个问题。**

## Problem

**大型三维环境（城市）的建模成本**。原文开篇即写：建模这样的大型环境 *"是一个非常昂贵的过程，可能需要多人年的人工"*（very expensive process and can require several man years worth of labor）。

**原文给出的行业对照数据**：电影《超人归来》（Superman Returns）的城市模型制作花费 **15 人年**。

已有工作各自能解决一半，但**无法合拢**：

- **Parish & Müller 2001**（Procedural Modeling of Cities）：能生成大型城市，但每栋建筑只是**简单体块 + shader 假装细节** → 几何细节不足；**且体块互不感知** → 窗/门会被相邻体块以不自然的方式切开（原文 Figure 2 左图专门展示这个伪影）
- **Wonka et al. 2003**（Instant Architecture）：能生成立面几何细节（split 规则），但**只对简单体块有效**；复杂体块需要"数量过量的拆分"，且**无法处理任意朝向的物体**（如坡屋面）

> **核心矛盾**：**"体块建模"（宏观、可变、任意朝向）与"立面细节"（微观、需秩序）之间没有一致的桥。** 复杂体块的并集会产生带凹口、多顶点、多洞的**一般多边形**——而规则写不了一般多边形。

## Historical Context

形状文法一脉的谱系（本文 Related Work 原文归纳）：

```text
生产式系统（Semi-Thue 过程 / Chomsky 文法 / 图文法 / 属性文法）
                ↓
Stiny 1975 《Shape Grammars》—— 建筑学：形状文法用于**人工**构造与分析建筑
（原文：推导本质复杂，"通常人工完成，或由计算机辅助、人类决定应用哪条规则"）
                ↓
图形学界的两条支线：
  ├─ Prusinkiewicz & Lindenmayer 1991：L-system + 龟形解释 → 植物建模的辉煌成绩
  └─ Stiny 1982 / Wonka 2003：set grammar 简化（更易计算机实现）→ split 规则
                ↓
    Parish & Müller 2001（城市，L-system + 简单体块 + shader）
    Wonka 2003（立面细节，split 规则）
                ↓
★ Müller et al. 2006 —— CGA shape：把两条支线合起来（本文）
                ↓
2007 Procedural Inc. 从 ETH 分出 → 2008.7 CityEngine 首商用版 → 2011 Esri 收购
（→ ArcGIS CityEngine；规则语言至今仍叫 CGA）
                ↓
2010s 进入游戏/影视管线（+ Houdini 生态）
                ↓
2026 ProxyBuild（LLM + 检索，回应"规则编写费力"）
```

## Core Idea

**CGA shape（Computer Generated Architecture shape）= set grammar 的扩展，一个顺序文法（sequential grammar）。**

**为什么不是 L-system 式的并行文法**（原文原话，值得记住）：

> 并行文法（如 L-system）**适合刻画"随时间生长"**（growth over time）；而**顺序应用规则才能刻画"结构"** —— 即特征与组件的**空间分布**。对建筑而言，"生长"这个概念"常常适得其反"（often counterproductive）。

**基本对象**：
- **Shape** = 符号（terminal / non-terminal）+ 几何属性 + 数值属性
- **Scope（作用域）** = 位置 $P$ + 三正交轴 $X,Y,Z$ + 尺寸 $S$ → 空间中的一个**定向包围盒**。**所有规则都作用在 scope 上**（L-system 龟形记号的思想演化）。

**产生式过程**（关键工程细节）：
1. 从 **axiom**（公理，如一块建筑用地 footprint）开始
2. 选一个 active shape → 用左侧规则替换 → **标记为 inactive 而非删除** → 加入新形状
3. **规则带优先级** → 修正的广度优先推导 → **保证"从低细节到高细节"受控推进**
4. 没有非终结符时终止

> **"标记而非删除"的意义**（原文明确）：保留推导树，**使规则可以查询整个形状层级，而不只是当前活动配置**——这是后文 occlusion 查询的基础。

**规则形式**：
```
id: predecessor : cond ; successor : prob
```
例：`1: fac(h) : h > 9 ; floor(h/3) floor(h/3) floor(h/3)`

**四种核心规则**：
- **Scope 规则**：`T(tx,ty,tz)` 平移 / `Rx/Ry/Rz` 旋转 / `S(sx,sy,sz)` 定尺寸 / `[` `]` 压弹栈 / `I(objId)` 实例化几何体
- **Split 规则**：`Subdiv("Y",3.5,0.3,3,3,3){ floor | ledge | ... }`（单轴/多轴/嵌套）
- **Repeat 规则**：`Repeat("X",2){ B }` → 重复数 = $\lceil Scope.sx / 2 \rceil$（自动适配尺寸）
- **Component split**：`Comp("faces"){A}` / `Comp("edges")` / `Comp("vertices")`——**拆到更低维度**（用"某轴 size=0 的 scope"编码低维形状，用 `S` 命令抬回高维实现挤出）

**🔴 本文的 key insight（论文自称"本文最关键的认识"）——两阶段范式**：

不要直接在"可见立面表面"上写规则（原文给出三条理由：可见面可能是一般多边形且不易计算；**"不清楚如何为一般多边形写有意义的规则"**；**"没有简单机制为立面文法指派非终结符，因为这些表面是算法的输出，而不是产生式规则的输出"**）。

**正确做法**：
```text
三维 scope 放置体块（质量模型 mass model）
        ↓ Component split 抽取面
得到"正确对齐、正确参数化"的二维 scope（含屋面等任意朝向面）
        ↓ 在二维 scope 上应用立面 / 屋顶规则
（且二维 scope 可被后续规则重新变回三维——如沿法线挤出）
```

> **这个"surface 是算法输出而不是规则输出"的陷阱判断，在 20 年后的生成式 AI 时代依然成立**：今天所有"生成的结果不可编辑"的问题，本质都是同一句话——**规则/语义没有作用在结构化的中间表示上。**

**两个一致性机制（让"体块"与"细节"对上账）**：

1. **Occlusion（遮挡查询）** —— 测试形状间相交，返回 `none / part / full`：
   - 支持 `"noparent"`（**排除父形状**——因为 split 的父形状永远遮挡子形状，这是必须的修正）
   - 支持 label 过滤（`Shape.occ("balcony")`）、距离扩大（`"distance", 4`）
   - 支持视线查询：`Shape.visible("street")` → **测试到街道的最短视线是否被遮挡 → 决定门开在哪面**（原文示例规则：`3: facade : Shape.visible("street") ; ... { tiles | entrance }`）
   - 典型用法：`1: tile : Shape.occ("noparent") == "none" ; window` / `2: ... == "part" ; wall` / `3: ... == "full" ; ε` ——**不行的地方换墙，全遮挡就删掉**
2. **Snapping（吸附线）** —— 把质量模型的所有面存为**全局构建平面** → 相交可得 **snap lines**：
   - repeat split 遇到 snap line：**按 snap line 分段后各自重复**（改变所有元素尺寸）
   - subdivide split 遇到 snap line：**只改动最接近切分的那两个形状**
   - 效果（原文 Figure 10）：**楼层线自动跨全部体块对齐**——例如塔楼收分（tapering）处，楼上部分被强制对齐到收分线以下

## Key Data（逐节核对，均为原文数字）

| 项目 | 数据 |
|---|---|
| 单楼生成速度 | **50k 多边形模型 ≈ 1 秒**（+0.5 秒写盘） |
| **庞贝（Pompeii）重建** | 与考古学家合作，**190 条手工规则**，36 个终端对象（+4 种树 + 环境）→ **14 亿（1.4 billion）多边形 @高 LoD** / 3100 万 @中 LoD / 17 万 @低 LoD |
| Beverly Hills 郊区模型 | ~150 条规则（含地块细分/植被/泳池/人行道），~1000 栋建筑，**~700M（7 亿）多边形**（不含树） |
| 规模上限 | *"十亿（billion）多边形量级的模型可在**一天内**生成"* |
| 行业对照 | 《超人归来》城市模型 = **15 人年** |
| **可用性测试** | 职业建模师：**第一天**学界面与工作流，之后**独立用两天做出一个小城市模型** |
| 渲染方案 | Pixar **RenderMan**（利用 delayed read archives 的 instancing + LoD）；屋面砖、柱头、窗格等小件用 **Maya** 补做 |

## Why It Works

- **规则可读、可复用**：同一套规则可生成大量变体（城市填充）；语义信息**在建模过程中就被指定**——这是"设计规则复用"的前提
- **避开布尔运算**：不依赖复杂易错的几何算法（boolean operations），**鲁棒性来自"规则在简单形状上定义、复杂多边形表面自然涌现"**（原文：无需执行"复杂易错的几何计算"）
- **推导速度与规模匹配**：不追求全局最优（原文自述"全局优化也许更好，但建模更难、时间可能高得不可接受"）——**工程上的取舍明确**

## Limitations（原文自述，三条都值得记）

1. **小尺度复杂细节低效**：屋面砖、柱头、窗格等用 Maya 手工生成——**"大规模结构自动化 + 小细节手工"**的边界 2006 年就已划出
2. **程序化有时生成不合理配置**（尤其从 GIS 任意足迹出发时）→ 原文提出未来方向：**"employ shape grammars for shape understanding"（用形状文法做形状理解）**
3. **🔴 实时渲染尚不可行**（原文 Discussion 原话）：*"我们正在合作构建实时渲染方案，但这需要**尚未开发的额外后处理算法**。一个主要挑战是为海量城市模型开发 LOD 技术。**因为我们目前不优化一致拓扑（consistent topology），现有算法会失败。**"*
   - → **这句话预告了后来十余年的自动 LOD / 虚拟几何研究**（Nanite 的方向）；也**划出了"离线生成"与"实时预算"之间那道至今存在的账本边界**

其他：学习曲线类似脚本语言；规则"写得不好只有原作者能懂"；与 L-system 的"生长"隐喻刻意保持距离。

## Game Development Relevance

- **CGA 是游戏/影视行业程序化城市的事实标准理论**：CityEngine 被明确应用于 game development / entertainment（见其官方定位）；本文 Pompeii / Beverly Hills 案例即"电影级大规模城市"的可行性证明
- **"离线生成量"与"运行时负载"是两本账**：本文生成 14 亿多边形的庞贝，但**渲染靠 RenderMan 离线**；实时侧只剩 LoD 问题——**这正是今天"PCG 生成 → 实时预算"分工的原型**
- **对分档工作的两条直接借条**（见 Learning Value）
- 与 **Houdini** 生态的关系：Houdini 的程序化建模（SOP + 规则/属性传播）与 CGA shape 同属"规则驱动生成"一脉（不同实现路径，未在原文中对照）

## Unreal Engine Relevance

- **UE PCG Framework（5.2+）**：同为"规则/图驱动的程序化生成"，但 UE PCG 是**运行时 + 编辑器混合**的图执行框架；CGA shape 是**离线文法推导**。**对照价值**：CGA 的两阶段（体块 → 表面 → 细节）与 PCG 图的"生成 → 过滤 → 变换"链是同一思路的两种表达。
- ⚠️ 无 Niagara / 材质侧映射（本文远早于这些系统）。

## Technology Evolution

```text
1975 Stiny 形状文法（建筑学，人工推导）
  ↓
1982 Stiny set grammar 简化 / 1991 L-system 植物（并行文法）
  ↓
2001 Parish & Müller 城市（简单体块 + shader）/ 2003 Wonka 立面（split 规则）
  ↓
2006 ★ CGA shape（本文）：复杂体块 + 一致细节的两阶段解法
  ↓
2007 Procedural Inc. 成立 → 2008 CityEngine 首商用版
  ↓  （规则语言 = CGA，市场：城市规划 / 游戏 / 影视 / 考古——庞贝案例即考古合作）
2011 Esri 收购 → Esri R&D Center Zurich（2020 年后更名 ArcGIS CityEngine）
  ↓
2010s–2020s：进入游戏管线 + Houdini 生态并行发展
  ↓
2026 ProxyBuild（LLM 推断角色 + 检索组装）—— 回应"规则编写费力"；
     并兑现 2006 写下的 "shape understanding" 方向 ★ 同日入库
```

## Relationships

### Based On
- Stiny 1975 形状文法（记名）；**Wonka et al. 2003 split 规则**（原文自述的直系祖先：*"我们基于 split 规则的思想构建"*）
- L-system（Prusinkiewicz & Lindenmayer 1991）：**记号启发**（scope 是龟形记号的演化），但文法类型刻意不同（顺序 vs 并行）

### Extends
- Parish & Müller 2001：从"简单体块 + shader 细节"扩展到"复杂体块 + 真实几何细节"

### Enabled
- **CityEngine（Esri）**：本文是它的理论基础与规则语言（CGA）来源
- 后续论文线（记名）：2007 Image-based Procedural Modeling of Facades（Müller 等）、2008 Interactive Visual Editing of Grammars、2008 Interactive Procedural Street Modeling、2009 Parallel Generation of L-Systems

### Followed By
- [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]] —— **20 年后的回答**：不写规则（LLM 解析属性 + 检索组装），但**保留"结构先于外观"的两阶段骨架**；且其 face-edge 语义推断正是本文"shape understanding"方向的兑现

### Contrasts
- **L-system（生长隐喻）** vs CGA（结构隐喻）——原文明确"生长对建筑常适得其反"
- **单体建模工具（Maya 等）** vs 规则系统：后者把"设计意图"而非"几何"变成可复用资产

## Personal Knowledge State

- `user_level: Normal`（推断：游戏开发背景足够听懂；形状文法形式化细节不要求）
- **三条可带走的原则（不需要读全文）**：
  1. **"表面是算法的输出，不是规则的输出"→ 规则必须作用在结构化的中间表示上**（2006 年的判据，今天评估一切"生成式资产不可编辑"问题依然适用）
  2. **"绝对 vs 相对"参数化**（原文用 `r` 后缀区分）：**建筑部件并非都等比缩放** —— 推而广之：**任何"整体缩放"方案都要先回答"哪些量是绝对的、哪些是相对的"**（与你的五档缩放问题直接同构：**特效参数也并非都随档位等比缩放**）
  3. **离线生成与实时渲染是两本账**：本文生成十亿多边形不眨眼，但实时化"需要尚未开发的后处理算法 + 新 LOD"——**"生成端多细"从第一天起就不等于"运行时多贵"**

## Learning Value

- **PCG 域的历史锚点 + 前沿对照**（与 ProxyBuild 同日入库，互为两端）
- **"两阶段推导"是一个可迁移的架构模式**：宏观（自由、可变）+ 微观（秩序、规则）之间**必须有一个显式的中间表示**（这里是"二维 scope"；在 ProxyBuild 里是"角色概率"；在游戏引擎里是"白模 + 材质槽位"）
- **容错设计思想**：occlusion 查询返回 `none/part/full` 三态 → **规则不是"全有全无"，而是"按可行性降级"**（能放窗放窗、不能放窗放墙、全遮挡删掉）——这是一个 2006 年就存在的"优雅降级"模板

## Visualization

![[程序化建筑_CGA shape 与 ProxyBuild 对照图解.html]]

## Notes

- **来源核对**：原文 PDF 已下载并**逐节核对**（10 页，来自 Peter Wonka 官方出版物页 `peterwonka.net`，`2006.SG.Mueller.ProceduralModelingOfBuildings.final.pdf`）。
- CityEngine 时间线（2007 分出 / 2008.7 首商用版 / 2011 夏 Esri 收购 / 2020.6 更名 ArcGIS CityEngine）来自多来源交叉（wikiwand、wikibin、中文百科），**非本文原文内容**。
- 一处原文自述的边界（值得留档）：**"我们不优化一致拓扑（consistent topology）"** —— 这是当时主动放弃的工程目标，后被证明是实时化的关键障碍。
