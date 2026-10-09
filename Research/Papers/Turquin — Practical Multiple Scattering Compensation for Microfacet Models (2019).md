---
type: paper
title: "Practical Multiple Scattering Compensation for Microfacet Models"
authors: [Emmanuel Turquin]
year: 2019
published: "2019（Technical Report；文档时间戳 2018-09-11——PDF 元数据核实；网络版由 Self Shadow 托管，ACM DL 等统一引作 Turquin 2019）"
venue: "Technical Report, Industrial Light & Magic（5 页）"
url: "https://blog.selfshadow.com/publications/turquin/ms_comp_final.pdf"
code: ""
project_page: "https://blog.selfshadow.com/publications/turquin/（Self Shadow 托管页；本次经原始 PDF 逐页核对——全文文本 479 行）"
category: [rendering, multiple-scattering, energy-compensation, ibl, production, real-time]
importance: A（经典）
historical_importance: 4
game_relevance: 5
production_readiness: "Industry Adopted（Unity HDRP 与 Google Filament 采用其 F0 缩放形态——Self Shadow 官方说明；完整版实装于 Isotropix Clarisse / RenderMan RIS / Mitsuba；TR 明确'可加入任何渲染器，含实时光栅化引擎'）"
user_level: Normal
status: unread
aliases: [Turquin 2019, Turquin TR, 缩放已有 lobe, rescaled specular lobe, Lagarde18, Multiple Scattering GGX, 最便宜的能量补偿, 路线⑤]
tags: [rendering, multiple-scattering, energy-compensation, ibl, production]
---

# Practical Multiple Scattering Compensation for Microfacet Models（Turquin, ILM, 2019）

> **入库 2026-10-09（Run 31）。** 作者 **Emmanuel Turquin**（Industrial Light & Magic）——**同一个人此前已两次出现在本库**：Kulla-Conty 2017 的 $F_{avg}$ 勘误指出者；Fdez-Agüera 2019 核对时 Filament 文档致谢的"缩放 lobe 观察"提出者。**本篇是他的原始技术报告（TR），此前库内只有二手转述（Filament 文档 + Hill 笔记脚注），今天升格为独立一手节点。**
> **它是 [[Multiple Scattering and Energy Compensation]] **五条补法路线的第 ⑤ 条**（"缩放已有 lobe"，最便宜版）的**原始文献**——用户 2026-10-08 自制五路线对比图解中指出该分支（"Rescale Original GGX"）此前正是唯一没有独立笔记的路线。**五条路线至此全部有独立节点。**
> **⚠️ 修正留痕（本次由原文核实）**：库内 2026-09-20 起把路线 ⑤ 记为"**只 IBL**"——**按 TR 原文，该限制不成立于方法本身**：$\rho = \text{gain}(\omega_o)\cdot\rho_{ss}$ 是** BRDF 级修正**，只依赖单次散射方向 albedo（与光源无关），原文明确"适用于任意 BSDF"且"可加入任何渲染器（含实时光栅化引擎）"。"只 IBL"疑为对 **④（Fdez-Agüera）** 限制的串行误记，或对 Filament 实装语境（$E(l)$ 与 IBL 预积分共享查表）的过度外推。**正确边界见 §Limitations。**（同类修正先例：Pixar→WDAS。）

## TL;DR

**"把已有高光 lobe 整体乘一个系数"——全部五条路线里最便宜的一条，便宜到只剩一次乘加，而且不新增任何项、任何采样、任何贴图（若复用 $\text{DFG}$ 表的通道）。**

核心三行：

$$k_{ms}(\omega_o)=\frac{1-E_{ss}(\omega_o)}{E_{ss}(\omega_o)},\qquad
\rho=\rho_{ss}+F_{ms}\,k_{ms}(\omega_o)\,\rho_{ss},\qquad
\boxed{\ \rho_{ss}\leftarrow\Big[1+F_0\,\frac{1-E_{ss}(\omega_o)}{E_{ss}(\omega_o)}\Big]\,\rho_{ss}\ }$$

