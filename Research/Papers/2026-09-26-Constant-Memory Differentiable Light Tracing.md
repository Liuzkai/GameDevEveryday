---
type: paper
title: "Constant-Memory Differentiable Light Tracing"
authors: [Linas Beresna, Eugene Fiume]
year: 2026
published: "2026-09-26 (arXiv)"
venue: "arXiv preprint（Simon Fraser University；cs.GR；14 页）—— 实现于 Mitsuba 3 + DrJit，CUDA/OptiX 后端。**2026-10-10 更新：正式版投出——SIGGRAPH Asia 2026 Technical Communications（标题 "Stochastic Graph Compression for Constant-Memory Differentiable Light Tracing"，arXiv 2610.10847，4 页，聚焦 ResLRB）**"
url: "https://arxiv.org/abs/2609.32920"
code: ""
project_page: ""
category: [differentiable-rendering, light-tracing, inverse-rendering, monte-carlo, memory, caustics]
importance: A-
historical_importance: 1
game_relevance: 3
production_readiness: Research
user_level: "Hard（推导层）/ Normal（结论层：记忆问题 + 两种压缩策略）"
status: unread
aliases: [Constant-Memory Differentiable Light Tracing, CMDLT, LRB, ResLRB, LRB-3-pass, 常数内存可微光迹追踪, 可微光追记忆问题]
tags: [differentiable-rendering, light-tracing, inverse-rendering, monte-carlo, memory-budget, caustics]
---

# Constant-Memory Differentiable Light Tracing（Beresna & Fiume 2026）

> **2026-10-10 更新（会议版到货）**：本工作的正式发表版出现——**"Stochastic Graph Compression for Constant-Memory Differentiable Light Tracing"**（arXiv 2610.10847；**SIGGRAPH Asia 2026 Technical Communications**，4 页）。该版本**聚焦 ResLRB 单方法**（原文："We describe Reservoir Light Replay Backpropagation (ResLRB)..."），可视为本预印本（14 页，含 ResLRB + LRB-3-pass 双策略）的**会议短文版**——按"同一研究实体"处理（更新本笔记，不新建）。同作者同日另有 [[2026-10-08-Sensitivity as an Arbitrary Output Variable for Differentiable Rendering|Sensitivity AOV]]（本库 10-10 入库）——**一篇机制、一篇接口，组稿阅读**。

## TL;DR

**可微渲染的"记忆问题"（反向模式必须存计算图）在相机侧被 [[Vicini — Path Replay Backpropagation (2021)]]（PRB）用"种子重放"解决后，本文把同一目标搬到光源侧（light tracing / particle tracing），并证明它需要全新的机制。** 一句话：

```text
相机路径：一条路径 → 一个像素       （one-path-one-pixel）
光路：    一条路径 → 最多 k 个像素  （每个散射顶点都能连到传感器）
          → 每个顶点的 adjoint 不同且依赖"未来"的连接，重放时拿不到
          → PRB 的两趟结构在常数内存下无法直接推广（这是结构问题，不是实现问题）

两个常数内存解法：
  ① ResLRB      —— 蓄水池把 k 个连接压成 1 个代表（两趟保住，方差↑）
  ② LRB-3-pass  —— 多一趟遍历 + "累加-减法"消耗下游 adjoint（期望与 naive AD 一致）
```

**对库的意义**：[[Differentiable Rendering]] 瓶颈线的**第二批材料**（第一批为 9-30 同日的 PRB 笔记）。它把"可微渲染的规模化"讲清楚了：**在内存常数化之后，剩下的问题全在时间与方差上**。[[LightOpt — Lights Optimization for Real-Time Rendering]] 做的事（优化光源）正是"光侧参数"的 inverse rendering——本文处理的**光源侧可微输运**是其最近的数学邻居之一。

## Problem

**反向模式微分 Monte Carlo 光输运，朴素实现需要存与环境交互规模同量级的计算图**——图的大小随路径长度（× 采样数）线性增长，深路径（次表面散射 >100 次事件）或高采样直接不可行。

