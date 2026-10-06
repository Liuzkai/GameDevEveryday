---
type: paper
title: "Dual Scattering Approximation for Fast Multiple Scattering in Hair"
authors: [Arno Zinke, Cem Yuksel, Andreas Weber, John Keyser]
year: 2008
published: "2008 (ACM SIGGRAPH 2008, ACM TOG 27(3), pp. 1-10)"
venue: "ACM Transactions on Graphics 27(3) (SIGGRAPH 2008)"
url: "https://dl.acm.org/doi/10.1145/1399504.1360631"
code: ""
project_page: "https://cemyuksel.com/research/dualscattering"
category: [hair, rendering, multiple-scattering, real-time, classical]
importance: A
historical_importance: 4
game_relevance: 4
production_readiness: Industry Adopted
user_level: Normal
status: unread
aliases: [Dual Scattering, Zinke 2008, 双散射]
tags: [hair, rendering, multiple-scattering, real-time, transport]
---

# Dual Scattering Approximation for Fast Multiple Scattering in Hair (Zinke / Yuksel 2008)

## TL;DR

**"浅色头发为什么必须算多次散射、又怎么在实时里算得起"——这篇给出了此后二十年一直被复用的答案：把多次散射拆成两个分量。**

1. **全局多散射 ΨG**：光**穿过发束体积、到达兴趣点邻域**的过程 → 近似为**沿单条原型路径（shadow path）的"事件序列统计"**：透射率连乘 × 纵向方差累加。实现上**"非常类似标准半透明毛发阴影技术"**（原文：*"becomes very similar to standard semi-transparent hair shadowing techniques"*）——只是每个像素/层存的不是单个 opacity，而是 7 个值；
2. **局部多散射 ΨL**：到达邻域**之后**的多次回散射 → 建模为**不含位置 x 的材质属性**（`fback`，预计算表查值），与 BCSDF 并排相加——"shader 只是 BCSDF 着色的简单扩展"。

分解公式：`Ψ(x,ωd,ωi) = ΨG(x,ωd,ωi) · (1 + ΨL(x,ωd,ωi))`。

**成绩单**（原文 Figure 1/10/11/12，路径追踪 = ground truth）：
**7.8 小时 → 离线 5.2 分钟 → 实时 14 fps**（50K 发丝）；预计算只需几秒；**全程无参数调整**——*"all calculations are based on computable or measurable values, so there is no unintuitive parameter tweaking"*。

三个"可搬走"的结论：

- **一条原型路径可以代表全部路径**——前提是散射事件统计独立 + 相邻发丝方向强相似（原文两条假设）；
- **"方差可加"让扩散可推导**——纵向散射是高斯，多次事件的扩散 = 方差求和（加上中心化的偏移），不需求解；
- **同一近似自带三档实现**（ray shooting / 体素 forward scattering map / GPU 分层）——质量-成本谱系的教科书样本。

## Problem

2003 年 [[Marschner — Light Scattering from Human Hair Fibers (2003)|Marschner]] 给出单散射物理模型后，一个事实被确认：**深色发准确，浅色发不行**——浅色发（低吸收）允许光穿透更深，**多次纤维散射（multiple fiber scattering）对整体发色的正确感知必不可少**。

但 2008 年时，算得起的都不物理、物理的都算不起：

| 路线 | 状态 |
|---|---|
| 路径追踪（真值） | 单帧 7.8–22 小时 |
| 光子映射（Moon-Marschner 2006 / Zinke-Weber） | 准确但**数十分钟-小时 + 高分辨率光子图内存**，不可交互 |
| 实时做法（当时） | 把多散射当**半透明阴影 + ad-hoc diffuse 项**——忽略纤维圆形截面；原文判词：*"gives hair a dull appearance"*，且**需要反复手调参数**，仍复现不了三大现象：**方向性 / 色彩偏移 / 连续模糊**（*directionality, color shifts, and successive blurring*） |

## Historical Context

- **出处**：Universität Bonn（Zinke、Weber）× Texas A&M（Yuksel、Keyser），SIGGRAPH 2008；资助含 NSF 与 DFG；
- **在毛发谱系中的位置**：1989 texel（Kajiya-Kay，经验）→ 2003 单散射物理（Marschner）→ 2004 工程三刀（[[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)|Scheuermann]]，实时近似单散射 + 发片）→ **2008 本文（多散射的实时近似）**——**[[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]] 笔记的 Technology Evolution 里"2008+ dual scattering"一格，今天正式落库**；
- **后续影响**：dual scattering（及其变体）成为影视/离线毛发渲染的**多散射标准近似**；"全局/局部分离"原则被作者自己点名为可推广到**雪、云、编织织物**（原文结语）。

