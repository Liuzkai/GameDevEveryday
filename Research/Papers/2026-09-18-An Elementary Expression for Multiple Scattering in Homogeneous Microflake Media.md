---
type: paper
title: "An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media"
authors: [Jonathan Dupuy]
year: 2026
published: "2026-09-17 (v1；标题页署 August 2026)"
venue: "arXiv 2609.20394 (cs.GR) — 未见 venue 标注"
url: "https://arxiv.org/abs/2609.20394"
code: ""
project_page: ""
category: [rendering, brdf, microfacet, multiple-scattering, energy-conservation, appearance-modeling]
importance: A
historical_importance: 3
game_relevance: 4
production_readiness: Research
user_level: Normal（概念层）+ Hard（推导层）
status: read
aliases: [Dupuy 2026, Elementary Expression for Multiple Scattering, 微面多次散射闭式解, Quadratic NDF]
tags: [rendering, brdf, microfacet, multiple-scattering, energy-conservation, pbr, appearance-modeling]
---

# An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media

> **一句话定位：单次散射丢了能量、多次散射只有随机模型、能量补偿只有查表近似——这篇用一个特定的 NDF 把"所有阶散射"的**精确**解写成了**一条初等公式**。**
>
> 它精确落库在你 PBR 学习线的"最后残留缺口 (a)"上。原文已下载逐节核对（19 页，含全部推导）。

## TL;DR

- 结论先给：作者构造出一个**一维单侧二次 NDF（quadratic NDF）**，使得半无限均匀微片（microflake）介质里随机游走的**完整**逃逸方向分布有闭式解，求和所有散射阶后得到

$$f_r(\omega_i,\omega_o)=\frac{z_i+z_o}{\pi\,(1+\omega_i\cdot\omega_o)},\qquad z=\cos\theta=n\cdot\omega$$

- 这条 BRDF **精确包含所有散射阶、且保能量**，同时保留单次散射模型才有的**初等求值**与**直接重要性采样**。
- 性质（原文摘要 + 第 6 节）：**法向入射时退化为 Lambertian**（$z_i=z_o=1,\ \omega_i\cdot\omega_o=1\Rightarrow f_r=1/\pi$）；**掠射时反射能量向镜面方向收拢**。
- 副产品：**单次散射阶恰好是单位粗糙度的单次散射 GGX**（原文 Eq. 34），且恰为 Chandrasekhar 单次散射项的 2 倍；**双次散射阶给出了新的闭式**（Eq. 36）。
- 最重要的限制（作者自述）：**这个 NDF 没有粗糙度参数**。"能否在保留这种闭式的同时引入一个扮演 roughness 的参数，尚待确定。"

## Problem

微面理论里有一个持续 10 年的尴尬三分格局：

| 路线 | 求值/采样 | 能量 | 代价 |
|---|---|---|---|
| **单次散射**（Walter 2007 一路） | 初等闭式（便宜） | ❌ **丢能量**（忽略微面间互反射） | 高粗糙度/掠射明显变暗 |
| **多次散射（随机）**（Heitz et al. 2016） | 需要随机数才能**求值**和采样 | ✅ 精确 | 不适合 raster 管线 |
| **能量补偿（查表近似）**（Kelemen 2001 → Kulla-Conty 2017） | 简单乘一项 | ⚠️ 近似（但够用） | 需要 LUT，且假设"多次散射是漫射的" |

原文把这个二元对立说得很清楚：

> "single-scattering models admit elementary closed-form expressions for evaluation and sampling but lose energy by neglecting inter-reflections, whereas multiple-scattering models preserve energy but lack an elementary closed-form characterization."

于是留下一个**明确的开问题**：

> **"Whether a Smith microfacet BRDF can combine exact multiple scattering with an elementary closed-form expression has remained an open question."**

这篇的贡献是**把它答成"可以"**。

## Historical Context

```text
Bouguer 1760（光的衰减律，微面思想的远古源头）
      ↓
Trowbridge & Reitz 1975（平均不规则度表示）+ Smith 1967（几何遮蔽）
      ↓
Blinn 1977（把微面理论带进 CG）
      ↓
★ 关键的等价关系（本文的地基）：Smith 假设 ⇒ 微面输运 ≡ 参与介质输运
      （Jakob et al. 2010 的微片/microflake 框架；Dupuy-Heitz-d'Eon 2016 的"微面-微片统一"）
      ↓
Heitz et al. 2016（Smith 模型下多次散射的随机"真值解"）
      ↓
Kelemen 2001 / Kulla-Conty 2017（把丢失能量用查表补回来的工程解）
      ↓
★ Dupuy 2026（用一个特定 NDF 把所有阶写成闭式）★ 今日入库
```

## Core Idea

整篇的支点是**一次视角切换**：

> **把 BRDF 当成随机游走的"逃逸方向分布"。**

