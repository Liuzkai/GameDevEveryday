---
type: paper
title: "Stable Fluids"
authors: [Jos Stam]
year: 1999
published: "1999-08（SIGGRAPH 99 Conference Proceedings, Annual Conference Series, pp. 121–128）"
venue: "SIGGRAPH 1999（署名 Alias|wavefront）"
url: "http://www.dgp.toronto.edu/people/stam/reality/Research/pdf/ns.pdf"
code: "文中自述求解器约 500 行 C（Vstep/Sstep 两个主例程；FISHPAK POIS3D 作线性解算器）"
project_page: "https://www.dgp.toronto.edu/people/stam/reality/Research/pub.html（作者主页；ns.pdf 直链至今可下）"
category: [fluid-simulation, navier-stokes, real-time, animation, classic]
importance: A
historical_importance: 5
game_relevance: 4
production_readiness: "Industry Adopted（Maya Fluid Effects 的直接理论来源；影视/游戏流体管线谱系起点）"
user_level: "Normal（结论层 25 分钟：四步分解 + '稳定性的代价是耗散' + 换空间彩蛋；推导层可跳过——原文自己说'Sections 2 可跳过，实现读 Section 3'）"
status: unread
aliases: [Stable Fluids, 稳定流体, Stam 1999]
tags: [fluid-simulation, navier-stokes, real-time, classic]
---

# Stable Fluids（Jos Stam, SIGGRAPH 1999）

> **入库 2026-10-13（Run 35）。** 作者主页直链（ns.pdf，8 页全文提取核对）。**库内"流体域"开域第一篇**——物理/仿真域此前只有粒子（[[Reeves — Particle Systems (1983)|Reeves 1983]]）与神经仿真样本，经典侧的"流体求解"整格空白，本篇补上。
> **一句话定位**：**"让流体模拟不爆炸"的那篇论文**——此前（[[Foster — Modeling the Motion of a Hot, Turbulent Gas (1997)|Foster-Metaxas]]）的显式求解器会"blow up"，时间步被稳定性卡死；Stam 用**拉格朗日回溯（特征线法）替代显式对流 + 隐式扩散 + 投影**，做出**无条件稳定**的完整 Navier-Stokes 求解器——"**不管时间步多大都不会爆炸**"，代价是"**数值耗散**"（原文自己交代：流会衰减得过快——"但在图形学里问题不大，因为交互系统里**动画师会不断施加外力把流'续命'**"）。
> **库内位置**：**"感知优先"设计哲学的 1999 年原文出处**——"在计算机图形学中，流体的**形状与行为**是第一位的，物理精度是次要的甚至无关的"（引言语）；与今日 [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]]（2026 感知路线）构成"**水的两条路**"跨 27 年对照；又与库内 [[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|Stam 1998*]]（同作者、[[Loop — Smooth Subdivision Surfaces Based on Triangles (1987)|细分曲面线]]）构成"**同一个人、两种换空间**"（细分数值化 ↔ 流体傅里叶化）。

## TL;DR

**背景**：想给动画师一个**实时交互**的流体工具（烟/火/水/纹理）——基于物理模型（而不是关键帧或过程式招式），让"涡旋、绕流"几乎自动出现。但当时 CG 界的完整 3D NS 求解器（Foster-Metaxas 1997，基于 Harlow-Welch 1965 的显式差分）有一个死穴：**时间步稍大就数值爆炸**（对流项 CFL 条件 uΔt/Δx < 1），交互性被稳定性锁死。

**做法（原文 Section 2/3，四步分解）**——把 NS 方程按项拆开顺序解，最后投影回"无散场"：

```text
每个时间步：w0 →（① 加外力）→ w1 →（② 对流）→ w2 →（③ 扩散）→ w3 →（④ 投影）→ w4

① 外力：w1 = w0 + Δt·f          （最简单，直接加）
② 对流：回溯粒子 + 线性插值       ← 稳定性的第一来源（拉格朗日视角）
   "站在新位置 x，取 Δt 之前你来自的那个位置的旧速度"
   最大原理：新场的最大值 ≯ 旧场最大值 → 不放大 → 不炸
   （把原来要显式差分的非线性项，变成"一个粒子追踪器 + 一个线性插值器"）
③ 扩散：隐式 (I − νΔt∇²)w = w0   ← 稳定性的第二来源
   （显式扩散在粘性大时不稳定；隐式 → 稀疏线性系统 → 多网格/FFT/FISHPAK）
④ 投影：解 Poisson 方程 ∇²q = ∇·w，再 w − ∇q → 保证无散（质量守恒）
   （Helmholtz-Hodge 分解的算子形式：任何场 = 无散场 + 梯度场）
```

