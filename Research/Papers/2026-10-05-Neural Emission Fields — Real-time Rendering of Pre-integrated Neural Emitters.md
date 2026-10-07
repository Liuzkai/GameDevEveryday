---
type: paper
title: "Real-time Rendering of Pre-integrated Neural Emitters"
authors: [Arno Coomans, Floor Verhoeven, Edoardo A. Dominici, Markus Steinberger]
year: 2026
published: "2026-10-05（arXiv v1, 2610.06762；Fri 10-2 提交 / Tue 10-6 公告组）"
venue: "arXiv Preprint（cs.GR；稿件模板显示 CGF / Eurographics 2027 投稿痕迹——Volume 46, Issue 2；未正式接收，按预印本处理）"
url: "https://arxiv.org/abs/2610.06762"
code: ""
project_page: ""
category: [real-time-rendering, lighting, emitters, neural-rendering, precomputation]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: "Research（机制全新；单发射器训练 1 分钟–3 小时、15–30 MB/发射器、全高清 1.2–2.5 ms 恒定——工程账已算清，但无引擎集成；作者组此前有 CGF 2024 光场 / CGF 2026 神经辐照度体积两条产品线前科）"
user_level: Normal
status: unread
aliases: [NEF, Neural Emission Fields, Neural Emitters, 预集成神经发射器, 神经发射场, Coomans 2026]
tags: [real-time-rendering, lighting, emitters, neural-rendering, precomputation]
---

# Real-time Rendering of Pre-integrated Neural Emitters（Coomans et al. 2026）

> **入库 2026-10-07（Run 29）。** TU Graz × **Huawei Technologies**（Steinberger 组：此前有 "Real-time neural rendering of dynamic light fields" CGF 2024、"Real-time rendering with a neural irradiance volume" CGF 2026——**"神经预计算"是他们的连续产品线**）。
> **一句话定位**：库内"预计算存法谱系"的**第五种**——把"积分结果本身"学成一个函数；也是"取消式优化"家族的新成员——**直接取消运行时积分**，而不是把它算得更快。

## TL;DR

**复杂光源（带外壳的灯具、几万多边形的发光网格、空间变化发光、可形变发光体）的直接光照，一直是实时渲染的顽疾。** 现有方案三选一：解析法（LTC 类，限制在简单几何、成本随顶点数线性增长）、蒙特卡洛采样（有噪、要降噪器）、代理表示（把"出射辐亮度"预计算成 4D 代理，**但运行时仍要对着代理积分**）。
本文的关键洞察：**"运行时那一层积分"可以整个不存在**——不再只预计算"离开光源的辐亮度"，而是**直接预计算"相对于接收点的、已积分的出射直接光照"**，学成一个神经场（NEF）。因为训练在**光源局部坐标系**里做，训好的 NEF 是**可携带照明资产**（平移/旋转/等比缩放免重训）。全高清单次网络求值 **1.2 ms（漫反射）/ 2.5 ms（光泽）**，且**与光源几何复杂度完全脱钩**（43.6 万顶点的 Dragon 上仍是 2.5 ms——解析法要 1951–3926 ms）。

## Problem

**"非平凡光源"的实时直接光照。** 四类难点（原文列举）：遮挡外壳内的灯丝、数万多边形的发光网格、空间变化（贴图化）发光、可形变发光体。它们共同造成：
- 解析法（LTC 等）只支持**简单多边形**光源，且成本**随光源顶点数线性增长**；
- 纯采样的估计器（含 ReSTIR）支持任意光源，但需要**高采样预算**才能无噪，残余噪声还得靠降噪器（引入时空伪影）；
- 代理表示（Canned lightsources / 辐射度体积 / 神经变体）把光源内部路径代价 factor out 了，**但接收端仍然要在运行时对代理积分**——"the remaining runtime integration is avoidable"是本篇的出发点。

## Previous Work

