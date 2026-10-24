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
    [[2026-10-21-Zhang — Matrix-Game 4.0 Harnessing Video Models for Real-Time Playable Game Worlds|Matrix-Game 4.0]]：三角色硬分工（agent 写规则 / 引擎管状态 / 视频模型管画面），跑到 83.4 FPS
    共同点：把"世界如何推演"与"世界如何呈现"解耦

层次 3：引擎内生成式增强
    [[DLSS 5 — Generative Neural Rendering|DLSS 5]]：传统管线照常运行，生成模型只负责最终外观增强
    规则问题根本不出现——因为渲染器从未接管规则
```

工业界正在从层次 1 向层次 2/3 收敛。**层次 2 和 3 的共同点是：显式 world state 不丢，生成模型不碰规则。**

**第三个设计维度：记忆（2026-09 新增）。** 一个世界模型还要回答"**过去观测到的内容存在哪、怎么取用**"——[[2026-09-21-WorldCrafter — Consistent Video World Model with Implicit 3D-aware Memory|WorldCrafter]] 给出的答案是把**记忆本身当预算**：不存历史帧、不物化几何，而是"**按目标视角把历史压进固定 token 预算**"（其记忆开销 0.062 s/chunk，仅为 depth-based warp 路线 1.346 s 的 1/21.7）。**判据因此升级：除了问"什么东西跨过了引擎/模型边界"，再加一问"世界状态存放在哪、按什么条件取用"。**

**第四个设计维度：状态的可读性——"能执行"因此"能标注"（2026-10-22 新增）。** [[2026-10-21-Zhang — Matrix-Game 4.0 Harnessing Video Models for Real-Time Playable Game Worlds|Matrix-Game 4.0]] 暴露了一个此前没人当设计点讲的东西：一旦规则以**可执行形式**存放在引擎里，**标注就不再需要模型推断**。它的 114,334 段训练配对，字幕不是 VLM 事后看图写的，而是**规则模板直接从 world state 渲染**出来的（"玩家在 3 秒内向右移动并跳跃"这类）。这让三件事同时成立：① 标注成本 ≈ 0 且零幻觉；② 配对天然像素对齐（引擎同帧出 low-poly 与 RGB）；③ 换一条规则就换一个数据分布，**数据规模由规则空间决定，不由拍摄预算决定**。

于是判据再加一问：**"这套 harness 除了出画面，还能不能出标签、出评测、出反事实？"** 只能出画面的属于渲染器；能出标签的是数据工厂；能出反事实（改一条规则、重跑一遍）的才是**实验平台**——MG4.0 是库内第一个三项齐备的样本。附带的工程结论是：它在 5090 上 83.4 FPS / 111 ms，靠的不是更小的模型，而是把"哪些东西每帧必须重算"逐项砍掉（三层 KV、30 个 causal DiT、TAEHV 编码复用）。**"实时"在这里是系统账，不是模型账。**

## Prerequisites

- [[Neural Rendering]]
- [[Generative Rendering]]
- [[Real-Time Rendering]]

## Evolution

- 早期：GameGAN / Genie（纯像素）
- 2026-08：[[Magpie — Real-Time World Renderer for Interactive Games]]（引擎/渲染解耦）
- 2026-08：Code World Model（LLM-as-executable-state，arXiv 2608.25927，UE CEO Tim Sweeney 公开关注）
- 2026-09：[[DLSS 5 — Generative Neural Rendering]]（产品化的层次 3）
- 2026-09：[[2026-09-21-WorldCrafter — Consistent Video World Model with Implicit 3D-aware Memory]]（**可交互探索 + 记忆预算**：隐式 3D-aware 记忆 / 视角条件压缩 / 蒸馏拿实时——"连世界状态都不物化"的路线；**北大 × 腾讯 ARC Lab**）
- 2026-10：[[2026-10-21-Zhang — Matrix-Game 4.0 Harnessing Video Models for Real-Time Playable Game Worlds|Matrix-Game 4.0]]（**层次 2 的完成形态**：可执行 harness + 规则模板字幕 + 三阶段因果蒸馏，83.4 FPS；把"实时"拆成九项系统优化）
- 背景信号：NVIDIA Cosmos 3 全线开源（世界模型作为生成式世界的基础设施工具化）；SIGGRAPH 2026 Workshop 宣称 3DGS 已成为生成式世界模型的原生输出格式

## Related Concepts

- [[Generative Rendering]]
- [[Neural Rendering]]
- [[Motion Generation]]（世界模型在动作维度的对应）

## Game Applications

- 原型期视觉外包：玩法先行的团队先用白模 + 生成渲染出"卖相"，美术管线后置（Magpie 的明确目标场景）
- NPC/模拟：LLM 维护世界状态 → AI NPC 长期记忆与因果（Code World Model 方向）
- 数据飞轮：游戏引擎作为世界模型的主要可交互数据源（[[#Notes]]）
- **自家引擎当数据工厂**（2026-10-22）：MG4.0 的做法可复述为——用 low-poly 白模 + 规则脚本量产"像素对齐配对 + 零幻觉字幕"，再把 RGB 侧换成自己的材质/光照后处理。对 NGR 这类已有引擎的团队，**这是唯一一条不必依赖外购视频数据的路径**；代价是要先有可脚本化的规则层（也就是 harness 本身）。

## Important Papers

- [[Magpie — Real-Time World Renderer for Interactive Games]]
- [[DLSS 5 — Generative Neural Rendering]]
- [[2026-09-21-WorldCrafter — Consistent Video World Model with Implicit 3D-aware Memory]]（★ 2026-09-23 入库：记忆维度样本）
- [[2026-10-21-Zhang — Matrix-Game 4.0 Harnessing Video Models for Real-Time Playable Game Worlds|Matrix-Game 4.0]]（★ 2026-10-22 入库：**层次 2 最强样本**——可执行 harness / 规则模板字幕 / 三阶段因果蒸馏 / 九项推理优化；A 级）
- [[2026-10-22-Liu — AnyGameRenderer Entity-Consistent Generative Rendering Across Games|AnyGameRenderer]]（★ 2026-10-24 入库：**同一条轴的相反取舍**——把"状态"喂成纯视频 + 字幕（弱表示、通用、慢），对照 MG 4.0 的 low-poly 代理（强表示、快、窄）。**并用实体级指标第一次给"渲染半边"立了验收尺**：temporal match / position match / unrendered / hallucination）
- Code World Model（arXiv 2608.25927，Watchlist）

## Personal Knowledge

Current Level: **Hard**

## Learning Gap

不需要现在就补。这个概念目前的最佳学习方式不是读世界模型论文，而是**跟踪接口设计**：每个新系统出来，只看"什么东西跨过了引擎/模型边界，什么没有"。这是 Normal 难度，且直接服务于你的预算框架演进判断。

## Next Learning Step

把 [[Magpie — Real-Time World Renderer for Interactive Games|Magpie]] 的接口清单（白模帧 + 相机位姿 + 初始化风格，规则变量零传递）与 [[DLSS 5 — Generative Neural Rendering|DLSS 5]] 的 condition buffer 清单并排看一遍——30 分钟，零新前置。

若只读一篇：**[[2026-10-21-Zhang — Matrix-Game 4.0 Harnessing Video Models for Real-Time Playable Game Worlds|Matrix-Game 4.0]] 的 §3（harness 三角色）与 §5（三阶段蒸馏）**，约 25 分钟。理由：它把"层次 2"从口号变成可抄的工程分层，而且它的九项优化逐条对应一份"每帧必须重算什么"的清单——这份清单可以直接拿来对照你自己的渲染管线做减法。

## Notes

一个常被忽略的事实（Code World Model 论文明确指出）：**游戏视频不是世界本身，而是程序执行结果的像素投影**。游戏与模拟器是当前世界模型最主要的可交互数据来源——这意味着游戏行业同时是世界模型的**生产者（数据）和消费者（渲染）**，这个位置很特殊。
