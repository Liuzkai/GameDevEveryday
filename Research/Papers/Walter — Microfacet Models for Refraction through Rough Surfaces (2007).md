---
type: paper
title: "Microfacet Models for Refraction through Rough Surfaces"
authors: [Bruce Walter, Stephen R. Marschner, Hongsong Li, Kenneth E. Torrance]
year: 2007
published: "2007-06-25"
venue: "EGSR'07 — 18th Eurographics Symposium on Rendering, pp. 195-206"
url: "https://doi.org/10.2312/EGWR.EGSR07.195-206"
code: ""
project_page: "https://www.cs.cornell.edu/~srm/publications/EGSR07-btdf.html"
category: [rendering, microfacet-theory, brdf, bsdf]
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal
status: reading
---

# Walter et al. 2007 — Microfacet Models for Refraction through Rough Surfaces

> 江湖名号：**GGX 论文**。你每天在 UE 材质里用的那个 D 项，就是这篇里的。

## TL;DR

一篇"以为在讲玻璃、结果改写了整个实时渲染"的论文。作者把微面理论从反射推广到**折射**（粗糙透射），用真实粗糙玻璃的实测数据做验证，为了拟合实测顺手**引入了一个新的法线分布，命名为 GGX**——它的长尾比 Beckmann 更贴近真实材质。论文同时系统讨论了 shadowing-masking（后来的 Smith height-correlated）的选择和重要性采样。

今天 UE / Unity / Filament / glTF 的默认 D 项全是 GGX。**这是实时 PBR 里被用得最多、但被读得最少的一篇。**

## Problem

微面模型在**粗糙反射**上非常成功，但**粗糙透射**（磨砂玻璃、蚀刻玻璃、次表面透光）缺少经过物理验证的模型：

- 界面光滑时用 Snell 定律即可；
- 界面粗糙时，光走哪个方向、能量怎么分布，当时没有既物理正确又被实测验证过的 BSDF；
- 更要命的是：**透射光至少要穿过两个界面**，路径比反射长得多，如果重要性采样不好，蒙特卡洛直接不可用。所以"能采样的完整模型"是刚需，不是锦上添花。

## Historical Context

```text
Torrance-Sparrow 1967（物理光学，微面理论原型）
        ↓
Blinn 1977（引入计算机图形学）
        ↓
★ Cook-Torrance 1981/1982：D · G · F 齐备，能量守恒，完整物理框架
        ↓
★ 本文 2007：把 half-vector 泛化以统一反射与折射；引入 GGX；系统讨论 G 与重要性采样
        ↓
Disney Principled BRDF 2012（生产参数化）
        ↓
★ Karis 2013（SIGGRAPH Course：Real Shading in UE4）→ GGX + Smith + split-sum 进引擎
        ↓
全工业默认（UE / Unity / Filament / glTF / Filament 一致收敛到 GGX + 金属度工作流）
        ↓
2026：[[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]] 从随机几何把 GGX 重新推导为特例
```

**一个值得记住的细节**：本文作者之一 **Kenneth E. Torrance 就是 Cook-Torrance 的 Torrance**。1981 年他提出框架，26 年后他署名把它推广到折射——这不是巧合，是这条谱系真的在同一个实验室（Cornell Program of Computer Graphics）里延续。

## Core Idea

微面理论的核心假设：粗糙表面 = 大量完美镜面的微小面片，其法线服从某个分布 $D(m)$。反射时，只有法线恰好等于半程向量 $h$ 的微面能贡献。

本文的关键推广是**把半程向量泛化**，使同一个机制既能处理反射（mirror-like 微面反射）又能处理折射（mirror-like 微面折射），从而得到**一个完整解析的 BSDF**（BRDF + BTDF 一体）。

## Technical Approach

1. **泛化半程向量**：用一个统一的 half-vector 构造同时覆盖反射与折射几何；
2. **完整解析 BSDF**：给出反射与透射两侧的全部方程（论文明说目标之一是"作为 implementor 的自包含参考"）；
3. **实测验证**：测了四种真实粗糙透射表面，发现粗糙透射有明显的行为——透射峰值会**从光滑折射方向向掠射角显著偏移**（类似粗糙反射的 off-specular 峰），微面模型能预测这个效应；
4. **引入 GGX**：为满足某些表面的拟合，提出一个新的法线分布，长尾比 Beckmann 长；
5. **重要性采样**：给出微面模型的高效采样方案与对应 PDF——透射必须穿两个界面，采样效率是生死问题。