- **解析**：Lambert 面光源闭式（Baum 1989）、Phong 接收（Arvo 1995）、**LTC**（Heitz 2016——本篇主要对照物）
- **多光源采样**：光源重要度排序 / 层级 / **ReSTIR**（Bitterli 2020）——"残留方差必须靠降噪器"
- **经典预积分**：PRT（SH 投影，但依赖**远场假设**，近场发光体不成立）、**辐照度体积**（Greger 1998；动态后继）——"近场强空间变化信号需要精心布探针 + 细离散"
- **代理光源**：Canned lightsources（4D 光场板）、辐射度体积 + VPL、神经变体（[43]）、小波表示（2025）——**共同局限：运行时仍需对代理积分**；以及"存储随信号维度指数增长"（7 维 → 加粗糙度 8 维 → 每维 32 样本 = **32⁸ ≈ 10¹²** 条目，原文用这个数字讲"诅咒"）

## Core Idea

**把"积分"整个搬进预计算：直接预计算"对接收点的、已积分的直接光照"。**

$$\underbrace{\int_{\mathcal{P}} L_o(\text{光源}) \cdot f(\text{接收点}) \, \cdots}_{\text{代理法：运行时还要积}} \quad\Longrightarrow\quad \underbrace{D_\theta(\tilde{x}, \tilde{n}, \tilde{\omega}_o, \alpha)}_{\text{NEF：一次网络求值}}$$

- 神经场参数化在**接收点坐标**：位置（3D）+ 法线（2D）+ 反射方向（2D）+ **材质（GGX 粗糙度 α）**——覆盖一整个连续材质族（第 8 维）；
- **训练在光源局部系**（$\tilde{x} = M^{-1}x$ 等）：世界空间查询在着色时映射进光源系——**模型因此与场景无关，成为可携带资产**（原文类比："much like a baked texture or precomputed BRDF"）；
- **双头架构**（diffuse / glossy 分离）+ **反射方向编码**——"Encoding the reflected direction is critical for capturing the sharp highlights"；
- 推论：**推理成本与光源几何复杂度解耦**（由网络尺寸决定，而非多边形数）；光源内部互反射、自遮挡、空间变化发光、形变——**都"白送"**（吸收进学习表示，零额外运行时成本）。

## Technical Approach

**架构与参数**（Appendix B）：多头 MLP **4 层 × 宽 128**（ReLU/头）；**多分辨率哈希编码**同时施加于位置与反射方向；存储 **FP16：15.22 MB（漫反射）/ 30.46 MB（GlossyNEF）**。

**训练**（§3.4）：光源置于局部系原点；在 **emitter-space**（= 局部系中轴向包围区，**每轴延伸到光源最大尺度的 20 倍**——远超实际需要的 5 倍）内采样接收点配置，路径追踪出目标值；Adam，lr 1e-3，batch 2¹⁶；**单次训练 1 分钟（简单平面）～3 小时（含内部互反射）**，按光源计一次性成本、跨场景摊销。

**运行时**（§3.5）：单次网络求值 + 漫反射/光泽反照率调制；**全高清 1920×1080：NEF 1.2 ms / GlossyNEF 2.5 ms**——**恒定**。emitter-space 外的点用"钳制到边界 + 解析衰减"启发式（视觉上难分辨）。

**关键对照表**（原文 Table，RTX 4090，全高清）：

| 方法 | 组件 | Quad（4 顶点） | Dragon（436k 顶点） |
|---|---|---|---|
| 解析（Baum 1989） | Diffuse | 0.03 ms | **1951.4 ms** |
| LTC（Heitz 2016） | Glossy | 0.07 ms | **3925.6 ms** |
| **Ours（NEF）** | Diffuse | 1.2 ms | **1.2 ms** |
| **Ours（GlossyNEF）** | Glossy | 2.5 ms | **2.5 ms** |

→ **性能交叉点在 267 个光源顶点**；越过之后 NEF 是唯一还在实时预算里的方案。