- $E_{ss}$ = 单次散射 BRDF 的方向 albedo（$F=1$ 时）——**唯一需要预存的东西**（TR：32×32 表；工程实装可复用已有 IBL 的 DFG 表）；
- $F_{ms}$ 从"精确形"一路简化到 $F_0$（论文给了四级简化链 §3.1.2，肉眼几乎不可分）；
- **实现形态**：不是新增 lobe，而是**把原有镜面项乘一个 gain**——"applied as a gain to the closure, instead of having to modify the closure itself"（原句）。采样、求值、PDF 全部原样复用。

代价：形状假设最粗（"多次散射 ≈ 缩小的原 lobe"）；非互易（只依赖 $\omega_o$）；不修低粗糙度电介质超额。**但它被 Unity HDRP 和 Filament 采用**——本库"研究 → 引擎"距离最短的一条。

## Problem

微面 BRDF 的单次散射假设丢能量（[[Multiple Scattering and Energy Compensation]] 的账）：

- 原文数字：**GGX、忽略 Fresnel 吸收时，$\alpha=1$ 处丢约 60%**；长尾分布（GTR / STD）**超过 90%**（Figure 2 的白炉测试）；
- 产业现状（原文描述）：高粗糙度材质"发暗发闷"，制作环节靠**手调 albedo** 补偿——"eye-balled, manual compensation... applied typically at the look development stage, and potentially breaking energy conservation in other ways"；
- 而"把账算对"的两条已知路（Heitz 2016 随机输运、Kulla-Conty 2017 查表 lobe）各有代价：前者**对既有渲染器不友好 + 计算昂贵**，后者**要新增表 + 新增 lobe（混入一个漫射形状的项，破坏解析重要性采样的最优性）**。

**本篇的问题意识**：还能不能更便宜？——便宜到"不改管线结构、不加资源、不换采样"。

## Historical Context

```text
1981  Cook-Torrance（单次散射形态定型——"洞"从此刻起就存在）
        ↓
2014  Heitz JCGT《Understanding the MSF》——把洞讲清楚，并留下三个"待办"（Q1/Q2/Q3）：
      Q1 "ρms 的形状可以用蒙特卡洛去研究" / Q2 "如果简单，可近似成单个 lobe" /
      Q3 "多次散射趋于平滑，应该能用解析函数或小查找表高效表示"
        ↓
2016  Heitz et al.（SIGGRAPH）：随机游走把输运推对——
      ★ 附带发现（Figure 15，被本篇引用）：**次级 lobe 不像漫射，而像"缩小的主 lobe"**
        ↓
2017  Kulla-Conty（Imageworks）：工程化——但选了"漫射形状"的附加 lobe
      （为互易性；与 Heitz 的形状观察相悖、自认近似）
        ↓
★ 2019 本篇（ILM）：回到 Heitz 的形状观察 + Q2/Q3——
      "既然像缩小的主 lobe，那就直接缩放主 lobe"
      → 五条路线中最廉价的形态；Unity HDRP / Filament 采用
```

**三条路线的"分叉点"一句话**：Heitz 把输运算对（贵）；Kulla-Conty 把账做平（要新表 + 漫射形状）；**Turquin 把账做平时顺手把'形状'也省了——直接用原 lobe**。

## Previous Work

原文 §2 逐条点名的关系（本篇是一份很好的"2001–2019 谱系综述"）：

