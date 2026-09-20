---
type: paper
title: "A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting"
authors: [Carmelo J. Fdez-Agüera]
year: 2019
published: 2019-01-22
venue: "Journal of Computer Graphics Techniques (JCGT) 8(1), 45–55（2019-02-01 修订，含勘误）"
url: "https://jcgt.org/published/0008/01/03/paper.pdf"
code: "论文内附完整 GLSL（Listing 1 / Listing 2）"
project_page: "https://jcgt.org/published/0008/01/03/"
doi: ""
category: [rendering, brdf, image-based-lighting, multiple-scattering, real-time]
importance: A
historical_importance: 4
game_relevance: 5
production_readiness: Production Ready
user_level: Normal
status: read
---

# A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting

> 入库于 2026-09-20。原文 PDF 已下载并**逐节核对**（11 页，JCGT 开放获取）。
> **它是"能量账本"这条线上唯一一篇从一开始就冲着实时来的论文**，而且它的答案极其反直觉地便宜：
> **"你用来算单次散射的那些预计算积分里，本来就含有模拟剩下几次弹射所需的全部信息。"**

## TL;DR

**不加任何新查表、不加任何新参数、不改管线结构**，只在现有的 IBL 镜面着色后面补几行算术，就让环境光下的镜面反射同时获得**能量守恒（高粗糙度不再发暗）**与**能量保持（低粗糙度不再在掠射边缘超额）**。

**核心洞见一句话**：

> **Epic 那张 EnvBRDF LUT 的两个通道相加，就是单次散射的方向 albedo。**
> `float Ess = f_ab.x + f_ab.y;` —— 而缺失的能量是 `Ems = 1 - Ess`。**表已经在跑了，账也就在里面。**

论文给出了**完整可用的 GLSL**（Listing 1 导体 / Listing 2 电介质）。原文对开销的表述是 **"the overhead introduced by multiple scattering code is very low" / "negligible cost"**（未给逐帧数字，因为成本就是几条标量指令 + 复用已有 irradiance 与 LUT 采样）。

## Problem

实时 IBL 用 [[Split-Sum Approximation]]（Karis 2013）把半球积分拆成"预滤波环境图 × 环境 BRDF LUT"，**但它只算单次散射**。于是**同一个表面有两个方向都不对**：

| 端 | 现象 | 原因 |
|---|---|---|
| **高粗糙度端** | 表面**变暗、颜色失饱和** | 微面间互反射的能量被丢掉（见 [[Multiple Scattering and Energy Compensation]]） |
| **低粗糙度端** | 掠射边缘**超额能量**（一圈不自然的亮边） | 另一侧的误差：漫反射项常被写成 $1-F_0$，忽略了掠射角 Fresnel 升高、本该有更多光留在镜面项 |

> 🔴 **"低粗糙度端超额"这一条，是本库此前没有正面记录过的另一半账。**
> Kulla-Conty 2017 与 Dupuy 2026 处理的都是"能量不够"；**这篇论文明确指出：也是"能量多了"**。原文用一张参考照片说明：**修好之后，球体边缘的暗边更接近真实照片**（Fig. 7：SS / MS / 参考照片三张对照）。

## Historical Context

```text
2013  Karis —— split-sum：单次散射的 IBL 变成"两次查表"
        ↓
2016  Heitz et al. —— Smith 多次散射的精确真值（随机求值，不可实时）
2017  Kulla-Conty —— 放弃分布正确性，补方向 albedo（path tracing 语境；多张 2D/3D 表）
2018  Hill —— 逐次弹射模拟逐次能量（更准 → 每次弹射一张表；只管解析光，不管 IBL）
2018  Lagarde & Golubev（credit Turquin）—— 只缩放已有 lobe（见下"第五条路线"）
        ↓
★ 2019  Fdez-Agüera —— 本文：**实时 IBL 侧收口**，零新增表
        ↓
2026  Dupuy —— 所有散射阶的精确闭式（无粗糙度参数）
```

**原文明确给了与前三者的定位区别**（这一段是它自己的"路线图"，值得直接记住）：