**彩蛋（Section 2.3，周期性边界）**：换到**傅里叶域**，一切变两行——扩散 = 乘一个**低通滤波器**（衰减 ∝ 正比于 Δt·ν·k²），投影 = **把速度向量里平行于波数 k 的分量删掉**（"无散场的傅里叶变换永远垂直于它的波数"）。**只需粒子追踪器 + FFT**。

**结果/落地**：2D/3D 实时交互（鼠标施力）；**涡旋自动出现**（投影步骤强制涡量）；用"**被平流输送的纹理坐标**"在低分辨率网格上做出细腻"缕状"烟雾（三套纹理坐标周期性重置、叠加——Max et al.）；SGI Octane（R10K，192MB）上交互。**求解器约 500 行 C**。

## Problem

- **痛点一句话**："Ideally, a user should be able to interact in real-time with a fluid solver **without having to worry about possible 'blow ups'**"（引言语）；
- 前史的问题清单（原文引言语自带谱系）：早期"流原语"拼装（粒子系统/树叶摇曳）**不响应外力**；随机湍流（质量守恒、自带旋转，但仍是"贴图式"静态风场——Stam-Fiume 自己的前作）；2D 涡方法（Yaeger-Upson / Gamito et al.）；Chen et al. 的 2D 实时水——"**既限二维、又不稳定**"；Kass-Miller 的浅水线性化——便宜但**丢掉了"流体最有趣的旋转运动"**；Foster-Metaxas 的完整 3D NS——**对，但显式求解器不稳定**。

## Historical Context

```text
1965  Harlow-Welch：CFD 里不可压流的经典差分法（Foster-Metaxas 的数学祖先）
1986  Yaeger-Upson：《2010》木星（2D 涡方法 + Poisson 求解）
1990  Kass-Miller：浅水方程线性化（快速稳定，但无旋转结构——原文点名不足）
1993  Stam-Fiume：湍流风场（质量守恒随机场；仍是"给定风场"而非"解出来的流"）
1995  Stam-Fiume：火与气体（扩散过程；速度场仍非动力学）
1996  Foster-Metaxas：液体（完整 3D NS 首次走进 CG）
1997  Foster-Metaxas：热湍流气体（显式差分；CG 界主流起点——但会爆炸）
1999  ★ 本篇：无条件稳定求解器（拉格朗日对流 + 隐式扩散 + 投影）
——"同一批人、同一目标：让 CG 用得起物理流体"；Stam 在此前 6 年的风/火工作全部指向这里
```

## Previous Work（原文视角）

- **过程式/原语流**：不响应外力（"do not respond dynamically to user-applied external forces"）；
- **显式 Eulerian 求解**（Foster-Metaxas）：稳定条件 uΔt ≤ Δx——"小间距/大速度 ⇒ 时间步必须很小"；爆炸后要重启；
- **已有替代数值技术**：Foster-Metaxas 本可以用的其他技术已存在但没被采用；本篇明确**借鉴 CFD 数值分析文献**（Chorin-Marsden 教材；特征线法；FISHPAK）——"**把问题放进数学框架，才能借用半个世纪的数值分析积累**"（Section 3.2 原话的转述）。

## Core Idea

**"要稳，就不要在'会放大的方向'上显式推进"**——三项手术：

```text
① 对流（非线性项）→ 换成拉格朗日视角：
   不"用旧场算新场"（显式差分），而是"回看粒子从哪来"（回溯 → 插值）
   ——稳定性的数学保证：最大值不增（maximum principle 的离散版直觉）
② 扩散（粘性项）→ 换成隐式：
   求解 (I − νΔt∇²)w = w0 ——系数矩阵不随时间步变化，大步安全
③ 投影（质量守恒）→ 独立成步：
   每个整步结束都强制无散 → 涡旋（vorticity）被"逼"出来 → 更有"流体味"
```