## Previous Work

- [[Marschner — Light Scattering from Human Hair Fibers (2003)]]——被补全的对象：其 BCSDF（R/TT/TRT 三叶、α/β 参数、2D 方位表 N）被本文直接复用，**本文只新增"多散射"的两项**；
- Moon & Marschner 2006（光子映射）/ Zinke & Weber 2006/2007（光子映射、filament 理论）——准确路线的成本黑洞；
- Yuksel et al. 2007（投影式纤维 GI）——省掉了 inter-reflection，精度受损；
- 半透明阴影技术族（Deep Shadow Maps / Opacity Shadow Maps / Deep Opacity Maps，Lokovic-Veach 2000 → Kim-Neumann 2001 → Yuksel-Keyser 2008）——**本文的全局分量直接站在它们肩上**；
- 参与介质近似（Premoze et al. 2004）——精神相通，但未处理毛发的高各向异性。

## Core Idea：双散射（Dual Scattering）

**多散射函数 = 全局分量 ×（1 + 局部分量）**：

```text
ΨG（全局）：光从外面穿过发束体积 → 到达 x 的邻域        ——"输运"问题
ΨL（局部）：光到达邻域后 → 在邻域内的多次回散射再离开 x ——"响应"问题
```

**为什么能这么拆（两条原文假设）**：

1. **散射事件统计独立**：光到达某点的能量*"不依赖实际路径，而依赖路径上散射事件的质量"*（*"the energy of light arriving at a point does not depend on the actual path, but the quality of the scattering events along the path"*）→ 簇的几何细节（如纤维间距）可忽略；
2. **相邻发丝方向强相似**（真实发型的最强统计特征）→ 路径概率分布彼此接近。

**前向 / 后向散射分裂**（把"哪些散射归哪个分量"讲清楚的关键）：

- **前向（≈TT，前半锥）**：浅色发下显著更强 → **全部划给全局**。后向路径要"偶数次后向"才能折返回 x，概率极低，**全局分量直接忽略双后向及以上**（原文：*"we disregard these double (or more) backward scattered paths"*）；
- **后向（R / TRT 类回散射）**：至少 1 次、最多计到 3 次后向的路径 → **全部划给局部**，且**建模为材质属性**（不含 x！原文：*"fback is not a function of x, which means that it is modeled as a material property"*）。

## Technical Approach

### 1. 全局分量：沿单条阴影路径的"事件序列统计"

对每条到 x 的光路，沿途每个散射事件贡献两个量：

| 量 | 含义 | 组合方式 |
|---|---|---|
| `āf(θ)` | 该事件的平均前向衰减（对 BCSDF 在前半锥积分） | **连乘**：`Tf = df · ∏āf(θk)` |
| `β̄²f(θ)` | 该事件的纵向散射方差（直接取自 BCSDF） | **求和**：`σ̄²f = Σβ̄²f(θk)` |

→ **透射（衰减）= 连乘；扩散（角度变宽）= 方差相加**。两个统计量就刻画了整条路径——这就是"原型路径"近似的全部内容：

> **不追几何，只追每一个散射事件的"衰减 × 扩散"两个标量。**

细节：

- **density factor `df`**：x 未必被纤维完全包围（只从部分方向收到多散射光）——实践中取常数（原文实验：0.6–0.8 合理，论文全部用 **0.7**）。**这是全篇唯一"可调"参数，且作者强调它有物理含义、理论上可精确计算**；
- **spread function**：方位向在几次散射后≈各向同性（常数 1/π）；纵向仍是窄高斯（与 BCSDF 的 M 项同族）——**"高斯 + 高斯 = 高斯"，让"输运后的角度分布"保持闭式**；
- 高斯合并时**方差相加 + 偏移累加**——多次散射的角度扩散不用数值求解，直接从单事件参数总和推出。

### 2. 局部分量：一张"回散射材质表"

`ΨL·fs ≈ db · fback(ωi,ωo)`，其中 `fback = (2/cosθ)·Āb(θ)·S̄b(ωi,ωo)`：

