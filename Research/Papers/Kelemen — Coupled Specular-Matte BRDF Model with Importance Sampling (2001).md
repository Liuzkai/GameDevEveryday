---
type: paper
title: "A Microfacet Based Coupled Specular-Matte BRDF Model with Importance Sampling"
authors: ["Csaba Kelemen", "László Szirmay-Kalos"]
year: 2001
published: 2001
venue: "Eurographics 2001 - Short Presentations"
url: "https://cg.iit.bme.hu/~szirmay/scook.pdf"
code: ""
project_page: ""
category: ["rendering", "pbr", "microfacet", "importance-sampling"]
importance: A
historical_importance: 4
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal
status: unread
---

# Kelemen — A Microfacet Based Coupled Specular-Matte BRDF Model with Importance Sampling (2001)

## TL;DR

把 Cook-Torrance 微面 BRDF 做了三步"减法"：① G 项与分母合并，min() 三项用公共项 $\frac{1}{1+\vec L\cdot\vec V}=\frac{1}{4\cos^2\beta}$ 一次近似——**(N·L)(N·V) 分母整个消失**；② D 与 F 依 $\cos\beta=H\cdot L$ 合并成单变量函数 $g_\lambda(\cos\beta)$——BRDF 变成 "H 的概率密度 × 一维查表"；③ 漫反射不再独立，乘上 $(1-a_{\text{spec}})$ 因子并与镜面**耦合**——能量守恒的漫反射侧最早通用形式。副产品：Ward 分布可逆采样、近乎最优的 BRDF 重要性采样（权重 ≈1）、off-specular peak 与"掠射角镜面化"全部保留。**附录给出了 Cook-Torrance 模型的完整光子概率推导**（含 $d\omega_H/d\omega_V=\frac{1}{4\cos\beta}$ 的立体角变换——半程向量采样的 Jacobian）。

![[Research/Files/Pictures/Kelemen 2001 — 图1 掠射角下镜面桌实拍.png]]
**读图**：同一张漆面桌从三个视角拍——视角越掠射，桌面越像镜子（镜面成分占比升高、漫反射消退）。这是全文要建模的物理现象，也是"matte 与 specular 的比例不能是常数"的实拍证据。（图 1，Kelemen & Szirmay-Kalos 2001）

## Problem

2001 年时的微面 BRDF（Cook-Torrance 形态）在工程上有三个硬伤：

1. **数值不稳 + 不可采样**：$f_r=\frac{P_H(\vec H)}{4(\vec N\cdot\vec L)(\vec N\cdot\vec V)}\,G\,F$ 里 $(N\cdot L)(N\cdot V)$ 分母可任意小；G 项的 min() 使 BRDF 不可微；且很难构造一个正比于 $\cos$ 加权 BRDF 的采样密度。
2. **漫反射与镜面被当成独立两层**：以往模型里 matte/specular 比例是常数，但实拍（图 1）与 Shirley et al. 1997 的耦合理想镜模型都表明掠射角下镜面占主导——旧模型在这个区域系统性偏暗。
3. **Beckmann 分布不可逆采样**：其 CDF 无闭式逆，重要性采样做不了。

## Historical Context

```text
Torrance-Sparrow 1966（物理光学：off-specular peak 实测）
  ↓
Cook-Torrance 1981（微面模型进 CG：D·G·F/(4(N·L)(N·V))）
  ↓
Shirley et al. 1997（coupled matte-specular：理想镜版本，(1-a) 耦合第一次出现）
  ↓
★ Kelemen & Szirmay-Kalos 2001（本篇：推广到 glossy + 重要性采样 + 一维查表）
  ↓
Ward 1992 的分布被重新启用为可逆采样密度（Gaussian/Beckmann 的可采样替代）
  ↓
Walter 2007（GGX + Smith 分离 G，现代 D/G/F 参数化定型）
  ↓
Karis 2013（UE4：GGX+Smith+split-sum，实时收口；移动端/IBL 常用 "Kelemen 近似"）
  ↓
Kulla-Conty 2017（把本篇的补偿公式工程化成 4KB LUT——能量补偿线的直接后裔）
```

三条谱系在本篇交汇：**微面理论线**（CT 的简化）、**能量守恒线**（matte 耦合 = Kulla-Conty 的代数雏形）、**采样线**（Ward 可逆采样 + 近最优 BRDF 采样——[[Veach — Optimally Combining Sampling Techniques for Monte Carlo Rendering (1995)|MIS 1995]] 同时代的应用侧）。

