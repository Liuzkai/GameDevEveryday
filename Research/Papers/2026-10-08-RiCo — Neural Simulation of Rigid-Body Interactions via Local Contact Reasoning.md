---
type: paper
title: "RiCo: Neural Simulation of Rigid-Body Interactions via Local Contact Reasoning"
authors: [Ruixiang Ouyang, Guanren Qiao, Fansen Meng, Yueci Deng, Ruixing Jin, Kui Jia, Guiliang Liu]
year: 2026
published: "2026-10-08（arXiv v1, 2610.12333）"
venue: "arXiv Preprint（CUHK-Shenzhen × DexForce × SCUT × Shenzhen Loop Area Institute；cs.CV；本次经 Fri 10-9 listing 组捕获）"
url: "https://arxiv.org/abs/2610.12333"
code: ""
project_page: ""
category: [neural-simulation, rigid-body, contact-dynamics, world-models, physics, point-cloud]
importance: B+
historical_importance: 0
game_relevance: 4
production_readiness: "Research（学术原型；MOVi 基准 + 真实多球台球实验；无代码/无项目页）"
user_level: "Hard（推导层）/ Normal（结论层：局部接触推理 + 接触保真指标）"
status: unread
aliases: [RiCo, Rigid-body Contact Reasoning, 刚体接触推理, 局部接触神经仿真]
tags: [neural-simulation, rigid-body, contact-dynamics, physics, world-models]
---

# RiCo: Neural Simulation of Rigid-Body Interactions via Local Contact Reasoning（Ouyang et al. 2026）

> **入库 2026-10-10（Run 32）。** **CUHK-Shenzhen（Kui Jia / Guiliang Liu 组）× DexForce × SCUT × Shenzhen Loop Area Institute**。经 **Fri 10-9 listing 组**捕获（10-08 提交，10-09 公告）。
> **一句话定位**：**把"接触是局部的"这一物理事实写成网络结构**——跨物体信息交换只允许发生在邻近表面点之间（稀疏局部接触邻域），物体内部再用 point Transformer 把局部接触"汇总"成刚体运动。**[[Neural Physics Simulation]] 的"接触维度"开线**。
> **库内位置**：物理仿真线（[[2026-10-07-Simulation Methods for Multiphysics Phenomena in Visual Computing|Multiphysics 地图]] 的"学习式仿真"侧）第 N 样本；与 [[2026-10-06-PhysLDM — Latent Diffusion for High-Fidelity Deformable Simulation|PhysLDM]]（可形变）、[[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]]（毛发）成"神经仿真三形态"（刚体 / 可形变 / 发丝）。

## TL;DR

**刚体接触本质上只发生在局部——但现有神经仿真器要么在网格上做消息传递（贵），要么在物体间做稠密注意力（更贵）。RiCo 让"跨物体推理"只发生在局部接触邻域，把"全局影响"交给物体内的自注意力：**

```text
三阶段：
① 稀疏接触邻域  每个表面点只收 K 个邻近外部候选（AABB 粗筛 + ρ 半径内 KNN）
                每个候选编成 14 维交互描述符（方向 u / 符号距离 δ / 法线 / 相对位移 /
                物理属性 φ̃ = [1/m, μ, e] / 静态-动态标志 σ）
② 对象内推理    距离加权融合（可学习温度 τ）→ 共享 point Transformer（gated MHSA）
                ——跨物体信息"只在①进来"，物体内部传播"怎么一起影响刚体运动"
③ 锚点解码      FPS 选 A 个锚点 → 预测二阶位移（Verlet 外推）→ Kabsch 对齐恢复刚体变换
                监督：直接预测 + 刚性投影 双向 loss（smooth-L1）
```

**结果（MOVi 基准）**：100 帧位置/朝向 RMSE **降低 31–38%**（vs HOPNet / RigidFormer）；**接触保真**：ΔPTR（穿透时间比差）**11.0%**（vs HOPNet 42.7% / RigidFormer 44.8%）、ΔMPD（平均穿透深度差）**2.22 mm**（vs ~287 mm / ~268 mm）；零样本泛化到 **270 物体**场景；真实多球台球实验给出 sim-to-real 初步证据。

## Problem

**碰撞让刚体动力学"非光滑"**（速度突变），而神经仿真器要么学不出这种突变，要么用错误的代价学：

