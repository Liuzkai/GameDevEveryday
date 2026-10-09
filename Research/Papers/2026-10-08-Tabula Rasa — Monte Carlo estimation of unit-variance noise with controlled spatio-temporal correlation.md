---
type: paper
title: "Tabula Rasa: Monte Carlo estimation of unit-variance noise with controlled spatio-temporal correlation"
authors: [Tobias Ritschel, Yang Zhou, Nick Milef, Mikhail Dereviannykh, Chen Liu, Christophe Hery, Carl Marshall]
year: 2026
published: "2026-10-08（arXiv v1, 2610.11653；**API 通道'未公告先见'捕获**；SIGGRAPH Asia 2026 Conference Papers）"
venue: "SIGGRAPH Asia 2026 Conference Papers（University College London × Meta Reality Labs；KIT）"
url: "https://arxiv.org/abs/2610.11653"
code: "https://github.com/facebookresearch/Tabula-Rasa"
project_page: ""
category: [noise, monte-carlo, generative-models, video-diffusion, npr, texture-synthesis]
importance: B+
historical_importance: 0
game_relevance: 3
production_readiness: "Research（开源代码；1024² 帧 2.44 ms @A100；'给现成路径追踪器加三行代码'的实现成本；方法有偏（原文自认））"
user_level: Normal（结论层）
status: unread
aliases: [Tabula Rasa, unit-variance noise, 时空相关噪声, 光传输噪声]
tags: [noise, monte-carlo, generative-models, video-diffusion, npr]
---

# Tabula Rasa: Monte Carlo estimation of unit-variance noise with controlled spatio-temporal correlation（Ritschel et al. 2026）

