---
type: concept
title: "Scalability and Quality Tiers"
user_level: Easy
tags: [performance, production, user-domain]
---

# Scalability and Quality Tiers

## Definition

同一份内容在不同硬件能力下，用**分档参数**产出不同画质/性能平衡的机制。

## Core Principle

分档不是"开关功能"，而是**在多维参数空间里定义若干条可行轨迹**。常见维度：分辨率、阴影质量、后处理、粒子数量、植被密度、LOD bias、动态光数量。

## NGR 现行五档

```
PC_High      基准，特效完整
PC_Low       削减后处理 / 降分辨率 / 降粒子
Android_High 移动端上限，需要注意 TBDR 带宽
Android_Mid  主流机型目标
Android_Low  保底，优先稳定帧率
```

**移动端与 PC 的分档不是同一条曲线。** Android 侧是 TBDR 架构（见 [[Tile-Based Rendering]]），带宽与 tile 内存是首要约束；PC 侧更偏算力与显存。用同一套降级顺序在两端都会踩坑。

## 🔴 一个正在发生的前提变化：档位从"调参数"变成"选管线"（2026-09-20 记录）

**上面的定义隐含一个前提**：所有档位都跑**同一条渲染管线**，差异只在参数（分辨率 / 质量 / 数量 / 频率）。

**2026-09-20 出现了一个把这个前提顶掉的样本** —— Remedy《控制：共振》：

| 事实 | 数据 |
|---|---|
| **所有光追预设都强制内置路径追踪 + 全局光照** | PT 不是独立高档位，而是"开了光追就必然进入的管线" |
| 原生 4K + 中等光追，**关闭全部升频** | RTX 5090 **48 fps**；RTX 5080 **< 30 fps**（TechPowerUp） |
| 原生 4K Ultra，**开完整 PT vs 不开** | 5090 **72 → 39**（**−46%**）；4090 64 → 27；5080 56 → 23（GameGPU） |
| PS5 性能模式 | 1440p/60（**内部渲染约 864p + FSR 升频**） |

**后果**：**中间档消失了。** 玩家面对的不再是"高/中/低"，而是 **"进不进 PT 管线"的二值选择**，以及"用多激进的升频把它拉回来"。同期 TweakTown 的评述把结论写成同一件事：**升频已从"可选的性能加成"变成"基线优化工具"**。

> **对分档设计的三条具体建议**：
> 1. **在 S/A/B/C × 五档的矩阵里，显式区分两类档位差异：「换参数」与「换管线」。** 后者不是"降一档"，而是**一次结构性切换**（GI 方案、阴影方案、材质 Shading Model 数、是否启用神经层）；
> 2. **"每档一套独立的资源上限"这个前提要重新审视** —— 换管线的档位需要**另一套上限**，而不是同一套上限的缩放；
> 3. **移动端多一条独立约束**：神经层的算力**可能要先跟功耗墙谈判**。2026-09-20 记录：Windows Auto SR 走 NPU，但 **NPU 与 CPU/GPU 共享整机功耗预算**，官方明说部分轻薄本会升温 / 掉续航 / 小幅掉帧。**对 Android 三档而言，"要不要给神经层留功耗预算"是前置决策，不是后期优化。**

### 🔴 2026-09-21 续记：档位不是"一根滑杆"，是"多个正交子系统的组合"

9-20 说"档位从性能阶梯变成渲染管线切换"，**9-21 拿到了更精确的形态** —— 查《控制：共振》的完整菜单后，它的档位结构是**笛卡尔积**，不是单值：

| 子系统 | 档位 |
|---|---|
| Ray Tracing Preset | Off / Medium / High / Ultra |
| **Direct Lighting**（含独立 Denoising 开关）| Off / High / Ultra |
| **Indirect Lighting**（含独立 Denoising 开关）| Off / Medium / High / Ultra |
| **Transparency** | Off / Medium / High / Ultra |
| Upscaler / Render Resolution | FSR / DLSS / DLSS(Legacy)；**直接暴露内部像素**（2560×1600 面板下 Quality = 1707×1067）|
| **DLSS Frame Generation** | Off / **2x / 3x / 4x / 5x / 6x / Dynamic（自设目标帧率）** |

> **由此得到两条比 9-20 更可操作的结论**：
> 1. **每一档需要定义的可能不是"资源上限缩放多少"，而是"这一档下哪些子系统被替换 / 关闭"** —— 因为子系统之间不构成单调的阶梯（Indirect 可以开而 Direct 关，帧生成倍率与画质无关）；
> 2. **新增了一个此前不存在的档位维度：帧生成倍率（2x–6x）。** 它把档位从"我画得多好"变成"**我补多少帧**"，而 `Dynamic` 模式更是把档位变成**闭环目标帧率**。
> **⚠️ 口径警告（本库连续第三天踩到同类问题）**：关于"最低光追档是否已包含 PT"，两个实测来源说法**互相冲突**（一个说 Ultra 才是 PT，一个说最低档就开了 PT）→ **按本库规矩不可合并，标为口径冲突，待官方规格页核实。**
> **🔴 另一条跨平台硬事实**：**RT Ultra 档需 DLSS 4.5 Ray Reconstruction + RTX Mega Geometry → 厂商绑定**。**含义：跨平台五档体系里，最高档可能在移动端 / AMD 端根本不存在** —— 分出这条档位之前，先确认它在目标平台上是否可达。

### 🔴 2026-09-21 新增约束：帧预算不只关乎流畅度，还关乎**玩法正确性**

