---
type: paper
title: "Practical Real-Time Hair Rendering and Shading"
authors: [Thorsten Scheuermann]
year: 2004
published: "2004 (SIGGRAPH 2004 Sketches, p.147)"
venue: "ACM SIGGRAPH 2004 Sketches (Talks) / ATI Research"
url: "https://doi.org/10.1145/1186223.1186408"
code: ""
project_page: "https://history.siggraph.org/wp-content/uploads/2022/12/2004-Talks-Scheuermann_Practical-Real-Time-Hair-Rendering-and-Shading.pdf"
category: [hair, rendering, real-time, classical, production]
importance: A
historical_importance: 4
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal
status: unread
aliases: [Scheuermann 2004, 实时毛发工程三刀]
tags: [hair, classic, rendering, real-time, sorting, early-z, production]
---

# Practical Real-Time Hair Rendering and Shading (Scheuermann 2004)

## TL;DR

**这是"游戏毛发为什么长这样"的答案文档**——2004 年，把 2003 年的物理模型（[[Marschner — Light Scattering from Human Hair Fibers (2003)|Marschner R/TT/TRT]]）第一次塞进实时渲染器，靠**三刀**：

1. **模型刀**：连续发丝 → **分层 2D 多边形发片**（layered patches）+ 每顶点 AO。理由：顶点负载低、排序问题简化、**贴美术管线**——今天游戏里"发片"成为默认表示，源头在这里；
2. **着色刀**：**两个高光项近似 Marschner 的双高光**——沿发丝**反向移位**、颜色/指数各不同，移位用**扰动切线**实现：`T' = normalize(T + s×N)`（s 存贴图）；次级高光乘噪声贴图 → 廉价的"闪烁感"。漫反射 `max(0, 0.75×N·L + 0.25)`（缩放偏置，避免背光死黑）；
3. **排序刀**：**取消运行时 CPU 空间排序**——预处理把发片按"距头距离"排好、存**静态索引缓冲**，运行时用**四趟渲染**逼近 back-to-front。附带一个漂亮权衡：第一趟 prime-Z 的存在是为了**避免 alpha-test 禁用 early-Z**（"多渲染一趟的成本被随后三趟 early-Z 的收益盖过"）。

全文**只有两篇参考文献**——[[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]]（它继承的经验模型）与 [[Marschner — Light Scattering from Human Hair Fibers (2003)]]（它近似的物理目标）。**这篇就是那两者之间那座工程桥。**

## Problem

2004 年的实时毛发面对三个"不可能"：

1. **发丝几何太贵**：几千根曲线几何在当时硬件上不可行（顶点负载 + 排序成本）；
2. **半透明必须排序**：发片是大量半透明层，正确的 alpha 混合需要 back-to-front；**运行时 CPU 空间排序**在每帧预算里不可接受；
3. **物理模型太贵**：Marschner 的双高光 + caustic + 三 lobe 是离线量级。

## Historical Context

- **上下文**：ATI Research 出品；用于同年的**实时动画《Ruby: The Double Cross》**（SIGGRAPH 2004 动画节）。面向当时的 SM2.0 级桌面硬件；
- **三条线的交点**：经验模型（Kajiya-Kay 1989，便宜但不物理）× 物理模型（Marschner 2003，物理但贵）× 实时预算（2004 的帧时间）；
- **后续影响**：它定义的"**发片 + 双高光 + shift map + 固定排序**"成为此后十几年游戏毛发的默认做法（TressFX/HairWorks 之前的时代，以及之后所有低配档的存量方案）。

## Core Idea / Technical Approach

### 1. 模型：分层发片 + 每顶点 AO

- 用**若干层 2D 多边形 patch** 近似毛发的体积感；
- **每顶点 AO**（预处理阶段算好）近似自阴影——**不做真实的自阴影系统**；
- 贴图组：base map（拉伸噪声，发色走 shader 常量）+ 不透明度贴图 + **specular shift 贴图** + specular 噪声贴图；
- 原文理由（三条）：**顶点处理器负载低于线渲染**；**深度排序问题被简化**；**与美术管线天然兼容**。

### 2. 着色：两 lobe 移位近似（本文最被沿用的部分）

| 组件 | 做法 |
|---|---|
| 漫反射 | `diffuse = max(0, 0.75×N·L + 0.25)`——缩放+偏置，**提亮背光侧**，整体更"柔" |
| 主高光（≈R） | 白、尖、**向发梢移位** |
| 次高光（≈TRT） | 受发色调制、宽、**向发根移位**，乘噪声贴图 → "sparkling" |
| 移位机制 | **扰动切线**：`T' = normalize(T + s×N)`，s>0 向梢 / s<0 向根；**s 从贴图查**（"等价于一张 tangent map"——normal map 的毛发版） |

