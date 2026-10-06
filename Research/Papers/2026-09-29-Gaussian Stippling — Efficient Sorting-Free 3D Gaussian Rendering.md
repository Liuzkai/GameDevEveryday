---
type: paper
title: "Gaussian Stippling: Efficient Sorting-Free 3D Gaussian Rendering through Hybrid Sampling and Spatiotemporal Reconstruction"
authors: [Zijian Huang, Suiliang Mai, Chuankun Zheng, Yuan Meng, Yuchi Huo]
year: 2026
published: "2026-09-29 (arXiv v1, Preprint)"
venue: "arXiv Preprint（浙江大学 CAD&CG 国家重点实验室）"
url: "https://arxiv.org/abs/2609.38488"
code: ""
project_page: ""
category: [rendering, gaussian-splatting, real-time, mobile, neural-reconstruction]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: Prototype（桌面 2.3–2.7× 3DGS；移动端 NPU 实测 73 FPS@540p）
user_level: Normal（GS 之上的新研究；排序线新节点）
status: unread
tags: [gaussian-splatting, rendering, sorting-free, stochastic, mobile, npu]
---

# Gaussian Stippling: Sorting-Free 3D Gaussian Rendering via Hybrid Sampling and Spatiotemporal Reconstruction（ZJU, 2026）

## TL;DR

**"随机透明"家族的下一代：不再单选一种随机流，而是按成本结构把每个高斯路由到两条随机流之一，再用跨场景的时空重建网络把噪声洗掉。** 两条流是——**fragment-based**（走硬件光栅化，成本 ∝ 光栅覆盖 / quad overdraw，适合大足迹高斯）与 **primitive-based**（逐点采样，成本 ∝ 采样数，适合小足迹高斯）；路由判据 = 不透明度 × 投影足迹。噪声由**保留"高斯感知线索"的轻量时空网络**（当前帧 + 前向重投影的历史帧）偿还。

结果：**直接吃未修改的 3DGS 资产**（零资产改动），1080p 下 **2.3–2.7× 标准 3DGS 吞吐**（1-spp 紧凑配置、场景专用网络）；4-spp / 16-spp 逐档升质（16-spp 大幅配置平均 PSNR **反超**标准 3DGS）；移动端 540p 实测 **92.6 FPS（裸渲染）/ 73.2 FPS（含 NPU 重建）**。

> **一句话定位**：库内"**取消排序**"线的第 5 个节点——前作（[[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]]）证明"排序可以不存在、噪声可以还债"；本文证明"**还债方式也可以按成本路由**"，并把整条路线搬上手机。

## Problem

标准 3DGS 的**排序 + alpha 混合**是性能瓶颈（详见 [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制]]）。**随机透明**（stochastic transparency，Enderton et al. 的经典思想）用"按 α 概率保留 / 丢弃 + 深度测试"替代有序混合，得到 alpha 混合的**无偏蒙特卡洛估计**——**免排序**，但低采样数（1-spp）噪声在自由导航下不可用。

随机透明已有的两条工程实现（本文直接建立其上）：

| 流派 | 代表作 | 成本结构 | 强项场景 |
|---|---|---|---|
| **fragment-based**（走光栅化） | StochasticSplats（Kheradmand et al. 2025） | ∝ 光栅覆盖（quad overdraw） | **大足迹**高斯（硬件加速便宜） |
| **primitive-based**（逐点采样） | Gaussian Point Splatting（Rijsdijk et al. 2026） | ∝ 采样数（不透明度 × 足迹） | **小足迹**高斯（免光栅化开销） |

**互补性就是机会**：一条流的贵区恰好是另一条的便宜区——**问题从"选哪条流"变成"怎么按成本路由"**。

## Core Idea

三个组件：

1. **成本感知混合光栅器**：算每个高斯的（不透明度 × 投影足迹）→ 大足迹走 fragment 流、小足迹走 primitive 流（原文路由条件形如 [α·footprint 阈值]）；两流用同一有效 α 函数与独立接受事件——**路由不破坏无偏性**（"每个基元只进一条流"）；
2. **高斯感知的时空重建网络**：随机渲染的每个样本附带**局部描述子（observation）**；把**当前帧 + 前向重投影的历史帧**的 observation 聚合，轻量 CNN 输出干净图像——**跨场景训练**的模型可直接在未见场景部署（无需逐场景重训、无需改资产）；
3. **扫描线外的工程层**：场景专用 vs 跨场景（Mini.）两种网络；移动端 = **FP16 全 MLP 跑 MediaTek NPU + 光栅化跑 Vulkan compute**，零拷贝缓冲、**NPU 推理第 N 帧与 GPU 渲染第 N−1 帧重叠**。