因为微面 BRDF 就是"光进介质、撞若干次、从介质里逃出来"的方向分布。于是 BRDF 定义的难点变成了：**逃逸方向分布能不能解析求出**。

步骤（原文第 2–5 节的骨架）：

1. **微片等价**：Smith 假设说微面朝向与位置**统计独立** → 微面阵列的输运等价于**半无限均匀微片介质的辐射输运**。碰撞服从 Beer-Lambert，方向偏折服从微片镜面相位函数——**两者都只由 NDF 参数化**（Eq. 3–5）。
2. **降维**：介质与边界平行方向均匀 → 横向坐标无关 → **只跟踪垂直深度**；入射方向的"每单位垂直深度的碰撞率"是 $\kappa_j=\sigma_j/|z_j|$（Eq. 6）。于是**前两阶散射的碰撞深度可以解析积分**。
3. **单侧 NDF 再现高度场输运**：单侧 NDF（只让 $z_m>0$）让每次反射**抬高方向**，与 mirror heightfield 的特征输运一致。这个结构让"任意给定方向序列的逃逸概率"有递推式（Eq. 28）。
4. **选一个能求和掉的 NDF**：这条是关键——引入**无参数的二次 NDF**

$$D(\omega_m)=\frac{2}{\pi}z_m^2\ (z_m>0)\ \Longrightarrow\ \sigma(\omega_j)=\frac{(1-z_j)^2}{4}$$

它代入微片相位函数后**可因式分解**为

$$\Psi(\omega_j,\omega_{j+1})=h(z_{j+1}\mid z_j)\cdot P_{r_j}(\phi_{j+1}-\phi_j)$$

其中 $P_{r_j}$ 是**圆 Poisson 核**，参数

$$r_j=\frac{q(z_{j+1})}{q(z_j)},\qquad q(z)=\sqrt{\frac{1-z}{1+z}}$$

5. **为什么能求和**：Poisson 核在**圆卷积下封闭**（$P_r * P_s = P_{rs}$，Eq. 42）。方位角可以解析边缘化，于是**逐阶递推变成一串 $S_k$ 的求和**，而这一串级数**能闭式求和**：

$$R(\omega_1,\omega_o)=2z_o\,P_\eta(\phi_o-\phi_1)=2z_o\frac{z_o-z_1}{2\pi(1-\omega_1\cdot\omega_o)}$$

回到常规入射方向即得 Eq. (1)。

## Technical Approach

### 求值（Eq. 1 本体）

一条公式、一次点乘、两个余弦。**没有任何 LUT、没有随机数、没有分支复杂度**。这对 GPU 是极友好的形态。

### 采样：把 BRDF 看成"弦盘变换的 Jacobian"

第 5 节给的是这篇在工程上最实用的一段：BRDF 的方位分布**就是** Poisson 核，因此可以用 **McCullagh 的 Möbius 变换几何采样**：

- 参数 $\eta$（复平面上的"支点"）作为圆盘内的一个 pivot；
- 取单位圆上均匀点 $c=e^{i\theta}$，过 $c$ 与 $\eta$ 的直线与圆再交于

$$v=\frac{\eta-c}{1-\bar\eta c}$$

- 该映射的角 Jacobian 恰好就是 Poisson 核，因此 $v$ 服从参数 $\eta$ 的 Poisson 分布。

**"用圆盘上的一个支点做几何变换来采样"** —— 这是可以脱离本文单独记住的一个技巧（它和作者 2023 年那篇 GGX VNDF 球冠采样是同一种"几何化采样"风格）。

### 验证

- **随机游走直方图 vs 闭式**：三个入射角（0°/45°/80°）的直方图与闭式重合（Fig. 7）；
- **方向光下四路对比**（Fig. 8）：Eq.(1) vs 随机游走 / Lambertian / 单次散射 GGX / 多次散射 GGX，四个入射角（0°/90°/130°/160°）；
- **IBL 下四路对比**（Fig. 9，uffizi.hdr）：原文结论——"our BRDF produces a diffuse-like appearance close to the Lambertian reference, while retaining a direction-dependent response. The single-scattering GGX model appears darker because it neglects inter-reflections."

## Key Contribution

1. **回答了一个明确的开问题**：Smith 微面 BRDF **可以**同时具备精确多次散射 + 初等闭式（存在性证明）。
2. **给出一个无参数的"能量正确"参考 BRDF**：它天然是"单位粗糙度"那一端，可以直接当作**低粗糙度端 / 高粗糙度端**的对照物与**渲染器能量守恒的自测靶子**（furnace test 的现成解析答案）。
3. **公布了一条新的双次散射闭式**（Eq. 36），以及"单次散射阶 = 单位粗糙度单次散射 GGX"的识别（Eq. 34）。
4. **一条方法论**：先找"能让级数求和掉的 NDF"，而不是先找"参数最好用的 NDF"。**先问可解性，再问参数化。**

