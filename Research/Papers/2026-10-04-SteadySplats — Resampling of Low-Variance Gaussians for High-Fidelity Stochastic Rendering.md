---
type: paper
title: "SteadySplats: Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering"
authors: [Felix Windisch, Thomas Köhler, Lukas Radl, Chris Wyman, Georgios Kopanas, Bernhard Kerbl, Markus Steinberger]
year: 2026
published: "2026-10-04（arXiv v1）；2026-10-06（v2 修订；Tue 10-6 公告组）"
venue: "arXiv Preprint（cs.CV 主分类 + cs.GR；2610.05576）"
url: "https://arxiv.org/abs/2610.05576"
code: ""
project_page: ""
category: [gaussian-splatting, real-time-rendering, transparency, denoising, resampling]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: "Research（Vulkan 原型渲染器；全管线在 RTX 5090 上实测；核心组件——颜色方差正则化 + 时空重采样——与既有 3DGS 训练管线兼容，但无引擎集成）"
user_level: Normal
status: unread
aliases: [SteadySplats, 稳态高斯, Stochastic OIT resampling, 随机透明重采样, low-variance gaussians]
tags: [gaussian-splatting, real-time-rendering, transparency, denoising, resampling]
---

# SteadySplats: Resampling of Low-Variance Gaussians（Windisch et al. 2026）

> **入库 2026-10-07（Run 29）。** 作者阵容 = **3DGS 原班人马 + NVIDIA RTX 组**：**Bernhard Kerbl（3DGS 一作）**、**Georgios Kopanas（3DGS 共同作者）**、**Chris Wyman（SVGF 共同作者）**、Steinberger（TU Graz）。
> **一句话定位**：库内"取消排序"线（[[Gaussian Splatting]] 5 节点表）的**第 6 节点，且是"收口"性质的一个**——此前该线的公认死穴是"**噪声抵消了性能收益**"（图 1 的动机段原话：*"accumulating hundreds of samples per pixel… directly negates the performance advantages of sorting-free rendering"*）；本篇从**表示（训练）+ 合成（重采样）两端同时降噪**，把 1 spp 从"能用但要忍"变成"可用"，并声称**收敛到与排序版 3DGS 的差距 L1 < 10⁻⁴**。

## TL;DR

**随机顺序无关透明（stochastic OIT）是绕开 GS 排序瓶颈的最优雅路线，但一直因可见噪声而不实用。** 本篇在**两端**同时下手：
1. **训练端（表示）**：颜色方差正则化 $\mathcal{L}_{var}$——惩罚"高方差基元配置"，让 3DGS 模型**天生少噪**（"生来就不欠噪声债"）；
2. **推理端（合成）**：**历史感知的空间重采样**（加速收敛）+ **时域重要度重采样**（镜头运动下保持一致；含软历史拒绝、速度自适应置信度、逐基元 STBN 蓝噪声）。

结果（Mip-NeRF 360，21 条相机轨迹、3,152 帧、对齐排序混合 ground truth）：**1 spp 下 31.44 dB vs 前作 18.34 dB（+13.1 dB）**；叠加时域重采样到 **35.09 dB**；**收敛极限与排序版 3DGS 的 L1 误差 < 10⁻⁴**。

## Problem

随机透明（stochastic transparency）对 GS 的吸引力："efficient and elegant"——但输出固有噪声，**人眼对高频噪声敏感**，要压到不可见需要累积**数百 spp**，直接把排序免掉省下的性能又吃掉。此前的噪声处理：超采样（更贵）或时域累积（伪影）——本篇要的是"**在低样本数和高样本数两端都成立的原理性降噪**"。

## Previous Work

- **基线**：**StochasticSplats**（ICCV 2025，Kheradmand / Vicini / Kopanas / Lagun / Yi / Matthews / Tagliasacchi）——排序免 + 随机透明；本篇的直接对照（1 spp: 18.34 dB）；
- 相关：加权和/混合透明（顺序无关但质量下降）、K-buffer OIT、**SVGF** 一类的时域降噪传统（Wyman 本人即 SVGF 作者——本篇把"时域复用"的成熟技艺移植进 GS 随机透明）。

## Core Idea

**"与其偿还噪声债，不如不让它欠"（个人概括）。** 三个组件、两处下手：

