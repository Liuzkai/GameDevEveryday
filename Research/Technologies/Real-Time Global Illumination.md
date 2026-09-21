---
type: technology
title: "Real-Time Global Illumination"
user_level: Normal
tags: [rendering, gi, production]
---

# Real-Time Global Illumination

## Overview

在交互帧率内求解间接光照。游戏工业的两大成熟路线 + 一条正在逼近的新路线。

## Architecture

**路线 A — 烘焙 + 动态近似（主流）**
```
静态：Lightmap / Irradiance Volume（烘焙）
动态：SSGI / DFAO / 屏幕空间探针
混合：Lumen（SDF + 表面缓存 + 屏幕空间 + 世界空间）
```

**路线 B — 实时路径追踪 + 去噪**
```
少量 spp 路径追踪 → 时空降噪器 → 输出
```

**路线 C — 神经 GI（正在逼近）**
```
G-buffer + 辅助 buffer → 神经网络 → 间接光分量
代表：Neural Radiance Caching（已产品化）、[[Lightweight Attention-based Indirect Illumination (AMD)]]
```

## Performance

| 方案 | 典型开销 | 状态 |
|---|---|---|
| Lumen | 数 ms（取决于设置） | Production Ready |
| RTGI + 降噪 | 依赖 RT 硬件 | Production Ready（高端 PC） |
| Neural Radiance Caching | ~2.6ms @1080p | Production Ready（RTX Remix） |
| AMD Attention GI | 45.56ms @512×512（未优化 FP32） | Research，**差两个数量级** |

## Memory

- Lumen 表面缓存：显著显存占用
- NRC：网络权重 + 世界空间网格
- 烘焙：磁盘 + 内存，但运行时最省

## Hardware

- 路线 A：全平台
- 路线 B：需要硬件 RT
- 路线 C（NRC）：需要 Tensor Core

## Production Challenges

1. 动态场景的一致性（物件移动后 GI 是否正确跟上）
2. 漏光与漏阴影
3. **与 VFX 的交互**：特效是否参与 GI？通常不参与（太贵），但这会造成特效与场景光照脱节
4. 移动端基本只烘焙（**2026-09-16 需修正**：见下）

## 移动端动态光照的第一个参考点（2026-09-16）

Arm × Sumo Digital 的《Neural Dawn》（UE 5.6.1）是**全球首款在移动端启用 MegaLights 的游戏**，把"移动端只能烘焙"这个长期前提撕开一个口子。

但要按**预算置换**而不是"性能提升"来理解：

- 它不是手机算力够了，而是**用神经超分（NSS 最多 -50% GPU workload / NFRU 最多 2× 帧率）省下的功耗去换动态灯光 + 光追**；
- **代价**：独占新一代 Arm Mali GPU + 深度管线定制。Arm 官方明确说 NSSD + MegaLights 的组合**在 UE 里不是 plug-and-play**；
- 结论：移动端动态光照**可用但非普适**。对分档预算的意义是"低端档也许可以用特性置换换取特性"，而不是"全面放开动态灯"。

关联：[[Arm Neural Graphics]]（NSS/NFRU/NSSD 与 Neural Dawn 完整档案）

## 🔴 补：动态光照的成本的地基与一条覆盖范围硬约束（2026-09-21）

### 地基：每盏动态灯的代价在 1978 年就被写下来了

