---
type: concept
title: "Gaussian Splatting"
user_level: Easy
tags: [rendering, 3d-representation, radiance-fields]
---

# Gaussian Splatting

## Definition

用大量**显式 3D 高斯椭球**作为场景表示，通过可微光栅化（splatting）实现高质量新视角合成。2023 年提出后迅速成为神经渲染领域最主流的显式表示。

## Core Principle

- 场景 = 一堆带位置、协方差（形状）、不透明度、颜色（通常用球谐表示视角相关颜色）的高斯
- 渲染 = 把高斯投影到屏幕 → 按深度排序 → alpha 混合
- 训练 = 可微光栅化 + 梯度下降 + 自适应密度控制（分裂/克隆/剪枝）

## Prerequisites

- [[Neural Rendering]]
- 协方差与椭球（线性代数）
- [[Differentiable Rendering]]
- Alpha blending / 半透明排序 → [[Overdraw]]（**每像素几十~几百层 ⇒ 它是 GS 的第一成本项**）

## Evolution

```
NeRF（隐式 MLP + volume rendering，慢）
        ↓
3D Gaussian Splatting（显式基元 + 可微光栅化，实时）
        ↓
工程化深水区（2026）：
  · 收敛速度 → 结构感知密集化（SADGS）
  · 重建质量 → 视频扩散先验（VidSplat）、物理+光学混合（GauSmoke）
  · 规模 → 光场显示（CoherentRaster）、4D 不确定性（GraphiXS）、
           语义抠资产（LangSplatV2）+ 稀疏体素（fVDB）
  · 渲染管线 → 随机光栅化（取消排序与混合）+ 时域神经去噪 ★ 2026-09-24 入库
       → 随机透明"混合路由"（fragment ⊕ primitive 按成本分拣）+ 跨场景重建 + 移动端 NPU ★ 2026-10-01 入库
  · 规模与预算 → **Factoring LOD（post-hoc 因子树 + 部署时单参数预算）+ 容量下限（Zador 律）** ★ 2026-10-05 入库
        ↓
成为生成式世界模型的原生输出格式
```

**4DGS 的三条表示路线（2026-10-28 补，随 GaussianSpline 入库）**——动态场景里"每帧一组参数"是默认值，但有三种方式绕开它：

| 路线 | 时间分辨率由什么决定 | 代表 | 代价 |
|---|---|---|---|
| 每帧独立状态 | **帧率** | Dynamic 3D Gaussians（per-timestep 优化 + 局部刚性） | 存储 ∝ 帧数 |
| 神经变形场 | MLP 的隐式光滑性 | Deformable 3DGS / SC-GS / GauFRe | 渲染时每高斯过一次 MLP |
| 结构化时空分解 | 体素/平面的**预设**分辨率 | 4DGS / 4K4D / K-Planes | 分辨率是全局超参 |
| **显式轨迹** | 多项式阶数 / 分段数 / **段宽** | STG / Gaussian-Flow / SplineGS → **[[2026-10-26-Yachil — GaussianSpline Adaptive Bezier Splines for 4D Gaussian Splatting\|GaussianSpline]]** | 局部控制强，但无空间耦合（每高斯独立轨迹） |

第四行内部的演化才是重点：**多项式（全局）→ 均匀分段（全局段数）→ 可学段宽（每高斯每段）**。而 [[2026-10-26-Yachil — GaussianSpline Adaptive Bezier Splines for 4D Gaussian Splatting|GaussianSpline]] 的消融给出一句可以直接搬走的判据：**段数从 2 加到 4 没有收益（31.61 ≤ 32.08 dB），把段放对位置值 +0.37~+0.91 dB**——**时间轴的分配方式比分配额度重要**。这与库内"分配四端"（生成/生长/保留/执行）是同一条规律换了一条轴。


**2026 年的关键信号**：SIGGRAPH 2026 Workshop 直接宣布 3DGS 已经成熟为**生成式世界模型的原生输出格式**。"世界模型生成 → 3DGS 表示 → 可交互世界"被行业默认为一条已成立的流水线。

