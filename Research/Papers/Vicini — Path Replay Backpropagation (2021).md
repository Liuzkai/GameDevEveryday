---
type: paper
title: "Path Replay Backpropagation: Differentiating Light Paths using Constant Memory and Linear Time"
authors: [Delio Vicini, Sébastien Speierer, Wenzel Jakob]
year: 2021
published: "2021-08 (ACM TOG 40(4), Article 108)"
venue: "ACM Transactions on Graphics（Proc. SIGGRAPH 2021）；EPFL Realistic Graphics Lab（RGL）"
url: "https://rgl.epfl.ch/publications/Vicini2021PathReplay"
code: "（方法实现于 Mitsuba 2；后续成为 Mitsuba 3 / DrJit 可微渲染核心路径）"
project_page: "https://rgl.epfl.ch/publications/Vicini2021PathReplay"
category: [differentiable-rendering, inverse-rendering, monte-carlo, memory, classical, path-tracing]
importance: S（经典）
historical_importance: 5
game_relevance: 3
production_readiness: "Research（但已是可微渲染的工具链标准件）"
user_level: "Hard（推导层）/ Normal（结论层：重放换存储 + 4 条判据）"
status: unread
aliases: [Path Replay Backpropagation, PRB, 路径重放反向传播, Vicini 2021]
tags: [classic, differentiable-rendering, inverse-rendering, monte-carlo, memory-budget]
---

# Path Replay Backpropagation（Vicini, Speierer & Jakob 2021）

## TL;DR

**可微渲染的"规模化开关"：让反向传播（gradient of loss w.r.t. 场景参数）在【常数内存 + 线性时间】内完成——用"重放"替代"存储"。** 这是本库 [[Differentiable Rendering]] 瓶颈线的**第一篇经典**（2026-09-30 与 [[Vicini — Path Replay Backpropagation (2021)|本文]] 的 2026 年光源侧后继 [[2026-09-26-Constant-Memory Differentiable Light Tracing]] 同日入库）。

一句话机制：

```text
naive reverse AD：把整个计算图存下来（内存 ∝ 路径长度 × 采样数）
checkpointing：在渲染里不可行（单个 checkpoint 就太大）
RB (2020)：改成独立的"伴随输运"（内存常数，但时间二次 + 不支持镜面）
★ PRB：两趟 ——
   Primal 趟：正常追踪，只记 (PRNG 种子, 最终辐射 L)
   Adjoint 趟：重新播种、重放同一条路径，逐顶点开【局部】AD 作用域反向传播
             → 任意时刻只有一个顶点的作用域活着 → 内存与路径长度无关
             → 重放成本与正常追踪同阶 → 时间线性
```

**对库的意义**：本库讲"存储 vs 计算"的权衡时（[[Reeves — Particle Systems (1983)]] 的"只存随机数表不存粒子参数"是 1983 版），**PRB 是这条原则在可微渲染里的完整形态**；也是 [[LightOpt — Lights Optimization for Real-Time Rendering]]"为什么必须离线、凭什么可行"的直接前置。

## Problem

**反向模式微分的光输运模拟，内存是硬账。**

- 渲染的"微分版本"需要的中间量访问顺序与"原版（primal）"相反——**排序冲突**使它们不能凭空重算，必须"以某种方式拿到"；
- 直接全存不可行：现代 GPU 每秒可产生 ~10¹² FLOPS 量级的中间状态（≈ 4 TB/s），存储跟不上；
- checkpointing（存少量检查点 + 重算）在渲染里同样不灵：**单个 checkpoint 就很大**，存储开销与访存延迟都不可接受。

已有答案 RB（Radiative Backpropagation, Nimier-David et al. 2020）把梯度重写成"**从相机出发的伴随辐射输运**"（梯度像光一样被发射、散射、被可微参数接收）——内存问题解决，但留了两条：

1. **时间二次**：每个散射顶点要递归估计入射辐射（估算伴随走一步，其自身又要递归）；
2. **不支持理想镜面**：Dirac delta 的 BSDF（光滑玻璃/金属）无法这样微分——而这恰恰是焦散设计、透明物体反重建等应用的核心。

另有一条"捷径"——**biased RB**（微分时把入射辐射 Lᵢ 直接设为 1，声称符号仍正确、收敛有保证）。**本文用实验证伪**：在一个封闭的金属兔子场景（1D 参数空间可穷举）中，biased RB **连符号都不对**——"即使从正确解出发也无法收敛"。

## Historical Context

