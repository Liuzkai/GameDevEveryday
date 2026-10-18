---
type: paper
title: "Generalized Resampled Importance Sampling: Foundations of ReSTIR"
authors: [Daqi Lin, Markus Kettunen, Benedikt Bitterli, Jacopo Pantaleoni, Cem Yuksel, Chris Wyman]
year: 2022
published: 2022-07-22
venue: "ACM Transactions on Graphics 41(4) (Proceedings of SIGGRAPH 2022), Article 75"
url: "https://doi.org/10.1145/3528223.3530158"
code: "https://github.com/DaqiLin/GrisReSTIRPT"
project_page: "http://graphics.cs.utah.edu/research/projects/gris/"
category: [rendering, ray-tracing, sampling, global-illumination, real-time, theory]
importance: A
historical_importance: 4
game_relevance: 5
production_readiness: Early Production
user_level: Normal
status: unread
aliases: [GRIS, ReSTIR PT]
---

# Lin — Generalized Resampled Importance Sampling Foundations of ReSTIR (2022)

## TL;DR

ReSTIR 家族的**统一理论**：把 Talbot RIS 从"独立同分布（i.i.d.）样本"推广到"**相关样本 + 未知 PDF + 跨域采样**"，第一次给"时空复用"补上**收敛保证**。核心对象是 shift mapping（跨像素路径映射）+ 三个权重层（贡献权重 W / 重采样权重 w / 贡献 MIS 权重 c）。三个理论后果直接改写工程实践：① 蓄水池的 M 从"样本计数"重释为"**置信权重**"，M-cap 从 trick 变成**理论必需**（不封顶会收敛到错误结果，图 2）；② canonical（本像素新生）样本占比 |R|/M 不掉到 0 是 O(1/M) 收敛的条件，"|R| > O(√M)"是硬门槛；③ 逐帧时域累积会因相关性**减速**，离线正确姿势是"spatial-only GRIS 独立帧 + 帧平均"。副产物 **ReSTIR PT**：1080p RTX 3090 上 40–80 ms 实时路径追踪 10+ 跳路径，Carousel 场景 MAPE 0.39 vs 纯 PT 1.63 / ReSTIR GI 0.45。

## Problem

- RIS（Talbot 2005）的收敛证明全部建立在 **i.i.d. 假设**上；ReSTIR 的核心操作——跨像素/跨帧复用——天然破坏独立性。
- 复用引入相关性后，Bitterli 2020 只证了**无偏**，没证**收敛**：无偏 + 不收敛 = 可能稳定地收敛到错误结果。
- 论文最锋利的教学样本（图 2）：两像素各积一个 1D 函数，迭代复用下方差卡在 10⁻¹ 不再下降——每轮只加 2 个独立样本，其余全是复制品，分布锁定在错误位置。
- 此前各家的"去相关努力"（随机化邻居、M-cap、样本验证）全是**经验补丁**，没人能回答"多大相关性算安全"。

## Historical Context

```text
1987  Rubin SIR（统计学的 RIS 原型：bootstrap filter 前身）
1995  [[Veach — Optimally Combining Sampling Techniques for Monte Carlo Rendering (1995)|Veach-Guibas MIS]]（多策略无偏合并；后在"重采样 MIS"中复活）
2002  Bekaert et al. 子路径顶点重连接（跨像素复用先祖）
2005  Talbot RIS（图形学形式化；i.i.d. 假设从这里写进理论）
2013-15  梯度域渲染的 shift mapping 家族（reconnection / random replay /
        half-vector copy，Lehtinen / Kettunen）——本文跨域工具箱的直接来源
2020  ReSTIR DI（直接光，无偏性证明但无收敛保证）
2021  ReSTIR GI（Ouyang：Jacobian 修正）/ ReSTIR 体积（Lin）/ RTXDI 架构
2022  ★本文 GRIS（统一理论）+ ReSTIR PT（应用验证）+ RTXDI SDK
2025  MegaLights 进 UE 5.6（家族产品化高峰——吃的正是本文的理论红利）
```

## Previous Work