### 原语本体：形状、核与容量（2026-10-29 补，随 Surface Splatting 2001 + Hexels 入库）

这条线此前在本页里只被当成"表示"讨论，其实底下还有一层——**"一个原语是什么"**。补上两个端点后就完整了：

| 年代 | 原语 | 支撑域形状 | 形状从哪来 | 原语内能表达多少 |
|---|---|---|---|---|
| 2000 | Surfels | 圆/椭圆 | 几何（法向） | 一个色 |
| **2001** | **[[Zwicker — Surface Splatting (2001)\|Surface Splatting]]** | **椭圆（EWA）** | **推导**（雅可比 $J$ 决定各向异性，$+I$ 决定带限） | 一个色 |
| 2023 | 3DGS | 椭圆（可学协方差） | 推导 + 学习 | 一个色 + 视角相关 SH |
| 2024 | 2DGS | 面元（仍是椭圆） | 同上 | 同上 |
| 2025 | Triangle / Mesh Splatting | 三角形 | **选**（多边形铺砌） | 顶点插值 |
| **2026** | **[[2026-10-27-Kaushik — Hexels Compact Hexagonal Primitives for Scene Rendering\|Hexels]]** | **六边形（软边）** | **选**（等周长下面积最大） | **一阶线性色场** |

**三件事必须分清**（混了就会得出错的结论）：

1. **核的形状**：3DGS 的 $\Sigma'=JW\Sigma W^\top J^\top$ 与 Surface Splatting 的 $JV_kJ^\top$ **是同一个式子**；差别只在低通项——EWA 是严格推导出的 $+I$（像素面积），3DGS 用 $+0.3I$ 的膨胀近似，Hexels 干脆省掉。**每省一次，minification 的保证就少一层。**
2. **原语的容量**：从"一个色"到"一阶色场"，改的是**一个原语能代表多少**。Hexels 的消融给出一个反直觉结论——**形状与容量互相替代，不是相加**（平滑目标上，31.33 → +3.80 加颜色场 / +4.29 换形状，但两者一起只有 37.07 而非 39.42）。
3. **覆盖次数**：Hexels 的 overdraw 从 **10.47 → 2.48 blends/px**，而这一项**与颜色场几乎无关**——它是纯形状效应。**颜色场买"表达量"，六边形买"覆盖次数"，这是两笔账。**

**对预算的直接意义**：GS 的预算表通常只有"高斯个数"一列。补上这两列后，"同预算"才有定义——**同原语数、同模型内存、同训练显存、同渲染 FPS 是四个不同的预算**，Hexels 那篇里 2DGS 对 3DGS 的差距就从 −2.64 dB 变到 −1.73 dB，**只因为分母换了**。

## Related Concepts

- [[Neural Rendering]]
- [[Differentiable Rendering]]
- [[Radiance Fields]]

## Game Applications

- 场景/资产捕获与重建
- VR 照片级写实
- 开放世界重建（[[Open World Reconstruction]]）

## Important Papers

- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]
- [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising]] —— **"排序瓶颈"的第四条路线（取消式）**：随机光栅化 + 时域神经去噪；全管线 2.1×、去噪 +1 ms 常数（与 TileGS 构成同题两端）★ 2026-09-24
- [[2026-09-29-Gaussian Stippling — Efficient Sorting-Free 3D Gaussian Rendering]] —— **随机透明家族第二代（⑤）**：fragment / primitive 双流按成本路由 + 跨场景时空重建 + 移动端 NPU 全管线（92.6 FPS 裸渲染 / 73.2 FPS 含重建 @540p）；直接吃**未修改** 3DGS 资产；1080p 达 2.3–2.7× 标准 3DGS 吞吐 ★ 2026-10-01
- [[Inverse Rendering for Modeling with Line Primitives]]（对比：显式线段 vs 体积基元）
- [[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD]] —— **LOD/预算线开线**：容量下限（误差 ∝ N⁻¹ᐟ²）+ 因子树（矩匹配聚合 119×）+ 单参数 c 扫连续内存-质量曲线；官方城市数据 71 fps @1080p 全 SH（vs Octree-GS 11.5 fps）；**"LOD 分层是比压缩率更本质的资产属性"（同内存 -2.12 vs 随机 -7.08 dB）** ★ 2026-10-05
- [[2026-10-08-PCAsplat — Gaussian Splatting with Local PCA Regularization|PCAsplat]] —— **几何质量线（新）**：可微局部 PCA 正则（作用在中心邻域、**视角无关**）修复 floaters——"看不见的高斯也有监督"；特征值谱正则（表面化 + 各向同性切向）+ 法线对齐；DTU/T&T 上表面质量提升、**下游（点云分割 / Poisson 重建）免修直用** ★ 2026-10-10
- [[2026-10-09-Efficient Confidence-Aware Densification for 3DGS|Efficient Confidence-Aware Densification for 3DGS]] —— **分配四端之"生长端"（密度控制）**：3DGS-MCMC 的"不透明度驱动"→"误差驱动"（梯度排名 × 低置信度加权采样 + 贡献度重定位）；24 场景 100k–2M 全预算超 MCMC；**"预算跟着缺口走，不跟着现状走"** ★ 2026-10-13
- [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]] —— **场景动态化（"注入派"）**：静态 3DGS 的流体动效注入（光流 → 3D 欧拉运动场 → cinemagraph 式对称变形）；"三不"（无捕获/无重训/无模拟）+ 2 MB 可携带运动场 + 实时循环 ★ 2026-10-13
- [[2026-10-21-Kim — RLSplat Resource-Linear Budget Control for Feed-forward 3D Gaussian Splatting|RLSplat]] —— **预算的成本侧（新）**：把“给定预算”从“事后裁剪”改为“前馈生成时按配额分配”——GSPO 策略网络学高斯配额 $k\sim\mathrm{Mult}(B,p)$（$\sum k_n=B$），资源线性解码器只用 $O(BK_{gs})$ 局部 cross-attn 取出；三个正则（Voronoi / offset / projection）分别撑住“不重叠 / 不漂移 / 可微”；2K 预算下 +0.9 dB、显存 **9.6×** 少、速度 **4.1×**；**关键诚实点：策略 24.83 dB vs Sobel 边缘启发式 24.76 dB，仅差 0.07 dB——“学到的是分配这个动作，不是更聪明的先验”** ★ 2026-10-22
- [[2026-10-21-Beche — KoalaGS Complexity-Aware Continuous Level of Detail for Gaussian Splatting|KoalaGS]] —— **连续 LoD 的排序键（新）**：用**优化前**的图像复杂度（delentropy，2D ρ≈−0.60 / 3D −0.64）做空间先验，训练时按 $p=\alpha e^{\tilde C}$ 迁移偏置 + complexity dropout（$\delta=0.2$），渲染时排序键 $S=\alpha+\lambda s/e^{\tilde C}$ + 残差方差正则；100%→0.1% 连续全谱评测（不是几个离散档）；**如实边界：对 MGS 排序只 +0.12/+1.23 dB，主要增益来自与“不排序”的 MCMC 对比——先验有用，但别夸大** ★ 2026-10-22

## LOD / 预算线（2026-10-05 开线）

排序线（"怎么渲染"）与压缩线（"表示存多少"）之外，GS 的第三条工程线：**"渲染多少"** —— 按预算与视角选择物化哪些 primitive。

| 层次 | 代表 | 回答 |
|---|---|---|
| 单操作点（3DGS 原生） | 训练成什么样、渲染就什么样 | — |
| **训练耦合 LOD** | Octree-GS / CityGaussian（预算在训练时定死） | "训练时就该分好层" |
| **post-hoc 因子树（新）** | **[[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD\|Budgeted-GS]]**（部署时单参数 c） | "**同一个模型服务不同显存设备**" |
| 理论层（新） | 同上：**容量下限**（误差 ∝ N⁻¹ᐟ²）+ 合并许可（Kakeya 2025） | "**到底需要多少 primitive**" |

