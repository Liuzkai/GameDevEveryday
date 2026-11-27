---
type: paper
title: "Physically Based Shading at Disney"
authors: ["Brent Burley"]
year: 2012
published: "2012-08"
venue: "ACM SIGGRAPH 2012 Course: Practical Physically Based Shading in Film and Game Production（课程讲义 v3，27 页含附录）"
url: "https://blog.selfshadow.com/publications/s2012-shading-course/burley/s2012_pbs_disney_brdf_notes_v3.pdf"
code: "BRDF Explorer 开源：github.com/wdas/brdf"
project_page: "https://media.disneyanimation.com/uploads/production/publication_asset/48/asset/s2012_pbs_disney_brdf_notes_v3.pdf"
category: ["PBR", "BRDF", "Material Model", "Production", "Disney"]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（《无敌破坏王》全片生产验证；随后成为 UE4 / Unity / glTF / USD 的默认材质参数化）"
user_level: Normal
status: unread
aliases: [Disney BRDF, Disney Principled BRDF, Burley 2012, 迪士尼原则化 BRDF, Principled Layers]
tags: [pbr, brdf, material-model, production, disney, metallic-roughness]
---

# Burley 2012 — 十个旋钮，把"物理"换成"可插值"

## TL;DR

这篇不是一篇"更好的 BRDF"，是一篇**关于参数化设计的论文**。

它的出发点是艺术家的一句抱怨：*we need our shading model to be art directable and not necessarily physically correct*。Burley 接受了这句话，然后把它翻译成五条设计原则，其中最关键的是第三与第五条：

> 3. Parameters should be zero to one over their plausible range.
> 5. **All combinations of parameters should be as robust and plausible as possible.**

于是模型不再是"一组物理量"，而是**一个每个点都对应一份合法材质的参数立方体**。这一步的后果比任何一项 BRDF 改进都大：**只有每个点都合法，插值才安全**——而材质分层、mask 混合、mip 过滤、贴图边界，全都是在插值。

## Problem

2012 年的状况是：物理模型（Cook-Torrance 一系）参数是物理量——$F_0$、粗糙度的统计量、折射率。它们**可测但不可调**：

- 美术拿到"$F_0=0.04$"不知道这是什么，也不知道往哪拧；
- 参数**没有归一化**，超出物理范围就出非物理结果；
- 更致命的是**不可插值**：两套物理参数线性混合，中间点往往既不像 A 也不像 B（Burley 原话：*interpolating between a rough and smooth material would always produce a rough result*）。

而生产里到处是插值：mask 混合两层材质、贴图被 mip 平均、遮罩边缘被抗锯齿过滤。**一个"只能取端点值"的模型，在真实管线里是不成立的。**

## Historical Context

```text
1981  Cook-Torrance     微面 BRDF：D·G·F 三项，参数是物理量
1982  Cook-Torrance 之后二十年：离线渲染用，实时用 Blinn-Phong 近似
2003  Matusik / MERL    100 个实测 BRDF（90×90×180 采样）公开 ⇒ 第一次能"拿数据说话"
2012  Burley（本篇）    把物理量重写成 10 个归一化旋钮；metallic = 两个模型的线性混合
2012  《无敌破坏王》     全片生产验证（除毛发外所有材质）
2013  Karis / UE4       实时化，MR 成为引擎默认 ⇒ 游戏侧全面跟进
2015+ glTF / USD        MR 成为交换格式
2026  ReMeDi            把"$C$ 必须匹配 $M$"从"鼓励"变成"构造"
```

它的位置很特别：**在它之前，"物理正确"是目标；在它之后，"参数化可用"成了目标，物理正确退成约束条件之一。** 本库 [[Physically Based Rendering]] 的谱系表里它一直是纯文本——今天补上。

## Previous Work

论文前半（§3–§4）是一次**对实测数据的系统观察**，用的是 MERL 100（Matusik 2003）：100 个各向同性 BRDF，每个采样成 90×90×180 的立方体（$\theta_h$ 轴被 warp 过，让采样靠近镜面峰）。四条观察，每条后来都变成一个设计决定：

