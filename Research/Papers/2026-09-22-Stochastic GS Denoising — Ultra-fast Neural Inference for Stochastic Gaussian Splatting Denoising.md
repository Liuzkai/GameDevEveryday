---
type: paper
title: "Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising"
authors: [Chenxiao Hu, Hao Zhang, Yanchen Zhang, Meng Gai, Guoping Wang, Sheng Li]
year: 2026
published: 2026-09-22
venue: "arXiv 2609.25604（cs.CV 主分类 + cs.GR 交叉；预印本；北京大学）
         —— 验证平台：2DGS + Vulkan/NVRHI 自研渲染器（RTX 3090）"
url: "https://arxiv.org/abs/2609.25604"
code: ""
project_page: ""
category: [gaussian-splatting, real-time-rendering, denoising, neural-rendering, pipeline]
importance: A-
historical_importance: 2
game_relevance: 4
production_readiness: Research
user_level: Normal
status: unread
---

# Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising

## TL;DR

**把 GS 的"排序 + alpha 混合"整个删掉，用"随机保留 + Z-buffer"代替，代价（1spp 蒙特卡洛噪声）用 ~1 ms 的时域神经去噪器补回来 —— 净账是赚的。**

- 全管线 **3.75 ms vs 排序混合版 17.96 ms（2.1×）**；裸随机光栅化 2.71 ms（2.9×）；
- 自由导航画质 **PSNR 29.80**（排序参考 32.96），把"没去噪的 TAA"（21.48）和"通用去噪器 SVGF"（25.57）都远远甩开；
- 去噪器开销 **≈ 1 ms 且几乎与场景无关** —— 一次典型的"**取消一个瓶颈 + 新增一个固定成本**"的管线置换。

> **一句话定位**：库内 [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制|"排序是 GS 的瓶颈"]] 这一判断，今天拿到**第三条出路** —— 不是优化排序（TileGS/FlashGS 路线），也不是换混合模型，而是**从管线里把排序删掉**，让"噪声"成为唯一需要偿还的债务。

## Problem

GS 的经典管线 = **每帧全局深度排序 + alpha 混合**。它的成本**随三个量线性增长**：基元数、视锥大小、输出通道数（原文 §1）。已有三条应对路线：

| 路线 | 代表 | 做法 |
|---|---|---|
| ① 优化排序混合 | FlashGS / Speedy-Splat | 保持结构，做剔除与调度 |
| ② 顺序无关透明 | weighted-sum / hybrid transparency | 去掉全局排序（仍混合） |
| ③ 重设计基元 | surfel / depth peeling | 换几何锚定方式 |
| **④ 随机光栅化（本文）** | StochasticSplats 2025 | **排序与混合全部删除** |

④ 的原理：每个片元**按不透明度概率决定"留 or 丢"**，被留者以不透明方式过普通深度测试 → 得到的是 alpha 混合结果的**无偏蒙特卡洛估计**。**但 1spp 的噪声在自由导航下完全不可用** —— 原版依赖多帧累积（假设静态相机）；通用 TAA/SVGF **失效**，因为：

1. 没有**无噪声的 G-buffer** 可作引导（随机光栅化只吐 visibility buffer）；
2. 每高斯排序不满足**视角一致性**（reprojection 前提）——本文用"**每像素深度排序的 2DGS**"作宿主来满足它。

## Core Idea

**为一个"只有噪声流、没有干净引导"的渲染器设计一个时域去噪器。** 三个组件：

**① 双 EMA 路径 —— 两个互补的估计器**

| 路径 | 输入 | 特性 | 历史上限 |
|---|---|---|---|
| **accumulated（累积）** | 原始随机样本 | 无偏、收敛后最好；**开头几帧最噪** | 128 帧 |
| **denoised（去噪）** | 空间滤波后的帧 | 立刻稳定但偏模糊；**覆盖短历史与 disocclusion** | 24 帧 |

**② 方差门控（variance gate）** —— 按 accumulated 路径的**标准误**逐像素混合：

$$g = 1 - r\bigl(1 - \mathrm{clamp}(\gamma\cdot \mathrm{err}, 0, 1)\bigr),\quad r = 1-e^{-n_s/\tau}\ (\tau=8),\quad \mathrm{err}=\sqrt{\mathrm{var}/(n_s+\epsilon)}$$

早期 $r\approx0$ → 信去噪路径（先抑噪）；历史积累后 → 逐步切回累积路径（要锐度）；误差尖峰（运动/遮挡）时拉回去噪路径。

**③ 学习 trust 取代人类启发式** —— 没有干净引导，于是把预算花在"**逐像素预测历史该信多少**"上。trust 由一个小 CNN 从 motion / depth / photometric 线索回归；**消融证明这是全系统最关键组件**（见下）。相机静止时**强制 $t_s=1$**（此时样本 i.i.d.，运行均值就是最优估计）。

