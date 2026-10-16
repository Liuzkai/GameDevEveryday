---
type: paper
title: "ReSTIR GI: Path Resampling for Real-Time Path Tracing"
authors: [Yaobin Ouyang, Shiqiu Liu, Markus Kettunen, Matt Pharr, Jacopo Pantaleoni]
year: 2021
published: 2021-08
venue: "Computer Graphics Forum 40(8) (Proceedings of HPG 2021), pp. 17-29"
url: "https://doi.org/10.1111/cgf.14378"
code: ""
project_page: "https://research.nvidia.com/publication/2021-06_restir-gi-path-resampling-real-time-path-tracing"
category: [rendering, ray-tracing, global-illumination, sampling, real-time]
importance: A
historical_importance: 4
game_relevance: 5
production_readiness: Early Production
user_level: Normal
status: unread
aliases: [ReSTIR GI]
---

# Ouyang — ReSTIR GI Path Resampling for Real-Time Path Tracing (2021)

## TL;DR

把 ReSTIR 从"直接光"推到"间接光"的第一篇完整方案：**候选不再是"光源上的点"，而是"从可见点出发 traced 一跳后打到的表面点"（sample point）**，权重 = 该点散射回来的 radiance。三缓冲（初始样本 / 时间蓄水池 / 空间蓄水池）全在屏幕空间，显存固定与场景尺寸无关。代价是两件 DI 没有的新账：**空间复用要做 Jacobian 概率空间变换**（方向采样的源 PDF 依赖每像素局部几何），**时域复用要加样本验证**（存储的 Lo 会过期）。1080p 3090 上 UE4 实现每帧 8–18 ms，无降噪 MSE 比路径追踪改善 **9.3×–166×**（无偏版）。

## Problem

- ReSTIR DI（2020）只解决直接光：候选采样在"光源表面"，目标函数只用局部 BSDF×cos×可见性。
- 间接光的积分域高维且复杂：BSDF 采样在"离峰间接光"（强光集中在少数方向，如门缝阳光）下失效；path guiding 的世界空间数据结构（八叉树/GMM/神经引导）在 GPU 上并行构建开销大，不适合实时。
- 目标：一种**GPU 友好、显存固定、无世界空间结构**的间接光采样算法。

## Historical Context

```text
1986  Kajiya 渲染方程
1997  VPL / many-light（光子图→虚拟点光源复用，有偏难控）
2002  Bekaert et al. 子路径顶点重连接（本文重连接策略的直接先祖）
2005  Talbot RIS（理论前置）
2017  Müller path guiding（八叉树×四叉树 SD-tree，离线主流）
2020  ReSTIR DI（直接光，SIGGRAPH Talks）
2021  ★本文 ReSTIR GI（间接光，HPG）+ 同会 Wyman-Panteleev "Rearchitecting
      Spatiotemporal Resampling for Production"（→ RTXDI SDK 架构）
2022  GRIS（Lin et al.，统一理论）
2025  MegaLights 进 UE 5.6（家族产品化高峰）
```

## Previous Work

- **VPL 系**（Keller 1997 / many-light / BDPT 变体）：从光源采子路径、顶点当虚拟光源——有偏、亮斑难控；
- **Bekaert 2002**：从相机侧子路径复用顶点的重连接——本文"几乎相同的顶点重连接策略"的直接源头，差别在 GPU 化与 RIS/蓄水池权重机制；
- **Path guiding**（Lafortune 1995 / photon map / Vorba-Herholz GMM / Müller SD-tree / 神经引导）：误差降低可观，但数据结构的并行构建与更新在实时预算内做不动；
- **降噪器路线**（SVGF 等）：在 1spp 输入上重建——本文与其**正交且互补**（图 12：默认时空降噪器下 ReSTIR GI 输入显著更干净，接触阴影细节保留）。

## Core Idea

**把"样本"从光源点改成表面点，把 RIS 的目标函数从"局部着色"改成"样本点散射回来的 radiance"**：

