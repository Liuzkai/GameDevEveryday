---
type: concept
title: "Differentiable Rendering"
user_level: Hard
tags: [rendering, optimization]
---

# Differentiable Rendering

## Definition

让**渲染结果对场景参数可求导**的渲染方法。即：给定一张目标图像，能反推出"场景参数该往哪个方向改，图像才更像目标"。

```
传统渲染:   场景参数 ──正演──> 图像
可微渲染:   图像误差 ──反传──> 场景参数梯度
```

## Core Principle

把渲染管线里每一个不可导的离散操作（光栅化的覆盖判定、深度测试、可见性跳变）用一个**可导的近似**替换掉，从而让梯度能从像素一路传回几何、材质、光照乃至相机。

常见近似手段：
- 边缘软化（软化 coverage，用 sigmoid 代替硬判定）
- 随机估计（stochastic rasterization，用采样积分代替确定性判定）
- 松弛（把离散决策放松成连续权重）

## Prerequisites

```
Vector Calculus（链式法则、雅可比）
      ↓
Optimization（梯度下降、学习率、loss 设计）
      ↓
Automatic Differentiation（计算图、前向/反向模式）
      ↓
Rasterization Pipeline（顶点→裁剪→光栅化→着色，你需要知道它在哪一步断了梯度）
      ↓
Differentiable Rasterization
      ↓
[[Differentiable Rendering]]
```

注意这条链里，**前四项是数学与基础图形学，不是前沿研究**。它们是可以快速补的。

## Evolution

```
Soft Rasterizer
      ↓
Differentiable Rasterizer（如 Pytorch3D / nvdiffrast）
      ↓
Inverse Rendering（从图像反推几何/材质/光照）
      ↓
应用分支：资产生成 / 外观编辑 / 管线优化（[[LightOpt — Lights Optimization for Real-Time Rendering]]）
```

## 记忆与时间轴（2026-09-30 新增 —— 这个领域的"规模化"故事）

> 反向模式微分要求**逆序访问中间状态**：primal 算完才轮到微分，顺序相反；中间量既不能随手重算（还没走到）、也不能全存（渲染中间状态 TB/s 量级）。
> **2020 → 2026 整条路线，就是把这笔"存不起"的账一笔一笔换掉**——每一代只换一栏成本。

| 代 | 方法 | 内存 | 时间 | 梯度质量 | 关键限制 |
|---|---|---|---|---|---|
| 2020 前 | **Naive AD**（全存计算图） | 线性增长 | 线性 | 精确 | 深路径/高采样直接爆显存（实测：640×360×1spp 的 SSS 场景耗尽 23GB） |
| 2020 | **RB**（伴随输运） | 常数 | **二次** | 无偏（简化版连符号都错） | 不支持理想镜面（Dirac delta） |
| 2021 | **PRB**（种子重放，相机侧）★ | 常数 | **线性** | 无偏 + 噪声模式与 AD 一致 | 只做 interior derivatives（轮廓跳变除外） |
| 2026 | **LRB 家族**（光源侧）★ | 常数 | +1 趟遍历 | 期望一致（或"方差换结构"） | 继承 particle tracing 弱点（SDS 不可达） |

**"重放"成立的三要素**（PRB 的结论层，可当判据用）：
① 随机源可重播（PRNG 种子）；② 状态可从种子精确重建；③ **相邻状态的雅可比低维可逆**（渲染循环状态只有 (L, β) → 4×4；神经网络千维中间层不满足）。

**"1:1 vs 1:k"边界**（2026 光源侧论文的核心结论）：重放法的适用边界 = **路径与输出的连接结构**——相机路径 1:1 可直接重放；光路 1:k（每个顶点都能 splat）必须在"**随机压缩**"与"**多趟累积**"之间选一个。双向 / 光子映射 / multi-light 同理。

**对本库体检法的贡献**：与"取消式优化"（它能不能不存在）互补——**"它能不能不存？"** 是该问法的新分支。

