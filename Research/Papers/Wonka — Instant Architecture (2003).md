---
type: paper
title: "Instant Architecture"
authors: [Peter Wonka, Michael Wimmer, François Sillion, William Ribarsky]
year: 2003
published: "2003-07（SIGGRAPH 2003；ACM TOG 22(3): 669–677）"
venue: "SIGGRAPH 2003 / ACM Transactions on Graphics 22(3)"
url: "https://doi.org/10.1145/882262.882324"
code: ""
project_page: "https://www.cg.tuwien.ac.at/research/publications/2003/Wonka-2003-Ins/（TU Wien 官方页；PDF 可直接下载）"
category: [procedural-generation, architecture, shape-grammar, classical]
importance: S
historical_importance: 5
game_relevance: 4
production_readiness: "Industry Adopted（思想层：CGA shape 与 CityEngine 的直接前置之一）"
user_level: "Normal"
status: unread
aliases: [Wonka 2003, Instant Architecture 2003, split grammar, split 文法, 立面生成]
tags: [classic, pcg, architecture, shape-grammar, split-grammar, rule-selection]
---

# Instant Architecture（Wonka, Wimmer, Sillion & Ribarsky 2003）

## TL;DR

**"立面细节"的自动化答案：split grammar（拆分文法）× 双重控制系统。** 建筑 = 从体块出发、被反复"拆分 / 转换"到窗台檐口级构件的一组带属性形状；**同一套规则库**（约 250 条规则 + 40 属性）生成全部风格，风格差异靠"高层属性 + 控制文法"调出来。两个关键发明：**attribute matching**（规则选择的**两级制**：硬条件过滤 + 软偏好抽样）与 **control grammar**（把"设计想法"按建筑原则**空间分布**——一层商铺、列/行一致性）。

> **一句话定位**：它第一次让"数百条规则的规则库"变得**可自动推导**——"**规则选择**（rule selection）"问题（本文自称 *"to our knowledge, the first paper to address the problem of rule selection for grammars with large rule databases"*）的出处。
>
> **谱系地位**：PCG 域**立面细节侧**的锚点——与 [[Parish-Müller — Procedural Modeling of Cities (2001)]]（城市规模）**同代互补**、被 [[Müller — Procedural Modeling of Buildings (2006)]]（CGA shape）**直接继承**；三篇是同一批人（Müller 与 Wonka 交叉合著）跨越 5 年的接力。**PCG 谱系自此四节点齐全：2001 城市 → 2003 立面 → 2006 合流 → 2026 推断。**

## Problem

城市重建 / 规划应用需要"快速生成大量不同风格的建筑"，而手工建模 *"labor intensive"*（原文开篇）。既有工具两头不靠：

- **L-system（植物、街道）不适合建筑**——原文判定句：*"a building is not designed with a growth-like process, but a sequence of partitioning steps"*（**建筑不是"长"出来的，是"切"出来的**）；
- **Stiny 形状文法**能描述建筑风格，但**推导靠人**（*"a human deciding on the rules to apply"*）→ 无法快速大范围建模。

核心矛盾：**规则库越大，随机选择越混乱**（原文 Figure 2 左→右：小规则库已"mildly chaotic"，规则越多 chaos 越盛）。把全部设计选择编码进规则本身在组合复杂度上不可行——**"自动推导"必须解决"选哪条规则"**。

## Historical Context

```text
1975  Stiny《Shape Grammars》：形状文法（建筑学，人工推导）
1980  Stiny：labeled shape / parameterized shape 形式定义（本文直接沿用）
1982  Stiny：set grammar（把形状当符号对象，免去困难的子形状匹配）
        ↓
1991  L-system 植物（并行文法，"生长"隐喻）
2001  Parish & Müller：城市（扩展 L-system，简单体块 + shader 假装细节）
        ↓
★ 2003 本文：立面（split grammar + 双重控制）——"规则选择"问题首次提出
        ↓
2006  Müller, Wonka et al.：CGA shape 合流（复杂体块 + 一致细节）
        ↓
2007–2011 Procedural Inc. → CityEngine 商用 → Esri   2010s+ 游戏/影视管线 + Houdini
        ↓
2026  ProxyBuild（LLM 推断 + 检索组装）——回应"规则编写费力"
```

与 Parish-Müller 2001 的分工（本文 Discussion 原话对位）：*"While their approach aims at quickly generating a large number of simple, yet diverse buildings, the focus of this work lies on producing complex geometric representations of individual buildings."*（**一个求"多而简"，一个求"少而精"**——立面细节是 2001 主动留在 shader 里没做的那一半。）

## Core Idea — 三个贡献（原文自述）

1. **split grammar**——受限的 set grammar。限制是**刻意设计**的：*"powerful enough for the modeling of buildings, but simple enough to allow a controlled and automatic derivation"*。两类规则：
   - **split rule**：把一个 basic shape 拆成一组子形状（如 cuboid → n×m×k 网格，分割面参数化）；**子形状永远包含在父形状体积内**（这条"包含性"直接解决 L-system 式"互相生长穿透"问题——原文 Growth Control 段：建筑里"物体长进彼此"不可容忍）；
   - **conversion rule**：把一个 basic shape 换成另一个（**不要求填满体积**——如 cuboid → 三棱柱做屋顶）。
   - **固定访问序**：推导中访问子形状的顺序预先固定（deterministic visit order）。
