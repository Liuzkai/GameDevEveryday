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

## Related Concepts

- [[Inverse Rendering]]
- [[Neural Rendering]]
- [[Linear Transport Theory]]（"反向传播 = 一条伴随输运"的母语坐标；9-30 补链）

## Game Applications

- **离线/editor-time 优化**：灯光布局优化（LightOpt）、材质参数拟合、LOD 生成
- 资产生成：从照片重建可渲染资产
- **不是**运行时技术——这一点必须分清

## Important Papers

- [[LightOpt — Lights Optimization for Real-Time Rendering]]
- [[Inverse Rendering for Modeling with Line Primitives]]
- [[Vicini — Path Replay Backpropagation (2021)]] ★ 2026-09-30 入库（**记忆轴起点**：相机侧"常数内存 + 线性时间"；重放三要素）
- [[2026-09-26-Constant-Memory Differentiable Light Tracing]] ★ 2026-09-30 入库（**光源侧补全**：ResLRB / LRB-3-pass；1:1 vs 1:k 边界）

## Personal Knowledge

Current Level: **Hard**

（初始化默认推断，待用户校正）

## Learning Gap

1. 自动微分的反向模式（为什么反向比前向快）
2. 光栅化在哪一步不可导、为什么
3. 离散量（比如"灯要不要留"）如何变成连续可优化量
4. **（2026-09-30 新增）"内存/时间"为什么是这门技术的硬约束**——读 [[Vicini — Path Replay Backpropagation (2021)]] 的**结论层**即可（不用碰推导）：重放三要素 + 两笔账分开结算

## Next Learning Step

先做**最小桥**，不要一上来就读可微光栅化论文：

1. 理解"渲染是函数，参数是自变量"这一句话
2. 找一个可微渲染的 toy 例子（比如用可微方式拟合一个三角形的颜色/位置）跑通
3. **（可选加分）读 [[Vicini — Path Replay Backpropagation (2021)]] 的结论层**："重放 = 用确定性换存储"——这是"为什么这类优化必须离线"的最清楚答案
4. 再读 [[LightOpt — Lights Optimization for Real-Time Rendering]] 的问题定义

完整路线见 [[Learning Path — Differentiable Rendering]]。
