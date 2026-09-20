---
type: concept
user_level: Normal
aliases: [Multiple Scattering, 多次散射, Energy Compensation, 能量补偿, Energy Conservation, fms, 能量守恒, Furnace Test]
prerequisites: [Microfacet Theory, BRDF, Physically Based Rendering]
first_introduced: "问题自 Cook-Torrance 1981 就存在；工程解 Kelemen 2001；理论真值 Heitz 2016；闭式解 Dupuy 2026"
---

# Multiple Scattering and Energy Compensation

> 建立于 2026-09-19；**2026-09-20 由"三条路线"扩为"五条"**（+Heitz 2016 精确真值、+Fdez-Agüera 2019 实时 IBL 版、+Lagarde/Turquin 最便宜版）。
> **它是 [[Physically Based Rendering]] 的 Learning Gap 里最后一块"账本"**：
> [[Microfacet Theory]] 给你 $D\cdot G\cdot F$，但**这个形式本身就结构性地忽略了微面之间的互反射**——于是表面"少收了一笔钱"。
> 这个笔记就是那笔钱的账：**少了多少、怎么量、五条不同的补法各缺哪一角。**

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

### 1. 路线账本（本概念的中心表；2026-09-20 由三条扩为五条）

| 路线 | 求值 | 采样 | 能量 | 参数自由度 | 修正范围 | 代表工作 |
|---|---|---|---|---|---|---|
| **单次散射**（基准，不是补法） | 初等闭式 ✅ | 闭式 ✅ | ❌ 丢能量 | 全（含 roughness / anisotropy） | — | Walter 2007（GGX+Smith） |
| **① 精确随机多次散射** | ❌ 需要随机数 | ❌ 需要随机数 | ✅ 精确 | 全 | **全部**（含分布 / 各向异性 / 透射） | [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] |
| **② 精确闭式多次散射** | 初等闭式 ✅ | 闭式 ✅ | ✅ 精确 | ❌ **无 roughness 参数** | 全部散射阶 | [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]] |
| **③ 能量补偿（新增查表 lobe）** | 便宜 ✅ | 便宜 ✅ | ⚠️ 近似（账平、分布是近似的） | 全 | **任意 BRDF、任意光源** | Kelemen 2001 → [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] |
| **④ 复用已有 LUT**（实时 IBL） | 便宜 ✅✅ | 便宜 ✅ | ⚠️ 近似（但**两端同修**） | 全 | ⚠️ **只 IBL**（窄光源不成立） | [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] |
| **⑤ 缩放已有 lobe**（最便宜） | **一条乘加** ✅✅✅ | 零新增 | ⚠️ 近似 | 全 | ⚠️ **只 IBL**；只补高粗糙度端 | Filament 文档记作 [Lagarde18]（credit Turquin） |

**这张表的读法（今天最值钱的一句）：**

> **"精确"、"便宜"、"有参数" 三样东西，目前还没有任何一条路线同时拿到。**

- Heitz 2016（①）拿到"精确 + 有参数"，代价是不能在 raster 管线里求值；
- Kulla-Conty 2017（③）拿到"便宜 + 有参数"，代价是牺牲分布的精确性，且要**新增**资源；
- Dupuy 2026（②）拿到"精确 + 便宜"，代价是**只有一个外观**（没有粗糙度旋钮）；
- Fdez-Agüera 2019（④）拿到"便宜 + 有参数 + **零新增资源**"，代价是**只覆盖环境光**；
- Lagarde/Turquin（⑤）拿到"**几乎无成本**"，代价是覆盖面更窄（只补高粗糙度端）。

**两条新增的读法（2026-09-20）：**

1. **成本与覆盖面严格反向**：⑤ 最便宜、范围最窄；③ 最贵、范围最广。**"要不要开"不是 yes/no，而是"你愿意为哪一块面积付钱"**；
2. **④⑤ 的边际成本约等于 0**（在已经跑 Sky Light 的场景里）—— **所以这类"几乎免费的正确性"应当默认开启，而不是当成分档项**。**真正需要分档的是 ③。**

