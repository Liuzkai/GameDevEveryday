---
type: paper
title: "Procedural Modeling of Cities"
authors: [Yoav I. H. Parish, Pascal Müller]
year: 2001
published: "2001-08-01 (SIGGRAPH 2001, pp. 301–308)"
venue: "ACM SIGGRAPH 2001 Conference Proceedings (Technical Papers)"
url: "https://doi.org/10.1145/383259.383292"
code: ""
project_page: ""
category: [procedural-generation, urban-modeling, l-system, cityengine, classical]
importance: S
historical_importance: 5
game_relevance: 4
production_readiness: Research
user_level: Normal
status: unread
aliases: [Parish & Müller 2001, CityEngine 2001, 程序化城市生成]
tags: [pcg, classic, l-system, city, cityengine, urban, production-pipeline]
---

# Procedural Modeling of Cities（Parish & Müller 2001）

## TL;DR

**程序化城市生成的奠基论文，也是 CityEngine 这一名字与系统的最早出处。** 它第一次把"从零生成一整座城市"做成一条完整流水线：

```text
输入 image maps → 路网（扩展 L-system）→ 地块分割 → 建筑几何（随机参数化 L-system）→ 立面纹理（layered grids）
```

**两个方法级贡献至今仍在被继承**：

1. **扩展 L-system**——把"参数计算"从产生式规则里**整体外移**到两个外部函数（`globalGoals` + `localConstraints`）：规则只生成"理想后继"（ideal successor，参数未赋值），参数由上层目标决定、再被局部环境修正。收益巨大：**全部示例只用 9 条规则（w + p1–p9）**，且**加新目标/约束完全不需要改规则**；
2. **self-sensitive L-system**——让生长语法**自己查询已有结构并修改自身**，使拓扑从"树状"变为"网络状"（道路要成环、要交叉，死胡同是例外）。

**对库的意义**：[[Procedural Content Generation]] 的**第二锚点**（继 [[Müller — Procedural Modeling of Buildings (2006)]] 之后），且两篇是**同一作者（Pascal Müller 为本文二作）相隔 5 年的直接接力**——2006 的 CGA shape 正是本文 *Future Work* 清单里"建筑生成"一节的原话兑现（见下文 Relationship）。

> **规模数据（原文）**：虚拟曼哈顿约 **13,000 栋建筑**——路网图生成 **<10 秒**，地块分割 + 建筑生成 **约 10 分钟**；另一例 **26,000 栋建筑**。→ 对 PCG 管线的"生成账本"有直接量级参考（离线时间成本）。

## Problem

2001 年做"虚拟城市"的几条路各缺一角（本文 Related Work 逐条点评）：

| 现有路线 | 缺什么 |
|---|---|
| 航拍图 + 计算机视觉重建 | 依赖照片输入，**无法"从零"创造**不存在的城市 |
| 城市可视化研究（数据管理/实时/内存） | 只管画得快，不管**生成** |
| Alexander《A Pattern Language》（250+ 条模式） | **非形式化**（not formalized），不能用于自动生成 |
| Space Syntax（Hillier） | **分析性**、依赖已有城市地图 |
| Shape grammar（Stiny 1975） | 构造二维图案与交互设计，**未被用于城市规模** |

**缺口**：没有系统能**从少量统计/地理数据出发、从零生成完整城市**（原文："To our knowledge, there is no such system available."）。

## Historical Context

两条历史支线的汇合点：

```text
L-system 植物线：Prusinkiewicz & Lindenmayer → 植物几何
   └─ 关键优势（本文引 Smith 1984）："database amplification"
      —— 少量规则放大出大量数据（压缩形式 → 解压形式）
   └─ Open L-system（Mech & Prusinkiewicz 1996）：环境交互（本文 streets 的
      "道路降低周围人口密度"机制直接援引此模型）

形状文法线：Stiny 1975《Shape Grammars》（建筑学，人工推导）
   └─ 本文未用（面向城市）→ 但 5 年后由同一作者团队改造为 CGA shape

城市需求侧：电影/游戏行业对"快速创建复杂环境"的高需求（原文明确点名）
```

**本文位置**：把 L-system 从"植物"推向"城市"，是 PCG 域**第一个大规模人工环境生成系统**。

## Previous Work

- **L-system 植物建模**（Prusinkiewicz 1990；Deussen et al. 1998 植物生态系统）——证明生长语法能生成"复杂而自洽"的大规模结构；
- **Open L-system**（Mech & Prusinkiewicz 1996）——证明 L-system 可以与环境互相影响（本文的 streets 部分沿用其"道路降低周边人口密度"的机制）；
- **城市交通研究**（Fuesser 1997）——本文的路网模式分类（raster vs radial）与"高速公路 vs 街道"双类型划分依据；
- **四城案例数据**（Focas 1998，The Four World Cities Transport Study）——纽约/巴黎/东京/伦敦的统计输入数据来源。

