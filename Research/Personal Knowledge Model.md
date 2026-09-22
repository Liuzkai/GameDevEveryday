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
| [[Particle Systems]] | **2026-09-21 入库，标 Easy**：VFX 域唯一历史锚点。**机制层（发射率/寿命/层级/混合）是你的日常工作；只有成本模型层被拆为 Normal**（屏占比密度 vs 绝对上限 / 发射器数=树节点数 / 随机数只在 Spawn 消费 / 加法混合是免排序唯一前提） |
| [[Shadow Mapping]] | **2026-09-21 入库，标 Easy**：补齐库里零覆盖的阴影域。**机制层（两遍渲染/bias/PCF/级联/cube map）不复述；只有成本模型层被拆为 Normal**（每灯 ≈ +1× 场景渲染 / 分辨率平方 / 全向光面数乘 / **阴影占比与材质复杂度反向**） |

### Normal — Primary Learning Zone（最高优先级）

| 知识 | 推断理由 |
|---|---|
| [[Motion Matching]] | 懂动画与 VFX 时序耦合，但未必深入检索/混合内部（9-17 起转静默项） |
| [[Procedural Content Generation]] | **2026-09-22 新建，标 Normal（推断）**：游戏开发背景对"程序化生成"概念不陌生（NGR 开放世界 + Houdini 方向整理），但形状文法形式化与图神经网络细节不要求。**⚠️ 若你自评 Houdini / UE PCG Framework 是日常域，说一声即可升 Easy** |
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

1. **本文件仍为推断值**（自 2026-09-07 建立，用户未做任何校正）——**第 15 天**。当前不影响工作（推送已按推断值自动调权）。**本次新增 1 个概念（[[Procedural Content Generation]] 标 Normal）——这是 9-21 以来首次重新增加 Normal 标签。**任何一次校正都会立刻改变推送重心，**仍建议做一次**；
2. **PBR / BRDF 的升档开关**：25 条自测通过即可标 Easy（我会在你标了之后停止推基础内容，转向其上的新研究）。**其中 24 条是纸面自测，唯一一条实测题是引擎侧 furnace test（30 分钟）—— 且 9-20 起升级为"查两头"：粗糙端是否变暗 + 光滑白色电介质球的掠射边缘是否有一圈偏亮**；
3. [[Motion Matching]] 的 40 分钟 Action 已于 9-17 降级为静默项——**想恢复随时说一声**；
4. **动态灯光维度复审**（UE 5.8 MegaLights 转 Production 的影响）已从 9-10 挂到 9-20，**已在 [[2026-W38]] 中列为下周第一优先级，不再逐日提示**；
5. **🔴 9-20 新增（对你的分档工作直接相关）**：**《控制：共振》把路径追踪 + 全局光照做成了所有光追预设的公共底座** → **"画质档位 = 在同一管线上调参数"这个隐含前提需要重新审视**。建议在 S/A/B/C × 五档矩阵里显式区分"**换参数**"与"**换管线**"两类档位差异。详见 [[2026-09-20]] 产业信号 2。
6. **🔴 9-21 更新（比 9-20 更尖锐，两条）**：
   - **档位实为"多个正交子系统的组合"**：[[2026-09-21]] 产业信号 1 查到《控制：共振》的菜单是 `RT Preset` × `Direct Lighting` × `Indirect Lighting` × `Transparency` × 各自 `Denoising` × **`帧生成倍率 2x–6x / Dynamic`** 的笛卡尔积。**你的五档可能需要定义"这一档下哪些子系统被替换"，而不是"资源上限乘多少"。** 另：**RT Ultra 档需 DLSS 4.5 Ray Reconstruction + RTX Mega Geometry → 厂商绑定**，跨平台体系里最高档可能在移动端/AMD 端根本不存在；
   - **神经渲染的开销会破坏锁帧依赖的玩法逻辑**：《铁拳 8》锁 60 → 神经渲染后 45 → **判定延迟、连招变慢动作**。**帧时间预算不只关乎流畅度，还关乎玩法正确性**；尤其**高帧率档（120 fps）下帧预算减半而特效开销不随帧率等比下降 → 建议单独复查高帧率档的特效预算**。