- **端到端 world model**：预测整个场景/物体的状态演化——**但刚体接触的物理作用域是局部的**（"only nearby surfaces can directly exchange contact forces"），全局推理把算力花在了不交换力的点对上；
- **网格法（message passing）**：用网格连通性传播信息（FIGNet / HOPNet）——有效但**贵且依赖网格**（拓扑结构、高阶关系构建耗时）；
- **点云法（RigidFormer 2026）**：去掉网格、用对象级注意力——**但"哪些点是势接触点"这件事没被编码**（"point clouds do not inherently encode which surface points may come into contact"），对象级表示把细粒度接触信息压粗了。

**RiCo 的答案**：保持点云表示，但**显式构造"稀疏局部接触邻域"**——让网络结构对齐物理作用域。

## Historical Context

```text
2016  Interaction Networks / 2018 GNS —— 学习式物理的先声（归纳偏置）
        ↓
2018-2020  particle-based simulators（Li et al. / Sanchez-Gonzalez et al.）——
        粒子表示 + 图网络；接触仍是"全图传播"
        ↓
2020  MeshGraphNets（Pfaff et al.）—— 网格消息传递成为主流结构
        ↓
2023  FIGNet —— 刚体接触：在网格"面"之间构造交互（保留接触周围几何）
2025  HOPNet —— 高阶拓扑：顶点/边/三角形/接触/对象五种实体统一表示
2026  RigidFormer —— 去网格：点云 + 对象级注意力（细粒度接触丢失）
        ↓
★ 2026 RiCo：点云 + 【点级】稀疏局部接触邻域——"跨物体推理的边界 = 物理作用的边界"
```

**一句话点评**：这条线的主题是"**表示越来越轻（网格→点云）、推理越来越准（对象级→局部接触级）**"——RiCo 补上的正是"局部性"这个物理先验。

## Previous Work

- **FIGNet（Allen et al. 2023）/ HOPNet（Wei & Fink 2025）**：网格系最强基线（HOPNet 有官方 checkpoint，被用作主要对照）；
- **RigidFormer（Dou et al. 2026）**：点云系最强基线（无官方代码，作者重实现）；RiCo 沿用其"锚点 + 刚体变换恢复"框架（"follow prior anchor-based formulations"）；
- **VPD / HCMT / PointWorld / RoboFlow4D**：其他对照与相邻工作（VPD 点级扩散、HCMT 时序 transformer）；
- **Kabsch 算法（1976）**：从锚点对应恢复最优刚体变换——古董工具继续打工。

## Core Idea

**"跨物体信息交换的边界，应当画在物理作用边界上。"**

| | FIGNet / HOPNet（网格） | RigidFormer（点云+对象级） | **RiCo（点云+点级局部）** |
|---|---|---|---|
| 跨物体信息载体 | 网格面 / 高阶实体 | 对象级 token | **表面点 × 稀疏接触邻域（K 个）** |
| "哪些点会接触" | 由网格结构隐含 | 未编码 | **显式构造（AABB + ρ 半径 KNN）** |
| 推理代价 | 网格规模 | 对象数² | **K × 点数（K ∈ {1,4,8} 即够）** |
| 细粒度接触细节 | 保留（但贵） | 压粗 | 保留 |

**关键设计：两条信息通路分离**——
1. **横向（跨物体）**：只经"稀疏接触邻域"进入（每点 K 个外部候选的动态/几何/物理属性）；
2. **纵向（物体内）**：point Transformer 让每个点看到"自己物体全表面的接触分布"，从而判断"这些局部接触合起来如何驱动平移与旋转"。
——"**跨物体不做稠密注意力**"是全文最贵的一句话（"avoids dense point-level interactions across the entire scene"）。

## Technical Approach

1. **接触邻域构造**：对象级 AABB 粗筛（最小间距 ≤ 接触半径 ρ）→ 保留对象的所有候选点 + 静态环境候选（解析投影查询）→ 每点保留 ρ 内 K 近候选；
2. **14 维交互描述符** $\eta$：方向 $u$ / **符号距离 $\delta$（沿候选局部切平面的带符号偏移——区分"哪一侧"）** / 候选法线 / 相对位移（接近-分离与切向滑动） / 候选物理属性 $\tilde{\phi}=[1/m,\mu,e]$ / 静动标志 $\sigma$；
3. **15 维点状态** $s$：相对质心位置 / 差分位移 / 法线 / 本物体物理属性 / 相对初始位置累计漂移；
4. **融合**：RMSNorm($\phi_s(s) + \sum_k \alpha_k \phi_c(\eta_k)$)，权重 $\alpha_k = \text{softmax}(-d_k/\tau)$（距离加权 + 可学习温度）——"优先邻近、保留多候选"；
5. **对象内 Transformer**：B 块 gated multi-head self-attention（sigmoid 门控调制注意力输出）+ FFN——**参数跨物体共享**；
6. **锚点解码**：FPS 固定 A 个锚点索引（rollout 全程不变）→ 锚点到全点注意力 → 预测**二阶位移** → Verlet 式外推（$2x_t - x_{t-1} + \Delta^2$）→ **Kabsch** 恢复 $(R,b)$ 作用于全体点/质心/法线；
7. **训练**：smooth-L1 双向监督——直接预测的二阶位移 + 刚性投影后的二阶位移 vs 真值（"rigid-projected 版本"迫使网络输出的锚点位移与某个刚体变换一致）。

