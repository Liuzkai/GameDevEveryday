---
type: paper
title: "An Inexpensive BRDF Model for Physically-based Rendering"
authors: [Christophe Schlick]
year: 1994
published: "1994"
venue: "Computer Graphics Forum 13(3): 233–246"
url: "https://doi.org/10.1111/1467-8659.1330233"
code: ""
project_page: ""
category: [brdf, microfacet, fresnel, real-time-rendering]
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal
status: read
aliases: [Schlick 1994, Schlick's approximation, Schlick 近似]
tags: [brdf, pbr, fresnel, classic, easy-candidate]
---

# An Inexpensive BRDF Model for Physically-based Rendering (Schlick 1994)

## TL;DR

**你在 UE 里用的那个 F 项，出自这篇。**

整篇论文提出的 BRDF 模型早已没人用了，但其中顺手给出的一个 Fresnel 近似活了下来，并且今天仍在你每天写的 shader 里：

$$F(\theta) = F_0 + (1 - F_0)\,(1 - \cos\theta)^5, \qquad F_0 = \left(\frac{n_1 - n_2}{n_1 + n_2}\right)^2$$

配合前两天的经典，**你的 D · G · F 三因子现在全部有了出处**：

| 因子 | 你现在用的是什么 | 出自 |
|---|---|---|
| **D** | GGX / Trowbridge-Reitz | [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] |
| **G** | Smith height-correlated | [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]（2026 年被重新推导为 height-field 极限） |
| **F** | **Schlick 近似** | **本文（1994）** |

Cook-Torrance 1981 给了框架（[[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]），Kajiya 1986 给了容器（[[Kajiya — The Rendering Equation (1986)]]），Walter 2007 给了 D 和 G，**今天这篇给你最后一个 F**。

## Problem

1994 年的处境：

- **经验模型**（Phong / Blinn-Phong）便宜但没有物理约束，不保能量守恒，参数与真实材质无关，换光源就崩；
- **完整物理模型**（Cook-Torrance 1981）正确但**太贵**：精确的 Fresnel 方程要处理偏振（Rs / Rp 两个分量）、复折射率、开方，每像素算一次在当时的软硬件条件下不可接受，更不用说塞进早期硬件；
- **测量/理论模型**（Ward 1992 等）各有取舍，但普遍参数不直观，美术调不动。

Schlick 的定位写得很清楚：他要的是一个**介于经验与理论之间的中间模型**——保留物理的主要结论（能量守恒、可逆性 reciprocity、微面理论），同时把光照中的众多现象以"物理上说得通"的方式纳入，但**为计算机图形量身定制**，只坚持两件事：

1. **Simplicity（简单）**：少量直观参数控制模型；
2. **Efficiency（高效）**：能适配 Monte Carlo 渲染，且**能硬件实现**。

第二条是它活下来的原因：它是**为硬件写的**。

## Historical Context

时间线定位：

```text
Phong 1975 / Blinn-Phong 1977 —— 纯经验，无物理
        ↓
Cook-Torrance 1981 —— D·G·F 框架 + 能量守恒（正确但贵）
        ↓
Ward 1992 —— 各向异性测量模型
        ↓
★ Schlick 1994 —— "中间模型"：物理约束 + 硬件可实现（本文）
        ↓
Walter 2007 —— GGX + Smith，D 与 G 的现代标准
        ↓
Disney Principled 2012（Burley）—— 采纳 Schlick 的 F
        ↓
Karis 2013（UE4 Real Shading）—— 随之进入 Unreal，成为工业默认
        ↓
UE5 Substrate —— Schlick 形式的 F 仍在
```

一个反讽的事实：**Schlick 那套完整 BRDF 模型已经被遗忘，只剩一个"顺手给的近似"被全世界用了 30 年。** 这本身就是一条值得记住的规律——**在工程里，活得久的常常不是最完整的方案，而是那个在正确抽象层级上做了正确妥协的部件。**

## Core Idea

论文的自我定位（原文摘要口径）：一个新 BRDF 模型，可视为**经验与理论之间的中介模型**。它遵守物理学的主要结论（能量守恒、可逆律、微面理论），并把光反射中的大量现象以物理上合理的方式纳入——不相干/相干反射、光谱改变、各向异性、自遮蔽、多次表面与次表面反射、均质与非均质材料的差异。

而我们今天只记得其中一条：**Fresnel 近似**。

### 那个公式

$$R(\theta) = R_0 + (1 - R_0)(1 - \cos\theta)^5, \qquad R_0 = \left(\frac{n_1 - n_2}{n_1 + n_2}\right)^2$$

- $\theta$：入射方向与法线的夹角（在微面模型里换成**半程向量**，见下）；
- $R_0$：垂直入射（$\theta = 0$）时的反射率，也就是**最小的那个反射率**；
- $(1-\cos\theta)^5$：一个从 0 单调升到 1 的五次曲线。

### 为什么是 5 次方

这是全文最值得看的一句：精确 Fresnel 曲线在 $\theta \in [0°, 90°]$ 上是一条**从 $R_0$ 平滑单调升到 1 的 S 形曲线**。Schlick 观察到：用 $(1-\cos\theta)$ 的**五次幂**能在整段上给出足够好的拟合，同时只需要一次减法、一次乘方、一次 lerp——**没有开方，没有偏振拆分，没有复数**。

> 这不是"推导出来的"，这是**拟合出来的**。指数 5 是一个经验选择，不是物理常数。记住这一点，后面 Limitations 里的误差分布就好理解了。

## Technical Approach

### 在微面模型里怎么用（这一步最容易搞错）

在 [[Microfacet Theory]] 的标准形式里：

$$f_r(l,v) = \frac{D(h)\,G(l,v,h)\,F(v,h)}{4\,(n\cdot l)\,(n\cdot v)}$$

**F 的角度不是 $(n\cdot v)$，而是 $(h\cdot v)$（或等价的 $(h\cdot l)$）**。也就是说：**Fresnel 是在"微面法线"上求的，不是在"宏观表面法线"上求的。**

这个细节是理解掠射白化的关键：

| 求 F 的角度 | 含义 | 后果 |
|---|---|---|
| 用 $(n \cdot v)$ | 把宏观表面当成一面镜子 | 只有整体掠射时才变白 |
| **用 $(h \cdot v)$**（正确） | 每个微面各自算 Fresnel | 粗糙表面上，即使宏观不掠射，也有大量微面处于掠射状态 → **粗糙高光整体偏白、边缘更亮** |

### 金属怎么办（生产里真正的做法）

精确 Fresnel 对**导体（金属）**要用**复折射率**，曲线形状与介电质完全不同，而且 $F_0$ 随波长变化——这就是金属的高光**带颜色**（金是金的、铜是铜的），而非金属高光**永远是白的**。

**Schlick 的公式对导体在物理上是不成立的**（它没有复折射率的位置）。工业上的做法是**绕过**：

> **不去算 $F_0$，直接把 $F_0$ 设成测得的该金属反射率（也就是美术给的 BaseColor）。**

在 UE 里就是：
- **非金属（dielectric）**：`Specular` 默认 0.5 对应 $F_0 \approx 0.04$，高光白色；
- **金属（metal）**：`Metallic = 1` 时 $F_0$ 直接取 `BaseColor`，漫反射归零——**高光带材质本身的颜色**。

**一个物理上错误的公式，靠一个参数化技巧变成了美术可控且视觉正确的东西。** 这是实时渲染里非常典型的一类妥协，值得单独记住。

### 更一般的形式（UE / Karis 的 split-sum 里实际在用的）

$$F = F_0 + (F_{90} - F_0)\,(1 - \cos\theta)^5$$

多出一个 $F_{90}$：$\cos\theta \to 0$ 时不强制取 1，而是取一个可调上限。粗糙表面的掠射反射达不到 100%，用 $F_{90}$ 可以把这一端压下来。**这就是 UE 环境光 BRDF LUT 里那条曲线**——如果你读过 Karis 2013 的 split-sum，看到的就是它。

## Key Contribution

1. 提出"**中间模型**"这一设计哲学：物理约束不丢，但把成本压到能进硬件——这是后来所有实时 PBR 的取舍范式；
2. 给出 Fresnel 的五次幂近似，**成为 30 年来实时渲染的默认 F 项**；
3. 明确把"**美术可调的参数少且直观**"作为设计目标，而不只是追求物理正确。

## Why It Works

1. **曲线形状对了**：精确 Fresnel 的主要视觉信息就是"从 $F_0$ 单调升到 1"，五次方拟合守住了这个形状；
2. **成本极低**：一条 `pow` + 一次 `lerp`，在现代 GPU 上基本免费，1994 年的硬件也扛得住；
3. **参数有物理意义**：$F_0$ 就是"垂直看过去有多亮"，美术能直接对着材质表填；
4. **两端严格正确**：$\theta = 0$ 时精确等于 $F_0$，$\theta = 90°$ 时精确等于 1。**误差只出现在中间段**（见下）。

## Limitations —— 误差到底有多大（实测算）

我用精确 Fresnel 方程（非偏振平均，$\tfrac{1}{2}(R_s + R_p)$）对比 Schlick 近似，采 101 个点：

| IOR $n$ | $F_0$ | 最大绝对误差 | 出现在 |
|---|---|---|---|
| 1.50（玻璃/塑料） | 0.0400 | **0.0357** | $\cos\theta \approx 0.09$（约 85°） |
| 1.33（水） | 0.0201 | **0.0599** | $\cos\theta \approx 0.11$（约 84°） |

误差的符号是**分段**的（以 $n=1.5$ 为例）：

| $\theta$ | 精确值 | Schlick | 偏差 |
|---|---|---|---|
| 0° | 0.0400 | 0.0400 | 0 |
| 26° | 0.0408 | 0.0400 | −0.0008 |
| 46° | 0.0509 | 0.0423 | −0.0086 |
| **60°** | **0.0892** | **0.0700** | **−0.0192（低估约 21%）** |
| 73° | 0.2078 | 0.2013 | −0.0064 |
| 84° | 0.5716 | 0.6069 | **+0.0353（高估约 6%）** |
| 90° | 1.0000 | 1.0000 | 0 |

**结论有三条，比"误差 0.036"这个数字重要：**

1. **30°–70° 区间系统性低估**，84° 之后转为高估；
2. **最坏点落在 85° 附近的"肩部"**——正好是掠射高光开始起飞的位置；
3. **但这个误差在最终像素里基本看不见**，因为 F 还要乘 $D$ 和 $G$ 两个因子，而 **$G$（masking-shadowing）在掠射角会趋近 0**，把 F 的误差一起压掉了。

> **这才是为什么一个"最大误差 3.6%"的近似能用 30 年：误差最大的地方，恰好是另一个因子把它抹平的地方。** 评估一个近似能不能用，不要孤立看它的误差，要看**误差在最终表达式里被谁乘掉了**。

另外三条真实局限：

- **对导体物理上不成立**（无复折射率），靠 $F_0$ 参数化绕过；
- **不含色散**（$n$ 随波长变化），做不出棱镜色散一类的效果；
- **指数 5 是拟合的**，不是对所有 IOR 都最优；特殊材质（如高 IOR 半导体）误差会更大。

## Game Development Relevance

**直接相关度 5/5。这是"你每天都在用但你未必知道出处"的那一类。**

具体到你的工作：

1. **VFX 半透明与掠射**：VFX 的片状粒子在掠射角变亮，相当一部分视觉贡献就来自 F 项。做 additive/alpha 特效时，如果某个贴片在边缘突然发白，第一反应应该是查 F，不是查 alpha。
2. **OverDraw 与 F 的关系**：F 让掠射更亮，掠射又往往对应**更大屏幕覆盖**的片——**"越费的地方越亮"**，这在预算上是坏消息。做 SABC 分级时，掠射可见的大面积特效应该单独加权。
3. **分档**：Schlick 在任何档位都是同样的成本（一条 pow），**不是分档变量**。真正随档位变的是 $D$ 的粗糙度精度与 $G$ 的近似阶数。**别把 F 当成优化对象，它没有优化空间。**

## Unreal Engine Relevance

- **材质节点**：`Fresnel` 节点默认就是 Schlick 形式（`Exponent` 默认 5、`BaseReflectFraction` 对应 $F_0$）。
- **Lit 材质**：`Metallic` / `Specular` / `BaseColor` 三条输入最终就在决定 $F_0$（见上表）。
- **环境光 split-sum LUT**：UE 的 EnvBRDF 近似用的就是带 $F_{90}$ 的广义 Schlick 形式（Karis 2013）。
- **Substrate（UE 5.2+）**：Schlick 形式的 F 仍然是底层积木之一。
- **不是优化点**：不要试图"简化 F 来提性能"，它已经是最便宜的项；真正贵的是 $D$ 的采样与多 lobe 叠加。

## Technology Evolution

```text
精确 Fresnel 方程（19 世纪物理，含偏振与复折射率）
        ↓
★ Schlick 1994 —— 五次幂近似，为硬件而生
        ↓
Disney Principled 2012 —— 采纳
        ↓
Karis 2013 —— 进 UE4，并推广为 F0/F90 广义形式（split-sum）
        ↓
UE5 Substrate —— 仍是默认
        ↓
2026 —— 神经渲染（DLSS 5）开始直接"学"材质外观，F 项可能在生成式路径里被绕过
                 （见 [[DLSS 5 — Generative Neural Rendering]]）
```

## Relationships

### Based On

- Fresnel 方程（物理光学，19 世纪）——本文是它的实时近似

### Extends / Productionizes

- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] — 本文把框架里最贵的 F 换成廉价近似，使实时化成为可能

### Related

- [[Microfacet Theory]] — F 是三因子之一；**注意 F 在 $h\cdot v$ 上求，不是 $n\cdot v$**
- [[BRDF]] — 本文是一族 BRDF 中的一个历史节点
- [[Physically Based Rendering]] — 实时 PBR 得以成立的三个妥协之一（另两个是 GGX 与 split-sum）
- [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] — 同属"你实际在用的东西"系列：它给 D 和 G，本文给 F

### Contrasts

- [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]] — 2026 的工作从随机几何**重新推导**出 GGX 与 Smith；F 项没有对应的"重新推导"工作，因为**它从一开始就是拟合而非推导**——这个对比恰好说明了三因子的性质差异。