```text
1976  Clark：分档 / working set —— "存什么"作为运行时问题（本库 9-29 入库）
1983  Reeves：粒子系统"只存随机数表"—— 用确定性替代状态存储的早期形态（本库 9-21 入库）
2020  RB：伴随输运（常数内存、二次时间、不支持镜面）
2021  ★ PRB：本文 —— 相机侧【常数内存 + 线性时间 + 无偏 + 支持镜面】
      · 学术谱系：EPFL RGL（Jakob 组，Mitsuba 生态）；log-derivative 技巧与 reversible programming（Gomez 2017 等）同源
2024  Yilmazer：重放推广到 MC PDE；Worchel 2025：动态几何；Finnendahl 2025：声学输运
2026  Constant-Memory Differentiable Light Tracing：把重放推广到【光源侧】★ 同日入库
```

**思想史坐标**：**"用确定性换存储"**——Monte Carlo 的伪随机数是可重播的，所以中间状态可以"重算"而不是"存下"。Reeves 1983 用"随机数只在生成时消费"的**不变式**让粒子系统可只存随机数表；PRB 用"**路径可从种子完全重建**"的不变式让整个反向传播不存图。**同一条原则，相隔 38 年。**

## Core Idea

**"重放 + 局部微分"，把微分问题变成"把同一条路径再走一遍"。**

1. **Primal 趟**：正常追踪 → 每条路径只记两样小东西：**PRNG 种子**与**最终辐射值 L**；
2. **Adjoint 趟**：重置 PRNG，用同一种子**走出完全相同的路径**；在每个顶点，只对"该顶点的散射计算"开局部 AD 作用域，按 log-derivative 恒等式反传：
   ```
   ∇θ f_s⁽ⁱ⁾ · (L̄ · L) / f_s⁽ⁱ⁾
   ```
   —— 不微分从 x₀ 到 x_k 的整条乘积，而是**用缓存的总辐射 L 当乘法权重**，只微分局部 f_s；局部梯度进参数后**立即销毁该顶点的 AD 图**。
3. 因此：**峰值内存 = 单个顶点的计算图**，与路径长度无关；总时间 = 追踪 + 重放 ≈ 线性。

**Attached（几何参数）的附加机制**：扰动几何/法线/折射率会**移動路径顶点本身**（后续所有顶点跟着动）。PRB 用**前向模式**跟踪相邻路径段的雅可比 J^ray（**位置-位置参数化**下为 4×4；作者试过位置-角度参数化，单位不相容、数值条件差）——沿路径乘积累积，重放时用其**逆**把 adjoint 投影回局部坐标系。

- **奇异点处理**：漫反射顶点的采样策略不依赖入射方向 → J 奇异 → 加**零均值随机对角噪声**（λ=0.01），且**两趟用同一噪声矩阵**以保持无偏（有证明，见补充材料）；
- **数学解释（漂亮的一步）**：整个方法可以读作"**迭代地应用局部雅可比的逆**"：
  ```
  J_h = [[1, Le], [0, fs]]  →  J_h⁻¹ = [[1, −Le/fs], [0, 1/fs]]
  ```
  —— 重放里"减去当前发射"就是在做 J_h⁻¹ 乘法。"我们不反转程序；我们反转相邻迭代之间那个低维雅可比。"（这个方法**不**能推广到一般神经网络的反向传播——那里的中间层是千维的；渲染的幸运在于**循环状态低维且结构简单**。）

## Technical Approach — 关键细节

- **三趟而非两趟**：实际实现是 Primal + 两个梯度相关 pass（pass 2 预计算"需要的信息"——辐射估计与 ray Jacobians；pass 3 累积参数梯度）。primal 用 4× 采样数（对梯度方差有利，Azinović 2019）。**多出来的这一趟是线性时间方案的固定成本**——简单问题上会显得比两趟老方法慢；
- **Detached vs Attached**：detached = 参数只改"值"（albedo、发射强度）；attached = 参数还改"几何"（法线/折射率/顶点位置）。**理想镜面必须用 attached**（要穿过 Dirac delta 微分）；
- **体积扩展（可微 delta tracking）**：null-collision 方法的离散决策（吸收/散射/null）+ 无界散射次数，此前无反模式解法。本文给出 detached 估计器（**比例项只微分一次**——微分"吸收/散射的选择"会强偏），并**首次以无偏方式求解 delta tracking 微分**（NE E 的透射率用嵌套 PRB）；
- **把噪声模式都还原**：结果不只是期望一致——**噪声模式与 conventional AD 相同**（只有浮点运算次序导致的微小数值差异）。文档里那句"the same noise pattern"是从业者会心一笑的细节。

## Key Contribution