> **一句判据（2026-09-20 新增，与上面并列）**：**"补能量"与"推输运"是两件事。**
> 前者只把**一个标量**（方向 albedo）补平、**不依赖任何物理假设** → 永远"对"，但分布可能是编的；
> 后者**从假设推出分布** → 分布对，但假设错了就全错。
> **看到任何能量补偿方案，先问它是哪一类**（①②是后者；③④⑤是前者）。

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
- **⚠️ 2026-09-20 补充：要查两头**。除"粗糙端是否变暗"之外，还要看**光滑白色电介质的掠射边缘是否有一圈偏亮**（超额能量，见 §4b）。**过去的 furnace test 只查"暗"，不查"亮"。**

**这是整个 PBR 体系里唯一一个"能一眼看出对不对"的验证方法**——它不需要参考照片，不需要美术判断。
**且它同时是这条学习线上唯一的实测题**（其余都是纸面自测）。

### 4. 五条补法（不是一条）

| 补法 | 补的是什么 | 额外假设 | 关键代价 |
|---|---|---|---|
| **① 精确随机**（Heitz 2016） | 真的把输运**采样**出来 | Smith 独立性 | **求值也要随机数 → 不可实时** |
| **② 精确闭式**（Dupuy 2026） | 真的把输运**算出来** | Smith 独立性 + 一个特制 NDF | 无 roughness 参数 |
| **③ 查表 lobe**（Kelemen 2001 / Kulla-Conty 2017） | 只补**方向 albedo 这一个标量** | 多次散射是**漫射**的 | 新增 LUT；分布近似；path tracing 语境 |
| **④ 复用 LUT**（Fdez-Agüera 2019） | 同上，但**信息取自已有的表** | 同上 + 可用 irradiance 近似二次以上散射 | **只 IBL**（窄光源不成立） |
| **⑤ 缩放已有 lobe**（Lagarde/Turquin） | 同上，连新项都不加 | 同上 + $F_{avg} \approx F_0$ | 只 IBL；只补高粗糙度端 |

**账平近似的核心巧思值得单独记住**（5 行数学，见 [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]）：

$$f_{ms}=\frac{(1-E(\mu_o))(1-E(\mu_i))}{\pi(1-E_{avg})},\qquad E_{avg}=2\!\int_0^1\!E(\mu)\mu\,d\mu$$

它的方向 albedo **恰好等于** $1-E(\mu_o)$，**正好补上原 BRDF 缺的那一份**，**且不依赖任何关于原 BRDF 的假设**。

**④ 的巧思是把"缺口"的来源换掉**（见 [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]]）：

$$E_{ss}=\underbrace{f_a+f_b}_{\text{Karis 的 EnvBRDF LUT 两个通道}},\qquad E_{ms}=1-E_{ss},\qquad F_{ms}=\frac{(F_0f_a+f_b)F_{avg}}{1-F_{avg}(1-E_{ss})}$$

> **一句可以直接背下来的话**：**缺口是 $1-E_{ss}$，而 $E_{ss}$ 早就已经在表里了。**

### 4b. 账本的另一半：不只是"少了"，也可能是"多了"

