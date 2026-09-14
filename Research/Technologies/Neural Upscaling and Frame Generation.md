---
type: technology
title: "Neural Upscaling and Frame Generation"
user_level: Normal
tags: [rendering, neural, production, shipping]
---

# Neural Upscaling and Frame Generation

## Overview

用神经网络在低分辨率渲染结果上重建高分辨率画面（超分），以及/或者插出中间帧（帧生成）。**这是目前唯一大规模工业落地（Industry Adopted）的神经渲染技术。**

## Architecture

```
低分辨率渲染帧 ┐
Motion Vectors ├→ 神经模型 → 高分辨率输出
历史帧/Temporal State ┐
Depth / Albedo / Normal ┘
```

三代演进：
- **DLSS 2 类**：时域超分，用 motion vector reprojection 复用历史样本
- **DLSS 3/4 类**：+ 光流插帧（Frame Generation / Multi Frame Generation）+ 光线重建
- **DLSS 5 类**：见下，已经不是重建了

## Algorithm

核心是**时域样本复用 + 学习式重建**。engine 提供的 motion vector 质量直接决定输出质量——这也是 VFX 特别容易出问题的地方（粒子运动常常没有正确的 motion vector）。

## Performance

| 技术 | 状态 | 典型开销 |
|---|---|---|
| 超分 | Industry Adopted | 净收益（低分辨率渲染省下的 > 模型成本） |
| 帧生成 | Industry Adopted | 净收益，但增加输入延迟 |
| 光线重建 | Production Ready | 替代传统降噪器 |
| **DLSS 5（生成式）** | Early Production | **净负收益，约 -50% 帧率** |

**厂商官方量化样本（2026-09-09，DLSS SDK 310.9.1）**：DLSS 4.5 光线重建 Preset F（第二代 Transformer）进公开 SDK——参数量 +20%、计算量 +35%；4K 显存分配 467.97 → 578.84 MB，RTX 5090 处理时间 1.83 → 2.12 ms。⚠️ 光线重建预设字母与超分预设不通用，接入时勿混。

## Memory

超分/帧生成需要额外保存历史帧与 motion vector buffer。移动端受限明显。

## Hardware

- 超分：多数厂商有跨硬件方案（FSR / XeSS）
- **DLSS 5：仅 GeForce RTX 50 系列**（硬件锁定）

## Production Challenges

1. **时域稳定性**：抖动、沸腾、拖影
2. **UI 处理**：UI 必须后于超分合成，否则糊
3. **半透明与粒子**：motion vector 不正确 → 拖影重灾区
4. **DLSS 5 特有**：需要美术手工调 mask 与强度参数，工作流成本

## Game Engine Integration

NVIDIA Streamline 统一集成层 + UE5 官方插件。作为后处理链末端的独立 stage。

## Unreal Engine Possibilities

- Post Process 链末端
- **对 VFX 的直接影响**：Niagara 必须输出正确的 motion vector / depth / albedo，否则神经层会放大错误
- 移动端：需要评估 FSR 类方案的替代路径

## Related Concepts

- [[Neural Rendering]]
- [[Generative Rendering]]
- [[Temporal Stability and Artistic Intent]]

## Related Papers

- [[DLSS 5 — Generative Neural Rendering]]

## Related Technologies

- [[Arm Neural Graphics]] — 移动端分支（NSS/NFRU/NSSD），2026-09-08 发布；与 DLSS 5 路线对立（卷积 vs 生成式、开放可重训 vs 黑盒）

## Personal Knowledge

Current Level: **Normal**

## Learning Gap

1. 时域 reprojection 的细节（为何需要 motion vector、如何处理遮挡 disocclusion）
2. 帧生成与输入延迟的权衡
3. 不同硬件厂商方案的差异

## Next Step

把它当作**你已有的知识**来用，而不是新知识去学。真正值得做的是：审查你负责的特效在开启超分/帧生成时，motion vector 是否正确输出——这是一个马上能做的检查项。