## Previous Work

- **Cook-Torrance 1981**：微面 BRDF 的原始形态，本文的简化对象（附录完整重推其概率结构）。
- **Shirley et al. 1997（技术报告）**：coupled matte-specular 模型——镜面侧是**理想镜**；本文第一次把耦合推广到 **glossy**（粗糙）镜面。
- **Lafortune & Willems 1994**：modified Phong 的物理化与采样——本文采样效率的对比基线（"比 Phong 采样更有效"）。
- **He et al. 1991**：完备物理模型——复杂到不可用，反衬"简化但要保性质"的路线。

## Core Idea

一句话：**把 G·F 除以分母后剩下的东西，重新组织成"一个采样密度 × 一个一维函数"，然后让漫反射看见镜面吃掉了多少能量。**

三步：

### ① G 项简化——分母消失

把 G 除以 $2(N\cdot L)(N\cdot V)$ 后，min 的三项各自改写，其中前两项可统一近似为 $\frac{1}{1+\vec L\cdot\vec V}$（原文式 2），并证明它恒小于第三项 $\frac{1}{2(N\cdot V)(N\cdot L)}$（证：$\frac{1}{1+\vec L\cdot\vec V}\le\frac{1}{2\cos\theta\cos\theta'}$，即 $2\cos\theta\cos\theta'\le\cos(\theta-\theta')+\cos(\theta+\theta')= \vec L\cdot\vec V+1$ 的平凡推论）。于是：

$$f_r^{\text{spec}}(\vec L,\vec V)=\frac{P_H(\vec H)\cdot F(\lambda,\vec H\cdot\vec L)}{4\,(1+\vec L\cdot\vec V)}=\frac{P_H(\vec H)\cdot F(\lambda,\cos\beta)}{4\cos^2\beta}=\frac{P_H(\vec H)\cdot F(\lambda,\cos\beta)}{\vec h\cdot\vec h}$$

其中 $\vec h$ 是**未归一化**半程向量，$\beta$ 是 $H$ 与 $L$（或 $V$）的夹角。**(N·L)(N·V) 分母连同 min() 一起消失**——数值稳定性与可微性问题同时解决。

### ② D 与 F 合并——"密度 × 一维表"

D（本文用 Ward 密度 $P_H^{\text{Ward}}=\frac{1}{m^2\pi\cos^3\alpha}e^{-\tan^2\alpha/m^2}$，α 为 H 与 N 夹角）依赖 α，F 依赖 cos β，两者**都只经过 H 参数化**。定义 $g_\lambda(\cos\beta)=(4\cos^2\beta)^{-1}\cdot F(\lambda,\cos\beta)$ 后：

$$f_r^{\text{spec}}=P_H(\vec H)\cdot g_\lambda(\cos\beta)$$

- 电介质：$g_\lambda$ 有较简单的公式；金属：复折射率使计算贵——**存成一维表，用内存访问替代计算**（"预计算存法"谱系在 BRDF 项上的最早应用之一）。
- 这个"密度 × 一维函数"的分解正是它**可精确重要性采样**的原因。

### ③ matte 耦合——能量守恒的漫反射

$$f_r^{\text{matte}}(\vec L,\vec V)=k(\lambda)\cdot\frac{(1-a_{\text{spec}}(\theta'))\,(1-a_{\text{spec}}(\theta))}{\pi\,(1-a_{\text{avg}}^{\text{spec}})}$$

- **$(1-a_{\text{spec}})$ 因子**：镜面反照率吃掉多少，漫反射就让出多少（入射、出射两侧各乘一次，保 Helmholtz 对称）；
- **$\frac{1}{1-a_{\text{avg}}^{\text{spec}}}$ 归一化**：用镜面平均反照率补回被压缩的总能量，使全反射材质总 albedo 恒等于 1（图 5 的两条曲线互补）；
- 这正是 **Kulla-Conty 2017 补偿公式的代数雏形**——后来者把 $(1-a)$ 项做成查表 lobe、把 $a_{\text{avg}}$ 精化为 $F_{\text{avg}}\cdot R_{\text{avg}}$ 并补证明。

## Technical Approach

**采样方案**（specular）：采样 H ~ $P_H$（Ward 密度：$\alpha=\arctan(m\sqrt{-\log(1-u)})$, $\phi=2\pi v$；或 Phong 密度：$\cos\alpha=u^{1/(n-1)}$），再镜像 $\vec L=2\cos\beta\cdot\vec H-\vec V$。立体角变换给出 $P_L=P_H/(4\cos\beta)$，于是 MC 权重：

$$\frac{f_r\cos\theta'}{P_L}=F(\lambda,\cos\beta)\cdot\frac{\cos\theta'}{\cos\beta}$$

高光泽时 β≈θ′，权重 ≈1——**近乎最优重要性采样**；最差情形比值有界（<2），优于 reciprocal Phong 的采样权重。这是"采样权重方差决定采样质量"（MIS 定理 1 的直观面）在具体 BRDF 上的完整实例。

**matte 采样**：因为 matte BRDF 是 $r(\theta')r(\theta)$ 可分离形式，定义累积反射函数 $I(\theta')=\int_0^{\theta'}(1-a_{\text{spec}}(\xi))\cos\xi\,d\xi$，其逆存一维表——精确重要性采样只需查表。

**完整 BRDF 的走样**：Russian roulette 三选一——以 $a_{\text{spec}}(\theta)$ 概率走 specular、以 $\tilde k(1-a_{\text{spec}})$ 走 matte、其余终止。

## Key Contribution

1. **Kelemen 形式** $P_H\cdot g_\lambda(\cos\beta)$：实时引擎 "Kelemen visibility/近似" 的原始出处——把 $G/(4(N\cdot L)(N\cdot V))$ 合并进一个只依赖 $(L\cdot H)$ 的项。
2. **能量守恒漫反射的通用公式**：(1-a) 压缩 + 平均反照率归一化——Kulla-Conty 的直接前身（[[Multiple Scattering and Energy Compensation]] 谱系的路线③源头）。
3. **Ward 分布的闭式逆采样** + 近最优 BRDF 采样权重的证明。
4. **附录：Cook-Torrance 的完整光子概率推导**——orientation（可见微面取向概率，含 $d\omega_H/d\omega_V=1/(4\cos\beta)$ 推导）× no shadow & mask（G 项从 l₁/l₂ 几何显式推出）× reflection（Fresnel = 光子存活率）三事件相乘。

## Why It Works

- **G 近似的合法性**不是"看起来差不多"：$\frac{1}{1+\vec L\cdot\vec V}$ 被证明恒 ≤ min 三项改写后的公共下界（原文给出平面/非共面两种情形的三角不等式证明），所以简化后的 BRDF **处处不高于**原 CT——能量守恒不会被近似破坏。
- **可采样性**来自"密度 × 一维函数"结构：采样 H 即采样了 BRDF 的全部方向依赖，剩下的一维因子进权重或查表。
- **耦合 matte 保对称**：两侧各乘 $(1-a_{\text{spec}})$ 且 $a_{\text{spec}}$ 只依赖极角，Helmholtz 对称自动成立；归一化常数 $s=\frac{1}{\pi(1-a_{\text{avg}}^{\text{spec}})}$ 由全反射情形的 albedo=1 条件闭式解出。

## Limitations

- **matte 分量自认非 physically based**（作者原话："physically plausible, but is not physically based"）——它是能量账本守恒的构造，不是多次散射输运的解；多次散射的真实方向分布（microflake 理论的对象）它不回答。
- **specular 只算单次反弹**：微面间多次散射的镜面能量直接丢给 matte 补——这正是 [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)|Heitz 2016]] 指出的系统误差来源（高粗糙度下偏暗）。
- **G 近似在高粗糙度 + 掠射角下偏离 Smith**：1/(1+LV) 与 Smith-GGX 的差异是后来 "Kelemen 近似 vs Smith 可分"讨论的内容；Walter 2007 之后主流实时管线在精度要求高时回到 Smith。
- 无透射/多层材质（概念页已有记录："只讲塑料"）。

## Game Development Relevance

- **你在 UE 里能直接找到它**：UE 的移动端/简化管线与 Filament 等实时渲染器中的 "Kelemen visibility"（如 $V=\frac{0.25}{(\vec L\cdot\vec H)^2}$ 形态的变体）就是本篇式 (3) 的工程化；[[Karis — Real Shading in Unreal Engine 4 (2013)|Karis 2013]] 的能量守恒 diffuse（(1-F) 压缩）同属本篇开创的耦合思路。
- **预预算视角**：$g_\lambda(\cos\beta)$ 一维表是"**用 1D 查表换每次逐光源的完整 Fresnel+G 计算**"的最早样本之一——BRDF 项的预算语言（计算 ↔ 查表 ↔ 精度）在此成型。
- **off-specular peak + 掠射镜面化**是"廉价但物理可信"的验收样本：任何实时 specular 项实现都该过这两条目检（对应你 SABC 验收清单里的"物理特征不丢"）。

## Unreal Engine Relevance

- 移动端材质与部分 IBL 路径的 Kelemen 近似（Niagara/材质编辑器层面的光滑项选择）；
- UE 的 energy-conserving diffuse（BaseColor 乘 (1-F)）是耦合思路的 2013 版；
- 若做移动端 SABC 低档位 shader 简化，"Smith G → Kelemen 1/(1+LV)" 是一档**有文献依据的降级选项**——成本下降一档、保留 off-specular peak。

## Technology Evolution

| 年份 | 节点 | 变化 |
|---|---|---|
| 1966 | Torrance-Sparrow | 实测 off-specular peak，物理光学微面模型 |
| 1981 | Cook-Torrance | 微面模型进 CG（Beckmann D + G + 完整 F） |
| 1992 | Ward | 各向异性测量 + 可逆 Gaussian 型密度（本篇借用） |
| 1997 | Shirley et al. | coupled matte-specular（理想镜版） |
| **2001** | **Kelemen-Szirmay-Kalos** | **glossy 耦合 + 可采样 + 一维表（本篇）** |
| 2007 | Walter et al. | GGX D + Smith 分离 G，现代参数化定型 |
| 2013 | Karis | UE4 实时收口（split-sum + (1-F) diffuse） |
| 2016-17 | Heitz / Kulla-Conty | 多次散射真值参照系 / 补偿公式工程化（4KB LUT） |

## Relationships

### Based On

- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]——简化对象 + 附录完整重推其概率结构
- Shirley et al. 1997 coupled matte-specular——(1-a) 耦合的原始形态（理想镜版，未入库，见概念页材料清单）

### Extends

- Shirley 1997 的耦合从理想镜推广到 glossy（Ward/Phong/Beckmann 密度均可）

### Related

- [[Veach — Optimally Combining Sampling Techniques for Monte Carlo Rendering (1995)|MIS (1995)]]——采样权重方差论断的理论框架；本文是"单密度近最优"的实例（MIS 是"多密度组合"的通解）
- [[Talbot — Importance Resampling for Global Illumination (2005)|Talbot RIS (2005)]]——BRDF 重要性采样的同代发展（RIS 线）
- [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)|Schlick (1994)]]——F 项的廉价近似（工程侧另一条简化路线）
- [[Microfacet Theory]] / [[BRDF]]——概念底座