- **Āb 平均回散射衰减** = 解析和 `Ā1 + Ā3`：
  - `Ā1 = āb·ā²f/(1−ā²f)`（向前 i 次、回 1 次、再向前 i 次回来——等比级数闭式）；
  - `Ā3`（3 次回散射）有同样漂亮的三重和闭式 `ā³b···/(1−ā²f)³`；更多次忽略（发纤维 `āb` 本身小）；
- `Δ̄b`（平均纵向偏移）与 `σ̄²b`（平均方差）也有从同一级数推出的**闭式近似**（Eq. 16/17）——**"闭式再省一步"：级数求和 → 近似式**；
- **预计算表**（Table 1）：`Āb(θ)`、`Δ̄b(θ)`、`σ̄²b(θ)`、`N^G(θ,φ)`（被前向多散射改写后的方位散射表）——**全部是"数值积分一次、运行时查表"**。

### 3. 渲染方程与着色：一个"简单扩展"

- 一般形式（Eq. 18–22）支持多光源 / 面光源 / IBL / 完整 GI——**只要"沿 shadow path 估计全局散射"仍可行**；
- **着色伪代码（Figure 5）** = 已有的 BCSDF 着色 + 三项：
  1. `fback` 项（局部分量，乘 direct fraction）；
  2. 前向散射照明分支：把 BCSDF 的 Gaussian 方差**加上 `σ̄²f`**（方差合并），并换上 `N^G` 方位表；
  3. 按 `direct fraction / Tf` 混合"直接照亮"与"前向散射照亮"两种情形。

### 4. 全局分量的三档实现（本文的工程骨架）

| 档 | 做法 | 成本地位 | 关键细节 |
|---|---|---|---|
| **① Ray shooting** | 从 x 沿 ωd 打一条光线，交点即散射事件 | 最简、最准 | 类似光线追踪阴影；需多点采样消噪 |
| **② Forward scattering maps** | 体素网格（0.5 cm）预计算 Tf/σ̄²f/direct fraction，双 pass | 中 | **多光源共享同一结构**——光源越多越划算 |
| **③ GPU（deep opacity maps 式）** | 光视角深度图 → 分层；每层存 7 值（RGB·Tf + RGB·σ̄²f + direct fraction） | 实时 | **4 层 / 8 张纹理**即高质量；"消费级显卡单 pass 多重渲染目标可一次生成" |

> **三档不是三篇论文，而是同一个近似的三个成本质量点**——与 [[Scalability and Quality Tiers|分档思维]] 完全同构：**换的是"输运信息的采样与存储方式"，公式与着色器一字不改。**

## Key Contribution

1. **双散射分解本身**：多散射 = 全局输运（沿原型路径统计）×（1 + 局部响应（材质属性））——**第一个既物理、又实时、还零调参的毛发多散射近似**；
2. **"原型路径"降维**：把"所有光路"压成"单条阴影路径上的事件序列"——配合透射连乘 / 方差求和两个可加统计量，这是全文最可迁移的方法论；
3. **三档实现 + 与半透明阴影技术的同构**：全局分量在工程上退化为"加强版半透明阴影"——**多散射的实时化没有发明新管线，而是把旧管线（deep opacity maps）的每像素载荷从 1 个 opacity 扩到 7 个值**。

## Why It Works

1. **统计独立的路径**：只要事件之间不相关，路径的**几何细节被"事件质量分布"吞并**——这是"一条路径代表全部"的合法性来源；
2. **高斯族的闭合**：纵向散射用高斯描述，多个事件的扩散仍可用方差叠加闭式表达——**选择的表示"恰好"让"多次"不增加复杂度**；
3. **前向-后向物理分裂**：`āf ≫ āb`（浅色发）让"全局≈前向、局部≈回散射"不仅是近似，也是物理上正确的分工——**两类光路恰配两类技术（阴影式输运 / 材质式响应）**。

## Limitations

- **原型路径假设在强空间光照变化下失效**：*"a hard shadow edge falling across the hair... our prototype path approximation would be less accurate along the shadow boundary"*（硬阴影边界不太准）；
- **稀疏发型 / 极低衰减**：平均路径太长时发型全局结构起作用，偏差上升；
- **相邻相似假设**：混乱发型（chaotic hair）可能违反；
- **density factor 仍是常数**（0.7）——作者承认理论上可精确计算，未做；
- **仅两个散射统计量**（透射 + 方差）：牺牲了完整角度分布的高阶自由度（窄高斯的形状假设）。

## Game Development Relevance

**4/5 —— 它和它上下的邻居共同构成"游戏毛发"的完整账本。**

