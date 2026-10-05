---
type: paper
title: "Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD"
authors: [Haipeng Wang]
year: 2026
published: "2026-10-02"
venue: "Eurographics 2027 投稿（preprint；CGF Vol.46 Issue 2 版式；arXiv:2610.03162）"
url: "https://arxiv.org/abs/2610.03162"
code: ""
project_page: ""
category: [rendering, gaussian-splatting, lod, scalability, budget, optimal-transport]
importance: A-
historical_importance: 2
game_relevance: 5
production_readiness: Prototype
user_level: Normal
status: unread
---

# Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD

> **入库 2026-10-05（Run #27）。** 作者 Haipeng Wang（东软 Neusoft）；EG 2027 投稿预印本（28 页 + 附录，10-2 提交、10-5 窗口捕获，**"未公告先见"**）。它是本库 [[Gaussian Splatting]] 的新一条线——**LOD / 预算线**——也是"预算"主题第一次以**理论定律**的形态出现：**误差 ∝ N^(-2/d)（Zador 定律）**，且给出**可测量、可认证**的容量下限。

## TL;DR

- **问题**：city-scale 3DGS 装不进消费级 GPU（百万 primitive / GB 级内存），且**"到底需要多少 primitive"无人能答**——训练靠"先长到巨大再压缩"、部署靠"随机子采样"，两边都在浪费；
- **理论**：把渲染写成**相空间（位置×方向）最优传输**问题 → **容量下限**（Zador 定律）：误差随预算 N 以 **N^(-2/d_ps)** 下降；表面场景相空间支撑 4 维 → **误差 ∝ N^(-1/2)**（"已发表压缩器平均斜率仅 -0.04——它们赚的是冗余，不是容量"）；
- **两个方法**：**Factoring LOD**（部署侧：任意训练模型 → 矩匹配聚合的因子树，**单一标量参数 c** 按视角选 LOD，一个模型扫连续内存-质量曲线）+ **Budget-Centered Training**（训练侧：直接在 floor 的 knee 处训练，**"生来就是正确尺寸"**）；
- **实测**：官方 MatrixCity 城市数据，**单张 4080 SUPER 消费级 GPU、1080p 原生、全 SH、71 fps**（vs Octree-GS 同协议 11.5 fps，**6.2×**）——"首个 post-hoc、full-SH、消费级 GPU 的城市级结果"；同一模型扫 **8/4/2 GB 三档显存目标**（1288/824/407 MB）。

## Problem

三个结构性障碍（原文列举）：

1. **训练侧**："标准配方 = 训练一个巨大模型再压缩"——primitives 为一个容量优化、后被扔掉一半，**花在被删部分上的训练时间是纯浪费**；
2. **部署侧**：流式渲染器（如 GPS）唯一的降级模式是**随机子采样**——同内存下质量惨败于 LOD（本文实测：859 MB 档，LOD -2.12 dB vs 随机 -7.08 dB）；
3. **根本问题**：**"我到底需要多少 primitive？"——只能靠习惯回答**，因为"有原则的预算-误差下限"在训练与部署两端都不存在。

## Core Idea

**把渲染写成相空间的率-失真（rate–distortion）问题**：

> "一个训练好的模型 = 率-失真曲线上的一个点；它的 primitive 数是它的**预算**，误差是预算的函数。"

两个数学半场 + 三个选择规则：

| 组件 | 内容 | 回答的问题 |
|---|---|---|
| **容量下限**（Zador 定律） | $\mathcal{R}(N)\asymp C_{d_{ps}}\|\lambda\|\,N^{-2/d_{ps}}$；表面场景 **-1/2** | 不可避免的误差是多少 |
| **合并许可**（Wang–Zahl 2025） | 加权并集体积估计（Kakeya 三维猜想的解决定理）→ **局部"过度覆盖"处可合并**，损失有界（只差常数） | 哪些组冗余 |
| **尺度截断** | 投影脚印 < 像素尺度的原子可丢弃（footprint t = c·d/f） | 什么可以丢 |
| **预算分配** | 各尺度带预算**指数衰减** $m_j\propto 2^{-j\gamma}$；per-tile $N_j^*\propto K_j s_j/d_j$ | 固定预算怎么分 |

**三条速率断言的族谱**（论文附加结论，一句可背）：**三角形 N⁻¹、点基元 N⁻¹ᐟ²、各向同性体素 N⁻¹ᐟ⁴**——"最优单元恰是各向异性协方差椭球原子，正是 3DGS 拟合的东西"。

## Technical Approach

### 1. Factoring LOD（部署侧）

