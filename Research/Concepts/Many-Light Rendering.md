---
type: concept
user_level: Normal
aliases: ["Many-Light Problem", "多光源问题", "Many Lights"]
prerequisites: ["[[Real-Time Rendering]]", "[[BRDF]]", "[[Rendering Equation]]"]
first_introduced: "Ward 1994（重要性排序）；作为独立问题成型于 Dachsbacher et al. 2014 STAR"
---

# Many-Light Rendering

## Definition

**场景里的灯多到"逐盏算"不划算时，怎么把光照算出来。** "多"不是绝对数量，而是**相对预算**：对光栅化管线，几十盏就可能"多"；对离线路径追踪，几万盏才叫多。

判据只有一句：**当"每像素要算几盏"成为瓶颈，你就在 many-light 问题里。**

## Core Principle

成本可以写成两个因子的乘积：

$$\text{光照成本} \;\approx\; \underbrace{\text{每像素要完整算几盏}}_{\text{查询侧}} \;\times\; \underbrace{\text{一盏算多贵}}_{\text{着色侧}}$$

**四个家族各自只动其中一个因子，或者动"场景里到底有几盏"这个前置量。**

| 家族 | 代表 | 动的是 | 账在哪付 | 一次付多少 | 治不了 |
|---|---|---|---|---|---|
| **① 剔除** | tiled（2009）/ clustered（2012）/ Tiled-Z（2017） | **每个像素看哪张列表** | 每帧 | 每 tile 一次列表构建 | 列表本身有多长 |
| **② 层次近似** | [[Walter 2005 — Lightcuts A Scalable Approach to Illumination\|Lightcuts 2005]] / MRCS 2007 / LightSlice 2011 | **一次查询代表多少盏** | 每帧（每视角） | 每像素几百条 shadow ray | 可见性（界取 1）；需光线追踪 |
| **③ 重采样** | [[ReSTIR]]（2020）/ MegaLights（2025） | **每帧抽哪几盏、复用谁的** | 每帧 + 历史 | 常数开销 + 时域状态 | 拖影 / 样本退化；有偏的 clamp |
| **④ 改写内容** | [[LightOpt — Lights Optimization for Real-Time Rendering\|LightOpt 2026]] | **场景里到底有几盏** | **资产时一次** | **~10 min/场景** | 镜面与视角相关效果（目标里没有） |

**这张表的用法不是"选一个"，而是"看你的瓶颈落在哪一格"**：

- 列表太长 → ①（但表还是长）→ ②；
- 每像素几百条光线太贵 → ③；
- **列表怎么删都还是长，因为内容里就有那么多灯 → ④**。

**四者可以叠加**：LightOpt 原文明确说它的输出"可与运行时光源选择算法配合"——**先把灯架改小，再在剩下的灯上做 ①②③**。

## 第二个轴：近似 vs 采样（比四个家族更重要）

| | 近似（②） | 采样（③） |
|---|---|---|
| 答案 | **确定性** | 随机 |
| 误差形态 | **系统偏差**（稳定的错） | **方差**（正确地抖） |
| 怎么验收 | 误差图 / **误差上界** | 方差 / 时间累积后的收敛 |
| 玩家容忍度 | 低（"这里一直不太对"） | **更低**（闪烁与噪点最刺眼） |
| 典型失效 | 高遮挡场景误差变大 | 拖影、收敛慢、运动处失效 |

> **这条分界线决定"验收指标"**：给近似方法报方差无意义，给采样方法报误差上界也无意义。**本库 11-14 那条"代理必须说出它漏掉哪一维"，在这里的具体形态是：②类必须报"最坏差多少"，③类必须报"P99 收敛时间"。**

## 第三个轴：灯数 vs 重叠数（LightOpt 表 2 的教训）

**"灯少"不等于"便宜"。** 一个像素被 30 盏灯照到，即使场景只有 30 盏灯，它也是 many-light 场景；反过来，1000 盏灯各自照不同区域，每像素只看 1–2 盏，剔除就能解决。

| 计量对象 | 数什么 | 谁在用 | 失效方式 |
|---|---|---|---|
| **灯数** | 场景里几盏 | 美术、资产规范 | **与像素成本脱钩**（LightOpt 表 2：灯数 −74%，重叠只 −23%） |
| **重叠数（ALO）** | 每像素平均几盏 | LightOpt、分块管线的成本模型 | 均值掩盖分布（一像素 30 盏、其余 0 盏，均值仍低） |
| **列表长度** | 每 tile 最大几盏 | clustered / Tiled-Z | 深度切分不当时虚高 |

**结论：预算表里"动态灯光 ≤ N 盏"这句话，缺一个计量对象。** 见 [[Budget Units]] 今日补记。

## Prerequisites

- [[Rendering Equation]] — 多光源求和就是方程里积分项的离散化
- [[BRDF]] — "一盏算多贵"的那一半
- [[Real-Time Rendering]] — 预算意识

## Historical Evolution

