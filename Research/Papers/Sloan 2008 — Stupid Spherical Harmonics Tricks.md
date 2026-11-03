---
type: paper
title: "Stupid Spherical Harmonics (SH) Tricks（节选：从 SH 抽取显式灯光）"
authors: ["Peter-Pike Sloan"]
year: 2008
published: 2008
venue: "GDC 2008 讲座同名伴生文档（作者主页自归档，ppsloan.org；非正式会议论文）"
url: "https://www.ppsloan.org/publications/StupidSH35.pdf"
code: "论文未给（作者主页有其他 SH 示例代码，本节选无）"
project_page: "http://www.ppsloan.org/publications"
category: ["spherical-harmonics", "light-probes", "light-extraction", "game-engine", "approximation"]
importance: A-
historical_importance: 4
game_relevance: 5
production_readiness: "Production（文档自述该技术已用于无顶点着色器的硬件；思想广泛见于引擎实践）"
user_level: Easy
status: unread
---

# Stupid Spherical Harmonics (SH) Tricks（节选：从 SH 抽取显式灯光）

## TL;DR

**"探针/SH 系数 ↔ 显式灯"这条转换链的游戏工业原始文献。** GDC 2008 讲座伴生文档，42 页覆盖 SH 的实践问题全集；本笔记聚焦其中 **Extracting Conventional Lights from SH** 一节（pp. 12–14，StupidSH35 版）：

- **一盏方向光 + 一盏环境光**逼近 SH 光照环境：给定方向 $d$，最优强度有**闭式解**（最小化所有法线上反射辐射的平方误差）；方向 $d$ 用"optimal linear direction"（线性系数构成的向量归一化）；
- **多盏灯**：从静电平衡生成的球面点集出发做梯度上升/BFGS 找 SH 环境的局部峰，峰即灯方向——1–6 阶所需起始点数 **{1, 3, 6, 10, 15, 22}**（按 2/3 保守半径）；
- 为什么要抽灯：**治 ringing**（解析灯没有负波瓣）、**做光泽反射**（只对抽出的灯算高光）、**省阴影贴图**（只对抽出的少数灯开 shadow zbuffer）。

对「灯光迁移」专题：这是 **① 目标域（方向域 SH 而非空间域 irradiance）→ ② 显式灯参数** 的解析路线，与 Gkaravelis 2016 的空间域组合优化正好各占一头；迁移工具若涉及"旧探针数据 → 新显式灯"，这篇是零拟合成本的起点方案。

![[Research/Files/Pictures/Sloan 2008 — 图 用1至3盏方向光近似SH光照环境.png]]
**读图**：用 1/2/3 盏方向光逼近一个 SH 光照环境（上行为原 SH 与抽灯结果的辐照度对比，下行为对应光照环境；十字展开图）。**1 盏方向光只剩主方向的团块，3 盏已能恢复主要光源结构**——对"探针里其实只有个位数主导光源"的游戏场景，抽取 2–3 盏灯往往就够。这也直观说明了为什么"optimal linear direction"在单主光源时有效、多光源时必须爬山找峰。（Sloan 2008，p.14 插图，无图号）

## Problem

引擎里烘焙/预计算的间接光常存为 SH 探针；但下游有些消费方**只吃显式灯**：老式硬件（无顶点着色器）、高光 BRDF（需要灯方向）、阴影（需要逐灯 shadow map）。问题：**给定 SH 光照向量，如何用最少的传统灯（方向光+环境光、或若干点/方向光）逼近它**，使反射辐射误差最小。

## Historical Context

SH 在图形学的普及由 Sloan 本人的 PRT（SIGGRAPH 2002）推动；到 2008 年 SH 已是游戏引擎间接光的事实标准（Half-Life 2 的 ambient cube、各引擎的 light probe）。但当时大量硬件/管线不能直接在 shader 里求值 SH——"从 SH 里把灯抽回来"是真实的工程需求。本文档是 GDC 2008 讲座的伴生文本，定位是**实践者陷阱与技巧集**，不是新理论。

## Previous Work

- SH 基础与 PRT：Sloan et al. 2002（见库内 [[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)]]）；
- "optimal linear direction" 引自文献 [44]（文档内引用，未核具体出处）；
- BFGS 优化引自 [29]；
- 球面均匀点集生成：静电排斥模拟（文档内给出自适应步长规则）。