## Core Idea

### 1. 流水线 = 四段串行的"生成器链"

```text
Geographical / Sociostatistical Image Maps（输入）
        ↓
Roadmap creation —— 扩展 L-system（highways + streets 两级）
        ↓
Division into lots —— 递归分割（最长、近似平行的边；面积阈值）
        ↓
Building generation —— 随机参数化 L-system（skyscrapers / commercial / residential）
        ↓
Textures —— layered grids（半程序化立面）
        ↓
Parser → 几何 + 着色器 → 任意多边形渲染器
```

每级的输出是下一级的输入——**这就是今天 Houdini SOP 链 / UE PCG graph 的直系原型**。

**输入数据的两个类别**（原文分类）：
- **地理图**：高程 / 水-植被边界；
- **社会统计图**：人口密度 / 区划图（住/商/混）/ 路网模式图 / 最大建筑高度图。

### 2. 扩展 L-system——把参数计算从规则里"搬出去"

**动机（原文）**：写复杂规则系统时，参数与条件缠在产生式里，"每次加一个新约束，大量规则都要重写"（extensibility 灾难）。

**修法**：L-system 每步只生成 **ideal successor（理想后继）**——结构对、参数空；随后两步外部函数填充：

```text
① L-system 生成 ideal successor（参数 UNASSIGNED）
        ↓
② globalGoals(ruleAttr, roadAttr) —— 按全局目标写入参数
     （人口密度 / 路网模式；控制角度、长度、分支延迟）
        ↓
③ localConstraints(roadAttr) —— 按局部环境修正参数
     （越界裁剪/旋转、交叉检测；找不到合法解 → 标记 FAILED → 下一轮删除）
```

**结果**：全部路网只需要 **9 条产生式**（公理 w + p1–p9），新增目标/约束只写函数、**规则一字不改**。这是"关注点分离"在语法系统上最干净的早期范例——**规则描述"什么后继"，策略回答"参数是多少"**。

### 3. self-sensitive L-system——打破树形拓扑

植物 L-system 天生是树（无环）；而"交通系统里死胡同是例外"，道路必须能**交叉、闭合、成网**（灵感旁证：血管生成，Meier 1999）。修法：在生长过程中**查询已有道路、并修改自身**——

- 新路段与既有路段相交 → **裁剪并生成交叉口**；
- 端点靠近既有交叉口 → **延伸接入**；
- 接近相交 → **延伸成十字**。

拓扑由此从 tree-like 变为 **net-like**——这是"生长语法生成非树结构"的最早完整方案。

### 4. 四种路网模式——global goals 的"可混合"设计

| 模式 | 规则 |
|---|---|
| **Basic** | 无叠加模式，纯人口密度自然生长（老城区） |
| **New York** | 全球/局部统一角度 + 单街区最大长宽（方格） |
| **Paris** | 围绕中心（可算可设）的放射轨道 |
| **San Francisco** | 沿最平高程走线 + 短陡街连接不同高度层 |

**多模式在同一地点同时激活时，按输入灰度图权重加权求和混合**——"网格与放射在一个城市里渐变过渡"由此免费获得。注意：**混合发生在参数层（因为参数已被外移），规则层无需知道这件事**。

### 5. 建筑与纹理

- **建筑**：随机参数化 L-system；三种类型由 zoning 图控制；从任意 ground plan 出发做变换/挤出/分支/终止，配屋顶、天线等几何模板；
- **LOD 的语法级方案（值得单独记）**：采用 Hart 1992 的 "decreasing apices" 受限类——**公理是建筑的 bounding box，每轮迭代可解释为一次几何细化**（refining step）。即：**细节层次 = 语法迭代深度**——LOD 与生成语法的统一，比手工做 LOD 链优雅得多；
- **纹理 = layered grids（半程序化）**：三条立面观察 → 假设（① 立面是叠置/嵌套网格，单元格功能趋同；② 特定单元格影响邻格尺寸位置；③ 不规则性影响整行整列而非单格）→ 数据结构：**interval group（一维区间组）→ layer（两组 + eval 函数 + col 函数）→ layer stack（未激活点降层求值）**。砖墙/门窗用照片扫描元素随机化，兼顾真实细节与内存。

## Technical Approach（关键实现细节）

**高速公路寻峰（globalGoals 的人口密度部分）**：每个 highway 端点沿半径预设范围**放射采样人口密度图**，以"到端点的反距离"加权求和，**取加权和最大的方向**继续生长——这是"高速公路连接人口中心"这句话的可执行版本（图 4）。

