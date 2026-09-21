---
type: concept
title: "Shadow Mapping"
user_level: Easy
aliases: [Shadow Map, Shadow Mapping, 阴影贴图, Depth Map from Light, 光源深度图]
prerequisites: [Real-Time Rendering, Tile-Based Rendering]
first_introduced: "1977–1978（Crow 提出两遍可见性；Williams 给出 Z-buffer 实现）"
tags: [rendering, shadow, lighting, dynamic-lights, image-space, classic]
---

# Shadow Mapping

> **本文件是本库此前完全缺失的一个域**（阴影在 §2 的覆盖清单里，但库里没有任何一篇阴影笔记）。
> 它同时是用户"**动态灯光**"这个预算维度的原始出处 —— **不做教学**（机制层 Easy），
> **只记录成本模型与它今天的三个分档旋钮。**

## Definition

一种**图像空间**的阴影判定方法：**从光源视角渲染一遍场景，只保留深度（"光源深度图"）；再从观察者视角渲染时，把每个可见点变换回光源视角，比较其深度与光源深度图的值** —— 若比光源看到的更远，则该点在阴影中。

**这是今天几乎所有实时阴影的基础**（CSM、Virtual Shadow Maps、MegaLights 都建立在它之上）。

## Core Principle

### 两次渲染 + 一个线性变换

```text
Pass 1（光源视角）：只写深度，不算着色  →  光源深度图 D_L
Pass 2（观察者视角）：正常渲染；每点 p → 变换到光源空间 → 比较 depth(p) 与 D_L
                                     p 更远 → 在阴影中
```

**成立的关键**：两个视角**都是 Z-buffer 视角**时，它们之间存在**线性变换**，可以把观察者视角的点映射到光源视角。这是 Williams 1978 相对 Crow 1977 的跨越 —— **"两遍渲染"是 Crow 提的，但"怎么把两遍对起来"是 Williams 解决的**。

> **⚠️ 命名史澄清（引用时请注意）**：Williams 1978 把光源视角的深度缓冲称为 **"depth map"**（该词引自 Levine et al. 1973），**不是 "shadow map"**。"shadow map" 是后起叫法。

### 🔴 成本模型（本概念最实用的部分）

| 成本项 | 量级 | 正比于 |
|---|---|---|
| **光源视角渲染（每盏灯）** | **≈ 1× 场景渲染**（无着色 → 所以总代价是"**约**"2×） | **灯数 × 场景复杂度** |
| **主渲染 + 判定** | 1× 场景渲染 | 场景复杂度 |
| **变换开销（后处理式）** | 常数 | **分辨率²** |
| **变换开销（逐点式）** | 变量 | 场景深度复杂度 |
| **内存** | **每盏灯 × 每面视图 × 分辨率²** | 灯数 × 视图数 × 分辨率² |

> **三条直接推论（对分档表可用）**：
> **① 一盏动态灯的代价单位是"又一遍完整的场景渲染"** —— 因此动态灯光的上限只能是 0–3 这种小整数；而粒子多 100 个的代价单位是"100 个点"。**两个维度上限数量级天生不同（3 vs 3000），不是拍脑袋。**
> **② 降分辨率省不掉那 1× 的场景渲染** —— 所以"灯太多"的解法通常是**关灯**，而不是降阴影质量。**这也正是 MegaLights 存在的理由。**
> **③ 全向光（点光）需要分扇区（常用 6 面）** —— **内存与渲染遍数同时乘以面数**，这是最容易被忽略的隐性乘数。

## Prerequisites

- [[Real-Time Rendering]] —— 帧缓冲、深度缓冲、光栅化
- [[Tile-Based Rendering]] —— 深度缓冲在移动端的带宽代价
- **光源视角的投影变换**（与观察者视角的相机变换同构）

## Historical Evolution

```text
1972  Newell/Newell/Sancha —— 隐藏面问题的一种解法
1974  Catmull —— Z-buffer（成本随深度复杂度线性增长；副产品是深度图）
1977  Crow —— 提出"两遍可见性即可算阴影"（限定平面多边形；点光需分扇区）
★ 1978  Williams —— Z-buffer 两视图 + 线性变换关联 → 曲面阴影首次可用；
                    系统处理 bias / 抖动 / 插值 / 低通滤波（PCF 的思想源头）
        ↓
2006  CSM（级联阴影贴图）—— 用多级把有限分辨率按距离分配
2008+ PCF / PCSS / VSM —— 硬边与噪声的处理（PCF 的物理意义 1978 年已写清）
        ↓
2015+ 阴影图集（atlas）+ 单遍多光 —— 把"每盏灯一遍"合并
2021  Virtual Shadow Maps（UE5）—— 逐页虚拟化，分辨率按需
        ↓
2024  MegaLights（UE 5.5）—— 大量灯共享结构化采样，绕开"每盏灯一遍"
2025-11  UE 5.7 —— MegaLights 转 Beta
2026-06  UE 5.8 —— MegaLights 转 Production-Ready
        ↓
        ★ 这条线上的每一个技术，都在绕开 1978 年列出的六个限制之一
```

