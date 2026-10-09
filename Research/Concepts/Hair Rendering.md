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
| **Kajiya-Kay** | 1989 | 经验 | 把头发当作**圆柱**处理：切向各向异性高光。极便宜，至今仍是大量游戏的默认 → ★ **2026-09-25 已入库**：[[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]]（含 texel 三要素、$\sin(t,l)$ 漫反射与圆锥高光的推导、与 Reeves 粒子系统的"对偶"关系） |
| **Marschner et al.** | 2003 | 物理 | 把头发当作**半透明圆柱**，拆出三个 lobe：**R**（直接反射，白高光）、**TT**（透射-透射，逆光亮边）、**TRT**（透射-反射-透射，有色次级高光 + 发丝透光）→ ★ **2026-09-18 已入库**：[[Marschner — Light Scattering from Human Hair Fibers (2003)]] |
| **双散射（Dual Scattering）** | 2008 | 物理近似 | **多次散射的实时近似**：全局（穿过体积到达邻域，沿单条原型路径统计）+ 局部（邻域内回散射 → 材质属性）→ ★ **2026-09-28 已入库**：[[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]]——**单散射只对深色发够用；浅色发的发色由多散射主导** |
| **能量守恒模型（d'Eon）** | 2011 | 物理（守恒） | **Marschner 的修正版**：球面高斯卷积重建守恒 M_p（非高斯、off-specular peak）+ 方位向取消求根改积分 → ★ **2026-10-07 已入库**：[[d'Eon — An Energy-Conserving Hair Reflectance Model (2011)]]——**"白环境实验"= 毛发版 furnace test；双散射的高粗糙度区间由此"扶正"；pbrt-v4 采用本模型** |

**三个 lobe 对应的视觉现象（这是判断"能不能砍"的唯一依据）：**

| lobe | 画面上的样子 | 丢了会怎样 | 分档优先级 |
|---|---|---|---|
| **R** | 沿发丝的白色高光带 | 头发变成哑光布 | 必留 |
| **TT** | 逆光下的透光轮廓 | **角色轮廓光消失，头发立刻"死"** | 倒数第二砍 |
| **TRT** | 有色次级高光（宽、暗） | 近景少一层层次，远景看不出 | **最先砍**（$\beta_{TRT}=2\beta_R$） |

图解：[[Hair R·TT·TRT 三叶散射图解]]

**为什么不用 [[Microfacet Theory]]？** 因为微面模型假设表面由**面**组成，而毛发是**细长散射体**——高光沿发丝方向被拉成一条带（各向异性），并且光会**穿过**发丝再出来（透射）。这两个现象微面模型的 D·G·F 三因子都描述不了。**更根本的一条（2026-09-18 补）**：纤维散射函数的度量是**"每单位长度"**（曲线辐照度 / 曲线强度），而微面 BRDF 是**"每单位面积"**，且积分域是**整个球面**而非上半球。**单位不同，就没法塞进同一个管线**——这才是引擎必须为毛发单开一个 Shading Model 的根本原因。

> 这条对你有直接价值：**你正在系统学 Cook-Torrance 的 D·G·F，而毛发恰好是这套框架失效的边界。** 知道一个理论在哪儿失效，和知道它怎么用一样重要。

### 第三层：分档（你的主场）

```
PC_High     → 发丝（可仿真、可 grooming、可风场）
PC_Low      → 发丝降根数 / 或发片
Android_H/M → 发片
Android_Low → 发片合并 / 壳
```

毛发的分档差异是角色里最剧烈的：**同一角色在最高档与最低档可能根本不是同一种资产。**

**着色侧的降档顺序（2026-09-18 由 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 给出依据）：**

```text
PC_High        R + TT + TRT（+ 多次散射近似）
PC_Low         R + TT（砍 TRT：宽、暗、远处不可辨）
Android_High   R + 简化 TT（逆光亮边必须留 —— 它是头发"活着"的唯一来源）
Android_Mid/Low  只留 R（退化为各向异性高光带）
```

> **一个"用颜色换预算"的位置**：Marschner 的吸收系数 $\sigma_a$ 是**三通道**的。低档把发色饱和度压低时，TT 与 TRT 的视觉贡献同时下降——**"把发色做得更朴素"本身就是一种性能优化**，不只是美术选择。

## Prerequisites

- [[BRDF]] — 毛发用的是各向异性 BRDF，不是标准微面 BRDF
- [[Physically Based Rendering]] — 毛发的"物理"是另一套物理（散射体，不是面）
- [[Scalability and Quality Tiers]] — 毛发是分档差异最大的角色资产

