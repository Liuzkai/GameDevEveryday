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
1997  ★ MLT（Veach-Guibas）：**"复用一条路径"的第一次**——路径成为 Markov 链状态，样本等权；但样本串行相关，进不了 GPU ★ 2026-10-30 入库
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
- [[Veach — Metropolis Light Transport (1997)|Metropolis Light Transport (1997)]] — **复用谱系的上一站**（见下）
- [[2026-10-30-Liao — From Walks to Skeletons A Reusable Transport Architecture|Walks to Skeletons (2026)]] — 复用谱系的另一支：复用"轨迹图"而不是"样本"

## 复用谱系：三站，同一条语法（2026-10-30 补）

把 MLT、ReSTIR、与今年的"骨架"放在一起看，**它们回答的是同一个问题——"上一次算出来的东西，这次还能不能用"**——而三者的差别全在**"被复用的对象"和"为什么它还有效"**：

| | MLT（1997） | ReSTIR（2020） | Walks to Skeletons（2026） |
|---|---|---|---|
| 复用的对象 | **一条路径** | **一个蓄水池（若干光源样本 + 权重）** | **一整张轨迹依赖图** |
| 复用方式 | 在路径空间里**变异**（走一小步） | 时域反投影 + 空间邻居**合并** | **挂接**到某个已完成节点 |
| 为什么还有效 | 路径空间的 $f$ **分片光滑** | 相邻像素/帧的**可见性相近** | WoS 的**马尔可夫性**（状态定了，后续与来路无关） |
| 合法性怎么保证 | 接受概率里的重要性比 | 无偏权重 $W$ 的归一化 | 测度变换 + 重要性比上界 $\Lambda$ |
| 可用性怎么保证 | 变异的接受率 | 有效样本数 $M$ / $M$-cap | $\Gamma_q(p)\le\Lambda_{\text{attach}}$ |
| 并行性 | ❌ **串行相关**（这是它一生的短板） | ✅ 可并行 | ✅ 可并行 |
| 失效方式 | 遍历性破缺（到不了另一片路径空间） | 时域拖影 / 样本退化 | **只能保住旧输运，发现不了新贡献** |

**三站共享的语法，可以当成一条判据背下来**：

> **任何"复用上次结果"的方案，都必须能写出一个重要性比；写不出来的都是有偏的，比值没有上界的都会炸方差。**

**而三站的失效方式各不相同，这更值得记**：MLT 卡在"到不了"（连通性），ReSTIR 卡在"太旧了"（时域失效），骨架卡在"看不见新东西"（覆盖率）。**同一套语法，三种死法——说明语法只保证合法性，不保证有用。**

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