- **Talbot 2005**：同域 i.i.d. 候选 + 校正因子 W；本文 §3.1/3.2 先把"不同 PDF + MIS 权重"收编为特例；
- **Bitterli 2020**：证明 ReSTIR DI 无偏（加权蓄水池 + MIS 平衡启发式）；无收敛证明；其 Eq.5/Eq.12 是本文 Eq.1/Eq.18 的退化情形；
- **Nabata 2020**：Talbot RIS 方差上界——但**不允许跨像素复用**；
- **Bauszat 2017**（梯度域路径复用）：最早在跨域复用里显式定义 shift mapping；
- **梯度域 shift mapping 家族**（Lehtinen 2013 / Kettunen 2015 / Hua 2019 综述）：reconnection、manifold exploration、random replay、half-vector copy——本文 §7 把它们改造成 GPU 实时版。

## Core Idea

**四级推广阶梯**（图 3 是全文地图）：

| | Talbot et al. 2005 | Talbot 2005 + MIS | GRIS（本文） |
|---|---|---|---|
| 候选样本 | 同域，同一 PDF p | 同域，不同 PDF pᵢ | **任意域 Ωᵢ，PDF 可未知** |
| 贡献权重 Wᵢ | 1/p(Xᵢ) | 1/pᵢ(Xᵢ) | **只需无偏估计 1/pᵢ(Xᵢ)**（E[W|X]=1/p） |
| 重采样权重 wᵢ | (1/M)·p̂·W | mᵢ·p̂·W | mᵢ(Tᵢ(Xᵢ))·p̂·W·\|∂Tᵢ/∂Xᵢ\| |
| 输出 | Y = Xₛ | Y = Xₛ | **Y = Tₛ(Xₛ)（shift 映射回 f 的域）** |

- **Shift mapping** Tᵢ: Ωᵢ→Ω——把邻居像素的路径"搬"到本像素的积分域，必须双射（不可逆就返回 undefined）；Jacobian 行列式入账。ReSTIR GI 的 Jacobian 修正从此有家：它就是 shift mapping 的变量变换代价。
- **三个权重层**各司其职：W（贡献权重，代替 1/PDF，允许"估计值"）；w（选谁的重采样权重）；c（**贡献 MIS 权重**——每个可实现 y 必须恰好被各域覆盖一次，Eq.17，这是跨域无偏的新增条件）。
- **GRIS 估计量**（Eq.18）：`W_Y = c_s(Y) · W_s/|∂T_s/∂X_s| · (Σw_j/w_s)`——Bitterli 的 Eq.12 与 Ouyang 的 Jacobian 修正都是特例。

## Technical Approach

**收敛理论（§5.7）——ReSTIR 参数表的理论出处**：

1. 总条件：Var[Σwᵢ] → 0（目标分布 p_Y 依均方收敛到目标 p̂）。
2. **canonical 样本占比**：wᵢ ≤ C/|R| + |R|/M ≥ γ 常数 ⇒ 方差 O(1/M)；硬门槛 **|R| > O(√M)**——违反它（如固定 1 个 canonical、无限扩邻居窗口）方差先降后**升**（图 8）。
3. **相关性衰减**：复用权重相关性 ρ(i,i+k) ≤ b_k 且 b_k → 0。M-cap 的作用：时域复用样本的相对权重封顶在 M_c/(M_c+1)，新样本永远占 ≥1/(M_c+1)——**M_c=∞ 时 b_k→1，一切保证失效**（图 2 的失败就是它）。
4. "reasonable distribution"（f ≤ C_f·p 逐点有界）防止奇异贡献——方差不是被平均掉的，是被**上界**控制的。

**ReSTIR 的重释（§6）**：

- 蓄水池 Mᵣ = **置信权重**（confidence weight），不是样本计数；合并时 MIS 权重 m = M_rᵢ/(M_r1+M_r2)；ReSTIR = **无偏探索性非马尔可夫链**——单帧是链的一个状态，静场下帧平均收敛。
- 离线正确姿势（§6.5）：**别做时域累积**（相关性减速，图 9b），用 spatial-only GRIS 渲染独立帧再平均——可证收敛。

**实时 shift mapping（§7）**：

- 重连接 shift（首间接顶点硬连接）：便宜，漫反射场景够用——**ReSTIR GI 隐式用的就是它**（作者原话："trades correctness for performance"）；
- **混合 shift**（本文新配方）：random replay 兜底 + 推迟的重连接；为省显存，基路径**预选唯一**满足可连接条件的顶点（只存一个顶点，不存全路径）；
- 两个新可连接条件：**距离条件**（‖x_{i+1}−x_i‖ ≥ d_max 才允许重连接，消除角落短段的几何项奇异）+ **按 lobe 连接性**（对 BSDF 各 lobe 分别测粗糙度阈值，解决 clear-coat 歧义）。

