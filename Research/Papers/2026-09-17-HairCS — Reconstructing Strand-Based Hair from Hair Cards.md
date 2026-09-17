---
type: paper
title: "HairCS: Reconstructing Strand-Based Hair from Hair Cards"
authors: [Zixuan Lu, Tongtong Wang, Yuefan Shen, Zhongtian Zheng, Chenfanfu Jiang, Yin Yang, Kui Wu]
year: 2026
published: "2026-09-15"
venue: "arXiv:2609.16465 (cs.GR)"
url: "https://arxiv.org/abs/2609.16465"
code: ""
project_page: ""
category: [hair, character-rendering, asset-pipeline, simulation]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: Research
user_level: Normal
status: unread
aliases: [HairCS]
tags: [hair, character, asset-pipeline, lod]
---

# HairCS: Reconstructing Strand-Based Hair from Hair Cards

## TL;DR

把游戏工业里最普遍的**发片（hair cards）资产自动转成发丝（strand-based）资产**，并且输出满足三条生产硬约束：发根长在头皮上、发根分布均匀、发量填得合理。

一句话价值：**它给"发片 → 发丝"这条 LOD 阶梯补了一部上行电梯。** 过去要么重新做一版发丝，要么永远停在发片档；现在存量发片资产本身可以当源。

> ⚠️ 本文只有 arXiv 摘要与篇幅信息（22 页 / 30 图 / 5 表），**未读全文**（无 HTML 全文版本）。下面方法部分以摘要为准，凡属推断的地方我都标了。

## Problem

实时毛发有两套表示，各自有明确适用档位：

| 表示 | 构成 | 优点 | 致命缺点 |
|---|---|---|---|
| **Hair cards（发片）** | 若干带 alpha 贴图的三角形/四边形条带 | 极廉价，光栅化友好，美术可控，移动端可用 | 没有发丝级几何；轮廓和剪影靠贴图骗；**不能被物理仿真驱动**；不能套用 grooming 修改器；各向异性高光只能烘 |
| **Strand-based（发丝）** | 显式曲线几何（成千上万根 guide/strand） | 可发丝级渲染、可物理仿真、可套 clumping/curling/noise 等修改器、轮廓真实 | 制作成本高一个量级；运行时几何吞吐与 OverDraw 压力大；低端档通常直接禁掉 |

工业现实是：**大量项目有一整库发片资产，但没有对应的发丝版本。** 发片是为了性能做的表示，一旦验收标准提高（影视化过场、PC_High 档、虚拟拍摄），就得重做——这是纯重复劳动。

## Historical Context

毛发渲染的模型侧早在 1989 年就有 Kajiya-Kay 各向异性经验模型，2003 年 Marschner 等人给出基于 R / TT / TRT 三 lobe 的物理模型（见 [[Hair Rendering]]）。但**资产侧**一直缺自动化：发片靠美术手摆，发丝靠 grooming 师一根根梳。近年的神经方法大多在做"从图片/扫描重建发型"，输入不是游戏里已有的资产格式。

本文把问题重设为：**输入不是照片，是发片本身**——即用低档表示当来源，去生成高档表示。这个方向选择是它真正的原创点。

## Previous Work

- 发片生成/优化：主流做法仍是美术手工或半自动摆片，研究侧关注贴图层面的 alpha 与剪影优化；
- 发丝重建：从多视角图像、点云、单图重建发型（需要采集数据，与游戏流水线脱节）；
- 本文差异：**不需要任何额外采集**，输入就是项目里现成的发片，输出直接进现有的发丝渲染/仿真/grooming 管线。

## Core Idea

一个自动化流水线，输入"一组带贴图的三角形/四边形条带"，输出"strand-based 发型"，同时满足三件事：

1. **保真**：保留原发片的外观与发型；
2. **增细**：补足发片没有的细尺度几何细节（发片只有低频轮廓，高频细节全在贴图里，转发丝时必须补出来，否则发丝版会比原版更"秃"）；
3. **合规**：满足发丝资产的生产硬约束——
   - 发丝**必须从头皮长出**（strands originate from the scalp）；
   - **发根分布均匀**（roots uniformly distributed）——否则会出现斑秃或局部过密；
   - **发量填充合理**（hair volume plausibly filled）——从表面片到体积填充，这是"片 → 丝"最难的一步：发片只描述一层壳，发丝要填满一个体积。

第 3 条是本文的技术核心，也是判断它能不能用的关键：**如果只做外观重建而不满足这三条，产出的发丝在仿真和修改器下会立刻穿帮。**

## Technical Approach

（摘要级）流水线 → 发丝表示，输出直接兼容三类下游：

- **strand-based rendering**（发丝渲染）
- **physics-based simulation**（物理仿真）
- **grooming modifiers**：clumping（成簇）、curling（卷曲）、noise（噪声）——这些是 grooming 师的手艺工具，能不能套用决定了美术是否愿意接手

**数据集**：`https://huggingface.co/datasets/HairCS2027/HairCS`

**验证覆盖**：短发、长发、卷发，以及**丸子头、马尾**这类复杂发型。最后这一项有实际意义——丸子头和马尾是发片最难表达的发型（发片是悬空的片，很难描述"扎起来"的拓扑），能覆盖说明不是只挑了好做的样例。

## Key Contribution

1. 首次把"发片 → 发丝"作为一个可自动化的资产转换问题提出并给出可行流水线；
2. 输出**不是孤立的新格式**，而是直接落在现有发丝渲染/仿真/grooming 生态里（这一条决定了它对工业有没有意义）；
3. 显式把三条生产约束（头皮发源、发根均匀、体积填充）作为方法的一部分而不是后处理。

