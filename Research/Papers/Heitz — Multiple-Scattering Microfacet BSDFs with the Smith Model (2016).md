---
type: paper
title: "Multiple-Scattering Microfacet BSDFs with the Smith Model"
authors: [Eric Heitz, Johannes Hanika, Eugene d'Eon, Carsten Dachsbacher]
year: 2016
published: 2016-07-11
venue: "ACM Transactions on Graphics (TOG) 35(4), SIGGRAPH 2016, 58:1–58:14"
url: "https://jo.dreggn.org/home/2016_microfacets.pdf"
code: ""
project_page: ""
doi: "10.1145/2897824.2925943"
category: [rendering, brdf, microfacet, multiple-scattering]
importance: A
historical_importance: 5
game_relevance: 3
production_readiness: Research
user_level: Hard
status: read
---

# Multiple-Scattering Microfacet BSDFs with the Smith Model

> 入库于 2026-09-20。原文 PDF 已下载并**逐节核对**（14 页，KIT 官方镜像 `jo.dreggn.org`）。
> **它是多次散射"三角形"的最后一只角**：[[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] 给工程解、[[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]] 给理论闭式，**这篇给的是"真值参照系"** —— 精确，但**求值本身要随机数**。

## TL;DR

把 Smith 微表面模型下的**多次散射推出来**（不是事后补一个 lobe），核心洞见是：

> **Smith 微面理论 = 微片（microflake）体积理论的一个特例，只是额外加了一条"强制锐利界面"的约束。**

于是"光在微面之间反复弹射"这件事被翻译成"光在一团特制介质里的随机游走"，用一个**很短的随机游走**（粗糙度合理时只需几次弹射）算出完整 BSDF。

**结果的性质**：精确（含所有散射阶）、能量守恒、互易、支持各向异性 Beckmann/GGX、**不使用任何预计算数据**（因此支持带纹理的 albedo / roughness / anisotropy）。
**代价**：**没有闭式**，求值要随机数；作者自己在 Limitations 里写了一句话，基本判定了它的工程定位 ——

> **"our model is unsuitable for real-time rendering where the shading has to be smooth with one sample per pixel."**

## Problem

微面理论在图形学里统治了 30 多年（近几十年几乎所有参数化 BSDF 都用它或被它启发），但**几乎所有流行 BSDF 只算单次散射**：

- $D(h)\,G\,F$ 里面的 $G$ 负责"这次弹射没被挡住"；
- 光**被挡住**时，单次散射模型把这份能量**当作消失**；
- 真实微结构里，被挡住的光会**弹到别的微面上**，最终仍有机会离开表面。

**"被挡住" ≠ "被吸收"。** 这就是全部能量丢失的来源。原文把这件事的后果写得很直接：需要正确计入微面间多次散射才能**保证能量守恒**，也才能表现出**粗糙金属高光里的强色彩饱和**、以及**粗糙电介质板的透射**。

因此原文明确把它列为微面理论**公认的未解问题**。

## Historical Context

| 年份 | 节点 | 贡献 |
|---|---|---|
| 1963 | Beckmann & Spizzichino | 用"随机朝向微面"建模粗糙表面的概念起源 |
| 1967 | **Smith** | 掩蔽（masking）统计模型：**高度与法线独立**——全篇的地基 |
| 1967 | Torrance & Sparrow | 物理侧的微面反射理论 |
| 1981 / 1982 | **Cook & Torrance** | 把单次散射微面模型引入图形学。⚠️ **同一工作有两个版本**：SIGGRAPH 1981 会议版（*Computer Graphics* 15(3)）与 *TOG* 1982 期刊版（1(1):7–24）。**本文引用的是 TOG 1982 版**，与本库 [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 不冲突，是同一研究实体的两个入口 |
| 2001 | Stam | 推广到反射 + 透射，形成完整 BSDF 框架 |
| 2001 | Kelemen & Szirmay-Kalos | 第一个通用能量补偿公式（本库 [[Multiple Scattering and Energy Compensation]] 已注明：**它是公式本体来源**） |
| 2007 | **Walter et al.** | GGX + Smith（本库已入库），单次散射形态定型 |
| 2014 | Heitz | 《Understanding the Masking-Shadowing Function》——本库 [[Microfacet Theory]] 的关键前置 |
| 2014 | **Heitz & d'Eon** | VNDF 重要性采样——**本篇相位函数采样的直接基础** |
| 2014 | Jakob et al. | 分层材质框架（推广 Kelemen 的启发式），但继承其**方位角不变性**这一非物理性质 |
| 2015 | Heitz et al. | **SGGX 微片分布**——把"体积里的各向异性散射"变成可用的数学对象 |
| **2016** | **本文** | **Smith 微面的完整多次散射辐射度量学** |

## Previous Work（原文的批评，值得单列）

原文对本库已有的两条路线给了**明确的技术批评**，这是判断"谁解决到哪一层"的依据：

| 工作 | 原文批评 |
|---|---|
| **Kelemen & Szirmay-Kalos 2001** | "**enforces an azimuthally invariant multiple-scattering lobe, which our ground-truth simulation shows to be inaccurate**" —— 它强制多次散射 lobe **方位角不变**，而本文的真值模拟显示**这一点不准**。另外**不处理各向异性与透射** |
| **Jakob et al. 2014** | 继承了同样的**非物理方位角不变性**；且实现很重（需要预计算傅里叶系数表），**对带纹理的材质适用性有限** |
| **两者共同的根本区别** | "**these BRDFs are not associated with microsurface models**" —— 它们**不与任何微面模型绑定**，不对"某个具体假设下的微表面会产生什么输运"做预测。**所以它们只能算"强制能量守恒的技巧"，不是模型。**而本文要的是"Smith 假设下**真实预测**的多次散射"，**能量守恒是副产品** |
| Oren-Nayar 1995 | 只做 V 形槽 + Lambertian 的**双次散射** |
| Koenderink et al. 1999 | 球面凹陷的解析互反射，形状特例 |

> **一句可以直接拿去用的判据**：**"补能量"和"推输运"是两件事。** 前者不依赖任何物理假设（所以永远对，但分布可能是编的）；后者依赖假设（所以分布对，但假设错了就全错）。**看到任何能量补偿方案，先问它是哪一类。**

## Core Idea

**把"表面"写成"体积"的极限。**

给定 NDF（Beckmann / GGX）+ 材质（导体 / 电介质 / 漫反射），**推导出一组自由程分布（free-path distribution）与相位函数（phase function）**，使得：

- 这团"微体积"的**平面平行散射结果，恰好等于 Smith 微面的 BSDF**；
- 并且**额外包含了高阶散射**。

一个容易误解的点，原文专门说明：**这个体积散射过程是"虚拟的"** —— 入射点与出射点**同一个位置，不发生任何位移**，所以出来的平面平行辐射度量学**仍然是一个 BSDF**，可以直接塞进现有渲染器。

> **这个"表面 = 体积极限"的写法，三天前（9-16）刚以另一个方向出现过**：[[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]] 从随机几何**推导出** Smith 的独立性假设；今天这篇**直接拿这个等价关系当工具**。**同一个等价关系，一篇在推它、一篇在用它的十年前版本**（GPIS 的对应关系早在 2010 年代的微面-微片统一工作里就建立了，见 Dupuy/Heitz/d'Eon 2016）。

## Technical Approach

### 1. 为什么必须放弃"乘积式" G2

经典微面用 $G_2(\omega_i,\omega_o,\omega_m)=G_1(\omega_i,\omega_m)G_1(\omega_o,\omega_m)$（Walter 2007 的做法），即**忽略高度相关性**。

但随机游走**必须知道光当前在哪一层高度**上，所以本文改用**高度相关**的 $G_2$（来自 Heitz 2014）：

$$G_2(\omega_i,\omega_o,\omega_m)=G_1^{local}(\omega_i,\omega_m)\,G_1^{local}(\omega_o,\omega_m)\,G_2^{dist}(\omega_i,\omega_o)$$

另一个关键构件是 Smith 的 $\Lambda$ 函数进入**远距离掩蔽**：

$$G_1^{dist}(\omega_i,h)=\big(C_1(h)\big)^{\Lambda(\omega_i)},\qquad G_1^{dist}(\omega_i)=\int G_1^{dist}(\omega_i,h)P_1(h)\,dh=\frac{1}{1+\Lambda(\omega_i)}$$

**一个顺带的结论**（原文明确写出）：在 Smith 假设下，**最终反射率与高度分布的具体选择无关** —— 用均匀高度分布或高斯高度分布，得到**同一个 BSDF**。这解释了为什么引擎里从来不需要"高度分布"这个旋钮。

### 2. 算法心脏：一行公式采样下一个交点高度（Alg. 1）

随机游走的每一步只做一件事 —— **下一个交点在哪一层高度**：

```text
Sample height  h_{r+1}(ω_r, h_r, U)
  if  U ≥ 1 − G1_dist(ω_r, h_r, ∞)      →  离开微表面， h_{r+1} = ∞
  else                                  →  与微表面相交
        h_{r+1} = C1^{-1}( C1(h_r) · (1−U)^{1/Λ(ω_r)} )
```

- 一句话读法：**"逃逸概率"与"下一层高度"共用同一个随机数**，两件事一次搞定；
- 游走初始化：能量吞吐 $e_1=1$、初始高度 $h_0=\infty$、初始方向 $\omega_1=-\omega_i$；
- 原文用 Table 3 验证这个交点模型符合 height-field 应有的性质：**向下走的射线必然相交且不会低于最低点；向上走的射线可能离开、也可能停在更高处**。这是一次**自洽性检验**（模型是否满足已知几何性质），不是数据拟合。

### 3. 相位函数：从微片框架推出来，又回归经典

因为是"高度场"，交点处的散射必须按微面法线分布来做。本文从微片框架推出相位函数，并按 VNDF 归一（对 $\omega_i$ 方向做掩蔽归一化），于是它**单向不互易**、但**整体 BSDF 互易**。

**一次很漂亮的交叉验证**：推出的可见法线分布是

$$D_{\omega_i}(\omega_m)=\frac{\langle\omega_i,\omega_m\rangle D(\omega_m)}{\cos\theta_i\,(1+\Lambda(\omega_i))}$$

—— **与经典微面推导（Heitz & d'Eon 2014 的 VNDF）完全一致**。原文自己把这一步当作"微片解释与经典微面推导相容"的证据。

## Key Contribution

1. 导出 Smith 微面**对应的体积介质的自由程分布**（Sec. 5）；
2. 导出**相位函数**（Sec. 6）；
3. 提出基于自由程与相位函数的**随机游走**方法（Sec. 7）；
4. 定义 Smith 模型的**多次散射 BSDF**（随机游走的统计期望，Sec. 8）；
5. **工程化**：解析重要性采样、无偏随机求值、以及三种降方差手段；并给出"作为普通材质插件接入渲染器"的说明。

## Why It Works

因为 **Smith 假设本身（高度与法线独立）就能被翻译成一组体积统计量**。一旦"表面"变成"半无限高密度介质"，多次散射就不再是需要发明的东西，而是**标准体积输运的自然结果**。所以：

> **能量守恒在这里不是约束、不是修正项，而是模型正确性的副作用。**

这解释了它为什么能同时做到"精确"和"支持各向异性"：**它没有引入任何额外的外观假设**，所有近似都在 Smith 假设这一层。

## Validation（都是原话与原文数字）

### 1. 对照显式随机 Beckmann 表面的光线追踪

用 Heitz 2015 的 Beckmann 表面生成法造出真实表面并做 raytrace，逐项比对：

- **自由程分布**（Fig. 14）与**出射方向与能量**（Fig. 15）均吻合。

**各向异性 Beckmann 实例（$\alpha_x=0.1,\alpha_y=1.0$，入射角 $\theta_i=1.5$）** —— 原文给出的对比数字：

| 材质 | 量 | 本文模型 | 模拟真值 |
|---|---|---|---|
| 电介质 | $E_r$（1/2/3 次弹射） | 0.395 / 0.098 / 0.008 | 0.403 / 0.088 / 0.010 |
| 电介质 | $E_t$（1/2/3 次弹射） | 0.426 / 0.065 / 0.006 | 0.434 / 0.057 / 0.005 |
| 导体（$F=1$） | $E_r$ | 0.542 / 0.398 / 0.059 | 0.561 / 0.389 / 0.049 |
| 漫反射（$a=1$） | $E_r$ | 0.762 / 0.181 / 0.056 | 0.774 / 0.174 / 0.051 |

### 2. White Furnace Test（Fig. 16）

常数白色环境 + albedo 为 1 时，**物体完全消失**。导体、电介质、漫反射三类微面材质**全部通过**。

### 3. 数值验证（补充材料）

一组单元测试：能量守恒、互易性、**高度采样与掩蔽阴影函数的一致性**、**与经典单次散射 BSDF 的一致性**，另有按 **R/T 事件序列**分解的 lobe 明细。

> 🔴 **这里有一个对本库很有价值的连接**：原文把粗糙电介质的 BSDF 按**反射(R)/透射(T) 的事件序列**分组（**TRT、TTR 等**），"每一个都简单，加起来才形成完整的复杂 lobe"。
> **这与 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 的 R / TT / TRT 是同一种思路** —— 一个来自纤维散射，一个来自微面输运，**两个完全不同的物理场景独立收敛到"按事件序列分解 BSDF"**。
> **给你的方法论价值**：**"哪个 lobe 可以先砍"的判据，在两个领域里都是"事件序列的可见性"**（毛发侧已总结为"TRT 先砍、TT 最后砍"，见 [[Hair Rendering]]）。这套判据不是毛发专属的。

## 关键量化（性能与方差，全为原文数字）

| 场景 | 相对单次散射的渲染时间 |
|---|---|
| Fig. 1 电介质板（纹理化 GGX，$\alpha=0\ldots1$，$\eta=1.5$） | **+19%**（并达成 100% 能量守恒） |
| Fig. 17 bottles 场景 | **+87%（"几乎翻倍"）** |
| Fig. 18 粗糙导体（GGX $\alpha=0.3$，光谱金 Fresnel） | **+24%** |
| Fig. 19 漫反射 / 电介质，$\alpha=0.1/0.5/1.0$ | **+6% / +50% / +62%** |

- **成本随粗糙度上升而上升**（原文原话："higher roughness will typically lead to more indirect bounces"）—— 这与"粗糙度越高丢的能量越多"是同一件事的两面；
- **方差**：加入多次散射后图像**方差约 +20%**；
- **视觉**：粗糙导体的**色彩饱和度明显提升**（Fig. 18）；不加多次散射的粗糙透射"看起来不自然，而且**很难靠调参补偿**，尤其是带纹理的粗糙度时"（Fig. 1 / 20）；
- **一个标志性数字**：**双次散射在某些情况下可占总反射或透射的 20%** —— 即"少收的那笔钱"不是零头。

## Limitations（原文自述，按重要性排序）

1. **🔴 不适合实时。** 原话：**"our model is unsuitable for real-time rendering where the shading has to be smooth with one sample per pixel."** 这是本笔记里最该记住的一句 —— 它**定义了后来所有实时方案要解决的问题**：
   - [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]：**不追求分布正确，只把方向 albedo 这个标量补平**（查表）；
   - [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]]：**复用已经存在的单次散射预计算表**。
2. **随机求值引入方差**（前述 +20%）。可部分缓解：单次散射项用闭式替代，只在高阶留方差；
3. **与其他积分器的兼容性**：Metropolis Light Transport 一类"消耗非确定数量随机数"的积分器效率会下降。

## 它是怎么把方差压下来的（三条，工程上可迁移）

| 手段 | 做法 | 代价 |
|---|---|---|
| **单次散射项换成闭式** | 把第一次弹射的贡献 $E_1$ 直接用闭式单次散射 BSDF 代替（等价于用平均的高度相关 $G_2$ 替换 $h_1$ 处取值） | 无（期望相同，**不引入偏差**） |
| **把"是否相交"的伯努利采样积分掉** | 强制游走继续，改用**乘上相交概率**来补能量；代价是需要一个终止判据（能量阈值或最大弹射数） | 需要终止条件 |
| **双向随机游走 + MIS 式权重** | BSDF 互易 ⇒ 从 $\omega_i$ 或 $\omega_o$ 起走期望相同，但**方差不同** → 随机选起点，给"贡献大的一侧低权重、贡献小的一侧高权重" | 实现复杂度 |

> **第一、二条合起来是一条通用工程姿势**：**"把一个随机决策替换成它的期望，然后手动记账"** —— 这与 [[Karis — Real Shading in Unreal Engine 4 (2013)]] 的"把能预存的移出运行时"是同一种思维方式的不同实例。

## Game Development Relevance

- **它是真值参照系，不是生产方案。** 任何实时能量补偿方案（查表 lobe / 缩放 lobe / 复用 LUT）的**正确性最终都定义在它上面**。**问"这个补偿方案准不准"，答案的标准就是"与 Heitz 2016 的模拟差多少"**；
- **"按事件序列分解 lobe"是一条跨材质的分档判据**（见上，与 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 的合流）→ 可直接支撑"这一档砍哪个 lobe"的决策；
- **"成本随粗糙度上升"** 对分档有直接含义：**同一个材质在不同粗糙度下的多次散射开销差别很大（+6% ~ +62%）**，所以"要不要开"不能按材质类型一刀切；
- **它给出了"能量补偿能改善什么"的视觉清单**：粗糙金属的**色彩饱和度**、粗糙电介质的**透射自然度**（且"很难靠调参补偿"）。

## Unreal Engine Relevance

- **UE 里没有、也不可能有它的实现**（随机求值 + 每像素多点采样，与光栅管线根本冲突）；
- 但它解释了 **UE 的材质模型为什么必须"内建"能量守恒而不是"事后补"**：（Substrate 的取向）——**事后补只能在方向 albedo 这一个标量上补平，分布形状补不回来**；
- **一个负向结论也值得记住**：**"物理更真实的方案不可用"在渲染里是常态**（本库里第 N 次出现）。**工程决策要按"哪条轴在评价"来分**：Heitz 2016 赢的是"分布正确性"，Karis 2013 赢的是"能不能预存"，Kulla-Conty 赢的是"能不能进引擎"。

## Technology Evolution

```text
1963  Beckmann & Spizzichino（随机微面几何）
1967  Smith（高度与法线独立 —— 全篇地基）
1981/82 Cook-Torrance（进图形学；单次散射形态定型）
        ↓
（此后 30 余年：能量丢失被当成"美术问题"）
        ↓
2001  Kelemen & Szirmay-Kalos（第一个通用补偿；但多次散射 lobe 方位角不变 = 非物理）
2014  Jakob et al.（推广到透射；继承同一非物理性；实现重）
2014  Heitz（掩蔽阴影综述）· Heitz & d'Eon（VNDF 采样）
2015  Heitz et al.（SGGX 微片分布 —— 微面↔微片统一的关键工具）
        ↓
★ 2016  Heitz et al. —— 本文：Smith 微面的完整多次散射（精确真值，随机求值，不可实时）
        ↓
2017  Kulla-Conty（放弃分布正确性，只补标量 → 可进引擎）
2018  Lagarde & Golubev（credit Turquin）：只缩放已有 lobe（零新增项，见下）
        ↓
★ 2019  Fdez-Agüera（复用已有 EnvBRDF LUT —— 实时 IBL 侧收口）
        ↓
★ 2026  Dupuy（特制 NDF 上所有散射阶的精确闭式 —— 唯一缺的变成"粗糙度参数"）
        ↓
（开放问题）带 roughness 参数的闭式 NDF？
```

## Relationships

### Based On

- **Smith 1967** —— 高度与法线独立，全篇唯一依赖的物理假设；
- [[Microfacet Theory]] —— 微面框架本身；
- **Heitz 2014**（掩蔽阴影）· **Heitz & d'Eon 2014**（VNDF 采样）· **Heitz et al. 2015**（SGGX 微片分布）—— 三个直接工具来源；
- **Jakob et al. 2010**（微片理论）—— "表面写成体积"的理论通道。

### Extends

- [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] —— 单次散射 Smith BSDF 的**严格超集**（本文证明：随机游走的单次散射分量**恰好**产生它）；
- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] —— 从"框架"推到"框架 + 被框架忽略的那部分"；
- **Kelemen & Szirmay-Kalos 2001 / Jakob et al. 2014** —— 从"强制能量守恒的技巧"升级为"由假设预测的输运"。

### Related

- [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]] —— 🔴 **三角形的一角对另一角**：**同一问题，同一个理论通道（微面↔微片），两种答案** —— Dupuy 精确且有闭式但**无粗糙度参数**；Heitz 精确且有全参数但**无闭式**。**两者都在 2016 年那篇微面-微片统一工作的延长线上**；
- [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] —— **对它的直接工程回应**：放弃分布正确性，只补方向 albedo；
- [[Marschner — Light Scattering from Human Hair Fibers (2003)]] —— **不是继承关系，是"同一种分解手法"**：按散射事件序列把 BSDF 拆成可单独取舍的 lobe（R/TT/TRT ⟷ R/T/TRT/TTR）；
- [[Participating Media]] —— 本文的技术通道就是"参与介质"的辐射度量学；
- [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]] —— **三天前的另一侧**：GPIS 从随机几何**推导** Smith 假设，本文**使用**同一等价关系（早十年的版本）；
- [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]] —— **反向对照**：Schlick 用"廉价到可任意求值"换掉精确 Fresnel 的可分解性，本文用"不可求值"换精确分布。**同一个权衡轴的两个极端**。