**为什么可以不管精度**（原文自陈设计哲学，两段都是原话转述）：

> "我们的方法在计算流体文献里找不到——它是为计算机图形学**定制**的。对工程应用精度不够：**太多'数值耗散'**（流比实验衰减得快）。但在图形学应用里这不太糟——**特别是交互系统里，动画师施加外力让流'保持活着'**。而且完全不衰减的流可能太混乱、难控制。"

> "**CG 里流体的形状与行为是第一位的，物理精度是次要的甚至无关的。**"

## Technical Approach（实现层，原文 Section 3）

- **网格**：cell-centered（vs Foster-Metaxas 的边点定义——"更好实现"）；速度双缓冲 U0/U1 轮换；标量场 S0/S1；
- **主循环**：`Swap → Vstep（addForce/Transport/Diffuse/Project）→ Sstep（addForce/Transport/Diffuse/Dissipate）`——**标量场（密度/温度/纹理坐标）走同一套管道**，多一个耗散项（除以 1+Δt·a）；
- **Transport**：RK2 粒子追踪（+ 高梯度区域自适应细分）+ **线性插值**——"**不用高阶插值：振荡与过冲会带来不稳定；高阶样条虽然稳定但会把流抹平**"（一手工程判据）；
- **线性解算**：理论上多网格最优；实践中用 **FISHPAK（POIS3D）**且"比我们自己的多网格还快"；扩散与投影共用它（附录 B 给全系数）；有内部边界时换多网格/松弛；
- **傅里叶版**（周期边界）：`addForce → advect → FFT → diffuse(乘衰减) → project(垂直化) → iFFT`——O(N log N)，"几行就能复述整个求解器"；
- **交互系统**：鼠标施力；可建立场间依赖（如温度场驱动浮力/湍流力）；噪声项 ∝ 密度（用户参数）产生"滚涌"效果；
- **纹理细节**：平流输送纹理坐标 + 三套周期重置/叠加（承接 Max et al. 1992）——**低分辨率网格上的高视觉复杂度**；单方向光单遍自阴影。

## Key Contribution

1. **CG 第一个无条件稳定的完整 NS 求解器**：拉格朗日 + 隐式取代显式——"不管时间步多大都不爆炸"，交互性从一开始被稳定卡死 → 解锁；
2. **特征线法用于"速度本身"**：此前该法多用于"在给定风场里输运密度"（原文点名 [13,18,21,5]）——**第一篇用回溯法更新速度场本身**；
3. **"四项分解 + 投影"的模块化求解器**：简单到"**约 500 行 C**"、可复现（CDROM 时代）；每一项都可以独立替换（换更好的对流/解算器）；
4. **"感知优先"的设计论证**：明确写出"**不稳定-耗散-可控性**"三者权衡与取舍立场——CG 流体求解器之后二十余年的工程共识蓝本；
5. **傅里叶域的两行解法**："扩散 = 低通、投影 = 垂直化"——**算子对角化**的最优雅样本。

## Why It Works

- **稳定性的本质 = "不放大"**：回溯 + 插值让新场 ≤ 旧场（离散最大原理）；隐式扩散把放大因子钉在 (0,1) 内；投影只减不增（去梯度分量）——**每个算子各自无放大 ⇒ 组合稳定**；
- **"耗散不是 bug，是交互系统的 feature"**：完全无耗散 = 混乱难控；动画师的外力输入就是"能量源"——**控制与被控**在同一个耗散率上达成平衡（对照 [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]]"用户提示保持动效"：27 年后同一种"人在环"设计）；
- **数学框架的杠杆**：Helmholtz-Hodge / Poisson / 特征线法都是"借来的"——**借对了框架，实现就剩 500 行**。

## Limitations（原文自陈）

