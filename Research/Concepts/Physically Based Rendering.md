---
type: concept
user_level: Normal
aliases: [PBR, Physically Based Shading]
prerequisites: [BRDF, Radiometry]
first_introduced: "理论奠基 1981（Cook-Torrance）；生产普及 2012-2013（Disney / UE4）"
---

# Physically Based Rendering

## Definition

以物理约束组织材质与光照的着色范式：材质用**能量守恒的微面 BRDF**描述，光照用**真实物理单位**表达，二者在渲染方程框架内求值。工程上的"三件套"：微面 BRDF（镜面项）+ 金属度工作流（参数化）+ 基于图像的光照（IBL，环境项）。

## Core Principle

PBR 的价值不在"更真实"，而在**更可预测**：

- 材质参数有物理含义 → 资产在任何光照环境下行为一致（"一次制作，到处正确"）；
- 美术不再为每个镜头手调高光颜色——Fresnel 自动给出；
- 能量守恒 → 光照强度可以按真实单位累积，场景打光有章可循。

对管线而言，PBR 本质上是**用物理约束消灭调参自由度**，把"美术手感"变成"可复用的资产规范"。

## Historical Evolution

```text
BRDF 形式化（Nicodemus 1977）
        ↓
微面理论（Torrance-Sparrow 1967 → Blinn 1977）
        ↓
★ Cook-Torrance 1981：完整物理框架（离线渲染逐渐采用）
        ↓
★ [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]：GGX 分布 + Smith height-correlated G（长尾更贴实测，**实时 PBR 的 D 与 G 两项都出自此文**）
        ↓
Disney Principled BRDF（2012，《无敌破坏王》生产验证：原则化参数化）
        ↓
★ Karis 2013：UE4 实时 PBR（Schlick F + GGX D + Smith G + split-sum LUT）
        ↓
全工业默认（UE / Unity / 自研引擎全部收敛到金属度工作流）
        ↓
★ Heitz 2016（多次散射的随机真值）→ ★ Kulla-Conty 2017（4KB 表补回能量，工程可用）★ 2026-09-19 入库
        ↓
Dupuy 2026 —— 特制 NDF 上所有散射阶的**精确初等闭式**（理论天花板，代价是无 roughness 参数）★ 2026-09-19 入库
        ↓
Substrate（UE 5.2+）：分层 lobe 框架，PBR 的可组合化扩展
```

## 实时化的关键近似（UE 版）

| 物理项 | 精确形式 | 实时近似 |
|---|---|---|
| F | 完整 Fresnel（波长相关） | Schlick 近似：$F_0 + (1-F_0)(1-\cos\theta)^5$（Karis 版用球面高斯去 pow） |
| D | Beckmann / 任意 NDF | **GGX**（[[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]），$\alpha=\text{Roughness}^2$ |
| G | 微面遮挡积分 | **Smith 近似**（UE 用 height-correlated Smith 变体，同源同文）；Karis 改写成 Schlick 形式配 $k=(\text{Roughness}+1)^2/8$ |
| 环境镜面 | 预滤波卷积 | **[[Split-Sum Approximation]]**：预滤波 cubemap mip 链 + EnvBRDF LUT（R16G16） |
| 环境漫反射 | 半球卷积 | SH 投影 9 系数（[[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]]） |
| **丢失的能量** | 微面间多次散射的真实输运 | **[[Multiple Scattering and Energy Compensation]]**：加一个 $(1-E(\mu_o))(1-E(\mu_i))/\pi(1-E_{avg})$ 的 lobe（32×32 表 ≈ 4KB），或选择不补（高粗糙度端偏暗） |

理解这张表 = 理解"实时 PBR 里没有新物理，只有便宜的近似"——这是评估任何"新着色技术"的基准姿势。**逐项来源见 [[Karis — Real Shading in Unreal Engine 4 (2013)]]（进引擎的那一步）。**

> **最后一行是 2026-09-19 补的，也是最容易被漏掉的一行**：前面所有近似都只影响"形状对不对"，**这一行影响的是"总量对不对"**。而且它**默认不一定开启** —— 所以在引擎里看到高粗糙度材质发闷，先查这一项。

## Prerequisites

- [[BRDF]]（数学核心）
- [[Microfacet Theory]]（★ D·G·F 的完整拆解与 45 年演化链，见 [[Microfacet D·G·F 几何图解]]）
- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]（框架源头）
- [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]（**你实际在用的 D 与 G 的来源**）

