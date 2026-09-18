---
type: paper
title: "DLSS 5: Generative Neural Rendering"
authors: ["NVIDIA Research (ADLR)"]
year: 2026
published: "2026-09-03"
venue: "NVIDIA Research / Product (GeForce RTX 50 Series)"
url: "https://research.nvidia.com/labs/adlr/DLSS5"
code: ""
project_page: "https://research.nvidia.com/labs/adlr/DLSS5"
category: ["Rendering", "Neural Rendering", "Upscaling", "Production"]
importance: "S"
game_relevance: "Critical"
production_readiness: "Early Production"
user_level: "Normal"
status: unread
tags: [neural-rendering, real-time, nvidia, dlss, generative]
---

# DLSS 5: Generative Neural Rendering

## TL;DR

DLSS 5 不再"重建一个更贵渲染结果的近似"，而是**直接生成最终画面的外观**。它把生成式先验（从真实世界外观学到的东西，如皮肤次表面散射、叶片透光）注入实时帧，同时用 3D 几何 + 引擎 motion vector + 艺术方向值做约束，保证不破坏 authored content。一步像素空间扩散模型，4K 实时，因果且确定性，专为帧间时间稳定训练。2026-09-03 随 NBA 2K27 首发，仅 RTX 50 系。

> 注（2026-09-15）：Gears of War: E-Day（10-6 发售）官方确认为 **DLSS 4.5** 套件（SR + Dynamic MFG + Reflex），**并非 DLSS 5**——早前"DLSS 5 随 E-Day 秋季首发"的预期证伪，DLSS 5 首发仍仅 NBA 2K27。

> **注（2026-09-17 补录核实，来源：NVIDIA 官方 DLSS 5 FAQ）**：上一条需要做重要补充，而不是被推翻——
> - **首发 ≠ 全面可用**。官方口径：DLSS 5 于 **2026 秋季**正式推出，届时首发阵容约 15 款：**AION 2、Assassin's Creed Shadows、Black State、CINDER CITY、Delta Force、Hogwarts Legacy、Justice、NARAKA: BLADEPOINT、NTE: Neverness to Everness、Phantom Blade Zero、Resident Evil Requiem、Sea of Remnants、Starfield、The Elder Scrolls IV: Oblivion Remastered、Where Winds Meet**。NBA 2K27 是"最先拿到"的那一款，不是唯一一款。
> - **首发阵容里中文项目占比极高**：Delta Force（腾讯）、Justice / NARAKA: BLADEPOINT / Where Winds Meet（网易）、NTE（Hotta）、Phantom Blade Zero（S-GAME）。官方列出的支持厂商含 **Tencent、NetEase、Hotta Studio、NCSOFT、S-GAME**。这对判断"国内同行的技术跟进节奏"是一个直接信号。
> - ⚠️ **最重要的 caveat：GTC 的早期预览 demo 是跑在两块 RTX 5090 上的——一块渲染游戏，另一块专门跑 DLSS 5 模型。** 官方称会针对单 GPU 优化，**最低 GPU 规格与显存/性能开销在 FAQ 里均标注为"待公布"**。也就是说：**截至本次核实，DLSS 5 的单卡实时成本仍然是个未知数。**
> - 定位官方原话：DLSS 5 **不替代** Path Tracing——PT 提供光照**准确性**（位置对不对），DLSS 5 提供光照**真实感**（像不像有更大的光预算和更好的材质）。二者叠加。
> - 集成方式：**NVIDIA Streamline SDK 或 UE5 插件**，"与集成 DLSS Frame Generation 类似"。
> - 官方明确列为增强对象的材质：**skin（SSS）、eyes、hair、fabric**，及 rim lighting / contact shadows。

**对生产就绪度的修正**：本笔记原标 `Early Production`。考虑到①单卡成本未公布、②最低 GPU 规格未公布、③首发阵容尚未全部落地，**在官方给出单卡性能与显存数据之前，对它的预算影响只能做定性判断，不能做定量假设。** 这不影响它对"预算定义域"的长期结构性冲击判断（见下），但影响"明年能不能进 PC_High 档"这类具体决策。

## Problem

- 前几代 DLSS 是**重建**：近似一个本需要更高渲染预算才能得到的参考输出。画质上限被"渲染器本身能表达什么"锁死。
- 实时算力/显存预算限制了两件事：authored 场景表示的精细度 + 渲染算法复杂度。结果：皮肤 SSS、叶片透光、接触阴影这类效果被砍。
- 通用生成模型拿来直接用有两个死穴：① 不保证保真于开发者 authored 的内容与结构；② 推理成本远超交互帧预算，尤其 4K。

## Core Idea

**3D-guided neural rendering（渲染器接地的生成式方法）**：

生成模型不是自由发挥，而是被当前渲染帧、引擎 motion vectors、跨帧 temporal state、以及 artistic-direction 值共同 condition。训练时再用 renderer-derived scene attributes 做 consistency supervision，把生成结果"钉"在 authored scene 上。

## Technical Approach

