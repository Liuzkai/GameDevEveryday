---
type: paper
title: "ProxyBuild: Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies"
authors: [Xiang Tang, Ruotong Li, Xiaopeng Fan]
year: 2026
published: 2026-09-20
venue: "arXiv 2609.23386（cs.CV 主分类 + cs.GR 交叉；预印本，15 页 / 9 图）"
url: "https://arxiv.org/abs/2609.23386"
code: ""
project_page: ""
category: [procedural-generation, 3d-generation, asset-pipeline, architecture]
importance: B+
historical_importance: 1
game_relevance: 3
production_readiness: Research
user_level: Normal
status: unread
---

# ProxyBuild: Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies

## TL;DR

**用"语义推断 + 检索组装"替代"从零写规则或从零生成几何"。** 输入一个**已分块但无语义标签**的建筑外壳（点云重建 / GIS 数据 / LoD2 段），网络预测**每个面、每条边应该承载什么建筑组件**（墙/屋顶/窗 vs 柱/梁/栏杆），再由 LLM 解析文本属性、从资产库检索对应组件、按硬约束对齐组装。核心主张一句话：**生成的结构应紧锚定在基础几何的拓扑（面与边）上**，而不是浮在包围盒里。

> **一句话定位**：**这是 [[Müller — Procedural Modeling of Buildings (2006)]] 里"rule authoring is laborious"这个问题在 20 年后的一个回答** —— 不写规则，改**推断规则的作用对象**；同时它把 2006 年 Discussion 写下的未来方向——*"employ shape grammars for shape understanding"*——真正做成了网络模块。

## Problem

文本 → 3D 建筑的三条现有路线各有硬伤（引言原文归纳）：

| 路线 | 代表 | 硬伤 |
|---|---|---|
| **程序化生成（PCG）** | 形状文法（[[Müller — Procedural Modeling of Buildings (2006)]]） | **依赖专家先验知识，规则硬编码 → 难以承接自然语言指令**；LLM 直接写规则也不行——**"LLM 缺乏精确的 3D 空间感知与连续数值推理能力"**，几何约束下的规则求解做不了 |
| **生成式 3D** | Meshy / Trellis 等 | 输出是**隐式神经场或单一网格** → **无结构层级信息 → 不可交互编辑、不可资产复用**（下游管线要的是可编辑件） |
| **混合式** | BuildingBlock | 用**包围盒 + PCG 松堆叠**引入结构先验 → **忽略细粒度建筑边界约束 → 漂浮组件 / 网格互穿** |

> **三者失败的共同点**：**结构与几何脱节。** PCG 有结构但接不住输入；生成式接住了输入但没结构；混合式的"结构"（包围盒）粒度太粗。→ 本文的立场：**结构必须锚定在几何自身的拓扑上。**

## Historical Context

放在库内的程序化生成线上（本期同日入库 [[Müller — Procedural Modeling of Buildings (2006)]]）：

```text
1975 Stiny 形状文法（人工推导）
   ↓
2001 Parish & Müller 城市（L-system + 简单体块 + shader 细节）
   ↓
2003 Wonka 立面（split 规则）
   ↓
2006 Müller CGA shape（两阶段：质量模型 → 立面细节）★ 同日入库
   ↓
2008+ CityEngine 商业化（规则语言 CGA 成为行业工具）
   ↓
2026 ProxyBuild —— 不写规则，改为"推断 + 检索组装" ★ 本篇
```

**2006 年原文的两处预言，2026 年被本工作兑现**：

- 原文 *Discussion* 自述"程序化方法有时生成不合理配置"→ 提出未来方向 **"employ shape grammars for shape understanding"**（用形状文法做形状**理解**）→ **本篇的 face-edge bigraph encoder 就是这个"理解模块"**（从无语义壳中恢复语义角色）。
- 原文摘要即写 **"rule authoring is laborious"**（规则编写费力）→ **本篇用"LLM 属性解析 + 检索"把规则编写降级为"角色推断 + 资产检索"**。

## Core Idea

**MAPP（Mesh-Anchored Procedural Proxy，网格锚定的程序化代理）= 中间表示**：

$$P = (\mathcal{G},\, \Pi_f,\, \Pi_e)$$

- $\mathcal{G}$：face-edge incidence graph（面-边关联图）
- $\Pi_f$：每个面节点的**角色概率分布**（墙 / 屋顶 / 窗……）
- $\Pi_e$：每个边节点的**角色概率分布**（柱 / 梁 / 栏杆……）
- 两类分布都含一个 **Null 角色**，用于过滤"不需要实例化"的背景图元

**关键设计：表示不是最终几何，而是"绑在壳上的控制量"。** 外壳先被预分块（pre-partitioned）：**每个面 / 每条边 = 一个原子 proxy slot**（无标签）。于是"生成"被拆成两道互不耦合的问题：

1. **Proxy Prediction（结构合理性）**：从壳的几何 + 拓扑上下文推断每个 slot 的角色 → 与文本风格无关
2. **Proxy-to-Asset Instantiation（风格多样性）**：文本做条件 → 检索资产 → 空间姿态对齐 → 组装