### Followed By

- [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]（2017，工程解）
- [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]]（2019，实时 IBL 解）
- **Hill 2018**（Self Shadow *A Multi-Faceted Exploration* part 2/3）—— 用路径追踪模拟逐次弹射的能量，比本文更准但需要**为每次弹射各烘一张表**，且**只处理解析光、未处理 IBL**；
- [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]]（2026，闭式解）

## Personal Knowledge State

- **user_level: Hard**（**结论层可按 Normal 读**）。
  - **Hard 的是推导**：Smith 随机输运的自由程分布与相位函数推导（Sec. 5–6）、微片框架的辐射度量学；
  - **Normal 可直接拿走的是四条结论**（不需要任何推导）：
    1. **"被挡住 ≠ 被吸收"**，这是能量丢失的全部来源；
    2. **它不可实时**（作者原话），所以实时方案必然"放弃分布、只补标量"；
    3. **成本随粗糙度上升**（+6% ~ +62%）；
    4. **"按事件序列分解 lobe"是跨材质的通用分档手法**；
  - **⚠️ 与 PBR 收口清单的关系**：**不计入 25 条**。它是清单外那条缺口的**验证手段**，不是清单项。
- **前置**（都在库）：[[Microfacet Theory]] · [[BRDF]] · [[Physically Based Rendering]] · [[Participating Media]]。