- **与本库三条既有线索的接点**：① "降档 = 降表示层级"（原始高斯 → 等价 plank → 聚合 plank）——与 Parish 2001 / ToCo-Mesh 同一母题；② "档位 = 部署时选择"的最干净形态（单标量扫全档）；③ **[[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)]] 的 working set 与可见复杂度上限，在此获得 GS 实现**；
- **与随机子采样的量化对照**（859 MB 档）：LOD **-2.12 dB** vs 随机 **-7.08 dB**——"接 GS 资产时先把 LOD 层级问清楚"。

### 分配四端（2026-10-12 ~ 10-13 沉淀）

"固定预算"之下的第四个问题——**"给定总量，怎么分配？"**——库内已集齐四端（分配对象不同）：

| 端 | 问题 | 节点 | 核心动作 |
|---|---|---|---|
| **保留端** | 选哪些？ | [[2026-10-09-SplatQ — Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection\|SplatQ]]（2026-10-12） | 分域 → 配额平滑（防独占）→ 域内联合选择（QUBO） |
| **生成端** | 放哪里？ | [[2026-10-09-CompactSplat — Spatially Adaptive Gaussian Distribution for Feedforward Scene Reconstruction\|CompactSplat]]（2026-10-12） | 内容自适应（复杂区密小 / 平坦区"用尺度换数量"）+ 推理旋钮 |
| **执行端** | 怎么跑得快？ | [[2026-10-09-H2-Ray — A Hybrid and Hierarchical Ray Tracing Method for Efficient 3D Gaussian Rendering\|H2-Ray]]（2026-10-12） | 负载均衡 + 遍历复用 + 混合求交 |
| **生长端** | **预算给谁？（密度控制）** | [[2026-10-09-Efficient Confidence-Aware Densification for 3DGS\|Confidence-Aware Densification]]（2026-10-13） | **误差驱动采样 + 贡献度重定位**（"预算跟着缺口走"） |

> **两条横向判据（四端共用）**：① **"分配依据测的是现状还是缺口？"**——按"现状"分配（不透明度/已建成度）会富者愈富（Confidence-Aware 的前景偏差）；按"缺口"（误差/欠重建）才是补短板；**倒挂分配（SABC）与"按缺口分配"是同一原则**。② **"防独占"**——SplatQ 的配额平滑与 Confidence-Aware 的误差采样是同一问题的两端实现。

## 排序线全景（2026-10-01 沉淀；2026-10-07 补第 6 节点）

"取消排序"自成为库内固定线索以来已到第 6 个节点——**从"优化"到"置换"到"路由"到"驯服随机性"**：

| 节点 | 路线 | 代表 | 动作 |
|---|---|---|---|
| ① | 优化排序混合 | FlashGS / Speedy-Splat / [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization\|TileGS]] | 保持结构，做剔除与调度 |
| ② | 顺序无关透明 | weighted-sum / hybrid transparency | 去全局排序（仍混合） |
| ③ | 重设计基元 | surfel / [[Everitt — Interactive Order-Independent Transparency (2001)\|depth peeling]] | 换几何锚定方式 |
| ④ | 随机光栅化（全删） | StochasticSplats / [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising\|Stochastic GS Denoising]] | 排序与混合全删，噪声靠神经偿还 |
| ⑤ | 随机透明混合路由 | [[2026-09-29-Gaussian Stippling — Efficient Sorting-Free 3D Gaussian Rendering\|Gaussian Stippling]]（2026-09-29） | 双流按成本分拣 + 跨场景重建 + 移动端落地 |
| **⑥** | **随机性驯服（表示 + 重采样）** | **[[2026-10-04-SteadySplats — Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering\|SteadySplats]]（2026-10-04）** | **训练方差正则 + 时空重采样：1 spp +13.1 dB、收敛 L1 < 1e-4——噪声税基本取消** |

> **第 6 节点的意义（2026-10-07）**：④ 时代的死穴是"压噪要数百 spp，抵消性能收益"；⑥ 把 **1 spp 变干净**（31.44 dB）且**高样本端收敛回排序质量**（L1 < 1e-4）——**"档位 = 采样数"（1 / 4–16 / 100+ spp）自此每一档都可交付**。与 ④ 的对照：**"测干净"（神经去噪）vs "生干净"（表示+重采样，无网络推理）**。