> **一句话**：**用一张贴图控制的"切线偏移量"，把两个圆锥高光搬到 Marschner 观察到的两个位置**——不是仿真物理，而是**把物理观察的视觉结论直接参数化**。

### 3. 排序：静态索引缓冲 + 四趟渲染（"取消式优化"）

- **预处理**：按距头距离（≈inside-out）排序**连通分量（发片）**（不是逐三角形），把绘制顺序写进**静态索引缓冲**；
- **运行时四趟**：

| 趟 | 状态 | 内容 |
|---|---|---|
| ① prime-Z | alpha-test 掩掉透明部分；关背面剔除；**不写颜色**；极简 shader 只输出 alpha | 先给不透明区铺好深度 |
| ② 不透明着色 | **Z-test = equal**；完整毛发 shader | 只给与①相同的像素着色 |
| ③ 背面透明 | 关 Z 写；**正面剔除**；Z-less | 所有背面透明区 |
| ④ 正面透明 | 开 Z 写；**背面剔除**；Z-less | 所有正面透明区 |

- **第①趟存在的真正理由（全文最值得记的一处）**：**避免在后续趟里使用 alpha-test——因为 alpha-test 会禁用 GPU 的 early-Z 剔除**。原文：*"the extra rendering pass is more than offset by the gains of early-Z culling in the following three passes"*。

### 4. 原文明说的前提假设（务必一并记住）

> *"we assume that the hair model only undergoes very moderate animation that doesn't move the hair patches much relative to each other. If this assumption doesn't hold, it is possible to use a more robust CPU-based spatial sorting scheme."*

**静态排序成立的前提 = 发片之间相对运动足够小。** 前提不成立时，回退到它刚省掉的 CPU 排序。

## Key Contribution

1. **首个可量产的实时毛发方案**（发片模型 + 双高光近似 + 免运行时排序），被工业界长期采用；
2. **"取消运行时排序"的早期范式**：把动态问题（排序）转化为静态资源（索引缓冲）+ 固定状态机（四趟渲染）；
3. **一个反直觉的工程权衡样本**：**多花一趟渲染，是为了让一个硬件优化（early-Z）生效**。

## Why It Works

1. **近似的是"观察结论"而不是"物理方程"**：Marschner 观察（双高光、移位方向、闪烁）→ 直接参数化成两个移位的高光项——**比近似方程便宜得多**；
2. **排序的静态化**：单个头部内发片相对位置在温和运动下近似不变 → 一次离线排序可用全片；
3. **状态机换灵活性**：四趟固定趟次 + Z 状态组合，逼近 back-to-front 的混合结果（**用流程换排序**）。

## Limitations

- **依赖"温和动画"假设**（原文自述；剧烈运动中发片相互穿插会露馅）；
- **per-vertex AO ≠ 自阴影**：没有任何真实阴影投射；
- **现象学近似**：两 lobe 不保能量、无 caustic、无真实倾角分布；
- **发片本身的近似**：体积感是"足够远看"的近似，近看会穿帮（正是 2026 年 LSS/发丝化要解决的问题）。

## Game Development Relevance

**5/5 —— 它是你日常场景里"发片毛发"一切做法的正典。**

1. **工程谱系**：今天大量引擎/教程中的"发片 + 双高光 + shift map"可追溯到这里（**具体引擎默认模型的归属仍以官方文档为准，本笔记不做断言**）；
2. **"取消式优化"的第三条样本，而且早了 20 年**：
   | 年份 | 工作 | 取消了什么 |
   |---|---|---|
   | 2004 | **本文** | **运行时 CPU 空间排序 → 静态索引缓冲 + 四趟渲染** |
   | 2026 | [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising\|Stochastic GS Denoising]] | 高斯排序（固定光栅化顺序 + 神经去噪） |
   | 2026 | [[2026-09-23-CuACD — A Fully GPU-Resident Approximate Convex Decomposition\|CuACD]] | kernel 边界与 host 同步 |
3. **新判据（今日最值钱）**：**看到"加一步前置工作/多一趟渲染"的方案，先问它解锁了什么** —— 本文多渲染一趟，解锁了随后三趟的 early-Z；
4. **"前提假设必须写在明面上"**：静态排序的前提（温和动画）是它给你留的回退路径——评估任何预计算方案时，把"它在什么条件下失效"当成规格的一部分；
5. **对 [[Real-Time VFX Performance Budgeting]]**：发片毛发的成本结构 = OverDraw × alpha 混合 × 排序敏感——**透明排序问题是"半透明 VFX"与"毛发"共享的成本项**。

## Unreal Engine Relevance

