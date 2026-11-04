---
type: concept
user_level: Normal
aliases: [PCG, Procedural Modeling, 程序化生成, 程序化建模]
prerequisites: []
first_introduced: "1970s（分形/噪声线）/ 1975（形状文法线）"
---

# Procedural Content Generation

## Definition

**用算法、规则或过程生成内容，而不是逐件手工制作。** 内容可以是几何（地形、植被、建筑、城市）、纹理、关卡、任务、叙事等一切游戏资产维度。

核心形式化：**把"内容"分解为"生成过程 + 参数/种子"**。

## Core Principle

**为什么游戏开发需要它**（2006 年论文原文给出的理由在今天依然成立）：

1. **规模**：手工程度做不完——[[Müller — Procedural Modeling of Buildings (2006)]] 原文例证：《超人归来》城市模型 = 15 人年；而规则化后**十亿多边形量级的城市可在一天内生成**
2. **多样性**：同一套规则 + 不同随机种子/参数 → 大量变体（城市填充、资产变体）
3. **可复用与可编辑**：生成的是**设计意图**（规则）而非**几何**——这使"设计资产"可以像代码一样复用、审查、迭代

**代价/边界**（同样重要）：

- **规则编写本身是专业技能**（2006 原文自述 "rule authoring is laborious"；2026 年 [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]] 仍在解决这个问题）
- **可能生成不合理的配置**（尤以从任意输入数据出发时为甚）
- **生成的量与运行时的代价是两本账**——离线生成十亿多边形 ≠ 实时能画（[[Müller — Procedural Modeling of Buildings (2006)]] 明确："实时化需要新的 LOD 技术"）

## Prerequisites

- 基础几何与建模概念（体块 / 面 / 网格）
- （文法线）产生式系统 / 形式文法的基本直觉
- （噪声线）基础随机数 / 信号概念

## Historical Evolution

