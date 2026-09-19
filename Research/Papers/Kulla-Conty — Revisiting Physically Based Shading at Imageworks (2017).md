---
type: paper
title: "Revisiting Physically Based Shading at Imageworks"
authors: [Christopher Kulla, Alejandro Conty]
year: 2017
published: "2017-08 (SIGGRAPH 2017 Course)"
venue: "SIGGRAPH 2017 — Physically Based Shading in Theory and Practice (course)"
url: "https://blog.selfshadow.com/publications/s2017-shading-course/imageworks/s2017_pbs_imageworks_slides_v2.pdf"
code: ""
project_page: "https://blog.selfshadow.com/publications/s2017-shading-course/"
category: [rendering, brdf, energy-conservation, multiple-scattering, pbr, production]
importance: S（经典）
historical_importance: 4
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal（研读中）
status: read
aliases: [Kulla-Conty 2017, Kulla Conty, Energy Compensation, 微面能量补偿, fms, Imageworks 2017]
tags: [rendering, brdf, microfacet, energy-conservation, multiple-scattering, pbr, production]
---

# Revisiting Physically Based Shading at Imageworks

> **一句话定位：它就是"高粗糙度为什么发闷"这个问题的工程答案——把单次散射丢掉的能量，用一块 4KB 的小表乘回去。**
>
> 这是 [[Physically Based Rendering]] 的 Learning Gap 里**最后一条具名缺口**。原文已下载逐节核对（139 页 slides 含 speaker notes 与 errata；关键页为 8–20、附录）。

## TL;DR

- 动机是**生产事故**，不是学术兴趣：美术为了压住能量，常见做法是**在单个球面上堆多个独立 lobe**，结果是**掠射角处凭空造能量**；另一个现象是有美术把木材的 IOR 调到 **100+**（物理无意义但"看着还行"）。
- 核心手法：微面 BRDF $f$ 在积分后**必然丢能量**（$E(\mu)<1$，忽略微面间互反射）。补法是**加一个新的 lobe**：

$$f_{ms}(\omega_o;\omega_i)=\frac{\bigl(1-E(\mu_o)\bigr)\bigl(1-E(\mu_i)\bigr)}{\pi\,\bigl(1-E_{avg}\bigr)},\qquad E_{avg}=2\!\int_0^1\! E(\mu)\,\mu\,d\mu$$

- **它能成立的原因极简单、且不依赖任何 BRDF 假设**：算 $f_{ms}$ 的方向 albedo，中间积分恰好就是 $E_{avg}$ 的定义，与分母相消，剩下 $1-E(\mu_o)$ —— **正好是 $f$ 缺的那部分**。原文（speaker notes）特意点出："This formula is stated without proof in the Kelemen paper... I haven't made any particular assumption about the BRDF. This method always works."
- **工程化的三个数字**（对你的预算维度直接有用）：$E$ 用 **32×32** 表够用，float 存下来**"just 4Kb"**；$E_{avg}$ 再用一张 **32 项** 1D 表按粗糙度索引；各向异性用一套参数化**直接忽略掉**。
- **一个被作者自己标为 errata 的点**：$F_{avg}$ 那一条公式有笔误（由 Emmanuel Turquin 指出），附录给了更好的近似。**引用时要走 v2 slides 而不是记忆。**

## Problem

"能量守恒"和"能量保持（energy preservation）"在原文里被分成两件事，这是本篇最容易被忽略的一个区分：

- **能量守恒（conservation）**：不凭空造能量；
- **能量保持（preservation）**：一个本该几乎全反射的材质（如白色塑料），其 albedo 应**接近 1**。

生产里两个都坏掉了：

1. **堆 lobe → 掠射造能量**。这是"物理正确"被当成"可以叠加"的典型误用；
2. **单次散射 → 高粗糙度丢能量**。这是更隐蔽的那一个：**因为它是"偏暗"而不是"爆亮"，在美术流程里通常被当成"美术没调好"。**

原文的两句话值得原样记住：

> "What was more concerning, however, was that adding multiple independent lobes was leading to excess energy being created, particularly at grazing angles."

> "…we wanted to ensure that our materials would always be energy conserving, even in layered cases…"

## Historical Context