## Results（原文数据）

**桌面（1080p，renderer-only FPS / 质量见原文 Table 1）**：

| 配置 | MipNeRF360 FPS | PSNR | 说明 |
|---|---|---|---|
| 标准 3DGS（参考） | 122 | 28.90 | 排序 + 混合 |
| StochasticSplats | 665 | 17.29 | 单流随机（1spp 裸噪声） |
| Gaussian Point Splatting | 621 | 17.28 | 单流随机 |
| **Ours 1-spp + S** | **880** | 17.28→（含 S 网络后 27.72） | **2.3–2.7× 3DGS 吞吐** |
| Ours 4-spp + S | 160 | 28.23 | 中档 |
| Ours 16-spp + L | 40 | **29.08（反超 3DGS）** | 高档：质量换吞吐 |

- **"1-spp / 4-spp / 16-spp"天然就是三档质量旋钮**（成本函数统一的档位语言）；
- 场景专用训练：**2.3–2.7×** 标准 3DGS 吞吐（1080p）；**跨场景网络**（源场景训一个 Ours-S，直测 DL3DV 五个未见场景）：**353 FPS / 21.53 PSNR**（高于 4-spp 随机基线 ~17.6，低于 3DGS 22.29）；**Mini-Splatting 紧凑资产 × 本文管线**：345 FPS / 26.92 PSNR（紧凑表示组合实测）；
- **移动端（540p）**：裸渲染 **92.6 FPS**（vs StochasticSplats 53.3 / GPS 49.0 / SortfreeGS 38.2 / MobileGS 30.4）；**含 NPU 重建 73.2 FPS**——对 3DGS-GL（7.15 FPS）是量级差；
- 移动端另有"算法不变"的工程刀：GPU 常驻剔除 + indirect draw、SH 求值前的几何剔除、RGB 前的可见性求值、48 字节投影记录、四顶点 quad 展开。

## Why It Works

- **成本互补是结构性的**：两条随机流的成本函数（覆盖 vs 采样数）方向不同——**路由 = 把每个基元放进它便宜的那条流**，总成本 ≈ min 而不是和；
- **噪声可还债**（承接前作判据）：1-spp 的噪声统计结构（每样本带高斯描述子 + 时域可重投影）配一个轻量网络即可偿还；
- **跨场景可迁移**：重建网络学的是"随机渲染噪声的统计规律"而非场景内容——**这份"规律"与场景无关**（与 [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]] 的"无干净引导时 trust 必须靠学"同源）；
- **不改资产**：不重训高斯、不预处理——**只换渲染管线 + 外挂神经层**（部署阻力最小的形态）。

## Limitations

- **质量仍有缺口**：1-spp 配置 PSNR 27.7 vs 3DGS 28.9（约 1.2 dB），要靠 16-spp 才反超（吞吐回落到 40 FPS）；
- 需要**训练重建网络**（场景专用或跨场景之取舍）；跨场景（Mini.）质量再降一点；
- 移动端为**平台特定实现**（MediaTek NPU）——换平台需重做（原文自述）；
- 仍属**近似渲染**：确定性排序渲染在极限质量场景仍是上限参考。

## Game Development Relevance

- **排序线意义（库内主线）**：本库"取消排序"线的**第 5 节点**——① 优化排序（TileGS/FlashGS）→ ② 顺序无关（weighted-sum）→ ③ 重设计基元 → ④ 随机光栅化全删（StochasticSplats / 北大 Stochastic GS Denoising）→ **⑤ 混合路由 + 跨场景重建 + 移动端**（本文）。**④→⑤ 的进步 = "单流随机 → 按成本双流路由"**；
- **移动端直接对照**：**92.6 / 73.2 FPS@540p** 是库内 GS 论文第一次给出**移动端 NPU 全管线实测**——对五档体系的 Android 三档是稀缺数据点（与 [[2026-09-29-ControlGS — Conditioning Neural Gaussians for Downstream-Processing-Aware XR Rendering|ControlGS]] 的"运行时状态条件化"合流：**移动端 GS 的全部动作都在围绕"成本函数 + 神经层"重写**）；
- **"档位 = 采样数"的现成语言**：1-spp / 4-spp / 16-spp 三档 = 成本与质量单调挂钩——**分档体系在 GS 管线里有了具体形态**；
- **"资产不变量"**：直接吃未修改 3DGS 资产——**内容管线零改动**（对"资产一次做、多平台分发"的流程是好形态）。