- **Burley 2015 Sheen**：定性补偿（经验高光），**不随 roughness 自动调整**——需要每次改动 BSDF 参数时手动重调（原文批评）；
- **Heitz 2014 JCGT**：理论框架 + 上面三个问题（Q1/Q2/Q3）。**本篇直接把自己定义为 Q2/Q3 的答卷**；
- **Heitz 2016**：精确但"involved + computationally very expensive"；**其 Figure 15 的形状观察是本篇的灵感来源**（"secondary lobes do not look diffuse but rather like scaled-down versions of the primary one"）；
- **Kulla-Conty 2017**：改编 **Kelemen & Szirmay-Kalos 2001** 的补偿公式——互易、稳健、快；但"elects to use a diffuse-looking ρms lobe, hence contradicting previous results from Heitz et al. 2016"（原文直接点出这个矛盾）。

## Core Idea

**把"补一个项"改成"缩放一个项"。**

Kulla-Conty 的 $f_{ms}$ 是一个**新增**的、方向分布是漫射的独立 lobe；Turquin 的 $f_{ms}$ 直接取**原单次 lobe 的形状**：

$$f_{ms}=F_{ms}\,k_{ms}(\omega_o)\,f_{ss}$$

于是总 BRDF 就是一个**纯缩放**：

$$\rho=(1+k_{ms}(\omega_o))\,\rho_{ss}=\frac{\rho_{ss}}{E_{ss}(\omega_o)}\quad(F=1\ \text{时})$$

**归一化逻辑（15 行内可自己推）**：若 $\rho=\rho_{ss}/E_{ss}(\omega_o)$，则

$$E(\omega_o)=\int \frac{\rho_{ss}(\omega_o,\omega_i)}{E_{ss}(\omega_o)}|\omega_i\cdot n|\,d\omega_i=\frac{E_{ss}(\omega_o)}{E_{ss}(\omega_o)}=1$$

自动成立。**所以"账平"不需要建新模型，只需要"除以自己的 albedo"。**

## Technical Approach

### 1. 能量项（$k_{ms}$）与它的表

$k_{ms}(\omega_o)=(1-E_{ss})/E_{ss}$——$E_{ss}$ 平滑，预存为 **32×32 表（$\cos\theta_o \times \sqrt{\alpha}$）**；带 $\gamma$ 尾参数的分布（GTR / STD）扩到 32×32×32。
**注意与 Kulla-Conty 的表格差异**：Kulla-Conty 需要 **2D（$E$）+ 1D（$E_{avg}$）两张表**；Turquin **只要一张 2D 表**（$F_{ms}$ 的简化链把 $E_{avg}$ 也省了，见下）。

### 2. Fresnel 项（$F_{ms}$）的四级简化链 ★

TR 的"极简主义"最集中的体现（§3.1.2）——四个版本逐级砍：

| 版本 | 公式 | 来源/依据 | 成本 |
|---|---|---|---|
| ① 复用 Kulla-Conty 精确形 | $F_{ms}=\dfrac{F_{ss}E_{avg}}{1-F_{ss}(1-E_{avg})}$ | 来自 Jakob 2014 §5.6 几何级数推导 | 最高（含 $E_{avg}$） |
| ② 拟合"平均行走深度" | $F_{ms}=F_{ss}^{\,\alpha\sqrt{\alpha}}$ | **对 Heitz 2016 模型实测**：平均随机行走深度从 1（$\alpha$=0）变到 ~2（$\alpha$=1），用 $1+\alpha\sqrt{\alpha}$ 拟合（Figure 7 右） | 中 |
| ③ 固定 $F_{ss}$ | $F_{ms}=F_{ss}$ | 因为 $F_{ms}$ 主要在 $\alpha\to1$ 时起作用，就取该处值 | 低 |
| ④ **裸公式** | $F_{ms}=F_0$ | 金/铜等物理 Fresnel 下 $F_{ss}\approx F_0$（原文实测："can be interchangeably used with no visual difference"） | **最低** |

> **原文原句**："no matter what Fresnel term is chosen between 12, 13, 14 and 15, the results are visually very close."
> **一句可背**：**"补回去的能量，90% 由 $1-E_{ss}$ 决定，Fresnel 修正只剩下一点点饱和度差异。"**

### 3. 最终公式与实现形态