## Why It Works

- **本质是把"BRDF 求值"翻译成"随机游走的逃逸分布"**，而逃逸分布在降维之后只是一个**一维深度问题**；
- **二次 NDF 是为求和量身挑的**：它的相位函数能分解成"高度项 × Poisson 核"，而 Poisson 核对卷积封闭 → 中间方向可以被解析地"吃掉"；
- 保能量是**构造出来的**（逃逸概率归一），不是事后修补的。

## Limitations（作者自己列的，值得原样记住）

1. **没有粗糙度参数**。"the proposed NDF is limited in the range of appearances it can represent as it lacks the typical roughness parameters of a microfacet NDF." — 也就是**一个 NDF 对应一个外观**（它落在 GGX $\alpha=1$ 那一端）；
2. **开问题外推**："It thus remains to be determined whether a parametric model can preserve such a kind of closed form while offering a parameter that plays the role of roughness."
3. **均匀半无限介质**假设（无分层、无纹理、无各向异性）；
4. **性能数据一篇未给**：没有 GPU 计时、没有 vs GGX 的指令数/带宽对比 —— 对工程判断是个明确的缺口。

## Game Development Relevance

- **对分档体系（你的核心关切）**：这篇给出的是一条"**零预计算、零显存**"的能量正确 BRDF。它和 [[Split-Sum Approximation]] 的"两趟查表（cubemap + R16G16 LUT）"构成**同一问题的另一族解法**——一边花显存换速度，一边花数学换显存。移动端（显存/带宽最紧）这条路在原理上更香，**但前提是补上粗糙度参数**。
- **对 OverDraw / 大面积半透明**：IBL 环境下"高粗糙度看起来偏暗"是能量丢失的可见后果之一。能量补偿属于**"看起来对"的低成本档位**选择；
- **不改变预算结论**：这篇是 Research 阶段（无参模型），不能直接进管线。

## Unreal Engine Relevance

- 映射到 **Material / HLSL** 层：如果未来出现带粗糙度参数的版本，它是一个可替换 GBuffer 着色路径的候选；
- **现在真正可用的是它作为"靶子"**：UE 的 Roughness = 1 球体（材质预览）在纯环境光下应该接近白色 —— 这就是 furnace test。**可以用它验证你对"能量守恒在引擎里的实际表现"的理解**（UE 的多次散射补偿不是默认开启的）。
- 与 **Substrate** 的关系：Substrate 走的是"分层 lobe + 能量守恒框架"，与本文的"单一闭式 NDF"是两条相反的哲学（分层 vs 单式）。

## Technology Evolution

```text
单次散射足够吗？            Walter 2007（GGX + Smith）         —— 便宜、丢能量
        ↓
多次散射有多重要？          Heitz et al. 2016                  —— 精确，但随机
        ↓
怎么在引擎里补回来？        Kelemen 2001 → Kulla-Conty 2017    —— 查表近似，工程可用 ★ 今日同时入库
        ↓
到底能不能有闭式？          ★ Dupuy 2026 ★                      —— 有，但还没有粗糙度旋钮
        ↓
（下一步）参数化的闭式 NDF？  ← 明确的开放问题
```

**一个结构性观察**：连续四天，渲染线的关键词其实一直是同一个 —— **"能量去哪了"**。
Schlick（1994）解决 F 的**廉价形式**；Walter（2007）把 D/G 定型；Karis（2013）处理**单次散射进引擎**；
9-18 的 Karis 明确指出第一误差源是 $n=v=r$（掠射反射被抺平）；今天 Dupuy 直接给出**掠射端精确解**，并且新模型的掠射特征是"能量向镜面向收拢" —— **和 $n=v=r$ 假设抹平的方向恰好相反**。这不是巧合，而是同一处物理在两条线上的两种处理。

## Relationships

### Based On

- **Smith 1967 遮蔽假设** + **Jakob et al. 2010 微片辐射输运框架** + **Dupuy, Heitz, d'Eon 2016（微面-微片统一）** — 这三条是地基；
- [[Microfacet Theory]] — 本文就是微面理论的一个封闭式解；**微面 ≠ 微片**：微面是表面（单位面积），微片是体积（单位长度/体积）。**"微面 ≡ 微片"这条等价关系本身是 2016 年那篇建立起来的，今天是它的第十年红利。**

### Extends

- **Heitz et al. 2016（Multiple-Scattering Microfacet BSDFs with the Smith Model）**：把它的随机"真值"在**一个特定 NDF 上**变成确定性的闭式 → 也就是**同一条路上从"可采样但不可求值"走到"可求值"**。

### Improves / 对照