```text
Kelemen & Szirmay-Kalos 2001（第一个通用解：预计算表）
       —— 只讨论塑料；没有直接处理"参数可变"，也没有透射
            ↓
Jakob et al. 2014（综合的分层材质框架）★ 同时也是 F_avg 那条思路的来源
       —— 框架完整但太重，对纹理化材质不实用
            ↓
Heitz et al. 2016（Smith 模型下多次散射的随机"真值"）
       —— 不需要预计算，但求值与采样都要随机数 → 不适配 raster 架构
            ↓
★ Kulla & Conty 2017（把 Kelemen 的公式"蒸馏"到可落地）★ 今日入库
       —— 4KB 表 + 解析 F_avg 拟合 + 忽略各向异性
            ↓
Fdez-Agüera 2019（JCGT，实时 IBL 版的多散射微面模型；被多家引擎采用）
            ↓
Dupuy 2026（[[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]]）
       —— 一个无参数 NDF 上，所有散射阶的精确闭式（理论答案）
```

**一条诚实的历史说明必须写进笔记**：Kulla-Conty 的补偿公式**不是他们发明的**，原文自己写明是"the technique presented in the Kelemen paper can be distilled down to the following formula"。**Kulla-Conty 的贡献是"把 Kelemen 的公式做成可用的工程方案"**（小表 + 平均 Fresnel 解析拟合 + 透射拆分）。做归属判断时别搞错。

## Core Idea

### 1. 先能量化"丢了多少"

定义**方向 albedo**

$$E(\mu_o)=\int_0^{2\pi}\!\!\int_0^1 f(\mu_o;\mu_i,\phi)\,\mu_i\,d\mu_i\,d\phi$$

若微面**纯反射**，则 $E=1$ 才是对的。**GGX 在高粗糙度与掠射角处 $E$ 明显小于 1** —— 这就是 furnace test（把表面放进均匀光源，理想情况应完全消失）里的那一圈"灰暗"。**furnace test 是一个可以直接在引擎里复现的自测**：均匀环境光 + 只有镜面项的材质，粗糙度扫一遍，看是否整体发白。

### 2. 再造一个 lobe 把差值填上

$$f_{ms}=\frac{(1-E(\mu_o))(1-E(\mu_i))}{\pi(1-E_{avg})}$$

**为什么这个形式能精确补齐**（原文第 11 页把推导完整走了一遍，这是本篇最有价值的 5 行数学）：

$$E_{ms}(\mu_o)=2\pi\!\int_0^1\!\frac{(1-E(\mu_o))(1-E(\mu_i))}{\pi(1-E_{avg})}\mu_i\,d\mu_i=\frac{1-E(\mu_o)}{1-E_{avg}}\underbrace{\int_0^1(1-E(\mu_i))2\mu_i\,d\mu_i}_{=\,1-E_{avg}}=1-E(\mu_o)$$

**要点三条**：
- 分子与被积函数**完全相同的因子结构**，所以 $(1-E(\mu_o))$ 能提出积分；
- 剩下的积分**按定义就等于** $(1-E_{avg})$，**与分母相消**；
- 结果 $1-E(\mu_o)$ **恰好等于原 BRDF 缺的那一块**。且 $f_{ms}$ **本身是对称的（互易）**。

### 3. 把"贵"的部分变成"小"

$E$ 是一个积分，没有闭式。但需要**空间可变**（roughness / anisotropy / IOR 都要贴图），全参数组合不可能穷举。三条工程化处理：

| 项 | 处理 | 成本 |
|---|---|---|
| **粗糙度** | **32×32** 表就够（float 存**"just 4Kb"**）；$E_{avg}$ 另用 **32 项 1D 表** | 极低 |
| **各向异性** | **"using a parameterization that lets us ignore it!"** —— 在选定参数化下各向异性不需要额外维度 | **0** |
| **Fresnel** | 找 **$F_{avg}$ 的解析拟合**（比查表粗一点，但是 0 采样） | 极低 |

### 4. Fresnel（吸收/透射的情形）

一旦表面吸收或透射，$E(\mu)<1$ 不再是"丢能量"。原文沿 Jakob 2014 的假设：

> **"Multiply scattered energy is diffused"** —— 多次散射的能量被打散成漫射，于是可以用**余弦加权平均 Fresnel**

$$F_{avg}=2\!\int_0^1\! F(\mu)\,\mu\,d\mu$$

再用**几何级数**把"在各次微面弹射中的反射"累加：

$$F_{avg}E_{avg}\sum_{k=0}^{\infty}F_{avg}^{\,k}(1-E_{avg})^{k}=\frac{F_{avg}E_{avg}}{1-F_{avg}(1-E_{avg})}$$