$$\rho=\Big(1+F_0\,\frac{1-E_{ss}(\omega_o)}{E_{ss}(\omega_o)}\Big)\rho_{ss}
\qquad\Longleftrightarrow\qquad
\text{gain}=1+F_0\Big(\frac{1}{r}-1\Big),\ r=E_{ss}$$

**三条实现优势（原文 §4 逐条）**：
1. **只依赖 $\omega_o$**（Kulla-Conty 依赖 $(\omega_o,\omega_i)$）→ **可以不过闭包、直接在闭包外面乘 gain**（"applied as a gain to the closure"）；
2. **形状完整保留** → **求值、采样、PDF 全部复用**（"reuse the exact same sampling and evaluation routines, with the same PDF"）——**对比 Kulla-Conty：混入漫射 lobe 后，解析重要性采样的最优性无法达成**；
3. **不新增项** → 更简单、也稍快。

### 4. 电介质（唯一需要扩表的情况）

导体可以"忽略 Fresnel 归一化"；电介质不行（$E_R$ 与 $E_T$ 的比例是界面守恒的一部分）。处理方式：把 (反射+透射) 的和 $E^S_{ss}=E^R_{ss}+E^T_{ss}$ 一起归一化：

$$\rho^S=\frac{\rho^R_{ss}+\rho^T_{ss}}{E^S_{ss}(\omega_o)}$$

代价：表变 **3D**（加 IOR 维；两类界面各一张 32×32×32）。**没有独立的 $F_{ms}$ 项**（Fresnel 已经进了 $E^S_{ss}$）。原文对玻璃的效果评价：**"maybe even more so than with conductors"**（Figure 8——粗糙玻璃的补偿效果比金属更明显）。

## Key Contribution

**在"研究 → 生产"的窄缝里，把方案压到最小形态——并且给出了一张完整的对照表（Table 1）：**

| | 次级 lobe 形状 | 互易性 | 最优采样 | 物理可信度 | 简单度 | 速度 | 灵活性 |
|---|---|---|---|---|---|---|---|
| Heitz 2016 | 与主 lobe 相似 | ✅ | ❌ | **A+** | C | C | B |
| Kulla-Conty 2017 | 方位不变（漫射状） | ✅ | ❌ | B | A | A | A+ |
| **本篇** | **与主 lobe 相同** | ❌ | **✅** | B+ | **A+** | **A+** | **A+** |

**成本实测（Mitsuba，对比 Heitz 2016 原版实现）**：简单导体测试 **7× 减速**、电介质透射测试高达 **15× 减速**——而"the visual difference appeared minimal"。

**"最小值"的证据**：$F_{ms}\to F_0$ 后，连 $F$ 的多项式求值都省了（原文还讨论了"若目标是实时，甚至可以为 $E_{ss}$ 找解析拟合，避免纹理访问"——列为 future work）。

## Why It Works

三条支撑：

1. **归一化恒等式**（上文推导）——"账平"只需除法；
2. **Heitz 2016 的形状观察**——次级 lobe"像缩小的主 lobe"，所以缩放主 lobe 是对分布的**正确一阶近似**（不是发明的假设）；
3. **单标量的充分性**——"补能量"路线（③④⑤）的共同前提：**先把方向 albedo 这个标量补平**，其余交给近似。缩放法把这个前提执行到了最彻底：**连"往哪个方向补"都交还给原 lobe 自带的方向分布**。

## Limitations

