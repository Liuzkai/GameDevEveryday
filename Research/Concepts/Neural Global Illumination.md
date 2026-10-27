---
type: concept
title: "Neural Global Illumination"
user_level: Hard
tags: [gi, neural, real-time]
---

# Neural Global Illumination

## Definition

用神经网络估计全局光照中的某个分量（通常是间接光或多 bounce 辐射度），替代传统路径追踪中昂贵/嘈杂的采样。

## Core Principle

神经 GI 方法分两大阵营：

| 阵营 | 优点 | 缺点 |
|---|---|---|
| **Screen-space** | 快，天然适配管线 | 视锥外的光进不来 |
| **Scene-wide / data-gathering** | 能补回屏幕外光 | 需要"收集数据"，塞不进标准管线 |

2026 年的主流研究方向是**弥合这两者**——见 [[Lightweight Attention-based Indirect Illumination (AMD)]]，其策略是"把预算从模型复杂度挪到输入丰富度"。

### 第二条分界线：要不要在目标场景上训练（2026-10-27 补）

上面那张表按**输入从哪来**分。还有一条正交的分界，按**权重从哪来**分——它比"屏幕空间 vs 场景级"更能决定成本结构：

| 阵营 | 代表 | 运行前要做什么 | 运行时成本 | 换一个场景 |
|---|---|---|---|---|
| **拟合派**（fit-per-scene） | NRC（2021）、神经辐照度缓存 | 在**目标场景**里跑训练（注入训练样本、更新权重） | 一次小网络查询（1080p ≈ 2.6 ms） | 权重作废，**重新训练** |
| **前馈派**（feed-forward） | [[2026-10-26-Zhang — RADIANT Object-Centric Feed-Forward Global Illumination\|RADIANT]]（2026） | 在**大规模合成数据**上离线训练一次（RADIANT：1.03 M 场景） | 一次完整前向（500 物体 512² ≈ 0.83 s） | 直接可用，**零训练** |

- **拟合派的成本藏在"每次换场景"**，前馈派的成本藏在"每次前向"——**两者都不是免费的，只是账单寄到的地址不同**；
- 前馈派要成立，必须先把"传输序列长度"压下来：RADIANT 的做法是把计价单位从**三角形 T** 换成**物体 N**（每物体 32 token，$O(T^2)\to O(N^2)$），再用 `invariant form-factor attention` 让 4 维几何不变量直接变成 query-key 内积——**不物化 $O(N^2)$ 的 bias 矩阵**；
- 判别一个"神经 GI"到底属于哪派，只要问一句：**"这个网络见过我没见过的场景吗？"** 答案若是"没有，它是为这个场景训的"→ 拟合派。

## Prerequisites

- [[Global Illumination]]（**问题与八代近似谱系**；其中"缓存族（RSM / VPL / Irradiance Caching / DDGI）"即本概念桥的具名缺口——**2026-10-06 起四节点全部落库**）
- **RSM / VPL——✅ 2026-10-03 双源入库**：[[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]]（pixel light + 每像素固定样本）· [[Keller — Instant Radiosity (1997)]]（VPL = 光路顶点变光源）；机制对照表见 [[Global Illumination]] 谱系第 6 行与 [[间接光缓存族_RSM 2005 与 Instant Radiosity 1997 双源图解|双源图解]]
- **接收侧缓存——✅ 2026-10-04 入库**：[[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)]]（辐照度缓存 = **NRC 的直接祖先**：缓存对象就是它定义的辐照度；误差容限 a 决定点密度）；见 [[接收侧缓存_Ward 1988 辐照度缓存图解|第三源图解]]
- **现代形态——✅ 2026-10-06 入库**：[[Majercik — Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields (2019)]]（**DDGI**：缓存族在光追时代的合流点——探针场 + 增量更新 + 滞后摊销；RTXGI 产品线起点，其 SDK v2.0 已并入 NRC/SHaRC）；见 [[探针辐照度场_DDGI 2019 图解|DDGI 图解]]
- [[Neural Rendering]]
- 实时渲染管线结构（G-buffer、deferred）

## Evolution

```
烘焙光照贴图（静态）
        ↓
Screen-space GI / SSGI
        ↓
Light Propagation Volumes / VXGI
        ↓
实时路径追踪 + 去噪（RTX GI）
        ↓
Neural Radiance Caching（NRC，2021，已在 RTX Remix 中产品化）
        ↓
神经 GI（2026：AMD attention GI、ArtiFixer 的单 pass GI 预测）
        ↓
        ★ 镜面支线开线（2026-10-09 入库）：[[2026-10-08-Neural Caching of Prefiltered Radiance for Specular Lighting]]
        —— 缓存对象从"视图无关的辐照度" → "反射方向参数化 + roughness 预滤波的辐射度"（split-sum 重建）
        —— **缓存族（1988 起）第一次正面处理"视图相关"这个前提**

★ 前馈支线开线（2026-10-27 入库）：[[2026-10-26-Zhang — RADIANT Object-Centric Feed-Forward Global Illumination|RADIANT]]
        —— 不是"缓存什么"的问题，而是"根本不要缓存"：网络直接输出整个传输的解
        —— 代价：必须在 1.03 M 合成场景上离线训练；换单位（三角形 → 物体）才把序列长度压进可算范围
        —— 现状：0.83 s/帧 ≈ 比实时的门槛慢 2 个数量级，但比路径追踪快、比逐场景拟合省
```