1. **毛发线的"多散射"节点落库**：[[Hair Rendering]] 的物理侧此前只有单散射（Marschner 2003）；本文补上"浅色发变暗/发色失真"这一整类现象的解法——**"发色看起来对不对"在浅色发上由多散射主导**（美术直觉的事实级依据）；
2. **今天最值钱的可迁移判据（一）："把问题拆成'输运'与'局部响应'两半"**——判据形态：**问结果里"光怎么到达这里"与"到了之后怎么散开"能不能各自用最合适的技术近似**（前者用阴影式降价管线、后者用材质式查表）。这与 PBR 能量账本的 **"补能量 vs 推输运是两件事"**（[[Multiple Scattering and Energy Compensation]]）互为跨域同构——**一个在微面尺度拆，一个在发束尺度拆**；
3. **可迁移判据（二）："一条原型路径代表全部路径"的先决条件**——先问"**结果的哪些成分依赖路径几何、哪些只依赖事件质量**"；只依赖后者的，几何可以整体丢给统计量。这是"降维近似"的合法性检查表；
4. **三档实现 = 分档样本**：同一个物理近似、同一套 shader，三档只换"输运信息怎么采样/存"（射线 / 体积网格 / 分层图）→ 对 [[Scalability and Quality Tiers]] 的又一实证：**"换表示层级"比"缩参数"更接近分档的本义**；
5. **对 [[Real-Time VFX Performance Budgeting]] 的边界说明**：这是一笔"**每光源 × 一遍光空间处理**"的账（全局分量）——与 [[Williams — Casting Curved Shadows on Curved Surfaces (1978)|阴影的"每灯 +1× 场景"]] 同族；**发丝/头发的多散射成本跟着"头发体积 × 光源数"走**，而不是跟着"发丝数"走（方差/透射都在体积统计里，与发丝几何解耦）。

## Unreal Engine Relevance

- UE 的毛发着色（Groom / Hair Shading Model）以 [[Marschner — Light Scattering from Human Hair Fibers (2003)|Marschner]] 单散射为基础；**"浅色发多散射补光"在引擎里属于后续增强项**——本文是理解该增强"为什么长这样"的原始文档（具体引擎实现是否使用 dual scattering 或变体，**以官方文档为准，本笔记不做断言**）；
- **全局分量与 UE 的毛发阴影线路同族**：光视角深度图 + 逐层不透明度累积 = "深度不透明度映射"谱系（[[Shadow Mapping]] 的成本账在这里的形态是"每个光源一遍分层的头发深度渲染"）；
- **与前向渲染的关系（对动态灯光维度复审的意义）**：这是一个"**每光源一遍、内容固定**"的管线，天然亲和前向式逐光源处理——与 [[2026-09-24]] 记录的 MegaLights"与前向渲染器不兼容"形成对照：**头发多散射这类专项管线，移动端（无 MegaLights）仍按本笔记的账本走**。

## Technology Evolution

```text
1989 Kajiya-Kay —— texel + 经验高光（离线）
        ↓
2003 Marschner —— 单散射物理（BCSDF：R / TT / TRT）
        ↓
2004 Scheuermann —— 实时工程三刀（发片 / 移位高光 / 取消排序）
        ↓
2006 Moon-Marschner / Zinke-Weber —— 光子映射多散射（准确但数十分钟-小时）
        ↓
★ 2008 双散射（本文）—— 全局/局部分解 + 原型路径：7.8h → 5.2min → 实时 14fps
        ↓
2008+ 被影视/离线管线长期采用（多散射标准近似）；"全局/局部"原则外溢到雪/云/织物
        ↓
2010s TressFX / HairWorks / UE Groom —— 发丝线与发片线并行
        ↓
2026 —— 资产侧（HairCS）/ 画面侧（DLSS 5 神经增强）/ 光追侧（《巫师 3》重制版 LSS 光追毛发）
```

## Relationships

### Based On
- [[Marschner — Light Scattering from Human Hair Fibers (2003)]] —— BCSDF、α/β 参数、方位表 N 全部复用；本文补的正是它留白的"多散射"
- [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]] —— 更上游的毛发渲染源头（texel 表示）

### Productionizes / Extends
- [[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]] —— 前者把单散射做进实时，本文把多散射做进实时；**两者共享"与半透明阴影技术同构"的血缘**（Scheuermann 的排序/自阴影策略 ↔ 本文的光视角分层）

