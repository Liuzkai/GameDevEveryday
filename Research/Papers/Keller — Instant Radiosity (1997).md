---
type: paper
title: "Instant Radiosity"
authors: [Alexander Keller]
year: 1997
published: "1997-08-03（SIGGRAPH '97, pp. 49–56；本文逐节核对用 TU Kaiserslautern 技术报告 287/97 版，1997-01）"
venue: "SIGGRAPH 1997（Proceedings of the 24th annual conference on Computer graphics and interactive techniques）"
url: "https://doi.org/10.1145/258734.258769"
code: ""
project_page: ""
category: [global-illumination, radiosity, quasi-monte-carlo, many-lights, vpl, classical]
importance: A（经典）
historical_importance: 5
game_relevance: 4
production_readiness: "Industry Adopted（思想层：VPL / '把光路顶点变成灯' 是全行业渲染器与后续实时 many-lights 线的公共祖先）"
user_level: "Normal"
status: unread
aliases: [Instant Radiosity, IR 1997, VPL, virtual point light, 虚拟点光源, 即时辐射度, Keller 1997]
tags: [gi, radiosity, vpl, many-lights, quasi-monte-carlo]
---

# Instant Radiosity（Keller 1997）

## TL;DR

**VPL（虚拟点光源）的起源论文。** 一句话机制：**把光路的每个顶点变成一盏"虚拟点光源"，然后让图形硬件为每盏灯渲染一遍带阴影的画面，最后在累加缓冲里加权求和**——"几秒出图"（1997 硬件），无需任何 kernel 或解的离散化（没有 form factor 矩阵、没有 meshing、没有伪影）。两个技术支点：**quasi-random walk**（Halton 低差异序列驱动的确定性粒子输运，无方差）+ 本文首创的 **jittered low-discrepancy sampling**（低差异点为骨架、抖动去走样——把 Monte Carlo 与 quasi-Monte Carlo 两个世界接起来）。

> **一句话定位**：库内 [[Global Illumination]] 谱系表第 6 行（缓存族）**"VPL"侧的源头**；与 [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]] 合读 = "缓存族"的两种记账（**随机光路 vs 结构化像素**）。从它长出的树：实时 many-lights（RSM → Splatting → ISM → …）与离线渲染器的虚拟光源族。

## Problem（1997 的上下文）

辐射度方法的经典问题（原文开篇逐条）：

- 经典辐射度（Galerkin 系）要把方程 kernel **投影到有限基**上 → **form factor 矩阵 O(n²)**、需存 kernel 与解的离散化、**meshing 伪影**；
- Monte Carlo 粒子输运不投影 kernel（只投影解），但有**方差**；
- 双向路径追踪连解的离散化都避开了（Veach & Guibas 1994），但**离实时太远**；
- 硬件则已经能实时渲染带阴影的纹理场景（Heidmann 1991；Segal et al. 1992）。

**本文的落点**：把"确定性粒子模拟（1996 年自己的 quasi-Monte Carlo radiosity）+ 硬件光照能力"结合起来——**既不投影 kernel、也不投影解**，直接得到"又快、又稳、又简单实现"的程序。

## Core Idea（两步算法）

```text
第 1 步（粒子生成 / 离线，CPU）：
  从光源出发做 quasi-random walk——N 条路径，光束在命中点被 ρ 衰减（分数吸收），
  持续 ⌊ρ·N⌋、⌊ρ²·N⌋ … 条路径
  → 路径的所有顶点 {(L_i, P_i)} = M 盏虚拟点光源（VPL），M ≤ N/(1−ρ)（线性于 N）

第 2 步（硬件渲染 / 每盏灯一趟）：
  对每盏 VPL 调用标准硬件光照（带阴影），得到一幅图；
  在 accumulation buffer 中按权重 1/N 累加
  → 所有图的叠加 = 漫反射全局光照解
```

**核心近似（原文公式 (2)）**：漫反射辐射度用离散密度逼近 $L(y) \approx \sum_{i=0}^{M-1} L_i \delta(y - P_i)$——"**光照场 = 一组点光源的集合**"。渲染算子 $T_{mn}$ 作用于每个点光源 = 一次硬件光照 pass。

**四个机制细节**：

1. **分数吸收替代俄罗斯轮盘**：路径数按 ρ 几何衰减（ρ^k·N）；"因为真实场景的漫反射率与均值偏差不大"——**每幅图的重要性依次递减但都被 1/N 加权，不会有一盏灯主导全图**；
2. **Quasi-random walk**：Halton 序列（radical inverse，基为素数）+ 用序列前两维做光源表面等距映射——**确定性、无方差、收敛更平滑**（原文引用自己 1996 的数值证据）；
3. **jittered low-discrepancy sampling（本文新概念）**：低差异点在栅格里被随机抖动 → **高频走样变成噪声、低频正确再现**（逼近 Poisson disk 性质）；同时用于抗锯齿与路径抖动；N-rooks 是它的特例；
4. **镜面扩展**：粒子按 BRDF 随机判定 specular/diffuse；命中平面镜面时**镜像生成"虚拟光源"**（虚光源照亮反射面金字塔区域；仅限平面多边形）。

