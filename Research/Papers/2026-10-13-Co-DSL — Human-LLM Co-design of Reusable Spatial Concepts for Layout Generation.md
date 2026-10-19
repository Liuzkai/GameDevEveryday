---
type: paper
title: "Co-DSL: Human-LLM Co-design of Reusable Spatial Concepts for Layout Generation"
authors: [R. Kenny Jones, Jiajun Wu, Maneesh Agrawala]
year: 2026
published: 2026-10-13
venue: "arXiv 2610.15290"
url: "https://arxiv.org/abs/2610.15290"
code: ""
project_page: ""
category: [pcg, layout-generation, llm, human-ai-interaction]
importance: B+
historical_importance: 0
game_relevance: 4
production_readiness: Prototype
user_level: Normal
status: read
---

# Co-DSL — Human-LLM Co-design of Reusable Spatial Concepts for Layout Generation

> Stanford（Jones/Wu/Agrawala——布局生成域的持续产出团队）。2026-10-14 cs.GR 周三批次（昨日 9 条主列表中的漏网件，今日补扫发现）。

## TL;DR

用户与 LLM **协同设计"可复用空间概念库"**，再由 LLM 用概念库写"布局程序"、求解器搜出满足程序的布局——两阶段分工：**人机协同建概念（LLM 写初版 → 用户看执行效果给轻量反馈 → LLM 修订）→ 声明式生成（LLM 只说"哪些概念成立"，求解器负责找出满足它们的布局）**。五个空间布局域验证：概念库让 LLM 捕捉到固定操作系统的布局模式，概念图表示是 LLM 可靠组合概念的前提。

## Problem

LLM 直接生成布局的两个老问题：① 一次性 prompt 生成难以沉淀**可复用的领域模式**（"吧台后面要有酒柜"这种工作室积累）；② 让 LLM 直接输出坐标，既慢又不可靠。固定操作集系统（.prev 工作）则缺乏个性化与扩展性。

## Core Idea

**"生成只做决策层"第 6 例**（库内家族：Magpie 引擎管规则 / ProxyBuild 拓扑代理 / PartLLM 分件 / ESG 物理一致 / 本篇概念约束）——且分工最彻底：

```text
阶段一：概念库共建
  用户给描述 → LLM 写成初始实现（DSL）
  → 用户在示例场景观察概念执行行为 → 轻量反馈 → LLM 修订
  → 沉淀为个性化概念库（可复用资产）

阶段二：声明式生成
  新布局请求 → LLM 写"布局程序"（只声明哪些概念应成立）
  → 程序实例化为 concept-graph（概念的声明式组合）
  → 求解器搜索满足全部概念的布局
```

关键表示是 **concept-graph**：LLM 不输出坐标，只指定概念组合关系——**组合的可靠性由表示保证**（消融证明：没有 concept-graph，LLM 直接组合概念的可靠性显著下降）。

## Key Contribution

1. **概念 = 可复用、可人机协同打磨的中间资产**：布局生成从"每次从零生成"变成"资产积累 + 组合"——PCG 的"规则"第一次以人机协同方式增量维护；
2. **决策/执行分离的极致形态**：LLM 的输出停留在"哪些概念成立"的声明层，几何求解交给 solver——幻觉被关在约束检查后面；
3. **生成程序带参数旋钮**：用户不改 prompt 即可探索"同族相关布局的分布"。

## Limitations

- 概念库质量上限 = 用户反馈的意愿与质量（冷启动靠人）；
- 五个布局域都是室内/语义布局类，自然地形/开放世界未验证；
- 求解器在大概念图上的搜索成本未给实时数字——离线工具形态；
- LLM 写 DSL 的错误模式（概念语义漂移）只在"轻量反馈"环节兜底。

## Game Development Relevance

- **关卡/场景 PCG 的工作流样本**："工作室概念库 + LLM 排布 + 求解器落地"可对照库内 PCG 线（[[Müller — Procedural Modeling of Buildings (2006)|CGA shape grammar]] 的规则由专家写死；Co-DSL 的规则由人机对话增量养成）——**规则获取方式的 20 年演化：手写 grammar → 学习 grammar → 对话养成 grammar**；
- 与 [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]] 对读：同为"LLM 做决策层"，ProxyBuild 锚定几何代理，Co-DSL 锚定概念约束——**锚定物从几何上升到语义**；
- ★ 2026-10-19 对照面出现：[[2026-10-16-Scene-Designer — Learning Design Intelligence for Agentic 3D Scene Generation|Scene-Designer]]——把设计知识**训练进模型**（agentic 直接操作 + 学习型 Critic），与本文"知识**资产化为概念库**"构成 LLM-PCG 的架构分岔：可审计可编辑（概念库） vs 端到端可学习（模型）；
- 概念库 = 可版本控制、可评审、可复用的**资产化规则**——与"生成物是资产、规则也是资产"的 PCG 资产观一致。

## Unreal Engine Relevance

- 无直接引擎映射（研究原型，DSL+求解器）；若产品化，对应 PCG framework 的规则资产层 / Editor 工具链（规则打磨 UI）。

## Technology Evolution

```text
2001/2003/2006  Parish-Müller / Wonka / Müller —— 手写 grammar（shape grammar 系）
2014~           学习式 PCG（GAN/VAE 时代）
2025~2026       LLM 决策层家族（ProxyBuild / PartLLM / Co-DSL）——"生成只做决策"成型
★2026 本篇       规则本身变成人机协同沉淀的可复用资产
```

## Relationships

### Based On

- LLM 代码生成 + 布局 DSL；作者前作（fixed-operation layout 系统）

### Related

- [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]] / [[2026-09-22-PartLLM — A Unified Multimodal Foundation for 3D Part Segmentation|PartLLM]] / [[2026-09-16-ESG — Generating Physically Consistent Dynamic 3D Scenes from Text|ESG]] — "生成只做决策层"家族（第 4/5/6 例）
- [[Müller — Procedural Modeling of Buildings (2006)]] / [[Parish-Müller — Procedural Modeling of Cities (2001)]] / [[Wonka — Instant Architecture (2003)]] — PCG 规则表示的三个历史锚点（规则由谁写：专家→数据→对话）
- [[Procedural Content Generation]]（概念）— 域锚点

### Followed By

- Watchlist：概念库共享/社区化；求解器实时化

## Personal Knowledge State

- **user_level: Normal（取三条抽象即可）**：① 决策/执行分离的"声明式"极致形态；② 规则资产化 + 人机协同养成；③ 表示保证组合可靠性。

## Learning Value

- 对预算工作的隐喻价值：**"约束声明"与"满足约束的搜索"分离**——与 SABC 预算"先定档位约束、再在约束内做质量取舍"同构。

## Notes

- 昨日批次 9 条主列表中唯一既未入库也未记名的条目——今日补扫升级入库（"漏网"记录在案：周三组判定时被"1 新+5 交叉+3 替换"的账目冲淡）。
- 作者组（Jones）在布局生成域连续产出，Watchlist 跟踪。