## Personal Knowledge

Current Level: **Easy**（2026-09-11 标 Easy，4 条 Mastery 判据全过）

讲解笔记（图解两轮，含嵌入图）：[[GS 图解 1 — 协方差与椭球：高斯的形状说明书]] · [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制]]

## Learning Gap（2026-09-11 全部闭环）

1. ~~为什么排序是它的性能瓶颈~~ → [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制]]（blending 不交换 × 百万级 × 每像素几十层）
2. ~~它为什么难进标准管线~~ → 同上：G-buffer 单表面假设被"每像素几十层雾"直接违反
3. ~~自适应密度控制在做什么~~ → 同上：训练时的发射器管理（split / clone / prune）

## Mastery Criteria

- [x] 说清为什么 3DGS 比 NeRF 快
- [x] 解释它的排序需求与半透明粒子排序的同构性
- [x] 说清它为什么不能直接进 deferred 管线
- [x] 判断一个给定场景该用 mesh 还是 3DGS

## Next Learning Step

~~→ [[Learning Path — Gaussian Splatting]]~~ ✅ 2026-09-11 完成，标 **Easy**。

后续按约定**停止基础推送**，只推建立在 GS 之上的新研究（动态 GS、GS 角色管线等）。

> **2026-09-24 更新**：首篇"GS 之上的新研究"已入库 —— [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]]（**排序瓶颈的"取消式"解法**：把排序与混合从管线里删掉，用 ~1 ms 神经去噪器偿还噪声债；自由导航 PSNR 29.80 vs ST-TAA 21.48）。**Easy 标记不变** —— 该文不含 GS 基础内容，全部价值在"管线置换 + 去噪器设计"两层。

> **2026-10-01 更新**：排序线第 5 节点 —— [[2026-09-29-Gaussian Stippling — Efficient Sorting-Free 3D Gaussian Rendering|Gaussian Stippling]]（**随机透明"混合路由"代**：fragment/primitive 双流按成本分拣 + 跨场景时空重建；桌面 2.3–2.7× 标准 3DGS、移动端 73.2 FPS@540p 含 NPU 重建；直接吃未修改资产）。**Easy 标记不变** —— 价值在"成本路由 + 移动端全管线"两层；**"档位 = 采样数（1/4/16-spp）"** 是该文提供的分档语言。

> **2026-10-05 更新**：**LOD / 预算线开线** —— [[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]]（post-hoc 因子树 + 容量下限）。**Easy 标记不变** —— 价值在"规模工程 + 预算理论"两层；对分档工作的直接接口：**"误差 ∝ N⁻¹ᐟ²"与"单参数扫连续曲线"**（详见 [[Scalability and Quality Tiers]] 10-05 续记）。

> **2026-10-07 更新**：**排序线第 6 节点（收口性质）** —— [[2026-10-04-SteadySplats — Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering|SteadySplats]]（**"生干净"路线**：训练端方差正则 $\mathcal{L}_{var}$ + 推理端历史感知空间重采样/时域重要度重采样）。**1 spp 31.44 dB（vs 前作 18.34，+13.1 dB）；收敛极限与排序版差距 L1 < 10⁻⁴**——排序免渲染的"画质税"基本取消。**Easy 标记不变** —— 价值在"降噪机制 + 采样-质量权衡"两层；判据：**"噪声的根源在第几层，修补就在第几层"**（对照 [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]] 的"网络偿还"路线）。

> **2026-10-13 更新**：**"分配四端"补齐（生长端）** —— [[2026-10-09-Efficient Confidence-Aware Densification for 3DGS|Efficient Confidence-Aware Densification for 3DGS]]（**固定预算加密从"不透明度驱动"改回"误差驱动"**：梯度排名 × 低置信度加权采样 + 贡献度重定位；"预算跟着缺口走"）。**Easy 标记不变** —— 价值在"分配判据 + 前景偏差诊断"两层；同时 [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]] 开"场景动态化（注入派）"微线索（静态场景 + ~2 MB 可携带运动场 = 无缝循环水体动效）。
