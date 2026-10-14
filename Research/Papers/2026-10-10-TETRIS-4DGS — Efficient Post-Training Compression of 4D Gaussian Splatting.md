---
type: paper
title: "TETRIS-4DGS: Efficient Post-Training Compression of 4D Gaussian Splatting"
authors: [Chengzhen Wu, Kelei Liu, Sicheng Li, Lu Yu, Yiyi Liao]
year: 2026
published: "2026-10-10（arXiv v1, 2610.14663；周三公告组 2026-10-14）"
venue: "ICME 2026（arXiv Preprint；cs.CV）"
url: "https://arxiv.org/abs/2610.14663"
code: ""
project_page: ""
category: [gaussian-splatting, 4dgs, compression, quantization, codec, deployment]
importance: B+
historical_importance: 0
game_relevance: 3
production_readiness: "Prototype（秒级压缩、>40×、兼容现成编解码器；代码未提）"
user_level: "Normal（结论层 10 分钟：'重要度驱动码率分配' + '把 4DGS 属性排成 codec 友好 2D map'）"
status: unread
aliases: [TETRIS-4DGS, 4DGS compression, 4D 高斯压缩]
tags: [gaussian-splatting, 4dgs, compression, deployment]
---

# TETRIS-4DGS: Efficient Post-Training Compression of 4D Gaussian Splatting（Wu et al. 2026）

> **入库 2026-10-14（Run 36）。** 周三公告组（cs.CV 周末清关件）捕获；提交 2026-10-10；ICME 2026。
> **一句话定位**：**动态高斯（4DGS）的"部署端"存储解法**——4DGS 能实时渲动态场景，但**存储开销大到无法落地**；既有压缩都是"逐场景优化"（慢、不可泛化）；本篇改走"**后训练、秒级、复用现成图像/视频编解码器**"：**重要度驱动的自适应量化**（按高斯重要度分配码率）+ **运动/重要度感知打包**（把 4DGS 属性重排成 **codec 友好的 2D map**）→ **秒级完成 40×+ 压缩**。
> **库内位置**：GS"预算/压缩线"的**部署端节点**——配额/分配三连（[[2026-10-09-SplatQ — Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection|SplatQ]] 保留端 / [[2026-10-09-CompactSplat — Spatially Adaptive Gaussian Distribution for Feedforward Scene Reconstruction|CompactSplat]] 生成端 / [[2026-10-09-H2-Ray — A Hybrid and Hierarchical Ray Tracing Method for Efficient 3D Gaussian Rendering|H2-Ray]] 执行端）之后，再加"**传输/部署端**"；"用现成编解码器"是"**不造新轮子、把问题翻译成已有格式**"的接口复用样本。

## TL;DR

**问题**：4DGS（动态 3DGS）存储开销"prohibitive"；既有压缩路线是**per-scene optimization**（逐场景优化）——对每个场景都要跑一遍优化，**慢、不泛化、不支持低延迟压缩场景**（流式/分发/端侧）。

**做法（两个组件 + 一个决定）**：

```text
决定：不造新编解码器——把 4DGS 的压缩"翻译"给现成的图像/视频编解码器

① 重要度驱动的自适应量化（importance-based adaptive quantization）
   按高斯的重要度级别分配比特——"码率跟随重要度"（refine bitrate allocation）
   且保持对既有 codec 的完全兼容
② 运动/重要度感知打包（motion- and importance-aware packing）
   利用 4DGS 的统计性质把属性（几何/外观/时间）重排成
   codec 友好的 2D map —— 让视频编解码器"愿意干活"（帧间冗余用起来）
```

**结果**：**秒级 >40× 压缩率**，保持高质量——"facilitating the practical deployment of 4DGS"。

## Core Idea

**"把新表示的问题翻译成旧工具的作业面"**：

```text
4DGS 属性本身不是 codec 友好的数据（无序、高维、混合统计）
        ↓ （打包：按运动/重要度重排）
标准 2D 图像/视频序列（codec 的主场：帧内空域 + 帧间时域冗余）
        ↓ （现成 codec：H.264/HEVC/AV1 一类的成熟工具）
压缩码流（秒级、40×+）
```

两个配套原则：**码率跟着重要度走**（与 SABC"预算配比"同构）；**压缩是一次后处理（post-training），不是逐资产优化**（"低延迟"要求反过来决定了方法形态）。

## Technical Approach（摘要级）

