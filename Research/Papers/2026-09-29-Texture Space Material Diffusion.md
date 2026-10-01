---
type: paper
title: "Texture Space Material Diffusion"
authors: [Jacob Munkberg, Peter Kocsis, Jon Hasselgren]
year: 2026
published: "2026-09-29 (arXiv v1)"
venue: "arXiv Preprint（NVIDIA）"
url: "https://arxiv.org/abs/2609.37654"
code: ""
project_page: ""
category: [rendering, materials, generative, diffusion, asset-pipeline]
importance: B+
historical_importance: 0
game_relevance: 4
production_readiness: Research（离线工具形态；多视图重建 17 视角 @2K 约 130 s / GB300）
user_level: Normal
status: unread
tags: [materials, texture, diffusion, nvidia, asset-pipeline, pbr]
---

# Texture Space Material Diffusion（NVIDIA, 2026）

## TL;DR

**把"材质生成"整个搬进纹理空间（UV 2D 域）做**：微调一个视频扩散 Transformer，让它直接在**纹理空间**里生成完整 PBR 材质（basecolor / height / roughness / metalness）。关键洞察是——**"从图像空间到纹理空间的投影是已知的"**，把这一步已知几何关系当**归纳偏置**写进模型，于是：

- 扩散过程**跨任意几何与 UV 参数化泛化**（学的是"材质长什么样"，不是"某个视角长什么样"）；
- **天然规避**多视图/视频扩散的**视图一致性问题**（纹理空间里一致性是结构性的，不用学）；
- 因为纹理空间是二维的，可以**直接复用视频扩散模型的强先验**（把"多视图渲染帧"当"视频帧"处理）。

能力集：文生材质 / 多视图照片 → 材质（**未知光照下**解调，输出完整 PBR 贴图，含 height）/ **材质升分辨率**（UV 图集上训的生成式超分，推理时可扩到 **8K 贴图 + 100+ 输入视图**）；并验证可泛化到**神经材质表示**。

> **一句话定位**：nvdiffrec 团队的下一步——**"在正确的空间里做生成"**：不在图像空间里追视图一致性，而是把问题搬到一致性天然成立的纹理空间。库内同族：[[2026-09-17-PBR-Latent — Physically Based Rendering in the Latent Space|PBR-Latent]]（把渲染方程改写进潜空间）——**"换空间"家族第 2 例**。

## Problem

材质的生成与重建现状（图像空间路线）：

1. **多视图/视频扩散生成材质**：要处理跨视图一致性（模糊、闪烁、细节漂移）；视图数越多越难；
2. **单图/少图重建**：光照与材质解耦（de-lighting）是出名的病态问题；
3. **升分辨率**：素材分辨率不足时，图像空间升频会把"视图依赖效果"一起放大（错误固化）；
4. 游戏/影视管线的真正目标格式是 **UV 纹理图集 + 完整 PBR 通道**——图像空间方法最终都要**额外投影/烘焙**一步，误差在投影处累积。

## Core Idea

**问题空间重写**：把资产材质视为 **2D UV 域信号**（而非 3D 视图信号），于是：

```text
图像空间路线：  多视图渲染 → 生成模型（要学视图一致性）→ 再投影回 UV
纹理空间路线：  UV 图集（2D）→ 生成模型（一致性天然成立）→ 直接就是目标格式 ★本文
```

**三个组件**（原文）：

1. **纹理空间视频 DiT**（finetune）：把"以固定相机轨道环绕物体的多视图渲染"看作"一段视频"，帧 = 视图；**几何投影（image → texture）是已知量**，用它把生成过程约束在纹理空间；配一个新提出的 **3D-aware 旋转位置编码**（编码每 texel 的世界坐标 + 帧 ID）——让模型知道"这个 texel 在 3D 里在哪、属于哪一帧"；
2. **多视图→材质重建**：给定已知几何的拍摄照片（未知光照、未知光照方向），模型**解调光照**，输出干净的 basecolor / roughness / metallic / height；
3. **生成式超分（upscaler）**：在**材质参数的 UV 图集**上训练的超分模型（低分辨率 PBR 图作引导）；配**免训练推理时增强**（noise rolling + coverage-aware expert aggregation）——把推理规模推到 **8K 纹理、100+ 输入视图**（超过训练规模）。

## Results

- **任务**：文生材质 / 多视图材质生成 / 材质超分 / 神经材质（proof-of-concept，用 Yu et al. 的神经材质表示做替换目标）；
- **量化**：合成评测集（BlenderVault 32 个留出模型）；分布指标 CLIP-FID / CMMD（协议：336² / 224²，受 CLIP 分辨率所限）——宣称在材质生成与重建上 SOTA（量+质）；
- **对照亮点**：DiffPT **过拟合于单光照重建、relighting 失败**（光照-材质解耦失败）——本文的解耦是评测重点；
- **成本**：训练 15k iterations / **32×A100**（渐进分辨率）；**推理重**——多视图模型 **17 视角 @2K 约 130 s（GB300 GPU）**——**离线内容工具，非运行时**；
- **规模**：8K 贴图、100+ 视图（推理时增强）；支持照片（posed photos，未知光照）与文本两种条件。

## Why It Works（可迁移抽象）

1. **"已知投影当归纳偏置"**：几何（UV 映射）是已知量，不该让网络去学它——**把它写进归纳偏置，网络只需学"材质本身"**。判据：*凡是"已知的确定性变换"，都应该出现在模型结构里，而不是训练目标里*；
2. **"2D 化换先验复用"**：纹理空间是 2D → 可以直接借**视频扩散**的权重（帧≈视图）；**降维不是妥协，是给先验迁移开门**（对照 [[GS 图解 1 — 协方差与椭球：高斯的形状说明书|GS]] 的类似降维操作：2DGS 复用 2D 表示）；
3. **"一致性由空间保证，而非由学习保证"**：视图一致性在纹理空间是**定义级**成立的——省掉了整整一类损失/正则；
4. **推理时增强扩规模**：训练规模之上，用"噪声滚动 + 覆盖感知专家聚合"把 8K/100+ 视图的推理拼出来——**训练规模 ≠ 使用规模**（与 [[Müller — Procedural Modeling of Buildings (2006)]] "离线生成量 ≠ 运行时负载"同族的"规模解耦"思路）。