## Core Idea

**在"辐照度环境图"空间做最小二乘，而不是在辐射度空间。** 把 SH 光照向量与候选灯都卷积归一化 clamped cosine 核变成 irradiance environment map，然后：

1. **单灯+环境光**：误差 $E(c,a)=\int (c\,L_d(s)*H_N + a\,L_a(s)*H_N - L_e(s)*H_N)^2 ds$ 对 $c,a$ 是二次型 → 闭式解（见下）；
2. **多灯**：SH 足够光滑，给定阶数存在"保证梯度上升收敛到任一峰"的有限距离 → 用覆盖半径小于该距离的球面点集做初值，爬山/BFGS 找全部局部峰作为灯方向，强度另行优化。

## Technical Approach

**归一化（p.12）**：若用 [0,1] 光照，尺度因子 $c=1/dot(L,V)$（$T$ 对齐 +Z 时前 6 阶系数：$\frac{1}{2\sqrt\pi},\frac{\sqrt3}{3\sqrt\pi},\frac{\sqrt5}{8\sqrt\pi},0,\frac{-1}{16\sqrt\pi},0$）。解析灯的归一化：锥角 $a$ 的锥灯 $1/\sin^2 a$；**方向光 $16\pi/17$**（不超过 4 阶时；5–6 阶另给）、**环境光 $2/\sqrt\pi$**（StupidSH35 版数字；SH36 版写作 $2\sqrt\pi$，两版表述有差异，**以实际代码验证为准，本笔记标"未核"**）。

**单方向光+环境光的闭式解**（方向 $d$ 固定）：

$$c=\frac{867}{316\pi}\,dot_0(\hat L_d,\hat L_e),\qquad a=\Big(\hat L_e[0]-c\,\frac{8\sqrt\pi}{17}\Big)\frac{\sqrt\pi}{2}$$

其中 $dot_0$ 忽略 DC 项，$\hat L_e[0]$ 是环境 DC 项。方向候选：归一化 $(-\hat L_e|_{线性系数的y}, \hat L_e|_{z}, -\hat L_e|_{x})$（原文排版如此，**分量顺序以原文为准**）。

**多灯抽取（pp.13–14）**：

- 起始点集：点间 $1/d^2$ 斥力、只保留切向分量、净力增大则步长减半/减小则加倍的自适应模拟，解静电平衡；
- 阶数 1–6 所需点数：**{1, 3, 6, 10, 15, 22}**（保守取 2/3 覆盖半径）；
- 峰搜索：少量 BFGS 迭代即收敛（作者经验）；梯度在归一化位置计算并乘归一化函数的 Jacobian（文档给出 $3\times3$ 矩阵显式形式，对角元 $(L^2-x^2)/L^3$、非对角 $-xy/L$ 等，$L=\sqrt{x^2+y^2+z^2}$）。

## Key Contribution

- 把"SH → 显式灯"从拍脑袋（取最大系数方向）升级为**有误差定义的闭式/半解析流程**；
- 明确指出**在 irradiance（卷积余弦后）空间定义误差**，等价于最小化所有法线上的反射辐射误差——这是"目标定义在光照函数上而非图像上"的早期样本；
- 多灯抽取给出**有收敛保证的初值构造**（覆盖半径论证），不是随机多起点。

## Why It Works

- 灯位/方向固定 ⇒ 强度是线性最小二乘（与 Gkaravelis 2016 利用的是同一条线性性，一个在方向域一个在空间域）；
- SH 低阶信号光滑 ⇒ 峰的数目有限、爬山有保证区域 ⇒ 初值点集可以**先验地**算出来，不用碰运气；
- 解析灯无负波瓣 ⇒ 天然规避 SH 投影的 ringing——**抽灯本身就是一种正则化**。

## Limitations

- 只处理**单个探针的方向域**；空间上多探针插值后再抽灯会引入插值误差，文档未讨论；
- 抽出的灯**不携带距离/位置信息**（方向光化），近场光照结构（灯在房间里的具体位置）丢失——对"迁移到显式灯"意味着这条路线只适合环境/间接光分量，主光源仍需空间域方法（Gkaravelis / Lipp 路线）；
- 多灯时"方向爬山 + 强度另优化"的具体联合优化细节未展开（"should be optimized in some fashion"）；
- 版本差异：网上同时流传 StupidSH35/36 等版本，个别公式数字不同（环境光归一化 $2/\sqrt\pi$ vs $2\sqrt\pi$），引用时须指明版本。