### Followed By

- [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]——GGX+Smith 参数化取代（D/G 分开更精确）
- [[Karis — Real Shading in Unreal Engine 4 (2013)]]——实时收口；能量守恒 diffuse 承接耦合思路
- [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]——补偿公式工程化（本篇路线③的直接后裔）
- [[Hammon — PBR Diffuse Lighting for GGX+Smith Microsurfaces (2017)]]——漫反射侧按 GGX+Smith 重做账本
- [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]]——指出单次散射+matte 补偿的系统误差，给出输运级解

## Personal Knowledge State

- `user_level: Normal`——与你的 Cook-Torrance 三因子学习焦点**直接对接**：
  - **G 项的物理来源**（你在研读的重点之一）在本文附录有最完整的"从几何到公式"推导：masking 概率 = $1-\frac{l_1}{l_2}$ 从图 10 的三角形正弦定理直接推出，shadowing 对称同理，min 合并——CT 的 G 不是拍的，是几何。
  - **式 (8)/(9) 的 $\frac{d\omega_H}{d\omega_V}=\frac{1}{4\cos\beta}$** 就是"采样半程向量的 Jacobian"——[[Ouyang — ReSTIR GI Path Resampling for Real-Time Path Tracing (2021)|ReSTIR GI]] 里 Jacobian 修正的 1D 教学版。
  - 附录的"三事件概率相乘"结构（取向 × 无遮挡 × 反射）是理解 D·G·F **为什么是乘积**的最干净讲法。

