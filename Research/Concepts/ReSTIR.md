---
type: concept
user_level: Normal
aliases: [ReSTIR, Reservoir Spatiotemporal Importance Resampling, ReSTIR DI]
prerequisites: [Resampled Importance Sampling, Reservoir Sampling]
first_introduced: 2020
---

# ReSTIR

## Definition

**Reservoir-based SpatioTemporal Importance Resampling**——把 RIS（候选生成 → 加权 → 重采样）装进加权蓄水池采样做成流式算法，再利用蓄水池的**可合并性**让相邻像素与上一帧共享候选池的实时重采样方法族。首篇（ReSTIR DI）解决"数百万动态光源的直接光"（[[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)|Bitterli et al. 2020]]）。

## Core Principle

三个组件各来自一件旧工具，组合点在"采样概率层做时空复用"：

| 组件 | 来源 | 在 ReSTIR 中的角色 |
|---|---|---|
| RIS | Talbot 2005 | 用易抽的源分布 + 目标分布权重，逼近难抽的目标分布 |
| 加权蓄水池采样 | Chao 1982 | 恒定内存的流式实现；**状态 (y, wsum, M) 可代表整条候选流** |
| MIS 平衡启发式 | Veach-Guibas 1995 | 跨像素/跨帧多源 PDF 合并时的无偏化（O(M)） |

复用链：`每像素 M 个候选 → 蓄水池 → 时域合并（motion vector 反投影）→ 空间合并（k 邻居 × n 轮）→ 有效候选数 kⁿM`。

## Prerequisites

- [[Resampled Importance Sampling]] — 三步重采样 + 校正因子（W 的来源）
- [[Reservoir Sampling]] — 流式加权选样容器（可合并性的数学基础）
- [[Veach — Optimally Combining Sampling Techniques for Monte Carlo Rendering (1995)|Multiple Importance Sampling]] — 无偏权重的出处（1995 原文已入库）

## Historical Evolution

```text
1982  Chao 加权蓄水池采样（磁带机流式选样）
1987  Rubin SIR（统计学里的 RIS 原型）
1995  MIS（Veach-Guibas：多技术无偏组合；三件旧工具之一，2026-10-18 入库）
2005  Talbot RIS（图形学形式化；候选池要全存）
2019  Moreau light BVH（实时数据结构派天花板）
2020  ★ReSTIR DI（直接光；"过滤 PDF 而非颜色"）
2021  ★ReSTIR GI（间接光；方向域候选 + Jacobian + 样本验证）
2022  GRIS（统一理论）+ RTXDI SDK（DI/GI/PT/ReGIR）
2025  MegaLights 进 UE 5.6（产品化高峰）
2026  读者框架文档四阶段化：采样→时空复用→重建→失效
```

## Important Papers

- [[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)]]（首篇/本库锚点）
- [[Ouyang — ReSTIR GI Path Resampling for Real-Time Path Tracing (2021)|ReSTIR GI (2021)]]（间接光版：候选改表面点 + Jacobian 修正 + 样本验证）
- [[Lin — Generalized Resampled Importance Sampling Foundations of ReSTIR (2022)|GRIS (2022)]]（统一理论：相关样本+未知 PDF+跨域 shift mapping；M=置信权重，M-cap 成理论必需；ReSTIR PT）
- [[Veach — Optimally Combining Sampling Techniques for Monte Carlo Rendering (1995)|MIS (Veach-Guibas 1995)]]（组合权重的理论根基；balance/power 启发式 + 定理 1/2；§5.4 近似权重许可 = O(M) MIS 的理论前提）
- [[Talbot — Importance Resampling for Global Illumination (2005)]]（理论前置）
- [[Schied — Spatiotemporal Variance-Guided Filtering (2017)]]（重建端正交互补）

## Related Concepts

- [[Global Illumination]] — ReSTIR GI 后进入 GI 八代谱系的"光追/复用"代
- [[Temporal Stability and Artistic Intent]] — 时域复用的失效与拖影问题在此汇聚
- [[Real-Time VFX Performance Budgeting]] — 参数集（M/k/n/N/clamp）本身是可分档预算表

## Technologies

- RTXDI（NVIDIA SDK）— `Reservoir.hlsli` 与首篇 Alg. 2 逐字段对应
- MegaLights（UE 5.6+）— stochastic direct lighting 产品线

## Game Applications

- 大规模动态光源场景（城市夜景 / 战斗技能灯光海 / emissive 资产密集场景）
- 需要每像素光照预算与"算总账"（而非限灯数）哲学的管线

## Personal Knowledge

- **Normal（结论层）**；推导层（GRIS 附录 A 定理链：权重等价/分布收敛/渐近方差/MIS 界）为 Hard 桥，前置已就位（RIS 方差劈分 Normal）。

## Learning Gap

- ~~GRIS 2022 未入库~~（**2026-10-17 入库**：四级推广阶梯 + 收敛条件清单 + M-cap 语义 + ReSTIR PT）；剩余：GRIS 附录 A/S.5 推导层未读；RTXDI `Reservoir.hlsli` 15 分钟只读未执行（挂起第 3 周）。

## Next Step

1. ~~读 [[Ouyang — ReSTIR GI Path Resampling for Real-Time Path Tracing (2021)|ReSTIR GI]] 结论层~~（2026-10-16 入库）；
2. ~~GRIS (2022) 入库补统一理论段~~（2026-10-17 完成——家族四篇 DI/GI/GRIS/RTXDI 结构齐了）；
3. 剩余推导桥：GRIS 附录 A 定理链（可与 DI §4.2 + GI Jacobian 推导组一个"复用理论推导周"）；
4. RTXDI `Reservoir.hlsli` 只读（固定提交已给，材料零成本）。
