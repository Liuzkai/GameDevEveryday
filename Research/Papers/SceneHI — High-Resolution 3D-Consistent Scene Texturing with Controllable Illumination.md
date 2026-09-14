---
type: paper
title: "SceneHI: High-Resolution 3D-Consistent Scene Texturing with Controllable Illumination"
authors: ["Athanasios Tragakis", "Marco Aversa", "Daniela Ivanova", "Chaitanya Kaul", "Roderick Murray-Smith", "Daniele Faccio", "Paul Henderson"]
year: 2026
published: "2026-09-09"
venue: "ECCV 2026"
url: "https://arxiv.org/abs/2609.10363"
code: ""
category: [rendering, texturing, generative, pcg]
importance: "B+"
game_relevance: "中（场景贴图生成 + 烘焙阴影，面向生产流程）"
production_readiness: "Prototype"
user_level: "Hard（方法侧）/ 概念可读"
status: unread
---

# SceneHI — High-Resolution 3D-Consistent Scene Texturing with Controllable Illumination

## TL;DR

把 2D 扩散模型的高分辨率、光照感知先验"抬升"到 3D：无需微调或优化，直接在复杂多物体场景上生成 3D 一致的高分辨率贴图，并把**几何一致的烘焙阴影**直接生成进贴图 atlas——明确以"接入生产流程"为目标。生成时间比现有场景级方法少 80%。

## Problem

场景级 3D 贴图生成的老三难：多视角一致性、高分辨率、光照合理性，此前无法同时满足；且多数方法产出与生产管线（烘焙光照贴图）脱节。

## Core Idea

- **精确解析的 pixel-to-texel 映射**：跨视角对齐扩散轨迹，保证严格几何一致
- **High-Resolution Latent Textures (HRLT)** 作为持久画布：相机视角在 latent pixel 空间做去噪，共享底图再逐级细化
- **光照感知生成 pass**：把几何一致的阴影直接烘进 atlas

## Key Contribution

首个在不微调的情况下把高分辨率 2D 合成能力直接带到 3D 物体上的场景级方案；烘焙阴影进 atlas 是有意的生产导向设计。

## Game Development Relevance

- 与 PCG / 场景资产生成线相关：程序化场景贴图 + 烘焙阴影是开放世界资产管线的真实需求
- "烘焙阴影进贴图"是离线友好的选择，但**与你的运行时预算体系是两种世界观**——烘焙省运行时灯光，牺牲动态性；对照 [[Scalability and Quality Tiers]] 的动态灯光维度看取舍

## Unreal Engine Relevance

低-中：产出形态（贴图 atlas + 烘焙阴影）理论上可进 UE 资产管线，但无引擎集成。

## Limitations

扩散模型先验决定风格上限；烘焙阴影意味着光源静态；场景级方法的交互编辑性未验证。

## Related Concepts

- [[Generative Rendering]]
- [[Scalability and Quality Tiers]]

## Related Papers

- [[RelightFormer — Feed-forward Generative Transformer for Multiview Object Relighting]]（相反方向：不烘死，可重打光）

## Personal Knowledge State

Current Level: Hard（方法侧）/ 概念可读

Reason: 扩散 + latent texture 细节被 [[Generative Rendering]]（Hard）阻塞；但"把阴影烘进贴图换运行时成本"的取舍用你的预算语言即可理解。

## Learning Path

不读全文。看 Figure 与"烘焙阴影"一节，建立"生成式资产如何对接烘焙管线"的直觉即可，约 10 分钟。

## Notes

与 RelightFormer 构成又一对对照：**烘死（SceneHI，省运行时）vs 可重打光（RelightFormer，保动态）**——贴图/灯光预算权衡的生成式版本。
