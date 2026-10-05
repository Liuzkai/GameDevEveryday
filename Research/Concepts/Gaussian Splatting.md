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
- Alpha blending / 半透明排序

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

**2026 年的关键信号**：SIGGRAPH 2026 Workshop 直接宣布 3DGS 已经成熟为**生成式世界模型的原生输出格式**。"世界模型生成 → 3DGS 表示 → 可交互世界"被行业默认为一条已成立的流水线。

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

## 排序线全景（2026-10-01 沉淀）

"取消排序"自成为库内固定线索以来已到第 5 个节点——**从"优化"到"置换"到"路由"**：

| 节点 | 路线 | 代表 | 动作 |
|---|---|---|---|
| ① | 优化排序混合 | FlashGS / Speedy-Splat / [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization\|TileGS]] | 保持结构，做剔除与调度 |
| ② | 顺序无关透明 | weighted-sum / hybrid transparency | 去全局排序（仍混合） |
| ③ | 重设计基元 | surfel / depth peeling | 换几何锚定方式 |
| ④ | 随机光栅化（全删） | StochasticSplats / [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising\|Stochastic GS Denoising]] | 排序与混合全删，噪声靠神经偿还 |
| **⑤** | **随机透明混合路由** | **[[2026-09-29-Gaussian Stippling — Efficient Sorting-Free 3D Gaussian Rendering\|Gaussian Stippling]]（2026-09-29）** | **双流按成本分拣 + 跨场景重建 + 移动端落地** |

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