## Personal Knowledge State

- **user_level: Normal（研读中）**。你正在系统学 D·G·F，D 与 G 已在 9-16 收口，今天 F 收口。
- 判断依据：你已在研读 Cook-Torrance 与 Walter，F 是同一条线上的最后一个因子，难度不高于前两者。

## Mastery 自测（5 条，过了即可把 F 标 Easy）

1. 写出 Schlick 公式，并说明 **$F_0$ 的物理含义**（垂直入射时的反射率）以及它怎么从两个介质的 IOR 算出来。
2. **微面模型里 F 的角度是 $h\cdot v$ 还是 $n\cdot v$？** 说错会怎样？（答：是 $h\cdot v$；用错会导致粗糙表面的高光不随掠射变白。）
3. 为什么**金属的高光带颜色、非金属的高光是白的**？用 $F_0$ 解释，并说明 Schlick 公式对金属物理上为什么不成立、工业上怎么绕过。
4. Schlick 的误差在哪个角度段最大？**为什么这个误差在实际画面里几乎看不见？**（答：约 85° 肩部；因为 G 项在掠射趋于 0，把 F 的误差一起压掉。）
5. 打开 UE，指出**哪几个材质输入在决定 $F_0$**，以及 `Fresnel` 节点的 `Exponent` 默认值是多少。

