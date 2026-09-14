---
type: paper
title: "Lightweight Attention-based Indirect Illumination"
authors: ["SungYe Kim", "Wojciech Uss", "Wojciech Kaliński", "Alexandr Kuznetsov", "Rama Harihara", "Harish Anand (AMD)"]
year: 2026
published: "2026-07"
venue: "SIGGRAPH 2026 (Poster)"
url: "https://gpuopen.com/learn/lightweight-attention-based-indirect-illumination"
code: ""
project_page: "https://gpuopen.com/learn/lightweight-attention-based-indirect-illumination"
category: ["Rendering", "Global Illumination", "Neural Rendering"]
importance: "A"
game_relevance: "Medium-High"
production_readiness: "Research"
user_level: "Hard"
status: unread
tags: [gi, neural-rendering, rsm, ivpl, amd, real-time]
---

# Lightweight Attention-based Indirect Illumination (AMD, SIGGRAPH 2026)

## TL;DR

实时 GI 的两难：screen-space 方法快但丢掉屏幕外光；scene-wide 方法能补回来但要"收集数据"，塞不进标准渲染管线。AMD 的取法**把预算从模型复杂度挪到输入丰富度**：给 screen-space G-buffer 额外喂一份从每个光源渲染的 **Reflective Shadow Map (RSM)**，用 RSM texel 当作 indirect virtual point lights (iVPLs)，从而重建屏幕外的间接光。模型很轻（2.19M 参数 / 432 GFLOPs），但**未优化 FP32 PyTorch 下 512×512 要 45.56ms**。

## Problem

- Screen-space GI 快，但视锥外的光进不来
- 已有的 scene-wide / data-gathering 神经方法不适配标准渲染管线
- 先前工作泛化到训练集外场景的能力差

## Core Idea

**把预算从"模型复杂度"转移到"输入丰富度"**——这句是本文最值得记住的一句话。加一路 RSM 渲染（本来很多管线就有），让轻量模型也能看到屏幕外几何。

## Technical Approach

- 只预测 **indirect illumination** 分量（diffuse / specular 分开预测再合成，利用光传输的线性性）
- 输入：
  - 主相机：direct illumination、1spp 单 bounce（噪声大但便宜）、G-buffer、scene size
  - RSM 相机：G-buffer + reflected flux
- 三个 encoder：iVPL MLP / pixel-light UNet（UNet 是为了从噪声 1spp buffer 聚合空间上下文）/ pixel-geometry MLP
- Multi-head attention：pixel-geometry embedding 作 query，光数据作 key/value
- 两个 MLP decoder → diffuse / specular

## Key Contribution

- 用 RSM-augmented input 让轻量模型具备屏幕外感知
- 能泛化到**训练时未见过的场景**（test-only scene 上重建出沙发/扶手椅/茶壶的复杂高光）
- 比同数据训练的 CNN 基线（BCNN）间接光明显更丰富

## Game Development Relevance

- "预算从模型挪到输入"是一条**通用的实时优化设计哲学**，对 VFX 同样成立：与其加神经元，不如给已有的 G-buffer / depth / velocity 多喂一路信息
- 但 45ms @512×512 意味着当前形态**离实时差两个数量级**——这是研究原型，不是可落地方案

## Unreal Engine Relevance

- 对应 Lumen 的 screen-space / world-space GI 混合策略
- RSM 在 UE 中并非常规产物，需要额外渲染 pass

## Limitations

- 未优化 FP32 PyTorch，45.56ms @ 512×512（AMD MI250 服务器卡）→ 与游戏 GPU 完全不可比
- 残存 color shift，靠加训练场景缓解（说明模型尚未到容量上限）
- 仅 11 个合成场景训练，泛化的边界未探明

## Related Concepts

- [[Neural Rendering]]
- [[Neural Global Illumination]]
- [[Reflective Shadow Maps]]
- [[Global Illumination]]

## Related Technologies

- [[Real-Time Global Illumination]]

## Related Papers

- [[DLSS 5 — Generative Neural Rendering]]
- [[LightOpt — Lights Optimization for Real-Time Rendering]]

## Personal Knowledge State

Current Level: **Hard**

Reason:
需要理解光传输、iVPL、注意力机制与神经渲染训练范式。

## Learning Path

1. 先理解 RSM / VPL 这类**经典** GI 近似（这是 Normal 层，且对你理解 Lumen 有直接价值）
2. 再看"神经网络替换 VPL 的 gather/shading"这一步
3. 本文

## Notes

2026-09-07 首次收录。Tier A，但注意区分：**这条的"工业信号"比"论文价值"更重要**——AMD 也在把神经 GI 当作实时方向推。对你而言，真正该先学的是 RSM/iVPL 这个经典概念，而不是这篇论文本身。