> **这个解耦的收益（论文用三组实验证明）**：同一文本可以驱动不同壳（形态各异、风格一致）；同一壳可以配不同文本（拓扑角色不变、资产材质全换）。**"结构"与"风格"成为两个独立可调的旋钮。**

## Technical Approach

**① Proxy Prediction Phase**
- **face-edge bigraph encoder**：在预分块的语义无关网格上，**显式建模面区域（墙/屋顶）与折线（栏杆/柱）之间的特征交互**——论文强调"显式地传播与交互"，跨越了包围盒的表示局限
- 训练需要带 MAPP 标注的建筑数据集（论文自建）
- **消融证明两条流缺一不可**：去掉 **Edge Stream** → CLIP-S 0.339 / UNI3D 0.407 / NCR 92.1 / **ACC 76.4**（**细长边界构件识别崩掉**）；Full Model → **CLIP-S 0.349 / UNI3D 0.438 / NCR 95.7 / ACC 91.5**

**② Proxy-to-Asset Instantiation Phase**
- LLM（**论文写明用 GPT-5**）解析文本中的**风格与属性参数**
- **资产检索**：从资产库 $\mathcal{L}_{asset}$ 检索兼容资产（库 = Sketchfab 高质量模型 + **Blender Geometry Nodes 程序化生成的多风格组件**，均带文本描述与包围盒尺寸元数据）
- **硬约束（hard constraints）适配器**：空间放置逻辑 + 姿态对齐；**消融显示：去掉硬约束不影响角色预测（ACC 仍 91.5）但 NCR 暴跌到 73.6%** → **互穿/漂浮伪影全部来自实例化阶段的空间校正，不是来自预测阶段**

**③ 可编辑性（第三组实验）**
- 加 / 删组件、调整尺寸、**手工修正个别面或边的角色预测**、直接替换资产与材质
- 论文原话：允许"**从概念到详细设计的无缝过渡**"（seamless transition from conceptualization to detailed design）

## Key Data（逐节核对，均为原文数字）

**人评**（Table 2；20 名志愿者盲评，200 次独立评分，1–10 分）：

| 方法 | Visual Appeal | Structural Plausibility | Text Consistency |
|---|---|---|---|
| **ProxyBuild** | **8.2** | **8.1** | **8.5** |
| BuildingBlock-Box | 7.5 | 7.7 | 8.4 |
| ShellMaker | 7.3 | 7.5 | 7.2 |
| Meshy | 7.1 | 6.7 | 7.9 |
| Trellis | 6.4 | 6.2 | 7.4 |

**两个新指标（值得记）**：
- **NCR（Non-Collision Rate）**：组件间非法体积相交（如窗插进墙）的比例反向指标
- **ACC（Proxy Accuracy）**：图编码器对每个拓扑图元角色分配的准确率
- 另有 FID / KID / CLIP-S / **UNI3D**（视图不变的 3D-文本对齐）

**泛化测试**：直接跑 **3DBAG LoD2 段**（真实城市数据）与 **BuildAnyPoint 点云重建网格**（不同拓扑密度）——证明"不挑壳的来源"。

## Limitations（原文自述 + 我的补充）

1. **图编码器性能被输入壳的拓扑有效性约束**（原文 Fig.8a）：分块太粗 → 立面缺层级与语义多样性；过度细分 → 图节点开销大 + **物理尺度太小反而不利于组件实例化**。解法：结构感知重网格预处理（自适应平面细分 / QSlim 简化 + QuadriFlow 重网格）
2. **资产检索依赖有限离线库**：遇到未见或稀有的文本风格，映射能力受限 → 未来方向"**on-the-fly 原生 3D 生成**"（动态开放词汇资产扩展）
3. **（我的评估）它是离线资产生成，不涉运行时**：无实时开销讨论，无 LOD / 流式讨论——**"生成端多细"与"运行时多贵"这本账没算**（对照：[[Müller — Procedural Modeling of Buildings (2006)]] 当年明确指出"实时化需要新的 LOD 技术"）
4. **核对边界**：全文 HTML 逐节核对（摘要 / 引言 / 3.1–3.3 / 4.1–4.3 / Discussion）；**Table 1 的具体数值（FID/KID）未逐格核对**（正文只给了"across all metrics 显著优势"的定性表述）；代码与项目页未找到。

## Game Development Relevance

- **直接对应"扫描/生成资产 → 可编辑资产"的管线环节**：点云重建网格（摄影测量 / NeRF / 3DGS 反推的壳）→ **语义可编辑组件**。这条链在 3A 生产的"实景转游戏"流程里有真实需求。
- **"可后编辑性"（post-editable）是游戏资产生成的关键约束**：本文的编辑能力（修正角色预测 / 替换资产）本质是**把美术的修正动作限定在"语义槽位"而不是"网格顶点"上**。
- **对分档/预算体系的间接价值**：**"结构（拓扑）与风格（资产）解耦"= 产线级的参数化**。它让"换风格"不必重做结构——这与五档画质里"同一资产、多档表现"的诉求同构，但**作用在离线侧**，不改变运行时账本。
- ⚠️ **与用户的 VFX 工作无直接交集**；价值主要在 **PCG 域补全 + 方法论的跨域对照**（见下）。

