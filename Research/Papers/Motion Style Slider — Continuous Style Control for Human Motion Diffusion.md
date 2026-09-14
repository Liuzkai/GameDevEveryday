---
type: paper
title: "Motion Style Slider: Endpoint-Supervised Continuous Style Control for Human Motion Diffusion"
authors: ["Chen-Chieh Liao (东京科学大学 / Cygames)", "Yichen Peng", "Yiyi Cai", "Yûi Ono (Cygames)", "Hiroki Hanaoka (Cygames)", "Erwin Wu", "Hideki Koike", "Shuichi Kurabayashi"]
year: 2026
published: "2026-09-08 (conference)"
venue: "ECCV 2026"
url: "https://gamehack.jp/383489"
code: ""
project_page: ""
category: ["Animation", "Motion Generation", "Production Workflow"]
importance: "A"
game_relevance: "High"
production_readiness: "Research"
user_level: "Normal"
status: unread
tags: [animation, style-transfer, diffusion, production-pipeline, game-studio-research]
---

# Motion Style Slider: Endpoint-Supervised Continuous Style Control

## TL;DR

游戏公司自己的研究所（Cygames Research）与东京科学大学的合作。传统动作风格迁移只能输出"喜/怒/哀"这类**预定义离散风格**；要做"再强一点""再收一点"的微调，就得重新拍 mocap 或补数据。本文只用两种数据——**不带情绪的标准动作** 和 **带风格的动作**（端点监督），就能训出"演技强度无级滑杆"的模型，中间强度的样本完全不需要准备。

## Problem

制作现场的真实痛点：风格微调的每次迭代都要重拍或补数据，时间与成本不可控。

## Core Idea

**Endpoint supervision**：只在两个端点（标准 / 满风格）上给监督，让模型自己学出中间连续谱。

## Technical Approach

- 基于 latent diffusion model 的人体动作编辑
- 训练数据只用两种端点（neutral / styled）
- 中间风格强度以滑杆形式连续可调
- 使用了 Cygames 大阪 mocap 工作室自采数据

## Key Contribution

- 把"离散风格选择"变成"连续强度控制"
- 关键是**数据成本极低**——不需要中间强度样本。这在生产环境里比模型精度更重要
- 由游戏公司主导、用自有 mocap 数据、面向真实制作痛点

## Game Development Relevance

这是本日**工业相关性最纯粹**的一条：它解决的不是学术 benchmark，而是"美术想再调一点点，要不要重拍"这个每个动画组都有的日常问题。

对你的延伸意义：同样的思路值得迁移到 VFX——**特效强度的无级调节**。技能 VFX 在 S/A/B/C 档之间切换时，目前是换一套参数/换一个系统；如果能做到"同一套资产 + 强度滑杆"，预算控制会简单得多。

## Unreal Engine Relevance

- 输出为动作数据，可进 anim sequence
- 思路可借鉴到 Niagara：用 User Parameter 驱动"强度"而非切换整套 emitter

## Limitations

- 风格定义依赖端点数据质量；端点选不好，滑杆中间会失真
- 未报告实时/引擎集成
- 仅人体动作，未覆盖非人形

## Related Concepts

- [[Neural Animation]]
- [[Motion Generation]]
- [[Production-Oriented ML]]

## Related Technologies

- [[Real-Time Generative Motion]]

## Related Papers

- [[STyMo — Fast and Controllable Few-Shot Motion Style Transfer]]
- [[MotionBricks — Scalable Real-Time Motions]]
- [[UniMate — One Unified Model to Animate Diverse Skeletons]]

## Personal Knowledge State

Current Level: **Normal**

Reason:
"用端点监督换连续控制"这个思想本身不需要扩散模型数学就能理解，你完全可以把它作为**设计原则**吸收。要完全读懂方法细节则需要 latent diffusion 基础。

## Learning Path

1. **先吸收设计原则**（不需要数学）：用最少的数据端点换最大的控制自由度
2. 把该原则映射到你的 VFX 分档：S/A/B/C 是 4 个离散端点还是 1 个滑杆？
3. 若要读方法：补 latent diffusion 基础

## Notes

2026-09-07 首次收录。今日 Tier A，且**对你的预算体系有可迁移的方法论价值**——这是少数几条"不用读懂模型也能用上"的论文。