```text
1970s–80s  分形与噪声线：Mandelbrot 分形地形 / Perlin 噪声（1985）
           → 地形、纹理、云的生成基础
1975       形状文法线：Stiny《Shape Grammars》（建筑学，人工推导）
1980s–90s  生长模拟线：L-system 植物（Prusinkiewicz & Lindenmayer 1991）
           → "规则生成自然形态"的范式确立
2001       ★ 城市：[[Parish-Müller — Procedural Modeling of Cities (2001)]]
           （**扩展 L-system**：参数从规则外移（ideal successor + globalGoals +
           localConstraints，9 条规则覆盖全部示例）；**self-sensitive**：树状→网络；
           layered grids 半程序化立面；LOD = 迭代深度；CityEngine 前身）
           ★ 2026-09-27 入库
2003       ★ 立面：[[Wonka — Instant Architecture (2003)]]
           （**split grammar**：拆分/转换两类规则 + "子形状含于父形状"；**双重控制**：
           attribute matching（规则选择两级制）+ control grammar（设计想法空间分布）；
           **"规则选择"问题出处**；与 2001 同代互补——"多而简" vs "少而精"）★ 2026-10-02 入库
2006       ★ CGA shape：[[Müller — Procedural Modeling of Buildings (2006)]]
           （复杂体块 + 一致细节；两阶段推导；CityEngine 的理论基础；
           同一作者团队——Müller 于 2001 论文任二作）
2007       ★ 约束求解线开线（与文法线同期、方法论相反）：[[Merrell 2007 — Example-Based Model Synthesis]]
           （**不写规则，数约束**：六向邻接表 + 全局传播 + 切块回滚 + 加权相似度；
           附录证明"能否补完"是 **NP-完全**（Planar 3-SAT 归约）⇒ "总成功"不可能；
           自述失效：例子过紧 → 输出"反复复制例子"）★ 2026-11-04 入库
2007–2011  生产化：Procedural Inc. → CityEngine 商用 → Esri 收购
2010s      Houdini 成为影视/游戏程序化管线主力；
           UE PCG Framework（5.2+）把 PCG 做成引擎内一等公民
           ⚠️ 注意：UE PCG 是**逐点独立求值**，属于"独立随机数生成器"——
           对有"沿某条线携带的量"的图案，出缝是结构性的（见 2026-11 条）
2016       WFC（Gumin，单头文件 C#）：**同一算法换了包装**（作者本人：nearly identical）
           → 独立游戏 / 生成艺术生态爆红，成为"程序化生成"的代名词之一
2021–2023  （约束线续）块求解器（Boris the Brave 2021 / Kleineberg 2023）：
           为无限世界铺**固定周期背景**、按块重解，**解不出来就保留背景**
2020s+     机器学习线：神经生成（GAN/扩散/NeRF/3DGS）进入资产与场景生成
2026       混合线：[[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]]
           （LLM 解析属性 + 网络推断结构角色 + 检索复用资产）
           —— "不写规则，推断规则的作用对象"
2026-09    混合线（续）：[[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction]]
           （单图 → 可编辑场景：几何/布局双流共生成；布局用"稠密有界对应"替代"稀疏无界位姿"）
           —— "不猜位姿，恢复对应、再交给几何对齐"（同一母题的第三天第三例）
2026-09    地形/水系线（新分支）：[[2026-09-27-Plate-Local River Generation]]
           （板块局部河流："确定性 + 地形感知 + 下坡保证 + 无限域"四者首次同时成立；
           懒加载逐板块缓存——"有界域 + 确定性"首次进入 PCG 生成算法内部）★ 2026-10-03 入库
2026-10    ★ 编辑线（新分支）：[[2026-10-19-InfiniDream — Open-Ended Photorealistic 3D Worlds Through Procedural Generation|InfiniDream]]
           （**不写新库、改既有库**：AST 深度 2 依赖闭包圈出"可编辑表面"，三类策略
            Tweak/Create/Promote；放置改为确定性视锥网格 + 五道闸门，拒绝让 VLM 猜坐标）
           —— "规则编写费力"的第三种解法：不是替你写，是替你**改**）★ 2026-10-21 入库
2026-10    ★ 调度线（新分支）：[[2026-10-20-Ma — WorldStream Streaming Interactive 3D Worlds with Hierarchical Planning and Persistent State|WorldStream]]
           （**PCG 第一次被当成"运行时"而不是"开局前"**：Macro 定结构 / Micro 定语义 →
           有界联合求解（独立 argmax 有 63% 不可执行）→ 确定性本地编译；
           两个前沿（语义跑在前面、几何只做眼前）+ 一份 tile record 撑住"卸载—重访—带编辑复原"）
           —— "边界承诺"首次成为 PCG 的一等对象：独立生成的瓦片之间必须共享一条权威记录）
2026-11    ★ 理论收口：[[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share]]
           （**"保留背景"被证明是数学必然**：背景就是那个被携带的量本身；
           违规密度地板 ≥1/r 阶、半径无穷也不行；必须沿该方向共享随机性；
           **相位周期必须整除分块周期**；只读例子的【证书】52 例 certify 13、其余 31 例零）★ 2026-11-04 入库
```

## Important Papers