这个因子**直接乘在 $f_{ms}$ 上**。

> ⚠️ **原文标注的 errata**：这一条式子（第二式）**原版有误，由 Emmanuel Turquin 指出**，v2 slides 与附录给了更好的近似。所以本文的引用**必须"确认是 v2"**。

## Key Contribution

1. 把 Kelemen 2001 的通用解**做成生产可用**：3 个低成本手段（32×32 表 / $F_{avg}$ 解析拟合 / 各向异性参数化消掉维度）；
2. **把证明补上**（Kelemen 原文只给公式、不给证明）—— 那条 5 行的相消推导，是工程师真正能"信"的部分；
3. **把"能量守恒"从审美问题变成可测量问题**（$E(\mu)$ / furnace test）；
4. **处理了透射**：把补能量拆成 $f_{ms}^R$ 与 $f_{ms}^T$，用一个 Ratio 分配反射/透射份额。

## Why It Works

- 它**没有在解输运方程**，它在**解一个"账平"问题**：$f$ 少了 $1-E$，那就造一个方向 albedo 恰为 $1-E$ 的 lobe 加回去。**不需要知道多次散射在物理上究竟怎么分布。**
- 代价与收益同时被这个视角解释清楚：**账平了，但分布是"漫射状假设"下的近似** —— 真实多次散射在掠射端会更集中（这正是 Dupuy 2026 给出的形状）。

## Limitations

1. **$\bar F$ 的漫射假设**：把多次散射当成漫射（Lambert 状）传播。这是全部近似误差的来源所在；
2. **只补"方向 albedo"这一个标量**，不保证角分布正确；
3. **各向异性被"参数化消掉"**，不是被解决；
4. **透射侧的 Ratio 是启发式的**；
5. 需要额外的 LUT 资源与一次额外采样 —— 移动端的小成本但不是 0。

## Game Development Relevance

- **它是"高粗糙度发闷"的标准解释与标准解**。你如果在引擎里看到 rough 金属/塑料在高粗糙度端明显发暗、颜色失饱和，先问"这个渲染器做没做多次散射补偿"；
- **对分档的直接含义**：
  - 补能量是**一块 4KB LUT + 一次额外采样**的固定开销 —— **这是一个典型"低成本档位开关"**：PC_High/Android_High 开、Android_Low 关（关掉的可见后果就是高粗糙度偏暗，属于"降档可忍受"的视觉退化类型）；
  - 与 [[Split-Sum Approximation]] 相加，**IBL 镜面半边的完整固定开销 = cubemap 预滤波 + EnvBRDF LUT + 能量补偿表（+1 次采样）**。**这条加法是你做 IBL 相关分档时应该能背下来的账。**
- **对资产规范的含义**：原文那条"IOR 100 的木头"是**资产侧不物理**的典型案例。**你的 SABC 分级里可以加一条资产级约束：IOR/金属度/F0 的允许范围写进取值表** —— 这条比性能预算更容易漏，但同样会造成档位间行为不一致。

## Unreal Engine Relevance

- **UE 的多次散射补偿不是默认全开**。这是本篇对你最贵的一条信息：**引擎"支持"某个物理特性 ≠ 引擎"默认"给你**。遇到能量相关的观感问题，先查引擎侧是否开启/是否在低档位关掉；
- **一个可做的小实测（不需要引擎改动）**：建一个纯金属圆球，Roughness 扫 0→1，放纯环境光（无方向光），截图对比"是否在粗糙端整体变暗"；再打开/关闭引擎的多次散射补偿，看差异。**这是 25 条 PBR 自测清单里唯一的"引擎侧实测题"**；
- 与 **Substrate** 的关系：Substrate 与本文是同一目标（能量守恒、分层）的下一代实现 —— 它把"补能量"内建在分层框架里，而不是"事后加一个 lobe"。

## Technology Evolution

```text
2001 Kelemen（有解但不可用）
   ↓
2014 Jakob（完整但太重）
   ↓
2016 Heitz（真值但随机）
   ↓
★ 2017 Kulla-Conty（可用 + 被证明正确）★ 今日入库
   ↓
2019 Fdez-Agüera（实时 IBL 版，被多家引擎采用）
   ↓
2026 Dupuy（无参数 NDF 上的精确闭式 —— 理论天花板）
```

## Relationships

### Based On