### Related
- [[Hair Rendering]] —— 本笔记补上该概念的**"多散射"缺口**（物理侧最后一块）
- [[Multiple Scattering and Energy Compensation]] —— **跨域同构**："补能量 vs 推输运"在毛发尺度的对应物 = "局部响应 vs 全局输运"
- [[Shadow Mapping]] —— 全局分量的实现家族（深度不透明度映射）是阴影技术的直系后代
- [[Linear Transport Theory]] —— 本文是"输运近似"的又一样本（把输运压缩为沿单路径的两个统计量）
- [[d'Eon — A Hitchhiker's Guide to Multiple Scattering (2022)]] —— 更一般的参考层（本文的全局/局部分解是其"多层次散射"思想的工程特例）
- [[Scalability and Quality Tiers]] —— 三档实现：换采样/存储方式，不换公式
- [[Real-Time VFX Performance Budgeting]] —— "头发体积 × 光源数"的成本口径

## Personal Knowledge State

- **user_level: Normal（毛发实时侧）**。判断依据：**"多散射让浅色发不暗"是美术/TA 的常识级结论（Easy 区）**；**新的是三处机制层**——为什么一条路径能代表全部路径、方差为什么可加、"全局像阴影/局部像材质"的分工依据。

### Mastery 自测（3 条；与 Marschner 5 + Kajiya-Kay 4 + Scheuermann 3 合并 = 毛发线 15 条）

1. **为什么单散射对浅色发不够？**（答：浅色发吸收低 → 光穿透深、多次散射贡献大；单散射模型只算一次弹射，浅色发会整体偏暗/发色失真——这是"能量账本"在毛发上的形态。）
2. **"一条原型路径代表全部路径"靠什么成立？**（答：两条假设——散射事件统计独立（能量只依赖事件质量不依赖路径几何）+ 相邻发丝方向强相似；于是只沿 shadow path 统计"衰减连乘 + 方差求和"两个标量即可。）
3. **全局 / 局部分量的分工依据是什么？为什么全局像"阴影"、局部像"材质"？**（答：前向散射（TT）强且沿光源方向输运 → 用光视角分层图式管线；后向散射（R/TRT）概率低、贡献低频且与位置无关 → 预计算成 `fback` 材质表。`fback` 不含 x 是"材质属性"论断的原文依据。）

## Visualization

![[双散射_Zinke 2008 全局局部与原型路径图解.html]]

含：全局/局部分解公式、single shadow path 的事件序列（透射连乘 + 方差求和）、前向/后向散射的路径分类、三档实现对照、7.8h→5.2min→14fps 成本阶梯。

## Notes

- **原文已下载并逐节核对**（作者主页 Preprint，10 页）：本笔记所有引语/公式/数字均出自原文——
  - `Ψ(x,ωd,ωi) = ΨG(x,ωd,ωi)(1+ΨL(x,ωd,ωi))`（Eq. 3）；`Tf = df∏āf(θk)`（Eq. 5）；`σ̄²f = Σβ̄²f`（Eq. 8）；`Ā1 = ābā²f/(1−ā²f)`、`Ā3 = ā³bā²f/(1−ā²f)³`（Eq. 11/13）；
  - density factor 0.6–0.8 / 全文 0.7（§3.1.1）；"4 层 / 8 纹理"（§4.1.3）；
  - 性能（Figure 1/10/11/12）：7.8h→5.2min→14fps；22h→9.6min / 4.6min / 5.8fps；17h→光子映射 67min→ray shooting 15min→FSM 7min；11.8h→11.2min→12fps（deep opacity maps 对照 18fps）；
  - 三条局限引语（§6）：原型路径受硬阴影边界影响 / 稀疏发型偏差 / chaotic hair 违反相邻假设；
  - 结语的名句：*"separating local and global multiple scattering is a very general principle... applicable... also for other highly scattering quasi-homogeneous structures such as snow, clouds or woven textiles."*
- 引用信息：ACM TOG 27(3)，SIGGRAPH 2008；DOI 10.1145/1399504.1360631；作者主页 `cemyuksel.com/research/dualscattering` 提供 Preprint（3.7MB）；
- 入库时机：2026-09-28（Run #20）。触发来源：[[Hair Rendering]] 的多散射缺口 + [[Scheuermann — Practical Real-Time Hair Rendering and Shading (2004)]] 笔记 Technology Evolution 中预留的"2008+ dual scattering"节点——**今日结清**；亦与《巫师 3》重制版（9-29）LSS 光追毛发的兴趣窗口对齐。