1. 把"重放（replay）"确立为可微渲染的通用机制：**种子重放 + 局部 AD（log-derivative 技巧）+ 雅可比可逆性**；
2. 同时拿到**无偏、常数内存、线性时间**，并覆盖**理想镜面**（attached）与**体积输运**（delta tracking）——把 RB 的两条限制同时消掉；
3. 用"1D 参数空间穷举"的方式**证伪 biased RB 的符号正确性**（兔子实验）——一条硬结论：**"有偏但方向对"的说法在光输运里不成立**；
4. 方法论副产品：**"反转本地雅可比"而非"反转程序"**——一个可以带着走的解释框架。

## Why It Works

1. **随机数的确定性是"免费的存储"**：MC 估计的随机性来自可控的伪随机流；只要两趟消费同一流，就能得到"同一条路径"——第一条路径留下的唯一必要信息是**标量 L**（以及种子）；
2. **低维 + 简单结构 = 可逆**：渲染的循环状态只有 (L, β)，其雅可比 2×2/4×4，逆矩阵有闭式。这是渲染领域的特殊幸运；
3. **无偏来自两点**：log-derivative 恒等式精确；重放精确重建路径（正则化用同噪声保持无偏）。

## Limitations

- **只做 interior derivatives**：轮廓/可见性跳变（silhouette discontinuities）不在范围内——需要边界积分（Li 2018）或重参数化（Loubet 2019 / Bangaru 2020）另行处理；本文明确把这块留给并发工作（Zeltner 2021）；
- **不解决"到达"问题**：SDS 这类一方向追踪不可达的路径，重放帮不上——那是采样问题（见 2024 DPM-G、2026 光源侧论文）；
- **三趟固定成本**：问题足够简单（两趟老方法的内存/时间都不是瓶颈）时，PRB 反而更慢；
- **反函数/奇异雅可比的工程性限制**：需要正则化；补充材料有证明，但工程实现有坑。

## Game Development Relevance

**3/5 —— 不在引擎里跑，但它是"离线可微优化"这条生产线的地基。**

- 对 [[LightOpt — Lights Optimization for Real-Time Rendering]]：LightOpt 这类"离线优化灯光布局/参数"的工具，**其可行性建立在"高维参数 + 少量样本就能拿到梯度"上**——PRB 正是把这件事从"内存不可行"变成"可行"的方法。本文引言有句话值得引用：**一张 768×768 的 RGB 贴图就有 170 万+ 参数**——这就是"为什么必须反向模式 + 为什么必须省内存"的量级说明；
- 对本库的**预算语言**：给你多一种"货币换算"——**内存常数化之后，"能不能做"取决于时间线性化；时间线性化之后，剩下的问题才是每迭代质量（方差）**。三轴（内存/时间/方差）独立结算，与"组件加速 ≠ 端到端加速"（本库 9-11 三连）同一族教训；
- **"它能不能不存？"**：本库"取消式优化"体检的新分支——执行层问"同步/排序能不能不存在"，算法层问"中间状态能不能不存"。PRB 是后者的首个完整样本。

## Unreal Engine Relevance

无直接运行时映射。接触点：离线光照优化/资产拟合工具链的算法储备；以及**"重放"思路对任何 UE 侧确定性模拟的启发**（只要模拟可从种子重放，就存在"用重算换存储"的优化空间）。

## Technology Evolution

```text
2018  可微路径追踪的起点（Li et al. 边缘采样；Kato / Liu 可微光栅化）
2020  RB：伴随输运（常数内存，二次时间，不支持镜面）
2021  ★ PRB：相机侧全线打通（无偏 / 常数内存 / 线性时间 / 镜面 / 体积）
2021+ 被吸收进 Mitsuba 3 生态；被扩展至：动态几何（2025）/ 声学（2025）/ MC PDE（2024）
2026  光源侧打通（ResLRB / LRB-3-pass）—— 本库同日的 [[2026-09-26-Constant-Memory Differentiable Light Tracing]]
```

**一句话历史**：**PRB 之后，"可微渲染能不能跑"的答案从"看显存"变成了"看时间"**。

## Relationships

### Extends

- **RB / Radiative Backpropagation（Nimier-David et al. 2020）**：本文的直接前身；消掉了它的两条限制（二次时间、不支持镜面），并证伪了它的 biased 变体。

### Related