## Why It Works

关键在于它**改变了问题的输入域**：不做"从零生成发型"，而做"从已有的合法低档表示升档"。低档表示已经编码了美术意图（发型、走向、大致轮廓），模型只需要补出高频与体积，而不是凭空猜一个发型。**这是"升维"而不是"生成"**——和本周反复出现的"生成模型管语义、确定性/已有资产管约束"是同一思路（参见 [[Magpie — Real-Time World Renderer for Interactive Games]]、[[2026-09-16-ESG — Generating Physically Consistent Dynamic 3D Scenes from Text]]）。

## Limitations

- **无量化性能数据**：摘要没给转换耗时、输出发丝根数、与美术手工版的对比评分。判断可用性还缺关键一环：**输出的发丝数是否落在实时预算内**（实时发丝通常是万级到十万级 guide，影视是十万级以上；自动化工具很容易产出远超实时预算的根数）。
- **未开源代码**，只放了数据集。
- 自动化产出能否达到 hero 角色（主角特写）的美术标准，存疑。更现实的定位是**配角/群演/NPC**，而不是主角。
- 发片本身的缺陷会被继承：原发片如果剪影就有问题，升档后未必更好。
- 未提 LOD 链：转出来的发丝怎么再降回发片档（用于低配），是另一半问题，本文没覆盖。

## Game Development Relevance

**直接相关度：4/5。理由不是技术多先进，而是它正好落在"分档"这件事上。**

你做五档画质（PC_High / PC_Low / Android_High / Android_Mid / Android_Low）与 SABC 分级，本质工作就是**为同一个美术意图维护多档表示**。毛发是全角色最贵的一项，也是分档差异最剧烈的一项：

```
PC_High    → 发丝（可仿真，可 grooming）
PC_Low     → 发丝降根数 / 或发片
Android_*  → 发片，甚至合并成壳
```

过去的痛点是：**这条阶梯只能往下走，不能往上走。** 为了性能先做了发片，就等于放弃了发丝档。HairCS 让"先发片、后升档"成为一条可行路径——这对"先保低端再补高端"的移动端优先项目尤其对口。

## Unreal Engine Relevance

- **UE Groom**（`.grooom` 资产 + Groom 组件）就是 strand-based 路径，支持 Niagara 驱动的物理与风场；发片则是普通 Static/Skeletal Mesh + masked 材质。两者在 UE 里是**两套完全不同的资产与渲染路径**，长期以来无法互转。
- 若该流水线可用，落点是：**批量把存量发片资产转成 Groom 资产**，再用 UE 的 LOD / 根数缩减 / 卡片化（UE 有 strands→cards 的烘焙思路）反向生成低配档——**一次升档，两端受益**。
- 注意：UE 的 Groom 在移动端支持有限，Android 三档大概率仍走发片。所以这条线的现实收益集中在 **PC_High / PC_Low**。
- 交叉信号：NVIDIA DLSS 5 官方 FAQ 明确把 **hair** 列为神经渲染要增强的对象之一（"micro-realism to complex objects such as eyes and hair"，见 [[DLSS 5 — Generative Neural Rendering]]）。**毛发正在被两条路径同时攻击：资产侧升档（HairCS）与画面侧神经增强（DLSS 5）。**

## Technology Evolution

```text
Kajiya-Kay 1989（各向异性经验模型）
    ↓
Marschner 2003（R/TT/TRT 物理模型）
    ↓
发片成为实时默认（性能妥协）
    ↓
UE Groom / TressFX / HairWorks（发丝进引擎，但只服务高端档）
    ↓
★ HairCS 2026 —— 发片 ⇄ 发丝 的自动升档，阶梯首次可上可下
```

## Relationships

### Based On

- 发片（hair cards）与发丝（strand-based hair）两套既有工业表示——见 [[Hair Rendering]]

### Related

- [[Hair Rendering]] — 本文是这一概念下"资产侧"的具体工作
- [[Scalability and Quality Tiers]] — 与你五档画质直接同构：同一美术意图的多档表示
- [[Real-Time VFX Performance Budgeting]] — OverDraw 是毛发最贵的成本项，与你的 OverDraw 预算同源
- [[Physically Based Rendering]] — 发丝 shading 用的是各向异性 BRDF，非标准微面模型（见 [[Hair Rendering]]）

### Contrasts

- 与"从图像/扫描重建发型"路线对比：输入不同（已有资产 vs 新采集数据），落地成本差一个量级

## Personal Knowledge State

- **user_level: Normal**。你显然知道发片与发丝的存在与代价（这是角色 VFX 的常识），但"两者之间能否自动转换"是新的。
- 不需要为这篇去学任何新理论——它的方法论价值在**资产管线决策**，不在数学。

## Learning Value

**今天从这篇拿走的应该只有一句话**：

> 分档阶梯是可以往上走的。先做低档不等于永远放弃高档。

把这句话翻译成你的工作语境：**当某个性能预算逼你砍掉一个高档特性时，先问一句"这个高档表示能不能从低档资产自动升上来/自动降下去"**，再决定是砍还是延后。这比直接砍更省未来的返工。

## Visualization

无（本文为管线类工作，图在论文内）。

## Notes

- 待跟进：是否开源代码、是否给出输出发丝根数与转换耗时、是否投 venue。
- 待判断：输出发丝数是否落在实时预算内——**这是它对你是否有用的唯一硬门槛**。
- 相关候选经典（[[Hair Rendering]] 的学习缺口）：Kajiya-Kay 1989（各向异性经验模型）、Marschner 2003（发丝物理模型，R/TT/TRT）。