- [[Parish-Müller — Procedural Modeling of Cities (2001)]] —— **城市级源头**（扩展 L-system：参数外移；self-sensitive 网络拓扑；四段流水线 = 今天"地形→路网→地块→建筑"生成链的原型；CityEngine 最早出处）★ 2026-09-27 入库
- [[Wonka — Instant Architecture (2003)]] —— **立面细节侧锚点**（split grammar + 双重控制；"规则选择"问题出处；CGA shape 的直接前置；与 2001 同代互补、与 2006 同一批人接力）★ 2026-10-02 入库
- [[Müller — Procedural Modeling of Buildings (2006)]] —— **程序化建筑的源头**（CGA shape；CityEngine 理论基础；**与 2001 同一作者团队、相隔 5 年的直接接力**——2006 兑现了 2001 的 Future Work）
- [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]] —— **2026 前沿样本**（推理 + 检索的混合范式；"结构锚定拓扑"原则；回应 2001 起就存在的"规则编写费力"）
- [[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction]] —— **2026 前沿样本（续）**（单图→场景；"稠密有界对应"替代"稀疏无界位姿"）
- [[2026-09-27-Plate-Local River Generation]] —— **自然地形 / 水系侧首个样本（2026）**（"确定性 + 地形感知 + 下坡保证 + 无限域"四者同时成立；板块双表示 + 懒加载逐板块缓存 + 构造性下坡保证——**"有界域"从空间/布局问题延伸到"生成算法自身的域划分"**；开放世界水系）★ 2026-10-03 入库
- [[2026-10-19-InfiniDream — Open-Ended Photorealistic 3D Worlds Through Procedural Generation|InfiniDream]] —— **"改规则"侧样本（2026）**（SCE：AST 深度 2 的依赖闭包 = 可编辑表面；Tweak/Create/Promote 三策略；确定性视锥网格放置 + 五道闸门；池化 N→K）。**"规则编写费力"的第三种解法——不替你写，替你改** ★ 2026-10-21 入库
- 记名（未单独入库）：Stiny 1975《Shape Grammars》；Prusinkiewicz & Lindenmayer 1991（L-system 植物）；Perlin 1985《An Image Synthesizer》
- [[2026-10-20-Ma — WorldStream Streaming Interactive 3D Worlds with Hierarchical Planning and Persistent State|WorldStream]] —— **"运行时 PCG"侧样本（2026）**（Macro/Micro 分层决策 + 有界联合求解 + 确定性本地编译 + 两个前沿 + tile record 持久化；**"边界承诺"首次成为一等对象**）★ 2026-10-28 入库
- [[Merrell 2007 — Example-Based Model Synthesis|Merrell 2007 — Example-Based Model Synthesis]] —— **约束求解线源头**（**"例子即规则"**：六向邻接表 + 全局传播 + 切块回滚 + 按例子出现次数加权；附 NP-完全证明，"总成功"不可能；自述"过紧会复读"。**这是 2016 之后独立游戏 PCG 主流的真正出处**，虽然名字被 WFC 占了）★ 2026-11-04 入库
- [[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share|IW 2026]] —— **约束线的理论收口**（**"局部决定不了"是可判定性质**：沿某条线被携带的量 ⇒ 独立随机数生成器违规密度 ≥1/r 且半径无穷也不行；必须沿该方向共享随机性；**分块周期必须整除相位周期**；只读例子可算的证书。**"降档不能解决问题"的最干净样本**）★ 2026-11-04 入库

> **"结构锚定"母题三例**（9-21 ~ 9-23 连续入库）：Müller 2006 锚在"二维 scope" → ProxyBuild 锚在"面-边拓扑" → Mira-Scene 锚在"像素 ↔ 规范坐标"。**锚什么可以变，但"生成必须作用在可对齐、可查询、可修正的中间表示上"不变。** Mira-Scene 还给出本母题目前最干净的量化判据：**稠密化不够，"有界"才是关键**（场景空间稠密预测 3D-IoU 0.379 vs 有界规范空间 0.727）。

> **"拆出去"三步曲（2026-10-02 随 Wonka 2003 入库沉淀）**：2001 把**参数**移出规则（ideal successor）→ 2003 把**设计想法 + 选择策略**移出规则（control grammar + attribute matching）→ 2006 把**表面**移出规则（两阶段中间表示）。**每次都是把"不属于规则职责的东西"移出去、给它独立的表示**——"规则只管结构"被三次推到极致。

> **"拆出去"第四步：把"衔接"移出规则（2026-10-28 随 WorldStream 入库）**。PCG 生成一块地很容易，生成**相邻的两块**很难——两块各自"合理"的规则会在接缝处给出对不上的路、断掉的水系。传统做法是让人手工拼（或干脆生成一整张大地图）。WorldStream 的解法是**把边界本身变成一份记录**：相邻瓦片读**同一条** boundary record（规范道路端口、边缘高程、生态过渡），各自把几何路由到它。于是"独立生成"与"接得上"第一次不矛盾。
>
> 附带一个反直觉的量化：**没有邻接上下文时，局部规划会把每块地都建满**——2,686 栋 vs 250 栋（10.7×），并出现城市/非城市的硬切边（边界跳变 0.361 vs 0.089）。**"局部最优"在 PCG 里表现为"过度建造"**：每一块都在最大化自己的内容量，因为没有"邻居已经承诺了什么"这个约束。这条对关卡设计同样成立——**没有全局配额的生成，一定会在每个局部都填满**。