**④ 空间滤波：几乎免费的各向异性核** —— 固定 4 层 mipmap 滤波（EWA 精神），核形状由**烘进高斯的 per-surfel 可学习 payload**（4 通道 neural-view 张量 + 每高斯 10 个 float）驱动；单像素滤波成本**≤ 32 次 texel 读取**。

**⑤ STAB 稳定化** —— 轻量 TAA 变体（邻域钳制 + 颜色一致性救援），压掉残余抖动（代价是少量 ghosting）。

## Key Numbers（全文表格核对）

**自由导航（MipNeRF360 7 条手录轨迹，对照排序参考）**

| 方法 | PSNR↑ | SSIM↑ | LPIPS↓ | CVVDP↑ | ms/帧↓ |
|---|---|---|---|---|---|
| Ref(1024)（上限） | 32.96 | 0.950 | 0.080 | 8.73 | – |
| Raw 1spp | 18.26 | 0.282 | 0.671 | 4.94 | 2.71 |
| ST-TAA（原版+累积） | 21.48 | 0.418 | 0.615 | 5.95 | 2.88 |
| SVGF（通用去噪器） | 25.57 | 0.785 | 0.297 | 6.29 | 2.86 |
| **Ours** | **29.80** | 0.867 | 0.244 | **7.57** | **3.75** |

**性能（每帧毫秒，7 轨迹均值）**：2DGS 官方实现 17.96 → 硬件光栅化重写 7.99 → 裸随机 1spp 2.71 → **本文全管线 3.75**。
→ 裸随机 = **2.9×**；全管线 = **2.1×**；去噪器 ≈ +1 ms 且近乎常数。加速比随内容浮动：**1.2×（厨房，低 overdraw）～ 3.5×（bicycle）** —— 与"排序混合的成本结构"一致。

**静态收敛**：16 帧输出距自身 128 帧质量仅差 **1.1 dB**（比 ST-TAA 收敛显著快）；128 帧在等采样下小胜 ST-TAA。

**消融（garden 轨迹）**：完整 26.63 / 去掉 trust 预测（$t_s\equiv1$）**22.51** / 换成启发式 trust **25.82** / 去掉空间滤波 23.52 / 去掉 STAB 25.94。
→ **学习 trust 是最大单组件；启发式（即使网格搜索出最优 σ）仍差 0.8 dB** —— "无干净引导时，引导要靠学"。

## Why It Works

- **账本结构**：排序混合的成本 ∝（基元 × overdraw × 通道）；随机管线 = **每像素固定成本 + 固定去噪开销**。场景越重，前者越贵、后者不变 → 置换在重场景稳赚；
- **噪声是"可还的债"**：蒙特卡洛噪声有明确统计结构（无偏 + 方差可估），恰是最适合时域+神经方法处理的一类误差；
- **双路径设计把"收敛速度"与"渐近质量"解耦**：去噪路径买时间（开头不噪），累积路径买质量（长期更锐），门控按不确定度自动切换 —— **一个可以搬走的架构模式**。

## Limitations

- **并不"全面更快"**：作者明确说，与高度工程化的排序管线（FlashGS）、顺序无关合成器（Mobile-GS）、紧凑 surfel 混合（Ye 2025）**绝对吞吐尚不能比**；本文的贡献被刻意收窄为"**去掉挡住随机路线的 MC 噪声**"；
- **仍有可见闪烁**（trust 网络容量所限），需靠 STAB 以少量 ghosting 换取稳定；
- 相对排序渲染器**仍有 PSNR 缺口**（29.80 vs 32.96 上限；静态端 28.02 vs 28.25）——宿主几何转换本身损失 ~1 dB；
- 场景特定 payload（每高斯 10 float）需逐场景训练；zero-shot 版质量略降（导航 PSNR −0.46）。

## Game Development Relevance

- **它是"换管线"这个动作最纯的样本**（[[Scalability and Quality Tiers]] 的 W39 议题）：不是"把排序调便宜"，而是**移除整个排序阶段 + 新增一个神经后处理阶段**。给"档位 = 换参数还是换管线"提供一张真实的账：**换管线的收益不是百分比，而是"成本函数变了"**（从 ∝ 场景复杂度 → 常数去噪开销）；
- **"神经后处理"的第四个预算项**：继 DLSS 5（超分/生成）、NLM（神经光追）、Arm NSS（神经阴影）之后，**去噪器也进了"神经层成本清单"** —— 且这里的数字异常干净：**+1 ms，几乎常数**。对预算体系：神经后处理可以按"**固定开销 + 显存**"建模，不随 SABC 分级浮动；
- **"没有干净引导怎么办"**：传统 TAA/SVGF 的世界观是"有 G-buffer 就够"；本文给出**失去 G-buffer 时的替代路线**（学习 trust）。对任何"抛弃了传统 G-buffer 的管线"（如未来的 GS / 点云 / 神经表示管线）都是可复用判据；
- **零成本副产品**：随机光栅化**天然产出"带噪深度缓冲"**，可能用于分层 Z 剔除（原文 future work）。