> 这 5 条 + [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 的 5 条 + [[Kajiya — The Rendering Equation (1986)]] 的 5 条 + [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] 的 5 条 = **PBR 收口的完整自测清单（20 条）**。

## Learning Value

1. **D·G·F 三因子今天全部有了出处**，你的 PBR 研读线在**来源侧**闭合；
2. 一个可复用的评估方法：**看近似的误差要连着看它在最终表达式里被谁乘掉**（上面 Limitations 第 3 条）；
3. 一个关于工程寿命的观察：**活下来的不是最完整的方案，是在正确层级上做了正确妥协的那个部件**（完整 Schlick BRDF 已死，它的副产品活着）。

## Visualization

![[Schlick 近似 vs 精确 Fresnel 图解.html]]

含：两条曲线对比（n=1.5 / n=1.33）、误差曲线与分段符号、以及"为什么误差看不见"的三因子相乘示意。

## Notes

- 引用信息已核实：Computer Graphics Forum, 13(3), 233–246, doi:10.1111/1467-8659.1330233。
- 误差数据为本次运行用精确 Fresnel 方程（$\tfrac{1}{2}(R_s+R_p)$）自行计算，101 采样点，见 Visualization。
- **下一个经典候选（[[Hair Rendering]] 方向）**：Kajiya-Kay 1989（各向异性经验模型）、Marschner 2003（发丝物理模型 R/TT/TRT）。