## Key Contribution

1. **理论**：GRIS 框架——相关性/未知 PDF/跨域三个"违禁项"全部转正，收敛条件成清单（ supp 覆盖、W 无偏、c 分割、Var[Σw]→0、canonical 占比、相关性衰减）；
2. **实践**：M-cap 与随机化邻居从 trick 升格为**理论义务**；给出"哪些调参会悄悄破坏收敛"的判据；
3. **系统**：ReSTIR PT（实时 10+ 跳路径追踪）+ 离线一致版本（独立帧平均）。

## Why It Works

- 把"必须知道 PDF"弱化为"必须能**无偏估计** 1/PDF"——于是复用过的样本（PDF 已不可追踪）可以继续进池子，层层 chained GRIS 仍然可证；
- MIS 分工干净：w 里的 resampling MIS 管选择概率，c 里的 contribution MIS 管贡献覆盖，二者都只要求"分割单位"（partition of unity）；
- 收敛的杠杆只剩一个标量 Var[Σwᵢ]——整个算法设计变成"让 wᵢ 尽量均匀"的工程问题。

## Limitations

- 目标函数用**灰度** p̂=|f|：只保证亮度收敛，彩色通道间比率不受控（彩色噪声，future work 明示）；
- 混合 shift 在局部决策受限处（强折射/多重镜面）仍会失败，作者指向 manifold exploration 补位；
- 实时档 40–80 ms/帧（1080p 3090）——离"游戏帧预算"仍差一个量级，落地靠 RTXDI 的裁剪版（DI/ReGIR）；
- M-cap 抑制旧样本置信后，动画场景的时域拖影问题仍交给降噪器与时域反馈（与 [[Temporal Stability and Artistic Intent]] 的失效账本相接）。

## Game Development Relevance

- **RTXDI / MegaLights 的理论地基**：M-capping、邻域随机化、"置信权重"语义从此有出处——调参时知道每个旋钮在保护哪条收敛条件；
- **预算视角（本库主线）**：ReSTIR 家族的"复用机制成本模型"三块拼图集齐——变换（Jacobian，GI/本文）+ 失效检测（样本验证，GI）+ **置信管理（M-cap，本文）**；SABC 分档里"每像素样本数 M / 邻居数 k / 轮数 n / M-cap"本身就是一张可直接分档的预算表（[[Real-Time VFX Performance Budgeting]]）；
- **图 2 是最便宜的教学样本**：给团队讲"复用不是免费的"时，两像素反例胜过十页推导。

## Unreal Engine Relevance

- **MegaLights（UE 5.6+）**：stochastic light sampling 的 M/k/n/M-cap 默认值全部落在本文理论的保护区内；调参文档里的"历史长度限制"即 M-cap；
- Lumen 的最终收集/反射与 RTXDI 混用时的相关性问题，可用本文 §5.7 的 b_k 衰减条件做定性判断。

## Technology Evolution

```text
Talbot RIS 2005（i.i.d. 理论）
  ↓ 复用 + 蓄水池            Bitterli 2020 ReSTIR DI（无偏，无收敛证明）
  ↓ 候选改表面点              Ouyang 2021 ReSTIR GI（Jacobian / 样本验证）
  ↓ ★统一理论                本文 GRIS 2022（收敛条件 + shift mapping 框架）
  ↓ 产品化                   RTXDI 2022 → MegaLights（UE 5.6, 2025）
  ↓ 离线一致性               本文 §6.5：独立帧 spatial-only GRIS + 平均
```

## Relationships

### Based On

- [[Talbot — Importance Resampling for Global Illumination (2005)]] — RIS 原型，本文 §3 全文回顾并收编为特例
- [[Reservoir Sampling]] — 蓄水池容器；本文把 Mᵣ 的语义改为置信权重
- 梯度域 shift mapping（Lehtinen 2013 / Kettunen 2015）— §7 工具箱来源

### Extends