### 双域对照：本页的"伴随"和物理侧的"伴随"不是同一件事（2026-11-13 补）

上面那套"重放 / 常数内存"的话，搬不到物理仿真上。原因不在工程，在**转置算子有没有物理对应物**：

| | 本页（光传输） | 物理仿真 |
|---|---|---|
| 算子 | **线性**传输算子 $T$ | 非线性时间积分器 |
| 转置 $T^\top$ 是什么 | **反向光路**——从相机往光源走，物理上真的存在 | 转置雅可比链，**没有物理解释** |
| 能不能"反着算一遍" | ✅ 可以（[[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization\|Lipp 2024]] 就是把它做成了算法） | ❌ 反向的动力学不是动力学 |
| 因此状态 | 可不存（重播随机数即可） | **必须存检查点**，或接受"我算过的那条轨迹"当参照物 |

**与本页"重放三要素"第三条精确对接**：③ 说"相邻状态的雅可比**低维可逆**"——渲染循环的状态只有 $(L,\beta)$，4×4 就够；而物理仿真的状态是**千维**（位置 + 速度 × 顶点数），既不低维，也谈不上可逆。**这不是"渲染侧先做出来了、物理侧还没跟上"，是第三条判据在两个域里给出了不同答案。**

配套的"内存账记在哪一层"（逐操作 / 逐段 / 逐对数段 / 不存）与方法本身的推导，见 [[Adjoint Methods]]——**本页是那条方法在光传输侧的落点，不是它的全貌**。另外提醒一句：物理侧 2026 年综述（[[2026-11-13-Shu — Same Physics Different Gradients A State-of-the-Art Report on Differentiable Physics Simulation|Shu 2026]]）的实测发现，在**实时迭代预算**下，"对理想化求解器求导"与"实际跑出来的那条轨迹求导"给出的梯度**几乎正交**（$\cos\angle$ 0.1~0.3），但优化结果可以一样好——**这动摇了"梯度角度 = 梯度质量"这个本页默认的验收习惯**。

## Related Concepts

- [[Inverse Rendering]]
- [[Neural Rendering]]
- [[Linear Transport Theory]]（"反向传播 = 一条伴随输运"的母语坐标；9-30 补链）
- [[Adjoint Methods]]（★ 2026-11-13 建页：伴随法的**通用形**——成本与参数维度无关、与状态维度成正比；本页是它在光传输侧的特化）
- [[Neural Physics Simulation]]（**必须分清**：那一页是"用网络替代求解器"，本页是"对渲染器求导"）
- [[Subset Selection]]（**下游**：可微化是"选谁"这个问题的解法家族 D——把选择接进梯度图；那一页给出它与"多阶段重打分""联合期望"的取舍）
- [[Mesh Simplification]]（本页在"简化"上的落点：那一页讲代价模型，本页讲"代价归谁"）

## Game Applications

- **离线/editor-time 优化**：灯光布局优化（LightOpt）、材质参数拟合、LOD 生成
- 资产生成：从照片重建可渲染资产
- **不是**运行时技术——这一点必须分清

## 灯架作为优化对象（2026-11-15 补：LightOpt 读完之后）

**这一族此前在库内的形象是"对渲染器求导，反解几何/材质/光照"；[[LightOpt — Lights Optimization for Real-Time Rendering|LightOpt 2026]] 补上的是"反解一整套灯架，且把它当资产"。** 三处值得单记：