| # | 组件 | 位置 | 机制 |
|---|---|---|---|
| ① | **颜色方差正则化** $\mathcal{L}_{var}$ | 训练（表示层） | 惩罚高方差配置 → 模型沿视线的颜色方差先天更低；训练 15k 步再加 15k 步 $\lambda_{var}$ 调参 |
| ② | **历史感知空间重采样** | 推理（像素合成） | 邻近样本按权重复用（"Neighbor Reuse" + 近似权重）→ 大幅加速收敛；K-buffer 承载 |
| ③ | **时域重要度重采样** | 推理（时间轴） | 重投影 + 邻域颜色矩 + **软历史拒绝** + **速度自适应置信度** + **逐基元时空蓝噪声（STBN）** |

## Technical Approach（数字要点）

- **基线对照表**（Mip-NeRF 360，vs 排序 α-blending ground truth，1 spp）：

| 方法 | Avg. PSNR | SSIM | LPIPS | tPSNR | Indoor | Outdoor |
|---|---|---|---|---|---|---|
| StochasticSplats（ICCV 2025） | 18.336 | 0.293 | 0.667 | 15.235 | 19.562 | 16.701 |
| StochasticSplats + 时域 | 21.954 | 0.552 | 0.524 | 19.450 | 23.313 | 20.141 |
| **Ours** | **31.437** | **0.845** | **0.345** | 28.410 | 32.978 | 29.381 |
| **Ours + 时域** | **35.085** | **0.937** | **0.209** | 32.449 | 37.161 | 32.318 |

- **方差-质量权衡（原文自述）**：$\mathcal{L}_{var}$ 会**稀释 L1/SSIM 训练信号**——完全收敛模型的峰值指标略降；但"**小量方差正则化带来几乎普遍增益**"（前 200 spp 全指标超越基线）；**最优 $\lambda_{var}$ 随采样数下降**（低 spp 用大、高 spp 用小）——原文图 5；
- **收敛/无偏簿记（§10 Bias——诚实细节）**：空间复用引入**非严格无偏**（自归一化重要性采样）；纯随机的"无偏性"在 **L1 上 136 spp、LPIPS 上 33 spp** 后开始反超——**即"重采样主导 → 无偏主导"有一个交叉图**（§9.3 Pareto 分析）；
- **STBN**：逐基元时空蓝噪声替代原 RNG——约 3 spp 后带来"小但不可忽略"的持续增益（图 9）；
- **性能**：与随机/确定性混合在**同一 Vulkan 渲染器**内对照（RTX 5090）；空间复用的 K-buffer 开销"小"（大 buffer 亦在可接受范围，图 7）——**每个增益都明码标价**。

## Key Contribution

1. **首个把随机 GS 的噪声问题"在源头两端"系统性解决的工作**——训练正则化（表示）+ 时空重采样（合成）；
2. **1 spp 可用性**（+13.1 dB 对前作）与**高样本端收敛回排序质量**（L1 < 10⁻⁴）——把"取消排序"从"牺牲画质换速度"改写为"**可以用更少的样本拿到同等画质，甚至更快**"；
3. 完整开源级工程细节（Vulkan 全管线、伪代码 §7、有限历史/有偏性讨论 §10）。

## Limitations

1. **非严格无偏**（空间复用的代价）——需要自归一化处理，且高样本端"无偏性反超"（交叉点已量化）；
2. **K-buffer 开销**：填充/读取邻域样本的显存与时间随 buffer 尺寸增长（"optimal buffer size scales with time budget"）；
3. **$\mathcal{L}_{var}$ 稀释训练信号**——峰值质量略降（权衡已给出）；
4. 未给出引擎集成/移动端路径（Vulkan 桌面原型）。

## Game Development Relevance

**4/5。它是库内 GS 排序线拖了三周的"最后一公里"收口，而且就在你 Easy 的 GS 域之上——纯增量、无基础内容。**