- **post-training 定位**：通用框架（不绑定某个 4DGS 变体/场景）——输入训练好的 4DGS，输出码流；
- **自适应量化**：高斯按重要度分级，各级不同比特预算——"importance levels"如何度量未在摘要展开（全文核）；
- **打包**：motion- and importance-aware packing——把"时间维/重要度维"的冗余暴露给 codec 的 2D 结构（对照：视频编码正是吃"帧间运动冗余"的）；
- **效率口径**：秒级（"in seconds"）、>40×、"maintaining high quality"（具体指标待全文）。

## Key Contribution

1. **后训练、通用、秒级的 4DGS 压缩框架**（区别于逐场景优化路线）；
2. **"重要度驱动码率" + "codec 兼容"** 两个约束同时满足——分配语言进入"传输端"；
3. **打包技术**：把 4DGS 的统计性质翻译成 codec 2D 结构的运动/重要度感知重排。

## Why It Works

- **翻译 > 重建**：视频编解码器是被数十亿设备验证过的压缩机器——**把 4DGS 重排到它的作业面上，借的是整个视频工业的积累**（"接口复用"的又一极致样本）；
- **重要度分级 = 非均匀预算**：数据的重要度天然不均——"均码压缩"浪费比特（同族判据：SplatQ 的配额、Confidence-Aware 的缺口采样）；
- **后训练 = 部署友好**：压缩耗时进入"秒级"区间后，**压缩本身不再是离线流水线的瓶颈**（可随分发/端侧场景使用）。

## Limitations（摘要级）

- "quality" 的具体口径（PSNR/SSIM/感知）与 4DGS 变体覆盖范围待全文核；
- 打包假设的统计性质对"高度非均匀运动"场景的适用性未知；
- 代码未见放出。

## Game Development Relevance

- **动态捕获内容的"分发"环节**：若 4DGS 一类表示进入游戏（过场、演出、背景动态资产），**存储/传输成本就是新的分档轴**——本篇给出"秒级 40×"的存在性证明；对照库内"运动层可携带"（[[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]] 的 2 MB 运动场）——**两条路（压缩动态表示 vs 静态表示+运动注入）都在压"动效的分发成本"**；
- **判据**：*"这项数据'压缩'该发生在哪一环？——逐资产优化（慢而精）还是后训练通用（快而泛）？"*——低延迟场景直接决定答案（对照 [[Animation Compression]] 的"运行时代码器 vs 装载时代码器"分工）。

## Unreal Engine Relevance

- 无直接集成；观察位：GS/4DGS 资产进引擎的"导入期压缩"环节（与 [[Gaussian Splatting]] 的部署线合流）。

## Technology Evolution

```text
GS 压缩/预算线（本库）：
  容量下限（Budgeted-GS：误差 ∝ N⁻¹ᐟ²，曲线管理）
  保留端（SplatQ：配额 + 联合选择）
  生成端（CompactSplat：内容自适应 + 稀疏度旋钮）
  执行端（H2-Ray：光追架构）
  ★ 部署端（TETRIS-4DGS：后训练 + codec 复用 + 重要度码率）——"传输/分发"补齐
  ——五端齐全：容量 / 保留 / 生成 / 执行 / 部署
```

## Relationships

### Related

- [[2026-10-09-SplatQ — Interaction-Aware Pruning of 3D Gaussian Splatting via Quadratic Selection|SplatQ]] / [[2026-10-09-CompactSplat — Spatially Adaptive Gaussian Distribution for Feedforward Scene Reconstruction|CompactSplat]] / [[2026-10-09-H2-Ray — A Hybrid and Hierarchical Ray Tracing Method for Efficient 3D Gaussian Rendering|H2-Ray]]——"分配"三端；本篇是第四端的**部署/传输**视角；
- [[2026-10-02-Budgeted-GS — Real-Time Large-Scale Gaussian Splatting via Factoring LOD|Budgeted-GS]]——"预算定在曲线哪个点"；[[Animation Compression]]——"误差界即档位"的动画侧对照（编码器分工同构）；
- [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]]——"动态内容分发的另一条路"（不存动态，存运动场）。

### Followed By

- 待观察：代码/全文数字；4DGS 部署管线的产品化（引擎导入器是否会内建此类"后训练压缩"）。

## Personal Knowledge State

- Normal（结论层 10 分钟：翻译给 codec + 重要度码率 + 秒级 40×）。

## Learning Value

- **一句可背**：**"当新表示不好压时，先把它翻译成旧工具熟悉的形状"**（4DGS → 2D map → 视频 codec）；
- **判据**：*"压缩的延迟预算落在哪？"*——逐场景优化 vs 后训练通用的分界线就是"能不能等"。

## Notes

- 周三公告组（10-14）捕获；摘要级记录；提交 2026-10-10；ICME 2026。