- **数值耗散偏大**（"衰减得比真实验快"）——精度不足以进入工程应用；
- **不支持自由边界（水面）**：原文明确列为未来工作——"水的自由表面在时间上动态演化，难得多"（对照：Foster-Metaxas 1996 的液体线与后来 FLIP 接过这一棒；而 **2026 的 [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]] 干脆连模拟都不做了**——"水的两条路"闭环）；
- 内部边界/复杂物体需要更贵的求解器（多网格/松弛）——本篇未展开；
- 无湍流级联/涡量增强（vorticity confinement 是 [[Fedkiw — Visual Simulation of Smoke (2001)|Fedkiw 2001]] 的补丁）——小尺度涡旋仍会被耗散掉。

## Game Development Relevance

- **实时流体效果的谱系起点**：游戏与影视的烟/火/水效果求解器（含引擎内 2D/3D 网格法）在方法上大量继承本篇的"半拉格朗日 + 隐式 + 投影"骨架；
- **产业落地实证链**（一手核证）：
  - **Maya Fluid Effects** = Stam 在 Alias|wavefront 的产物 → **2008 奥斯卡技术成就奖**（Duncan Brinsmead, **Jos Stam**, Julia Pakalns, Martin Werner"设计并实现 Maya Fluid Effects 系统"）；→ 后续 **Autodesk Bifröst** 延续流体线；
  - **Houdini FLIP**（粒子-网格混合）是"**另一条**"主流路线（影视管线标准）——与网格法互补：**网格法省粒子、FLIP 省体积损失**；
  - **Stam 本人 2019 年加入 NVIDIA** 任图形研究员；2003 年 GDC 的《Real-Time Fluid Dynamics for Games》为其游戏向版本；
- **对 VFX 预算工作的直接映射**：**网格法的成本 ∝ 网格单元数 × 步数**（原文：O(N)，"任何算法至少要看一遍每个格子"）——与你的五维预算同宗：**"分辨率 × 步数"就是流体特效的预算轴**；"降档 = 降网格 + 降步数（或换 FLIP 粒子数）"；
- **两条路的选型判据（喂给 2026）**：*"这一处需要物理演化，还是只需要视觉可信？"*——物理线（Stam→FLIP→引擎流体）管"要跟交互响应"的；感知线（Make It Flow 一类）管"氛围水"的。

## Unreal Engine Relevance

- UE 内的网格类实时流体（Niagara Fluids 实验插件的 2D/3D Gas/Liquid 模板）走的就是"**网格 + 每步推演**"家族，方法骨架与本篇同源；游戏里更常见的廉价替代（翻页纹理/flipbook、Niagara 粒子烟）本质是"把演化离线预烘焙"——**"实时演化 vs 离线烘焙"正是本篇 500 行求解器与当代引擎特效的同一个分界问题**；
- 与 UE 水体系统（Water）的关系：水体渲染（波面/FFT/网格 LOD）与流体求解（本篇）是两件事——但"感知优先"的路线选择（Water 系统也不做完整 NS）与 Stam 1999 的立场一致。

## Technology Evolution

```text
流体模拟（CG 谱系，本篇为枢纽）：
  1965 Harlow-Welch（CFD 差分）→ 1990 Kass-Miller（浅水，无旋转）→
  1996/97 Foster-Metaxas（完整 NS，显式/不稳定）→
  ★ 1999 Stable Fluids（无条件稳定：拉格朗日对流 + 隐式 + 投影）
    ├─ 2001 Fedkiw et al.《Visual Simulation of Smoke》：+ 涡量约束（找回小尺度涡）
    ├─ 2003 Stam《Real-Time Fluid Dynamics for Games》（GDC）：游戏向版本
    ├─ 2005 Zhu-Bridson：FLIP 引入图形学（粒子-网格混合）→ Houdini FLIP 成为影视标准
    ├─ 2008 Maya Fluid Effects 拿奥斯卡（Stam 为获奖人之一）
    └─ 2020s+ 神经流体（学习式仿真/神经涡量）……
  ↕（同作者的另一条线）
  1998 Stam 细分求值（同一台"递归→特征基"机器）× 1999 Stable Fluids（同一台"算子对角化"机器）
  ——"换空间"二连：细分换特征基、流体换傅里叶
  ↕（27 年后的呼应）
  2026 Make It Flow：水的"感知路线"——连模拟都不需要（"注入"运动）
```

