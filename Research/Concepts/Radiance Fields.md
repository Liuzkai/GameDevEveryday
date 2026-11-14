---
type: concept
user_level: Normal
aliases:
  - Radiance Field
  - Neural Radiance Field
  - NeRF
  - 辐射场
prerequisites:
  - "[[Rendering Equation]]"
  - "[[Participating Media]]"
  - "[[Global Illumination]]"
first_introduced: "NeRF (Mildenhall et al., 2020)，概念更早见于光场与辐射度文献"
---

# Radiance Fields

## Definition

**辐射场 = 空间中的一条"查询规则"**：给定一个位置 $\mathbf x$ 与一个方向 $\mathbf d$，回答"沿这个方向、在这个点上，有多少辐射亮度、有多少不透明度"。

$$L:\ (\mathbf x,\mathbf d)\ \longmapsto\ (L_o,\ \sigma)$$

它不是一个数据结构，是**一类接口**。凡是能回答这个查询的东西都是辐射场——MLP 可以（[[Mildenhall 2020 — NeRF Representing Scenes as Neural Radiance Fields|NeRF]]），一堆带朝向的椭球也可以（[[Gaussian Splatting]]），哈希网格 + 小网络可以（Instant-NGP），甚至一张多平面图也可以。

**本库把"辐射场"与"高斯泼溅"分成两个概念**，不是因为它们数学不同，而是因为**它们的成本结构完全不同**：

| | 隐式辐射场（NeRF 系） | 显式辐射场（GS 系） |
|---|---|---|
| 存什么 | 网络权重（几 MB） | 图元（几十万~几百万个） |
| 查询一次 | 一次 MLP 前向（~9 层） | 一次投影 + 混合 |
| 一帧的账 | **光线数 × 每光线采样数 × 每样本前向** | **图元数 × 覆盖像素 × 排序** |
| 换分辨率 | 免费（连续） | 要重投影 |
| 改场景 | 要重训 | 可以局部编辑 |

**同一个接口，两种完全不同的成本函数**——这就是为什么它们值得分成两页。

## Core Principle

### 一、场的"稀疏性"决定了它能不能被存下来

一个三维场景的完整辐射场是 5 维的（3 位置 + 2 方向），直接离散化是 $N^5$。所有实用方法都在做同一件事：**假设它稀疏或低秩**。

- NeRF 假设它**能被一个小 MLP 拟合**（隐式正则）；
- 3DGS 假设它**能被有限个各向异性高斯覆盖**（显式支撑）；
- 哈希网格假设它**在局部是低频的**（多分辨率特征）。

**这三种假设对应三种失效方式**：NeRF 在极端细节上糊、3DGS 在大面积无纹理区堆冗余图元、哈希网格在高频处出现块状伪影。**假设就是你买到的东西，也是你会栽的地方。**

### 二、体渲染公式是这两个体系的公共底座

$$C=\int T(t)\,\sigma(t)\,L_o(t)\,dt,\qquad T(t)=e^{-\int\sigma}$$

离散后：

$$\hat C=\sum_i T_i\big(1-e^{-\sigma_i\delta_i}\big)L_i,\qquad \alpha_i=1-e^{-\sigma_i\delta_i}$$

**$\alpha_i$ 这个形式是全篇的关键**：它让"体积"与"alpha 混合"在代数上是同一件事。**3DGS 之所以能把 NeRF 的公式原封不动搬进光栅化器，就是因为这个式子**。

### 三、四种存储形态：答案 / 证据 / 算子 / 场

这是本库反复撞到的一张表，今天把它固定下来：

| 形态 | 存什么 | 改什么不用重算 | 代表 |
|---|---|---|---|
| **答案** | 辐照度 / lightmap | 改视角 | [[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)|Ward 1988]]、DDGI |
| **证据** | 光子 / 事件记录 / 多视角图 | 改 BRDF、改吸收、改融合权重 | [[Jensen — Global Illumination using Photon Maps (1996)|Jensen 1996]]、[[2026-10-30-Liao — Walk on Multiscale Media Compiling Editable and Composable Transport Medium Prefabs|WoMM]] |
| **算子** | 传输矩阵 / SH 系数 | 改光照 | [[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)|PRT 2002]]、RADIANT |
| **场** | 一个可查询的函数 | **改视角、改分辨率** | [[Mildenhall 2020 — NeRF Representing Scenes as Neural Radiance Fields|NeRF 2020]]、3DGS、L3G |

**判据**：**先问"下游要问几个问题"，再问"哪个问题最贵"。** 只问一个（这一点多亮）⇒ 存答案；要问无穷多个（任意视角）⇒ 存场；要问"如果换成另一个 BRDF 呢" ⇒ 存证据；要问"换一盏灯呢" ⇒ 存算子。

> **场的独特之处**：它是唯一一种"**查询次数不影响存储**"的形态。代价是**每次查询都要付钱**——这正是 NeRF 30 秒一帧、3DGS 能实时的全部原因。

## Prerequisites

- [[Rendering Equation]]：体渲染是它的无散射特例；
- [[Participating Media]]：吸收-发射模型的积分与离散化；
- 基础：坐标变换、方向参数化（球坐标 / 6D）、MLP 的可微性。

## Historical Evolution