## Learning Value

1. **Normal → Easy 桥**：把"实时 PBR 的常数与形式从哪来"的最后一块拼上——Kelemen 形式是 Smith 之外你实际会遇到的第二套 G/Vis；
2. **跨线连接**：PBR 线（CT/能量补偿）与采样线（可逆密度/Jacobian/近最优权重）在本文交汇——两线各取一半正好是全文；
3. **预算迁移**："计算 ↔ 一维查表"的替换样本，可直接写进 SABC 档位的 shader 成本表。

## Visualization

![[Research/Files/Pictures/Kelemen 2001 — 图2 off-specular peak 反射瓣.png]]
**读图**：三个入射角（22.5°/45°/67.5°）的反射瓣——瓣的最大值不在理想镜面方向（斜线）而在其下方，即 off-specular peak；且入射越掠射瓣越扁长（趋近镜面）。（图 2，Kelemen & Szirmay-Kalos 2001）

![[Research/Files/Pictures/Kelemen 2001 — 图3 镜面反照率曲线对比 Cook-Torrance.png]]
**读图**：新 BRDF（Phong/Ward/Beckmann 三种采样密度）与原 Cook-Torrance 的镜面 albedo 随入射角变化——四条曲线几乎重合：简化没有破坏能量特征；低粗糙度（右）时全部曲线在掠射角翘起（镜面化）。（图 3，Kelemen & Szirmay-Kalos 2001）