| 前作 | 原文的评述 |
|---|---|
| Karis 2013 | 单次散射的预计算基线（本文的一切都建立在它之上） |
| Kulla-Conty 2017 | **"the reasoning is analogous to the one presented in this article, but they compute several 2D and 3D look-up tables, and their results are intended for path tracing and not for real-time IBL."** |
| Hill 2018a | 把 Fresnel 在多次弹射中的展开写成**几何级数**（本文的推导与之同构） |
| Hill 2018b | 用路径追踪模拟**逐次弹射**的能量，**比本文更准**，但要为每次弹射预计算一张表，**且不处理 IBL** |

> **一句判据（本文自证）**：**"更准"和"能用"在这里又一次分家** —— Hill 2018b 更准但更重且不覆盖 IBL；本文选择"够准 + 零新增资源"。**这是本库第三次记录同一条工程规律**（Heitz 2016 更准但不可实时；Kulla-Conty 更通用但不在实时语境）。

## Core Idea（推导链，逐式核对）

### ① 完美反射体：$1 = E_{ss} + E_{ms}$

设 $F=1$（消除 Fresnel 项）。**能量守恒就是"方向 albedo 恒等于 1"**：

$$1=E_{ss}(\omega_o,r)+E_{ms}(\omega_o,r)\;\Longrightarrow\;E_{ms}=1-E_{ss}$$

而这个 $E_{ss}$ **恰好就是 Epic 那张 LUT 在做的事**（把 scale 与 bias 两项相加，原文 Eq. 2 直接说明）。

于是：**加一个额外 lobe $f_{ms}$，其方向 albedo 恰为 $1-E_{ss}$，缺口就被精确补上。**

### ② 把它 split-sum 化：二次以上散射当作"漫射"

$$L_o=\int (f_{ss}+f_{ms})\cos\theta_i L_i\,d\omega_i
=\underbrace{\int f_{ss}\cos\theta_i L_i}_{\text{Karis split-sum}}+\underbrace{\int f_{ms}\cos\theta_i L_i}_{f_{ms}\text{ 的 split-sum}}$$

对第二项**再拆一次**，并假设"参与二次以上散射的能量已被均匀漫射"：

$$\int f_{ms}L_i\cos\theta_i\,d\omega_i\;\approx\;\Big(\int f_{ms}\cos\theta_i\,d\omega_i\Big)\Big(\int \frac{L_i}{\pi}\cos\theta_i\,d\omega_i\Big)=(1-E_{ss})\cdot\text{irradiance}$$

**这个近似对均匀环境是精确的**（所以 furnace test 完美通过），对大范围光源（天穹）很好；**对窄光源（解析点光/方向光）不成立** —— 这是本文的关键限制，后文单列。

### ③ 引入 Fresnel：几何级数，并推出 $E_{avg}$ 与视角相关

不能简单令 $F_{ms}=F_{ss}$（后者依赖视线方向，前者是"来自所有方向的衰减"）。把逐次弹射写成几何级数（每次逃逸比例 $E_{avg}$、每次吸收比例 $1-F_{avg}$）：

$$E=F_{ss}E_{ss}+\sum_{k=1}^{\infty}F_{ss}E_{ss}(1-E_{avg})^kF_{avg}^k$$

**与 ① 的 $1=E_{ss}+\sum E_{ss}(1-E_{avg})^k$ 对照，可直接解出**：

$$\boxed{E_{avg}=E_{ss}}$$

> **这一步是全篇最漂亮的地方**：能量逃逸比例**不是常数，而是与视角相关** —— "**有些方向上光需要更多次弹射才能逃出去**"。
> **这也解释了为什么直接用一张粗糙度-视角无关的表会不准。**

### ④ $F_{avg}$ 有解析解（Schlick 的红利）

$$F_{avg}=2\pi\!\int_0^{\pi/2}\!\big(F_0+(1-F_0)(1-\cos\theta)^5\big)\frac{\cos\theta}{\pi}\sin\theta\,d\theta\;=\;F_0+\frac{1-F_0}{21}=\frac{1+20F_0}{21}$$

### ⑤ 用起来的那个式子