**局部约束对非法区域（水/公园）的三档处置**（很实用的一手）：
1. **裁剪（prune）**：缩短路段长度至合法范围；
2. **旋转（rotate）**：在最大角度内旋转至完全落入合法区——由此**自动产生"沿河/沿公园边界走"的道路**；
3. **有条件接受 + 标记（highway 专属）**：允许高速公路跨越非法区一定长度，打 flag，进几何阶段**换成桥或两个隧道入口**。

> 第三档是很好的模式：**不是"拒绝违例"，而是"允许 + 标记 + 后期替换"**——把约束的解决推迟到手上有更多信息的阶段。

**曼哈顿实验（原文结果）**：用曼哈顿岛的扫描图做输入——最老的城区用 **Basic**（无模式）、较新城区用 **New York** 规则；局部约束让高速公路**沿岛海岸线转向**；**"生成系统提出的桥位与真实桥梁位置非常接近"**——早期证据：规则系统能自发逼近真实城市的组织逻辑。

## Key Contribution

1. **第一个从零生成完整大城市的系统**（roadmap → lots → buildings → textures 全链）；
2. **扩展 L-system**（ideal successor + globalGoals + localConstraints）——"参数外移"的可扩展性设计；
3. **self-sensitive L-system**——生长语法生成网络拓扑；
4. **layered grids**——半程序化立面的区间组/层数据结构；
5. **LOD 的语法级方案**（细节层次 = 迭代深度）；
6. **工程结果**：13,000 建筑（路网 <10s / 建筑 ~10min，实时平台 Division dvreality 5.0 展示）、26,000 建筑（Maya 3.0 离线渲染）。

## Why It Works

- **分离"生成结构"与"决策参数"**：两套逻辑（结构规则 / 目标与约束）可以各自演化——规则稳定、策略灵活；
- **让语法"自感知"**：城市的本质是网络不是树，查询-修改机制补上了 L-system 缺失的那一维；
- **半程序化纹理**："能生成的生成（网格结构），生成不了的扫描（砖/窗细节）"——在 2001 年内存条件下，这是唯一兼顾真实感与可行性的路线。

## Limitations（均为原文自述）

1. **建筑只有几何外观，没有功能**："the functionality of the buildings can not be represented using only these simple rules"——每栋建筑是挤出的体块 + 模板屋顶，**且体块互不感知**（这一问题由 2006 CGA shape 的 occlusion 查询解决）；
2. **每个 style texture 必须手工定义**："each style texture has to be defined manually"——半程序化的"半"卡在人工；
3. **演示软件不支持动态纹理生成**（实时版用了常规纹理集）；
4. **没有交通流模拟**（因此路网容量含义被显式忽略）。

## Game Development Relevance

- **城市/开放世界的生成原型**：四级流水线 = 今天"地形 → 路网 → 地块 → 建筑"标准生成链的源头；
- **"参数外移"的可迁移判据（本文最值钱的一条）**：**任何规则/生成系统，先问"哪些部分该从规则里移出去"**——因为"每加一个约束就要改规则"是规则系统扩展性的头号杀手。今天的对应物：UE PCG graph 把"结构（图）"与"参数（表/噪声/mask）"分离、Houdini 把规则与参数分成节点与属性；
- **约束三档处置**（拒绝 / 修正 / 允许+标记+替换）→ 大世界生成里的"软硬约束分级"；
- **成本账本**：13K 建筑 ≈ 10 分钟（2001 硬件）——**"database amplification" 的另一面是"生成时间成本"**，离线生成量与运行时负载是两本账（与 [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]] 同一条账本的 25 年后版本）；
- **LOD = 语法迭代深度**：给"生成式 LOD"提供了一个比"手工 LOD 链"更系统的框架。

## Unreal Engine Relevance

- **不强映射**（2001 年的系统早于任何现代引擎），但两条结构对应清晰：
  - **流水线序 = 图依赖序**：roadmap → lots → buildings 就是 UE PCG Framework 里 node graph 的串联依赖；"同一数据在每级被变换、被下一级消费"的模型完全一致；
  - **参数与结构分离**：UE PCG 的"graph 管结构 + 参数表/属性管数值"是同一设计哲学的现代形态——**本文可以给这个设计提供"为什么这样做"的历史论证**。

## Technology Evolution

```text
1975  Stiny《Shape Grammars》（建筑学，人工推导）
1984  Smith：L-system 的 "database amplification"（少量规则 → 大量数据）
1990  Prusinkiewicz & Lindenmayer：L-system 植物建模
1996  Mech & Prusinkiewicz：Open L-system（环境交互）
        ↓
★ 2001  本文 —— 城市级 L-system 扩展（参数外移 + 自敏感网络）
        ↓
2003  [[Wonka — Instant Architecture (2003)]]（立面 split 规则 + 双重控制；补细节侧）✅ 2026-10-02 入库
        ↓
2006  Müller et al.《Procedural Modeling of Buildings》——同一作者团队
      CGA shape 合并两线；把本文的"简单体块"批评为需要解决的对象
        ↓
2007–2011  Procedural Inc. → CityEngine 商用 → Esri 收购
        ↓
2010s+  Houdini / UE PCG Framework 成为程序化管线主力
        ↓
2026  ProxyBuild（LLM 推断 + 检索组装）——回应"规则编写费力"的另一种答法
```