PRB（2021）对**视点路径追踪**解决了这个问题：从 PRNG 种子重建同一条路径 + 常数内存反向重放。但它的成立依赖一个重要性质——**one-path-one-pixel**：

- 相机侧：一条路径只贡献一个像素 → 图像空间 adjoint L̄ 是**单个标量**，重放开始前就能查到；
- 光源侧：一条光路在**每个散射顶点都能连到传感器（splat）**，最多贡献 k 个像素 → 不存在单一 adjoint，**每个顶点的 adjoint 取决于该顶点之后（未来）的传感器连接**，而前向重放还没走到那里。

作者的结论：**这是结构性障碍**——buffered 两趟变体（把 splat 位置/辐射值记下来）只能降低常数因子，**记忆仍然随路径长度线性增长**。要常数内存，必须换机制。

## Historical Context

```text
2020  Radiative Backpropagation (Nimier-David et al.) —— 把梯度变成独立的"伴随输运"问题
        · 解决了记忆：不存中间状态
        · 但把时间变成【二次】：每个顶点都要独立 MC 估计入射辐射
        · 且不支持理想镜面（Dirac delta 的 BSDF）
2021  PRB (Vicini et al.) —— 种子重放 + 局部 AD，相机侧【常数内存 + 线性时间】★ 已入库
        · 前提：one-path-one-pixel；且"相邻散射顶点的低维雅可比可逆"
        · 后续被推广：动态几何 (Worchel 2025)、声学输运 (Finnendahl 2025)、MC PDE (Yilmazer 2024)
2024  DPM-G（可微光子映射）—— 攻【采样】问题（SDS 路径不可达）；本文说它攻的是另一半
2026  ★ 本文 —— 攻【记忆】问题的光源侧：可微光迹追踪的两条常数内存路线
```

**两条线的关系（本文自己的表述，值得引用）**：DPM 与本文都瞄准"相机侧方法够不到的光输运区域"，但**瓶颈不同**——DPM 改渲染器解决**采样问题**（SDS 路径不可达）；本文改反向传播结构解决**记忆问题**（AD 图随路径长度增长）。DPM 靠"每迭代只采样一部分像素"把内存压有界；本文**每个像素、每个样本都处理**，内存界来自重放结构本身。

## Core Idea

**核心观察：需要的东西其实只有一个标量序列之和。**

对一条光路 x̄=(x₀…x_k)，重放时顶点 x_j 的梯度需要被"下游所有传感器连接的 adjoint 加权贡献之和"加权：

```
需要：Σ_{i>j} L̄ᵢ · Lᵢ   （下游 adjoint × splat 辐射）
```

这个和在前向重放里"还没路过"，所以两趟拿不到、存下来又线性增长。**两个解法正是对"如何在不存逐顶点数据的前提下拿到这个和"的两种回答**：

| | 策略 | 代价 | 梯度质量 |
|---|---|---|---|
| **ResLRB** | 把 k 个候选连接**随机压缩成 1 个**（加权蓄水池采样） | 两趟结构保住，**方差上升**（≈ 丢弃连接数） | 期望无偏 |
| **LRB-3-pass** | **不丢连接**：多一趟遍历，先累加 δ_total=Σ L̄ᵢLᵢ，再在反向趟里**逐顶点减法消耗** δ_rem | **多一趟遍历**（+ 前 2 趟的 Jacobian 运输） | **与 naive AD 期望一致**（且"同噪声模式"） |

## Technical Approach

### ResLRB（两趟，压缩）

1. **Primal 趟**：标准光迹追踪循环；每个有效传感器连接 (Lᵢ, uᵢ) 以权重 ωᵢ=ℓ(Lᵢ)（亮度摘要）流式送入**容量 1 的加权蓄水池**（替换概率 ωᵢ/Ωᵢ）；路径终止后只留下一个代表 R。被选中的连接以 Ω_k/ω_R **重加权后 splat**（保持对"全部 k 个 splat 之和"无偏）。
2. **Adjoint 趟**：重新播种 PRNG，重放到深度 R；在每个顶点用 log-derivative 恒等式（同 PRB）按 **L̄_R · Ŝ / f_s⁽ⁱ⁾** 加权（只有一个 adjoint 标量可查）。
3. 关键细节：**蓄水池更新逻辑必须在前向重放里"空转重放"一遍**（只为推进 PRNG 流，选择结果已知）——重放方案对随机数流的强依赖可见一斑。

