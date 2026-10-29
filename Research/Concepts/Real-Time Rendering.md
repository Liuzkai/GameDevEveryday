---
type: concept
user_level: Easy
aliases:
  - Real-Time Rendering
  - 实时渲染
  - RTR
prerequisites: []
first_introduced: 1960s–1970s（光栅显示与可见面算法）;实时作为工程目标从飞行模拟器时代确立
---

# Real-Time Rendering

## Definition

**在硬性帧预算内（16.7ms@60fps / 8.3ms@120fps / 移动端更紧）持续生成图像的工程学科与技术族。** 它不是某个算法，而是"**一批算法在时间轴上的共存方式**"——每一代实时渲染技术的衡量标准都不是"画得最对"，而是"**在预算内画得足够好**"。

> ⚠️ 本库对本概念的读法：**你的专业域（Easy），机制层不复述**——本笔记是**地图与索引**：这个域里什么在库里、什么在活跃、你当前的收口点在哪里。

## Core Principle

**实时渲染的永恒矛盾：想要的对光与材质的模拟 → 永远超过硬件能力 → 所有技术的价值 = 它把"超过"改写成了什么。**

本库反复使用的同一问法（从 [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)]] 到 Mega Geometry）：

> **成本 ∝ 物体空间复杂度是"原罪"；每一次优化都是在把成本改写成"成本 ∝ 可见复杂度"**（或 ∝ 屏幕占比 / ∝ 输出分辨率 / ∝ 帧率目标）。

## 领域地图（含库内状态与你当前的层级）

```text
管线骨架
  ├─ 光栅化 / 延迟渲染 / 前向渲染        → [[Tile-Based Rendering]](N) · [[GPU-Driven Rendering]](N)
  ├─ 可见性与剔除                        → Clark 1976（LOD/层级剔除）· Nanite
  └─ 分档与预算 ★你的象限                 → [[Scalability and Quality Tiers]](E) · [[Real-Time VFX Performance Budgeting]](E)

着色
  ├─ 微面 BRDF（D·G·F）                  → [[Microfacet Theory]](N) · [[BRDF]](N) · [[Physically Based Rendering]](N)
  ├─ 能量账本（多次散射补偿）              → [[Multiple Scattering and Energy Compensation]](N)
  └─ 特殊材质                            → [[Hair Rendering]](N) · [[Participating Media]](N)

光照
  ├─ 直接光 + 阴影                        → [[Shadow Mapping]](E) · [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]
  ├─ 全局光照（间接光）                   → [[Global Illumination]](N) → [[Real-Time Global Illumination]](T) · [[Neural Global Illumination]](H)
  └─ 动态灯光现代化                       → MegaLights（UE 5.8 Production，PC only）· [[LightOpt — Lights Optimization for Real-Time Rendering]]

重建与后处理
  ├─ 时域稳定性                          → [[Temporal Stability and Artistic Intent]](N)
  └─ 神经重建/生成                       → [[Neural Upscaling and Frame Generation]](T) · [[Generative Rendering]](H)

神经渲染（Hard 区）
  → [[Neural Rendering]](H) · [[Gaussian Splatting]](E) · [[Differentiable Rendering]](H，9-30 开线)
```

## Historical Evolution

```text
1960s-70s  光栅显示 / 可见面算法（[[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)]]）
1970s-80s  着色模型（Gouraud / Phong）→ 物理模型（[[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]）
1983       粒子与程序化特效进管线（[[Reeves — Particle Systems (1983)]]）
1986       渲染方程的完整表述（[[Kajiya — The Rendering Equation (1986)]]）——此后一切光照的容器
1987       **渲染架构第一次被整体设计**：[[Reyes — The Reyes Image Rendering Architecture (1987)]]——
           八阶段管线 / 微多边形约半像素边长（Nyquist）/ ε 平面 / 16 子像素 jitter 统一抗锯齿·运动模糊·景深
           **代价：先着色后判可见 ⇒ 结构性 overdraw**（这条权衡 30 年后被 visibility buffer 反向解掉）
1990s      纹理映射普及 / 多分辨率（细分、LOD 工程化）
1998-2002  **点/面元这条支线自成一路**：QSplat（层次点）→ [[Zwicker — Surface Splatting (2001)]]（屏幕空间 EWA，带限进核）
           —— 它不实时（42.9 万点 @512² 仅 0.7 fps），但把"splat 的核"这件事一次做对了；20 年后由 3DGS 兑现
2000s      可编程着色器 → 延迟渲染；PBR 标准化前夜
2013       PBR 进引擎（[[Karis — Real Shading in Unreal Engine 4 (2013)]]）——实时渲染与物理渲染合流
2010s-20s  虚拟几何（Nanite）/ 光追普及（DDGI、Lumen）/ 虚拟阴影贴图
2020s-26   神经渲染分三线进入：重建（DLSS）→ 生成（DLSS 5）→ 神经几何/光照；
           档位体系开始分裂为"性能阶梯"与"管线切换/硬件门槛"（[[2026-09-Monthly|本月月报]] 记录）
```