![[Research/Files/Pictures/Kelemen 2001 — 图4 漫反射分量 BRDF 瓣.png]]
**读图**：耦合 matte 分量的 BRDF 瓣（左）与反射 radiance 瓣（右）——入射角越大瓣越偏向掠射方向且隆起，漫反射不是各向同性的 π/1 常数。（图 4，Kelemen & Szirmay-Kalos 2001）

![[Research/Files/Pictures/Kelemen 2001 — 图5 镜面与漫反射反照率互补.png]]
**读图**：m=0.8 时 specular albedo（实线）与 matte albedo（虚线）随入射角此消彼长，**总和恒为 1**——"镜面吃多少、漫反射让多少"的耦合账本，也是图 1 实拍现象的数值形态。（图 5，Kelemen & Szirmay-Kalos 2001）

![[Research/Files/Pictures/Kelemen 2001 — 图8 不同视角的镜面漫反射耦合.png]]
**读图**：同一塑料场景三个视角——视角越掠射，底座越像镜子（反射变强、漫反射色消退），与图 1 实拍桌子的行为一致：耦合模型正确重现"材料随视角变脸"。（图 8，Kelemen & Szirmay-Kalos 2001）

![[Research/Files/Pictures/Kelemen 2001 — 图9 半程向量立体角变换几何.png]]
**读图**：以 L 为极轴的球坐标里，dω_H 与 dω_V 的水平/垂直弧长关系（θ_H=β, θ_V=2β, dV_vert=2dH_vert）→ 推出 $d\omega_H/d\omega_V=\frac{1}{4\cos\beta}$——采样半程向量的 Jacobian，式 (8) 的几何来源。（图 9，Kelemen & Szirmay-Kalos 2001）

![[Research/Files/Pictures/Kelemen 2001 — 图10 masking 遮挡几何.png]]
**读图**：masking 边界情形的楔形几何——出射光恰擦过相邻微面尖端时 l₁=l₂，正弦定理给出 Pr(not masking)=min(2(N·H)(N·V)/(V·H),1)；G 项从这幅图"长"出来。（图 10，Kelemen & Szirmay-Kalos 2001）

## Notes

- 原文：Eurographics 2001 Short Presentations，DOI 10.2312/egs.20011003；PDF 取自作者官方页（cg.iit.bme.hu/~szirmay/scook.pdf，Budapest 技术大学 Szirmay-Kalos 实验室），11 页，本次运行已全文提取核对（式 1-9、G 近似证明、Ward 采样闭式、matte 归一化常数、附录 CT 推导）。
- **入库理由**：[[Multiple Scattering and Energy Compensation]] 概念页"经典候选排队"中的最后一个低优先候选（"已在多处引用"）——入库后路线③的公式源头有独立节点；同时是读者 PBR 复习窗口（近编辑 Heitz/Fdez-Agüera 两篇）与采样线（MIS/RIS/ReSTIR 刚收官）的交点。
- 致谢彩蛋：该 BRDF 曾实现为 **Maya 插件**（Alias Wavefront 捐赠许可）——"理论进 DCC"在 2001 年的样本。
- 名字备忘：Kelemen（人名 Csaba Kelemen）勿与 Kulla-Conty / Karis 混淆——谱系口诀：**Kelemen 给公式，Kulla-Conty 给工程，Karis 给引擎**。
