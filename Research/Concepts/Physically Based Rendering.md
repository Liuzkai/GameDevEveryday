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
GGX 分布（Walter 2007，长尾更贴实测）
        ↓
Disney Principled BRDF（2012，《无敌破坏王》生产验证：原则化参数化）
        ↓
★ Karis 2013：UE4 实时 PBR（Schlick F + GGX D + Smith G + split-sum LUT）
        ↓
全工业默认（UE / Unity / 自研引擎全部收敛到金属度工作流）
        ↓
Substrate（UE 5.2+）：分层 lobe 框架，PBR 的可组合化扩展
```

## 实时化的关键近似（UE 版）

| 物理项 | 精确形式 | 实时近似 |
|---|---|---|
| F | 完整 Fresnel（波长相关） | Schlick 近似：$F_0 + (1-F_0)(1-\cos\theta)^5$ |
| D | Beckmann / 任意 NDF | GGX |
| G | 微面遮挡积分 | Smith 近似（UE 用 height-correlated Smith 变体） |
| 环境镜面 | 预滤波卷积 | split-sum：环境贴图预滤波 + BRDF LUT |

理解这张表 = 理解"实时 PBR 里没有新物理，只有便宜的近似"——这是评估任何"新着色技术"的基准姿势。

## Prerequisites

- [[BRDF]]（数学核心）
- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]（框架源头）

## Related Concepts

- [[Real-Time Rendering]]
- [[Inverse Rendering]]（PBR 参数是逆渲染要反解的目标）

## Game Applications

- UE 默认 Lit Shading Model；
- 材质预算：基础 PBR ≈ 固定开销（LUT 查表），多 lobe（Clear Coat 等）≈ 成倍开销——Shading Model 数量是 DrawCall 之外的隐藏预算维度；
- Substrate 分层材质的性能风险：lobe 组合数爆炸。

## Important Papers

- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]——直接光 / 镜面侧
- [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]]——环境光 / 漫反射侧（IBL 的 SH 半边）

## Personal Knowledge

Current Level: **Normal**（主动研读 D/G/F 物理来源中，2026-09 信号）

## Learning Gap

- D/G/F 物理来源（随 Cook-Torrance 线收口）
- split-sum 预积分的推导直觉（为什么能把 2D 积分拆成两个 1D/2D 查表）

## Next Step

Cook-Torrance 检查表 5 条全过后，PBR 与 BRDF 可同时升 Easy——它们是同一知识体的两个切面。