| 观察（图） | 现象 | 变成哪个决定 |
|---|---|---|
| **镜面尾**（图 9） | 绝大多数材料的镜面瓣**尾巴远长于** Beckmann / Blinn-Phong / Gaussian（三者在那个宽度上几乎一样，都拟合不了）；钢的峰值 **> 400** | **GTR**（$\gamma$ 可调）；主瓣 $\gamma=2$（= GGX/TR），清漆层 $\gamma=1$（= Berry） |
| **掠射回溯反射**（图 6） | 光滑材料在掠射角**变暗**（Fresnel 遮挡），粗糙材料**变亮**（回溯峰）；转折与 roughness 强相关 | 漫反射项的 $F_{D90}=0.5+2r\cos^2\theta_d$ |
| **Fresnel 形状**（图 10） | 多数材料在掠射处的抬升**比理论 Fresnel 更陡** | 这就是 Torrance-Sparrow 的"离镜面峰"；$G$ 与 $1/(4\cos\theta_l\cos\theta_v)$ 联合**放大**了 Fresnel |
| **方向反照率**（图 11） | 大多数材料前 70° 基本平，之后随 roughness 分化；**极少有材料 albedo 超过 0.3** | $G$ 的粗糙度 remap（见下） |

**"极少有材料 albedo 超过 0.3"这一句值得单独拎出来**：它是对"把 base color 画亮"这个习惯最直接的反驳，而且是**实测统计**，不是审美意见。

## Core Idea

**把"物理量"换成"归一化的感知量"，并且要求参数空间的每一个点都合法。**

五条原则（原文 §5.1）：

1. 用**直觉**参数，不用物理参数；
2. 参数**尽量少**；
3. 参数在合理范围内**归一到 0–1**；
4. 允许在合理范围外**继续推**（但要有意义）；
5. **所有参数组合都要尽可能稳健、尽可能可信**。

第 5 条是全部重量所在。它意味着模型要被设计成"**参数立方体里没有坏点**"——而这件事只能靠**逐项重参数化**做到，不能靠加约束。

## Technical Approach

### ① 十个标量 + 一个颜色

`baseColor`、`subsurface`、`metallic`、`specular`、`specularTint`、`roughness`、`anisotropic`、`sheen`、`sheenTint`、`clearcoat`、`clearcoatGloss`。

**`metallic` 的定义是全文最重要的一句话：**

> metallic: the metallic-ness (0 = dielectric, 1 = metallic). **This is a linear blend between two different models.** The metallic model has no diffuse component and also has a tinted incident specular, equal to the base color.

把它写成公式就是后来所有引擎里的那两行：

$$D=\mathrm{lerp}(C,0;M),\qquad S=\mathrm{lerp}(0.04,C;M)$$

**"metalness 是插值权重"这个理解，是 2026 年 [[2026-11-27-Fei — ReMeDi Single-Image PBR Material Estimation via Reparameterization|ReMeDi]] 的全部地基**：既然它是插值权重，那么两端必须是完整材质，中间值才成立。

### ② 三处"为了插值"的重映射

| 项 | 做法 | 理由（原文） |
|---|---|---|
| **specular → $F_0$** | 参数 0–1 线性映射到入射镜面反射率 $[0,0.08]$，对应 IOR $[1.0,1.8]$；中点 = 1.5 = 默认 | *real-world incident reflectance values are so unintuitively low*——直接给 0.04 美术不敢填 |
| **roughness → $\alpha$** | $\alpha=\text{roughness}^2$ | 感知线性；且"粗糙↔光滑插值永远得粗糙" |
| **roughness → $\alpha_g$**（阴影项） | 先把 $r$ 重映射到 $[0.5,1]$ 再平方：$\alpha_g=(0.5+r/2)^2$ | 艺术家反馈"小粗糙度时高光太烫" |

三处都是同一个动作：**把非线性的物理量拉直，让"中间值"落在人期待的位置**。

### ③ 漫反射：$F_{D90}$

$$f_d=\frac{\text{baseColor}}{\pi}\big(1+(F_{D90}-1)(1-\cos\theta_l)^5\big)\big(1+(F_{D90}-1)(1-\cos\theta_v)^5\big),\quad F_{D90}=0.5+2r\cos^2\theta_d$$

乘子在掠射处等于 $F_{D90}$，**落在 $[0.5,\,2.5]$**：光滑材料掠射**减半**（能量被 Fresnel 反射掉、进不去），粗糙材料掠射**涨到 2.5 倍**（从微表面侧面进出的回溯反射）。Burley 承认这是 empirical 的（"a novel empirical model for diffuse retroreflection"），但两条理由都硬：它匹配 MERL 的观察，且艺术家喜欢。

### ④ GTR：一个可调指数的分布族

$$D_{\rm GTR}=\frac{c}{(\alpha^2\cos^2\theta_h+\sin^2\theta_h)^\gamma}$$

