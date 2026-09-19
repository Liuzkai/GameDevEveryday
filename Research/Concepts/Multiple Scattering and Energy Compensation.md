---
type: concept
user_level: Normal
aliases: [Multiple Scattering, 多次散射, Energy Compensation, 能量补偿, Energy Conservation, fms, 能量守恒, Furnace Test]
prerequisites: [Microfacet Theory, BRDF, Physically Based Rendering]
first_introduced: "问题自 Cook-Torrance 1981 就存在；工程解 Kelemen 2001；理论真值 Heitz 2016；闭式解 Dupuy 2026"
---

# Multiple Scattering and Energy Compensation

> 建立于 2026-09-19。**它是 [[Physically Based Rendering]] 的 Learning Gap 里最后一块"账本"**：
> [[Microfacet Theory]] 给你 $D\cdot G\cdot F$，但**这个形式本身就结构性地忽略了微面之间的互反射**——于是表面"少收了一笔钱"。
> 这个笔记就是那笔钱的账：**少了多少、怎么量、三条不同的补法。**

## Definition

**单次散射（single scattering）假设**：入射光打到某个微面上，反弹一次就离开表面。微面 BRDF 的标准形式

$$f_r(l,v)=\frac{D(h)\,G(l,v,h)\,F(v,h)}{4\,(n\cdot l)\,(n\cdot v)}$$

**只描述一次反弹**。真实微结构表面里，光会在微面之间**反复弹射若干次**才离开——这部分能量被这个公式**完全忽略**（不是数值误差，是结构性缺失）。

**结果**：方向 albedo

$$E(\mu_o)=\int_{0}^{2\pi}\!\!\int_{0}^{1} f(\mu_o;\mu_i,\phi)\,\mu_i\,d\mu_i\,d\phi$$

在高粗糙度与掠射角处 **$E<1$**，表面看起来**比物理上应该的暗**。

> **"Multiple scattering（多次散射）"** = 这件事本身（微面间互反射的物理输运）。
> **"Energy compensation（能量补偿）"** = 把这件事造成的缺口用工程手段补回去的做法。
> **两个词不要混用**：前者是物理，后者是工程近似。

## Core Principle

### 1. 三维账本（这是本概念的中心表）

| 路线 | 求值 | 采样 | 能量 | 参数自由度 | 代表工作 |
|---|---|---|---|---|---|
| **单次散射** | 初等闭式 ✅ | 闭式 ✅ | ❌ 丢能量 | 全（含 roughness / anisotropy） | Walter 2007（GGX+Smith） |
| **精确随机多次散射** | ❌ 需要随机数 | ❌ 需要随机数 | ✅ 精确 | 全 | Heitz et al. 2016 |
| **能量补偿（查表近似）** | 便宜 ✅ | 便宜 ✅ | ⚠️ 近似（账平、分布是近似的） | 全 | Kelemen 2001 → [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] |
| **精确闭式多次散射** | 初等闭式 ✅ | 闭式 ✅ | ✅ 精确 | ❌ **无 roughness 参数** | [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]] |

**这张表的读法（今天最值钱的一句）：**

> **"精确"、"便宜"、"有参数" 三样东西，目前还没有任何一条路线同时拿到。**

- Heitz 2016 拿到"精确 + 有参数"，代价是不能在 raster 管线里求值；
- Kulla-Conty 2017 拿到"便宜 + 有参数"，代价是牺牲分布的精确性；
- Dupuy 2026 拿到"精确 + 便宜"，代价是**只有一个外观**（没有粗糙度旋钮）。

**所以"多次散射"这件事目前是一个被三篇论文从三个方向同时逼近、但都还差一角的三角形。**

### 2. 为什么单次散射会丢能量（这一条要能自己讲出来）

不是实现 bug，是**公式形态决定的**：

- $D(h)\,G\,F$ 里，$G$ 项负责"这次弹射没有被挡住"；
- 光**被挡住**时，微面理论的处理方式是"这一份能量消失"；
- 但在真实微结构里，被挡住的光**会弹到别的微面上，最终仍有机会离开表面**。

**"被挡住" ≠ "被吸收"。** 单次散射把前者当成了后者 —— 这就是能量丢失的全部来源。

> **一个推论**：能量损失随**粗糙度**上升而加剧（微面朝向越散乱 → 互遮挡越多），并在**掠射角**处最严重。**这与你在引擎里看到的现象一致：rough 材质在掠射边缘发闷。**

### 3. 怎么量：方向 albedo 与 Furnace Test

- **方向 albedo $E(\mu_o)$**：把 BRDF 按余弦加权在整个入射半球积分。纯反射表面**理想值是 1**；
- **Furnace Test**：把物体放进**完全均匀的辐射场**渲染。一个纯反射表面应该**完全消失**（与背景不可分）。$E<1$ 时它会显出一团"灰影"；
- **Furnace test 是可执行的自测**：引擎里建一个纯金属球（Metal=1、无贴图），Roughness 从 0 扫到 1，放在只有环境光（无方向光、无阴影）的场景里截图。