## Key Contribution

| 贡献 | 当时的意图 | 后来的实际影响 |
|---|---|---|
| 粗糙折射 BTDF | 主目标（论文标题） | 影响中等：实时里很少算真折射 |
| 泛化 half-vector | 手段 | 成为标准写法 |
| **GGX 分布** | 顺手提出，为了拟合部分实测表面 | **改变行业**：成为实时 PBR 的默认 D 项 |
| Smith / height-correlated masking-shadowing 的系统讨论 | 实用性讨论 | 成为实时 G 项标准 |
| 重要性采样方案 | 离线刚需 | 后来实时 IBL 的 split-sum 也受益于同一套思路 |

**论文标题和它的历史地位完全错位**：它以为自己在解决磨砂玻璃，实际上它解决了"实时材质用什么 D"。

## Why GGX 赢了

Beckmann 是从**高斯高度场**推出来的，高光衰减快（尾部短）。真实材质（尤其是磨砂/车漆/布料的掠射表现）有**更长的尾**——高光中心之外还有一圈明显的"辉光"。

GGX 的分布（在光学中即 Trowbridge–Reitz 1975 的结果，Walter 等人引入图形学并命名为 GGX）衰减更慢：

$$D_{GGX}(h) = \frac{\alpha^2}{\pi\left((n\cdot h)^2(\alpha^2-1)+1\right)^2}$$

长尾带来的直接后果：

- 粗糙度变化时高光形状更"自然"，不会出现 Beckmann 那种"从点状突然炸开"；
- 掠射角有更合理的能量分布；
- **美术更好调**——Roughness 滑杆的手感更线性、更可预测。

最后一条才是它真正赢的原因：GGX 不是"更物理"所以赢了，是"更物理 + 更好调"同时成立。

## 与 UE / 实时 PBR 的对应

[[Physically Based Rendering]] 里那张"实时化的关键近似"表，来源就在这里：

| 项 | 本文的角色 |
|---|---|
| **D** | 直接提供 GGX（UE 默认） |
| **G** | 系统讨论 shadowing-masking；Smith height-correlated 形式成为实时标准 |
| **F** | 不是本文贡献（Schlick 1994），但本文把它放进统一框架 |

UE 里那个 `D_GGX * Vis_SmithJoint * F_Schlick` 的三件套，**D 和 G 两项都追溯到这篇**。

## Smith 独立性假设 —— 以及它今天的地位

Smith 的 masking-shadowing 把"被遮挡"和"被遮蔽"当成两个可以相乘的独立事件：

$$G(l,v,m) = G_1(l,m)\,G_1(v,m), \qquad G_1(s,m) = \frac{1}{1+\Lambda(s)}$$

其中对 GGX：

$$\Lambda(s) = \frac{-1+\sqrt{1+\alpha^2\tan^2\theta_s}}{2}$$

这个"独立"是一个**假设**：微面高度之间无相关性。它便宜、好用、能量守恒，但物理上并不严格。

**今天（2026-09-16）有一件事值得你知道**：[[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]] 证明了在 **height-field 极限**下，一个更一般的局部条件近似**退化为 Smith 的独立性假设**。也就是说——

> Smith 的独立性不再只是一个"为了算得动而做的假设"，它有了一个明确的理论位置：**它是某个更一般理论在高度场极限下的结果。**

读到这里你会得到一个非常实用的判断框架：以后看到任何"改进 G 项"的论文，第一句话就该问——**它放松了 Smith 的哪个假设？**

## Limitations

- **单次散射**：经典微面模型只算单次，粗糙度越高能量损失越明显（"高粗糙度变暗"）。多次散射补偿是后来的工作（Heitz et al. 2016 的多重散射 GGX / Imageworks 的 energy-preserving 修正），本文没有；
- **Smith 假设**：高度与斜率不相关，见上；
- **GGX 各向同性**：各向异性需要额外参数化（后来的 GTR / 各向异性 GGX）；
- **折射部分**在实时里基本没用上——实时仍然用近似透射（如 UE 的 thin translucent / subsurface approximations），不是本文的 BTDF。

## Game Development Relevance

**5 / 5**。没有任何一篇渲染论文比这篇更"天天在用"：

- UE 默认 Lit 的 D 项 = GGX；
- Unity / Filament / glTF / three.js 全部一致；
- 材质 Roughness 的手感、掠射能量、环境镜面的 spread，全部由这个分布决定。

