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
| [[Motion Matching]] | 懂动画与 VFX 时序耦合，但未必深入检索/混合内部（9-17 起转静默项） |
| [[BRDF]] | 2026-09-14 新信号：用户主动研读 Cook-Torrance D/G/F 物理来源（非推断，用户自述） |
| [[Physically Based Rendering]] | 同上，与 BRDF 同一条学习线；**来源侧（D·G·F 出处）9-17 闭合、工程侧（进引擎 + IBL 查表）9-18 闭合、能量侧（多次散射）9-19 闭合**，收口清单 25 条 |
| [[Multiple Scattering and Energy Compensation]] | 2026-09-19 新建：**PBR 最后一块账本**（单次散射丢的能量 + 三条补法各缺哪一角）。判据是你日常接触的高粗糙度材质观感；**含唯一的引擎侧实测题（furnace test）** |
| [[Split-Sum Approximation]] | 2026-09-18 新建：环境光镜面反射的实时近似（UE Sky Light / 反射捕获的底层）；判据是你日常接触的固定开销项 |
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

- [[Motion Matching]] — 6 条判据（静默中）
- [[Tile-Based Rendering]] — 5 条判据
- [[GPU-Driven Rendering]] — 4 条判据
- [[Gaussian Splatting]] — 4 条判据 ✅ 2026-09-11 达成（见 [[GS 图解 1 — 协方差与椭球：高斯的形状说明书|图解 1]]、[[GS 图解 2 — 排序瓶颈、管线冲突与密度控制|图解 2]]）
- **PBR / BRDF 收口清单 — 25 条**（Cook-Torrance 5 + Kajiya 5 + Walter 5 + Schlick 5 + **Karis 5**，2026-09-18 由 [[Karis — Real Shading in Unreal Engine 4 (2013)]] 补齐最后一组；其余 20 条见各论文笔记末尾）
  - **清单之外的最后一条具名缺口（多次散射能量补偿）已于 2026-09-19 闭环、2026-09-20 由"三角形"扩为完整谱系**：[[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]（工程解）+ [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media]]（理论解）+ [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]]（**精确真值**）+ [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]]（**实时落地，零新增资源**）。**不计入 25 条**，标为"读了，不是会了"；
  - **⚠️ 25 条里唯一的引擎侧实测题（唯一待做动作，9-20 升级为两项检查）**：① 纯金属球 + 只有环境光 + Roughness 0→1 截图 → **粗糙端是否整团变暗**；② **光滑白色电介质球 → 掠射边缘是否有一圈偏亮（超额能量）**；③ 切换多次散射补偿开关对比。做法见 [[多次散射_五条补法路线与实时落地图解]] 第 5 节。**两项都做完即可把 [[Multiple Scattering and Energy Compensation]] 标 Easy。**
- **[[Hair Rendering]] — 5 条判据**（2026-09-18 随 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 建立：三条光路 ↔ 三个视觉现象 / 黑发为何无次级高光 / 双高光机制 / 微面为何不适用 / 砍留优先级）
- [[Differentiable Rendering]] — 4 条判据（在 [[Learning Path — Differentiable Rendering]]）
- [[Neural Rendering]] — 5 条判据（在 [[Learning Path — Neural Rendering]]）

---

## 待用户处理的推断项（每次运行检查）

1. **本文件仍为推断值**（自 2026-09-07 建立，用户未做任何校正）——**第 13 天**。当前不影响工作（推送已按推断值自动调权），但**本周新增 7 个概念、其中 5 个标为 Normal，推断误差正在累积**。任何一次校正都会立刻改变推送重心，**建议本周内做一次**；
2. **PBR / BRDF 的升档开关**：25 条自测通过即可标 Easy（我会在你标了之后停止推基础内容，转向其上的新研究）。**其中 24 条是纸面自测，唯一一条实测题是引擎侧 furnace test（30 分钟）—— 且 9-20 起升级为"查两头"：粗糙端是否变暗 + 光滑白色电介质球的掠射边缘是否有一圈偏亮**；
3. [[Motion Matching]] 的 40 分钟 Action 已于 9-17 降级为静默项——**想恢复随时说一声**；
4. **动态灯光维度复审**（UE 5.8 MegaLights 转 Production 的影响）已从 9-10 挂到 9-20，**已在 [[2026-W38]] 中列为下周第一优先级，不再逐日提示**；
5. **🔴 9-20 新增（对你的分档工作直接相关）**：**《控制：共振》把路径追踪 + 全局光照做成了所有光追预设的公共底座** → **"画质档位 = 在同一管线上调参数"这个隐含前提需要重新审视**。建议在 S/A/B/C × 五档矩阵里显式区分"**换参数**"与"**换管线**"两类档位差异。详见 [[2026-09-20]] 产业信号 2。

---

## 一处值得复用的分层模板（9-20 建立）

[[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] 的 `user_level` 标为 **Hard**，但笔记里**显式分列了两层**：

| 层 | 内容 | 可读性 |
|---|---|---|
| **推导层** | Smith 随机输运的自由程分布与相位函数、微片辐射度量学 | **真正的 Hard** —— 不必现在动 |
| **结论层** | ①"被挡住 ≠ 被吸收" ②它不可实时 ③成本随粗糙度上升 ④按事件序列分解 lobe | **Normal 可直接拿走，不需要任何推导** |

**结论：`user_level: Hard` 不等于"这篇不能读"。** 面对 Hard 材料时，**先问"它的结论层是不是 Normal 的"** —— 如果是，就把两层分开记，Hard 的那层挂着等前置补齐即可。**这与本库 §14 的 Hard 策略（找最小前置、搭桥）是同一件事的一个更省力的入口。**

---

相关：[[2026-09-07]] · [[2026-09-20]] · [[2026-W38]] · [[2026-09]] 技术雷达