## Relationships

### Based On

- **L-system 植物建模**（Prusinkiewicz & Lindenmayer）——生长语法的形式与龟形解释；
- **Open L-system**（Mech & Prusinkiewicz 1996）——环境交互机制（streets 降低周边人口密度）；
- **Hart 1992**（"decreasing apices" L-system 受限类）——建筑 LOD 方案的理论依据；
- **交通研究**（Fuesser 1997；Focas 1998）——双类型道路与四城输入数据。

### Extends

- **[[Procedural Content Generation]]** 的谱系——把 L-system 从"自然形态"扩展到"城市系统"。

### Followed By

- **[[Müller — Procedural Modeling of Buildings (2006)]]** ——**本文 *Future Work* 的直接兑现**：
  - 本文原文 *"Buildings generation: ... dividing the space of a house into functional units and combining them to generate new buildings by means of a similar production system as used for the road creation"* → **2006 的 CGA shape 就是这件事**；
  - 本文 *"Visualization: ... assisting the user to find grid structures and facade elements through methods of computer vision and automatically creating the texture style shader"* → 2006/2007 的立面建模线 + 2026 的自动推断线；
  - 关系是**批评性接力**：2006 明确指出本文的建筑是"简单体块 + shader 假装细节"、且**体块互不感知**（窗/门会被相邻体块切开）——2006 的 occlusion 查询解决的正是本文留下的问题。
- **CityEngine 产品线**（Procedural Inc. 2007 → Esri 2011）：论文里的学术原型系统名 "CityEngine" 一路成为商业产品名（规则语言至今叫 CGA）。

### Related

- **[[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]]** —— 相隔 25 年的**同题问答**：本文 *"each style texture has to be defined manually"* / *"rule authoring"* 的人工瓶颈，在 2026 由"LLM 解析 + 网络推断结构角色 + 检索组装"作答——**"不写规则，推断规则的作用对象"**；
- **[[Wonka — Instant Architecture (2003)]]**——同代工作，负责立面细节侧（split 规则 + control grammar）；✅ 2026-10-02 入库；
- **Alexander《A Pattern Language》**——本文批评其"非形式化"，但"模式"思想与后面的 pattern rules（New York/Paris/…) 一脉相承；
- **[[Reeves — Particle Systems (1983)]]** ——**同族对照**：都是"少量规则/参数 → 大量数据"的生成器（database amplification），且都在论文里留下了**成本账本**（Reeves：粒子数上限；本文：13K 建筑 ≈ 10 分钟）。

## Personal Knowledge State

- `user_level: Normal`（与 [[Procedural Content Generation]] 域一致）
- 无 PBR/毛发线式的"自测收口"需求；**价值在三条可迁移抽象**（见下）

## Learning Value

**三条可迁移抽象（不需要读全文）**：

1. **"规则只描述结构，参数由外部策略决定"**（ideal successor 机制）——判据：*任何规则系统，先问"哪些部分该从规则里移出去"*；
2. **"让约束'有条件通过'：允许 + 标记 + 后期替换"**（highway 跨水 → 桥/隧道）——约束处理不是二值（拒绝/接受），而是三档；
3. **"细节层次 = 迭代深度"**（decreasing apices）——LOD 可以是生成语法的副产品，而不是独立的资产链。

## Visualization

![[程序化城市_Parish-Muller 2001 扩展 L-system 图解.html]]

## Notes

- **原文核对**：2026-09-27 完成逐节核对（PDF 8 页全文抽取，44.9K 字符）；下载路径经多轮探测：**Berkeley CS285 课程镜像**（`people.eecs.berkeley.edu/~sequin/CS285/PAPERS/Parish_Muller01.pdf`）——"**老论文先找大学课程镜像**"路径再次生效（自 Kajiya-Kay 1989 起第 4 次）；备用镜像 `ii.uni.wroc.pl/~anl/dyd/seminarium/2001_zima/parish.pdf` 未使用；
- 作者单位（原文）：Yoav I H Parish（ETH Zürich）、Pascal Müller（Central Pictures, Switzerland）——**注意 Müller 当时不在 ETH**（2006 论文时他已在 ETH）；
- 论文中的 "CityEngine" 为**学术原型**；商业 CityEngine（Procedural Inc. → Esri）是其后续演进（公开事实，细节待需要时核实）。