> **与"拆出去"相反的一条线（2026-11-04 随 Merrell 2007 + IW 2026 入库沉淀）**：上面四步都在「把不属于规则职责的东西移出去、给它独立表示」；**约束线一开始就不写规则**——把"风格"交给一个**例子**，把"合法"交给一张**邻接表**，把"怎么填"交给**搜索**。两种世界观的代价完全不同：文法线的资产是**规则**（可读、可改、可审、能解释），约束线的资产是**例子**（做得出、感觉得到、但没人能说出它为什么长这样）。**这不是进步关系，是两条并存的路**——2007 与 2006 同年同会，就是这两种哲学并排站着。
>
> **两条线在 2026 年撞上了同一个问题：分块。** WorldStream 用 boundary record 承诺衔接；Merrell 的块求解器用"保留背景"承诺衔接；[[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share|IW 2026]] 证明**这两个办法的能力上限不同**：边界能承诺**不变量**（边界上定死，有效性沿行把它带过去），承诺不了**相位**（相位的周期必须整除分块周期，接缝对齐也没用）。
>
> **一条可以直接用的判据**：先问你的图案有没有"**必须沿某条线保持一致**"的量（砖缝位置 / 栅栏间距 / 道路接续 / 铺装对缝）。有 → 局部生成必然出缝，而且**加大 chunk 只是把缺陷从 1/8 降到 1/512**（$1/r$ 地板，永远到不了零）；没有 → 独立随机就够了。**判据比参数重要。**

> **"决策 / 构造分离"的三种资产形态（2026-10-21 随 InfiniDream 入库沉淀）**：[[2026-10-13-Co-DSL — Human-LLM Co-design of Reusable Spatial Concepts for Layout Generation|Co-DSL]] 让 LLM 养成**概念库**（资产化）；[[2026-10-19-InfiniDream — Open-Ended Photorealistic 3D Worlds Through Procedural Generation|InfiniDream]] 让 LLM 编辑**程序化代码库**（模型化）；[[2026-10-16-Scene-Designer — Learning Design Intelligence for Agentic 3D Scene Generation|Scene-Designer]] 把判断力**训进模型**。三条路的共同前提完全一致：**VLM 只做离散、可验证的决策，连续几何交给确定性算法**——差别只在"被编辑的那个资产"长什么样。

## Related Concepts

- [[Particle Systems]] —— **程序化生成的同族**：Reeves 1983 的"粒子系统树 + 随机过程"是**动态现象**的程序化生成（火焰/云/爆炸）；建筑/地形是**静态资产**的程序化生成。**共享同一组成本法则**（屏占比 × 密度、层级数量）
- [[Procedural Materials]] / [[Texture Synthesis]]（未入库，记名）—— 材质与纹理维度的程序化生成
- [[World Models for Games]] —— "生成"的另一种口径：世界模型生成的是**体验/状态**，PCG 生成的是**内容**

## Technologies

- **Houdini**（影视/游戏程序化管线主力；SOP + 属性传播）
- **CityEngine / ArcGIS CityEngine**（CGA shape 的商业化身，规则语言即 CGA）
- **UE PCG Framework**（5.2+；图执行框架，可与 Niagara / 地形 / 水体系统交互——**离用户最近的落地形态**）
- [[Wave Function Collapse]]（约束求解式 PCG 的代表，独立游戏生态常用；**算法本体 = [[Merrell 2007 — Example-Based Model Synthesis|Merrell 2007]]**，2016 年 Gumin 的单头文件实现让它进入生态）
- Blender Geometry Nodes（[[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]] 的资产库即用它生成多风格组件）

## Game Applications

- **开放世界**：地形、植被、道路、聚落、**水系**（2026 首个样本：[[2026-09-27-Plate-Local River Generation]]）（NGR 一类项目的基建）
- **城市/建筑**：CityEngine 一线（本文案例：庞贝重建 / Beverly Hills）
- **资产变体**：同一资产的程序化变体（降本）；ProxyBuild 展示的"同壳不同文本"即风格变体生成
- **关卡/地牢**：[[Wave Function Collapse]] 一线（以撒的结合、Bad North 等）
- **动态现象的预生成**：粒子系统的"生成过程"本身（Reeves 1983）

## Personal Knowledge

