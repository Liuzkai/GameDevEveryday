---
type: paper
title: "Scene-Designer: Learning Design Intelligence for Agentic 3D Scene Generation"
authors: ["Xuanyu Liu", "Hongyan Feng", "Miao Pan", "Sunlai Chen", "Lizhou Liang", "Bing Sun", "Wenqi Zhang", "Xuhong Zhang"]
year: 2026
published: 2026-10-16
venue: "arXiv:2610.23121 (cs.CV)"
url: "https://arxiv.org/abs/2610.23121"
code: "https://github.com/ZJU-OmniAI/Scene-Designer"
project_page: "https://zju-omniai.github.io/Scene-Designer/"
category: ["pcg", "ai-agents", "scene-generation", "llm"]
importance: B+
historical_importance: 0
game_relevance: 3
production_readiness: Prototype
user_level: Normal
status: unread
---

# Scene-Designer — Learning Design Intelligence for Agentic 3D Scene Generation

## TL;DR

浙江大学团队给 agentic 场景生成补上"**设计智能 = 会设计 + 会评判**"两半：诊断发现 GPT-5.2 这类通用模型①纠错低效（明明一步能挪开的挡窗物体，18% 的案例要挪 5 次以上；物体碰撞类 5+ 步占 32%、最多 17 步）②评判不行（两场景对比 72%，四场景排序掉到 15%）。解法是两份数据集（Scene-OTA-35K 设计经验：轨迹压缩 110→30 步 + 逐步推理标注；Scene-OTC-15K 评判经验：**"有序质量退化"——三种扰动强度自动生成质量排序**，不靠人工偏好标注）+ 两阶段训练（joint SFT → Scene-GRPO：冻结 Critic 给"分支排序"与"子母改善"双局部奖励）。结果：Critic pairwise 89% / 四场景排序 76%（vs GPT-5.2 的 72% / 15%）；交互朝向错误 5+ 步纠正占比 40%→7%、窗口遮挡 21%→5%。**代码、项目页、模型全开源。**

## Problem

Agentic 场景生成（LLM 按步骤摆家具/建场景）暴露两类通用模型的结构性短板：

1. **会认错但不会高效改错**——缺"设计经验"：GPT-5.2 在 100 个场景的诊断里能发现错误，但简单错误也要试错多次（"移动障碍物"这种一两条动作能解决的事，18% 案例 5+ 步）。
2. **不会评优劣**——缺"评判经验"：pairwise 72% 尚可，四场景增量排序 15% 接近瞎猜；失败模式是"看见开阔就加分，看不见功能分组"。

根因：现有场景数据集（3D-FRONT 等）只有**静态场景状态**，没有"怎么设计的"与"什么算好"的过程性监督。

## Core Idea

**"评判能力也是被训练的资产。"** 设计与评判互补、共同训练、互相增强：

- **设计经验**（Scene-OTA-35K）：用 GPT-5.2 + SceneSmith 脚手架生成 1K 场景的构建轨迹，再用 **Causal Trajectory Consolidation** 压缩——用状态变化量 $\Delta^{\text{state}}_i$ 与错误减少量 $\Delta^{\text{error}}_i$ 标记疑似无效操作，GPT-5.2 复核后剔除，回放验证终态不变。平均轨迹 110 步 → 30 步，再对保留动作生成步级推理（OTA 监督）。
- **评判经验**（Scene-OTC-15K）：对高质量场景施加三级扰动（轻度布局偏移 → 功能破坏 → 严重物理违规）× 三种施加策略，**扰动强度自动给出从优到劣的完整排序**——可扩展偏好监督的干净配方（仅"同扰动不同物体"策略需人工核）。
- **Scene-GRPO**：从 Scene-Builder 复制出冻结 Critic 与可训练 policy；每个编辑步分支采样多个候选动作，Critic 给两个局部奖励——**branch-ranking**（兄弟场景谁好）+ **improvement**（子场景是否真比母场景好，抑制无效修改）——外加全局与格式辅助奖励。

## Key Contribution

- 三个数字：Critic 89%/76% vs GPT-5.2 72%/15%；纠错效率（5+ 步占比）交互朝向 40%→7%、窗口遮挡 21%→5%；轨迹压缩 110→30。
- **扰动排序式偏好监督**：把"标注偏好"变成"控制退化"——这是本文最可迁移的方法论（任何"质量有明确定义维度"的领域都能套）。
- 测试时 Critic 还能继续引导生成（选更好的编辑分支），训练后的评判资产在推理侧复用。
- 全开源（GitHub + HuggingFace + 项目页）。

## Why It Works