## Relationships

### Based On

- **Chorin & Marsden**（Helmholtz-Hodge 分解与投影方法的处理框架，原文明言 p.36ff）；**Harlow-Welch 1965**（不可压差分法祖本）；**Foster-Metaxas 1997**（被取代的对象——也是本篇的出发点："我们最初想基于他们的工作"）。

### Extends

- **Stam-Fiume 系列**（1993 湍流风、1995 火与气体、1997 通用框架）：把"非动力学的速度场"升级为"动态求解的速度场"（原文自陈"fundamentally different"之处：首次用特征线法更新速度本身）；
- **Max et al. 1992**（纹理坐标平流可视化）——从"可视化风场"扩展到"合成流体动画的纹理细节层"。

### Related

- [[Reeves — Particle Systems (1983)|Reeves 1983]]——本篇参考文献 [15]：粒子系统是"流原语"时代的前辈；
- [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow（2026）]]——"水的两条路"：模拟（本篇）vs 注入（感知）；共享"人在环"设计；
- [[Stam — Exact Evaluation of Catmull-Clark Subdivision Surfaces (1998)|Stam 1998]]（同作者）——"换空间"姐妹篇；
- [[Participating Media]] / [[Neural Physics Simulation]]——体渲染与神经仿真两侧的接口。

### Followed By

- [[Fedkiw — Visual Simulation of Smoke (2001)|Fedkiw 2001]]（涡量约束——待入库）；**Zhu-Bridson 2005**（FLIP 进图形学）；**Stam 2003 GDC**（游戏向）；**Houdini FLIP / Maya Fluid Effects / Bifröst**（产业形态）；2020s 神经流体。

## Personal Knowledge State

- Normal（结论层 25 分钟：四步分解 + "稳定=不放大" + 耗散取舍 + 换空间彩蛋；推导细节可跳过——原文自称 "application oriented reader can skip Section 2"）。
- 与你的背景接口：**Houdini FLIP 的使用经验**正好是"另一条路线"（粒子-网格混合）的实践侧——本篇补的是"网格线"的理论来源与因果链。

## Learning Value

- **三句可背**：① "**回看粒子从哪来**"（特征线法——比差分省心且稳）；② "**稳定性的代价是耗散，而耗散在交互系统里可以是 feature**"（动画师续命）；③ "**投影就是删掉梯度分量**"（无散 = 垂直化）；
- **判据**：*"我的仿真能不能接受'不精确但稳定'？谁负责把能量/误差补回来？"*——本篇答案是"交互者（人）"；对照 [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]]（提示在环）、[[Turquin — Practical Multiple Scattering Compensation for Microfacet Models (2019)|Turquin]]（gain 补回）——**"近似 + 谁补账"的家庭相册**；
- **cost 模型**：网格法 O(单元数)——**流体特效的预算轴 = 分辨率 × 步数**（可直接进你的五维预算讨论）。

## Visualization

![[水双路_Stam 1999 与 Make It Flow 2026 对照图解]]

## Notes

- **核证记录**：作者主页 pub.html 核对（"Jos Stam, 'Stable Fluids', In SIGGRAPH 99 Conference Proceedings, pp. 121-128"）；ns.pdf 8 页全文 pypdf 提取 + 逐页精读；参考文献表（FISHPAK 附录 B 系数、RK2、Foster-Metaxas 引用）均出自原文；
- **产业事实核证**：Maya Fluid Effects 2008 奥斯卡（获奖人含 Stam）；2005 奥斯卡技术成就奖（细分曲面，与 **Catmull、DeRose** 共享——**库内 10-10~10-12 三天刚补齐的三人组，此处合流**）；2019 奥斯卡科技成就奖（细分曲面）；SIGGRAPH CG Achievement Award 2005；职业轨迹（Alias 1997–2006 / Autodesk 2006–2018 / NVIDIA 2019–）——来源为 Wikipedia 与作者本人 keynote 材料（visgraf.impa.br 存档），关键奖项与从业时间线双源一致；
- 库内首篇"流体域"经典；[[Fluid Simulation]] 概念笔记同日建立。
