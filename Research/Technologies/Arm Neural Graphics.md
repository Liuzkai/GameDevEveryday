---
type: technology
title: "Arm Neural Graphics (NSS / NFRU / NSSD)"
user_level: Normal
tags: [rendering, neural, mobile, production, industry-signal]
---

# Arm Neural Graphics (NSS / NFRU / NSSD)

> 2026-09-08 Arm Everywhere China（上海）发布。条目源于产业事件而非论文。

## Overview

Arm 首款 **AI-native 移动 GPU：Mali G2-Ultra NX**（随 CSS for Mobile 2 平台发布）。神经加速器直接集成在 **shader core 内部**，复用 GPU 内存系统/一致性缓存/控制结构——不是外挂 NPU，是渲染管线内的神经算力。

三件神经图形技术：

| 缩写 | 全称 | 功能 | 对标 |
|---|---|---|---|
| NSS | Neural Super Sampling | 低分辨率→高分辨率重建 | DLSS SR |
| NFRU | Neural Frame Rate Upscaling | 中间帧生成 | DLSS FG |
| NSSD | Neural Super Sampling & Denoising | 超分 + 光追降噪一体 | DLSS RR |

## 与 DLSS 5 的路线对立（最重要的认知）

| | Arm Neural Graphics | [[DLSS 5 — Generative Neural Rendering|DLSS 5]] |
|---|---|---|
| 网络类型 | **卷积网络**（"不想象、不创造"） | 生成式扩散（注入学到的外观先验） |
| 开放性 | **网络权重公开**（GitHub/HuggingFace），可用自家游戏数据重训匹配美术风格 | 闭源黑盒，不可重训 |
| 平台 | 移动端功耗/带宽预算内 | RTX 50 桌面独占 |
| 集成 | UE 插件 + Vulkan ML 扩展 + 自研引擎 SDK | Streamline + UE5 插件 |
| 哲学 | "take it and make it" | "take it or leave it" |

Arm 产品负责人原话点名 DLSS 5："DLSS 就算有软件支持也跑不了手机，它超出那些预算。"

## 官方数字（厂商口径，待第三方验证）

- 相对原生渲染：**4× 性能效率、-70% 外部显存流量**
- 官方演示游戏 Neural Dawn（与 Sumo Digital 合作）：上代 GPU 极限 40 FPS → 新 GPU 60 FPS
- 宣称 **UE MegaLights 在移动端变得可行**
- 第三代光追单元 + Opacity Micromaps：演示中帧率 +30%、光追负载 -70%
- NFRU 叠加后支持移动端 120 FPS 游戏会话

## 已确认接入方（对中国手游研发异常关键）

- **腾讯游戏 Central Tech：MagicDawn 引擎接入**；Arena Breakout Infinite（暗区突围）做 NSSD 演示
- **网易**：Where Winds Meet（燕云十六声），Messiah 引擎，首批出货
- **Unity 中国**：团结引擎接入
- **Infold**：Infinity Nikki（无限暖暖）NSS 集成

## Production Readiness

**Early Production → Production Ready 边缘。** 硬件未上市（搭载设备未公布），但 SDK/插件/重训工具链已开放，且有明确的第一方游戏阵容。比 DLSS 5 更早进入"开发者可碰"阶段。

## 对你的直接意义

1. **你的五档画质体系的移动端三档（Android_High/Mid/Low）即将多一个预算维度**：神经超分开/关、NFRU 开/关。这会像当年 DLSS 之于 PC 档一样，把"原生分辨率预算"变成"低内部分辨率 + 重建预算"
2. **可重训 = 可定制**：NSS 网络可以用 NGR 自己的数据重训——如果 NGR 走腾讯 Central Tech 的 MagicDawn 管线，这件事可能不需要你推动就会发生
3. **MegaLights-on-mobile 的官方背书**：你的"动态灯光数"预算维度（S≤3/A≤2/B≤1/C=0）在中长期可能不再以"灯光数"为约束单位
4. VFX 的老问题在新平台重演：**半透明/粒子的 motion vector 质量**将决定 NSS/NSSD 输出质量——与 [[Neural Upscaling and Frame Generation]] 中 DLSS 侧的教训完全同构

## Related Concepts

- [[Neural Rendering]]
- [[Scalability and Quality Tiers]] ← 直接作用对象
- [[Temporal Stability and Artistic Intent]]

## Related Technologies

- [[Neural Upscaling and Frame Generation]]（本条目是其移动端分支）

## Related Papers

- [[DLSS 5 — Generative Neural Rendering]]（对立路线）

## Personal Knowledge

Current Level: **Normal**（概念已懂， SDK 细节与重训流程未碰）

## Learning Gap

1. Vulkan ML 扩展的能力边界（什么样的网络能进 shader core 内的加速器）
2. 重训管线的实际成本（采集自家游戏数据 → 重训 → 部署的流程与周期）

## Next Step

不需要主动学习。**保持跟踪 MagicDawn 引擎侧动态**——如果 Central Tech 开放内部接入，第一时间评估 NGR 的 Android 档预算模型是否需要加"神经重建"维度。
