---
type: learning-path
target: "[[Neural Rendering]]"
status: Learning
created: 2026-09-07
---

# Learning Path — Neural Rendering

## Target

[[Neural Rendering]] → [[Generative Rendering]]

**为什么值得学它**：不是让你去训模型，而是让你能判断"某个神经方法能不能进管线"。对你而言，这个判断力比模型细节重要得多。

## Current Knowledge

### Easy

- [[Real-Time Rendering]]
- [[Real-Time VFX Performance Budgeting]]
- [[Niagara]]
- [[Scalability and Quality Tiers]]
- TAA / 时域累积（作为锚点）

### Normal

- [[Neural Upscaling and Frame Generation]]
- [[GPU-Driven Rendering]]
- [[Tile-Based Rendering]]
- [[Gaussian Splatting]]
- [[Temporal Stability and Artistic Intent]]

### Hard

- [[Neural Rendering]] ← 目标
- [[Generative Rendering]]
- [[Neural Global Illumination]]
- [[Differentiable Rendering]]

## Knowledge Gaps

1. 采样/重建/滤波的信号处理直觉
2. 扩散模型与一步生成模型的差别
3. "帧预算内跑神经网络"的工程感受（tensor core、量化、算子融合）
4. conditioning 具体把什么喂给模型

## Recommended Bridge

**从你已经掌握的东西出发，不要从论文出发：**

1. **TAA → 超分**（1h）
   你已经懂 TAA。超分就是"带更强先验的时域重建"。先把这层说通。

2. **信号处理的三个词**（1h）
   采样、混叠、重建滤波。懂这三个词，就能读懂所有超分论文的一半。

3. **conditioning 清单**（1h）
   读 [[DLSS 5 — Generative Neural Rendering]] 的 condition 列表：rendered frame、motion vectors、temporal state、artistic-direction values。**这是你作为渲染工程师唯一真正需要关心的接口层**，也是你现有知识能直接覆盖的部分。

4. **一步 vs 多步**（2h）
   传统扩散要几十步去噪；一步模型（consistency / MeanFlow 类）直接出结果。理解为什么前者上不了实时。

5. **时域稳定性为什么难**（1h）
   见 [[Temporal Stability and Artistic Intent]]。

6. **再回头看 [[Generative Rendering]] 的三代对比表** — 此时应该能自己填出来。

## Recommended Papers

按顺序：

1. [[Neural Upscaling and Frame Generation]]（技术笔记，你已 Normal）
2. [[DLSS 5 — Generative Neural Rendering]]（读 condition 与 limitation，不读模型结构）
3. [[Lightweight Attention-based Indirect Illumination (AMD)]]

## Practical Exercise

**审查项（马上可做）**：挑一个你负责的高频运动特效，在开启超分/帧生成时：

- Niagara 是否正确输出了 motion vector？
- 粒子是否有正确的 depth / albedo buffer？
- 快速镜头运动下是否有拖影/沸腾？

这不需要任何新知识，但会直接告诉你：**你的特效在神经渲染时代是否会翻车。**

## Mastery Criteria

- [ ] 说清"重建参考"与"生成外观"的本质区别
- [ ] 列出神经渲染进管线的三个硬门槛，并说明各自为什么难
- [ ] 判断一个给定的神经方法在给定硬件上是否有可能实时
- [ ] 说清你的 VFX 需要向神经层提供哪些 buffer、质量如何保证
- [ ] 在 UE 中说明一个神经渲染 stage 大致接在管线的什么位置

## Status

**Learning** — 2026-09-07 建立。第 3 步（conditioning 清单）可以立刻开始，不需要任何前置。