```text
训练好的模型（均值/旋转/尺度/不透明度/SH）
    ↓ 每个 primitive → plank（μ, R, h=3σ, α）
GPU 八叉树（叶容量 4096）
    ↓ 叶内按尺度分带 ℓ = ⌊log₂(s/s_base)⌋
每（叶,带）→ 批量矩匹配 → ≤256 个"等价 plank"
    ——守恒量：加权体积 W、质心 c*、惯性张量 T*（SH 颜色进一阶矩）
    ↓ 内部节点自底向上同样聚合
构建：几秒（单 GPU）；聚合表示比输入小 119×
```

**渲染**：像素脚印调度器（**单一标量 c**）——阈值 $t = c\cdot d/f$：

| 层 | 条件 | 渲染 |
|---|---|---|
| raw | 成员可分辨（$\bar{s}\ge t$） | 原始高斯 |
| fine | 簇可分辨、成员不可（$\bar{s}<t\le\bar{h}$） | fine 等价 plank |
| coarse | 亚像素（$\bar{h}<t$） | 单个聚合 plank |

- **c=1 = 可分辨参考档**；c 增大 → 渐进合并亚 c 像素内容；c<1 → 保留亚像素成员（超分辨）；
- **选择是分辨率不变的**（800² 与 1080p 同预算下物化集合**相同**）——修掉了朴素阈值调度的漂移；
- **全程 GPU**：选择与收集直入光栅器缓冲，"no CPU-side collection or host-to-device upload"（**"取消 host 回环"家族新样本**）。

### 2. Budget-Centered Training（训练侧）

- 从测量的 floor 曲线定位 **knee**（最陡窗口斜率最大处）N*，**直接在 cap=N* 训练 MCMC densifier**；
- 用 **θ 冗余度图**（从局部因子化计算）替代 opacity 剪枝信号：**"局部多重度高（过度覆盖）的地方，正是可以最小误差移除的地方"**；
- 基于速率的停止准则：验证改善 < 0.5 dB 时冻结密度控制（**消融：没有它默认 densifier 过剪枝 -0.56 dB**）。

### 3. 实验（预注册协议，13 场景 + 官方城市数据）

**Floor 测量**：

| 项目 | 结果 |
|---|---|
| 合成卡通对照 | 斜率 -0.549/-0.548/-0.551（**正中 Zador 定律**，R²=0.997） |
| 13 公开场景 | 斜率 seed 稳定（±0.004-0.011）；最陡三带窗口 -0.41~-0.53 **横跨 -1/2**（预注册区间之外的"注册负结果"，诚实报告；SfM 点云 TWO-NN 内在维度 1.9-2.05） |
| 对偶证书 | 每个（预算,视角）给出误差下界；room 上紧度 0.28→0.03 |
| **已发表压缩器前沿** | Compact3DGS / EAGLES / Mini-Splatting / RadSplat：斜率占 **[-0.28, +0.09]，均值 -0.04**——"**移除 60-70% primitives 成本 <1 dB**"，而 floor 定律陡一个数量级 → **"它们的增益来自冗余，不是接近容量"** |

**BCT（vs 已发表压缩器 / 自建 train-then-prune）**：

- room 529k：**31.94 dB**（3 seeds）vs Compact3DGS 30.23（**+1.7 dB**）；670k 时 **33% fewer primitives** 达到 1M 的同等质量（31.21 dB）；
- counter +0.24/+0.58 dB；bonsai 640k +0.86；kitchen +1.70；garden +0.24/+0.44；**仅 flowers 落后**（诊断性：其 knee 在全预算、θ_waste=1.0**无可省冗余**——"source-limited, not redundancy-limited"）；
- **单次训练 78±7 min vs 传统 175 min**（2.2× 快且质量更高，A100 同卡）。

**Factoring LOD（MatrixCity 官方城市数据）**：

| 项目 | 数字 |
|---|---|
| 单模型扫显存档 | c=4/8/16 → **1288/824/407 MB**（8/4/2 GB 级目标）；质量保持到 2GB 档（该视角 \|ΔPSNR\|≤0.27 dB） |
| 近无损窗口 | c=4 渲染 68% primitives，**-0.05 dB**（城市航拍） |
| 实时 | v3 retrain（6.89M, full SH）：官方 207 视角轨迹 **71 fps** @1080p 原生（4080 SUPER）；p5=55.2；轨道探测 5.99M 全量 49.5 fps |
| 容量梯子 | 0.34M/0.68M/1.38M/2.59M → 76.8/65.0/59.0/57.0 fps（连续、部署时选择） |
| **vs Octree-GS** | 同视角同质量（25.43 vs 25.73 dB，分辨率匹配）：**71 vs 11.5 fps（6.2×）**；**VRAM 峰值 4.4 vs 30.4 GiB（6.9×）** |
| 架构差异 | Octree-GS：8.01M anchors → 解码 ~80M Gaussians（2.5GB）+ 每帧 MLP 解码；本文：6.89M 静态显式 splats（1.7GB）直接喂光栅器**无解码** |
| LOD vs 随机 | 859 MB 档：LOD -2.12 dB vs 随机子采样 -7.08 dB |
| 内存账 | ~20.8 dB 操作点 = 110.6 MB vs GPS full 1926 MB（**17×**） |
| 构建成本 | 10M tree 峰值 6.45 GiB（8GB 3060 Ti 可建；输出 2.53GB） |