**与采样法/代理法的等误差对比**：
- vs **ReSTIR DI**（M=32, K=5）：达到 NEF 的误差水平需 **~256 spp（不降噪）或 ~32 spp（降噪）**——均超实时预算；但 ReSTIR 更通用（光源任意运动；NEF 要求构型训练时固定，支持整体刚体变换与"预编排相对运动"）；
- vs **oracle 代理**（理想代理：查询零成本、无噪）：配学习式重要性采样仍需 **~734 采样/像素**；仅这 734 次 GGX 求值（1080p）就要 **2.9 ms**——**已超过 NEF 的总推理 2.5 ms**；对照网络 [43] 的前向：**103 ms/光源样本/全高清帧**。

**可见性**（§5）：NEF 只含**无遮挡**发射场——配 **~1 条阴影射线/像素 + 降噪器作比率估计**即可得到可信软阴影（"further samples producing diminishing returns"）——**与标准实时可见性管线兼容**。

## Key Contribution

1. **重新表述直接光照为"预集成、接收点感知"的量**——消掉此前所有路线保留的运行时积分；
2. **NEF 表示**：双头 diffuse/glossy + 反射方向编码 → 单次求值的无噪直接光 + 准确高光；
3. **光源局部系训练 → 可携带照明资产**（平移/旋转/等比缩放免重训，"一次训练摊薄到所有场景与实例"）；
4. **覆盖非平凡光源**：内部互反射、空间变化发光、可形变（时间参数化，如 fireball）。

## Limitations

1. **多光源线性**：N 个光源 = N 个 NEF 求和，∝ N（作者提出共享解码 MLP / 光照层级两条未来路线）；
2. **构型固定**：任意独立光源运动需重训（支持整体刚体变换 + 预编排相对运动 + 时间参数）；
3. **仅无遮挡直接光**：软阴影需额外 1 spp 阴影射线 + 比率估计；
4. **训练数据稀疏风险**：重封装光源（窄孔出光）会让路径追踪训练信号变稀疏（建议 light tracing / 自适应采样，均为未来工作）；
5. 容量默认按最难光源（Pierced Sphere）配——简单光源"过配"（可缩，见附录）。

## Game Development Relevance

**4/5。给"发光体照明"开出第三条路，且它正好落在你的两个工作域交界处。**

- **VFX 侧（推断）**：淬炼/技能特效里那些**发光道具、机关、水晶、符文**属于典型"非平凡发射体"——有外壳遮挡、有内部结构、有脉动。NEF 的"训练一次、全场景复用"模式**很像为每类特效光源资产做一张'照明烘焙卡'**（15–30 MB/个）：放哪里、转多少、缩多少都不用重训，**形变与内部互反射白送**。⚠️ 边界：原文演示的是**连续几何的发光体**（含 fireball 式形变动画）；**粒子系统类光源（Niagara 粒子发光）不在其验证范围**——别直接外推。
- **预算维度（与动态灯光台账的关系——个人解释）**：
  - MegaLights 是"**每像素采样预算**"（多、简、动态的灯 → 分摊采样）；
  - **NEF 是"零采样路由"**（单个复杂发射体 → 预集成 + 单次求值）；
  - 两者互补而非替代：**画面里"最贵的那个灯"（结构复杂 + 每帧都在）是 NEF 的理想客户**，其余动态灯走 MegaLights/传统账。
  - "固定样本预算"家族借此补齐终点：1978 每灯 +1× → 1997 每 VPL → 2005 每像素 ~400 → 2019 每帧 m×n → 2026 MegaLights 每像素预算 → **NEF：0 采样（单次求值）**。
- **成本结构可背的几句话**：单次求值 **1.2/2.5 ms 与几何无关**；交叉点 **267 顶点**；存储 **15/30 MB**；训练 **1 分钟–3 小时/光源**。
- **与 [[LightOpt — Lights Optimization for Real-Time Rendering|LightOpt]] 的关系（推断）**：LightOpt 优化"**灯该有几盏**"（外观约束下减数量），NEF 优化"**一个复杂的灯该多贵**"（预集成替代采样）——两条正交的灯光成本削减路线，可同案使用。

## Unreal Engine Relevance