你做的**性能预算**也和它有关：D·G·F 的求值开销是**每像素固定**的，多一个 lobe（Clear Coat、Cloth、Hair）就是乘一份——所以"Shading Model 数量"是一个隐藏的预算维度，与 DrawCall 并列。理解 D·G·F 到底是什么，才能判断"这个材质特性值不值这份开销"。

## Unreal Engine Relevance

- **Material / HLSL**：`D_GGX` / `Vis_SmithJointApprox` / `F_Schlick` 就在引擎 shader 源码里，可以对照着读；
- **Material 预算**：见上，lobe 数 = 着色成本乘数；
- **Substrate（UE 5.2+）**：分层 lobe 框架，本质是把"一个 GGX lobe"变成"多个 lobe 的组合"——GGX 仍是底层积木。

## Relationships

### Based On

- Torrance-Sparrow 1967（微面理论原型）
- Blinn 1977（引入 CG）
- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] — **且本文作者之一即为 Torrance 本人**
- Smith 的 masking-shadowing 原始工作

### Extends

- 把微面理论从 BRDF 扩展到完整 BSDF（含 BTDF）

### Related

- [[Microfacet Theory]] — 本文是它的现代标准形态
- [[BRDF]] — 本文是 BRDF 家族里被用得最多的具体实现
- [[Physically Based Rendering]] — 本文贡献了实时 PBR 的 D 与 G 两项
- [[Kajiya — The Rendering Equation (1986)]] — 容器；本文填的是里面的 $f_r$
- [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]] — 45 年后把本文的 D 与 G 重新推导出来

### Followed By

- Disney Principled BRDF 2012
- Karis 2013（UE4 Real Shading）→ 工业普及
- Heitz 2014《Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs》→ 把 G 项讲透
- 多次散射 GGX 修正（2016 前后）

## Personal Knowledge State

**Normal，正在读**。你目前研读 Cook-Torrance 的 D/G/F 物理来源——这篇是这条线上的**第二个锚点，而且是离你日常最近的一个**：

- Cook-Torrance 1981 给你**框架**（为什么要 D·G·F）；
- **本文给你"你实际用的是哪一个 D、哪一个 G"**。

## Mastery 自测（5 条）

能通过这 5 条，[[Microfacet Theory]] 可以从 Normal 标 Easy：

1. **为什么需要 D？** 说出 $D(m)$ 的物理含义与归一化条件（$\int D(m)(n\cdot m)\,d\omega_m = 1$），并解释为什么是投影面积的积分而不是普通积分。
2. **GGX 凭什么赢？** 解释"长尾"在视觉和美术调参上的具体后果，并说清它相对 Beckmann 的取舍。
3. **G 项在补什么？** 解释 masking 与 shadowing 的区别，以及为什么没有 G 项时掠射角会整体偏亮（能量不守恒）。
4. **Smith 的假设是什么？** 说出"高度与斜率不相关"，并说明它在 2026 年被放到了什么理论位置（height-field 极限）。
5. **映射到 UE**：指出 UE 默认 Lit 的 D / G / F 分别是什么，并说明多加一个 lobe 对每像素着色开销的影响量级。

## Learning Value

- 这篇论文的**读法建议**：跳过折射（BTDF）那一半，只看 Section 关于分布与 shadowing-masking 的讨论。你的目标不是磨砂玻璃。
- 读完你会获得一个**评估新着色技术的基准姿势**：任何号称"更好的高光"的工作，先问它改的是 D、G 还是 F，代价是多少 ALU。

## Visualization

![[Microfacet D·G·F 几何图解.html]]

## Notes

- 引用已核实：EGSR'07: Proceedings of the 18th Eurographics conference on Rendering Techniques, pp. 195–206, published 2007-06-25, DOI 10.2312/EGWR.EGSR07.195-206（Eurographics Digital Library / ACM DL 一致）。
- 作者机构：Cornell University, Program of Computer Graphics（Kenneth E. Torrance 为 Cook-Torrance 1981/1982 的作者之一）。
- PDF 公开可读：https://cseweb.ucsd.edu/~viscomp/classes/cse168/sp25/readings/EGSR07-btdf.pdf
- 本文入选今日经典线的原因：你正在研读 D/G/F，而**你实际在用的 D（GGX）和 G（Smith）就出自这篇**。它是知识图谱里"最常用却缺失"的那个节点。