**这是整个 PBR 体系里唯一一个"能一眼看出对不对"的验证方法**——它不需要参考照片，不需要美术判断。

### 4. 三条补法（不是一条）

| 补法 | 补的是什么 | 额外假设 | 关键代价 |
|---|---|---|---|
| **解析闭式**（Dupuy 2026） | **真的把输运算出来** | Smith 独立性 + 一个特制 NDF | 无 roughness 参数 |
| **随机真值**（Heitz 2016） | 真的把输运采样出来 | Smith 独立性 | 求值也要随机数 |
| **账平近似**（Kelemen 2001 / Kulla-Conty 2017） | 只补**方向 albedo 这一个标量** | 多次散射是**漫射**的 | LUT + 分布近似 |

**账平近似的核心巧思值得单独记住**（5 行数学，见 [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]）：

$$f_{ms}=\frac{(1-E(\mu_o))(1-E(\mu_i))}{\pi(1-E_{avg})},\qquad E_{avg}=2\!\int_0^1\!E(\mu)\mu\,d\mu$$

它的方向 albedo **恰好等于** $1-E(\mu_o)$，**正好补上原 BRDF 缺的那一份**，**且不依赖任何关于原 BRDF 的假设**。

### 5. 一个可迁移的审近似判据

> **看到任何"补能量 / 预计算 / 近似"方案，先问：为了让结果可预存或可闭式化，它额外假设了什么？这个假设在哪个角度/参数区间会肉眼可见？**

- Kulla-Conty：假设多次散射是漫射的 → 高粗糙度掠射端可能偏平；
- Dupuy：假设一个固定 NDF → 完全没有粗糙度自由度；
- Karis 2013（[[Karis — Real Shading in Unreal Engine 4 (2013)]]）：为了让 IBL 的 $F_0$ 能提出积分，额外假设了 $n=v=r$ → 掠射反射被抹平（**作者自己说这是第一误差源**）。

**三条同属一套审查姿势。**（承 9-17 "误差被谁乘掉"、9-18 "可预存需要什么假设"。）

## Prerequisites

- [[Microfacet Theory]] — 必须先懂 $D/G/F$ 各管什么，才看得懂"哪一项丢了能量"；
- [[BRDF]] — 要会算方向 albedo（半球积分 + 余弦权重）；
- [[Physically Based Rendering]] — 能量守恒与能量保持在生产语境里的区别。

## Historical Evolution

```text
1981  Cook-Torrance（微面框架定型，单次散射的形态从此刻起就定了）
        ↓
多年  "高粗糙度发闷"被当成美术问题，靠手调补偿
        ↓
2001  Kelemen & Szirmay-Kalos（第一个通用补偿公式：查表）—— 只讲塑料，无透射
        ↓
2014  Jakob et al.（综合分层框架；提出 F_avg 的思路）—— 完整但太重
        ↓
2016  Heitz et al.（Smith 模型下多次散射的随机真值）—— 精确但不可求值
        ↓
★ 2017  Kulla & Conty（把 Kelemen 公式工程化：32×32 表 = 4KB + 解析 F_avg + 忽略各向异性）
        —— 被多家引擎/工作室采用
        ↓
2019  Fdez-Agüera（JCGT，实时 IBL 版，进一步修正）—— 未入库
        ↓
2010s 微面-微片统一（Dupuy/Heitz/d'Eon 2016）→ 把表面问题写成体积问题
        ↓
★ 2026  Dupuy（特制二次 NDF 上所有散射阶的精确闭式）—— 理论天花板，但无粗糙度参数
```

## Important Papers

| 论文 | 角色 | 状态 |
|---|---|---|
| [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] | 单次散射的定型（GGX + Smith） | ✅ 入库 |
| [[Karis — Real Shading in Unreal Engine 4 (2013)]] | 单次散射进引擎；明说 $n=v=r$ 是第一误差源 | ✅ 入库 |
| [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] | **工程解 + 证明**（本篇） | ✅ 入库 |
| [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]] | **理论天花板**（精确闭式） | ✅ 入库 |
| Heitz et al. 2016, *Multiple-scattering Microfacet BSDFs with the Smith Model*, TOG 35(4) | 精确真值（随机） | ❌ 未入库，**优先级高** |
| Kelemen & Szirmay-Kalos 2001, Eurographics Short | 公式源头 | ❌ 未入库（较老，可只在本文引用） |
| Fdez-Agüera 2019, JCGT, *A Multiple-Scattering Microfacet Model for Real-Time IBL* | 实时 IBL 版 | ❌ 未入库 |
| d'Eon, *A Hitchhiker's Guide to Multiple Scattering* | 系统性手册 | ❌ 未入库，**优先级高** |
| Hammon 2017, *PBR Diffuse Lighting for GGX+Smith Microsurfaces* | **漫反射侧**的同类问题 | ❌ 未入库 |