### LRB-3-pass（三趟，累加-减法）

1. **Pass 1（Primal）**：正常追踪 + splat → 成像 → 求 loss → 反传到胶片管线得到**每像素 adjoint 张量 δI**（只读查找表）。
2. **Pass 2（Accumulate）**：重放同一路径；每个有效连接处从 δI 读出 L̄ᵢ（**按重建滤波器加权查找**），累加 **δ_total = Σ L̄ᵢ · Lᵢ**。
   - ⚠️ 必须是**逐项乘积之和**，不是 (ΣL̄ᵢ)(ΣLᵢ)——后者多出随路径长度增长的交叉项。
3. **Pass 3（Backward）**：再重放一次，δ_rem 初始化为 δ_total；每个顶点：① 反传当前 splat 计算（∇θ f_splat⁽ⁱ⁾·L̄ᵢ）；② **减法消耗** δ_rem ← δ_rem − L̄ᵢLᵢ，此后 δ_rem 恰为"下游和"；③ 用 δ_rem 按 log-derivative 恒等式反传局部 BSDF。
   - δ_rem 本身已含 L_j 权重，所以 PRB 里那个"剩余辐射因子"不再需要——**与 PRB 的 subtraction 模式直接同构**。

### Attached 扩展（几何/法线/折射率）

- **光侧特有的新自由度**：相机侧扰动只改"收集到的辐射"（像素坐标由相机光线固定）；**光侧扰动改的是"通量被送到哪里"**——sensor 坐标 uᵢ 本身是路径几何的函数，必须一起微分（position adjoint ∂ℒ/∂uᵢ，**在相机侧没有对应物**）。
- 实现：前向模式累积 ray Jacobian J^ray_{0→i}（4×4，位置-位置参数化；重放时用其逆投影 adjoint 回局部系）+ sensor-connection Jacobian J^sensor_i（3 色 + 2 位移 = 5×4）→ J^splat。
- 存储：attached 每样本只多一个 4×4 矩阵 + 一个 4 分量向量，**仍与路径长度无关**。
- 正则化：雅可比在漫反射顶点奇异（采样策略不依赖入射方向）→ 加零均值随机噪声（同 PRB 方案）。

## Key Contribution

1. **结构分析**：证明 PRB 的两趟重放**不能**在常数内存下推广到光迹追踪（one-path-many-pixels 破坏了"预可查的单一 adjoint"），且 buffered 变体保持线性内存——**这不是实现细节，是连接结构（connectivity structure）的必然**。
2. **两个常数内存反向模式可微光迹追踪器**（ResLRB / LRB-3-pass），各支持 detached 与 attached 两套公式。
3. **在 16GB 消费级卡上**给出完整的内存/时间/方差表征（8 个积分器变体 × 6 个场景），并完成**镜面焦散的几何逆问题**（优化透镜高度场以生成目标焦散图案，1000 次 Adam 迭代，四个积分器收敛到同一结果）。

## Why It Works

- **重放的本质是"确定性换存储"**：Monte Carlo 用伪随机数 → 路径可由种子完全重建 → 中间状态可以用"再算一遍"代替"存下来"。这条哲学在相机侧成立后，本文证明**它的适用边界就是"路径与输出的连接结构"**——相机侧是 1:1，光侧是 1:k，双向/光子映射/multi-light 也都是 1:k（本文明确点名），都需要各自适配。
- **两个策略是通用积木**：随机压缩（reservoir）与多趟累加（accumulate-then-subtract）不依赖光迹追踪本身。作者点出自然的下一步是**相机侧 PRB + 光源侧 LRB-3-pass 的双向组合**。

## Limitations