- **Kelemen & Szirmay-Kalos 2001**（*A Microfacet Based Coupled Specular-Matte BRDF Model with Importance Sampling*, Eurographics Short Presentations）—— **公式的直接来源**，Kulla-Conty 的工作是蒸馏与工程化；
- **Jakob et al. 2014**（*A Comprehensive Framework for Rendering Layered Materials*, TOG）—— $F_{avg}$ 与"多次散射是漫射的"这个假设的来源，也是"分层"这条线的源头。

### Improves / 对照

- **Heitz et al. 2016**（*Multiple-scattering Microfacet BSDFs with the Smith Model*, TOG）：原文把它明确定性为**"found ground truth but only as a stochastic model"**，并给出弃用理由 —— **"Requires many random numbers to sample and evaluate / Poor fit for our rendering architecture"**。这是**"物理更真 ≠ 工程更可用"的一个教科书级案例**；
- **[[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]]**：🔴 **同一问题的两端** —— Kulla-Conty = 近似 + 全粗糙度可用；Dupuy = 精确 + 无粗糙度参数。**今天的核心价值就是把这两篇并排读。**

### Related

- [[Microfacet Theory]]：$E(\mu)$ 就是微面 BRDF 的**能量账本**。**"单次散射丢能量"是微面理论固有的、而非实现缺陷** —— 因为互反射就是被 $D\cdot G\cdot F/(4\,\cos\cos)$ 这个形式**结构性地忽略掉**了；
- [[Split-Sum Approximation]]：同一套 IBL 管线里相邻的两步；
- **Furnace test**：本文反复使用的验证手段（分层相干玻璃、sheen、coat 各有一组）。**这是一个可以写进材质验收流程的检查项**；
- *The perception of hazy gloss*（Vangorp et al., Journal of Vision 2017，见原文 References IV）—— **能量补偿的知觉依据**：为什么"补对了"看起来才对。这条把物理与观感连起来，值得单列为后续候选。

### Followed By

- **Fdez-Agüera 2019**（*A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting*, JCGT）—— 下一节点，仍未入库；
- 引擎实现：Unity HDRP / Godot / 多家自研管线中的 "multiple scattering" 开关。

## Personal Knowledge State

- **定位：Normal 可读，且是"能量守恒"这条子线的收口点。** 前置只有两条：你会算方向 albedo（一次半球积分）、你懂 [[Microfacet Theory]]。
- **它在 PBR 收口清单里的位置**：**不计入 25 条** —— 但它是清单之外**最后一个具名缺口**。标记为"读了"，不是"会了"。
- **一句话检验你是否真的懂了**：能不能回答"为什么 $f_{ms}$ 的分母是 $1-E_{avg}$，而分子是两个 $(1-E)$ 的乘积？" —— 能说出"因为 $E_{avg}$ 的定义正好是那个 leftover 积分"，就算懂了。

## Learning Value

**今天最值钱的一句判据：**

> **"物理更真实"和"工程更可用"是两条独立的评价轴 —— 而 Heitz 2016 → Kulla-Conty 2017 是这句话最干净的一个证据。**

**一条可以直接拿去用的推演**：原文说"这个方法对任何 BRDF 都成立、不需要任何假设"。**那么它同样适用于你没有闭式的那些 lobe** —— 比如毛发（[[Hair Rendering]]）与布料。**这是一个从"渲染"跨到"你的领域"的现成桥梁**：VFX 里如果有大面积半透明/绒面材质在低档位发闷，问的第一个问题就是"这个 lobe 的 $E(\mu)$ 有没有被补"。

## Visualization

![[多次散射能量补偿_三条路线图解.html]]

## Notes

- 原文为 **139 页 slides（含 speaker notes 与 errata）**；引用请走 **v2**；
- 与本篇同一 SIGGRAPH 2017 course 里还有一篇对你直接相关的：**Hammon, *PBR Diffuse Lighting for GGX+Smith Microsurfaces*（GDC 2017）** —— 漫反射侧的同类问题（GGX+Smith 下的 diffuse 也会丢能量），**这是"漫反射也漏能量"这条线的入口，可作后续经典候选**；
- 同 course 还有 **Heitz, *A Simpler and Exact Sampling Routine for the GGX Distribution of Visible Normals*（2017）** —— 与今天 Dupuy 那篇的"几何化采样"同源，构成一条隐线；
- 与本篇并列的另一本"补能量手册"：**d'Eon, *A Hitchhiker's Guide to Multiple Scattering*（2016/2022, self-published）** —— Dupuy 2026 也引了它。**未入库，优先级高。**
