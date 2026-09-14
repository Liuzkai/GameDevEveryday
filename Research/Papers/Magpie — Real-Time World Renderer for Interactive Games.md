---
type: paper
title: "Magpie: Real-Time World Renderer for Interactive Games"
authors:
  - Xiaoyu Zhan
  - Xinyu Wang
  - Xiaohong Zhang
  - Huanjie Zhu
  - Tengjiao Sun
  - Pengcheng Fang
  - Jiaxing Yu
  - Yanwen Guo
  - Dongjie Fu
year: 2026
published: 2026-08-27 (arXiv technical report)
venue: arXiv tech report（Mogo AI + 南京大学 + University of Southampton）
url: https://arxiv.org/abs/2608.27168
code: ""
project_page: https://zhanxy.xyz/Magpie-website
category:
  - generative-rendering
  - world-models
  - game-engine
  - system
importance: A
game_relevance: Very High（系统架构层）
production_readiness: Prototype（作者自认离可交付游戏渲染器仍有差距）
user_level:
  - Normal
status: read
---

# Magpie: Real-Time World Renderer for Interactive Games

> 收录理由：发布于 8 月 27 日，略超 24h 窗口，但作为 [[DLSS 5 — Generative Neural Rendering]] 的**系统级对照样本**，属 Important Recent Discovery。

## TL;DR

把游戏拆成两半：**引擎继续管规则**（碰撞、状态、事件、玩法判定），**独立的 Render Server 用蒸馏后的 5B 视频模型把引擎输出的白模帧"重画"成最终画面**。规则变量一字节都不传给渲染侧——模型只负责"看起来好"，不负责"世界怎么运转"。

实测：单张 H100，渲染吞吐 32.2 FPS @ 1280×768，玩家操作到对应画面延迟 ≈1.55s。

## Problem

视频生成模型已经能"画"出逼真画面，但游戏不是电影：同一个操作必须永远产生同一个结果。撞墙就是撞墙——如果纯像素模型"猜"下一帧，今天弹开、明天穿墙，游戏就废了。**规则信息不会出现在像素里，模型无从学起。**

## Core Idea

**Gameplay/Visual 完全解耦：**

```
Game Engine（规则 + 世界状态 + 碰撞判定）
    │  输出：白模帧（无材质、无光照，只有几何占位）+ 相机位姿
    ↓
Render Server（蒸馏 5B 视频模型）
    │  初始化时接收：text prompt + 首帧风格图
    │  运行时接收：白模帧作为持续去噪条件；相机位姿检索历史相关帧
    ↓
最终画面
```

关键设计约束：**玩家动作、状态变量、对象属性、事件信号全部留在引擎侧，绝不传给 Render Server。** 这是用接口设计保证"规则可复现"，而不是指望模型学出规则。

## Technical Approach

- 训练数据：人工采集 **≈300 小时** UE 场景交互视频，覆盖移动、视角、驾驶、坐下、碰撞交互、待机；白模/高保真画面/相机位姿/结构化交互记录四路同步
- 模型：5B 视频模型蒸馏版，单 H100 部署
- 相机位姿 → 检索历史帧，维持视角一致性与一定程度的时序连续

## Key Contribution

这是**第一个把"生成式游戏渲染"作为完整系统工程问题**来陈述并给出诚实数字的公开方案。它的价值不在效果，而在：

1. **接口划分**：什么信息允许跨越引擎/模型边界——这是所有同类系统都要回答的问题
2. **诚实的性能报告**：32.2 FPS、1.55s 延迟，作者明说离可交付还远
3. **数据配方**：300h 人工游玩配对数据，给出了"造这种系统要多少数据"的第一个锚点

## Game Development Relevance

**对你是结构性相关。** 昨天 Daily 的判断："神经渲染把一部分表达力外包出去后，粒子/贴图/灯光预算项的含义会变。" Magpie 是这句话的极端形态：**整个材质/光照/特效管线都被外包**，预算模型里只剩下白模几何和碰撞体。在这个世界里：

- VFX 不再是一组 emitter 预算，而是**训练数据里的分布** + prompt
- "分档"不再是降 emitter 数，而是换模型规模 / 降分辨率 / 降帧率
- 美术意图通过**首帧风格图 + text prompt**注入——与 DLSS 5 的 artistic-direction values 异曲同工

这不是明天的事，但这是你预算框架的**长期自变量**，值得持续跟踪。

## Unreal Engine Relevance

直接在 UE 场景上采集训练数据。架构上是"UE 作为 gameplay server + 外部 render server"，与 UE 本身的渲染栈无关——某种意义上，它是 UE 渲染栈的**替代物假想实验**。

## Limitations

- 1.55s 端到端延迟 → 只能算"可交互演示"，不是可玩
- 单 H100 32 FPS → 成本是单机游戏的两个数量级以上
- 时序一致性靠相机位姿检索历史帧，长时间游玩的状态漂移未验证
- 风格只在初始化时注入，运行时无法调（对比 DLSS 5 的运行时 condition）

## Related Concepts

- [[Generative Rendering]]
- [[World Models for Games]]
- [[Neural Rendering]]

## Related Papers

- [[DLSS 5 — Generative Neural Rendering]] — 同一范式跃迁的另一端：DLSS 5 在传统管线上做生成式增强，Magpie 用生成模型替代整条管线
- [[MotionBricks — Scalable Real-Time Motions]] — "生成模型成为运行时系统"在动画侧的对应

## Personal Knowledge State

Current Level: **Normal（接口层）/ Hard（模型层）**

Reason: 与 DLSS 5 相同的策略——不读模型结构，读**接口清单**。引擎与 Render Server 之间传什么、不传什么，这部分你现有知识完全够。5B 视频模型的蒸馏与推理属于 Hard，暂不需要。

## Learning Path

读 Figure 1 与系统架构节即可，30 分钟。值得回答的问题：**如果引擎只输出白模，你的 VFX 预算体系里哪些维度还幸存？**（答案：碰撞体几何、触发事件、相机——全是 gameplay 侧，渲染侧一个不剩。）

## Notes

- 国内团队（Mogo AI + 南京大学），值得跟踪后续版本
- 同期还有西湖大学 Code World Model（LLM 维护可执行 world state + 视频模型出画面），思路互补，见 [[2026-09-08]] Watchlist
