---
type: paper
title: A Reflectance Model for Computer Graphics
authors:
  - Robert L. Cook
  - Kenneth E. Torrance
year: 1981
published: "1981-07 (SIGGRAPH '81)；期刊版 1982-01 (ACM TOG 1(1): 7-24)"
venue: SIGGRAPH 1981 / ACM Transactions on Graphics
url: https://dl.acm.org/doi/10.1145/357290.357293
code: ""
project_page: ""
category:
  - rendering
  - brdf
  - classical
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal
status:
  - reading
---

# A Reflectance Model for Computer Graphics (Cook & Torrance, 1981)

> 你正在主动研读 D/G/F 三因子的物理来源——这就是源头论文。本笔记按"它继承了什么、改了什么、开启了什么"来写，不重复教科书推导。

## TL;DR

第一次把物理光学中的微面元理论（Torrance-Sparrow, 1967）完整落地为计算机图形学可用的反射模型：高光 = **D**（微面法线分布）× **G**（几何遮挡）× **F**（菲涅尔）三因子乘积，加上能量守恒的漫反射项。所有现代 PBR（GGX、Disney、UE 默认 Shading Model）都是它的直系后代。

## Problem

1981 年之前，CG 的"高光"全是经验拟合：

- Phong（1975）/ Blinn-Phong（1977）：`cosⁿα` 调指数凑形状，**无物理含义**；
- 不区分导体与电介质：金属和非金属用同一套参数，靠美术手调颜色假装是金属；
- 无能量概念：亮度随便超 1，换光照环境就穿帮；
- 无法解释实测现象：掠射角高光增强、金属有色高光、off-specular 峰（高光峰不在镜面反射方向上）。

## Historical Context

```text
辐射度量学建立（Nicodemus 1977 定义 BRDF）
        ↓
Torrance-Sparrow 微面理论（JOSA 1967，物理光学领域，CG 圈外）
        ↓
Blinn 1977 把微面分布引入 CG（但用简化 Fresnel = 常数）
        ↓
★ Cook-Torrance 1981：补齐完整 Fresnel + 光谱相关 + Beckmann 分布
        ↓
Ward 1992（各向异性）/ Schlick 1994（廉价近似）/ Oren-Nayar 1994（漫反射侧）
        ↓
GGX / Trowbridge-Reitz（Walter et al. 2007，长尾分布）
        ↓
Disney Principled BRDF 2012（生产可用参数化）
        ↓
Karis 2013：split-sum 预积分，UE4 实时 PBR → 全工业默认
```

Cook 和 Torrance 当时都在 Cornell Program of Computer Graphics——正是辐射度法和 Cornell Box 的同一批人在推动"渲染要物理正确"这条路线。

## Previous Work

- **Torrance-Sparrow（1967）**：提出微面统计模型解释真实表面的 off-specular 反射，但发表在 JOSA，面向光学工程，形式不可直接用于渲染；
- **Blinn（1977）**：率先把 Torrance-Sparrow 搬进 CG，但 Fresnel 项取常数（意味着所有材质掠射角行为相同——物理上错误）；
- **Phong 系**：纯经验。

Cook-Torrance 的增量 = **把 Blinn 省略掉的物理补回来**：完整 Fresnel 方程 + 波长（光谱）依赖 + 更贴合实测的 Beckmann 分布。

## Core Idea

表面不是理想镜子，而是无数朝向随机的微小镜面（microfacet）。宏观 BRDF 是微面行为的统计积分，恰好可以分解为三个可独立理解的因子：