> **入库 2026-10-09（Run 31）。** **UCL × Meta Reality Labs**（Ritschel 组；**Christophe Hery / Carl Marshall**——前 Pixar 渲染班底）。SIGGRAPH Asia 2026 Conference Papers；[代码开源](https://github.com/facebookresearch/Tabula-Rasa)。
> **一句话定位**：**让"噪声"遵守光路几何。** 生成一段**单位方差 + 时空相关**的高斯噪声序列——"同一个世界点在不同像素/帧里，拿到的必须是同一个噪声值"。**单位方差是给扩散模型的接口契约，时空相关由光传输诱导**。1024² 帧 2.44 ms（A100），给现成路径追踪器加三行代码。
> **库内位置**：**"渲染器 ⇄ 生成模型"的统计接口层**第一个节点——把"仿真诱导的统计（MC）"与"生成模型的数学要求（扩散）"正面接起来。

## TL;DR

**一个被两拨人分别忽视的接口规范。**

- 扩散视频模型（Video Diffusion）**需要**时间相关的噪声：帧间噪声若独立，模型学到的"运动"会被噪声闪烁污染；
- 但噪声**同时必须**是单位方差（unit variance）：方差偏离 1 会直接劣化生成质量（这是扩散模型训练/推理的数学前提）；
- 传统"时间相关的噪声"方案（对噪声场做形变/光流 warp、布朗桥、栅格化枚举）**在保持相关性时破坏了方差**（或反之、或慢）；
- **本篇**：把噪声**定义在光传输采样结构上**——噪声值由"世界空间样本怎么被采样"决定：共享同一个世界点的两次采样，共享同一个随机变量 → **相关性天然由几何诱导**（遮挡后重现、跨视角、跨帧自动一致）；
- **方差的救法**：噪声 = "像素重建估值的随机场"——用 MC 同时估计"重建"与"方差"（**"sketching"** 概念，来自数据库文献）；最终实现动作简单到：**"数一数访问了多少个随机变量，然后做一个平方根和一个除法"**（原文原句大意）；
- **诚实标注**：方法**有偏**（估计量的非线性变换 + 有限直方图）。

## Problem

"生成模型要的噪声"和"渲染器能产的噪声"是两种东西：

| 需求方 | 要什么 | 现状 |
|---|---|---|
| 视频扩散/生成流水线 | 时间相关（同世界点跨帧同值）+ 单位方差 | 现有方案要么破坏方差、要么慢、要么只支持特定条件 |
| 渲染器（路径追踪） | 噪声按采样结构自然产生 | 天然有"光传输诱导的相关性"，但**没人把它规范化成"单位方差 + 可控相关"的接口** |

**关键句（原文）**："the challenge is to combine the aims of correlation and variance control."

## Historical Context

```text
1990s+   MC 渲染的噪声："敌人"——降噪 / 滤波 / 低差异序列（目标：去相关、降方差）
        ↓
2018+    扩散模型崛起 → 噪声从"敌人"变成"载体"（图像/视频生成的原料）
        ↓
2024    视频侧尝试：噪声的时间相关要用特殊构造（枚举/布朗桥）——注意力在"生成端"
        ↓
★ 2026  本篇：把问题搬回"渲染端"——
        "渲染器本来就在按光路生成相关性噪声；
         缺的只是把它规范成生成模型能消费的接口（单位方差 + 可控相关）"
```

**方向反转**：过去 30 年渲染做的是"去掉噪声的相关性"（去噪）；本篇做的是"**制造**特定相关性的噪声"——**同一套 MC 工具，服务生成模型而非对抗它**。

## Previous Work

- **Chang et al. 2024 / Deng et al. 2025**：视频噪声相关的先行方案（组合枚举 / 布朗桥）——本篇的对照；
- **"Sketching"（数据库文献，Flajolet & Martin 1985 / Whang et al. 1990 线）**：从样本中估计"和/方差"的统计工具——本篇的核心借用（**"数据库的统计工具进了渲染器"**）；
- **Nimier-David et al. 2022 / Hery 2018**：估计量非线性变换的偏差来源（本篇自认偏差的来源之一）；
- **低差异/蓝噪声（Huang 2024 / Zhou 2025）**：空间侧的目标（本篇 target 单位方差 + 时间相关；空间侧留作扩展方向）。

## Core Idea

**"噪声 = 光传输的随机数账本。"**

```
① 渲染器本来就"按世界空间采样"：一条路径在 某世界点 用掉的随机变量，
   和另一条路径再次经过该点时用掉的随机变量，是同一个（或可重放同一段）。
② 于是： "哪些输出像素共享同一个世界点" → 由几何决定；
        "共享的像素拿到相同噪声" → 由随机数结构决定；
        → 时空相关性 = 光传输的副产品（无需光流、无需 warp）。
③ "单位方差" = 在"共享/复用随机变量"的过程中，记录"每个输出实际访问了多少个随机变量"，
    用统计学（sketching）同时估计方差 → 归一化回 1。
```

**一句话**：**用"数账"（访问了多少随机变量）把"复用随机数"带来的方差偏差精确地修回来**——复用产生相关性，数账保持方差。

## Technical Approach（要点）

1. **子像素分解 + 方差估计**（§3.2）：把像素值写成若干随机变量之和 → 联合估计"重建 + 方差"（sketching）；
2. **Lattice Hashing Noise + 预滤波**（§3.2/3.3）：噪声生成与查询的实现层（原文含"为什么需要预滤波"的问题-解法）；
3. **实现形态**：
   - 路径追踪器：**"三行代码"**（跟踪随机变量访问数 + 缩放）；
   - **栅格化器也可实现**（补充材料含伪代码）；
   - **时间并行**（frame-parallel）；
4. **性能**：1024² 帧 **2.44 ms（A100）**；原文："state of the art speed while being parallel over time"。

## Key Contribution

1. **接口的发现与规范化**：把"生成模型要的噪声"重新表述为"渲染器采样结构的一个统计规范化问题"——并给出可实施形式（单位方差 + 几何诱导相关）；
2. **"Sketching"跨域借用**（数据库 → 渲染）：渲染器中"同时估计值与其方差"的工具箱；
3. **极低的实施成本**（三行 + 并行 + 两算法都能上）——**"接口型"论文的典型价值形态**；
4. **坦白有偏**，并给出偏差来源清单。

## Why It Works

- **相关性来自结构而非后处理**：不用光流估计、不用 warp 修正——跨帧一致性"免费"（只要几何可见性一致）；
- **方差可修复**：共享随机变量造成的方差（低于 1）可以通过"访问计数"精确记账并还原——**统计上"知道欠了多少"**；
- **服务端与消费端解耦**：渲染器只管"产生正确统计量的噪声"，生成模型只管"消费"——中间用"单位方差"这个**契约**对齐。

## Limitations

- **有偏**（原文自认：非线性变换 + 有限直方图；"developing an unbiased estimator might be a rewarding undertaking"）；
- 现目标为**时间相关 + 单位方差**；空间侧（蓝噪声/黄金噪声）留作未来工作；
- 生成质量下游验证限于其演示任务（超分/风格化/纹理合成等）；**"为扩散模型服务"的端到端生产验证尚未完成**；
- 相关性的"可控性"以光传输结构为边界（要"违反几何的相关"就超出方法范围）。

## Game Development Relevance

- **风格化渲染 / NPR**：演示场景之一（stylized shading design）——**"噪声的艺术指导"**（让噪声按几何/纹理语义走，而非逐帧闪烁）在风格化管线里是每日问题；
- **动画纹理合成**（§5.3）：程序化纹理的"时间上不闪烁但空间上还是白噪声"曾是手工艺；本篇给了原理化做法；
- **视频生成管线的前置条件**：如果未来游戏管线里出现"渲染 → 扩散精修/插帧"的环节（[[Neural Upscaling and Frame Generation|神经超分/插帧]] 的延伸），**喂给它的噪声规格**就是一个新的管线参数——本篇是它的定义者之一；
- ⚠️ 判断：目前**不是**立即可搬的工具（目标场景是研究/风格化实验），价值在**接口认识**：当你的管线里出现"生成模型"时，问一句"**它要的随机数是什么规格？谁来生产？**"。

## Unreal Engine Relevance

- 无直接映射。间接：**后处理链中的噪声资产**（胶片颗粒、风格化抖动、Neural 系后处理）——若未来接扩散类后处理，噪声的时空规格会成为接口问题；本篇可作为概念参照。

## Technology Evolution

```text
【噪声的角色演化】
  敌人（要去掉）—— 1990s-2010s：降噪 / 去噪 / 低差异序列
        ↓
  载体（要制造）—— 2020s：扩散模型的原料；"什么噪声"决定生成质量
        ↓
★ 2026 本篇：把"噪声规格"变成渲染器与生成模型之间的接口契约
  （单位方差 = 数学要求；时空相关 = 由光传输诱导）

【"渲染器作为统计引擎"】
  路径追踪器此前被当作"图像生成器"；本篇把它当作"符合几何的随机场生成器"——
  输出物从"颜色"扩展到"统计结构"。
```

## Relationships

### Based On

- **MC 估计 + 路径追踪**（渲染侧标准工具）；
- **Sketching**（数据库统计文献：Flajolet & Martin 1985 / Whang 1990 线）——跨域借用；
- **Lattice hashing noise**（实现层）。

### Related

- **扩散 / 视频生成（[[DLSS 5 — Generative Neural Rendering]] / [[Generative Rendering]] 系）** —— 消费端：噪声规格的"客户"；
- **[[Temporal Stability and Artistic Intent]]** —— **同一枚硬币**：那边研究"渲染的时间稳定性"（消除闪烁），这边制造"受控的时间相关"（保留结构）——**"时间上的随机性"从"缺陷"变成"可设计量"**；
- **[[Neural Upscaling and Frame Generation]]** —— 管线位置相邻（后处理链）；噪声规格是该类组件的输入契约的一部分；
- [[2026-09-25-DiffusionShadow — Diffusion-based Shadow Caching for Neural Volume Rendering|DiffusionShadow]] —— 对照：那边用扩散模型**当缓存**（消费渲染数据），这边给扩散模型**供数据**——生成与渲染的双向接口化。

## Personal Knowledge State

- **user_level: Normal（结论层）**。前置：MC 采样概念（[[Global Illumination]] 内）+ 时域噪声经验（你做 VFX/渲染的日常）。**读法（≈10 分钟）**：Abstract → §1 两个例子（同世界点跨像素/跨帧同值）→ §3.4 Summary → §6 坦白段。

## Learning Value

1. **"接口型创新"样本**：不发明新算法，**发现两组人之间缺失的规格说明**（单位方差 + 相关结构），并用已有工具（MC + sketching）实现——**"两拨人各干一半的活，中间那张纸没人写"**；
2. **"方差与相关不可兼得"的工程案例**：任何"复用随机数/重采样/权重复用"的系统（含 TAA、时空去噪）都在这个 trade 里——本篇给出了"记账修复"的范式（**用显式计数弥补复用导致的统计偏差**——与 [[2026-10-04-SteadySplats — Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering|SteadySplats]] 的"方差账"同族）；
3. **工具跨域**（数据库 sketching → 渲染）：值得放进"工具箱迁移"清单。

## Visualization

（本节点暂不新增图解——"同世界点同噪声"的核心图示已在 Abstract 与 §1 一图说清。）

## Notes

- **双通道战果**：本篇与 [[2026-10-08-Neural Caching of Prefiltered Radiance for Specular Lighting|NRC-Spec]] 均为 10-08 API 侧先见；
- **作者阵容留痕**：Hery（前 Pixar，渲染/着色方向）× Marshall（前 Pixar）× Ritschel（UCL）——**影视渲染班底进入"渲染 ⇄ 生成"接口问题**是一个值得注意的动向；
- **代码可试**：facebookresearch/Tabula-Rasa（与三行代码的说法一致，适合列入可试清单，若要验证"给现成渲染器加噪声规格"）。