- **继承 particle tracing 全部弱点**：多数光路打不中传感器的场景低效；**SDS（镜面-漫反射-镜面）路径不可达**——那是采样问题，要靠可微光子映射（DPM-G）一类改渲染器的方法，不是本文的目标；
- **同 PRB 一样只做 interior derivatives**：不处理轮廓/不连续（silhouette boundary）——需要边界积分或重参数化（Li 2018 / Bangaru 2020 / Xu 2023），作者列为未来工作；
- **LRB-3-pass 不是免费的**：多一趟遍历 + attached 的 Jacobian 运输使其慢于 detached 对应物；本文明确"**没有单一方法在三轴（内存/方差/时间）上占优**"；
- **ResLRB 的方差在稠密连接场景最坏**（丢弃最多连接时）；在稀疏焦散场景又几乎无收益（多数时候只有 1 个候选，压不压一样）——两头不讨好的中间地带。

## Game Development Relevance

**3/5 —— 不是运行时技术，但对"离线优化为什么必须离线、凭什么可行"这条线是关键补强。**

1. **[[LightOpt — Lights Optimization for Real-Time Rendering]] 的数学邻居**：LightOpt 优化的是光源参数；本文处理**光源侧可微输运**的规模化。读它有助于理解"为什么光源优化在算法上可行、在算力上贵"；
2. **"预算"的另一种货币**：本库此前把成本写成"每灯 +1× 场景渲染"（[[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]）——**离线可微优化里的货币则是"内存 / 时间 / 方差"三轴**。本文的实践者指南可以直接抄给任何"要不要上可微优化"的决策：
   - 路径短（内存不是问题）→ naive AD 最简单最快；
   - 超显存 → 常数内存法，**默认选 LRB-3-pass**（期望一致、多数配置更快）；
   - ResLRB 只在**高连接密度 + 每迭代墙上时间紧**时有竞争力；
3. **可复用的判据（本篇最值钱）**：**「重放 = 用确定性换存储」的适用边界 = 路径与输出的连接结构**。面对任何"存不起中间状态"的模拟（粒子、流体、布料），先问：**它的连接结构是 1:1 还是 1:k？** 1:1 可以直接重放；1:k 需要压缩或累积结构。

## Unreal Engine Relevance

无直接运行时映射（研究阶段）。远期可能的接触点：**离线灯光优化工具链**（LightOpt 类工具）、**caustic 设计工具**（本文演示的透镜焦散逆问题即"让光斑长成你想要的样子"），以及"编辑器内可微烘焙"的算法储备。

## Technology Evolution

```text
2020  Radiative Backpropagation（伴随输运；常数内存、二次时间；不支持镜面）
2021  PRB（种子重放；相机侧常数内存 + 线性时间）★ 本库 9-30 入库
2024  DPM-G（改渲染器攻 SDS 采样问题）
2026  ★ 本文（重放推广到光源侧：ResLRB / LRB-3-pass）
        └─ 点名同类问题：双向方法 / 光子映射 / multi-light 都破 one-path-one-pixel
        └─ 自然下一步：相机侧 PRB + 光源侧 LRB-3-pass 的组合
```

**与"取消式优化"（本库 W39 主线）的关系**：那条线问"它能不能不存在"（排序/同步/边界）；本条线问另一个方向——**"它能不能不存"**。两者合起来是一对完整的体检：**面对瓶颈，先问"能不能不存？"，再问"能不能不存在？"，最后才问"能不能变便宜？"**

## Relationships

### Based On

- **[[Vicini — Path Replay Backpropagation (2021)]]**：本文的全部结构都建立在 PRB 的重放框架与 log-derivative 恒等式上；本文补的是"PRB 之后"。

### Related

- **⟨同周对偶⟩ ⟷ [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]] / [[2026-09-14-Gaussian Light Transport]]**：GPIS 走"假设降级为极限"（数学侧），本文走"存储换成重算"（系统侧）——**同一周内"省掉中间产物"的两种语言**；
- **⟷ [[Linear Transport Theory]]**：本文的方程（path integral / measurement contribution）就是输运算术的光源侧形式；"输运"母题在"反向传播 = 一条伴随输运"里再次显形；
- **⟷ [[Differentiable Rendering]]**：本文与 PRB 一起构成该概念的"记忆与时间轴"；读法建议见概念笔记新增章节；
- **⟷ [[Inverse Rendering]]**：本文的验证任务（透镜焦散 = 几何参数的逆问题）是 inverse rendering 的"镜面链"一侧。