- **非互易**（依赖 $\omega_o$ 而非 $(\omega_o,\omega_i)$）：原文自评"could be a deal breaker when the resulting BSDF is used with bidirectional light transport"（BDPT 类双向渲染器），但又说"unlikely to produce any significant artefacts"——**权衡点：你的渲染器是否吃互易性**；
- **$F_{ms}$ 的可分离假设是粗近似**（原文自认"very coarse approximation"：方向不变 + 与 $\rho_{ms}$ 分离）；
- **对窄光源的补偿精度未做系统量化**——TR 自己把这条列为 future work："Comparisons in this report have been largely of a qualitative nature"（本库注：这一缺口后来由 d'Eon 2022 手册的球面 albedo 拟合与后续工作部分填补）；
- **只补高粗糙度端**：gain 在低粗糙度趋近 1（$E_{ss}\to1$）→ **不修低粗糙度电介质的掠射超额能量**（那是 Fdez-Agüera ④ 的职责——修的是"多发了一笔钱"的另一端）；
- **修正留痕的边界**（见文首）：方法**不限光源**（BRDF 级 gain）；但工程实装的"廉价性"依赖 $E(l)$ 与 IBL 预积分共享查表——**在只有解析光、没有 IBL 表的管线里要自己补一张 $E_{ss}$ 表**。

## Game Development Relevance

- **引擎采用事实**：Self Shadow 官方说明——"adopted within **Unity's HDRP** as well **Google's Filament**"；本篇另有完整版实装于 Clarisse / RenderMan RIS / Mitsuba；
- **"几乎免费的正确性"的标准样本**：在一个已经跑 IBL 的管线里，它 = **一次乘加 + 复用已有表**——比 ④（Fdez）还少（④还要几条标量 + irradiance 查表）。**这类项应当默认开启，而不是当成分档项**（沿库内 9-20 判据）；
- **"gain to the closure"是一个可迁移的集成模式**：不改闭包、只乘系数——**"最小侵入的物理修正"**。做管线工具时，"能不能只乘一个 gain"应是审新方案的第一问；
- **分档含义**：若必须分档，它属于"关掉后能看出但能忍"的最轻一档（高粗糙度金属发暗、失饱和）。

## Unreal Engine Relevance

- **principle 级映射**：UE 的 Sky Light 镜面路径 + 材质侧 gain。若要在 UE 里复现（研究方向参考），位置与 ④ 相同——**镜面着色段、输入已有 EnvBRDF LUT（R16G16）**；
- ⚠️ **不要与 UE 自身的 "Energy Conservation" 术语混淆**：UE 的 `r.Shading.EnergyConservation` 系实现与本文公式并不保证一致（用户图解已列检查点，**动手前跑 furnace test 看当前状态**——与既有实测项同一动作）；
- **TR 明说"可加入实时光栅化引擎"**：本篇不是"只离线"的论文——它是给实时光栅化写的。

## Technology Evolution

五条路线的完整谱系见 [[Multiple Scattering and Energy Compensation]]（Historical Evolution 段）。本篇对应的行：

```text
2018  Lagarde & Golubev（SIGGRAPH 2018 Advances 课程"Multiple Scattering GGX"节；
      credit Emmanuel Turquin）—— 课程版先落地
        ↓
★ 2019 本篇 TR —— 完整版：四级 Fms 简化链 + 电介质 3D 表 + 与 Heitz/KC 的对照表
        ↓
2019+ Filament / Unity HDRP 采用（F0 缩放形态；E(l) 与 DFG 表共享）
        ↓
2026-09  Fdez-Agüera 2019 核对时反向发现：本形态正是库内"研究 → 引擎距离最短"的
         那一条 → 本库 2026-10-09 补上独立节点
```

**本篇在"工程化"这条支线上的位置**：Kulla-Conty 2017 是"把公式做成可用方案"；**本篇是"把可用方案再砍一半"**——两者合起来构成"互易 vs 最省"的二选一（原文结论段原句：生产环境里最合适的就这两个，取决于渲染器多需要互易性）。

## Relationships

### Based On

- [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] —— **形状观察的来源**（次级 lobe ≈ 缩小的主 lobe；Figure 15）＋ $F_{ms}$ 简化链的实测对象（平均行走深度）；
- [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] —— $F_{ss}$ / $E_{avg}$ 的定义与 ① 级公式直接复用；Fresnel 修正同源；
- Kelemen & Szirmay-Kalos 2001（经 Kulla-Conty 转接）—— 补偿的思想源头（库内未入库，见概念表）。