## Historical Evolution

```text
Kajiya-Kay 1989 —— texel（密度场）+ 各向异性经验模型；"近看时切回真实几何"的分档原则（**2026-09-25 入库**）
        ↓
★ Marschner et al. 2003 —— R / TT / TRT 物理模型，影视级（**2026-09-18 入库**）
        ↓
★ Scheuermann 2004 —— **实时工程首次落地**：发片模型 + 两 lobe 移位近似 + **取消运行时排序**（静态索引缓冲 + 四趟渲染）（**2026-09-26 入库**）
        ↓
★ Zinke-Yuksel 2008 —— **多散射实时近似**：全局/局部分解 + "一条原型路径代表全部路径"；7.8h → 5.2min → 实时 14fps（**2026-09-28 入库**）
        ↓
★ d'Eon et al. 2011（Weta）—— **守恒重建**：球面高斯卷积重推 M_p + 取消求根 + 任意阶（TRRT）；"白环境实验"验证（**2026-10-07 入库**）
        ↓
★ Chiang et al. 2016（WDAS）—— **生产化一跳**：near-field 取消 70 点求积（把宽度积分外包给路径追踪器；着色 ~20× / 整体 >10×）+ logistic 替代 wrapped Gaussian + 第四叶折无穷阶（furnace test 通过）+ 感知均匀参数化（**2026-10-08 入库**；此前误记 Pixar，已更正）
        ↓
发片 + 双高光 + 固定排序成为实时默认（此后十几年游戏毛发的主流做法）
        ↓
UE Groom / TressFX / HairWorks —— 发丝进引擎，只服务高端档
        ↓
★ 2026 HairCS —— 发片 ⇄ 发丝 自动升档，阶梯首次可上可下
        ↓
DLSS 5 —— 把 hair 列为神经渲染要"增强微真实感"的对象之一
        （见 [[DLSS 5 — Generative Neural Rendering]]）
        ↓
★ 2026-09 《巫师 3》重制版（9-29）—— LSS 路径追踪毛发（HairWorks 增强）
        → 毛发成为 GPU 光追的原生曲线基元（"光追档"成为新的最高档）

【仿真轴】（2026-10-06 开线，与渲染轴独立）
1992-2025  悬臂梁 → Cosserat rod / DER → GPU 求解器（数千股"高端配置实时"）
2023-2026  神经三代：GroomGen（数据驱动）→ Quaffure（准静态）→ Neuralocks（惯性损失）
★ 2026-10 Neuroll —— 神经时间积分器（镜像经典 I/O）+ 模拟器在环 + 随机视界
        → 3000 股 0.460 ms/帧；动态帧数 2000 打满（前作 131）；密度无关
```

> **"表示跟着观察尺度走"的完整三代**：texel ↔ 几何（1989，手动）→ 发片 ↔ 发丝（2026，HairCS 可自动升档）→ 光追曲线基元（2026，LSS）。**你的毛发分档表未来会多出一档：光追毛发。**
>
> **两个轴的分工（2026-10-06 补）**：本概念此前全部内容 = **渲染轴**（"头发看起来像什么"，关键词 = 表示）；**仿真轴**（"头发怎么动"，关键词 = 积分）由 [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]] 开线——毛发在库内第一次有"怎么动"的节点。两轴可分别调档：渲染侧 = 发片/发丝/光追基元；仿真侧 = off / 准静态 / 动态（一个网络三态）+ 刚度标量。

## Important Papers