## Personal Knowledge State

- **分层读法**（沿用本库 9-20 模板）：
  - **推导层 = Hard**：路径积分记号、逐顶点 adjoint 的推导、Jacobian 运输的细节——暂不必动；
  - **结论层 = Normal 可直接拿走**（4 条，不需要任何推导）：
    1. **可微渲染的"规模化"= 内存常数化 + 时间线性化**；PRB 解决了相机侧，本文解决光源侧；
    2. **重放法的适用边界是"路径-输出连接结构"**（1:1 可直放；1:k 要压缩或累积）；
    3. **两个通用积木**：随机压缩（方差换结构）vs 多趟累加（遍历换精确）；
    4. **实践者默认值**：超显存后选"多趟累加"型（本文为 LRB-3-pass），压缩型只在高密度 + 时间紧的角落里赢。
- 关联到你的学习线：这是 [[Learning Path — Differentiable Rendering]] 里"为什么这类优化必须离线做"的**具体一例**——不是因为算法不行，而是**内存与时间是硬账**，而 2020→2026 的整条线就是在还这两笔账。

## Learning Value

1. **判据（可复用）**："**它能不能不存？**"——确定性随机源 + 可重放的模拟 = 用重算换存储；前提是"逆操作存在且便宜"（低维雅可比可逆）；
2. **一对抗衡**：压缩（随机、便宜、方差升）vs 累积（多趟、精确、遍历贵）——这个取舍形态在图形学里反复出现（本库已见：GS 排序取消用"固定顺序 + 神经去噪"做同类置换）；
3. **历史观**：2020 RB（常数内存、二次时间）→ 2021 PRB（相机侧全都要）→ 2026 本文（光源侧全都要）——**"全都要"的路线每 5 年推进一个"侧"**。

## Visualization

![[可微渲染_记忆与重放路线图解.html]]

## Notes

- 作者注记：**Eugene Fiume** —— SFU 应用科学学院前院长、多伦多大学计算机系前主任（1998–2004）、SIGGRAPH 2001 论文主席、Eurographics Fellow / 加拿大皇家学会 Fellow / SIGGRAPH Academy（2020）。长期从事视觉建模与物理光传输，经典工作包括"用扩散过程描绘火焰与气体现象"（Stam & Fiume, SIGGRAPH 1995）、"用反向投影的区域光源快速阴影算法"（Drettakis & Fiume, 1994）。**本文是其团队在可微渲染方向的新工作**；
- 实验环境：RTX 5080（16GB）+ Ryzen 9 9950X，CUDA/OptiX，Mitsuba 3 + DrJit（符号模式把追踪循环录成单一融合 kernel，以保持常数内存）；6 个场景（Living Room / Glass Slab / Tumbler / Mirror / Glass Panel / lens）；
- 关键实验数字（可直接引用）：
  - **Naive AD（Mitsuba ptracer）在 Living Room 场景 path length 32 即 16GB 显存耗尽**；buffered 两趟同线性增长，且因"为所有潜在连接预分配"**有时比 naive AD 更早爆内存**；
  - 两个新方法在所有测试路径长度（4–128）下**内存平线**；
  - 焦散优化场景（path length 4）：**ptracer 反而最快**（无重放开销）——常数内存不是"更快"，是"更深的路径上还能跑"的入场券；
  - 作者感谢匿名审稿人**提出 LRB-3-pass 公式**；Living Room 场景来自 Benedikt Bitterli。
- 窗口说明：9-26 提交，**9-29 listing 公告**（本次运行的 24h 窗口内）；cs.GR 主分类。