## Unreal Engine Relevance

- **对照物是 UE PCG Framework**（5.2+）：UE PCG 是"图执行框架 + 规则/属性驱动"，ProxyBuild 是"图结构 + 神经推断角色"。**两者可以形成一个有趣的追问：UE PCG 的图里那些'决定放什么'的节点，哪些可以换成'推断'而不是'手写'？**
- 无 Niagara / 材质侧映射（不涉及）。

## Technology Evolution

```text
规则驱动的程序化生成（2001–2008，CityEngine 一线）
        ↓  规则编写成本 + LLM 接不住几何约束
生成式 3D（2023–2025，NeRF/3DGS/扩散 → 单网格，不可编辑）
        ↓  结构缺失 → 下游管线用不了
混合式（包围盒 + PCG 松堆叠，BuildingBlock 一类）
        ↓  粒度太粗 → 漂浮/互穿
拓扑锚定的推断 + 检索组装（ProxyBuild 2026）★ 本篇
        ↓  下一条可能的路（原文自述）：资产库 → on-the-fly 生成（开放词汇）
        ↓  更远：动态图细分 / pooling（网络侧自适应拓扑）
```

## Relationships

### Based On
- [[Müller — Procedural Modeling of Buildings (2006)]] —— **同日入库、互为前后**：CGA shape 定义"规则驱动的两阶段生成（质量模型 → 立面细节）"；ProxyBuild 保留"两阶段"骨架，把"人写规则"换成"网络推断角色 + LLM 解析属性 + 检索组装"。
- 形状文法 / 程序化建筑线（Parish & Müller 2001、Wonka 2003）：PCG 侧的祖先（记名，未单独入库）。

### Related
- [[Magpie — Real-Time World Renderer for Interactive Games]] —— **同一个"分层收敛"母题的远亲**：Magpie 里"引擎管规则、生成模型只重画白模"；ProxyBuild 里"结构语义靠推断、几何靠检索复用，谁都不从零生成"。两者的共同点：**让生成模型只碰它最擅长的那一层**。
- [[ESG — Generating Physically Consistent Dynamic 3D Scenes from Text]] —— 同为"文本 → 场景/建筑"的生成式路线对照：ESG 追求物理一致的动态场景，ProxyBuild 追求结构可编辑的静态建筑。
- [[HairCS — Reconstructing Strand-Based Hair from Hair Cards]] —— **同题异构的"表示升档"**：HairCS"发片 → 发丝"、ProxyBuild"无语义壳 → 语义组件"——**都是"从不可编辑表示恢复到可编辑表示"**。

### Contrasts
- **Trellis / Meshy（生成式单网格）** vs ProxyBuild（结构 + 检索）：**"生成几何" vs "生成结构、复用几何"**。
- [[DLSS 5 — Generative Neural Rendering]] / [[2026-09-17-PBR-Latent — Physically Based Rendering in the Latent Space]] —— 与本篇构成**"生成模型只做决策层"的三个不同深度**：DLSS 5 只做画面重建、PBR-Latent 只补物理残差、ProxyBuild 只做结构决策。

## Personal Knowledge State

- `user_level: Normal`（推断：PCG 域概念 + 游戏开发背景够用；形状文法、图神经网络细节不要求）
- 建议**只取三句话**：
  1. **"生成的结构应紧锚定在基础几何的拓扑上"** —— 评估一切"生成式资产"方案时先问：**它的结构锚在哪里？**
  2. **"可编辑性 = 修正动作落在语义槽位而非网格顶点上"** —— 进产线的自动生成资产的硬门槛。
  3. **消融的直接教训**：**结构预测准 ≠ 摆放正确**（ACC 不变而 NCR 暴跌）——**"知道放什么"与"放得对"是两笔账**。
- ⚠️ 若对 PCG / 自动资产生成无兴趣，本篇可只保留上面第 1、3 两句。

## Learning Value

- **PCG 域在库中的第一个 2026 样本**（配合 [[Müller — Procedural Modeling of Buildings (2006)]] 完成"源头 + 前沿"双锚点）。
- **"生成不碰资产本体"模式的又一实例**：本地图（结构决策）与几何/外观（检索复用）分离。
- 新指标 **NCR / ACC** 可作"结构化生成"类工作的验收语言。

## Visualization

![[程序化建筑_CGA shape 与 ProxyBuild 对照图解.html]]

## Notes

- 提交 2026-09-20 06:11 UTC；cs.CV 主分类、cs.GR 交叉；**它躲过了 cs.GR 的 recent 页**（非主分类），由 API `submittedDate` 窗口查询捕获——**这是本月第三次验证"API + recent 页双通道"的必要性**。
- 引用 [5]、[12] 应为形状文法/PCG 一脉文献（未核对具体条目）。
- "GPT-5"为论文原文写法（[33]），按原文记录。