- `user_level: Normal`（推断：游戏开发背景对"程序化生成"概念不陌生；**形状文法形式化细节与图神经网络不做要求**）
- **库内重要连接**：用户工作（NGR / 开放世界）中 **PCG 生成的场景 = VFX 的承载容器**——PCG 产出多少几何/多少半透明植被，会直接影响运行时特效预算的"底噪"。**这是 PCG 域与你的预算工作的真实交叉点。**
- 另：用户在整理游戏开发知识库时已涉及 Houdini 方向 → 本概念的"技术"清单可直接对接
- **五条可迁移抽象（2001 / 2003 / 2026 河流线陆续提炼，均不需要读全文）**：
  1. **规则只描述结构、参数由外部策略决定**（ideal successor）——判据：任何规则/生成系统，先问"**哪些部分该从规则里移出去**"；
  2. **约束"有条件通过"：允许 + 标记 + 后期替换**（高速跨水 → 桥/隧道）——约束处置是三档（拒绝/修正/条件接受），不是二值；
  3. **细节层次 = 迭代深度**（decreasing apices）——LOD 可以是生成语法的副产品，不必是独立资产链；
  4. **"选择策略"独立于"规则内容"**（[[Wonka — Instant Architecture (2003)]]）——规则库越大越需要独立的规则选择层（attribute matching：硬条件过滤 + 软偏好抽样）；判据：*约束越多，检查"怎么选"是否已从"规则本身"里拆出来*。
  5. **"先定域、后算局部"**（2026 河流线，[[2026-09-27-Plate-Local River Generation]]）——**确定性构造的域划分**让"需要全局视野"的问题逐块独立（**懒加载的前提是确定性**：可任意丢弃、重建、并行）；判据：*你面对的问题能否切进一个"可确定性构造的分区"，使每块独立可算？*（与"有界性"母题、chunk streaming / World Partition 同源）
  6. **"可改范围"要可机械判定，"放哪里"要能搜出来**（2026 编辑线，[[2026-10-19-InfiniDream — Open-Ended Photorealistic 3D Worlds Through Procedural Generation|InfiniDream]]）——自由度不是"给不给权限"，而是**权限的拓扑范围**（AST 深度 2 的依赖闭包）；位置不是"预测出来的"，而是**搜索 + 五道布尔闸门**出来的（可见 / 碰撞 / 间距 / 表面策略 / 投影面积）；判据：*你的工具里"哪些参数能被上层改"是约定还是可计算？"放哪"是猜的还是枚举的？*
  7. **先找"局部决定不了的那个量"，再谈参数**（2026 约束线，[[Merrell 2007 — Example-Based Model Synthesis|Merrell 2007]] + [[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share|IW 2026]]）——**"过紧会复读"和"加大 chunk 也只是变小"是同一件事的两端**：前者是例子只允许一个世界，后者是多个世界必须沿整条线共享随机性。判据：*你的图案里有没有一个量必须沿某条线保持一致？有 ⇒ 它必须被显式提供，任何"加大分辨率/加大块"的做法只是把缺陷改成 $1/r$，永远到不了零。*
- **成本账本（2001 原始数据）**：13,000 栋建筑 ≈ 路网 <10 秒 + 建筑约 10 分钟（2001 硬件）——"database amplification"的另一面是**生成时间成本**；与"离线生成量 ≠ 运行时负载"是同一条账本。

## Learning Gap

- 无硬性缺口。若要深入：
  - **形式化层**（形状文法的推导/优先级/`r` 相对值机制）→ [[Müller — Procedural Modeling of Buildings (2006)]]
  - **2026 前沿层**（拓扑角色推断、检索组装、NCR/ACC 指标）→ [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]]

## Next Step

1. （可选，30 分钟）如果在项目中用过 UE PCG：对照 Müller 2006 的"两阶段"（体块 → 表面 → 细节），检查自己图里"结构决策"与"细节填充"是否分离——**分离则改档/换风格便宜，耦合则贵**。
2. （可选，10 分钟对照）打开手边任意一个生成工具的图/节点网络，找一找 **"结构"与"参数"的边界**在哪——2001 的 ideal successor 机制就是这个边界的最早形态（规则只管结构、函数管参数）。
3. （观察项）ProxyBuild 式的"推断 + 检索"是否渗入引擎工具链（UE PCG 的自动规则生成一类）。