$$\boxed{F_{ms}=\frac{(F_0f_a+f_b)\,F_{avg}}{1-F_{avg}(1-E_{ss})}},\qquad E_{ss}=f_a+f_b$$

其中 $f_a,f_b$ 就是 **Karis 的 LUT 两个通道**（$F_{ss}E_{ss}=F_0f_a+f_b$）。

### ⑥ 导体与电介质的最终组合

**导体**（原文 Listing 1）：

$$L_o=(F_0f_a+f_b)\cdot\text{radiance}+\underbrace{F_{ms}E_{ms}}_{\text{新加的一项}}\cdot\text{irradiance}$$

**电介质**（原文 Listing 2）—— 多一项漫反射，并且**把常见的 $E_d=1-F_0$ 换掉**：

$$1=F_{ss}E_{ss}+F_{ms}E_{ms}+E_d\;\Longrightarrow\;E_d=1-(F_{ss}E_{ss}+F_{ms}E_{ms}),\qquad K_d=\text{albedo}\cdot E_d$$

> **这就是"低粗糙度端超额能量"被修掉的地方**：$E_d$ 不再盲目取 $1-F_0$，而是**先把镜面侧（含多次弹射）实际拿走的部分扣掉，剩下的才给漫反射**。所以**两个端都被修正**（原文 Fig. 6：高粗糙度补回缺失能量；低粗糙度消除边缘超额）。

## 原文 GLSL（导体，可直接照抄）

```glsl
vec3 Fr = max(vec3(1.0 - roughness), F0) - F0;
vec3 kS = F0 + Fr * pow(1.0 - ndv, 5.0);
vec2 f_ab = textureLod(uEnvBRDF, vec2(ndv, roughness), 0).xy;
vec3 FssEss = kS * f_ab.x + f_ab.y;

float lodLevel = roughness * numEnvLevels;
vec3 reflDir = reflect(-eye, normal);
vec3 radiance  = getRadiance(reflDir, lodLevel);   // 预滤波镜面
vec3 irradiance = getIrradiance(normal);           // 余弦加权辐照度

// ——— 多次散射：零新增查表 ———
float Ess  = f_ab.x + f_ab.y;                      // ← 就是它
float Ems  = 1.0 - Ess;
vec3  Favg = F0 + (1.0 - F0) / 21.0;
vec3  Fms  = FssEss * Favg / (1.0 - (1.0 - Ess) * Favg);

return FssEss * radiance + Fms * Ems * irradiance;
```

电介质版只把最后两行换成：

```glsl
vec3 Edss = 1.0 - (FssEss + Fms * Ems);
vec3 kD   = albedo * Edss;
return FssEss * radiance + (Fms * Ems + kD) * irradiance;
```

## Key Contribution

1. **指出"缺失的信息已经在现有预计算里"** —— 不需要 Kulla-Conty 式的**新增** 2D/3D 表；
2. 给出**逐式推导**（含 $E_{avg}=E_{ss}$ 这一步），使方案不是拟合而是可解释的近似；
3. **同时处理两个方向**：高粗糙度补能量、低粗糙度消超额（电介质 $E_d$ 的重新定义）；
4. **给出可直接使用的 GLSL**；
5. 明确的适用范围：**任何能用 split-sum 预计算（或有解析拟合）的 BRDF 都适用**，不限于 GGX。

## Why It Works

因为它**没有试图把"分布"算对，只把"账"补平** —— 与 Kulla-Conty 同一条思路，但**把"账"的来源换成了已有的表**。

关键的三层假设，逐层都很干净：

| 层 | 假设 | 何时成立 |
|---|---|---|
| ① | 二次以上散射能量**均匀漫射** → 可用 irradiance 近似 | **均匀环境精确**；大面积光源很好；**窄光源不成立** |
| ② | 每次弹射与第一次**行为相同**（只是把衰减后的 irradiance 当光源） | 因为反射发生在随机朝向的微面上，**光确实会越来越漫射** —— 与 split-sum 自身的假设同源，自洽 |
| ③ | 电介质不显式建模 diffuse↔specular 界面来回散射 | 电介质镜面反射低且不饱和，**最终总会逃出** |