## Important Papers

| 论文 | 年份 | 角色 |
|---|---|---|
| [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] | **1978** | **本概念的定义性论文**：成本模型、六个限制、bias / 抖动 / PCF 的原始形态 |
| **Crow 1977（*Shadow Algorithms for Computer Graphics*）** | 1977 | 提出"两遍可见性"与点光分扇区（**未入库**）|
| **Catmull 1974（Z-buffer）** | 1974 | 全部基础（**未入库**）|
| [[LightOpt — Lights Optimization for Real-Time Rendering]] | 2026 | **同域对照**：把灯光数与位置做成可微优化 —— "动态灯光上限可推导化" |
| [[RelightFormer — Feed-forward Generative Transformer for Multiview Object Relighting]] | 2026 | 灯光线对照（生成式重打光）|

## Related Concepts

- [[Real-Time Rendering]]
- [[Tile-Based Rendering]] —— 深度缓冲带宽
- [[Real-Time Global Illumination]] —— **阴影只解决"直接可见性"，不含间接光**；Lumen / MegaLights 都在补这半边
- [[Scalability and Quality Tiers]] —— **"成本只看分辨率、与场景复杂度无关"是"档位可以按分辨率定义"的根本理由**
- [[Real-Time VFX Performance Budgeting]] —— **动态灯光维度的来源**
- [[Participating Media]] —— **阴影贴图对体积介质失效**（粒子/雾需要 shadow map 采样或自有方案）
- [[Particle Systems]] —— **对照**：粒子"无阴影问题"（因其为发光点光源），而实体几何必须处理阴影 —— **两者成本结构完全不同**

## Technologies

- **CSM（Cascaded Shadow Maps）** —— 解"只在视锥内投影"的限制
- **PCF / PCSS / VSM** —— 解硬边与噪声
- **阴影图集（Shadow Atlas）** —— 多光共享一张图
- **Virtual Shadow Maps（UE5）** —— 逐页虚拟化
- **MegaLights（UE 5.5→5.8）** —— 把"每盏灯一遍"改成"共享结构化采样"
- **Ray-traced / Path-traced shadows** —— 《控制：共振》把 PT 做成所有光追预设的公共底座（见 [[2026-09-20]]）

## Game Applications

- **用户当前工作**：动态灯光预算（`≤3 / ≤2 / ≤1 / 0`）的成本依据
- **待办对应**：[[Personal Knowledge Model]] 待办第 4 条（**UE 5.8 MegaLights 转 Production → 动态灯光维度复审**）
- 开放世界光照预算、开放世界角色动画的落地阴影

## Personal Knowledge

Current Level: **Easy**

**按本库 9-20 建立的分层模板拆两层**：

| 层 | 内容 | 状态 |
|---|---|---|
| **机制层** | 两遍渲染、光源深度图、bias、PCF、级联、cube map | **Easy —— 不复述、不推送** |
| **成本模型层** | **① 每盏灯 ≈ +1× 场景渲染（无着色，故"约"2×）② 横向分辨率平方级 ③ 全向光按面数乘 ④ 阴影占比取决于材质复杂度** | **Normal —— 唯一值得动手验证的部分** |

## Learning Gap

**不是"不会阴影"，而是"灯光预算的论证是经验性的"**：

- `动态灯光 ≤3 / ≤2 / ≤1 / 0` 目前的依据大概率是"实测跑出来的经验值"；
- **它可以被推导**：单位是"完整渲染遍"（2× 中的 1×）+ 全向光的面数乘数 + 分辨率相关项；
- **最有用的一条**：**阴影占比与材质复杂度反向** —— 材质越贵，阴影相对越便宜；材质越简（低档画质往往会简化材质），**阴影就越接近 50%**。**含义：低档画质下砍灯的收益比高档更大。**

## Next Step

1. **（30 分钟，可做，直接对接待办）** 在材质极简的测试场景抓一次 GPU 剖析，看 **Shadow Depths / Shadow Projection** 与 **Base Pass** 的占比，验证"材质复杂度决定阴影占比"这条 1978 年就写明的规律；
2. **（复审用）** 面对 MegaLights，**要问的不是"快不快"，而是"它的成本函数现在关于哪个变量线性"** —— 因为 Williams 1978 的成本结构（每灯 1×）并未消失，只是换了变量名；
3. **（VFX 侧硬约束，需官方核实）** 二手来源称 **MegaLights 不支持半透明物体 / 流体 / 云 / 发丝，也不支持前向渲染** → **你的特效半透明领域不在 MegaLights 覆盖范围内**，特效打光仍走 1978 的账。**动手前请在官方 release notes 核实。**
4. **（判据）** 能说出"**降阴影分辨率解决不了灯数问题**"，即说明成本模型已经吃透。

---

相关：[[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] · [[Real-Time VFX Performance Budgeting]] · [[Real-Time Global Illumination]] · [[Scalability and Quality Tiers]] · [[LightOpt — Lights Optimization for Real-Time Rendering]]
