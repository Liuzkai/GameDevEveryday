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
4. 移动端基本只烘焙

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