$\gamma=1$ 是 Berry、$\gamma=2$ 是 Trowbridge-Reitz（= GGX）、$\gamma=3/2$ 恰好等价于 $\theta=2\theta_h$ 处的 Henyey-Greenstein 相函数。有闭式积分 ⇒ 可归一化、可重要性采样（附录 B）。**"可解析积分"是这条族的硬门槛**，也是它能进生产的原因。

### ⑤ §5.7 分层：为什么"中间值合法"是生产需求

这一段是全篇最被低估的：

> which parameters needed to be spatially varying, and the answer was **all of them**; if an artist simply wants to put two different materials on a surface and mask between them, then they will need to interpolate between all of the parameters. Also, **the mask will be filtered and at the blurred edge of the mask the material response must remain plausible.**

于是他们干脆把"插值"升级成架构：**Principled Layers** —— 一套材质 = 一叠预设 + mask 混合，像 Photoshop 图层。原文的评价是 *phenomenally successful, greatly simplifying workflow, improving material consistency*。

**这句话就是 2026 年 ReMeDi 要解决的问题的原始表述**：mask 边缘的 $M=0.37$ 必须是一份合法材质。Burley 用"归一化 + 处处可信"保证它；ReMeDi 用"两端完整 + 按覆盖率混合"保证它。**同一个问题，隔了十四年，两种证明方式。**

## Key Contribution

1. **一套"处处合法"的参数化**（1 色 + 10 标量），使线性插值在材质空间里是安全的。
2. **`metallic` 作为两个完整模型之间的插值权重**——这是今天所有 MR 工作流的定义。
3. **GTR 分布族**（$\gamma$ 可调）与三处感知重映射。
4. **Principled Layers**：把"分层 + mask"变成主工作流，而不是打补丁。
5. **BRDF Explorer**（开源）：一个"用图像切片看 BRDF"的工具，把"读数字"换成"看图"。

## Why It Works

**因为"所有组合都合法"这个要求，等价于"参数空间的每一点都要能被渲染出来"，而这件事只能靠逐项重参数化，不能靠约束。**

看它的代价结构就明白：

- 换成归一化参数 ⇒ 失去"参数即物理量"的可测性（$F_0$ 要从 specular 反算）；
- 允许超范围 ⇒ 失去"物理上不可能"这个保护；
- 用 empirical 的 $F_{D90}$ ⇒ 失去"从第一原理推导"的资格。

**它主动放弃了"每个参数都可测"，换来了"每个组合都可用"。** 对生产而言这笔交易是赚的：美术不需要知道 $F_0$，他需要知道"这排滑块怎么推都不会崩"。

第二个"为什么"在 §5.7：**一旦每个点都合法，材质就可以被当作向量来插值**——于是分层、mask、预设库、mip、抗锯齿全部自动成立。**这是把"美术规范"变成了"数学性质"。**

## Limitations

1. **它是课程讲义，不是同行评议论文。** 27 页里有 8 页是 BRDF Explorer 的附录文档；核心方法部分约 12 页。
2. **没有一张定量误差表。** 全篇的"拟合得好"是**可视化证据**（BRDF 切片、点光源响应、albedo 曲线）+ **一部电影的生产证据**。想引用"误差多少"的人会失望。
3. **GTR 的 $\gamma$ 是自由参数，但没有给拟合流程。** 原文只说"preliminary fitting results suggest typical values of $\gamma$ between 1 and 2"。
4. **只有各向同性 + 一个各向异性旋钮**；没有光谱、没有偏振、没有衍射。
5. **清漆层不扣基底能量。** 原文的理由是"能量很小"（*represents a relatively small amount of energy so we don't subtract any energy from the base layer*）——但"多小"没量化。
6. **sheen / subsurface 都是经验项**，后者明确不是次表面输运的替代（"won't bleed light into the shadows or through the surface"）。
7. **MERL 数据本身的偏差没被讨论**：$\theta_h$ 轴被 warp、掠射数据是外推的、次表面效应被烘进去了（图 15 的自陈）。所以"拟合 MERL"这件事的上限本身有限。

## Game Development Relevance

**这是你的材质系统的语法书。** 今天 UE 的 Lit shading model、Unity 的 Lit、glTF 的 `pbrMetallicRoughness`、USD 的 `UsdPreviewSurface`，参数集都能追到这一页。