## Related Concepts

- [[Participating Media]]（表面 ↔ 介质的对偶；2026 工作已把两者统一）
- [[Multiple Scattering and Energy Compensation]] ★ 2026-09-19（PBR 的"能量账本"；本概念 Learning Gap 的最后一条）

- [[Real-Time Rendering]]
- [[Inverse Rendering]]（PBR 参数是逆渲染要反解的目标）

## Game Applications

- UE 默认 Lit Shading Model；
- 材质预算：基础 PBR ≈ 固定开销（LUT 查表），多 lobe（Clear Coat 等）≈ 成倍开销——Shading Model 数量是 DrawCall 之外的隐藏预算维度；
- Substrate 分层材质的性能风险：lobe 组合数爆炸。

## Important Papers

- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]——直接光 / 镜面侧
- [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]]——环境光 / 漫反射侧（IBL 的 SH 半边）
- [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]]——F 项
- [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]——D（GGX）与 G（Smith）
- [[Karis — Real Shading in Unreal Engine 4 (2013)]]——★ **实时化那一步（入库 2026-09-18）**：三因子换廉价形式 + [[Split-Sum Approximation]] 环境光 + 材质模型定型（BaseColor/Metallic/Roughness，非金属 $F_0$=0.04）
- [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]——★ **能量侧那一步（入库 2026-09-19）**：单次散射丢掉的那部分能量怎么补回来（32×32 表 ≈ 4KB）+ **Furnace Test** 这个可执行的自测方法
- [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]]——★ 理论天花板（2026，所有散射阶的精确初等闭式）

> **至此本概念三侧齐备：来源侧（D·G·F 的物理出处）2026-09-17 闭合，工程侧（进引擎 + IBL 查表）2026-09-18 闭合，能量侧（多次散射）2026-09-19 闭合。**

## Personal Knowledge

Current Level: **Normal**（主动研读 D/G/F 物理来源中，2026-09 信号）

## Learning Gap

- ~~D/G/F 物理来源（随 Cook-Torrance 线收口）~~ ✅ 2026-09-17 收口（D/G 于 9-16、F 于 9-17）
- ~~split-sum 预积分的推导直觉~~ ✅ 2026-09-18（[[Karis — Real Shading in Unreal Engine 4 (2013)]] + [[Split-Sum Approximation]]）
- ~~多次散射能量补偿~~ ✅ 2026-09-19（[[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] + [[Multiple Scattering and Energy Compensation]]）
- **剩余唯一缺口：你自己的 IBL 分档实测数据**（见 Next Step 第 2 条）。**这是清单里唯一需要动手的一条，其余都是纸面自测。**

## Next Step

1. **收口自测**：Cook-Torrance / Kajiya / Walter / Schlick / Karis **各 5 条，共 25 条**（Karis 的 5 条在 [[Karis — Real Shading in Unreal Engine 4 (2013)]]）。**全过即可把 [[BRDF]] 与 PBR 同时标 Easy**——它们是同一知识体的两个切面。
2. **一次可做的实测（30 分钟，唯一需要动手的一条）**：**Furnace Test** —— 纯金属球 + 只有环境光 + Roughness 0→1 截图 + 切换引擎侧多次散射补偿对比。做法见 [[多次散射能量补偿_三条路线图解]] 第 5 节。**做完即可把 [[Multiple Scattering and Energy Compensation]] 标 Easy。**
3. **另一次可做的实测**（不需读论文）：同一场景下 **Sky Light 镜面开/关** 与 **LUT 精度 R16G16→R8G8** 两组对比的 GPUTime / 显存差 → 可直接支撑反射类材质与特效的分档表。