| 论文 | 角色 |
|---|---|
| [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]] | ★ 2026-09-25 入库：**实时侧经验源头**（texel + $\sin(t,l)$ 漫反射 + 圆锥高光）；"渲染时间与几何复杂度无关"与"texel↔几何切换"两条原则的出处 |
| [[Marschner — Light Scattering from Human Hair Fibers (2003)]] | ★ 2026-09-18 入库：R/TT/TRT 物理模型，**本概念的理论锚点**（含 Table 1 全套典型参数、$\alpha_{TT}=-\alpha_R/2$、$\alpha_{TRT}=-3\alpha_R/2$） |
| [[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]] | 资产侧：发片 → 发丝 自动升档 |
| [[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]] | ★ 2026-09-26 入库：**实时工程侧**——"怎么把 2003 的物理模型塞进 2004 的硬件"：发片模型 / 两 lobe 移位近似（扰动切线 + shift 贴图）/ **取消运行时排序**（静态索引缓冲 + 四趟渲染） |
| [[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]] | ★ 2026-09-28 入库：**多次散射实时近似**——全局/局部分解、原型路径（透射连乘 + 方差求和）、fback 材质属性、三档实现；"浅色发"账本的闭合项 |
| [[d'Eon — An Energy-Conserving Hair Reflectance Model (2011)]] | ★ **2026-10-07 入库（能量页）**：Weta 生产模型——**守恒 M_p**（球面高斯卷积；白环境实验验证）+ 方位向取消求根（积分 + 高斯检测器）+ 任意阶（TRRT ≈ 白发掠射 15%）；pbrt-v4 采用 |
| [[Chiang — A Practical and Controllable Hair and Fur Model for Production Path Tracing (2016)]] | ★ **2026-10-08 入库（生产化页）**：WDAS 生产模型——**near-field（宽度积分外包给路径追踪器，单次求值 O(1)）+ logistic 方位分布（可解析归一/采样）+ 第四叶（无穷高阶折几何级数闭式）**；毛发着色 ~20×、整帧 >10×（869→76 min）；感知均匀六参数；"让暴力路径追踪毛发在生产可行"（Hyperion 实装；前记 "Pixar" 系误记，已更正） |
| [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling]] | ★ **2026-10-06 入库（仿真轴首节点）**：神经时间积分器 = 镜像经典积分器 I/O（前状态+刚度+碰撞几何入、下一状态出）；模拟器在环 + 随机视界；3000 股 **0.460 ms/帧**、动态帧数 **2000**（前作 131）、密度无关扩到 12 万股；strand-space 规范化（运动比 0.632 vs 世界系 0.342） |

**六节点已齐（生产化闭合）**：经验侧（Kajiya-Kay 1989）、物理侧（Marschner 2003）、实时工程侧（Scheuermann 2004）、多散射近似（Zinke-Yuksel 2008）、**能量守恒（d'Eon 2011）**、**生产化（Chiang 2016）** 均入库。**物理、工程与生产主干全部闭合，且能量账跨材质域接通**（与表面侧 Kulla-Conty / Hammon 同题）；剩余为纯可选深挖（d'Eon 2014 non-separable）。**仿真轴 2026-10-06 开线**（Neuroll）——"头发怎么动"独立于渲染轴，另行积累。

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
- **（2026-09-25 更新）着色侧的两头现已齐备**：经验侧 [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]] + 物理侧 [[Marschner — Light Scattering from Human Hair Fibers (2003)]]，两条共 **9 条 Mastery 自测**。
- **（2026-09-26 更新）实时工程侧补齐**：[[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]] 入库，自测扩至 **12 条**（+3：两 lobe 移位机制 / "不排序"的前提假设 / early-Z 那一刀为什么值得多一趟）。
- **（2026-09-28 更新）多散射节点补齐**：[[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]] 入库，自测扩至 **15 条**（+3：单散射为何对浅色发不够 / "原型路径代表全部路径"的成立条件 / 全局局部为何像"阴影"与"材质"）——**15 条通过即可把"毛发着色（含实时工程与多散射）"整体标 Easy**。
- **（2026-10-07 更新）能量页补齐 + 用户已启动阅读**：[[d'Eon — An Energy-Conserving Hair Reflectance Model (2011)]] 入库，自测扩至 **18 条**（+3：高斯 M_p 不守恒的至少两条原因 / "白环境实验"为什么要求总反射率为 1 / d'Eon 为何取消求根、代价收益各是什么）。**用户侧信号（10-06）：Marschner 深读进行中**——自制 R/TT/TRT 光路对照 SVG、三处精确勘误（含"$A$ 不能直接当作完整散射函数 $S$"）、挂载原文 PDF；此前的 PBR 研读法（读原文→自绘图→写勘误）正在毛发线复现。**d'Eon 2011 是 Marschner 的自然下一站（能量页）**。
- **（2026-10-08 更新）生产化页补齐**：[[Chiang — A Practical and Controllable Hair and Fur Model for Production Path Tracing (2016)]] 入库，自测扩至 **21 条**（+3：积分搬家的无偏性与代价 / 第四叶代表全部高阶的两个前提 / 感知均匀参数化为何是生产采用的"决定项"）。**渲染着色六节点（1989→2016）自此全闭合**；用户侧仍处于 Marschner 深读中段——**顺序建议不变**（Marschner → d'Eon 2011 → [可选] Chiang 2016 → 21 条自测），本篇非必经。
- **（2026-09-29 更新）"实战观察"数据已出**：《巫师 3》重制版今日 18:00 上线，评测解禁数据先行——**LSS 路径追踪毛发首发**（HairWorks 的光追演进形态）；IGN 实测（RTX 5080 @4K PT + DLSS Perf）：**开 HairWorks 30 fps ↔ 关 HairWorks 44 fps**、关 PT 55 fps。**两条可用结论**：① **毛发至今仍是帧预算重项**（2015 HairWorks → 2026 LSS，十一年同题）；② **LSS 是"换表示"（观感升级），不是"降成本"**——与 LOD 原理同构（换表示层级的目的是**在同等预算下换质量**，成本量级由"每像素覆盖"决定）。
- 与你当前学习线的交汇点：**毛发是 D·G·F 框架的失效边界**——学 PBR 时把它当作反例记住，比多记一个公式有用。**同时它还是"预算与几何解耦"（Kajiya-Kay 1989）与"档位=换表示"（HairCS / LSS）的最佳案例库。**

## Learning Gap

1. ~~**最大缺口：着色模型源头缺失。**~~ ✅ **2026-09-18 已补齐**：[[Marschner — Light Scattering from Human Hair Fibers (2003)]] 入库，R/TT/TRT 的物理来源、参数表、"白高光 / 有色次级高光 / 逆光亮边"的对应关系全部到位（含 5 条 Mastery 自测）。
2. ~~**R / TT / TRT 三个 lobe 分别对应什么视觉现象**~~ ✅ 同篇补齐（见上表）。
3. ~~**实时侧源头缺失**~~ ✅ **2026-09-25 再补**：[[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]] 入库（texel 三元组、两条着色公式、"预算与几何解耦"、"texel↔几何切换"；含 4 条 Mastery 自测）。~~**剩余缺口：实时化工程文**~~ ✅ **2026-09-26 闭合**：[[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]] 入库（发片模型 / 两 lobe 移位近似 / 取消运行时排序 + early-Z 权衡；含 3 条 Mastery 自测）。~~残余仅剩 dual scattering 类多次散射近似（可选）~~ ✅ **2026-09-28 闭合**：[[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]] 入库（全局/局部分解 + 原型路径 + 三档实现；含 3 条 Mastery 自测）。**毛发着色谱系（1989 → 2003 → 2004 → 2008）全部闭合。**
4. **实时毛发的真实成本结构**：发片 vs 发丝在同一角色上的 DrawCall / OverDraw / 显存对比。这一块没有公开基准，需要你自己测。
5. **（2026-10-07）能量账的当代接口**：d'Eon 2011 的守恒 M_p 已是 pbrt-v4 实装；**UE 具体采用哪个 M_p 形式未核实**（待核实项）——若后续需要对照引擎实现，这是第一个查证点。

## Next Step

1. ~~**补实时侧经典：Kajiya-Kay 1989（优先）**~~ ✅ **2026-09-25**。~~Scheuermann 2004（TRT 实时近似）~~ ✅ **2026-09-26**——**经验 / 物理 / 实时工程三层已齐**。~~dual scattering 类近似（可选）~~ ✅ **2026-09-28**——**多散射层也齐**。~~能量守恒（可选深挖）~~ ✅ **2026-10-07**（d'Eon 2011 入库）。~~Chiang 2016（生产化）~~ ✅ **2026-10-08**（入库）——**着色谱系六节点全闭合（1989/2003/2004/2008/2011/2016）**。剩余完全可选：d'Eon 2014（non-separable）。
2. **（当前推荐路径）**：Marschner（进行中）→ **d'Eon 2011 能量页**（+30–60 分钟，读法见其笔记）→ [可选] **Chiang 2016 生产化页** → **21 条自测**；"白环境实验"可在引擎里做一次最低成本的实现体检（零吸收材质 + 均匀白环境 → 应该看不见）。
3. **顺手可做的一次实测**（不需要读论文）：在 NGR 里挑一个代表性角色，测同一角色在发片与发丝两种表示下的 **OverDraw 与 GPUTime**，落成两个数字。这比任何论文都更能支撑你的分档判断。
4. **一个可以直接试的降档实验**：把某个低档发片材质上的 TT（逆光亮边）关掉，看角色在逆光场景下的观感退化程度——**这是验证"TT 不能砍"这条判断的最快方式**。
5. 跟踪 [[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]] 是否放出**输出发丝根数**——那是它对你是否有用的唯一硬门槛。

## Notes

- 2026-09-17 由 [[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]] 触发建立。
- 有趣的会合：同一天，NVIDIA DLSS 5 官方 FAQ 把 hair 列为神经渲染增强对象。**毛发正在被两条路径同时攻击：资产侧升档（HairCS）与画面侧神经增强（DLSS 5）。**
- **2026-09-18**：理论源头补齐（[[Marschner — Light Scattering from Human Hair Fibers (2003)]]），本概念不再是"有尾无头"；同时新增"着色侧降档顺序"与"σa 用颜色换预算"两条可直接用于分档的判断。剩余缺口从"理论"转到"实时化"（Kajiya-Kay 1989 / Scheuermann 2004 / dual scattering）。
- **2026-09-25**：**实时侧源头补齐**（[[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]]）——texel 三要素、$\sin(t,l)$ 漫反射与圆锥高光、**"渲染时间与几何复杂度解耦"**（= 预算是"用表示换来的"）、**"近看时切回真实几何"**（= 分档的 1989 版本）。同日产业侧：《巫师 3》重制版（9-29）宣布 **LSS 路径追踪毛发**——"表示跟着观察尺度走"的第三代（光追曲线基元）落地。**毛发三代表示（texel / 物理 lobe / 光追基元）在同一天会齐。**
- **2026-09-26**：**实时工程缺口闭合**（[[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]]）——"发片 + 双高光 + 固定排序"这套**游戏毛发默认做法的源头文档**；同时是"**取消式优化**"的 2004 年版本（取消运行时 CPU 空间排序）。**三节点（1989 / 2003 / 2004）齐全，自测 12 条。**
- **2026-09-28**：**多散射节点闭合**（[[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]]）——"浅色发的发色由多散射主导"这一整类现象的解法；三条可迁移抽象入库（原型路径的合法性检查 / 方差可加 / 三档实现=换采样与存储）；**与 PBR 能量账本"补能量 vs 推输运"跨域同构**。**四节点（1989 / 2003 / 2004 / 2008）齐全，自测 15 条。**
- **2026-09-29**：《巫师 3》重制版上线（LSS 毛发）——**实测：HairWorks 开关 = 30 vs 44 fps（5080 4K PT）**。记录两条判断：**① 光追档正式进入商业首发**（"光追基元"这个新档位不再是研究展望）；**② "换表示 ≠ 降成本"**——LSS 把毛发搬进光追管线换来观感，但每像素覆盖率决定的成本量级不变（与 Kajiya-Kay 1989"渲染时间与几何复杂度解耦"合读：**解耦的是"几何复杂度"，不是"屏幕覆盖"**）。第 13 项"实战观察"数据点已备。
- **2026-10-06**：**仿真轴开线**（[[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]]）——毛发问题在库内第一次分成两个正交轴：**渲染 = 表示（本概念既有全部内容）**；**仿真 = 积分（Neuroll 起）**。Neuroll 与渲染侧自测（15 条）**无绑定关系**——它是 [[Neural Physics Simulation]] 域的样本，不改变"毛发着色"的 Easy 判据进度。可对照阅读点：Zinke 2008"一条原型路径代表全部路径"与 Neuroll"逐发丝独立网络"都在用**"线性可分解"**这把钥匙打开毛发问题。
- **2026-10-07**：**能量页补齐（五节点闭合）**（[[d'Eon — An Energy-Conserving Hair Reflectance Model (2011)]]）——Weta 生产模型：守恒 M_p（球面高斯卷积 + 白环境实验）+ 取消求根 + 任意阶。三条要点入库：① "**守恒是构造出来的，不是修正出来的**"；② "**白环境 → 看不见**"= 毛发版 furnace test（最便宜的守恒体检）；③ **失效区即价值区**——差异集中在浅色发 + 高粗糙度 + 掠射（也就是双散射身后的场景），**深色发 + 中粗糙的画面旧近似"够用"**（分档判断材料）。**同日用户侧信号：Marschner 深读进行中**——本篇正好是其直系下一站。**自测 15 → 18 条。**
- **2026-10-08**：**生产化页补齐（六节点闭合）**（[[Chiang — A Practical and Controllable Hair and Fur Model for Production Path Tracing (2016)]]，WDAS）——"为路径追踪重写"：near-field 取消 70 点求积（**把宽度积分外包给渲染器的采样**，单次求值 O(1)；毛发着色 ~20×、整帧 >10×：869→76 min）+ logistic 方位分布（解析归一/采样）+ 第四叶折无穷阶（furnace test 通过）+ 感知均匀参数化（六参数）。三条要点：① "**让最擅长做积分的系统去做积分**"；② "**参数化 = 生产采用的决定项**"；③ **"内层积分的三种归宿"判据**（算 / 外包 / 预集成——本篇选外包，NEF 选预集成）。**自测 18 → 21 条**；同时**更正此前 "Pixar" 误记**（本篇为 Disney 动画工作室，2016 年作者均在 WDAS）。
