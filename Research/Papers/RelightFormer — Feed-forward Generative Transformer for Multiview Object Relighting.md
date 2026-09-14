---
type: paper
title: "RelightFormer: Feed-forward Generative Transformer for Multiview Object Relighting"
authors: ["Hejun Wang", "Jinxi Li", "Junwei Jiang", "Shiwei Mao", "Hu Cheng", "Shouwang Huang", "Bo Yang"]
year: 2026
published: "2026-09-07"
venue: "SIGGRAPH Asia 2026"
url: "https://arxiv.org/abs/2609.07414"
code: "https://github.com/vLAR-group/RelightFormer"
category: [rendering, relighting, generative, neural]
importance: "B+"
game_relevance: "中"
production_readiness: "Prototype（离线）"
user_level: "Hard（[[Generative Rendering]] 下游）"
status: unread
---

# RelightFormer — Feed-forward Generative Transformer for Multiview Object Relighting

## TL;DR

前馈生成式 Transformer 直接做物体重打光，**完全跳过显式本征分解**（不估 albedo/normal/材质），用 latent illumination module 把目标环境图经 cross-attention 注入空间特征。配套发布 LOD 数据集（90K 物体 × 39K 光照）与代码。与 [[LightOpt — Lights Optimization for Real-Time Rendering]] 同属"灯光"问题域，但路线完全相反。

## Problem

图像重打光传统上走逆渲染管线：先估本征属性再打光——病态优化、误差累积。单图生成式方法又缺多视角线索，几何与材质交互理解不足。

## Core Idea

```
多视角输入图（无序，permutation-invariant PE）
        ↓
视频基础模型改造的 Transformer 主干
        ↓ latent illumination module（环境图 → cross-attention 注入）
目标光照下的重打光结果
```

不重建"物体是什么"，直接生成"物体在该光照下看起来怎样"。

## Technical Approach

1. 从视频基础模型适配架构，获得强生成先验
2. **Latent illumination module**：目标环境图动态注入空间特征
3. **置换不变位置编码**：多视角输入无顺序偏置
4. 自建 **Laval Objaverse Dataset (LOD)**：90K 物体 × 39K 独特光照

## Key Contribution

- 重打光从"逆渲染问题"改写为"条件生成问题"，SOTA 视觉质量 + 零样本泛化
- 代码与数据均开放（vLAR group）
- 单视角 / 多视角 / 新视角重打光三任务统一

## Game Development Relevance

- **与 LightOpt 的路线对照**（这是本文对你最大的价值）：

| | LightOpt | RelightFormer |
|---|---|---|
| 灯光表示 | 显式灯光参数（可微优化） | 隐式 latent（生成） |
| 输出 | 灯光数量/位置/强度 | 重打光后的图 |
| 与你预算的关系 | **直接**（动态灯光数上限） | 间接（资产预览/概念验证） |

- 潜在用途：VFX 概念阶段的快速光照氛围预览；营销图/ KV 的重打光。非运行时技术。

## Unreal Engine Relevance

无运行时映射。可作为 DCC 旁路工具：资产进引擎前先做光照一致性预检。

## Limitations

- 离线生成，无实时路径
- 物体级（Objaverse 分布），场景级与角色级未验证
- 生成路线的固有问题：物理正确性无保证，[[Temporal Stability and Artistic Intent]] 风险

## Related Concepts

- [[Generative Rendering]]
- [[Neural Rendering]]
- [[Inverse Rendering]]（本文是它的"绕过"路线）

## Related Papers

- [[LightOpt — Lights Optimization for Real-Time Rendering]] ★ 同问题域相反路线

## Personal Knowledge State

Current Level: Hard

Reason: 属 [[Generative Rendering]]（Hard）下游。但**不需要读懂架构**——读"显式优化 vs 隐式生成"的路线对照即可，这部分在你的 Normal 范围内。

## Learning Path

不建桥。Watchlist 条目。若 [[Differentiable Rendering]] 升到 Normal，回看本文与 LightOpt 的对照会更有收获。

## Notes

数据集 LOD（90K×39K）本身就是资产：若做灯光相关研究，这是现成的训练/评测资源。