## Related Concepts

- [[Microfacet Theory]] — 本概念是它的**必然副产品**：有互遮挡就必然有互反射，有互反射就必然有"忽略互反射"的误差；
- [[Split-Sum Approximation]] — **同一根 IBL 管线上的相邻两步**：split-sum 解决"积分太贵"，能量补偿解决"积分算少了"。**两者叠加才是 IBL 镜面半边的完整固定开销**；
- [[Participating Media]] — 多次散射的**理论通道**：Dupuy 2026 的整个推导就是把表面写成半无限微片介质；
- [[Hair Rendering]] — **另一个"微面 BRDF 失效"的边界**（单位不同）；毛发里的多次散射由 Marschner 的 TRT 等明确路径处理，因为纤维内部吸收是主要的、可以显式建模；
- **Chandrasekhar 单次散射项** — 长期基准；Dupuy 2026 证明它的 2 倍恰是单位粗糙度的单次散射 GGX。

## Technologies

- **引擎侧的 "Multiple Scattering / Energy Compensation" 开关**（Unity HDRP、自研管线、部分 UE 配置）——**注意：UE 里它不是无条件默认开启的**；
- **$E(\mu)$ / $E_{avg}$ LUT**（32×32 + 32 项 1D ≈ 4KB float）—— 一个典型的"**低成本档位开关**"：
  - 开启 = +1 次纹理采样 + 4KB 常驻；
  - 关闭 = 高粗糙度端偏暗、颜色失饱和（属于"降档可忍受"的退化类型）。
- **Substrate（UE5）** — 把能量守恒内建在分层框架里，而不是事后加一个 lobe。

## Game Applications

- **材质观感**：高粗糙度金属/塑料"发闷"的**标准解释与标准解**；
- **分档**：见 [[Scalability and Quality Tiers]] —— 这是一条"**关掉后能看出但能忍**"的档位差异，适合放在中低档关闭；
- **资产规范**：Kulla-Conty 原文提到美术把木材 IOR 调到 100+ 的案例。**建议在资产取值表里加入 IOR / 金属度 / F0 的允许范围** —— 这类"资产侧不物理"比性能问题更容易漏，而且会让不同档位的行为不一致；
- **材质验收**：**furnace test** 可以直接作为材质检查项（不需要参考照片）。

## Personal Knowledge

- **user_level: Normal。** 前置只有 [[Microfacet Theory]] 与 [[BRDF]]，都已经在库；
- **本概念不计入 PBR 的 25 条收口清单** —— 它是清单之外的**最后一条具名缺口**，今天标为"读了，不是会了"；
- **一句话检验**：能说出"被挡住的光不是被吸收了，所以单次散射把它当成消失是错的"即算抓住核心。

## Learning Gap

1. **推导层（Hard，不必现在动）**：Dupuy 2026 的 Poisson 核卷积封闭与级数求和；Heitz 2016 的随机 Smith 输运。**不要为了这两个停住 PBR 的收口；**
2. **数值层（Normal，可快速补）**：$E(\mu)$ 与 $E_{avg}$ 的具体形状（作为 roughness 与 $\mu$ 的函数）。**最好的补法是看一眼 Kulla-Conty 的那两张表**；
3. **实测层（Easy，可以先做）**：引擎里那个 Furnace Test。**这是 25 条自测清单里唯一的引擎侧实测题。**

## Next Step

1. **把 furnace test 做成一次真实的自测**（30 分钟内可完成）：
   纯金属球 → Roughness 0→1 → 只有环境光 → 截图；再切换引擎侧的能量补偿开关对比。**做完这一条，本概念即可从 Normal 升 Easy**；
2. 后续经典候选（按优先级）：**Heitz et al. 2016**（精确真值，补齐"三角形"的另一角）> **Hammon 2017**（漫反射侧的同类问题，与你的 diffuse 认知直接相关）> **d'Eon 的 Hitchhiker's Guide**（系统性手册）；
3. 与 [[Split-Sum Approximation]] 合并成一张"I BL 镜面半边固定开销表"（cubemap 预滤波 + EnvBRDF LUT + 能量补偿表），纳入 [[Real-Time VFX Performance Budgeting]] 的参考账。

## Notes

- 本概念与 [[Split-Sum Approximation]] 是**同一天方法论的两个实例**：**"想省，就先问自己额外假设了什么"**；
- ⚠️ 归属提醒：**Kulla-Conty 的补偿公式本体来自 Kelemen 2001**，Kulla-Conty 的贡献是把公式做成可用的工程方案并补上证明。**不要写成"Kulla-Conty 发明了能量补偿"。**