## Key Data（原文数字）

| 项目 | 数据 |
|---|---|
| 规模样例 | N = 128 路径 → **296 幅图**（PAL 720×576）；402 个四边形场景 |
| 时间 | **24 秒**（Silicon Graphics Onyx，Reality Engine 2，75MHz R8000，Heidmann 阴影法；原文注：换 Segal 等人的阴影技术"至少快一倍"）|
| 复杂场景 | 会议室 **39,584 个图元、248 个光源（VPL）、N=128** |
| 复杂度 | **O(N·K)**（N = 路径数、K = 场景元素数）；VPL 数 M ≤ mean-path-length × N（平均路径长 1–10） |
| 实时变体 | 滑动窗口保留最近 N 条路径的图（隐式时间抗锯齿）；"每时间步预计只生成 l 幅图"——允许实时帧率的条件 |

**原文自陈的两个低采样率问题**（诚实记账）：① **弱奇异**——VPL 与受光点距离趋近 0 时画面值过调、被 clip 到帧缓冲上限；② 单盏灯的色彩对全图影响大——但被 1/N 权重压到"基本不可感知"。

## Limitations

- **漫反射主体**：镜面只支持平面虚拟光源的"窄门"（非平面/复杂镜面要另走光追）；
- 确定性版本只出**静帧**；实时变体必须放弃分数吸收（方差回归 + 需要伪随机吸收）；
- **成本单位是"每盏 VPL 一遍渲染"**——这正是 8 年后 [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]] 原文引用的那句：*"With Instant Radiosity, dynamic objects and lights are possible, but many rendering passes are required."*；
- 纹理分辨率过低会出伪影（渲染进纹理的实时变体）。

## Historical Context & Technology Evolution

```text
1986  Kajiya：渲染方程（被本文直接求解的对象）———— [[Kajiya — The Rendering Equation (1986)]]
1990  Arvo & Kirk：粒子输运（MC 侧的先行）  [记名]
1990  Haeberli & Akeley：accumulation buffer（"累加"的硬件形态）  [记名]
1991  Heidmann：Real Shadows – Real Time（硬件阴影 pass；本文实测用它）  [记名]
1992  Segal et al.：FFD 纹理映射阴影  [记名]
1993–94  Lafortune & Willems / Veach & Guibas：双向路径追踪（"不离散化解"的另一条路）
1995–96  Keller：准 Monte Carlo 辐射度（本文的确定性粒子前身）
★ 1997 本文：quasi-random walk + 硬件逐灯 pass + 累加 = "几秒 GI"；VPL 概念诞生
        ↓ （致谢里感谢 Marc Stamminger 提供 Reality Engine 2）
2003  Dachsbacher & Stamminger：Translucent Shadow Maps（VPL 思想进 shadow map 像素）
2005  Dachsbacher & Stamminger：Reflective Shadow Maps——"多趟"压成"一趟固定样本 gather"
2005+ Lightcuts / 各类 many-lights 以 VPL 集为对象做聚类与降维  [记名]
2008  Ritschel et al.：Imperfect Shadow Maps（海量 VPL 的阴影代理解法）  [记名]
2021  Neural Radiance Caching / 2026 AMD attention GI——"缓存族的神经续命"
2026  MegaLights：每像素固定光照采样预算——"光照复杂度预算"的当代工程形态
```

## Relationships

### Based On

- [[Kajiya — The Rendering Equation (1986)]] —— 求解对象；
- **Keller 1996（Quasi-Monte Carlo Radiosity）**（记名）——确定性粒子模拟的前身，本文把它与硬件连起来。

### Extends / Contrasts

- **Contrasts**：经典辐射度（form factor 矩阵）← 本文：**不投影 kernel 也不投影解**；
- **Extends**：与 [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]] 并列为缓存族的两种记账（**VPL 侧源头 vs RSM 侧源头**，对照表见 RSM 笔记）。

### Followed By

- [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]] —— 直接引用并改良"多趟"为"固定样本 gather"；
- [[Lightweight Attention-based Indirect Illumination (AMD)]]（2026）——神经 GI 的输入之一就是 VPL/RSM 结构（iVPL / pixel-light 编码器）；
- 离线渲染器中的"虚拟光源"族（所有现代 production renderer 的 many-lights 能力均与这条线同源）——**"光路顶点可以当灯用"是 1997 年立的规矩**。

## Why It Works（本库读法）