## Important Papers

- 本库"预算起源三件套"：[[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)]] · [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] · [[Reeves — Particle Systems (1983)]]
- PBR 四源头：[[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] · [[Kajiya — The Rendering Equation (1986)]] · [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] · [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]]
- 工程化：[[Karis — Real Shading in Unreal Engine 4 (2013)]]
- **架构层第一篇**：[[Reyes — The Reyes Image Rendering Architecture (1987)]] —— 不是某个效果的论文，而是**整条管线怎么排**的论文（八阶段 / 自然坐标 / 线性 / 局部性 / 无上限模型）；**今天所有"先可见后着色"的设计都是在给它做反面**（2026-10-27 入库，二手重建）
- **splat 的数学源头**：[[Zwicker — Surface Splatting (2001)]] —— 屏幕空间 EWA 的闭式核 $\rho_k(x)=\frac{1}{|J|}G_{JV_kJ^\top+I}(x-m(u_k))$；**那个 $+I$ 就是"像素有面积"这条事实，抗锯齿因此是核的性质而不是后处理**。2023 年 3DGS 的 $\Sigma'=JW\Sigma W^\top J^\top$ 与它是同一个式子（低通退化为 $+0.3I$ 的膨胀）；2026 年 [[2026-10-27-Kaushik — Hexels Compact Hexagonal Primitives for Scene Rendering|Hexels]] 把支撑域从"推导出的椭圆"换成"设计出的六边形"（2026-10-29 入库）
- 完整清单见 [[Index]]

## Related Concepts

- [[Global Illumination]] —— 本域"最贵的一半"（间接光）；实时化路线见其条目
- [[Scalability and Quality Tiers]] —— 实时渲染的"社会契约"：**同一份内容在不同预算下都得成立**
- [[Linear Transport Theory]] —— 所有光照的数学母语（表面 ↔ 体积）
- [[Budget Units]] —— **本域的每一个时代都在换计价单位**（三角形 → 像素 → 微多边形 → 探针 → token）

## Technologies

- [[Real-Time Global Illumination]] · [[Neural Upscaling and Frame Generation]] · [[Arm Neural Graphics]]
- （引擎出口）Lumen / MegaLights / Nanite / TSR / Niagara

## Game Applications

- 一切运行时的画面；本库应用层入口：[[AAA Real-Time VFX]] · [[Open World Character Animation]]

## Personal Knowledge

- **Easy（专业域）**——机制层不复述。**活跃收口点**（2026-09-30 状态）：
  1. [[Physically Based Rendering]] / [[BRDF]]：25 条自测 + furnace test（两项检查）→ 标 Easy；
  2. [[Hair Rendering]]：15 条自测 + 巫 3 实战观察 → 标 Easy；
  3. 动态灯光维度复审（MegaLights 语义重述）——执行项，E-Day 10-6 为最终检验。

## Learning Gap

- 基础机制层：**无**（专业域）；
- 唯一持续缺口在**神经渲染侧**：[[Neural Rendering]] / [[Differentiable Rendering]] / [[Generative Rendering]]（三个 Hard 目标，各有桥）。

## Next Step

- 本概念作为**总目录**使用：需要哪个子域直接跳转；**新的 Frontier 笔记入库时，检查是否需要挂到本表**（保持地图与库同步）。
- 本笔记建立于 2026-09-30（月结日库内质量修复：此前本链接悬空 23 天）。

## Notes

- **为什么现在才建**：首日建库时本概念被列为"基础锚点"但文件一直缺失（Index / PKM / Learning Paths 多处引用悬空）——2026-09-30 链接完整性检查时发现并补齐。同期补齐的还有 [[Global Illumination]]。