1. **`metallic` 的语义**：它是**插值权重**（覆盖率），不是"金属程度"。这一条直接决定"$M=0.5$ 该长什么样"，也决定 mip 与抗锯齿该怎么想。
2. **0.04 的出处**：$[0,0.08]$ 的中点，同时=I OR 1.5 的精确 Fresnel 值。见 [[Metallic-Roughness Workflow]]。
3. **$\alpha=R^2$**：为什么 UE 的 Roughness 滑块在低端"手感很细"——那是感知重映射，不是物理。
4. **"albedo 极少超过 0.3"**：可以直接拿去做材质审查的红线。
5. **分层的合法性**：Substrate / 多层 Lit 的每一层都要满足"中间值合法"，否则 mask 边缘会露馅。

## Unreal Engine Relevance

- **UE4 的 Shading Model 就是 Karis 2013 对本文的实时化**：D 用 GGX（GTR $\gamma=2$）、G 用 Smith + Schlick 形式重映射（$k=(\alpha+1)^2/8$）、F 用 Schlick + 球面高斯、环境光走 [[Split-Sum Approximation]]。
- **三处可直接对照的滑块**：`Metallic`（本文的 $M$）、`Roughness`（本文的 $r$，UE 里已是 $\alpha=r^2$ 之后的语义）、`Specular`（默认 0.5 = 本文 specular 参数的中点 = $F_0=0.04$）。
- **Substrate 是本文 §5.7 的直系后代**：分层 + 每层自己的参数 + 层间 mask。**Burley 担心的"mask 边缘必须合法"，在 Substrate 里是逐层都要满足。**
- **一个可做的检查**：把 `Specular` 从 0.5 拉到 1.0（$F_0=0.08$），看有多少材质"突然像塑料"。**能看出差别的那些，就是 $F_0\neq0.04$ 的材质。**

## Technology Evolution

```text
2003  MERL 100              第一次有 100 个实测 BRDF 可查
        ↓
2012  Burley / Disney       归一化参数 + metallic 插值 + GTR + Principled Layers
        ↓                   《无敌破坏王》：look-dev 美术提前下线（"material re-do" 消失）
2013  Karis / UE4           实时化；MR 成为引擎默认
        ↓
2015+ glTF / USD / OpenPBR  MR 成为交换格式
        ↓
2017  Hammon / Kulla-Conty  能量侧修补（漫反射 half-loss + 多次散射）
        ↓
2026  ReMeDi                "端点必须完整"被显式建模 ⇒ 中间值按构造正确
```

**这条线四十年只做了一件事：把"材质"从"一组物理量"变成"一个可以被插值的向量空间"。**

## Relationships

### Based On

- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] —— 微面框架（$D\cdot G\cdot F$）；本文换的是 D 的族与全部参数化，不是框架。
- [[Microfacet Theory]] —— GTR 只是 $D$ 的一族；$G$ 用 Smith，$F$ 用 Schlick。
- [[Dana 1999 — Reflectance and Texture of Real-World Surfaces|Dana 1999]] —— 与 MERL 同源的"实测优先"传统：先有数据，再有模型。

### Extends

- [[Metallic-Roughness Workflow]] —— 本文是它的出生证明；那一页把"$M$ 是覆盖率""0.04 的出处""三处交叉失效"整理成可操作清单。

### Related

- [[Physically Based Rendering]] —— 本文是它谱系表里"生产参数化"那一格。
- [[Karis — Real Shading in Unreal Engine 4 (2013)]] —— 直接下游，进引擎的那一步。
- [[Split-Sum Approximation]] —— $F_0$ 在环境光里被预积分成 LUT；这使"改 $F_0$"在实时里是免费的。
- [[Multiple Scattering and Energy Compensation]]、[[Hammon — PBR Diffuse Lighting for GGX+Smith Microsurfaces (2017)]] —— 本文留下的两册能量账，2016–2017 才补上。
- [[BRDF]] —— 本文在那一页的谱系里是"生产参数化"一格。
- [[2026-11-27-Fei — ReMeDi Single-Image PBR Material Estimation via Reparameterization|ReMeDi 2026]] —— 十四年后，把 §5.7 那句"mask 边缘必须合法"做成了可训练的约束。

### Followed By

- 所有 MR 工作流的实现（UE / Unity / glTF / USD / OpenPBR）
- [[2026-11-27-Fei — ReMeDi Single-Image PBR Material Estimation via Reparameterization|ReMeDi 2026]] —— 单图反解侧的第一次"按构造保证一致"

## Personal Knowledge State