2. **attribute matching**——形状与规则**都带属性**；规则选择分**两级**：
   - **确定性匹配 MDV**（硬条件）：逐属性做区间重叠测试 + containment 标志 + 优先级累加；任一属性失败 → 该规则得 −∞ 被剔除；
   - **随机选择 MSV**（软偏好）：候选集内按符号上的统计分布 fSD（在规则区间中点取值）连乘，再乘**每条规则预计算的随机数 pr**——pr 每条建筑只算一次，**保证"同一次推导里选规则的一致性"**（符号相同 ⇒ 选择相同）。
3. **control grammar**——一个"非常简单的带属性上下文无关文法"，专门管**设计想法的空间分布**。终点 = 属性改写命令 ⟨c, a, v⟩（c = 空间定位/作用域：行列层号、all/first/last；a = 属性名；v = 值）。每次 split 时被调用，把属性"按建筑原则"写进新形状——**一层商铺、第 1/3 列竖向强调、第 2/3 层横向强调**都是这样写出来的。

### 关键机制细节

- **basic shape** = ⟨简单凸体（中心在原点）+ 3 个带标签点（正坐标轴与面交点）+ 符号与属性⟩；示例库 10 种形状（cuboid / cylinder / prism 三种定义了 split 规则）；
- **"排除默认"技巧**：containment 标志 `cr=true` 使规则只在"符号区间被包含于规则区间"时可用——默认区间 [−∞,∞] 永远不满足 ⇒ **盲窗（blind window，纯装饰赝窗）只在被显式请求时出现**（"特殊行为默认关闭、必须显式开启"的 2003 版本）；
- **三个控制层级**（给不同角色）：改文法（最强，设计师 + 建筑师）→ 改初始形状属性（style / age / use，可自动按分布抽样）→ 改规则属性（细粒度微调）；
- **属性双重职责**：既编码/传播材质信息（大粒度 → 小粒度推理：style → 具体颜色），又**驱动规则选择**（属性匹配的输入）——一套机制同时管"长相"和"怎么长"。

## Key Data（逐节核对，均为原文数字）

| 项目 | 数据 |
|---|---|
| 规则库 | 约 **250 条规则、40 个属性**；10 种 basic shapes（3 种有 split 规则） |
| 规则库制作 | 约 **2 周**（素材：Mitchell 1990 + Moser 1985 + 作者团队对伦敦/巴黎/亚特兰大/维也纳建筑的实地分析） |
| 生成速度 | 单建筑 **1,000–100,000 多边形，1–3 秒**（Intel Pentium 4, 2 GHz）；截图截自**实时渲染引擎**的交互查看 |
| 规模外推（原文估计） | 真实城市规划应用需要 **2,000–3,000 条规则** |
| 输入接口 | ArcView（GIS：建筑 footprint + 属性）或命令行（`wall material=15, shop=1`） |
| 迭代细化（Fig 10） | 无属性 → +"shop + color variation" → +"第 1/3 列竖向强调 + 第 2/3 层横向强调" |

## Why It Works

- **限制即力量**：把"什么不能做"写进文法（子形状内含于父、基本形状限定凸体、固定访问序）→ 换来自动推导的**可靠性**；其余自由度全部转移到**属性**与**控制系统**上；
- **"拆出去"三步**：设计想法从文法里 factor out（control grammar）、选择策略从规则内容里拆出去（attribute matching）、参数由外部策略决定（继承 2001 的 ideal successor）——**"规则只管结构"这件事被推到极致**；
- **两级匹配 = 可行性 + 偏好**：先硬条件剔除不可行，再按统计分布抽偏好——与"先保证、后质量"的一般工程模式同构（且随机数的**预计算**解决了"随机性与一致性"的表面矛盾：随机一次，全程复用）。

## Limitations（原文自述）

- 复杂细节（科林斯柱头级）应由**外部建模包**做完、当终端形状导入；哥特窗等复杂构型不支持；
- 修改规则 *"by no means trivial"*——预期角色分工：设计师 + 建筑师造文法，普通用户只改属性；
- 规则库规模有限（250 条），真实应用需 2,000–3,000 条（**未给出规模化成本方案**）；
- **LOD / 实时化未涉及**（后被 Müller 2006 与后续 LOD 研究填补；当年 Survey 即指出 LOD 是实时应用的必要条件）。

## Game Development Relevance

- **城市场景管线的"另一半"**：2001 给城市骨架、本文给立面细节、2006 合流为 CGA/CityEngine——今天 UE PCG / Houdini 管线里"楼体生成 + 立面规则"仍是同一种分工；
- **"2 周规则库 → 无限建筑变体"**：设计资产复用（规则 > 几何）的最早完整样本之一（对照《超人归来》15 人年）；
- **属性推理链**（style → material → color）提示"**档位参数也可以分层定义、逐级推理**"——与 2001 的"参数外移"、2006 的"绝对/相对"同一条演化线（这对预算矩阵的"高层目标 → 低层数值"组织方式有直接参考）。