- One-step pixel-space diffusion model（不是多步采样，为帧预算而生）
- Condition：rendered frame + engine motion vectors + carried temporal state + artistic-direction values
- 训练：renderer-derived scene attributes 做一致性监督
- 推理：causal、deterministic、逐帧固定算力预算、最高 4K
- 控制项：
  - `Structure Intensity` → 高频细节（接触阴影、AO、反射）
  - `Tone Intensity` → 低频光照/材质响应；置 0 可完全保留原帧色彩
  - 语义 AI mask：可区分角色 / 环境分别调强度
  - per-pixel uplift control masks
- 集成：NVIDIA Streamline + UE5 plugin，可与 SR / MFG / RR 叠加

## Key Contribution

1. 第一个**生成最终显示外观**而非重建参考的 DLSS。
2. 第一个**产品化、实时**的生成式渲染模型。
3. 把"保留艺术创作意图 / 帧间时间稳定 / 4K 实时"三个生产硬门槛作为显式设计目标写进架构。

## Game Development Relevance

**对性能预算的冲击是结构性的。** 之前"同屏粒子数 / 贴图尺寸 / 动态灯光数"这些预算项，本质是在为"渲染器表达力不足"买单。DLSS 5 这类方法把一部分表达力外包给学到的外观先验——意味着：

- 中长期看，部分 VFX 的写实度可以由神经层补，而不必全部由粒子/贴图层硬堆 → **预算模型的自变量会变**
- 但代价明确：已有报告显示帧率约降一半；且只覆盖 RTX 50 PC 端
- 对移动端（Android 三档）短期**零影响**

## Unreal Engine Relevance

- 官方 UE5 插件 + Streamline，属于 Post Process 之后的最终 stage
- 与 Niagara 的关系：目前是**叠加增强**，不是替代。粒子该给的结构信息（motion vector、depth、albedo）仍然是生成模型的 condition 输入 → **Niagara 输出的 buffer 质量会直接影响神经层增强的稳定性**
- MegaLights / Lumen 的 GI 结果会成为生成模型的 ground，错误会被放大也可能被修正

## Limitations

- 仅 GeForce RTX 50 系列（Hardware lock）
- 帧率代价显著（社区/开发者反馈约 -50%）
- 玩家接受度存疑（TechPowerUp  poll：58% 玩家不希望 AI 动他们的游戏）
- 激进设置下会改变角色面部与光照（Skyrim mod 已有明显案例）
-  Kunstlerische Intent 的"保留"依赖开发者手工调 mask 与强度，工作流成本不低

## Related Concepts

- [[Neural Rendering]]
- [[Generative Rendering]]
- [[Real-Time Rendering]]
- [[Temporal Stability]]
- [[Artistic Intent Preservation]]

## Related Technologies

- [[Neural Upscaling and Frame Generation]]
- [[Real-Time Global Illumination]]

## Related Papers

- [[Lightweight Attention-based Indirect Illumination]]
- [[LightOpt — Lights Optimization for Real-Time Rendering]]

## Personal Knowledge State

Current Level: **Normal**

Reason:
你熟悉实时渲染管线的预算与瓶颈（Niagara、Overdraw、DrawCall、五档画质），能理解"上游渲染输出作为下游模型 condition"这一结构。尚未系统掌握的是：一步扩散模型如何在毫秒预算内跑完、temporal state 如何跨帧传递、deterministic 推理与随机采样的差别。

## Learning Path

1. 先补 [[Neural Rendering]] 的概念框架（不用推公式）
2. 搞清 one-step diffusion / consistency model 与多步采样的区别（这一步是 Normal → Easy 的关键）
3. 读 NVIDIA 的 DLSS 5 技术页 + Streamline 集成文档，看 condition buffer 清单
4. 对照你的淬炼预算：列一张"哪些预算项未来可能被神经层外包"的清单

## Notes

2026-09-07 首次收录。这是本日 Tier S 条目——不是因为效果好，而是因为它**改变了性能预算的定义域**。对你负责的 NGR 淬炼预算而言，需要关注的不是"要不要用"，而是"当它成为 PC_High 档的既有前提时，S/A/B/C 各档的粒子与灯光预算是否还该按原逻辑分"。

**2026-09-18 补记（媒体口径，非官方）：数值格式本身是一道硬门槛。**
社区把 DLSS 5 的构建**从 FP8 改为 FP16** 之后，它开始在 RTX 20/30 系上跑起来（更慢）。原因：**FP8 只有 Blackwell（RTX 50）有硬件加速**，其他卡只能退回 FP16/FP32 路径。

> 这条对你有实际价值：**神经渲染的门槛不只在算力总量，还在"数值格式有没有硬件加速"。** 做分档时，"这一档能不能跑神经层"可能要按**格式支持**而不是按**算力代际**来划——这是一个新的分档维度。
>
> 另一条同日产业信号：AMD 被传正在开发对标方案（代号 "Neural Lighting"），可能限定 RDNA 5，且**除屏幕空间外还调用引擎侧几何数据**（含视野外几何）。若成立，"屏幕空间 vs 引擎感知"会成为下一代神经渲染的阵营分界——见 [[Lightweight Attention-based Indirect Illumination (AMD)]] 的 2026-09-18 补记。
