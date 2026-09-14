---
type: personal-knowledge-model
created: 2026-09-07
status: INFERRED — 待用户校正
---

# Personal Knowledge Model

> ⚠️ **本文件目前是推断值，不是你的真实输入。**
> 建立于 2026-09-07 首次运行，基于对用户工作背景的已知信息推断。
> **请直接修改**——这是整个推荐系统的状态源。

## 如何校正

两种方式，任选其一：

**方式 A（推荐）：改 frontmatter**
打开任意 Concept / Technology / Paper 笔记，修改：
```yaml
user_level: Easy      # 或 Normal / Hard
```

**方式 B：改本文件的分级清单**
直接在下面三个清单里移动条目，我会同步到对应笔记。

**方式 C：直接告诉我**
"把 X 标记为 Easy" —— 我会更新对应笔记的 frontmatter。

---

## 推断依据

已知的用户背景：

- NGR（王者荣耀世界）项目，淬炼系统相关工作
- 为新角色**技能 VFX 设计性能预算**
- 已建立"技能 × SABC 分级 × 实测指标 × 五档画质"的评价框架
- SABC 分级依据：触发频率 × 难度 × 伤害；占比约 S 10% / A 13% / B 42% / C 34%
- 已设计五个量化维度上限：Niagara 发射器（12/8/4/2）、同屏粒子（3000/1200/400/100）、贴图尺寸（2048/1024/512/256）、动态灯光（3/2/1/0）、VFX 时长（3s/2s/1s/0.5s）
- 预算配比原则：S/A/B/C 按 35-40% / 30% / 20% / 10%（与数量占比倒挂）
- 五档画质：PC_High / PC_Low / Android_High / Android_Mid / Android_Low
- 实测指标：Particles / DrawCalls / Primitives / OverDraw / CPUTime / GPUTime

由此推断：用户对**实时渲染工程、UE/Niagara、性能剖析、跨平台分档**是 Easy；对**神经方法、可微渲染、生成模型**是 Hard。

---

## 当前分级（推断）

### Easy — 已完全掌握，不主动推送

| 知识 | 推断理由 |
|---|---|
| [[Real-Time VFX Performance Budgeting]] | 用户正在**定义**这个领域 |
| [[Niagara]] | 日常工作工具 |
| [[Scalability and Quality Tiers]] | 已在设计五档画质体系 |
| [[Real-Time Rendering]] | 能定义粒子/贴图/灯光预算，必然已掌握 |
| [[Gaussian Splatting]] | 2026-09-11 标 Easy：排序/管线/密度控制三 Gap 闭环，4 条判据全过 |

### Normal — Primary Learning Zone（最高优先级）

| 知识 | 推断理由 |
|---|---|
| [[Motion Matching]] | 懂动画与 VFX 时序耦合，但未必深入检索/混合内部 |
| [[BRDF]] | 2026-09-14 新信号：用户主动研读 Cook-Torrance D/G/F 物理来源（非推断，用户自述） |
| [[Physically Based Rendering]] | 同上，与 BRDF 同一条学习线；库已备 [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 收口检查表 |
| [[Tile-Based Rendering]] | 做移动端分档，应有概念但未必系统 |
| [[Neural Upscaling and Frame Generation]] | 作为渲染工程师应已接触，未必深入 |
| [[GPU-Driven Rendering]] | 与 DrawCall 预算相关，应是部分掌握 |
| [[Temporal Stability and Artistic Intent]] | 有工程直觉，未必形式化 |

### Hard — 需建桥，不强行推送

| 知识 | 阻塞原因 |
|---|---|
| [[Neural Rendering]] | 缺信号处理直觉 + 训练侧词汇 |
| [[Generative Rendering]] | 缺一步生成模型基础 |
| [[Differentiable Rendering]] | 缺自动微分 + 可微光栅化 |
| [[Neural Global Illumination]] | 缺经典 GI 近似（RSM/VPL）作前置 |
| [[Neural Animation]] | 被 [[Motion Matching]] 阻塞 |
| [[Motion Generation]] | 被 [[Motion Matching]] 阻塞 |
| [[Inverse Rendering]] | 被 [[Differentiable Rendering]] 阻塞 |
| [[Neural Physics Simulation]] | 仅缺 attention 词汇；桥：粒子知识（Easy）+ "super-token merging ≈ 粒子 LOD" 一个类比 |

---

## 关键依赖链（Prerequisite Graph）

```
                    ┌─ [[Neural Rendering]] ──┬─ [[Generative Rendering]]
                    │                          └─ [[Neural Global Illumination]]
[[Real-Time Rendering]] (Easy)
        │
        ├─ [[Tile-Based Rendering]] (Normal)
        │
        └─ [[GPU-Driven Rendering]] (Normal)
                    │
[[Neural Upscaling]] (Normal)
        │
        ↓
[[Motion Matching]] (Normal)  ←── ★ 关键瓶颈
        │
        ↓
[[Neural Animation]] (Hard) → [[Motion Generation]] (Hard)

[[Differentiable Rendering]] (Hard) ←── ★ 第二个瓶颈
        │
        ↓
[[Inverse Rendering]] (Hard)
```

**两个瓶颈值得优先打通**：

1. **[[Motion Matching]]** — 一旦 Easy，解锁动画侧 3 条线
2. **[[Differentiable Rendering]]** — 一旦 Normal，能读懂 [[LightOpt — Lights Optimization for Real-Time Rendering]]，直接作用于你的预算工作

---

## 反馈信号

你的标记变化本身就是信号：

- 你把某知识从 Normal 改成 Easy → 我会停止推送其基础内容，并开始推**建立在它之上的新研究**
- 你把某知识从 Hard 改成 Normal → 说明桥搭对了，我会沿同一路径继续
- 你长期不改某 Hard 知识 → 我会降低该方向的推送权重，转入 Watchlist

---

## 已建立的 Mastery Criteria

以下知识有**客观**掌握判据，不用凭感觉：

- [[Motion Matching]] — 6 条判据
- [[Tile-Based Rendering]] — 5 条判据
- [[GPU-Driven Rendering]] — 4 条判据
- [[Gaussian Splatting]] — 4 条判据 ✅ 2026-09-11 达成（见 [[GS 图解 1 — 协方差与椭球：高斯的形状说明书|图解 1]]、[[GS 图解 2 — 排序瓶颈、管线冲突与密度控制|图解 2]]）
- [[Differentiable Rendering]] — 4 条判据（在 [[Learning Path — Differentiable Rendering]]）
- [[Neural Rendering]] — 5 条判据（在 [[Learning Path — Neural Rendering]]）

---

相关：[[2026-09-07]] · [[2026-09]] 技术雷达