- **排序线全景更新为 6 节点**（沉淀进 [[Gaussian Splatting]]）：① 优化 → ② 顺序无关 → ③ 重设计基元 → ④ 随机光栅化（全删）→ ⑤ 随机路由 → **⑥ 随机性驯服（表示+重采样双侧降噪）**；
- **"档位 = 采样数"语言再获支撑（推断）**：1 spp（移动/低配）→ 4–16 spp（中端）→ 100+ spp（收敛到排序质量）——**"spp 就是 GS 的档位旋钮"** 这一判断现在有了"每一档都干净"的前提（此前 1 spp 档是"脏"的）；
- **与 [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]]（北大，9-22）构成同题双解**：那条路是**学一个神经去噪器**（运行时 +1 ms 常数、网络偿还噪声债）；本篇是**原理性重采样 + 训练正则化（无网络推理）**——"**测干净 vs 生干净**"两种哲学，可对照阅读；
- **训练期正则化 = 新成本项（推断）**：做 GS 资产管线时，"训练时用哪个 $\lambda_{var}$"会成为一个**生产参数**（低样本设备交付低-λ 模型？）；与 [[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]] 的"部署时单参数 c"呼应——**GS 资产正在长出"交付参数"这一层**。

## Unreal Engine Relevance

- 无直接 UE 映射（GS 尚未进 UE 主线）；但机制可迁移（推断）：**"训练期把资产做得对下游渲染有利"** 的思路，与 UE 侧 Nanite/虚拟纹理的"离线烘焙换运行时"同构——**"资产质量不只由保真度定义，还由'它对渲染器友不友好'定义"**。

## Technology Evolution

```text
随机透明（OIT）传统（2000s–2010s，图形学通用技术）
        ↓ 移植到 GS
StochasticSplats（ICCV 2025）：随机光栅化，噪声裸奔（18.3 dB @1spp）
        ↓
2026-09 Stochastic GS Denoising（北大）：神经去噪器偿还（"测干净"）
2026-10 ★ SteadySplats（本篇）：训练正则 + 时空重采样（"生干净"）
        → 1 spp 31.4 dB；高样本收敛回排序质量（L1 < 1e-4）
        → 排序免渲染的"画质税"基本取消
```

## Relationships

### Based On

- StochasticSplats（ICCV 2025）——问题与基线（库外记名）
- SVGF 时域降噪传统（Wyman 血统）——重采样机制的技艺来源

### Improves

- StochasticSplats——1 spp +13.1 dB；高样本端收敛到排序质量

### Contrasts

- [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]]——同题双解："学一个去噪器" vs "让模型和样本自己干净（无网络推理）"

### Related

- [[Gaussian Splatting]]（排序线第 6 节点）；[[2026-09-29-Gaussian Stippling — Efficient Sorting-Free 3D Gaussian Rendering|Gaussian Stippling]]（同家族：噪声的分布形态也被研究）

## Personal Knowledge State

- **user_level: Normal（推断）**——GS 基础你已 Easy；本篇全部价值在"降噪机制 + 采样-质量权衡"两层，不含基础内容。
- **读法（25 分钟）**：动机段（"negates the performance advantages"）→ 三组件表 → 结果表 → §10 有偏性讨论（诚实细节）→ 图 5（λ 与样本数的关系）。

## Learning Value

- **一条判据**：**"噪声从哪来，就从哪还。"** 前作从输出端测掉（网络）；本篇从生成端（训练目标）与合成端（重采样）两头还——**评估任何降噪方案先问"噪声的根源在第几层，你的修补在哪一层"**。
- **一条预算纪律**：**"非严格无偏"也是一种明码标价**——工程上"近似但可控 + 交叉点已量化"比"理论完美"更值钱（原文把交叉点画出来了：L1 136 spp / LPIPS 33 spp）。

## Visualization

（暂无——三组件表 + 结果表已表达；如后续需要，可将"1spp → 高 spp 的质量-成本 Pareto"制成图。）

## Notes

- **来源核对**：arXiv 2610.05576 **v2**（HTML 全文抽取：结果表 18.336/31.437/35.085、+13.1 dB、L1 < 10⁻⁴、λ∈[0,5]、136/33 交叉点、STBN、RTX 5090 / Vulkan 均逐条核对）。
- **窗口状态**：v1 提交 10-4、v2 修订 10-6；由 Tue 10-6 公告组捕获（cs.CV 主分类——**双通道分工第 N 次验证**：API 窗口因提交日在窗口外查不到、recent 页公告分组命中）。
- **作者阵容核对**：Bernhard Kerbl = 3DGS（SIGGRAPH 2023）第一作者；Chris Wyman = SVGF（HPG 2017）作者之一、NVIDIA；Georgios Kopanas = 3DGS 共同作者。**"3DGS 原班 + SVGF 血统"做随机透明降噪——阵容本身就是信号。**