- 每像素的 visible point（G-buffer 可见点）按源 PDF 抽一个方向 → trace 得 sample point（样本点）；
- 在样本点用 NEE+MIS 路径追踪估 **Lo(x_s→可见点方向)** —— 存进 SAMPLE 结构（图 3：可见点几何 + 样本点几何 + Lo + 复用随机数）；
- 目标函数故意用最简版 **p̂ = Lo(x_s, −ω_i)**（式 10），**不乘本地 BSDF×cos**（式 9）——作者实测"对单像素次优，但对空间复用更稳"：它保住了"对邻居也有用"的样本；
- 之后与 ReSTIR DI 完全同构：时间蓄水池 → 空间蓄水池 → RIS 估计器 L̂ = f(z)·W(z)。

在样本点的路径追踪跑 n 跳 → 得 n+1 跳 GI（图 5）：**重连接 = 邻居像素直接复用整条路径的贡献**。

## Technical Approach

三个屏幕空间缓冲（图 4）：初始样本 / 时间蓄水池 / 空间蓄水池。

![[Research/Files/Pictures/Ouyang 2021 — 图4 算法工作流：三缓冲与时空复用.png]]
**读图**：三个球体示意"从可见点（红点）抽方向打出的样本点"，条带表示样本有效性（黄=有效、×=无效）；下方三个网格是屏幕空间三缓冲——初始样本（暗、稀疏）→ 时间蓄水池（混合上一帧）→ 空间蓄水池（融合邻居，亮度收敛）。出处：图 4，Ouyang et al. 2021（HPG）。

1. **初始采样**（Alg. 2）：G-buffer → 抽方向 → trace → 样本点 NEE+MIS 估 Lo。1 跳 NEE = 1-bounce GI；
2. **时间复用**（Alg. 3）：motion vector 反投影上一帧蓄水池，WRS 随机合并；失败则重置双蓄水池；
3. **空间复用**（Alg. 4）：随机邻居的时间蓄水池合并入空间蓄水池——**只复用时间蓄水池、不复用空间蓄水池**（图 10），偏置不跨轮累积；样本数过低的蓄水池才允许复用邻居的空间蓄水池提收敛（搜索半径自适应减半，最小 3px）；
4. **着色**：f（本地 BSDF×cos）× 空间蓄水池的 W。

![[Research/Files/Pictures/Ouyang 2021 — 图2 ReSTIR 与 ReSTIR GI 机制对比.png]]
**读图**：(a)(b) 原版 ReSTIR——候选打在光源（橙条）上，无效样本被丢弃，好样本跨像素/跨帧共享；(c)(d) ReSTIR GI——候选是方向 trace 打到的场景表面点（×），复用时把"样本点散射的 radiance"带回新像素，2-bounce 样本也能复用。出处：图 2，Ouyang et al. 2021（HPG）。

### 相对 ReSTIR DI 的两个理论增量

**① Jacobian 修正（式 11）**——DI 直接采样光源，源 PDF 与像素局部几何无关；GI 用方向采样（uniform/cosine/BSDF），**同一份面积度量 PDF 在不同像素的立体角空间里形状不同**。把像素 q 的样本复用到像素 r 时，PDF 必须除以 |J_{q→r}|（重连接 Jacobian）。计算开销极小，"没有理由不加"——图 7 顶行（不加）地面出现照明不连续 + 墙脚过亮，中行（加）与底行参考一致。

![[Research/Files/Pictures/Ouyang 2021 — 图7 Jacobian 修正前中后对比.png]]
**读图**：阳光打亮墙面、间接照亮地板。上：忽略 Jacobian——地板出现照明断裂、墙脚过亮；中：加 Jacobian——伪影消失；下：路径追踪参考。出处：图 7，Ouyang et al. 2021（HPG）。

**② 样本验证（A-SVGF 启发）**——存储的 Lo 是"快照"，光源/几何一动就过期，且蓄水池偏好保留亮样本数帧 → 间接光更新滞后。每几帧用**与初采相同的随机数**重 trace 重算 Lo，超出容差即清空蓄水池；遮挡变化则补阴影射线。验证间隔可按场景动态程度调整。

