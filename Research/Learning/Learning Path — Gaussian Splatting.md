---
type: learning-path
target: "[[Gaussian Splatting]]"
created: 2026-09-09
---

# Learning Path — Gaussian Splatting

## Target

[[Gaussian Splatting]]：Normal → **Easy**

目标不是"知道 GS 是什么"（你已经是），而是补齐三条已识别的缺口，达到能独立判断工程取舍的程度：

1. 为什么排序是它的性能瓶颈
2. 它为什么难进标准（deferred）管线
3. 自适应密度控制在做什么

## Current Knowledge

### Easy（你的地基，本路径全部从这里出发）

- [[Real-Time VFX Performance Budgeting]] — 粒子数/Overdraw/DrawCall 的成本直觉
- [[Niagara]] — 拉格朗日粒子、半透明渲染、排序问题的日常经验
- [[Scalability and Quality Tiers]] — 分档思维
- [[Real-Time Rendering]] — 前向/延迟管线、alpha blending

### Normal（本路径要推动的）

- [[Gaussian Splatting]] ← 目标
- [[Tile-Based Rendering]] — 顺带加固（GS 光栅化器是 tile-based 的）

### Hard（本路径**不**覆盖，学了反而不划算）

- [[Differentiable Rendering]] — GS 训练用到它，但理解 GS 工程不需要理解梯度怎么算。**跳过。**
- [[Neural Rendering]] — 只需知道 NeRF 与 GS 的对比结论（见 Step 2）

## Knowledge Gaps

| 缺口 | 最短桥（从 Easy 出发） |
|---|---|
| 排序瓶颈 | 你懂半透明粒子排序 + Overdraw → GS 是同一问题 × 百万级规模 × per-tile |
| 管线兼容性 | 你懂 deferred 的 G-buffer 单表面假设 → alpha blending 依赖顺序，二者直接冲突 |
| 自适应密度控制 | 你懂粒子发射器的 spawn/kill → 分裂/克隆/剪枝就是"训练时的粒子数量控制" |

## Recommended Bridge

```
Step 0  原论文骨架（~1h）
    Kerbl et al. 2023《3D Gaussian Splatting for Real-Time Radiance Field Rendering》
    只读：§4 表示 + §5 优化/密度控制 + Figure 2 管线图
    锚点：协方差投影到屏幕 = "粒子在屏幕上的 footprint"的推广
        ↓
Step 1  排序与管线（~1h）
    不读新教材，用你的半透明知识反推两个 Gap；
    验证：读 [[TileGS]] 引言，看它的问题陈述是否与你的反推一致
        ↓
Step 2  NeRF 对比（~40min，可选但推荐）
    只需结论：NeRF = 隐式 MLP + ray marching，每像素上百次网络查询
    GS = 显式椭球 + 光栅化投影，一次投影替代全部查询
    材料：NeRF 原论文 Figure 2/3 即可，不读全文
        ↓
Step 3  最新进展（知识库内已有，~1.5h）
    [[TileGS]]（排序）→ [[Compact Neural Appearance]]（存储）→ [[GradRig]]（形变）
    三篇工程化论文各取一个预算语言数字：排序成本 / 192B→28B / 形变在协方差
```

## Recommended Papers

- Kerbl et al. 2023（原论文，arXiv:2308.04079）
- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]
- [[Compact Neural Appearance Models for Efficient Gaussian Splatting]]
- [[GradRig — Differentiable Weights for Skinned Gaussian Splat Deformation]]

## Practical Exercise

**最小思维实验（不用写代码）**：

拿你熟悉的一个 S 级技能特效（比如同屏 3000 粒子的爆发技），回答：

1. 如果把粒子换成高斯椭球来做体积感，排序成本会发生什么变化？（提示：粒子数 × 每高斯 footprint 大小）
2. 这个特效能进你项目的 deferred 管线吗？现在不能的话，GS 版本也不能——说出同一个理由
3. 训练时的"自适应密度控制"对应你工作里的哪个操作？（发射器粒子数的运行时调整）

能把这三题答顺，三个 Gap 就都闭环了。

## Mastery Criteria

沿用 [[Gaussian Splatting]] 笔记中的四条：

- [x] 说清为什么 3DGS 比 NeRF 快（→ Step 2）
- [x] 解释它的排序需求与半透明粒子排序的同构性（→ Step 1）
- [x] 说清它为什么不能直接进 deferred 管线（→ Step 1）
- [x] 判断一个给定场景该用 mesh 还是 3DGS（→ Step 0 + Step 3）

**全部打勾后，告诉我"GS 标 Easy"**，我会同步 frontmatter 并停止基础推送，之后只推建立在 GS 之上的新研究（如动态 GS、GS 角色管线）。

## Estimated Cost

总计 ~4 小时，可分两次：Step 0+1 一次，Step 2+3 一次。

## Status

**Done — 2026-09-11**：用户宣布 "GS 标 Easy"。三个 Gap 经两轮图解讲解闭环，产出讲解笔记：

- [[GS 图解 1 — 协方差与椭球：高斯的形状说明书]]
- [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制]]

基础推送停止；之后只推建立在 GS 之上的新研究（动态 GS、GS 角色管线）。

---

相关：[[Gaussian Splatting]] · [[Personal Knowledge Model]] · [[2026-09-09]]