## Unreal Engine Relevance

- GS 目前仍非 UE 官方管线成员（社区 / 实验性接入为主）；本文对引擎侧的参考价值在**两个形态**：① **"渲染管线 + 外挂神经后处理"**（与 DLSS 式后置神经层的预算形态同构——神经层成为固定开销项）；② **移动 NPU 管线参考**（FP16 NPU 推理与 GPU 渲染重叠——"算力异构化"在 GS 上的首次完整演示）。

## Technology Evolution

```text
随机透明（Enderton 等，2010s 图形学思想）
        ↓
StochasticSplats 2025 / Gaussian Point Splatting 2026：单流随机（两条平行实现）
        ↓
Stochastic GS Denoising（PKU, 2026-09）：随机光栅化 + 时域去噪（2.1×）
        ↓
【本文】混合路由（fragment ⊕ primitive）+ 跨场景时空重建 + 移动端 NPU ★ 入库
        ——"成本互补 → 按成本路由"，GS 免排序路线进入"系统化"阶段
```

## Relationships

### Based On

- 随机透明（Enderton et al.）——"用离散可见性样本估计连续混合"的经典思想；
- StochasticSplats（fragment 流）与 Gaussian Point Splatting（primitive 流）——两条被路由的上游实现。

### Extends

- [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising]] —— ④ 家族前作（单流 + 去噪）；本文把"去噪"升级为"**先分拣再重建**"，并给出移动端完整形态。**升级点**：前作去噪器带场景泛化但整条管线偏桌面；本文的"跨场景重建 + NPU 流水线"是工程化的下一站。

### Related

- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]] —— 同题对偶的另一端（把排序做便宜 vs 把排序删掉）；
- [[Compact Neural Appearance Models for Efficient Gaussian Splatting]] —— 同为"GS 工程化三刀"之一（那条是**存储/表示**侧压缩，本条是**渲染管线**侧置换）；
- [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制]] —— 排序瓶颈的机制说明。

### Followed By

- 待观察：重建网络的小型化（移动端 NPU 推理占比）、与 4D / 动态 GS 的组合、其他移动平台的移植。

## Personal Knowledge State

Current Level: **Normal**（GS 之上的新研究；你已 Easy 的 GS 基础不复述）。拿走三层：

1. **"成本互补 → 按成本路由"**——与你的预算思维同构：**每个基元都该走它便宜的那条路**；
2. **"档位 = 采样数"**——1/4/16-spp 三档的质量-成本单调语言；
3. **"移动端全管线实测"**——92.6 / 73.2 FPS@540p + NPU 异构流水线，是 Android 档位讨论的新基线。

## Learning Value

- **对分档工作**：GS 管线的三档实测（1/4/16-spp × 桌面/移动）——**"档位 = 换采样数 + 换重建网络规模"**的完整样本；
- **对"取消式优化"线**：⑤ 号节点到齐后，"排序条线"可作为**一条完整个案**（从优化 → 置换 → 路由的系统化演进）在需要时成文；
- **对神经层预算**：移动端重建网络 = 继超分 / 生成 / 光追之后的**第四个神经预算项在 GS 上的落地实例**（NPU 上跑，与渲染重叠）。

## Visualization

（本日主图解挂在"缝合对子"上：[[Kovar — Motion Graphs (2002)]] / [[2026-09-29-Length-varying Neural Motion Stitching via Cluster Transition Graph]]；GS 排序线五节点全景可参照 [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制]] 与本文 Evolution 段）

## Notes

- **窗口状态**：9-29 提交、**未公告先见**（API `submittedDate` 通道捕获，run 前不在任何 listing 分组）——双通道分工第 7 次验证；
- 机构：浙江大学 CAD&CG 国家重点实验室（Yuchi Huo 组——库内 9-30 曾评 ZJU"平面反射 GS SLAM"为 B；本篇为同校新作，量级不同）；
- 质量口径：三个标准 GS 基准（Mip-NeRF360 / Tanks&Temples / Deep Blending）；移动端测 Truck / Room / DrJohnson 三场景；
- 与 [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]] 的**关键区别**：前作用"每像素深度排序的 2DGS"当宿主满足视角一致性；本文**直接在标准 3DGS 资产上做**——两者的宿主假设不同（工程含义：本文的接入面更宽）。
