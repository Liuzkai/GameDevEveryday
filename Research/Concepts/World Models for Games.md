---
type: concept
user_level: Hard
---

# World Models for Games

## Definition

让模型学习"世界如何演化"（而不仅是"下一帧像素长什么样"），从而支持交互式生成可探索、可操作的环境。在游戏语境下，核心矛盾是：**像素模型能学会"画面怎么变"，但学不会"规则怎么运行"**——规则、状态、因果不会出现在像素里。

## Core Principle

当前路线分裂为三个层次：

```
层次 1：纯像素预测（Genie 类）
    动作 + 历史帧 → 下一帧
    问题：规则靠猜，不可复现，不可调试

层次 2：引擎/代码管规则，模型管画面
    [[Magpie — Real-Time World Renderer for Interactive Games|Magpie]]：引擎保持 world state，视频模型只重画白模
    Code World Model（西湖大学+NTU）：LLM 用代码维护可执行 world state，视频模型负责呈现
    共同点：把"世界如何推演"与"世界如何呈现"解耦

层次 3：引擎内生成式增强
    [[DLSS 5 — Generative Neural Rendering|DLSS 5]]：传统管线照常运行，生成模型只负责最终外观增强
    规则问题根本不出现——因为渲染器从未接管规则
```

工业界正在从层次 1 向层次 2/3 收敛。**层次 2 和 3 的共同点是：显式 world state 不丢，生成模型不碰规则。**

## Prerequisites

- [[Neural Rendering]]
- [[Generative Rendering]]
- [[Real-Time Rendering]]

## Evolution

- 早期：GameGAN / Genie（纯像素）
- 2026-08：[[Magpie — Real-Time World Renderer for Interactive Games]]（引擎/渲染解耦）
- 2026-08：Code World Model（LLM-as-executable-state，arXiv 2608.25927，UE CEO Tim Sweeney 公开关注）
- 2026-09：[[DLSS 5 — Generative Neural Rendering]]（产品化的层次 3）
- 背景信号：NVIDIA Cosmos 3 全线开源（世界模型作为生成式世界的基础设施工具化）；SIGGRAPH 2026 Workshop 宣称 3DGS 已成为生成式世界模型的原生输出格式

## Related Concepts

- [[Generative Rendering]]
- [[Neural Rendering]]
- [[Motion Generation]]（世界模型在动作维度的对应）

## Game Applications

- 原型期视觉外包：玩法先行的团队先用白模 + 生成渲染出"卖相"，美术管线后置（Magpie 的明确目标场景）
- NPC/模拟：LLM 维护世界状态 → AI NPC 长期记忆与因果（Code World Model 方向）
- 数据飞轮：游戏引擎作为世界模型的主要可交互数据源（[[#Notes]]）

## Important Papers

- [[Magpie — Real-Time World Renderer for Interactive Games]]
- [[DLSS 5 — Generative Neural Rendering]]
- Code World Model（arXiv 2608.25927，Watchlist）

## Personal Knowledge

Current Level: **Hard**

## Learning Gap

不需要现在就补。这个概念目前的最佳学习方式不是读世界模型论文，而是**跟踪接口设计**：每个新系统出来，只看"什么东西跨过了引擎/模型边界，什么没有"。这是 Normal 难度，且直接服务于你的预算框架演进判断。

## Next Learning Step

把 [[Magpie — Real-Time World Renderer for Interactive Games|Magpie]] 的接口清单（白模帧 + 相机位姿 + 初始化风格，规则变量零传递）与 [[DLSS 5 — Generative Neural Rendering|DLSS 5]] 的 condition buffer 清单并排看一遍——30 分钟，零新前置。

## Notes

一个常被忽略的事实（Code World Model 论文明确指出）：**游戏视频不是世界本身，而是程序执行结果的像素投影**。游戏与模拟器是当前世界模型最主要的可交互数据来源——这意味着游戏行业同时是世界模型的**生产者（数据）和消费者（渲染）**，这个位置很特殊。