```text
1986  Kajiya 渲染方程            把"光怎么走"写成积分，但没说场怎么存
1996  Jensen 光子图              第一次"存证据"——但存的是表面上的点，不是空间中的场
2002  PRT                       存算子——把"场"换成"基函数系数"
2019  SRN / DeepVoxels / NV     第一次用连续函数表示场景（模糊、或受分辨率限制）
2020  ★ NeRF                    5D 函数 + 体渲染 + 位置编码 + 分层采样，四件套齐
2021  Mip-NeRF                  把"一条光线"换成"一个锥"——抗锯齿进到表示里
2022  Instant-NGP               场 + 哈希网格：训练从天到分钟
2023  3DGS                      场的显式采样：渲染从秒到毫秒
2026  L3G-World                 高斯里存**别人要的 latent**——场开始"按消费者定制"
        ↓
      "表示 → 更快的表示 → 按需定制的表示"
```

**注意最后一步的方向变化**：NeRF 与 3DGS 都在优化"**怎么查得快**"，而 2026 的 L3G 优化的是"**查出来的东西谁能直接用**"——**接口从"通用"走向"专用"，这是任何表示成熟后的必然。**

## Important Papers

| 论文 | 角色 |
|---|---|
| [[Mildenhall 2020 — NeRF Representing Scenes as Neural Radiance Fields|NeRF 2020]] | 定义者：5D 函数 + 体渲染 + 位置编码 |
| [[Gaussian Splatting]]（概念页） | 显式化：把场采样成图元，用光栅化替代查询 |
| [[2026-10-29-Choi — L3G-World Real-Time Infinite World Generation with Latent 3D Gaussian Memory|L3G-World 2026]] | 接口化：场里存的不再是颜色，而是下游模型要的 latent |
| [[Zwicker — Surface Splatting (2001)|Surface Splatting 2001]] | 史前史：屏幕空间 EWA 核，splat 的数学源头 |
| [[2026-10-27-Kaushik — Hexels Compact Hexagonal Primitives for Scene Rendering|Hexels 2026]] | 原语形状的选择：六边形 vs 高斯 |

## Related Concepts

- [[Gaussian Splatting]]：**同一接口的显式实现**。两页的分工是"成本结构"而不是"数学"；
- [[Neural Rendering]]：辐射场是它的核心表示；
- [[Participating Media]]：吸收-发射模型与体积积分，是辐射场的渲染底座；
- [[Global Illumination]]：辐射场可以被当作 GI 的一种"表示层"选择（存场）；
- [[Streaming]]：**当场的存储超过显存时**，问题就变成流送——L3G 的 $O(S)$ 就是这条交界上的样本。

## Technologies

- **NeRF 系**：位置编码、分层采样、哈希网格、锥 tracing；
- **GS 系**：各向异性高斯、EWA 投影、可微光栅化、密度控制（clone/split/prune）；
- **混合系**：3DGS + 网格提取、GS + 神经外观、GS + latent（L3G）；
- **工程**：CUDA 光栅化器、排序（或免排序）、分块（tile）渲染。

## Game Applications

- **扫描 / 数字化**：实景转可查询场景（NeRF 出数据、GS 出实时）；
- **过场与预演**：离线质量、在线播放（需要预烘焙或转 GS）；
- **不可用之处**：**光照 baked-in**。游戏要改光、要昼夜、要动态阴影——**辐射场本身不提供这些**，必须外挂（GS + 延迟着色、或 NeRF + 重光照网络）。**这是它进引擎最大的墙，比速度更难绕。**
- **对 VFX 的隐喻价值**：辐射场是"**按查询计费**"的极端样本——**它的成本公式（光线 × 采样 × 前向）与粒子系统的成本公式（粒子 × 像素 × 层数）是同构的**：都是"数量 × 每单位的处理次数"。

## Personal Knowledge

`Normal`。已读 [[Mildenhall 2020 — NeRF Representing Scenes as Neural Radiance Fields|NeRF 2020]]（2026-11-02 入库，`status: unread`）；GS 侧沿 [[Gaussian Splatting]] 的既有记录（基础 Easy）。

**边界**：能说清"场 vs 图元"的成本差别、能说清体渲染公式与 $\alpha_i$ 的来源；**不能说清**的是位置编码的谱分析、哈希网格的冲突分析、以及重光照方向的全部工作。

## Learning Gap

- **重光照**：辐射场把光照烤进颜色里，这是它进游戏最大的障碍。库内这块几乎是空的（只有 PRT / DDGI 那条"存算子"的线）；
- **动态**：4D 辐射场（时空）在库里只有 GS 侧的几个样本，NeRF 侧没有；
- **"场"与"可见性"的关系**：辐射场不做剔除、不做遮挡查询——**它和"虚拟几何 / 可见性剔除"那条线还没有接上**。

## Next Step

1. 若要补重光照：**NeRF → NeRV / Tensor4D → GS + 延迟着色**，三步走；库内先补"GS 重光照"一篇即可（这是本库目前最明显的空白）；
2. 若要补动态：从 4DGS 那几篇往回读（[[2026-10-26-Yachil — GaussianSpline Adaptive Bezier Splines for 4D Gaussian Splatting|GaussianSpline]] 是现成入口）；
3. 若要接引擎：读"GS 与光栅化管线混合"的实现（本库有 Hexels / 3DGS 变体，但**没有一篇讲"GS 怎么和 GBuffer 共存"**）。