**2026-09-20 补**。此前本笔记只记录"能量不够"。[[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] 明确指出**另一端也有账**：

| 端 | 现象 | 原因 |
|---|---|---|
| 高粗糙度 | 表面**发暗**、颜色失饱和 | 微面间互反射的能量被丢掉 |
| **低粗糙度** | 掠射边缘一圈不自然的**亮边**（**超额**能量） | 漫反射项常被写成 $E_d=1-F_0$，忽略了**掠射角 Fresnel 升高、本该有更多光留在镜面项** |

修法（电介质）：

$$E_d=1-(F_{ss}E_{ss}+F_{ms}E_{ms}),\qquad K_d=\text{albedo}\cdot E_d$$

**实操含义**：**furnace test 要查两头** —— 粗糙端是否变暗（旧检查项），**光滑端是否有一圈偏亮**（新检查项）。后者更容易被漏掉，因为它看起来"更干净"而不是"更暗"。

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
2013  Karis（split-sum 进引擎 —— 单次散射的 IBL 变成"两次查表"）★ 本库已入库
        ↓
2014  Jakob et al.（综合分层框架；提出 F_avg 的思路）—— 完整但太重
        ↓
★ 2016  Heitz et al.（Smith 模型下多次散射的精确真值）★ 2026-09-20 入库
        —— 精确、有全参数，但求值要随机数；作者自述"unsuitable for real-time"
        ↓
★ 2017  Kulla & Conty（把 Kelemen 公式工程化：32×32 表 = 4KB + 解析 F_avg + 忽略各向异性）
        —— 被多家引擎/工作室采用 ★ 本库已入库
        ↓
2018  Hill（a：Fresnel 的几何级数展开；b：逐次弹射模拟 → 更准但每次弹射一张表，且不管 IBL）
2018  Lagarde & Golubev（credit Emmanuel Turquin）：F_avg 进一步简化为 F_0 + 只缩放已有 lobe
        —— 与 split-sum 共享同一张 DFG 表（本库第 ⑤ 条路线；原始书目条目待核实）
        ↓
★ 2019  Fdez-Agüera（JCGT，实时 IBL 版：缺口 = 1 − Ess，而 Ess 已经在表里）★ 2026-09-20 入库
        —— 零新增资源；**并且第一次处理了"低粗糙度端超额能量"**
        ↓
2010s 微面-微片统一（Dupuy/Heitz/d'Eon 2016）→ 把表面问题写成体积问题
        ↓
★ 2026  Dupuy（特制二次 NDF 上所有散射阶的精确闭式）—— 理论天花板，但无粗糙度参数
```

**三种"补法"分别在解决不同层的阻塞，这条线因此可以按"谁卡住了什么"来读：**

```text
Heitz 2016 卡在"求值要随机数"        →  Kulla-Conty 2017 放弃分布正确性，只补标量
Kulla-Conty 2017 卡在"要新增表 + path tracing 语境"  →  Fdez-Agüera 2019 复用已有表
Fdez-Agüera 2019 卡在"只覆盖 IBL"    →  仍然空着（解析光侧没有免费方案）
Dupuy 2026 卡在"没有 roughness 参数"  →  明确的开放问题
```

## Important Papers

| 论文 | 角色 | 状态 |
|---|---|---|
| [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] | 单次散射的定型（GGX + Smith） | ✅ 入库 |
| [[Karis — Real Shading in Unreal Engine 4 (2013)]] | 单次散射进引擎；明说 $n=v=r$ 是第一误差源；**④ 的一切都建立在它的 LUT 上** | ✅ 入库 |
| [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] | **路线 ③ 工程解 + 证明** | ✅ 入库 |
| [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]] | **路线 ② 理论天花板**（精确闭式） | ✅ 入库 |
| [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] | **路线 ① 精确真值（随机）**；"不可实时"这句话定义了后面所有工作 | ✅ **2026-09-20 入库** |
| [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] | **路线 ④ 实时 IBL 版**（零新增资源；首次处理低粗糙度端超额能量） | ✅ **2026-09-20 入库** |
| Lagarde & Golubev 2018（credit Emmanuel Turquin） | **路线 ⑤ 最便宜版**（只缩放已有 lobe） | ⚠️ 经 Filament 官方文档转述核实，**原始书目条目待核实** |
| Kelemen & Szirmay-Kalos 2001, Eurographics Short | 公式源头 | ❌ 未入库（较老，可只在本文引用） |
| Hill 2018, *A Multi-Faceted Exploration* part 2 / part 3 | 比 ④ 更准（逐次弹射模拟），但每次弹射一张表、且不处理 IBL | ❌ 未入库，**可作为"更准 vs 能用"的又一实例** |
| d'Eon, *A Hitchhiker's Guide to Multiple Scattering* | 系统性手册 | ❌ 未入库，**优先级高** |
| Hammon 2017, *PBR Diffuse Lighting for GGX+Smith Microsurfaces* | **漫反射侧**的同类问题 | ❌ 未入库，**优先级高** |

## Related Concepts

- [[Microfacet Theory]] — 本概念是它的**必然副产品**：有互遮挡就必然有互反射，有互反射就必然有"忽略互反射"的误差；
- [[Split-Sum Approximation]] — **同一根 IBL 管线上的相邻两步**：split-sum 解决"积分太贵"，能量补偿解决"积分算少了"。**两者叠加才是 IBL 镜面半边的完整固定开销**；
- [[Participating Media]] — 多次散射的**理论通道**：Dupuy 2026 的整个推导就是把表面写成半无限微片介质；
- [[Hair Rendering]] — **另一个"微面 BRDF 失效"的边界**（单位不同）；毛发里的多次散射由 Marschner 的 TRT 等明确路径处理，因为纤维内部吸收是主要的、可以显式建模；
- 🔴 **一条 2026-09-20 新发现的跨领域合流**：[[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] 把粗糙电介质 BSDF 按 **R/T 事件序列** 分解（TRT、TTR 等 lobe），而 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 按 **R/TT/TRT** 分解毛发散射 —— **两个完全不同的物理场景，独立收敛到"按散射事件序列分解 BSDF"**。
  → **可操作结论：[[Hair Rendering]] 里那条"TRT 先砍、TT 最后砍"的分档判据，其底层手法不是毛发专属的。**"哪个 lobe 可以先砍"的通用判据是"事件序列的可见性"。
- **Chandrasekhar 单次散射项** — 长期基准；Dupuy 2026 证明它的 2 倍恰是单位粗糙度的单次散射 GGX。

## Technologies

- **引擎侧的 "Multiple Scattering / Energy Compensation" 开关**（Unity HDRP、自研管线、部分 UE 配置）——**注意：UE 里它不是无条件默认开启的**；
- **③ 的 $E(\mu)$ / $E_{avg}$ LUT**（32×32 + 32 项 1D ≈ 4KB float）—— 一个典型的"**低成本档位开关**"：
  - 开启 = +1 次纹理采样 + 4KB 常驻；
  - 关闭 = 高粗糙度端偏暗、颜色失饱和（属于"降档可忍受"的退化类型）。
- **④ 的"零新增资源"版（2026-09-20 新增）** —— **这是本库目前"研究 → 引擎"距离最短的一项**：
  - **不新增任何表、不新增任何采样**，只增加几条标量运算 + 复用已有的 irradiance；
  - 输入就是**已有的 EnvBRDF LUT 采样结果**（UE 用的正是 **R16G16**，见 [[Split-Sum Approximation]]）；
  - **结论：在一个已经跑着 Sky Light 的场景里，它的边际成本约等于 0 → 应当默认开启，而不是当成分档项。**
- **⑤ 的"一条乘加"版** —— 连新增项都不需要（直接把已有的镜面 lobe 缩放），但**覆盖面最窄**（只补高粗糙度端，不修电介质超额能量）；
- **Substrate（UE5）** — 把能量守恒内建在分层框架里，而不是事后加一个 lobe。**注意这与 ③④⑤ 是两条哲学**：一条"内建"，一条"事后补标量"。**事后补只能补方向 albedo 这一个数，补不回分布形状**（[[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] 是这句话的证据）。
- **⚠️ 覆盖边界（必须写进任何技术文档）**：**④⑤ 只修 IBL，不修解析光。**"环境光那半边修好了" ≠ "能量补偿已经做了"。

## Game Applications

- **材质观感**：高粗糙度金属/塑料"发闷"的**标准解释与标准解**；**低粗糙度电介质掠射边缘的"亮边"**是同一个账的另一端（§4b）；
- **分档**：见 [[Scalability and Quality Tiers]] —— 这是一条"**关掉后能看出但能忍**"的档位差异，适合放在中低档关闭。**但注意：只有 ③ 值得当成分档项；④⑤ 的边际成本约等于 0，应当默认开启**；
- **⚠️ 一个容易误判的场景**：**在动态灯光/解析光占主导的场景（如 MegaLights 大量点光）里，④⑤ 不解决能量问题。**"IBL 侧已修"容易被误读成"能量补偿已完成"；
- **资产规范**：Kulla-Conty 原文提到美术把木材 IOR 调到 100+ 的案例。**建议在资产取值表里加入 IOR / 金属度 / F0 的允许范围** —— 这类"资产侧不物理"比性能问题更容易漏，而且会让不同档位的行为不一致；
- **材质验收**：**furnace test** 可以直接作为材质检查项（不需要参考照片），**且要查两头**（暗 / 亮）。

## Personal Knowledge

- **user_level: Normal。** 前置只有 [[Microfacet Theory]] 与 [[BRDF]]，都已经在库；
- **本概念不计入 PBR 的 25 条收口清单** —— 它是清单之外的**最后一条具名缺口**，标为"读了，不是会了"；
- **一句话检验**：能说出"被挡住的光不是被吸收了，所以单次散射把它当成消失是错的"即算抓住核心；
- **第二条一句话检验（2026-09-20 新增）**：能说出 **"环境光那半边修好了，不代表灯光那半边也修好了"**。

## Learning Gap

1. **推导层（Hard，不必现在动）**：Dupuy 2026 的 Poisson 核卷积封闭与级数求和；**Heitz 2016 的 Smith 随机输运自由程 / 相位函数推导**（两者都属于"知道结论就够"的一类）。**不要为了这些停住 PBR 的收口；**
2. **数值层（Normal，可快速补）**：$E(\mu)$ 与 $E_{avg}$ 的具体形状（作为 roughness 与 $\mu$ 的函数）。**最好的补法是看一眼 Kulla-Conty 的那两张表**；**更快的补法是看 ④ 的那一行 `Ess = f_ab.x + f_ab.y`** —— 它告诉你这张表在你引擎里已经在跑了；
3. **实测层（Easy，可以先做）**：引擎里那个 Furnace Test（两头都查）。**这是 25 条自测清单里唯一的引擎侧实测题。**
4. **实现层（Normal，可选但有价值）**：④ 的 GLSL 论文自带、零新增资源 —— **如果实测发现引擎当前没有能量补偿，补它几乎是"照抄 + 验证"的工作量**。

## Next Step

1. **把 furnace test 做成一次真实的自测**（30 分钟内可完成）：
   纯金属球 → Roughness 0→1 → 只有环境光 → 截图；**再加一个光滑白电介质球查掠射亮边**；然后切换引擎侧的能量补偿开关对比。**做完这一条，本概念即可从 Normal 升 Easy**；
2. **经典候选排队（2026-09-20 更新）**：~~Heitz et al. 2016~~ ✅ 已入库 → ~~Fdez-Agüera 2019~~ ✅ 已入库 → 现在最高优先级是 **Hammon 2017**（`PBR Diffuse Lighting for GGX+Smith Microsurfaces` —— **漫反射侧也漏能量**，与 diffuse 认知直接相关）> **d'Eon, *A Hitchhiker's Guide to Multiple Scattering***（系统性手册）> **Hill 2018 part 2/3**（"更准 vs 能用"的又一实例）；
3. 与 [[Split-Sum Approximation]] 合并成一张"IBL 镜面半边固定开销表"（cubemap 预滤波 + EnvBRDF LUT + 能量补偿），纳入 [[Real-Time VFX Performance Budgeting]] 的参考账。**④ 让这张表多了一行"成本 ≈ 0"**。

## Visualization

- [[多次散射_五条补法路线与实时落地图解]]（2026-09-20）—— **当前版本**：五条路线对照表 + Heitz 随机游走单步流程 + Fdez-Agüera 数据流 + 两头 furnace test 检查项 + 勘误提醒；
- [[多次散射能量补偿_三条路线图解]]（2026-09-19）—— 早期版本（当时只有三条路线），第 5 节仍是 **furnace test 的具体做法**，仍可参考。

## Notes

- 本概念与 [[Split-Sum Approximation]] 是**同一天方法论的两个实例**：**"想省，就先问自己额外假设了什么"**；
- ⚠️ 归属提醒：**Kulla-Conty 的补偿公式本体来自 Kelemen 2001**，Kulla-Conty 的贡献是把公式做成可用的工程方案并补上证明。**不要写成"Kulla-Conty 发明了能量补偿"。**同理，**Heitz 2016 不是"发明了能量补偿"，而是第一次把输运推对**（原文自述：在图形学与物理两侧，**首篇**完整推导并验证 Smith 微面散射辐射度量学的工作）；
- ⚠️ **勘误提醒（2026-09-20 新增，务必沿用）**：**本概念涉及的两篇工程文献都有官方勘误** —— [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] 的 $F_{avg}$ 一式（Emmanuel Turquin 指出，v2 修正）；[[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] 的 6/7/10/11/15 式与 Listing 1、2（JCGT 论文页勘误栏，2018-02-01 修订）。**错误全在常数与因子层** —— 照抄进 shader 不报错，只会慢半拍地表现成"看起来有点不对"。**以后处理预计算/近似类论文，先找勘误页。**