- **⟷ [[Differentiable Rendering]]**：本文是该概念"记忆与时间轴"的起点节点；与 2026 光源侧论文构成"相机侧/光源侧"对；
- **⟷ [[Reeves — Particle Systems (1983)]]**：**"用确定性换存储"的最早样本 vs 完整形态**——Reeves 只存随机数表（依赖"随机决策只消费一次"的不变式），PRB 只存种子 + 标量（依赖"重放可精确重建"的不变式）。38 年后同一原则；
- **⟷ [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)]]**：Clark 的 working set 是"时间/空间权衡"的运行时形态（存什么），PRB 是同一权衡在"微分计算"里的形态（重算什么）；
- **⟷ [[2026-09-26-Constant-Memory Differentiable Light Tracing]]**：本文的 2026 年光源侧后继；读者路径：先本文（相机侧），再彼文（光源侧）。
- **⟷ [[LightOpt — Lights Optimization for Real-Time Rendering]] / [[Inverse Rendering]]**：应用层与概念层。

### Followed By

- 重放框架的后续扩展：动态几何（Worchel et al. 2025）、时域声学输运（Finnendahl et al. 2025）、MC PDE 的分支随机游走（Yilmazer et al. 2024）、光源侧（2026）。

## Personal Knowledge State

- **分层读法**（沿用 9-20 模板）：
  - **推导层 = Hard**：log-derivative 恒等式的推导、attached 的 Jacobian 运输、delta tracking 的估计器——挂起；
  - **结论层 = Normal 可直接拿走**（4 条，不需要推导）：
    1. **"重放"三要素**：确定性随机源（种子）+ 可精确重建的路径 + 低维可逆雅可比 → 即可用重算换存储；
    2. **两笔账分开结算**：内存常数化 ≠ 快——低路径长度下 naive AD 仍最快；PRB 的价值在"深路径上还能跑"；
    3. **"有偏但方向对"不可信**：biased RB 的符号可能全局错误（兔子实验）；精度问题在光输运里是**收敛性问题**；
    4. **本库判据新分支**："它能不能不存？"——确定性系统里，中间状态的存储可以被重放替代。
- 与 [[Learning Path — Differentiable Rendering]] 的关系：本文**结论层**可在桥的第 1–2 步之后直接读（不需要学完推导）；它是"为什么这类优化必须离线做"的最清楚答案。

## Learning Value

1. **判据（可复用）**：**面对任何"存不起"的确定性模拟，先问"它能不能不存？"**——只要 (a) 随机源可重播、(b) 状态可从种子重建、(c) 逆操作存在且便宜，存储就可以换成重算；
2. **一对抗衡的范本**：PRB vs RB 是"内存优化 vs 时间优化"如何重新分配成本的完整案例——**优化不是消除成本，是选择把成本放在哪一栏**（与"离线生成量 ≠ 运行时负载"同构）；
3. **论文写作层面**：1D 参数空间穷举（兔子实验）是"证伪一个已有方法的假设"的漂亮手法——把高维问题冻结到一维，让错误无处藏身。

## Visualization

![[可微渲染_记忆与重放路线图解.html]]

## Notes

- 文献信息（RGL 官方页 + PDF 逐页核对）：ACM TOG 40(4), Article 108, August 2021（SIGGRAPH 2021）；PDF 来自 RGL 官方镜像（`d38rqfq1h7iukm.cloudfront.net`，24 MB，14 页）；
- 关键实验数字（可直接引用）：
  - **内存（detached，Coin / Dragon / Living Room）**：Conv. AD 21.6 / 21.4 / 21.5 GB → **本文 0.3 / 0.2 / 0.2 GB**（RB 同为常数内存）；
  - **时间（同场景，秒）**：Conv. AD 3.3 / 9.3 / 18.4 vs 本文 3.7 / 9.0 / 19.0 vs RB 10.0 / 60.7 / 128.3（**RB 的二次增长肉眼可见**）；
  - **Attached 内存**：Conv. AD 11.5 / 5.8 / 11.5 GB → 本文 0.2 / 0.2 / 0.1 GB；
  - **Dragon 次表面散射**（640×360、1 spp、>100 次散射事件）：Conv. AD 单次即耗尽 23GB 显卡；RB 时间不可忍受；本文常数内存 + 线性时间；
  - **体积（delta tracking）**：等时间对比中 Conv. AD 45 分钟 0 迭代；RB 3 分钟 4 迭代；本文 1:32 完成 64 迭代（比 biased RB 的 1:47/96 迭代收敛更稳）。
  - 硬件：Intel i7-7800X + TITAN RTX（23 GiB），OptiX 7.2，Mitsuba 2。
- **两处引用级别细节**：① 三趟结构（primal + 2 个梯度 pass）是"避免偏差"的必要设计（引用 Azinović 2019 的发现）；② "同一噪声模式"（same noise pattern）是本文与 conventional AD 一致性强度的最强表述；
- 本笔记的**下游**：[[2026-09-26-Constant-Memory Differentiable Light Tracing]]（2026，光源侧）——两篇合读即"可微渲染记忆问题"的完整故事。