- UE 的**发片头发**（Masked 材质 + 各向异性高光 + 多层片）与本文同族；**UE Groom 发丝**则是"几何侧"路线——两者在分档表上各占一段，正是"表示跟着观察尺度走"；
- **alpha-test 与 early-Z 的权衡**在当代仍是移动端分档的老问题（不同硬件实现不同，结论以实测为准）：**低配档的材质策略里"少用 alpha-test 换硬件遮挡剔除"这类取舍，就是本文第①趟的当代版本**；
- 与 [[Niagara]]：Groom 物理走 Niagara（几何侧的代价）；发片侧没有物理，**这个差异本身就是分档依据**。

## Technology Evolution

```text
1989 Kajiya-Kay —— texel + sin(t,l) + 圆锥高光（离线）
        ↓
2003 Marschner —— R/TT/TRT 物理模型（影视/离线）
        ↓
★ 2004 Scheuermann —— 工程桥：发片 + 两 lobe 移位近似 + 取消运行时排序（本文）
        ↓
2004-2015 —— 成为游戏默认：发片毛发 shader 谱系（shift map 四处开花）
        ↓
2008+ dual scattering（多次散射近似）/ TressFX / HairWorks / UE Groom —— 发丝线独立发展
        ↓
2026 —— 三条线并行：
        · 资产侧：HairCS 发片→发丝自动升档
        · 画面侧：DLSS 5 神经增强
        · 光追侧：《巫师 3》重制版 LSS 路径追踪毛发
```

## Relationships

### Based On
- [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]] —— 基础着色模型（本文在其上叠加移位高光）

### Approximates / Productionizes
- [[Marschner — Light Scattering from Human Hair Fibers (2003)]] —— 被近似与工程化的物理目标（双高光观察 → 两个移位高光项）

### Related
- [[Hair Rendering]] —— 本笔记补上该概念的**"实时化工程"缺口**（其 Learning Gap #3 的闭合项）
- [[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]] —— "发片"这一表示在 20 年后的自动升级入口
- [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising]] / [[2026-09-23-CuACD — A Fully GPU-Resident Approximate Convex Decomposition]] —— "取消式优化"同族（2026 版）
- [[Real-Time VFX Performance Budgeting]] —— 半透明排序/OverDraw 成本项
- [[Scalability and Quality Tiers]] —— 表示跟着观察尺度走

## Personal Knowledge State

- **user_level: Normal**。判断依据：**"发片便宜""alpha 混合贵"这些结论在你的 Easy 区**；**新的是三处机制细节**——移位高光怎么实现、"不排序"靠什么成立、early-Z 那一刀为什么值。

### Mastery 自测（3 条，与 Marschner 5 + Kajiya-Kay 4 合并 = 毛发线 12 条）

1. **两 lobe 的"移位"具体怎么做？**（答：扰动切线 `T' = normalize(T + s×N)`，s 从贴图查；正负号决定移向发梢/发根。）
2. **"不排序"为什么可行？前提假设是什么？**（答：预处理静态索引缓冲 + 四趟渲染；前提是发片间相对运动足够温和——不成立则回退 CPU 排序。）
3. **为什么多渲染一趟（prime-Z）反而整体更快？**（答：避免后续趟使用 alpha-test——alpha-test 会禁用 early-Z；一趟的成本被三趟的 early-Z 收益盖过。）

## Visualization

![[实时毛发三刀_Scheuermann 2004 图解.html]]

含：三刀各自的做法与理由、移位切线几何（T/T'/N 与两个高光位置）、四趟渲染状态表、以及"2004 取消排序 → 2026 取消排序"的跨越 22 年的对照。

## Notes

- **原文已下载并逐条核对**：SIGGRAPH History Archive 官方 PDF（1 页 sketch 全文）+ 配套 talk slides（23 页，含像素 shader 实现与 Early-Z 讨论）。本笔记所有引语/公式/数字均出自原文：
  - `diffuse = max(0, 0.75×N·L+0.25)`（sketch "Hair shading" 节）
  - `T' = normalize(T+s×N)` 与 shift 贴图（同上）
  - *"Instead of executing a spatial sorting step on the CPU at run-time, we render the opaque and transparent hair regions in separate passes to resolve visibility."*（引言）
  - *"we assume that the hair model only undergoes very moderate animation…"*（局限性节）
  - Early-Z 段落（sketch 末节 + slides p.16）
- 引用信息：ACM SIGGRAPH 2004 Sketches（Talks），DOI 10.1145/1186223.1186408；作者供职于 ATI Research（`thorsten@ati.com`）；
- 入库时机：2026-09-26（Run #18）。触发来源：[[Hair Rendering]] 的 Learning Gap #3"实时化工程文"（挂账自 9-17）与 [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]] 笔记的"下一步"条目——**同日结清**；
- 挖掘路径记录：`history.siggraph.org` 的 SIGGRAPH 2004 Talks 归档直接提供 PDF；slides 版（23 页）来自 Semantic Scholar 镜像——**"老论文先找会议历史归档 + slides"这条路径再次生效**（自 Kulla-Conty 经验）。