7. **🔴 9-21 新增：两项 30 分钟实测（都可直接做，直接把经验值换成依据）**：
   - **① 阴影占比实验**：材质极简场景抓 GPU 剖析，看 **Shadow Depths / Shadow Projection vs Base Pass** 占比 → 验证 **"材质越简，阴影越接近 50%"**（1978 年即写明的规律）。**含义：低档画质砍灯的收益比高档更大。**
   - **② 每千像素粒子数**：把现有绝对上限（3000/1200/400/100）与 **"屏占比 × 密度"** 对照一次 —— 绝对上限是**统计的**，屏占比密度是**几何的**。若两者在最重镜头下差很多，说明上限没抓住真正的成本变量。
   - 材料：[[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] · [[Reeves — Particle Systems (1983)]] · [[预算五维_1978-1983_源头图解]]
8. **⚠️ 9-21 硬约束（二手，待官方核实）**：二手来源称 **UE 5.8 MegaLights 不支持半透明物体 / 流体 / 云 / 发丝，也不支持前向渲染** → **特效打光不在 MegaLights 覆盖范围内，仍走 1978 的成本法则（每盏灯 +1× 场景）**。**"动态灯光变便宜"不能直接推到 VFX 侧。动手前请核实官方 release notes。**
9. **🟡 9-21 观察：库里首次出现"来自另一个客户端"的改动**（不是校正，但是engagement 信号）。运行开始时 `git push` 被拒（`fetch first`）—— 远程比本地多两个提交（`7ea6834 Sync`、`02ae272 Sync`，来自另一设备的 Obsidian Git / 同步客户端）。**唯一的实质内容改动是 [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 的 frontmatter 被重排**（内联数组 → 块列表、去引号，`status: studying` → `status: [reading]`）。
   - **⚠️ 关键：这不是 `user_level` 变更** —— `user_level: Normal` **原样未动**。该重排是 **Obsidian 属性编辑器的格式副作用**，**不能据此认为用户校正了 PKM**。
   - **但有一条正向信号**：**Cook-Torrance 笔记被打开并保存过**，而该笔记正是 **PBR 收口清单里 25 条中的 5 条所在**（且含 9-19 之前你主动研读的那条线）。**与"用户正在走 PBR 自测清单"这个判断一致**，但仍属弱推断。
   - **已把此情形写入规则文件 §41 Rule 9**（远程分歧的判定与处理流程 + "frontmatter 重排 ≠ user_level 变更"的判别规则），下次运行改为**推送前先 fetch**。
10. **🔴 9-22 新增（借 CGA shape 的一条分档体检，30 分钟可做）**：CGA shape（[[Müller — Procedural Modeling of Buildings (2006)]]）用 `r` 后缀显式区分"**会缩放的相对值**"与"**不缩放的绝对值**"——原文明确：**建筑部件并非都等比缩放**。
    - **同构到你的五档**：**特效参数也并非都随档位等比缩放**。建议给预算矩阵里每个参数标一个"**绝对 / 相对**"属性（如贴图尺寸、动态灯数偏绝对；时长、粒子密度可相对）。**凡是"整体乘个比例"的缩放方案，先检查这一项。**
    - 另借一条成本原则（同日）：**"离线生成量与运行时负载是两本账"** —— 与"成本不会消失只会转移"同构（PCG 生成端多省/多花，运行时另算）。
11. **（观察项）UE6 时间线**：2026-05 公布，**首作《火箭联盟》UE6 版已进入职业测试、2027 上线**（9-21 官宣）。含义：NGR 一类的 UE5 项目大概率全周期在 UE5 线上，但**引擎换代节奏在加快**（UE5 公布→首作约 3 年；UE6 约 1.5 年）。生产管线规划值得记一笔。

---

## 一处值得复用的分层模板（9-20 建立，9-21 首次用于新建概念）

[[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] 的 `user_level` 标为 **Hard**，但笔记里**显式分列了两层**：

| 层 | 内容 | 可读性 |
|---|---|---|
| **推导层** | Smith 随机输运的自由程分布与相位函数、微片辐射度量学 | **真正的 Hard** —— 不必现在动 |
| **结论层** | ①"被挡住 ≠ 被吸收" ②它不可实时 ③成本随粗糙度上升 ④按事件序列分解 lobe | **Normal 可直接拿走，不需要任何推导** |

**结论：`user_level: Hard` 不等于"这篇不能读"。** 面对 Hard 材料时，**先问"它的结论层是不是 Normal 的"** —— 如果是，就把两层分开记，Hard 的那层挂着等前置补齐即可。**这与本库 §14 的 Hard 策略（找最小前置、搭桥）是同一件事的一个更省力的入口。**

### 🔁 9-21 扩展：这个模板也可以用在 **Easy** 域上，用来安全地往库里加"你本来就懂的东西"

9-21 新建的两个概念（[[Particle Systems]]、[[Shadow Mapping]]）走的都是这条路：

| 层 | 内容 | 状态 |
|---|---|---|
| **机制层** | 粒子五步循环 / 发射率 / 寿命 / 层级；阴影两遍渲染 / bias / PCF / 级联 | **Easy —— 你的日常，不复述、不推送** |
| **成本模型层** | 屏占比密度 vs 绝对上限 · 发射器数=树节点数 · 随机数只在 Spawn 消费 · 加法混合是免排序唯一前提 · 每灯 ≈ +1× 场景渲染 · 分辨率平方 · 全向光面数乘 | **Normal —— 值得动手验证的部分** |

**这解决了此前的一个两难**：Easy 域的历史锚点**本该入库**（§12 例外第 4 条），但**整篇笔记很容易变成"教你已经在做的事"**。
**拆两层之后，Easy 域可以安全入库** —— **机制层只写"不复述"，写出来的全是成本模型层。**
**判据**：一个已经 Easy 的域，**还能否贡献 Normal 的材料？** 能，就入库；不能，就只留一行索引。

---

相关：[[2026-09-07]] · [[2026-09-22]] · [[2026-09-21]] · [[2026-W38]] · [[2026-09]] 技术雷达