### Contrasts

- [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] —— **互易 ↔ 最省的对偶**：KC 选漫射形状换互易；本篇选形状正确换掉互易与成本。**"次级 lobe 到底像什么"是两份文档里被明确记录的公开分歧**（Heitz 形状 vs KC 漫射）。

### Related

- [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] —— 路线 ④：同样"零新增资源"但依赖 IBL 的 irradiance 近似；**两句互补的"缺口观"**：④"缺口是 $1-E_{ss}$，而 $E_{ss}$ 早在表里"→ ⑤"既然在表里，那就直接用除法补"；
- [[Hill — A Multi-Faceted Exploration (2018-2019)]] —— 同期"更准 vs 能用"的另一条（Hill 在 KC 路线内部继续精化）；
- [[d'Eon — A Hitchhiker's Guide to Multiple Scattering (2022)]] —— 本篇自认缺失的"定量比较"，手册侧有可对照的球面 albedo 基线；
- [[Split-Sum Approximation]] —— $E(l)$ 与 DFG 表的血缘（Filament 形态的工程根）。

### Followed By

- Unity HDRP / Filament / Clarisse / RenderMan RIS 的实装形态；
- （开放）TR 自己列出的 future work：给平滑查找表找解析拟合（"避免昂贵的纹理访问"）；不分离 Fresnel 的 3D表方案。

## Personal Knowledge State

- **user_level: Normal**。本库"多次散射"四篇核心材料（Kulla-Conty / Heitz / Fdez-Agüera / Dupuy）用户已读；本篇 = **五条路线里最后一块独立拼图**；
- **与用户的直接关系（2026-10-08 信号）**：用户在自制五路线图解里已把本条列为 "Rescale Original GGX"，并写下近似正确的公式（"f_total ≈ f_ss × gain"）——**本篇补的是"gain 的四级简化链"与"非互易/单端"这两个细节**；
- **读法建议（≈20 分钟，按原文章节）**：§3.1.2 的简化链（Fig 7）→ §4 成本对比（7×/15×）+ Table 1 → §5 结论段（"两个候选、看互易性"）。

## Learning Value

1. **"提纯"式研究的一个范式样本**：不是发明新模型，而是**沿着别人的观察（Heitz 形状）把方案一路砍到不能再砍**——四级简化链每一步都标注了"依据是什么、砍掉后差多少"；
2. **"gain to the closure"集成模式**：最小侵入物理修正的模板；
3. **互易性作为可议价的属性**：工业界为一分钱可以放弃形式互易（BDPT 语境例外）——**"什么属性是可以放弃的"是分档设计的前置问题**。

## Visualization

![[多次散射_路线⑤_Turquin缩放lobe与简化链图解.html]]

## Notes

- **引名说明**：Filament 文档的 `[Lagarde18]` 条目（Lagarde & Golubev，SIGGRAPH 2018 课程）与本篇 TR 是**同一方法的两份文档**——课程版先到、TR 版最全（Self Shadow 原话："he goes into all of the details of his approach and various shortcuts"）。**引用时：方法溯源用本篇 TR（2019），课程归属记 Lagarde & Golubev（2018）。**
- **勘误连续性**：本条线三篇工程文献里，Kulla-Conty 与 Fdez-Agüera 都有官方勘误——**本篇（TR）未见勘误记录**（本次核对 PDF 全文无 errata 页）；但注意其 §4 表格是"定性"性质。
- **同步修正**：[[Multiple Scattering and Energy Compensation]] 路线表 ⑤ 行、[[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] 的"三选一"表与 [[Hill — A Multi-Faceted Exploration (2018-2019)]] 的相关行已随本篇入库同步更新（"只 IBL"→ 按原文修正）。