- **Easy**：MR 工作流的操作层；`Metallic` / `Roughness` / `Specular` 三个滑块在哪；为什么金属没有漫反射。
- **Normal**：五条原则里第 3、5 条的分量（"处处合法"是插值的前提）；三处感知重映射各自解决什么问题；$F_{D90}$ 的两端（0.5 / 2.5）分别对应什么物理现象；"albedo 极少超过 0.3"。
- **Hard**：GTR 的归一化与重要性采样推导（附录 B）；$G$ 与 $1/(4\cos\theta_l\cos\theta_v)$ 如何联合放大 Fresnel（§4.3 的论证）；MERL 拟合的实际过程（$\gamma$ 怎么定）。

## Learning Value

**三个可迁移动作：**

1. **设计任何"给美术的接口"时，先问"参数空间的每个点都合法吗"**。合法 ⇒ 可以插值、可以分层、可以被 mip 平均；不合法 ⇒ 你必须禁止中间值，而管线迟早会造出中间值（mask 过滤、LOD 混合、动画过渡）。
2. **"物理正确"与"可调"是两个目标，冲突时要明说放弃哪个**。Burley 明说放弃前者的一部分，换来的是"美术提前下线"——**这是一个可被验收的收益，而"更物理"不是。**
3. **实测统计比审美意见有力**。一句"极少有材料 albedo 超过 0.3"，比十页"要注意能量守恒"更能改变一个团队的习惯。

## Visualization

![[Research/Files/Pictures/Burley 2012 — 图16 十个归一化旋钮的扫描.png]]

**读图**：十行 = 十个标量参数，每行从 0 扫到 1，其余参数固定。**这张图就是"处处合法"的证据**：整张网格里没有一个球是"坏的"——没有负能量、没有黑斑、没有爆炸的高光。注意三行：`metallic` 从黄（介质，漫反射为主）平滑过渡到金（金属，有色镜面），中间那几个球**都是有意义的材质**（像"半金属漆"），这正是 ReMeDi 说的"中间值是覆盖率"；`roughness` 从镜面到哑光，中间没有"突然变糙"的跳变（$\alpha=r^2$ 的功劳）；`specular` 整行几乎看不出变化——因为 $F_0$ 从 0 到 0.08 在漫反射主导的材料上确实不显眼，**这也解释了为什么美术会乱推这个滑块**。（图 16，Burley 2012）

![[Research/Files/Pictures/Burley 2012 — 图19 金与蓝橡胶之间的插值.png]]

**读图**：11 个球，从"高光金属金"线性插值到"蓝色橡胶"，**逐参数线性插值**——没有用任何混合技巧。中间那五个球不是"半金半胶"的怪物，而是**每一种都像真实材料**：暗金 → 古铜 → 灰金属 → 深灰塑料 → 深蓝塑料。下面那条渐变带是同一组参数铺在一个平面上（mask 混合的连续版本）。**这张图是 §5.7 的全部论据**：mask 模糊边缘上会出现 $M=0.37$ 这样的值，而它必须是这种"像材料"的东西。十四年后 ReMeDi 的 $C=\mathrm{lerp}(C_d,C_m;M)$ 就是在**显式构造**这张图里的每一个球。（图 19，Burley 2012）

## Notes

原文 PDF 已归档：`Research/Files/PDF/Burley 2012 - Physically Based Shading at Disney.pdf`（4.6 MB，27 页含附录）。

- **版本说明**：这是 course notes **v3**（2012-08 首发，后有修订）。引用时标"course notes"而不是"paper"，因为它的证据结构与论文不同（可视化 + 生产，不是对照实验）。
- **库内此前只有二手**：[[BRDF]] 的谱系行写着 "Disney Principled 2012 → Karis UE4 2013"，但 2012 那端一直是纯文本；[[Karis — Real Shading in Unreal Engine 4 (2013)]] 在库内已读，**上游缺失**。今天补上，PBR 谱系的 2012–2013 两格终于都是链接。
- **一个容易被忽略的贡献**：BRDF Explorer（开源）把"看 BRDF"从读表变成看图。**工具本身就是论证的一部分**——没有它，§3–§4 那些观察无法被读者复核。
- 与 [[Hammon — PBR Diffuse Lighting for GGX+Smith Microsurfaces (2017)]] 的分工：Burley 的漫反射是**经验拟合**（$F_{D90}$），Hammon 是**从 GGX+Smith 同源推导**（"最多丢一半"）。**同一个现象，两种合法性来源。**

**相关每日记录**：[[2026-11-27]]