## Key Contribution

1. **"局部接触邻域 + 对象内推理"的两级结构**：把物理局部性写成归纳偏置——跨物体只在接触邻域、影响汇总在物体内部；
2. **接触保真度指标体系**：`ΔPTR`（相对真值的穿透时间比差）+ `ΔMPD`（相对真值的平均穿透深度差）——**"轨迹精度"看不见的接触质量被单独度量**（见下，这是本篇最可迁移的一条）；
3. **零样本规模化 + sim-to-real 初步证据**：小场景训练 → 270 物体；多球台球实拍。

## Why It Works

- **结构对齐物理**：接触力只能由邻近表面传递——网络的信息通路与物理的作用通路同构（"network bias = physics"）；
- **符号距离是"便宜的接触先验"**：不带符号的距离只说"多近"，带符号才说"在哪一侧"（是否已侵入）——消融显示它**主要影响接触保真而非轨迹精度**（ΔPTR 11.0%→23.6%、ΔMPD 2.22→7.86 mm）；
- **锚点 + 刚性投影**：把"每点自由回归"降维成"一个小刚体变换的参数估计"（A 个锚点 → 6 自由度），运动学约束（刚体性）直接进结构；
- **K 不需要大**：K∈{1,4,8} 表现稳定（K=4 最优 ΔPTR）——**局部接触的"信息量"集中在最近邻**，验证了稀疏性假设。

## Limitations

- **MOVi 是合成数据**（简单几何：球/基本形/复杂形三档），真实场景只有多球台球一个实验（"preliminary evidence"）；
- **刚体假设**：不处理形变体（对照：[[2026-10-06-PhysLDM — Latent Diffusion for High-Fidelity Deformable Simulation|PhysLDM]] 走可形变）；
- **泛化方向不对称**：从几何多样（MOVi-B）训到简单分布（Sphere/A）非常好（0.085 m），但从简单训到复杂仍吃力（"generalization to more complex geometries remains dependent on the diversity of the training distribution"）——**训练分布的几何覆盖是前提**；
- 回归式预测的固有边界：混沌/多接触拥塞场景的分布建模问题未触及（对照 PhysLDM"混沌判据"）；
- 无代码 / 无项目页。

## Game Development Relevance

- **游戏物理的"学习式预测"方向**：当前用途是**embodied AI / world model**（预测交互演化），不是替代 Chaos/Havok 的确定性求解器——但"**接触局部性 + 刚体约束进结构**"这两条设计对任何"物理预测模块"都通用；
- **"接触保真"指标体系可直接借用**：游戏里刚体互穿（穿模）是常见 bug——`ΔPTR / ΔMPD` 这种"相对真值/相对上一版"的穿透统计，比"看视频有没有穿帮"严谨得多；**做物理资产/碰撞代理验证时可照搬这个口径**（与 [[2026-09-23-CuACD — A Fully GPU-Resident Approximate Convex Decomposition|CuACD]] 的凸分解验证需求同向）；
- **对分档的远期含义（推断）**：学习式物理若进运行时，"物理精度"会成为新的档位维度——但当前距生产远（合成基准 + 单实验），**只记入观察清单**。

## Unreal Engine Relevance

- 无直接 UE 映射（研究形态）。原理映射：Chaos Physics 的"碰撞响应预测"、物理动画（ragdoll / 次级运动）的远期神经化；**若未来做"物理 LOD"**（远处物体用学习模型预测、近处用真解算），RiCo 的"局部接触 + 刚体约束"就是该形态的一种模板；
- 与 [[2026-10-07-Simulation Methods for Multiphysics Phenomena in Visual Computing|Multiphysics 教程]] 的衔接：教程指出 PBD/XPBD 是游戏物理的约束求解主线——**RiCo 是"约束求解"的反面教材的另一极**（学出来的接触响应 vs 投影出来的约束满足），对照阅读能更快定位"学习式方法在什么条件下才会进入引擎"。