$$f_r = \frac{F(\theta_i')}{\pi} \cdot \frac{D(\alpha) \cdot G}{(N \cdot L)(N \cdot V)}$$

- **D — 分布项（Distribution）**：有多少比例的微面法线恰好朝向半程向量 H（只有这些微面参与把 L 反射到 V）。原文用 **Beckmann 分布**（高斯型，粗糙度 m 控制宽度）；
- **G — 几何项（Geometry）**：微面之间的互相遮挡——shadowing（光进不来）与 masking（反射出不去）。取三个比值的最小值，掠射角时显著衰减；
- **F — 菲涅尔项（Fresnel）**：单个微面（作为理想镜面）的反射率，由**完整 Fresnel 方程**计算，依赖折射率 n 与消光系数 k——这是**波长相关**的，所以金属高光带色、非金属高光中性，且所有材质在掠射角 F→1。

漫反射项与高光项做能量配平（高光拿走的能量漫反射就拿不到），这是模型自洽的关键。

## Why It Works

三因子各自对应一个真实物理机制，而不是三个调参旋钮：

| 因子 | 物理机制 | 可观测后果 |
|---|---|---|
| D | 微面朝向的统计分布 | 高光形状/大小/锐利度；off-specular 峰 |
| G | 微面间遮挡 | 掠射角高光反而被压暗（粗糙面）|
| F | 界面电磁边界条件 | 掠射角反射率→1；金属有色高光 |

后来所有改进（Beckmann→GGX、Smith G 的精确化、Schlick 近似 F）都只是**在同一框架内换更好的近似**，框架本身 45 年没被推翻。

## Limitations

- 微面是**统计独立**假设：不考虑微面间多次散射（粗糙面能量损失，后来的 multi-scattering 补偿项修这个）；
- 漫反射侧仍是 Lambert，Oren-Nayar 之前无法解释粗糙漫反射的 retro-reflection；
- 完整 Fresnel + Beckmann 含三角/指数运算，1981 年很贵，2013 年之前的实时管线用不起——**实时 PBR 的历史本质上是"给 Cook-Torrance 找廉价近似"的历史**（Schlick F、GGX D、Smith G、split-sum 预积分）。

## Game Development Relevance

- UE 默认 Shading Model（Lit）的镜面项 = **GGX D + Smith G + Schlick F**，即 Cook-Torrance 框架的实时近似版——你材质面板里的 Roughness 就来自 D 项的分布宽度参数；
- 金属度工作流（Metallic = 0/1）是对 F 项"导体 vs 电介质"二值的参数化；
- 对你的预算体系：PBR 材质是**默认着色开销的基线**，理解 D/G/F 就知道 Roughness 为什么几乎免费（查 LUT）、Clear Coat 为什么翻倍（第二个镜面 lobe = 第二套 D·G·F）。

## Unreal Engine Relevance

- `Shading Models / Lit`：UE 官方文档明言基于 Cook-Torrance 微面模型（Karis, SIGGRAPH 2013 "Real Shading in Unreal Engine 4"）；
- Split-sum IBL 预积分（环境 BRDF LUT）是对该模型做环境光卷积的离线化；
- Substrate 的多 lobe 框架是其在分层材质上的扩展。

## Technology Evolution

属于 [[Physically Based Rendering]] 谱系的**奠基节点**。其上承 [[BRDF]] 的形式化定义与 Torrance-Sparrow 理论，下启 GGX → Disney → 实时 PBR 全线。

## Relationships

### Based On

- Torrance & Sparrow, "Theory for Off-Specular Reflection from Roughened Surfaces"（JOSA 1967）
- Blinn, "Models of Light Reflection for Computer Synthesized Pictures"（SIGGRAPH 1977）
- Nicodemus et al. 的 BRDF 形式化（1977）

### Extends

- Blinn 的微面模型：补全 Fresnel 与光谱依赖

### Followed By

- GGX（Walter 2007）、Disney Principled（2012）、Karis UE4 实时化（2013）
- [[LightOpt — Lights Optimization for Real-Time Rendering]] 一类现代工作默认材质侧已是 PBR，优化目标移到了灯光侧

## Personal Knowledge State

`Normal`，状态 `studying`——你正在主动研读三因子的物理来源，这是正向路径。读这篇时建议对照 UE 文档里的 GGX/Schlick 版本，观察"哪些被近似掉了、为什么敢近似"。

## Mastery Criteria（Normal → Easy 检查表）

- [ ] 能用一句话说清 D/G/F 各自的物理机制与可观测后果
- [ ] 能解释为什么金属高光有色、非金属高光中性（F 的波长依赖）
- [ ] 能解释掠射角"所有材质都变镜子"（F→1）
- [ ] 能说清 UE 的 Roughness / Metallic 分别对应三因子中的哪一部分
- [ ] 能说清 GGX 相对 Beckmann 改了什么（长尾巴 → 更真实的高光晕散）

## Notes

- 期刊版（TOG 1982）与会议版（SIGGRAPH 1981）内容一致，引用任其一即可；
- 原文的 D 用的是 Beckmann 而非后来流行的 GGX——读现代实时 PBR 资料时注意这个替换。
