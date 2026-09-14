---
type: concept
title: "GPU-Driven Rendering"
user_level: Normal
tags: [engine, architecture, performance]
---

# GPU-Driven Rendering

## Definition

把传统由 CPU 承担的可见性判定、剔除、LOD 选择、批处理与 DrawCall 生成，整体迁移到 GPU 上执行的渲染架构。CPU 只提交少量间接命令。

## Core Principle

```
传统：CPU 遍历对象 → 剔除 → 发 DrawCall      （CPU 是瓶颈）
GPU-Driven：GPU compute 剔除 → 生成可见列表 → ExecuteIndirect   （CPU 只发几个命令）
```

收益：DrawCall 数量不再受 CPU 限制；可以做更细粒度的剔除（per-cluster、per-triangle）。

## Prerequisites

- [[Real-Time Rendering]]
- [[GPU Architecture]]
- Compute Shader / Indirect Draw

## Evolution

```
CPU 视锥剔除
        ↓
GPU 视锥 + 遮挡剔除（HZB）
        ↓
Virtual Geometry / Nanite（per-cluster LOD + 软件光栅化小三角形）
        ↓
GPU-Driven 全流程（含动画、VFX 剔除）
```

## Related Concepts

- [[Real-Time Rendering]]
- [[Virtual Geometry]]
- [[Scalability and Quality Tiers]]

## Game Applications

- 开放世界大规模实例化
- [[AAA Real-Time VFX]]（VFX 的 GPU 侧剔除与 LOD）

## Important Papers

- [[LightOpt — Lights Optimization for Real-Time Rendering]]

## Personal Knowledge

Current Level: **Normal**

## Mastery Criteria

- [ ] 说清 ExecuteIndirect 与传统 DrawCall 的差别
- [ ] 解释 HZB 遮挡剔除的原理与代价
- [ ] 说清为什么它对 DrawCall 预算的意义
- [ ] 判断 VFX 该走 GPU-Driven 还是传统路径

## Next Step

与你的 DrawCalls 预算项绑定：明确列出哪些 VFX 场景的 DrawCall 是当前架构下的硬约束，哪些可以被 GPU-Driven 消掉。