## Technology Evolution

```text
【接触建模的表示谱系】（本库首次成线）
网格面交互（FIGNet 2023）→ 高阶拓扑（HOPNet 2025）→ 点云对象级（RigidFormer 2026）
    ↓ 每代都在"减去一层结构"
★ 点云点级局部接触（RiCo 2026）
    ——"减结构"的尽头是"把物理局部性直接当结构"

【神经仿真三形态】（库内现状）
刚体（★ RiCo）· 可形变（PhysLDM）· 发丝（Neuroll）
    ——三种表示，同一条问题线索："哪一步交给学、哪一步保留物理"
```

## Relationships

### Based On

- **RigidFormer（Dou et al. 2026）**——锚点框架直接沿用（"follow prior anchor-based formulations"）；RiCo 是它的"点级化修正"；
- **Kabsch 算法（1976）**——刚体对齐恢复的数学工具。

### Extends

- **FIGNet / HOPNet**——同为"接触感知的神经仿真"，RiCo 把"接触结构"从网格/高阶实体换成"稀疏局部邻域"（更轻且更显式）。

### Related

- [[Neural Physics Simulation]]——归属概念（接触维度开线）；
- [[2026-10-06-PhysLDM — Latent Diffusion for High-Fidelity Deformable Simulation|PhysLDM]]——同域对照：PhysLDM 处理可形变体 + 混沌分布问题；RiCo 处理刚体 + 局部接触；**两者的"判据"互补**（混沌判据 vs 局部性判据）；
- [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]]——对照："镜像经典 I/O"（Neuroll）vs "结构对齐物理"（RiCo）——**神经仿真"偷懒的两种姿势"**；
- [[2026-09-14-Gaussian Light Transport|渲染侧"局部性"对照]]——注意别过度连接：光传输的局部性（光路局部介质）与接触的局部性（表面局部作用域）是同构直觉，但领域不同，仅作思维工具。

### Followed By

- （待观察）"局部接触"结构是否被引入可形变/流体域（如"只有相邻粒子交换约束"的显式化）。

## Personal Knowledge State

- **user_level: Hard（推导层）/ Normal（结论层）**。前置：粒子机制（[[Particle Systems]] Easy）+ Multiphysics 教程（[[2026-10-07-Simulation Methods for Multiphysics Phenomena in Visual Computing|地图层]]）——**结论层（"局部接触推理"是什么、为什么省、指标怎么看）可直接读**；注意力/训练细节与实现属于 [[Neural Physics Simulation]] 的既有缺口；
- **读法建议（≈20 分钟）**：Abstract → Figure 1（三种接触建模对照：mesh-based / RigidFormer / ours）→ §3.1 描述符与邻域（两张向量公式）→ Table 1 数字 → Fig. 3(a) 接触保真（+ 消融）→ §4.5 sim-to-real；
- **与用户的关系**：物理域自 Multiphysics 教程（10-08）后的第二个材料——**"接触"是游戏物理最常见的问题域**（碰撞、堆叠、破坏物块）。

## Learning Value

1. **"结构对齐物理"训练法**：把"作用域"直接写成信息通路的边界——比"给网络更多数据"便宜；与 [[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]] 的"预算定在结构上"同族；
2. **指标看不见的价值（本日最值钱判据）**：signed distance 消融——**轨迹 RMSE 几乎不变、接触保真度翻倍变差**。教训：*评估一个输入/正则有没有用时，要先问"它对应的价值由哪个指标度量"*——否则会像本文之前的所有基线一样"在错误指标上看起来不错"；
3. **"符号信息"的又一次出场**：从距离到符号距离（哪一侧）一字之差——**几何问题里"带方向的量"总是比"纯量"多带一层信息**（对照渲染里的"有向距离场"）。

## Visualization

（本节点暂不新增图解——三阶段管线与局部性对照已在 TL;DR 与 Core Idea 表表达。）

## Notes

- **窗口记录**：Fri 10-9 listing 组捕获（10-08 提交、10-09 公告）；编号 2610.12333 为 API 侧"10-8 提交"段——本组成员，**API 通道本日对 10-9 提交无输出（见 Daily §窗口核对，新增观察项）**；
- **数字口径**：31–35%（位置）/ ~38%（朝向）为"vs 最强基线"的相对降幅；绝对 RMSE 见 §TL;DR（100 帧 0.115 m / 11.40° MOVi-A）。