## Unreal Engine Relevance

- 不直接映射 UE 现有模块（验证平台为 Vulkan 自研渲染器 + 2DGS 宿主）；GS 也尚未进入 UE 主渲染管线；
- 但**架构模式可映射**：①"两个互补估计器 + 不确定度门控"与 UE 的 TSR 后处理哲学同构；②"去噪器坐在流之外、只依赖 visibility stream 的输入契约"= 一个**可插拔后处理 stage** 的设计范式（原文 §6 明说：与剔除/压缩/更快的随机光栅化**组合而非竞争**）。

## Technology Evolution

```text
2023  3DGS：排序 + alpha 混合成为事实标准
        ↓
2024-25  三条工程路线并行：
        优化排序（FlashGS）、顺序无关（weighted-sum）、换基元（surfel/peeling）
        ↓
2025  StochasticSplats：第四条路线 —— 排序与混合全删，但噪声无解（需静态相机）
        ↓
2026  ★ 本文：给随机路线配上"时域神经去噪器" → 自由导航可用 + 2.1× 净收益
        （宿主要求：视角一致性 —— 用"每像素排序 2DGS"满足）
        ↓
候选方向：零成本带噪深度 → 分层 Z 剔除；与更快随机光栅化/更紧凑表示组合
```

## Relationships

### Based On

- **StochasticSplats（Kheradmand et al. 2025）** —— 随机光栅化的来源；本文移除其"噪声无解"限制
- [[Gaussian Splatting]] —— 宿主表示（每像素排序 2DGS 变体）；本篇是"GS 之上的新研究"（[[Personal Knowledge Model]] 中"GS 基础推送已停"后的预期方向）
- **TAA（Karis 2014）** —— 被借鉴（STAB）也被超越（不带干净引导不足以工作）
- **SVGF（Schied 2017）** —— 被设为"通用去噪器"对照，暴露出"缺干净引导"时的失效模式

### Related

- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]] —— **同题对偶**：TileGS 把排序做便宜（局部化），本文把排序取消 —— **"GS 排序瓶颈"的两端答案**
- [[From Splats to Silicon — Rethinking Computational Efficiency of 3DGS]] —— 同为"先量账再谈优化"：一个量数据搬运，一个量管线置换后的**每毫秒账**
- [[GS 图解 2 — 排序瓶颈、管线冲突与密度控制]] —— 排序瓶颈的机制说明（blending 不交换 × 百万级 × 每层）
- [[Temporal Stability and Artistic Intent]] —— "抖动 vs ghosting"的取舍再次出现（STAB 的代价）
- [[Neural Upscaling and Frame Generation]] / [[Real-Time Global Illumination]] —— 神经组件进入实时管线的同族实例（此处是"神经去噪"）

## Personal Knowledge State

- **user_level: Normal**。属于"GS 之上的新研究"（GS 本体已 Easy，本篇不讲 GS 基础）；前置（TAA、时域累积、去噪器基本概念）在你的日常域内；
- **三条一句话检验**：
  1. 能说出"随机光栅化**取消**了排序，把成本从'∝ 场景复杂度'换成'每像素 + 固定去噪开销'"；
  2. 能说出"**没有干净 G-buffer 时，启发式 TAA/SVGF 失效，trust 必须靠学**"；
  3. 能说出"双路径 = **无偏慢收敛 + 有偏快稳定**，按不确定度混合"。

## Learning Value

- **对分档工作的直接弹药**：提供"换管线"账本的具体形态（成本函数改变 + 新增固定项 ≈ 1 ms）；
- **对"神经层预算"的补位**：去噪是继超分/生成/光追之后的第四个神经预算项，且量级已知（常数级）；
- 方法论："噪声可还债"——**评估任何被"随机化"加速的管线时，先问它的噪声统计结构是否可被时域+学习方法偿还**。

## Visualization

![[GS_随机光栅化_排序取消与去噪成本置换图解]]

## Notes

- 通讯作者 Sheng Li（PKU）；作者邮箱字段在 HTML 版有笔误（Hao Zhang 行显示 lisheng@pku.edu.cn），以作者列表为准；
- 原文提供补充材料与复现说明（supplements）；两个演示视频（YouTube / Bilibili）在 comment 字段；
- 与库内 9-22 档案的关系：**本文于 9-22 提交、9-24 listing 放出**（属 9-23 分组），当日窗口交叉核验（API `submittedDate`）与 listing 双通道同时覆盖。