## Limitations（原文自述）

1. **🔴 只修 IBL，不修解析光。** 原文原话：这个近似 "**is still a good approximation for lights that cover a big portion of the hemisphere... but not for narrow lights where most energy comes from a single direction (e.g., analytical point or directional lights)**"。
   → **含义**：**你已经补平了环境光那半边，但点光/方向光/聚光灯那半边还是空的**。这一条直接说明了为什么 [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] 仍不能被本文替代（它不依赖"光源分布是漫射"这个假设）；
2. **不显式建模 diffuse↔specular 的多次往返**（对电介质可接受，理由见上）；
3. **仍是"补标量"**：分布形状不对（多次散射被当成漫射），高粗糙度掠射端可能偏平。

> **⚠️ 官方勘误（必读，且必须引用修订版）** —— JCGT 论文页勘误栏原文（本次已核对）：
> ```
> Equation 6,7,10: missing factors of 1/π in integrals.
> Equation 11: missing factor of π.
> Equation 15: missing dot product operator.
> Listing 1,2: Constant used in computing Favg adjusted by π.
> ```
> 论文页标注 **"Updated for errata: 2018-02-01"**。
> **实操含义**：**照抄 Listing 时，$F_{avg}$ 的常数以勘误后的实现为准。**业界广泛使用的形式是 $F_{avg}=F_0+(1-F_0)/21=(1+20F_0)/21$（与 Filament 引擎文档一致）——**引用本文时请注明"已勘误版"。**
> **一条方法论观察**：**本库这条线上的两篇工程文献（Kulla-Conty 2017、本篇）都有官方勘误。** 预计算/近似类论文的公式最容易在常数与因子层出错 —— **照抄之前先找勘误页，这一步值得成为固定动作。**

## 顺带发现的第五条路线：只缩放已有 lobe（更便宜，已实装）

在核对本文时，从 **Filament 引擎官方文档**（Google Filament *Physically Based Rendering* 文档，包含 DFG 预积分章节）取到另一条**比本文更便宜**的路线，应当补进本库的路线账：

$$f_r(l,v)=f_{ss}(l,v)+F_0\Big(\frac{1}{r}-1\Big)f_{ss}(l,v),\qquad r=\int_\Omega D(l,v)V(l,v)\,\langle n\!\cdot\!l\rangle\,dl$$

```glsl
vec3 energyCompensation = 1.0 + f0 * (1.0 / dfg.y - 1.0);
Fr *= energyCompensation;   // 直接缩放已有的镜面 lobe
```

- **核心洞见**：**$r$ 就是 LUT 的第二个通道 `dfg.y`**（Filament 的 DFG LUT 存 `x = Fresnel 加权项`、`y = 非 Fresnel 项`，而 $E_{ss}$ 的重构恰好是 `mix(dfg.xxx, dfg.yyy, f0)`）。
- **它把 $F_{avg}$ 进一步简化成 $F_0$**，于是**连新的一项都不需要** —— **零新增查表、零新增项、一条乘加**。
- Filament 文档把它记为 **[Lagarde18]，作者 Lagarde 与 Golubev，并将该观察致谢给 Emmanuel Turquin**（即本库 9-19 记录的 Kulla-Conty 勘误指出者 —— **同一个人，两次**）。
- ⚠️ **来源标注**：**本条的公式与 shader 代码来自 Filament 官方文档，已逐字核对；但 `[Lagarde18]` 的**原始书目条目本次未取到**（Filament 文档正文的参考文献段未包含在可抓取内容中）。** 标为"待核实原始出处"。另：Filament 文档也把 [Heitz16] 标注为 **"not suitable for real-time rendering"** —— 与 Heitz 原文自述完全一致，是一次独立佐证。