值得注意：**NRC 已经产品化**（RTX Remix、《半条命 2 RTX》、《传送门 RTX》），说明神经 GI 不是纯未来技术。1080p 下典型开销约 2.6ms。
**NRC 的覆盖是"漫反射半边"**——镜面（视图相关、roughness 相关）此前无人用缓存接住；2026-10-09 入库的镜面变体是这条支线的第一个节点（Ward 1988 的辐照度缓存能成立，恰恰因为"答案是视图无关的"——**这个前提一旦被打破，缓存就得换表示**）。

## Related Concepts

- [[Neural Rendering]]
- [[Global Illumination]]
- [[Real-Time Global Illumination]]

## Game Applications

- Lumen 的替代/增强
- 老游戏重制（RTX Remix 路线）

## Important Papers

- [[Lightweight Attention-based Indirect Illumination (AMD)]]
- [[DLSS 5 — Generative Neural Rendering]]
- [[2026-09-14-Gaussian Light Transport]]（⚠️ 非神经对照样本：显式 13D 高斯基函数 + 残差优化，视角无关、毫秒渲染、低显存——证明"神经"不是实时 GI 的唯一解；2026-09-14 入库）
- [[2026-10-08-Neural Caching of Prefiltered Radiance for Specular Lighting]]（**镜面缓存支线首节点**，2026-10-09 入库）：NRC 变体——反射方向参数化 + roughness 预滤波辐射度 + **split-sum 重建**（复用 BRDF 积分图）；1080p 实时、同性能下收敛更快；限制：继承 split-sum 掠射近似、动态场景未证明（**变体论文——诚实标注"初步"**）
- **前馈分支（2026-10-27 入库）**：[[2026-10-26-Zhang — RADIANT Object-Centric Feed-Forward Global Illumination|RADIANT]]（每物体 32 surface-patch token + `invariant form-factor attention`；32.22 dB per-image / 27.65 dB pooled，领先 RenderFormer-V2 3.3 dB；500 物体 512² = 0.83 s——**研究级，不是实时**；关键贡献是**换单位**：$O(T^2)\to O(N^2)$）
- **经典前置（桥材料，2026-10-03/04 入库，三源齐）**：[[Keller — Instant Radiosity (1997)]] · [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]]——**AMD 论文的 RSM 输入通道就建在后者之上**（"加一路 RSM 渲染让轻量模型看到屏幕外几何"）；[[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)]]——**NRC 缓存的对象（辐照度）即本篇定义**："缓存介质"从插值公式换成神经网络，问题定义未变

## Personal Knowledge

Current Level: **Hard**

## Learning Gap

对你而言性价比最高的不是"神经"部分，而是**经典部分**：RSM / VPL / iVPL 这套近似 GI 的思路。它同时是：
- 理解 Lumen 的前置
- 理解所有神经 GI 论文的前置
- Normal 难度，几个小时可补

**✅ 2026-10-03/04/06 状态更新**：经典部分的**原始文献已四节点入库**（[[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)]] 接收侧 + [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]] 光侧结构化 + [[Keller — Instant Radiosity (1997)]] 光侧随机 + **[[Majercik — Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields (2019)]] 现代形态**；配 [[接收侧缓存_Ward 1988 辐照度缓存图解|第三源图解]]、[[间接光缓存族_RSM 2005 与 Instant Radiosity 1997 双源图解|双源图解]] 与 [[探针辐照度场_DDGI 2019 图解|DDGI 图解]]）——**桥的经典侧自此完整**。按 2026-10-05 PKM 校正：桥材料已备、**实际阅读动作（30 分钟级）待执行**；神经侧（训练/推断）另列 Hard。iVPL（Imperfect VPLs）方向按需再补。

## Next Learning Step

先补 RSM 与 VPL（~~机制表见 [[Global Illumination]] 谱系第 6 行~~ ✅ **2026-10-03 双源入库**：[[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]] / [[Keller — Instant Radiosity (1997)]]），再沿 [[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)]]（接收侧）与 [[Majercik — Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields (2019)]]（现代形态）把"缓存→场→神经"的介质演化串起来，最后回来看神经方法。顺序反了会浪费大量时间。