![[Research/Files/Pictures/Ouyang 2021 — 图11 样本验证抑制动态光照偏置.png]]
**读图**：阳光突变后 12 帧。左上：原始光照；右上：无样本验证——左墙残留过期照明；左下：有样本验证——迅速适应新光照；右下：收敛结果。出处：图 11，Ouyang et al. 2021（HPG）。

**无偏化**：空间复用的偏置来自"邻居源 PDF 覆盖不到本像素的定义域"——完整修法是 trace 可见性射线做 |Z| 权重计数（Alg. 4 第 16–19 行，与 DI 的 MIS 修正同思路）；廉价替代是几何相似度拒绝（深度/法线）。

## Key Contribution

1. **ReSTIR 家族从 DI 到 GI 的关键一步**——证明"蓄水池时空复用"框架对高维间接光同样成立；
2. **Jacobian 修正**：指出方向采样系 ReSTIR（GI/后续 GRIS）与光源采样系（DI）的本质差异，给出 O(1) 修正；
3. **样本验证机制**：为"复用历史快照"补上失效检测——与 SVGF 历史长度回退、DI 的 M clamp 同属"复用必须自带失效条件"设计律；
4. UE4 与 Falcor 双实现（本文在**引擎内**落地，非仅测试床）。

## Why It Works

- **方差来源被搬家**：难找亮路径（低概率大贡献）→ 复用把邻居找到的好路径"借"过来，等效样本数 kⁿM；
- **目标函数 p̂=Lo 的稳健性**：不绑定本地 BSDF，样本跨像素通用性好——"为邻居保留样本"是空间复用区别于逐像素优化的关键取舍；
- **显存 O(像素) 而非 O(场景)**：48 B/蓄水池（法线压 4B + 半精度 Lo），1080p 三缓冲共 475 MB；每帧 ~570 MB 读 + 285 MB 写。

## Limitations

- **预算**：8–18 ms 对 30fps 游戏（GI 预算 <2ms）仍贵——作者给出的降档路线：降蓄水池分辨率 + 深度缓冲几何法线 + SH 记录低频光照再用法线恢复细节（已是后续实时 GI 的标准配方）；
- **Lambertian 假设**：实现里样本点的散射 radiance 被当作各方向相同（省掉方向变化），样本点非漫反射时引入误差——**高光泽多跳 GI 基本失效**（复用时样本点贡献落不到邻居的 specular 瓣内）；
- **屏幕空间表示**：快速运镜的新现像素采样不足 → 高噪；纯镜面可见时 visible point 需跳到第一非镜面点，邻居的可见点在场景里不再相邻，空间复用退化；
- 偏置集中在**阴影边缘**（跨阴影复用）+ glossy 地板（目标函数含 specular 分量时样本聚集在反射方向，恰逢复杂可见性，图 9）。

## Game Development Relevance

- **UE5 Lumen 之前夜**：本文在 UE4 内验证了"屏幕空间缓冲 + 光追样本复用"路线；其后 Lumen final gather 与 5.6 MegaLights 的"复用派"血统即在此；
- **预算方法论三个可搬走的具体数字**：48 B/蓄水池（存储预算）、25% 像素跑多跳（把最贵的算力只花在部分像素——与 Talbot 等时间准则同族的"预算劈分"）、验证间隔随动态性调整（失效检测本身的成本也要进预算）；
- **"复用必须自带失效条件"第三例**：DI 的 M clamp（陈旧权重）→ SVGF 历史回退 → 本文样本验证（快照重算 + 同随机数对照）。设计律在"复用 Lo 快照"这种更强形式的复用上升级为**主动重验证**。

## Unreal Engine Relevance

- UE4 全屏 pass 实现（光栅化 G-buffer + 两个全屏复用 pass），双缓冲引入一帧延迟（高帧率下无感）；
- 对 NGR 淬炼系统的映射：技能 VFX 若带自发光/动态灯，其间接光贡献目前靠 Lumen——理解本文才知道"为什么动态灯越多 Lumen 越要降档"（复用假设被高频变化打破 → 验证/重置频率上升 → 预算被失效检测吃掉）。