> **给你的直接含义（三选一，成本递减）**：
> | 方案 | 新增资源 | 新增运算 | 修正范围 |
> |---|---|---|---|
> | Kulla-Conty 路线 | 新 2D+3D 表（≈4KB+） | 多次采样 + 一个 lobe | 任意 BRDF、任意光源 |
> | Fdez-Agüera 路线 | **零** | 几次标量 + 复用 irradiance | **只 IBL**；同时修两端 |
> | Filament/Lagarde 路线 | **零** | **一条乘加** | **只 IBL**；只补高粗糙度端的能量（不修电介质超额） |
> **这是本库目前最清晰的一条"预算 vs 修正范围"权衡表**，可以直接放进分档讨论。

## Game Development Relevance

- **🔴 它给出了"关掉能量补偿会看到什么"的准确描述**（这是分档判断最需要的东西）：
  - 关掉 → **高粗糙度金属/塑料发暗、颜色失饱和**；**低粗糙度电介质的掠射边缘出现一圈不自然的亮边**；
  - 开启 → 粗糙金属**颜色饱和度回升**；光滑塑料的**边缘更接近真实照片**（原文用参考照片对照）；
  - **都属于"看得出但能忍"的降档差异**（尤其是仅 IBL 时）→ 适合中低档关闭。
- **它是"固定开销"里最便宜的一项**：Fdez-Agüera 路线**零新增资源**、**零新增采样**，只增加几条标量运算。**在一个已经跑着 Sky Light 的场景里，它的边际成本约等于 0** —— **这类"几乎免费的正确性"应当默认开启，而不是当成分档项**。
- **⚠️ 但它的边界必须写进分档文档**：**它只修 IBL。** 所以在"动态灯光 / 解析光"占主导的场景（如 MegaLights 大量点光），它**不解决**能量问题 —— **"环境光那半边修好了、灯光那半边还是空的"是一个容易被误当成"已经做了能量补偿"的陷阱。**
- **与 [[Split-Sum Approximation]] 的关系可写进预算表**：这两篇加起来才是 **IBL 镜面半边的完整固定开销** —— cubemap 预滤波 + EnvBRDF LUT + （**同一张 LUT** 免费带出的）能量补偿。

## Unreal Engine Relevance

- **映射点非常明确**：这套东西对应 UE 的 **Sky Light 镜面反射路径**。若要在 UE 里复现，位置是**材质图里 SkyLightReflection 之后的镜面着色段**（Custom 节点 / 材质函数），**输入就是已有的 EnvBRDF LUT 采样结果（UE 用的正是 R16G16，见 [[Split-Sum Approximation]]）**；
- **不需要新的贴图资产、不需要改渲染管线** → **这是本库目前"研究 → 引擎"距离最短的一篇**（论文自带 GLSL，且不引入任何新资源）；
- **⚠️ 但要先核实 UE 当前版本是否已经内建**：UE 的能量补偿状态在不同版本/配置下不一致（[[Multiple Scattering and Energy Compensation]] 已记录"UE 里它不是无条件默认开启的"）。**动手前先在引擎里跑一次 furnace test 看当前状态** —— 这正好与你要做的那次实测是同一个动作。

## Technology Evolution

见 [[Multiple Scattering and Energy Compensation]] 的 Historical Evolution 段（本文对应其中 **2019** 那一行，已在 2026-09-20 由"未入库"改为"已入库"）。

## Relationships

### Based On

- [[Karis — Real Shading in Unreal Engine 4 (2013)]] / [[Split-Sum Approximation]] —— **整篇论文都是它的延长线**：全部信息取自它预计算的两张东西（预滤波 cubemap + EnvBRDF LUT）；
- [[Microfacet Theory]] —— $D\cdot G\cdot F/(4(n\cdot l)(n\cdot v))$ 的框架；
- [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]] —— $F_{avg}$ 的解析解**只有在 Schlick 形式下才存在**（第三次红利：先给可分解性、再给可预积分、再给可解析平均）。

### Extends

- [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] —— **同一条思路（补标量）在实时 IBL 语境下的重做**，把"新增表"换成"复用表"；
- **Hill 2018a**（Fresnel 的几何级数展开）—— 本文的级数推导与之同构；
- **Kelemen & Szirmay-Kalos 2001** —— 公式的最初源头（经 Kulla-Conty 中转）。

### Improves

