---
type: concept
user_level: Normal
aliases: [Hair, 毛发渲染, Hair Cards, Strand-Based Hair, Groom]
prerequisites: [BRDF, Physically Based Rendering, Scalability and Quality Tiers]
first_introduced: "Kajiya-Kay 1989（各向异性经验模型）；Marschner et al. 2003（物理模型）"
---

# Hair Rendering

> 建立于 2026-09-17。这是渲染侧**最后一个明显的结构性缺口**：你的 VFX / 分档 / 角色工作天天碰到毛发，但库里此前没有任何毛发节点。

## Definition

毛发的实时渲染，本质是**在"每根头发都是一根细长散射体"这一物理事实**与"每帧只有几毫秒"这一预算事实之间找折中。

它不是一个 shader 技巧，而是**资产表示 + 着色模型 + 分档策略**三层绑在一起的问题。这三层任选其一都会影响另外两层，这是毛发比其他材质难做的根本原因。

## Core Principle

### 第一层：资产表示（决定了一切）

| 表示 | 几何 | 谁在用它 | 成本特征 |
|---|---|---|---|
| **Hair cards（发片）** | 若干带 alpha 贴图的三角/四边形条带 | 绝大多数游戏，含全部移动端 | 几何极少；**OverDraw 与 alpha 混合是主要成本**；无 early-Z，排序敏感 |
| **Strand-based（发丝）** | 显式曲线几何（guide + 插值 strand） | PC 高端档、影视、UE Groom | 几何吞吐大；可仿真、可 grooming；**OverDraw 依然高** |
| **Mesh / cap（壳）** | 整块头皮外壳 + 毛发贴图 | 远景、低配、群演 | 最便宜；只能远看 |

**关键事实：发片和发丝在游戏工业里长期是两套完全独立的资产与渲染路径，不能互转。** 直到 [[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]] 出现，才有了自动升档的可能。

### 第二层：着色模型（与你正在学的 PBR 有明确分歧）

这是本概念最需要你注意的一点：**毛发不用你正在学的标准微面 BRDF。**

| 模型 | 年份 | 性质 | 说明 |
|---|---|---|---|
| **Kajiya-Kay** | 1989 | 经验 | 把头发当作**圆柱**处理：切向各向异性高光。极便宜，至今仍是大量游戏的默认 |
| **Marschner et al.** | 2003 | 物理 | 把头发当作**半透明圆柱**，拆出三个 lobe：**R**（直接反射）、**TT**（透射-透射）、**TRT**（透射-反射-透射，产生次级高光与"发丝透光"） |
| 后续 | 2008+ | 工程化 | 双散射近似、能量守恒修正、实时近似 |

**为什么不用 [[Microfacet Theory]]？** 因为微面模型假设表面由**面**组成，而毛发是**细长散射体**——高光沿发丝方向被拉成一条带（各向异性），并且光会**穿过**发丝再出来（透射）。这两个现象微面模型的 D·G·F 三因子都描述不了。

> 这条对你有直接价值：**你正在系统学 Cook-Torrance 的 D·G·F，而毛发恰好是这套框架失效的边界。** 知道一个理论在哪儿失效，和知道它怎么用一样重要。

### 第三层：分档（你的主场）

```
PC_High     → 发丝（可仿真、可 grooming、可风场）
PC_Low      → 发丝降根数 / 或发片
Android_H/M → 发片
Android_Low → 发片合并 / 壳
```

毛发的分档差异是角色里最剧烈的：**同一角色在最高档与最低档可能根本不是同一种资产。**

## Prerequisites

- [[BRDF]] — 毛发用的是各向异性 BRDF，不是标准微面 BRDF
- [[Physically Based Rendering]] — 毛发的"物理"是另一套物理（散射体，不是面）
- [[Scalability and Quality Tiers]] — 毛发是分档差异最大的角色资产

## Historical Evolution

```text
Kajiya-Kay 1989 —— 各向异性经验模型（圆柱假设），成为实时默认
        ↓
Marschner et al. 2003 —— R / TT / TRT 物理模型，影视级
        ↓
发片成为实时默认表示（性能妥协，与理论发展并行）
        ↓
UE Groom / TressFX / HairWorks —— 发丝进引擎，只服务高端档
        ↓
★ 2026 HairCS —— 发片 ⇄ 发丝 自动升档，阶梯首次可上可下
        ↓
DLSS 5 —— 把 hair 列为神经渲染要"增强微真实感"的对象之一
        （见 [[DLSS 5 — Generative Neural Rendering]]）
```

