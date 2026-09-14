---
type: learning-path
target: "[[Differentiable Rendering]]"
status: Learning
created: 2026-09-07
---

# Learning Path — Differentiable Rendering

## Target

[[Differentiable Rendering]]

**为什么值得学它**：它是 [[LightOpt — Lights Optimization for Real-Time Rendering]] 的前置。而 LightOpt 直接对应你手上"动态灯光上限"（S≤3/A≤2/B≤1/C=0）这个预算维度。学会它，你能把经验值变成推导值。

## Current Knowledge

### Easy（可直接作为桥墩）

- [[Real-Time VFX Performance Budgeting]]
- [[Niagara]]
- [[Scalability and Quality Tiers]]
- [[Real-Time Rendering]]
- 光栅化管线的工程理解

### Normal

- [[GPU-Driven Rendering]]
- [[Tile-Based Rendering]]

### Hard（目标与中间层）

- [[Differentiable Rendering]] ← 目标
- [[Neural Rendering]]
- [[Inverse Rendering]]

## Knowledge Gaps

1. 自动微分的反向模式
2. 光栅化在哪一步不可导
3. 离散量如何松弛成连续可优化量
4. loss 设计（外观误差用什么度量）

## Recommended Bridge

按**这个顺序**，不要跳：

1. **链式法则与计算图**（1-2h）— 只需要理解"反向传播 = 链式法则 + 复用中间结果"
2. **光栅化的梯度断点**（1h）— 问自己：coverage 判定、深度测试、可见性跳变，哪个不可导？答案是全都不可导
3. **软化与随机化**（2h）— 理解两类近似：软化边缘 vs 随机采样估计
4. **最小可微渲染实验**（半天）— 用现成框架（PyTorch3D / nvdiffrast）拟合一个三角形的位置与颜色，跑通一次梯度回传
5. **离散松弛**（1h）— 理解"灯要不要留"如何用连续权重 + 稀疏正则表达
6. **读 [[LightOpt — Lights Optimization for Real-Time Rendering]] 的问题定义** — 此时你应该能看懂它在优化什么、为什么必须离线

## Recommended Papers

- [[LightOpt — Lights Optimization for Real-Time Rendering]]（目标应用，先读问题定义即可）
- [[Inverse Rendering for Modeling with Line Primitives]]（进阶）

## Practical Exercise

**最小原型**：在一个已有的 NGR 技能场景里，手工挑 5 盏动态灯，尝试回答：
- 关掉其中任意一盏，画面差异有多大？（用人眼，不是 PSNR）
- 有没有两盏灯在功能上重叠？
- 如果只能留 2 盏，留哪两盏？

把你的答案记下来。等学完可微优化，回来看这个答案是否与优化器一致。**这个对照实验本身就是 mastery 判据。**

## Mastery Criteria

能同时做到以下四点，即可把 [[Differentiable Rendering]] 标记为 Easy：

- [ ] 说出渲染管线中至少 3 个梯度断点，并各给出一种可导近似
- [ ] 解释为什么这类优化必须离线做
- [ ] 说清离散决策如何变成连续优化
- [ ] 判断一个给定的 VFX/渲染问题是否适合用可微优化求解

## Status

**Learning** — 2026-09-07 建立，尚在 Bridge 第 0 步。