## Limitations

- **重推理**（130 s @17 视角 2K）——目前是**内容生产工具**的形态，不是实时方案；
- 图像模型的通用限制仍在（无视图一致性的模型会糊）；**解耦失败**在极端光照/材质上大概率仍会发生（未给定量失败率）；
- 神经材质只做到 proof-of-concept（替换目标潜在量）；
- 无代码/项目页（截至本日）；评测集为合成（BlenderVault）——真实拍摄的定量未深入（重建任务给了定性）。

## Game Development Relevance

- **TA / 资产管线正面相关**：文生材质 + 照片转 PBR + **8K 材质升分**，三个动作都在当前内容生产流程的痛点上（素材分辨率不足 → 现在的做法是重做/缝补）；
- **"兼容目标格式"**：输出直接就是 PBR 图集（含 height）——**跳过"再烘焙"这一步**是相比图像空间路线的工程优势（对照 [[SceneHI — High-Resolution 3D-Consistent Scene Texturing with Controllable Illumination|SceneHI]] 的"生成式资产烘焙对照"）；
- **"离线重活"定位**：130 s / 材质的成本形态与"离线生成量 ≠ 运行时负载"同构——**放在导出前的内容管线里是合理成本**（批量生成/升分）；
- **对分档体系**：**8K 上限 + 推理时增强扩规模**——"素材规格的档位天花板被生成式超分抬高了"（贴图尺寸预算的 2048/1024/512 上限未来面对的是"生成式放大"这类工具能力，而非采集分辨率）。

## Unreal Engine Relevance

- UE 材质管线（Virtual Texture / Nanite 材质 / Texture Graph）目前**不内置**生成式超分/纹理扩散；本类工具的现实接入形态是 **DCC 前置**（Substance 家族 / 自研管线里的批处理工具）；
- 对引擎侧的远期含义：**"纹理导入时的神经后处理层"**（与 DLSS 式推理期神经层的形态呼应）——资产分辨率预算 + 导入期生成式增强的组合。

## Technology Evolution

```text
图像空间材质生成（多视图 / 视频扩散）：Star* / DiffPT / Kocsis et al. …（一致性与解耦难题）
        ↓
nvdiffrec 线（Munkberg et al. 2022）：可微渲染反解材质/光照（优化式）
        ↓
【本文】纹理空间扩散（生成式 + 已知投影偏置 + 视频先验复用）★ 入库
        ——"空间重写 + 先验迁移"取代"在图像空间硬学一致性与解耦"
```

## Relationships

### Based On

- 视频扩散 Transformer 先验（文生视频/图像基础模型）——被复用的算力与知识来源；
- 可微渲染反解线（作者组 nvdiffrec，Munkberg 等 2022）——同组"从图像理解材质"的积累。

### Extends

- 材质生成从"图像空间任务"变为"**纹理空间任务**"——改变问题空间本身；
- 生成式超分从"图像升频"扩展到"**UV 图集升分**"（含 PBR 多通道联合）。

### Related

- [[2026-09-17-PBR-Latent — Physically Based Rendering in the Latent Space|PBR-Latent]] —— **"换空间"家族**：那条把渲染方程写进潜空间，本条把材质生成写进纹理空间；
- [[SceneHI — High-Resolution 3D-Consistent Scene Texturing with Controllable Illumination|SceneHI]] / [[RelightFormer — Feed-forward Generative Transformer for Multiview Object Relighting|RelightFormer]] —— 库内生成式材质/重光照线（本条是该线在"目标格式 + 分辨率规模"上的最强工程形态）；
- [[Physically Based Rendering]] —— 输出即 PBR 五通道，是 PBR 资产层的生成式入口。

## Personal Knowledge State

Current Level: **Normal**。三条可拿走：

1. **"已知投影当归纳偏置"**（确定性变换进结构、不进损失）；
2. **"2D 化换先验复用"**（降维是为了借用别的域的预训练权重）；
3. **"训练规模 ≠ 使用规模"**（推理时增强把 8K/100+ 视图拼出来）。

## Learning Value

- **对资产管线思维**：从"生成材质"到"**直接生成目标格式的材质**"——生成式工具与管线对接的成熟度信号；
- **对"换空间"判据**：与 [[2026-09-17-PBR-Latent — Physically Based Rendering in the Latent Space|PBR-Latent]]、[[2026-09-25-ControlGS — Conditioning Neural Gaussians for Downstream-Processing-Aware XR Rendering|ControlGS]]（换条件空间）合流——**"先问该在哪个空间里做，再问怎么做"**。

## Visualization

（无需独立图解；本日主图解见缝合对子）

## Notes

- **窗口状态**：9-29 提交，Wed 9-30 listing 公告（本次主窗口）；cs.CV 主分类、cross-list cs.GR；
- 作者组：**Munkberg = nvdiffrec 一作**（Extracting Triangular 3D Models, Materials, and Lighting From Images, CVPR 2022）——本库首次直接收录该组作品；
- 评测集：BlenderVault（合成 3D 资产）；指标 CLIP-FID / CMMD；分辨率上限受 CLIP 评测协议约束（生成能力本身到 8K）；
- 与库内"神经材质"线的关系：证明超分/生成框架**可平移**（替换潜变量为神经材质编码）。