[[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] 给出了今天所有"动态灯光预算"的成本法则：

> *"The cost of determining the shadows associated with **each light source** is **roughly twice the cost of rendering the scene without shadows**, plus a fixed transformation overhead which depends on the image resolution."*

**含义（对 GI 与灯光预算的分工很关键）**：

- **动态灯光的成本单位是"一遍完整场景渲染"**（光源视角无需着色，故为"约"2×）→ 这就是为什么动态灯光上限只能是 0–3；
- **降阴影分辨率解决不了灯数问题** —— 分辨率相关项只占图像空间那部分，省不掉那 1× 场景渲染；
- **这也正是 MegaLights 存在的理由**：把"每盏灯一遍"改成"大量灯共享结构化采样"。
  → **复审 MegaLights 时该问的不是"快不快"，而是"它的成本函数现在关于哪个变量线性"。** 因为 1978 的成本结构没有消失，只是换了变量名。
- **六条 1978 年列出的限制至今全部活着**（视锥内才能投影 → CSM；全向光需分扇区 → cube map；内存乘数；量化与走样；横向分辨率平方级；大透视加剧量化）—— **CSM / PCF / VSM / MegaLights 都在绕开其中之一**。详见 [[Shadow Mapping]]

### 🔴 硬约束（二手，待官方核实）：MegaLights 不覆盖半透明与特效

多个二手来源（含百科条目）记 UE 5.8 MegaLights 的限制为：**不支持半透明物体、流体、云、发丝；不支持前向渲染**。

> **⚠️ 来源分级**：全部为二手。方向与社区已知限制一致，**具体数字（"约 70 盏灯以上才显著收益""RTX 4080 上提升可达 50%"）未经官方文档核对**。
>
> **【对本技术笔记的直接影响】**：**本页 Production Challenges 第 3 条（"特效是否参与 GI？通常不参与"）现在多了一条同源的解释** —— 不只是"太贵"，而是 **MegaLights 这条新的动态光照路径在结构上不覆盖半透明物体**。
> **结论：MegaLights 让动态灯光变便宜，但推不到 VFX 侧。** 特效打光仍归 1978 那条账管（每盏灯 +1× 场景，或至少 +1 遍受光物体）。**这对 [[Real-Time VFX Performance Budgeting]] 的动态灯光维度是一条独立的、必须单列的约束。**

关联：[[Shadow Mapping]]（成本模型与六条限制的完整展开）· [[预算五维_1978-1983_源头图解]]

## Game Engine Integration

- UE：Lumen / SSGI / Lightmass
- 外接：NVIDIA RTX GI、RTX Remix（NRC）

## Unreal Engine Possibilities

- **MegaLights 已在 UE 5.8 转 Production-Ready（2026-09，State of Unreal 2026 官方确认）**——路径 Experimental(5.5)→Beta(5.7)→Production(5.8)；官方定位本世代主机 60fps + 大量带阴影动态光。⚠️ 已触发 SABC 动态灯光维度复审条件（见 [[2026-09-14]]）；Gears E-Day（10-6）为首个 3A 实战样本
- **Lumen Lite（UE 5.8 新增）**：2× Lumen 速度，Switch 2 60fps / 低端 PC——低端画质档 GI 新选项，与 Android 三档的策略同源
- NRC 若通过 RTX Remix 路径进入，会改变老项目重制的 GI 策略
- 长线观察：[[2026-09-14-Gaussian Light Transport]] 类"显式基函数烘焙"若成熟，可能抬高静态场景 GI 质量上限（含移动端）

## Related Concepts

- [[Neural Global Illumination]]
- [[Global Illumination]]
- [[Reflective Shadow Maps]]

## Related Papers

- [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] —— **"动态灯光"成本法则的地基**（每盏灯 ≈ +1× 场景渲染）
- [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]]（烘焙路线 A 的数学根源：SH9 探针）
- [[Lightweight Attention-based Indirect Illumination (AMD)]]
- [[LightOpt — Lights Optimization for Real-Time Rendering]]
- [[DLSS 5 — Generative Neural Rendering]]

## Personal Knowledge

Current Level: **Normal**

## Learning Gap

1. Lumen 的内部结构（SDF trace、表面缓存、屏幕空间回退）
2. 为什么特效通常不参与 GI，以及这带来的观感问题
3. NRC 的"预测辐射度比路径追踪更干净"这个反直觉事实的原理

## Next Step

结合 [[LightOpt — Lights Optimization for Real-Time Rendering]]：先厘清"直接光/动态光预算"与"GI 预算"是**两个独立的预算池**，你的 SABC 模板目前只覆盖了前者。