### 4. 组合性（一个漂亮的收口）

**BCT-born 模型过因子树几乎无损**：room 上 BCT-born（529k）树 34.19 dB（距自己 root 0.16 dB）vs full-born（1.66M）树 collapse 到同预算 **22.83 dB**——**"生来正确尺寸的模型没有冗余可丢"**。训练侧与部署侧由此闭环。

## Key Contribution

1. **Factoring LOD**：post-hoc、单参数 c、renderer-agnostic（rasterizer/GPS 都验证）、分辨率不变的城市级 LOD 层；
2. **Budget-Centered Training**：把"预算"变成训练处方——所有匹配预算点上领先 train-then-prune，单次训练更快更好；
3. **可测量 + 可认证的容量下限**：13 场景预注册验证 + 对偶证书（每预算每视角的误差下界）+ "已发表压缩器平坦前沿"的统一定位。

## Why It Works / 为什么值得信

- **诚实的预注册 + 注册负结果**：13 场景斜率落出预注册区间（-0.55~-0.45）时如实报告并给出解释（SfM 维度 1.9-2.05、阅读窗口审计），**没有把数据"拗"回定律**——这类纪律在预印本里罕见；
- **三条相互印证的证据链**：合成对照（正中定律）+ 实测括号（横跨）+ 对偶证书（下界收紧），比单点拟合稳健得多；
- **理论 → 实现 → 数字三层对齐**：collapse license → 矩匹配聚合（119×）；truncation → 像素脚印阈值（c）；allocation → 尺度带指数衰减（八叉树分带）。**没有"理论归理论"的脱节**。

## Limitations

- **单卡评估**；透明与极端遮挡区域未覆盖；**内存流式（多分辨率数据的按需 I/O）是明确的 future work**（当前是"构建后常驻 + 选择"）；
- 证明层覆盖合成渲染器族 / 原子与场表示；三个开放项：可见性指数界的锐度、隐式场 SGD 的可训练性率、非合成渲染器（焦散/参与介质）；
- **预印本未经同行评审**（EG 2027 投稿中）；无公开代码信息；"首个 post-hoc full-SH 消费级城市级结果"的表述以作者口径为准（值得后续跟踪 EG 2027 接收情况）；
- 层切换（tier transitions）的**迟滞与感知验证**未做——"residual popping is confined to tier boundaries"（潜在集成风险点）。

## Game Development Relevance

- **"显存分档硬约束"的 GS 侧答案**：与 [[Scalability and Quality Tiers]] 记录的 E-Day/Mega Geometry 趋势（12GB 门槛、显存成为分档硬约束）完全同向——本文把"同一城市模型服务不同显存设备"做成**部署时单参数**；
- **"档位 = 部署时选择"的最干净形态**：c 参数 = "档位条件化"线索（《控制：共振》Dynamic / ControlGS / XeSS 3）在此得到"**一个标量扫全部档位**"的极端样本；
- **对开放世界引擎的映射**：Factoring LOD 的形态（八叉树 + 尺度带 + 单参数调度）与 UE 的 **World Partition / Nanite 集群剔除**在结构上同构（见 Unreal 节）；
- **"LOD vs 随机子采样"的量化对照（-2.12 vs -7.08 dB）可直接进入"GS 资产规范"话语**：未来引擎接 GS 资产时，"有没有 LOD 层级"是比"压缩率"更本质的指标。

## Unreal Engine Relevance

- **结构同构**：GPU 八叉树 + 尺度带 ≈ Nanite cluster 层级的思想移植到 GS；"物化工作集预算 K"≈ 流式纹理/Nanite 的 residency 预算——**若 UE 未来引入 GS 管线适配层，这类"前缀 LOD + 容量预算"是候选架构**；
- **"取消 host 回环"样本**（选择与收集全 GPU）与库里 CuACD / Stochastic GS 家族同判据；
- 无直接可移植代码（预印本），**列为技术观察项**：EG 2027 结果 + 是否开源。

## Technology Evolution