## Important Papers

| 论文 | 角色 |
|---|---|
| Kajiya & Kay 1989 | 各向异性经验模型，至今仍是实时默认 |
| Marschner et al. 2003 | R/TT/TRT 物理模型（**待入库，本概念最大缺口**） |
| [[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]] | 资产侧：发片 → 发丝 自动升档 |

**待补（下一批经典候选）**：Kajiya-Kay 1989、Marschner 2003。这两篇才是毛发着色的真正源头，目前库里只有 2026 的资产侧工作，属于"有尾无头"。

## Related Concepts

- [[Microfacet Theory]] — **Contrasts**：毛发是微面模型失效的边界（细长散射体 vs 面）
- [[BRDF]] — 毛发 BRDF 是各向异性 + 含透射的特例
- [[Physically Based Rendering]] — 毛发的"PBR"是另一套 PBR
- [[Real-Time VFX Performance Budgeting]] — **OverDraw 是毛发与 VFX 共同的最大成本项**
- [[Scalability and Quality Tiers]] — 毛发是分档差异最大的角色资产
- [[Niagara]] — UE Groom 的发丝物理由 Niagara 驱动

## Technologies

- **UE Groom**（`.groom` 资产 + Groom 组件 + Niagara 物理）：strand-based 路径；移动端支持有限
- **发片**：普通 Mesh + masked / alpha 材质，全平台可用
- **AMD TressFX / NVIDIA HairWorks**：历史上的发丝方案
- **MetaHuman**：UE 侧的高保真角色，含毛发资产管线

## Game Applications

- **主角**：通常发丝 + 仿真，是 PC_High 档的画质门面
- **配角 / NPC**：发片为主；若 HairCS 一类工具成熟，**配角发丝化**是最大的收益面（量大、要求低、正好匹配自动化的质量上限）
- **群演 / 远景**：壳或合并发片
- **移动端**：几乎全部发片；发丝目前不是 Android 三档的现实选项

## Personal Knowledge

- **user_level: Normal**。判断依据：你做角色 VFX 与分档预算，"发片便宜 / 发丝贵"这类结论显然在 Easy 区；但**毛发的着色模型（Kajiya-Kay / Marschner）与"发片↔发丝能否互转"**这两块是新的。
- 与你当前学习线的交汇点：**毛发是 D·G·F 框架的失效边界**——学 PBR 时把它当作反例记住，比多记一个公式有用。

## Learning Gap

1. **最大缺口：着色模型源头缺失。** 库里有 2026 的资产侧工作，却没有 Kajiya-Kay 1989 与 Marschner 2003。这导致本概念目前"有尾无头"。
2. **R / TT / TRT 三个 lobe 分别对应什么视觉现象**——不理解这个，就无法判断"为什么毛发不能直接用标准 PBR 材质"。
3. **实时毛发的真实成本结构**：发片 vs 发丝在同一角色上的 DrawCall / OverDraw / 显存对比。这一块没有公开基准，需要你自己测。

## Next Step

1. **补经典**：Marschner et al. 2003（R/TT/TRT）——这是毛发渲染的理论锚点，优先级最高；Kajiya-Kay 1989 可作为同批补充。
2. **顺手可做的一次实测**（不需要读论文）：在 NGR 里挑一个代表性角色，测同一角色在发片与发丝两种表示下的 **OverDraw 与 GPUTime**，落成两个数字。这比任何论文都更能支撑你的分档判断。
3. 跟踪 [[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]] 是否放出**输出发丝根数**——那是它对你是否有用的唯一硬门槛。

## Notes

- 2026-09-17 由 [[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]] 触发建立。
- 有趣的会合：同一天，NVIDIA DLSS 5 官方 FAQ 把 hair 列为神经渲染增强对象。**毛发正在被两条路径同时攻击：资产侧升档（HairCS）与画面侧神经增强（DLSS 5）。**