## Mastery Criteria（5 条，纸面自测）

1. 用一句话说出**单次散射丢能量的机制**，并说明为什么损失**随粗糙度上升、在掠射角最严重**；
2. 说出本文把"表面"变成"体积"之后，**哪一条性质是"副产品"而不是"约束"**（答案：能量守恒）；
3. 说出本文**为什么必须放弃乘积式 $G_2$**（答案：随机游走需要知道当前高度，必须高度相关）；
4. 说出本文**为什么不能用于实时**（作者原话层面），并说出实时方案为此放弃了什么；
5. 说出 **Alg. 1 那一行公式在做哪两件事**（逃逸判定 + 下一层高度采样，共用同一个随机数）。

> **⚠️ 归属提醒**：本文是**多次散射真值的第一篇完整推导**（原文自述："to the best of our knowledge, we are the first in either graphics or physics to derive and validate the complete radiometry of Smith microsurface scattering"）—— **但"第一个补能量"的是 Kelemen 2001**，两者不是一回事。**不要写成"Heitz 发明了能量补偿"。**

## Visualization

图解见 [[多次散射_五条补法路线与实时落地图解]]（同一张图覆盖本笔记、[[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]、[[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] 与 [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]] 四条路线 + Furnace Test 的做法）。

## Notes

- 2026-09-20 入库。**原文 PDF 已下并逐节核对**（14 页，来源 `https://jo.dreggn.org/home/2016_microfacets.pdf`，KITopen 记录 ID 1000058892）。本笔记中所有数字（+19% / +87% / +24% / +6%·+50%·+62% / 方差 +20% / 双次散射占比 20% / Fig. 15 的 $E_r,E_t$ 表）均取自原文，未使用二手转述。
- **⚠️ 引用版本提醒**：本库 [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 用的是 SIGGRAPH 1981 会议版；本文引用的是 *TOG* 1982 期刊版（1(1):7–24）。**同一研究实体，两个版本，都正确。**
- 作者的致谢里有一句很能说明这篇论文的血统：感谢 **Joe Letteri**（Weta）"many inspiring conversations encouraging the volume interpretation of surfaces" —— **"把表面当体积"这个洞见来自电影渲染一线的对话**。