## Technology Evolution

```text
Bekaert 2002（顶点重连接）
  ↓ + Talbot RIS + 蓄水池
ReSTIR DI 2020（直接光；光源域候选）
  ↓ 候选改成表面点 + Jacobian + 样本验证
★本文 2021（间接光；方向域候选）
  ↓ 统一理论
GRIS 2022（一般化重采样；DI/GI 同一定理）
  ↓ 工程化
RTXDI SDK（DI/GI/PT/ReGIR 模式）
  ↓ 产品化
MegaLights（UE 5.6）/ 读者框架四阶段
```

## Relationships

### Based On

- [[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)|ReSTIR DI 2020]] —— 框架直接上游（RIS+WRS+时空复用）
- [[Talbot — Importance Resampling for Global Illumination (2005)|Talbot RIS 2005]] —— RIS 估计器与校正因子
- Bekaert et al. 2002 —— 顶点重连接策略的先祖（未入库）

### Extends

- 把 ReSTIR 的候选域从"光源表面"推广到"方向 trace 的表面点"——**间接光版**

### Related

- [[Schied — Spatiotemporal Variance-Guided Filtering (2017)|SVGF 2017]] —— 正交互补（重建端）；本文样本验证机制受 A-SVGF 启发
- [[ReSTIR]] —— 方法族概念锚点
- [[Global Illumination]] —— GI 八代谱系的"光追/复用"代成员

### Followed By

- GRIS 2022（统一理论）/ RTXDI（SDK 化）/ MegaLights（UE 5.6 产品线）

## Personal Knowledge State

- **Normal**：结论层可读（三缓冲数据流 / 两个增量的直觉 / 预算数字）；推导层（Jacobian 行列式的完整推导、|Z| 无偏化证明）后置。

## Learning Value

- 读后应能回答：① 为什么样本点存的不是"颜色"而是 Lo（答：复用发生在采样概率层，W 才是颜色系数——"过滤 PDF 而非颜色"的 GI 版）；② 为什么 GI 需要 Jacobian 而 DI 不需要（答：源 PDF 是否依赖像素局部几何）；③ 样本验证为什么必须用与初采相同的随机数（答：否则 Lo 差异来自采样噪声而非场景变化，容差失效）；④ 为什么空间蓄水池只复用时间蓄水池（答：偏置不跨轮累积）。

## Visualization

![[Research/Files/Pictures/Ouyang 2021 — 图1 ReSTIR GI 等时间对比（1080p 3090 无降噪）.png]]
**读图**：两跳 GI 等时间对比。左：路径追踪 1spp（8.0 ms）——间接光几乎全黑（阳光集中在小区域，找路极难）；中：ReSTIR GI 1spp（8.9 ms）——MSE 改善 15.1×；右：参考图。出处：图 1，Ouyang et al. 2021（HPG）。

![[Research/Files/Pictures/Ouyang 2021 — 图10 数据流：空间蓄水池只复用时间蓄水池.png]]
**读图**：每帧数据流——各像素新样本进自己的时间蓄水池；空间蓄水池**只**吸收邻居的时间蓄水池（橙色箭头），偏置不跨轮累积；仅当输入样本数过低时才复用邻居的空间蓄水池（灰色箭头，右上注释）。出处：图 10，Ouyang et al. 2021（HPG）。

（图 2 / 图 4 / 图 7 / 图 11 见正文对应节。）

## Notes

- 与 DI 互补关系的一个细节：本文目标函数有两种（式 9 含本地 BSDF×cos、式 10 只用 Lo），实测式 10 更稳但方差略高——**"目标函数为复用服务，不为单像素最优服务"**；
- 论文自评与 path guiding 的对比口径：guiding 建结构贵、复用便宜但偏置要管理——两者后来在 RTXDI 的 ReGIR（光源侧蓄水池+结构）里部分合流；
- 1080p 475 MB 的三缓冲（含双缓冲）在 2026 年看仍是"复用派"的典型显存账——对照 [[Real-Time VFX Performance Budgeting]]：**复用机制的存储/带宽账要单列**。