- [[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)]] — DI 的无偏性 → 本文补收敛；Eq.12 成为 Eq.18 特例
- [[Ouyang — ReSTIR GI Path Resampling for Real-Time Path Tracing (2021)]] — GI 的 Jacobian 修正 → 本文把它一般化为 shift mapping 框架；重连接 shift 即 GI 的隐式选择

### Enables

- RTXDI（2022，生产化 SDK）— 理论保护下的参数裁剪
- ReSTIR PT 后续：many-light shadow maps（EG 2025）、Area ReSTIR（SG 2024）等 shift mapping 变体
- MegaLights（UE 5.6，2025）— 游戏引擎产品化高峰

### Related

- [[Schied — Spatiotemporal Variance-Guided Filtering (2017)]] — 重建端互补；相关性引入的拖影最终由重建管线兜底
- [[Temporal Stability and Artistic Intent]] — M-cap 与时域复用的失效/拖影账本

## Personal Knowledge State

- **Normal（结论层）**：GRIS 估计量四层结构、M-cap 语义、canonical 占比条件——可与既有 DI/GI 结论层直接对接；
- **Hard 桥**：附录 A 定理链（A.1 权重等价 / A.2 分布收敛 / A.3 渐近方差 / A.4 MIS 界）与补充材料 S.5 证明——前置已就位（RIS 方差劈分 Normal、概率论基础），可作专项推导周。

## Learning Value

- 读通了它，ReSTIR 家族三篇（DI/GI/GRIS）从"三套算法"变成"一套理论的三次特例化"；
- "无偏但不收敛"（图 2）是全库"复用必须自带失效条件"账本的第 4 例——前 3 例（RPF 重投影验证 / DDGI 无限史剔除 / ReSTIR GI 样本验证）都是工程补丁，本文第一次给出**理论判据**。

## Visualization

![[Research/Files/Pictures/Lin 2022 — 图3 RIS到GRIS四级推广对照.png]]

**读图**：全文地图——同一套 RIS 流程在三个理论档位下的差异。左列 Talbot i.i.d.（同一 PDF，W=1/p）；中列多 PDF + 重采样 MIS 权重 mᵢ；右列 GRIS：候选可来自任意域、PDF 可以查不到（W 只需无偏估计），权重里多了 shift 映射的 Jacobian |∂Tᵢ/∂Xᵢ|，输出也要先映射回来（Y=Tₛ(Xₛ)）。ReSTIR DI/GI 的公式全部是右列的退化情形。（图 3，Lin et al. 2022）

![[Research/Files/Pictures/Lin 2022 — 图2 两像素复用收敛到错误结果.png]]

**读图**：为什么"无偏"救不了"不收敛"——左：两像素各积一个双峰 1D 函数（实/虚线）；右：方差随迭代的变化。纯 Monte Carlo（蓝）一路下降；带空间复用的 ReSTIR（橙）卡在 10⁻¹ 平台——每轮只引入 2 个新独立样本，其余全是复制品，分布锁定在错误结果上。这是 M 不封顶（M_c=∞）时 ReSTIR 的缩影。（图 2，Lin et al. 2022）

![[Research/Files/Pictures/Lin 2022 — 图1 ReSTIR PT与基线等时对比.png]]

**读图**：理论的应用验证——Carousel 与巴黎歌剧院场景 1920×1080、80 ms/帧等时对比。主图为 ReSTIR PT（10 跳路径），小面板从上到下：纯路径追踪 / ReSTIR GI / 本文 ReSTIR PT / 参考图。MAPE：Carousel 1.63 / 0.45 / **0.39**；歌剧院 1.28 / 0.39 / **0.33**——高光互反射、折射等高频光照收益最大。（图 1，Lin et al. 2022）

## Notes

- 双一作：Daqi Lin（Utah）与 Markus Kettunen（NVIDIA）贡献相等；CODE 开源（ReSTIR PT 参考实现）。
- 与 RTXDI `Reservoir.hlsli` 的对接仍是待办：本文 §6.2 的置信权重语义 + M-cap 即其中 M 的官方解释——15 分钟只读材料 100% 就绪（挂起第 3 周）。
- 后续候选（家族收尾）：Kettunen 2021 ReSTIR PT 前身（短文版）可跳过，本文已覆盖；Cond. ReSTIR（SGA 2023）/ Area ReSTIR（SG 2024）等变体按需记名。