## Game Development Relevance

- **这是游戏工业内部的文档**（Microsoft，GDC），技术动机全部来自真实管线：无顶点着色器硬件、逐灯阴影成本、高光需要灯方向；
- 对迁移工具：若旧系统的间接光以探针/SH 形式存在，**每个探针抽 1–3 盏方向光**可以作为"新显式灯体系"的零成本初值，再交给连续拟合精调——与 Schwengerer 的均匀网格初始阵并列的另一种初始化策略，且带误差界意识；
- "抽灯治 ringing"提醒：直接对 SH 做拟合会把投影误差（负波瓣）带进新系统，抽灯再拟合更干净。

## Unreal Engine Relevance

- UE 的 Lightmass 烘焙产物含光照探针（ILC/ Volumetric Lightmap 的 SH）；本方法可用于"烘焙探针 → Lumen/显式灯"的迁移桥；
- 引擎内类似实践：把天空光/IBL 主方向抽成 Directional Light（TrueHDRI Panel 插件"最亮区域分离成 Directional Light"即此思路的产品化，见收集页清单）；
- 注意 UE 的 SH 约定（归一化、/π、yzx 顺序）与本文档的教科书约定不同，落地时按引擎端 GetShaderConstantsFromNormalizedSH 的约定对齐。

## Technology Evolution

PRT（2002，SH 进引擎）→ **本文档（2008，SH 的工程化全集，含抽灯）** → Ambient Dice（Iwanicki & Sloan 2017，二十面体基替代 SH，治 ringing 的另一条路：换基而不是抽灯）→ 引擎 IBL→灯拆分插件（TrueHDRI 等）。**"隐式表示 → 显式灯"的需求二十年没变，变的是隐式表示本身**（SH → 探针体 → 神经场）。

## Relationships

- [[Gkaravelis 2016 — Inverse Lighting Design using a Coverage Optimization Strategy]]：互补两端——Sloan 在**方向域**解析抽灯（无空间位置），Gkaravelis 在**空间域**组合选灯（无方向分布）。
- [[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization]]：Lipp 的 HSH 目标是曲面域的；若旧数据是探针而非 lightmap，Sloan 这条路线提供目标构造与初值。
- [[LightOpt — Lights Optimization for Real-Time Rendering]]：实时灯数优化对应物。
- [[Global Illumination]] / [[Physically Based Rendering]]：上游概念页。

## Personal Knowledge State

- 本笔记只覆盖 pp.12–14（抽灯节）+ 相关归一化讨论，文档其余部分（ringing 治理、SH 积、code 附录）未逐节核；
- 抽灯节全部公式数字亲见原文（StupidSH35 版，文本层可提取）；环境光归一化因子的版本差异已标"未核"；
- p.14 插图（1/2/3 方向光近似）已提取并入笔记（见上）。

## Learning Value

- **"换个域定义误差"的示范**：在 irradiance 空间定义最小二乘 ⇒ 闭式解；这个"先卷积再拟合"的视角对设计迁移目标的度量直接有用（lightmap 的"可信频段"问题同理——先想清楚误差定义在哪个函数空间）；
- **初值构造先于优化器**：多峰问题的解质量由初值覆盖决定，Sloan 用球面覆盖半径给出先验点数——与 Gkaravelis 的细网格、Schwengerer 的"尽可能小必要地大"是同一思想在三种表示下的变体；
- 42 页文档是 SH 工程化的"避坑全集"，值得作为 SH 主题的案头参考留存。

## Visualization

（见 TL;DR 与上文嵌入的 p.14 原图；本文档抽灯节仅此一幅插图。）

## Notes

- 获取渠道：作者主页 ppsloan.org 公开归档（StupidSH35.pdf，42 页，文本层完整）。
- 同名 GDC 2008 讲座；文档自述"最新版本见作者主页"。
- 引用时注意版本号（SH35/SH36 公式有差异）。