1. **"解的表示"决定了成本结构**：辐射度选"面片间的矩阵"，就背上了 O(n²) 与 meshing；选"点光源集合"，就把问题变成了"硬件已经会做的事（多光源带阴影渲染）"——**同一问题的成本函数由表示的选择决定**（与 Zinke-Yuksel"方差可加"、PRB"重放换存储"同族）；
2. **确定性采样的工程红利**：无方差 = 不闪、可复现、误差有界——**"确定性"在 1997 是抗噪手段，在 2026 是多人一致性的地基**（你的开放世界业务里"确定性"以另一副面孔出现）；
3. **权重 1/N 的均衡术**：单盏灯影响被结构性压制（≤1/N），使"弱奇异"等局部病态不致命——**用权重结构吸收病理个案**，很工程。

## Game Development Relevance

- **"大量小光源"的原始范式**：粒子光 / 特效光 / 间接光斑在图形学上同构——理解 VPL 的"每灯一遍 vs 固定预算"两代成本语言，直接服务于"动态灯光维度"与 MegaLights 的评估（RSM 笔记已给出与 1978 账本的衔接）；
- **与 NGR 的间接接口**：Hybrid GI / APV 等烘焙体系下，"一个 bounce 的实时补充分量"常常正是 RSM/VPL 家族的现代形态——**当你的场景需要"间接光也随动态光源动"时，账本要回到这篇的原始提问**；
- **确定性 vs 随机**：quasi-random（确定性低差异）与 pseudo-random（伪随机）的分工（本文 jittered LSD 同时用了两者）——**与《巫师 3》三 build 切换、多人一致性等问题的底层词汇相通**。

## Unreal Engine Relevance

- 无直接 UE 实现；它是**概念史**不是工具史；
- 与 UE 的接口在词汇层：`Lightmass`（静态 GI 的辐射度后裔）/ Lumen（另一支）/ MegaLights（"多光源"的当代形态）——**"多光源怎么记账"的三个时代在 UE 里同框**；
- 若给"移动档一个 bounce"做技术选型：IR/RSM（2005 级）→ Lumen Lite（2026 级）之间其实是一条连续谱。

## Personal Knowledge State

- `user_level: Normal（结论层）`——读法：**"两步算法 + 一句台词"**（粒子走 → 顶点变灯 → 逐灯渲染累加；"光照场 = 一组点光源"）；
- 前置链条：[[Global Illumination]]（谱系表）→ 本篇 → [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]]；
- **桥的连接点**：[[Neural Global Illumination]] 的"经典部分"（RSM/VPL/iVPL）至此**两个源头齐**——Hard → Normal 桥的具名前置闭合。

## Learning Value

**四条可迁移抽象**：

1. **"把无限的量换成有限集合"**（间接光 → M 盏 VPL）——缓存族的共同台词；判据：*这个无限积分/无限集合，能否用一个"代表性样本集合+权重"换掉？*（与"一条原型路径代表全部路径"（毛发）、"一条线性基代表全部"（GALA 2026）跨域同构）；
2. **"用确定性换方差"**：低差异序列 = 更平滑收敛、可复现、误差有界——**"确定性"是一种可购买的质量**（价格是放弃随机性带来的某些统计便利）；
3. **"让硬件做它会做的事"**：VPL 的聪明处不在数学而在**选择问题形态**——"多光源带阴影渲染"是 1990 年代硬件唯一快的事，所以把 GI 的解表示成它（与 RSM"信息已在 shadow map 里"的转向同源：**先问硬件/结构能免费给你什么**）；
4. **"每个近似都要有主导权重结构"**：1/N 权重 + 分数吸收的组合，使病态个案（弱奇异）不致命——**结构性兜底 > 逐案打补丁**。

## Visualization

![[间接光缓存族_RSM 2005 与 Instant Radiosity 1997 双源图解.html]]

## Notes

- **来源核对**：TU Kaiserslautern 机构库（KLUEDO）的 **Interner Bericht 287/97（1997-01）** 全文 PDF（pypdf 提取 11 页逐节核对）——为 SIGGRAPH '97 论文（pp. 49–56）的技术报告版，正文内容一致；页码/DOI 经 CrossRef 核实（DOI: 10.1145/258734.258769）。
- 两个版本的措辞差异：技报版摘要"Rendering rates of a few seconds"，与 SIGGRAPH 版一致（原文"few seconds"指整幅图，非交互帧率——**"instant"在 1997 指"几秒"**，对照 2005 RSM 的 4.2–27.5 fps）。
- **历史彩蛋**（与 RSM 互见）：本文致谢 *"Marc Stamminger, providing access to the Reality Engine 2"*——8 年后 Stamminger 成为 [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]] 的共同作者；两篇论文之间有一条**人**的连线。
- 入库日：2026-10-03（Run #25）；[[Global Illumination]] 谱系表第 6 行"VPL"自此从"记名待入库"改为 ✅。