《铁拳 8》（物理引擎锁 60 Hz）开启神经渲染后 **60 → 44–47 fps（平均 45）**，后果不是"不够流畅"，而是 **命中判定延迟、招式变慢动作、连招与格挡时序错乱**。

> **对分档的直接含义**：
> - 若游戏逻辑（判定 / 动画 / 物理）对帧率有锁定或隐式依赖，**"某一档把特效推过帧预算"会改变判定时序，而不是仅仅掉帧**；
> - **高帧率档（如 120 fps）的帧预算是 1/2，而特效开销通常不随帧率等比下降** → **高帧率档的特效预算需要单独复查一遍**，不能从 60 fps 档按比例推导。

### 🔴 2026-09-24 续记：MegaLights 官方口径落定 —— "动态灯光维度"的账本重述（一手核验）

库内从 9-10 挂到 9-24 的"UE 5.8 MegaLights 官方支持边界"问题**今日结案**（UE 5.8 文档页 + release notes 逐句核对）：

| 维度 | 官方口径（UE 5.8） |
|---|---|
| 性能结构 | **开销基本恒定**；"无阴影与有阴影光源差别不大" → PC/主机档的灯光维度**与灯数脱钩** |
| 通用限制 | **与前向渲染器不兼容**（仍在） |
| 现存限制 | 定向光云阴影 / SSS 厚度估算 / 水·云·异质体积·局部体积 不支持；**半透明已有 Froxel 路径（不再是"不支持"）** |
| **Niagara 粒子光源** | **已支持**（逐发射器 "Allow Mega Lights" + "Cast Shadows"）← **更正 9-21 二手记录** |
| **平台** | PS5 / XSX\|S / PC；**不支持移动端、Switch、上代主机** |
| 档位开关 | `r.MegaLights.Allow 0` **可按 Scalability Level / Device Profile 禁用**（引擎原生档位钩子） |

> **对五档矩阵的三条重述**：
> 1. **PC 两档**：动态灯光从"每盏灯 +1× 场景渲染"（1978 法则）变为"**恒定开销 + 光照复杂度预算**"——"≤3/≤2/≤1/0"的含义需重述为"**同像素重要光源数 / 降噪质量预算**"；
> 2. **Android 三档硬性无缘**：MegaLights 不支持移动端 → 移动端仍走 1978 的账（每灯 +1×），且"前向渲染不兼容"再叠一层约束；
> 3. **特效打光口径更正**：9-21 的"特效打光不在 MegaLights 覆盖内"作废 —— **Niagara 粒子光源已被官方支持**（稀疏性/投影数为软性建议，待实测确认）。

## Prerequisites

- [[Real-Time Rendering]]
- [[GPU Architecture]]
- [[Tile-Based Rendering]]

## Related Concepts

- [[Real-Time VFX Performance Budgeting]]
- [[Overdraw]]
- [[Niagara]]
- [[Particle Systems]] —— **"屏占比 × 密度"是比"绝对数量上限"更几何的分档变量（1983 年即给出）**
- [[Shadow Mapping]] —— **"成本只看分辨率、与场景复杂度解耦"是"档位可以按分辨率定义"的根本理由**

## 🔴 分档的祖先：五个预算维度的原始出处（2026-09-21）

| 维度 | 祖先 | 成本法则 |
|---|---|---|
| 同屏粒子数 / 发射器数 | [[Reeves — Particle Systems (1983)]] | 成本随**数量线性** → 上限可大（数百~数千）；**单位 = "一个点"** |
| 动态灯光 | [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] | 成本随**灯数线性**（每灯 ≈ +1× 场景渲染）→ 上限必须极小；**单位 = "一遍完整场景渲染"**（**移动端三档仍成立；PC/主机档已因 MegaLights 重述 —— 见上 9-24 续记**） |
| 贴图尺寸 | [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] | 横向分辨率是**平方级**代价 → 每档只能翻倍 |
| VFX 时长 | [[Reeves — Particle Systems (1983)]] | 寿命以帧计、生成时决定 |

见 [[预算五维_1978-1983_源头图解]]。

**🔴 一条横贯所有维度的"表示律"（2026-09-25 补，出自 [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]]）**：

> **"细节表示跟着观察尺度走；当细节小到不该用几何时，预算从几何维度迁移到纹理维度。"**

- 原文两句原话：**"the rendering time of a texel is independent of the geometric complexity of the surfaces that it extracts"**（渲染时间与几何复杂度**解耦** —— 档位预算能成立的前提）；**"We should switch from the texel representation to actual geometry when viewing the model at this resolution"**（**近看切回几何** —— 分档阶梯的 1989 版本）；
- 当代同构：**发片 ⇄ 发丝**（[[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]]，自动升档）、**LSS 路径追踪毛发**（《巫师 3》重制版，2026-09 产业）——**"表示跟着尺度走"在 37 年后仍然是档位设计的底层原则**；
- **对五维矩阵的含义**：每一档不仅要问"参数给多少"，还要问"**这一档用哪种表示**"——毛发/植被/远景是最先暴露这条问题的资产。

## Game Applications

- [[AAA Real-Time VFX]]

## Personal Knowledge

Current Level: **Easy**

## Next Step

你的分档目前是**人工定义**的参数集合。值得探索的方向：把"画质档位"定义为**外观误差阈值**，再由优化自动求解各档参数——这正是 [[LightOpt — Lights Optimization for Real-Time Rendering]] 在灯光维度上做的事。