- [[Split-Sum Approximation]] —— 把它的"只算单次散射"补齐（**在 IBL 范围内**）；
- 常见引擎实现里的 $E_d=1-F_0$ —— 换成 $1-(F_{ss}E_{ss}+F_{ms}E_{ms})$，**修掉低粗糙度端的超额能量**。

### Contrasts

- [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]] —— **两个极端**：Dupuy 要的是**分布精确**（代价：没粗糙度参数）；本文要的是**能不能进实时管线**（代价：分布被当成漫射）。
- [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] —— **本文被设计出来的理由**就是 Heitz 的那句"unsuitable for real-time"。

### Related

- [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]] —— **本文用到的两项里，`irradiance` 就是它**（SH 投影那套）。**所以这篇论文同时踩在 2001 与 2013 两块基石上**；
- [[Multiple Scattering and Energy Compensation]] —— 本概念的"第五条路线"（见上）；
- [[Marschner — Light Scattering from Human Hair Fibers (2003)]] · [[Hair Rendering]] —— **一个未被本文覆盖的边界**：毛发没有 split-sum，所以这条"复用 LUT"的路子对它**不适用**（毛发需要自己的能量账）。

## Personal Knowledge State

- **user_level: Normal**（**结论与实现层都很可读，不需要任何随机输运基础**）。
  - **它不是 Hard 的**：整个推导只用到"半球积分 + 余弦权重"这一级工具；
  - **可执行程度是全库最高的一篇之一**：论文自带 GLSL、零新增资源、对应关系明确到具体某个 LUT 通道；
- **⚠️ 与 PBR 收口清单的关系**：**不计入 25 条**。它是**执行层**的材料 —— 25 条里唯一那条实测题（furnace test）**做完之后，如果发现引擎当前没有能量补偿，这篇就是补它的方案**。

## Mastery Criteria（5 条，纸面自测）

1. 说出为什么 **"EnvBRDF LUT 的两个通道相加 = 单次散射方向 albedo"**（提示：LUT 是对 $F_0$ 的 scale 与 bias）；
2. 说出本文**为什么能零新增查表**（提示：缺的信息是 `1 - Ess`，$E_{ss}$ 已经在表里）；
3. 说出 **$E_{avg}=E_{ss}$ 这一步的物理含义**（提示：能量逃逸比例与视角相关 —— 有些方向需要更多次弹射才能逃出去）；
4. 说出本文**能修什么、修不了什么**，并解释"修不了"的那个为什么必须由别的方案接手；
5. 说出电介质那条 `Edss = 1 - (FssEss + Fms*Ems)` **比 `1 - F0` 多修了哪一端**（答案：低粗糙度掠射端的**超额**能量，不是缺失）。

> **一句话检验**：能说出 **"环境光那半边修好了，不代表灯光那半边也修好了"**，即算抓住了本文的边界。

## Visualization

[[多次散射_五条补法路线与实时落地图解]] —— 含本文的**数据流图**（EnvBRDF LUT → $E_{ss}$ → $E_{ms}$ → $F_{ms}$ → 合成）与五条路线的成本/修正范围对照。

## Notes

- 2026-09-20 入库。**原文 PDF 已下并逐节核对**（11 页，`https://jcgt.org/published/0008/01/03/paper.pdf`）；**勘误栏已从 JCGT 论文页逐字核对**。
- **发表信息**：Received 2018-10-01；Recommended 2018-11-12；Published 2019-01-22；**Updated for errata 2018-02-01**（原文标注如此，年份疑似论文页笔误，按页面原文照录不改）。
- **作者背景**：Carmelo J. Fdez-Agüera，署名地址都柏林 —— **本文是个人独立署名**（不是大厂研究院产出）。**这本身是个信号：实时渲染的能量补偿改进，是能由单个工程师完成的规模的工作。**
- **方法经验（本次新增，务必沿用）**：**核对精确/预计算类论文的公式时，先找勘误页。** 本次两篇经典（本篇与 Kulla-Conty 2017）**都有官方勘误**，且都是**常数与因子层**的错误 —— 这类错误照抄进 shader 不会报错，只会慢半拍地表现成"看起来有点不对"。
