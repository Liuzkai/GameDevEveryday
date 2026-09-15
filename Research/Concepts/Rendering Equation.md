---
type: concept
user_level: Normal
aliases: [渲染方程, Light Transport Equation]
prerequisites: ["[[BRDF]]", "Radiometry"]
first_introduced: "Kajiya 1986"
---

# Rendering Equation

## Definition

描述稳态光传输的积分方程：表面点 $x$ 朝方向 $\omega_o$ 的出射辐射亮度 = 自发光 + 半球面上所有入射光经 [[BRDF]] 加权、余弦衰减后的积分。

$$L_o(x, \omega_o) = L_e(x, \omega_o) + \int_{\Omega} f_r(x, \omega_i, \omega_o)\, L_i(x, \omega_i)\, (n \cdot \omega_i)\, d\omega_i$$

## Core Principle

- **递归性是本质**：$L_i$ 本身由同一个方程在其他表面点上定义 → 光传输天然是全局的、无穷次弹射的，这是 GI 一切困难的根源；
- **容器与内容**：方程是通用容器，材质差异全部收进 $f_r$；[[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]（D·G·F）是 $f_r$ 的一个具体实现；
- **无解于解析，解于采样**：第二类 Fredholm 方程，Neumann 级数 = "按弹射次数展开"，Monte Carlo path tracing 是其通用数值解，收敛律 $1/\sqrt{N}$。

## Prerequisites

- [[BRDF]]（$f_r$ 的定义与性质）
- Radiometry（辐射亮度/辐照度等物理量）
- 基础概率与积分（Monte Carlo 估计）

## Historical Evolution

```text
辐射度量学（物理传统）
  ↓
Whitted 递归光线追踪（1980，镜面特解）
Radiosity（Goral et al. 1984，漫反射互反射特解）
  ↓
★ Kajiya 统一表述（1986）→ [[Kajiya — The Rendering Equation (1986)]]
  ↓
双向 PT / MLT / Photon Mapping（1993-1997，求解器多样化）
  ↓
离线渲染 PT 革命（2013-2015，Arnold/RenderMan/Hyperion）
  ↓
实时近似栈：RTX（2018）→ Lumen / 降噪 / SH Probe
  ↓
前沿：神经 GI、显式基函数（GLT）、可微渲染——都是该方程的新近似族
```

## Important Papers

- [[Kajiya — The Rendering Equation (1986)]]（提出 + path tracing 发明）
- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]（$f_r$ 侧奠基）
- [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]]（漫反射特例的 SH 压缩解）
- [[2026-09-14-Gaussian Light Transport]]（2026 前沿：残差优化直接拟合解）

## Related Concepts

- [[Global Illumination]]（方程的"非直接光"部分）
- [[Physically Based Rendering]]（方程 + 能量守恒 $f_r$ 的工程化）
- [[Real-Time Rendering]]（方程在帧预算下的妥协总集）

## Technologies

- 烘焙（Lightmass / Lightmap）= 离线解缓存
- SH Irradiance / Light Probe = 漫反射压缩解
- Lumen / 硬件 RT = 实时近似
- Path Tracer 参照模式 = 离线真值

## Personal Knowledge

`Normal`。容器侧结构已建，结合 BRDF（内容侧）即闭合 PBR 主链。

## Learning Gap

- Monte Carlo 估计与方差/收敛律的定量直觉（$1/\sqrt{N}$ 如何换算成采样预算）；
- 体传输方程（参与介质推广）——[[Volumetric Rendering]] 方向待建。

## Next Step

- 完成 [[Kajiya — The Rendering Equation (1986)]] 笔记中的 5 条 Mastery 自测；
- 用"每个引擎特性在妥协方程的哪一项"的框架重读 Lumen 官方文档。