## Unreal Engine Relevance

- ⚠️ 无直接引擎映射（2003 年远早于 UE PCG 等系统）。结构对应：control grammar ≈ 生成后处理里"**按行列/层批量写属性**"的节点（作用域命令）；attribute matching ≈ 规则表驱动的选择器；split/conversion ≈ 网格切割与替换类节点。

## Technology Evolution

```text
1975 Stiny 形状文法（人工推导）
  ↓
2001 [[Parish-Müller — Procedural Modeling of Cities (2001)]]：城市（L-system，简单体块）
2003 ★ 本文：立面（split grammar + 双重控制）——"规则选择"首次被作为问题提出
  ↓
2006 [[Müller — Procedural Modeling of Buildings (2006)]]：CGA shape 合流（复杂体块 + 一致细节；
     Müller 与 Wonka 交叉合著）
  ↓
2007+ CityEngine（→ Esri） 2010s+ 游戏/影视管线 + Houdini 生态
  ↓
2026 [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]]：
     LLM 推断 + 检索（"不写规则"线）——本文 Future Work"用图像训练文法"的 23 年回响
```

## Relationships

### Based On

- **Stiny 1975 / 1980 / 1982**（记名）：shape / labeled shape / set grammar 的形式定义直接沿用（"the appeal of shape as a formal concept is that spatial relations and dependencies can be consistently encoded in the grammar itself"）。

### Related（同代互补 / 对位）

- [[Parish-Müller — Procedural Modeling of Cities (2001)]]：同一问题空间的两半——**城市骨架 vs 立面细节**；两种文法形态（L-system vs split grammar）；本文 Discussion 明确对位（"多而简" vs "少而精"）。

### Followed By

- [[Müller — Procedural Modeling of Buildings (2006)]]：**直接继承**——原文自述 *"我们基于 split 规则的思想构建"*；作者交叉（Müller：2001 二作 → 2006 一作；Wonka：2003 一作 → 2006 二作）；2006 并用 occlusion 查询回应本文"只对简单体块有效"的边界。

### Enabled

- **CityEngine / CGA shape 的直接前置之一**（"split"概念进入 CGA 语法）。

### Links（前沿）

- [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]]（记名级关联）：回应同一个"规则编写费力"瓶颈；且本文 Future Work 写着 *"We see the most potential for future enhancements in the use of images as training data for creating and improving the grammar"*——**"从数据里学文法"的最早表态**。

## Personal Knowledge State

- `user_level: Normal`（与 Müller 2006 同级；形状文法形式化细节不要求）
- **三条可带走的判据**（不需要读全文）：
  1. **"规则选择"正名**：规则库越大，越需要把"怎么选"从"规则内容"里**拆出来独立成模块**——判据：*约束越多，检查"选择策略"是否已独立于"规则本身"*（对今天的 AI 工具参数化、档位选择同构）；
  2. **硬条件与软偏好分两级**：先"能不能用"（区间重叠 + containment），后"多想要"（分布抽样）——任何"从很多候选中挑"的系统都适用；
  3. **"排除默认"**：默认值覆盖不了特殊选项——**危险 / 特殊行为必须显式开启**（盲窗机制的 2003 版本）。

## Learning Value

- **PCG 谱系补全**：四节点齐（2001 城市 / **2003 立面** / 2006 合流 / 2026 推断）——立面细节侧的缺失被填上，且"2001↔2003↔2006"是**同一批人的接力**（比"同域不同团队"的谱系更强的一条血缘）；
- **与 2001 的对照**（同日入库时的"拆出去三步曲"）：
  - 2001：**参数**从规则里移出去（ideal successor / globalGoals）；
  - 2003：**设计想法**从规则里分解出去（control grammar）+ **选择策略**从规则里拆出去（attribute matching）；
  - 2006：**表面**从"直接写规则"移出去（两阶段：体块 → 表面 scope → 细节）——**"往中间表示上退一步"的三连**。

## Visualization

![[程序化立面_Wonka 2003 split grammar 与规则选择图解.html]]

## Notes

- **来源核对**：原文 PDF 已下载并逐节核对（9 页；TU Wien 计算机图形学组官方页 `cg.tuwien.ac.at`，`Wonka-2003-Ins-Paper.pdf`，14.6 MB 高清版）。
- **PDF 获取路径记录**：HAL（inria.hal.science）SSL 受限 → 换 **TU Wien 官方归档**成功（unverified SSL 上下文 + 直接命中 `-Paper.pdf`）——"**机构 / 作者组归档**"路径第 7 次生效；Wayback 429 的老问题当日未再纠缠（3 次失败即转替代路径）。
- 年代注：两级匹配公式配合原文 Figure 8（WIN 符号 6 条规则 → 4–6 号被剔除）读最清晰：三种剔除机制（区间不交 / 单向包含 / 排除默认）在图中各有示例。