```text
1976  Clark —— "可见复杂度固定上限" + working set + 递归下降（分档的祖先）★ 本库已入库
        ↓
1996  Hoppe Progressive Meshes —— mesh 的连续 LOD（本文引用）
2013  QSplat / Potree —— 点云的流式层级（本文引用）
2023  3DGS —— 显式 splat 表示（single operating point）
        ↓
2024  Octree-GS / CityGaussian —— **训练耦合**的 LOD（预算在训练时定死）
2026  out-of-core streaming / LoD-aware training —— 同为训练耦合路线
        ↓
★ 2026  Budgeted-GS（本篇）—— **post-hoc**：一个已训练模型 + 部署时预算；
         并首次给 GS 补上"容量下限"（Zador 律）与合并许可（Kakeya 2025 应用）
"One floor, two methods: deployment renders exactly what fits;
 training grows exactly what the scene needs."
```

## Relationships

### Based On
- Zador 定律 / 高分辨率量化理论（Graf & Luschy 2000；DeVore 1998）——容量下限的数学源
- **Wang–Zahl 2025**（三维 Kakeya 集猜想解决的并集体积估计）——合并许可；**据作者称是"Kakeya 理论在渲染的首次应用"**
- [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)]] —— 概念祖先（可见复杂度上限 / working set）

### Extends
- Octree-GS / CityGaussian —— **从训练耦合 LOD 到 post-hoc、部署时预算**（互补轴）
- 3DGS 压缩线（Compact3DGS / EAGLES / LightGaussian / PUP）——把"压缩"重定义为"在 floor 上定位操作点"

### Related
- [[Gaussian Splatting]] —— 本条为 **LOD/预算线**（与"排序线""压缩线"并列的新线）
- [[Scalability and Quality Tiers]] —— "容量下限"作为分档第三种语言（见 Daily）
- [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising]] / [[2026-09-29-Gaussian Stippling — Efficient Sorting-Free 3D Gaussian Rendering]] —— 同族的"补丁式优化 vs 结构式重设计"对照：**排序线在省"怎么渲染"，本篇在省"渲染多少"**
- [[2026-09-26-Constant-Memory Differentiable Light Tracing]] —— 无直接血缘，但同属"把成本函数改写成另一个变量的函数"（那边是内存↔时间，这边是内存↔质量）

### Followed By
- （观察项）EG 2027 评审结果；Memory streaming 扩展（作者列出的 future work）；**"部署时预算"是否会进入引擎侧工具链**

## Personal Knowledge State

- **user_level: Normal** —— 建立在 [[Gaussian Splatting]]（Easy）之上；理论层（最优传输/量化论）**不必现在读通**，结论层四条可直接拿走：
  1. **误差 ∝ N^(-1/2)（表面场景）**——"预算翻倍，误差降到 70%"的一把尺子；
  2. **已发表压缩器的斜率大多在平坦区（-0.04）**——读任何 GS 压缩论文先问"报告的是冗余收益还是容量收益"；
  3. **单参数 c 扫连续曲线**——"档位 = 部署时选择"的具体形态；
  4. **LOD vs 随机子采样同内存对照（-2.12 vs -7.08 dB）**——LOD 层级是"比压缩率更本质"的 GS 资产属性。
- **与你工作的接口（最重要）**：这篇的"容量下限 + 每视角预算选择"是**你 SABC 预算体系在 GS 领域的理论化版本**——你的五维预算是"经验上限"，它给的是"**先证下限、再定预算**"的方法论样本（对偶证书 = "预算的证明材料"的一种形态）。

## Learning Value

- **"预算语言"第三轨**（本库的独特沉淀）：样本定预算（1978→2026 四连）× 误差定预算（Ward 1988 a 容限）× **率-失真定律（本篇）**——三者并存：**"预算定在样本上 / 误差上 / 曲线上的哪个点"**；
- **"降档 = 降表示层级"的又一实证**（Parish 2001 迭代深度 / ToCo-Mesh 自适应细分 / 本篇矩匹配聚合——同一个母题的第 N 个样本）；
- **方法论**："**测量先于设计**"的极端执行（预注册 + 注册负结果 + 对偶证书）——与本库"看论文先找降级消融"（OREO）并列为"读论文的两种防御"。

## Visualization

![[Budgeted-GS_容量下限与因子树图解.html]]

## Notes

- **窗口捕获方式**：10-2 提交、10-5 运行时刻 listing 未公告，由 **API submittedDate 宽窗 [10-2 ~ 10-6]** 捕获（"未公告先见"通道，双通道分工的又一实例）；同窗口另有 2 条记录项见 [[2026-10-05]]；
- **同作者当天两条**（另一条为 LLM agent 动力论，与游戏无关，记名）；作者单位为东软（Neusoft）；
- **一处术语注意**：本文的 "budget" 指"primitive 数量预算"（率-失真的 rate），与"显存预算/算力预算"是相关但不同的量——引用时保持口径区分；
- 数字核验：全文数字取自 arXiv HTML 版（`/html/2610.03162v1`），关键数字（71 fps / 6.2× / 4.4 vs 30.4 GiB / 33% / 119× / 78 vs 175 min）均有原文出处段落。