1. **目标不一定是图像，也不一定是辐射场——可以是"辐照度场"。** Lipp 2024 把目标搬到曲面辐射场（HSH），LightOpt 退一步只要**余弦加权的辐照度** $E(x)$。**退这一步换来两件事**：BRDF 从积分内部消失（可微性好得多）、以及**目标与相机彻底无关**（一份解跨视角跨平台）。代价写在它的 Limitations 第一条：**镜面与视角相关效果不在目标里。**
2. **可微需要一个"软支撑"。** 灯的硬作用半径在数学上是阶跃；LightOpt 的做法是**优化时把半径 ×2 + 平滑几何惩罚**，让"灯跨过边界"这件事有梯度。**这是本库第二次遇到"硬截断 vs 软场"**（第一次是 11-12 DTM 的 $\alpha$ 覆盖率在纱线边缘不连续）。**可迁移的判据：任何"影响半径/覆盖掩码/可见性开关"在求导前都必须先被换成连续量，否则梯度是 0 或无穷。**
3. **离散量没有被"变连续"，而是被绕开了。** 见 Learning Gap 第 3 条的补记。

## Important Papers

- [[LightOpt — Lights Optimization for Real-Time Rendering|LightOpt 2026]] ★ **2026-11-15 重写（读者中文全译触发）**——**"目标选辐照度"这条路的产品化样本**：位置/强度/颜色/方向/**个数**同时是变量，动态增删并灯，ToD 关键帧积分；五场景实测活跃灯 −20%~−53%、延迟着色 1.37–2.07×；**三处可直接搬的判断**：① 目标是辐照度 ⇒ **解与相机无关 ⇒ 一份灯架跨平台**（这是它对流程的贡献，比对帧率的大）；② **2× 半径 + 平滑惩罚 = 软支撑**，硬截断不可导；③ **灯数不是预算，每像素重叠数（ALO）才是**——表 2：业界砍 74% 灯数只降 23% 重叠，它砍 53% 灯数降了 56% 重叠
- [[Inverse Rendering for Modeling with Line Primitives]]
- [[Vicini — Path Replay Backpropagation (2021)]] ★ 2026-09-30 入库（**记忆轴起点**：相机侧"常数内存 + 线性时间"；重放三要素）
- [[2026-09-26-Constant-Memory Differentiable Light Tracing]] ★ 2026-09-30 入库（**光源侧补全**：ResLRB / LRB-3-pass；1:1 vs 1:k 边界）——**2026-10-10 会议版到货**（SIGGRAPH Asia 2026 TC："Stochastic Graph Compression"，聚焦 ResLRB）
- [[2026-10-08-Sensitivity as an Arbitrary Output Variable for Differentiable Rendering|Sensitivity AOV]] ★ 2026-10-10 入库（**输出/检查层开线**："**导数是一等渲染输出**"——sensitivity AOV + deferred shading 式"算一次看多处" + 固定/自由双相机分离；同作者 SIGGRAPH Asia 双篇之二）
- [[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization|Lipp 2024 — View-Independent Adjoint Light Tracing]] ★ 2026-10-30 入库（**目标轴开线：优化对象不必是图像**。此前这一族全部"目标住在像素里"，于是相机成了必须承担的变量；Lipp 把目标搬到 **3D 曲面上的辐射场**（顶点 × 半球谐波 HSH），相机整个消失，目标可以直接画在几何上或**直接拿烘焙 lightmap 当目标**。**两条硬结论**：① **AD 在光照追踪上系统性失效**——每条 ray 携带**恒定辐射通量**，与方向/距离无关，所以位置/朝向参数在 AD 里得不到正确导数（这不是精度问题，是方向错）；② 解析伴随的"一次求导 ≈ 一次 primal"在这条路上成立，**内存与时间都与参数个数无关**。**与 [[2026-09-26-Constant-Memory Differentiable Light Tracing]] 是同一条线（光照追踪侧的可微化），但关注点不同：那篇解"常数内存"，这篇解"无相机目标"**）
- [[2026-11-25-Oh — DiffSimp Differentiable Mesh Simplification|DiffSimp 2026]] ★ 2026-11-25 入库（**下游轴开线：可微管线再往前接一环**。此前是"渲染 → 几何"；现在把**简化**插在中间变成"渲染 → 简化 → 几何"。**核心手法**：离散塌陷决策照常做，但固定这一次的历史 $\sigma$，只对塌陷映射 $C_\sigma$ 求导——**"不放宽度量、只对决策之后的连续映射求导"**。工程上靠"把顶点二次型当可微中间变量"把雅可比变成块对角（同轮并行）+ 归约森林（跨轮只求导一次）；**代价**：前向比 CPU QEM 快 61–319×，反向比 autograd 重放快 37–75×，端到端只多 1.25×）

## Personal Knowledge

Current Level: **Hard**

（初始化默认推断，待用户校正）

## Learning Gap

1. 自动微分的反向模式（为什么反向比前向快）
2. 光栅化在哪一步不可导、为什么
3. 离散量（比如"灯要不要留"）如何变成连续可优化量 —— **2026-11-15 补：LightOpt 给了半个答案，而那个"半个"本身是结论。** 它**没有**把"删/并/增"变成可微操作：删用重要性阈值 $\varepsilon_i$、并用 Chamfer 距离 + 三个并列判据（降重叠 / 80% 正面 / 锥角 ≤130°）、增用 DBSCAN 聚类。**原文明确把"让这些操作完全可微"列为未来工作。** 所以工程上的正确姿势是：**连续部分（参数）交给梯度，离散部分（个数/拓扑）交给"判据 + 阈值 + 重跑"，并且承认后者没有最优性保证。** 这条与 11-13 可微物理那篇的"接触不光滑三障碍"是同一族问题。
   **2026-11-25 补：这一条现在有四个答案，可以按"离散决策要不要进循环"分类。**
   | 答案 | 做法 | 代表 | 代价 |
   |---|---|---|---|
   | ① **放松** | 把硬判定换成连续权重（软化覆盖 / 松弛） | Soft Rasterizer 一族 | 前向与真实光栅化脱节 |
   | ② **不碰** | 离散交给判据 + 阈值 + 重跑 | [[LightOpt — Lights Optimization for Real-Time Rendering\|LightOpt]] | 没有梯度信息，靠重跑来搜 |
   | ③ **冻住再求导** | **固定这一次的离散解，只对"决策之后的连续映射"求导** | [[2026-11-25-Oh — DiffSimp Differentiable Mesh Simplification\|DiffSimp]]（固定塌陷历史 $\sigma$） | 梯度只对这一次实现负责（有偏）；需可微前端 |
   | ④ **推迟提交** | 决策不当场做：当概率搜、最后一次硬提交 | [[2026-11-25-Jeong — LMnOP Pruning 3DGS via Leave-Many-Out Importance\|LMnOP]]（8 轮不删任何高斯） | 每次评估要全模型 |
   **③ 与 ④ 的共同点比差别更重要**：**两篇都不在优化循环里改离散决策。** 一个把它冻住，一个把它推迟——**"离散决策不能进循环"，是 2026 年两次独立的工程确认。** 这也回答了本条最初的问法：不是"怎么把离散变连续"，而是"**怎么围着离散做文章**"。
4. **（2026-09-30 新增）"内存/时间"为什么是这门技术的硬约束**——读 [[Vicini — Path Replay Backpropagation (2021)]] 的**结论层**即可（不用碰推导）：重放三要素 + 两笔账分开结算

## Next Learning Step

先做**最小桥**，不要一上来就读可微光栅化论文：

1. 理解"渲染是函数，参数是自变量"这一句话
2. 找一个可微渲染的 toy 例子（比如用可微方式拟合一个三角形的颜色/位置）跑通
3. **（可选加分）读 [[Vicini — Path Replay Backpropagation (2021)]] 的结论层**："重放 = 用确定性换存储"——这是"为什么这类优化必须离线"的最清楚答案
4. 再读 [[LightOpt — Lights Optimization for Real-Time Rendering]] 的问题定义

完整路线见 [[Learning Path — Differentiable Rendering]]。