- 概念对应物（**均为工程推断，未核实**）：类似 "Proxy Mesh + Impostor 光照" 的思路被推到极致——UE 侧最接近的现有物是 **Sky Light 的预捕获**（预计算光照入资产）；NEF 是"per-emitter 的预积分光照探针场"。
- 落地形态（推断）：Editor 工具——选中发光体资产 → 训练 NEF → 存为衍生资产（与 Nanite 网格同级别的资产化管线）；运行时在着色器里做一次小型 MLP 推理。
- **不适用面**：移动端前向渲染管线里嵌入 MLP 推理的现实性未验证；原文也未提移动端。

## Technology Evolution

```text
预计算代理线：
  4D 光场板（1998）→ 辐射度体积 + VPL（2000s）→ 神经变体（[43]）→ 小波（2025）
        ↓ 共同残留：运行时仍要对代理积分
★ 2026 NEF（本文）：把"积分"本身预计算掉 —— 预集成 + 神经场 + 可携带资产
        ↓（对照线：LTC 解析法成本 ∝ 顶点数；采样法 ∝ 预算）
```

## Relationships

### Based On

- 多分辨率哈希编码（Müller 2022 — Instant NGP 家族）——网络表示层
- 代理光源谱系（Canned lightsources 1998 / 辐射度体积 / 神经代理 [43]）——问题结构与对照

### Contrasts

- [[LightOpt — Lights Optimization for Real-Time Rendering]]（灯光数量优化 vs 单体预集成）；LTC（Heitz 2016，库外记名）——解析法由本篇在 267 顶点处超越
- **采样法家族**（ReSTIR 2020）——"更通用但需 32–256 spp 等误差"

### Related

- [[Scalability and Quality Tiers]]（动态灯光台账的另一条路）；[[Real-Time Global Illumination]]（光照成本的结构变化）
- [[2026-09-25-DiffusionShadow — Diffusion-based Shadow Caching for Neural Volume Rendering|DiffusionShadow]]——**预计算存法谱系**同族（生成式记忆 ↔ 本篇"把积分结果学成函数"，两者都是"存储问题 → 推理问题"的转化）

### Followed By

- 共享解码 MLP / 光照层级（作者列出的两条线性化路线，未实现）

## Personal Knowledge State

- **user_level: Normal（推断）**。你的灯光/VFX 预算视角可以直接接住本文的"成本账"层；神经网络内部（哈希编码、MLP 细节）按需即可。
- **读法建议（30–45 分钟）**：TL;DR → §1 的核心洞察段（"runtime integration is avoidable"）→ 对照表（267 顶点交叉点）→ §4.3 的 734 样本/oracle 对比（图 7）→ §5 的两条诚实的限制（可见性 / 多光源线性）。

## Learning Value

- **一条可迁移判据**：**"先问'这段计算能不能整个不存在'，再问'能不能算得更快'。"** 本篇是"取消运行时积分"的完整样本——比"优化积分"高一个数量级（436k 顶点上 1952 ms → 1.2 ms）。
- **一条边界纪律**：预集成不是免费的，它把成本换成了**"训练 + 存储 + 构型假设"**（1 分钟–3 小时 / 15–30 MB / 构型固定）——**"省下的运行时成本去哪了"永远要问**（本库既有纪律的又一样本）。

## Visualization

（暂无——机制为"预算换预计算"单一结构，对照表已足够表达；如需可将 267 顶点交叉点与 734 样本对照做成图。）

## Notes

- **来源核对**：arXiv 2610.06762 v1（HTML 全文抽取核对：数字、表格、附录 B/D、§4.1–4.3、§5 全部取自原文 HTML；数值均逐条命中）。
- **窗口状态**：提交于 10-2 前后、公告 10-6（Tue 组）；由 recent 页公告分组 + API 窗口双通道捕获（10-7 运行覆盖）。
- **"预计算存法谱系"第 5 种**（本库归档框架——非原文）：① 全存 → ② 解析表（Karis / Kulla-Conty）→ ③ 复用（Fdez-Agüera）→ ④ 生成式记忆（DiffusionShadow）→ **⑤ 积分结果函数化（本篇）**。