设计与评判共享同一组质量维度（物理有效/功能正确/布局连贯）——学评判等于给设计装了内部奖励模型；Scene-GRPO 的 improvement 奖励直接惩罚"试错式抖动"，把 SFT 阶段模仿到的"高效纠错"从模仿变成被优化的目标。论文还观察到设计与评判能力的**互相增强**趋势（练评判提升设计、练设计提升评判）。

## Limitations

- 室内家具场景域（3D-FRONT 资产系）；开放世界/地形/关卡结构未覆盖；
- 底座仍是 GPT-5.2 生成的轨迹再压缩——"经验"的上限受生成器质量约束（鸡生蛋问题靠人工筛选兜底）；
- 质量维度三分法（物理/功能/布局）对游戏关卡还缺"玩法性/叙事性"维度；
- 训练成本与基础模型规模未披露细节（附录）。

## Game Development Relevance

- **与 [[2026-10-13-Co-DSL — Human-LLM Co-design of Reusable Spatial Concepts for Layout Generation|Co-DSL]] 构成 PCG 生成侧的一对答案**：Co-DSL 把设计知识资产化为**概念库**（规则层），生成只做决策层；Scene-Designer 把设计知识训练进**模型**（agentic 直接操作）。两条路线的取舍——可审计可编辑（概念库）vs 端到端可学习（模型）——是团队引入 LLM-PCG 时要做的第一个架构决策。
- **"评判智能单独训练"直连游戏自动化 QA**：关卡/白盒的可玩性评审若能像 Scene-OTC 那样"受控降级自动生成排序"，评审基准的构建成本会大幅下降（对照库内 [[MOONWALK — Intent-Evidence-Action Alignment for Animation VFX Review|MOONWALK]] 的评审工作流）。
- 纠错效率诊断法（按错误类型统计"1 步解决率 vs 5+ 步占比"）本身就是一个好的 AI 工具验收指标模板。

## Unreal Engine Relevance

- 潜在落点：UE PCG 框架 + 外挂 agent 的白盒布局生成；Critic 可对接自动化关卡体检（物理/导航/玩法面覆盖检查）。
- 当前为研究原型（Prototype），未到生产；概念库路线（Co-DSL）离生产工具链更近。

## Technology Evolution

```text
LayoutGPT / Holodeck 2023-24（LLM 直接出布局）
  ↓
I-Design / SceneSmith / SceneWeaver 2024-26（agentic 迭代构建：工具执行 + 反馈）
  ↓
★ Scene-Designer 2026（本篇：设计+评判双经验训练，Critic-guided RL）
  ↓
（对照线）Co-DSL 2026：概念库人机协同养成——资产化 vs 模型化的分岔
```

## Relationships

### Related

- [[2026-10-13-Co-DSL — Human-LLM Co-design of Reusable Spatial Concepts for Layout Generation|Co-DSL]]——同域对照：规则资产化 vs 端到端 agentic
- [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]]——"生成只做决策层"线（本文是该线的**反面样本**：决策层本身被训练）
- [[Procedural Content Generation]]——概念底座
- [[MOONWALK — Intent-Evidence-Action Alignment for Animation VFX Review|MOONWALK]]——评审工作流的对照（人工评审对齐 vs 学习型 Critic）

### Followed By

- 待观察：开放世界域扩展；Critic 资产的跨域复用；与概念库路线的混合（概念库约束 + 学习型 Critic）

## Personal Knowledge State

`user_level: Normal`——取三条抽象即可：①设计智能 = 设计 + 评判两半；②扰动排序 = 可扩展偏好监督配方；③概念库路线 vs 模型路线的分岔。细节（数据管线/奖励函数）不必展开。

## Learning Value

1. **"评判是被训练的资产"**——对 VFX 评审、关卡体检、AI 测试都成立的视角；
2. **受控降级造数据**的配方模板（SABC 分档思维的同构物：控制退化强度 = 控制档位）；
3. LLM-PCG 的架构决策框架（资产化 vs 端到端）。

## Notes

- arXiv 2610.23121v1，2026-10-16 提交（美东周五批，2026-10-19 首扫窗口）；浙大 OmniAI Lab + Universal Identification Technology (Hangzhou)；39 页。
- **开源三件套齐全**（GitHub / HuggingFace / 项目页）——记名转正无障碍，后续核重点看社区复现。
- 引言中 GPT-5.2 四场景排序"~50%"与实验部分"15%"口径不同（前者为 Fig.3 图读数、后者为正文报告值）；本笔记采用正文报告值 15%。
- 与 Co-DSL 的对照关系已写入双方笔记（Co-DSL 侧后续补一行回链）。