```text
1994  Ward            按最大贡献排序，逐盏测可见性直到误差达标（绝对界 ⇒ 线性）
1997  Keller          Instant Radiosity：间接光 = 几千~几百万盏虚拟点光（问题被放大）
1998  Paquette        光树（无阴影）
2005  ★ Lightcuts     树 + 切 + 每簇误差上界 + 2% 感知阈值 ⇒ 次线性
2007  MRCS            全图共享一个 cut（放弃逐像素自适应，换并行/相干）
2011  LightSlice      按 tile 切（粒度回到空间）
2014  Dachsbacher STAR 把这条线写成综述，"many-light methods"成为固定名词
2018  Conty Estevez & Kulla  树 → 重要性采样（目的转换：少算 → 少抽）
2020  ReSTIR          复用（时域 + 空间），另一条语法
2025  MegaLights      每 tile 可见灯表 + 随机直接光（游戏侧当前形态）
2026  ★ LightOpt      反方向：不改查询，改资产
```

**两个转折点值得记**：

1. **2018 年的"目的转换"**：同一棵光树，早先是**近似**（代表灯逼近一簇），后来是**采样**（按簇分布抽一盏）。**看到"tree"要先问是哪一种。**
2. **2026 年的"方向反转"**：前三十年都在"给定灯集合，把查询做便宜"；LightOpt 第一次认真做"**把灯集合本身做小**"。**这不是算法的进步，是把优化对象从运行时挪到了资产。**

## Important Papers

- [[Walter 2005 — Lightcuts A Scalable Approach to Illumination|Walter 2005 — Lightcuts]]（A 经典·历史重要性 5）：次线性 + 可算误差界 + 2% 感知阈值
- [[LightOpt — Lights Optimization for Real-Time Rendering|Chen 2026 — LightOpt]]（S）：改写灯架，ALO 与灯数脱钩
- [[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)|Bitterli 2020 — ReSTIR DI]]（A 经典）：复用语法
- [[Lin — Generalized Resampled Importance Sampling Foundations of ReSTIR (2022)|Lin 2022 — GRIS]]：重采样的统一理论
- [[Keller — Instant Radiosity (1997)|Keller 1997]]：把"灯多"变成常态的那一篇

## Related Concepts

- [[ReSTIR]] — 家族 ③ 的核心
- [[Resampled Importance Sampling]] — ③ 的数学底座
- [[Neural Global Illumination]] — 另一个方向的"少算"（学一个函数代替求和）
- [[Scalability and Quality Tiers]] — 档位 = 误差容限，而不是"另一套灯"
- [[Budget Units]] — 计价对象的清单（ALO 是候选的第 10 种）

## Technologies

- **MegaLights**（UE 5.6+）— ③ 的产品化：每 tile 可见灯表 + 随机直接光
- **RTXDI**（NVIDIA SDK）— ③ 的参考实现（`Reservoir.hlsli`）
- **Clustered / Tiled-Z Deferred** — ① 的产品化
- **Lightmap / DDGI / PRT** — 另一条路：**把"多灯"在烘焙期消掉**（见 [[Global Illumination]] 谱系）

## Game Applications

- 城市夜景、赛道灯带、技能光海、emissive 资产密集场景
- **跨平台分档**：同一套灯架 + 不同误差容限（LightOpt 表 3：相对加速 1.5–1.7×，绝对节省在低端机差 100 倍）
- **资产入库校验**：精简灯架 → 报"灯数 / 峰值 ALO / 误差"三个数

## Personal Knowledge

- **Normal（结论层）**：四个家族各自动哪个因子、近似 vs 采样的验收差别、灯数 ≠ 重叠数。**这三条足以在评审里定位"我们的多光源问题属于哪一类"。**
- **Hard**：Lightcuts §4.1 的误差界推导、GRIS 附录 A、自适应树分裂的概率构造。**按需，不阻塞使用。**

## Learning Gap

- **没有"每 tile 最大列表长度"的实测数据**——本库所有 ALO 都是均值，而管线成本更接近最大值。**缺一篇把分布画出来的工作。**
- **可见性界/可见性可微**：Lightcuts 取 1、LightOpt 取 1，**两代人都绕开的那个洞**。谁把它填上，两边的天花板都会抬高。
- **技能光（时变、闪烁、变色）不在任何一篇的模型里**：ToD 只处理平滑关键帧。**这是游戏侧最实际、文献侧最空的一块。**

## Next Step

1. 拿一个实际场景，**同时**记三个数：灯数 / 峰值 ALO / 每 tile 最大列表长度——**看它们是否同向变化**（这是判断"该用哪个家族"的最小实验）。
2. 若要深入 ②：读 [[Walter 2005 — Lightcuts A Scalable Approach to Illumination|Lightcuts]] 图 3 与图 8（≈25 min）。
3. 若要深入 ④：读 [[LightOpt — Lights Optimization for Real-Time Rendering|LightOpt]] 表 2 与图 5（≈40 min）。