- **Kulla-Conty 2017（[[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]）**：🔴 **同一问题的两条完全不同答案** —— Kulla-Conty 是**近似 + 一小块 LUT 覆盖全粗糙度**；Dupuy 是**精确 + 无参数**。**一个是工程解，一个是理论解，今天的价值恰恰在于把两者并排放在一起看。**
- **[[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]]**（9-16）：GPIS 从随机几何**重新推导出 GGX / Beckmann 与 Smith**，并证明 **height-field 极限 → Smith 独立性假设**；今天 Dupuy 的开篇第一句就是 **"Smith 假设 ⇒ 微面输运 ≈ 参与介质输运"** —— 两篇在同一条链上，一个**给出假设**，一个**用假设算出答案**。
- **[[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]**：Eq. (34) 说明**本文单次散射阶 = 单位粗糙度单次散射 GGX**，即 Walter 的模型在 $\alpha=1$ 端的特例。

### Related

- [[Split-Sum Approximation]]：**都是 IBL 镜面半边**，但一个用查表换时间、一个用闭式换参数自由度；
- [[Participating Media]]：本文的整个推导就是**把表面问题写成体积问题**（这正是它能用 Beer-Lambert 与逃逸概率的原因）；
- Chandrasekhar 单次散射项（原文 Eq. 16）：**它是一个长期存在的基准，本文证明该基准的 2 倍，恰好是单位粗糙度下的 GGX 单次散射**。

### Followed By

- **参数化的闭式 NDF**（作者明确点名的下一步）；
- 基于本文做**分档**：低粗糙度端用 GGX+能量补偿，高粗糙度端换成闭式 —— 这只是我的推测，原文没有提。

## Personal Knowledge State

- **概念层：Normal 可读，且几乎不需要增加前置知识。** 你要读懂到"能用起来"只需要三个概念：①Smith 假设 = 微面互相不知道对方在哪；②BRDF = 逃逸方向分布；③保能量 = 逃逸概率归一。
- **推导层：Hard。** 圆柱对称下的分解、Poisson 核卷积封闭、级数求和都属于应用数学工具，不是渲染专有 —— **不必现在懂，也不要为了它停住**。
- **它在你 PBR 线上的位置：残留缺口 (a) 的"理论答案"已到位。** 25 条收口清单**不变**（这条不计入清单，原因是它没有进入任何引擎；**学它是为了把"能量补偿"这件事的上下界一次看全**）。

## Learning Value

**今天最值钱的一句（可迁移判据）：**

> **把"看起来不对"翻译成"能量少算了多少"，然后问"少算的那部分是怎么被补回来的"——是精确补回、近似补回，还是根本没补。**

三个具体可用判据：

1. **看到一个"高粗糙度偏暗"的材质 → 先问它的渲染器有没有做多次散射能量补偿**（UE 默认没有；这就是为什么高粗糙度金属在 UE 里容易显闷）；
2. **看到一个能量补偿方案 → 先问它额外假设了什么**（Kulla-Conty：假设多次散射是漫射的、可以用 $\bar F$ 与 $E_{avg}$ 描述）；
3. **看到一个"闭式解" → 先问它放弃了什么自由度**（本文：放弃了粗糙度）。

## Visualization

![[多次散射能量补偿_三条路线图解.html]]

## Notes

> **一条几乎与内容无关、但值得单独记下来的诚实披露**：原文 "Acknowledgements and Backstory" 写明，这个项目起于 2022 年 3 月，作者当时"手里大部分推导都有了，只差那个二次 NDF"，尝试失败后停摆了几年；后来用 ChatGPT 5.6 复检自己的笔记，**由模型先做了若干次 GGX 的失败尝试，然后找到了这个二次 NDF**。原文写："Credit to OpenAI for finding this NDF: your product is changing the way knowledge is made."
>
> **我的判断（不是原文的话）**：这是一条比论文本身更值得留意的信号 —— **一个 2022 年人类研究者卡死的纯数学搜索，2026 年由模型补完。** 它不改变论文的价值，但改变了"什么算难的"这个判断。记录在此，不做引申。

- 原文 19 页，含完整推导；作者**独立署名**（无合作者）；
- 引用了 Bitterli & d'Eon 2022（position-free path integral）、Cui et al. SIGGRAPH Asia 2023（invariance principle 做多次弹射）、d'Eon 的 *A Hitchhiker's Guide to Multiple Scattering* —— 这三条是同一条技术脉络上**还没进你库里**的几个节点，可作后续经典候选；
- 与作者 2023 年那篇 **GGX VNDF 球冠采样**（Dupuy & Benyoub, *Sampling Visible GGX Normals with Spherical Caps*, CGF 2023）是同一个人的两种"几何化采样"风格 —— 后者已进引擎生态，**是这条线上目前最"可落地"的作品**；
- ⚠️ 尚未见 venue 标注，也未见代码/补充材料。
