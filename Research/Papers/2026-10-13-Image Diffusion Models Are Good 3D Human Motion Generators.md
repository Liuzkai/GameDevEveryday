---
type: paper
title: "Image Diffusion Models Are Good 3D Human Motion Generators"
authors: [Siyuan Yang, Chong Wang, Hao Cheng, Jun Liu, Bihan Wen, Alex Kot, Hedvig Kjellstrom]
year: 2026
published: 2026-10-13
venue: "arXiv 2610.14254"
url: "https://arxiv.org/abs/2610.14254"
code: ""
project_page: ""
category: [animation, motion-generation, diffusion, text-to-motion]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: Research
user_level: Normal
status: read
---

# Image Diffusion Models Are Good 3D Human Motion Generators

> NTU Singapore（Yang/Wen/Kot）× KTH（Kjellström）。2026-10-14 cs.CV 周三批次；昨日记名，本日升级入库。

## TL;DR

把 3D 人体动作序列**排列成图像**（拓扑感知的 joint–time 骨架图 + 互补根运动流），让图像扩散架构（ImageNet 预训练 ResNet 编码器 + SiT 生成器）直接处理文本→动作——**不需要为每种骨架定制模型**：单一 autoencoder + 单一 SiT 同时覆盖 HumanML3D 与 KIT-ML 两种异构骨架定义（**不做重定向、不做关节 padding**），两个基准上都是 SOTA，且共享训练让两个基准**同时**涨分。

## Problem

Text-to-motion 模型通常绑定数据集专属的骨骼表示：HumanML3D（22 关节）与 KIT-ML 关节数、索引约定都不同——向量表示里这些差异既改变维度又改变"向量第 i 位对应哪个关节"，跨数据集共享模型必须显式处理。此前做法：要么各训各的，要么把数据重定向到统一骨架（引入 retarget 误差）。问题：**能不能不动数据、只动表示，让图像生成架构直接吃下异构骨架？**

## Core Idea

**"翻译给旧工具"的动画版**（对照 TETRIS-4DGS 把 4DGS 翻译成视频 codec）：把动作翻译成图像，让图像领域的全部积累直接复用。

1. **拓扑感知 joint–time 图像**：根中心局部关节轨迹按解剖学重排成 256×256 骨架图——关节序 = 左腿→左臂→躯干+头→右臂→右腿（镜像布局），肢体内按运动链排序（左肢远端→近端、右肢近端→远端）；关节轴获得粗粒度解剖学组织，不同关节数的数据集共享同一图像分辨率——**空间维度与原生关节数解耦**。
2. **互补根运动流**：朝向变化、航向对齐平面位移、根高度这些**全局动力学**不共享关节级空间组织，单独走一条潜流。
3. **根条件化解码**：图像解码器以 root latent 为条件（conditioner 特征注入第一个上采样级之前）——局部关节与全局运动耦合，但潜表示分离。
4. **文本条件 SiT 联合生成双潜流**：流匹配速度预测，双流独立噪声初始化、t=0→1 联合演化——生成全程互相感知。
5. **解码空间监督**：生成时从预测干净潜变量解码 3D 关节位置，把约束压回运动空间。
6. **跨骨架共享**：单一 motion autoencoder + 单一 SiT 覆盖两数据集（带数据集条件）；**各数据集保留自己的骨架定义**，解码时用各自专属的 image→motion 恢复算子 G_d。

## Key Numbers

- HumanML3D / KIT-ML 数据集专属模型均 SOTA（MARDM 协议，对比 T2M-GPT/MMM 等 token 系与 MDM/MLD 等）；
- 共享训练后**两个基准同时再涨**（HumanML3D Top-1 R-Precision 0.517→0.533，FID 0.145→0.106，来自 ImageNet 初始化；数据集条件加入后 FID 0.278→0.093）；
- 消融：去掉根条件化解码 FID 0.106→0.116——"全局运动上下文帮局部解码"可量化。

## Why It Works

- **图像不是任意容器**：拓扑感知排序让"图像相邻"≈"解剖相邻"，卷积的局部性假设与骨骼的运动学局部性对齐——表示自带先验（对照 Neuroll 的"表示即先验"）。
- **成熟架构的迁移红利**：ImageNet 预训练权重直接可用（消融证明 +0.038 FID 收益）——表示兼容的收益不只是"能跑"，是"能继承预训练"。
- **分离但耦合的双流**：全局（root）/局部（articulation）分工与 MegaLights"复用的样本不同就不该硬合并"同哲学——**该分开的表示分开，该耦合的地方用条件注入耦合**。

## Limitations

- 拓扑排序仍是手工设计（人形专属；非人形骨架需重排）；"镜像布局"对对称动物的迁移未验证；
- 图像分辨率固定 256×256——超长序列/高频细节的分辨率压力；
- 单一 autoencoder 共享两数据集后，数据集条件是必要补丁（0.278 vs 0.093）——表示共享还没完全消化分布差异；
- 无实时性数据（研究级采样步数），游戏内运行成本未测。

## Game Development Relevance

- **动作生成域新阵营锚点**："图像扩散做动作"与库内 token 系（MotionBricks）/MM 检索系（Motion Matching）三足格局——三条路线的表示互不相容，这篇证明了图像系的天花板；
- **"跨骨架共享 ≠ 重定向"是 [[Motion Retargeting]] 的第二种答案**：CAST 在捕捉侧跨拓扑、本篇在生成侧跨拓扑——**"不动数据、动表示"**可能减少管线里的 retarget 环节；
- 对角色管线：单一模型服务多种骨架资产（人形变体、不同 DCC 导出）是内容生产端的实际诉求。

## Unreal Engine Relevance

- 直接映射：离线生成 →导入 Anim Sequence/Motion Matching 数据库；与 [[Motion Matching]] 的关系是"生成数据 vs 检索数据"的供给关系；
- 无引擎内实时方案（采样成本），近期不构成 runtime 技术。

## Technology Evolution

```text
2022  MDM（扩散直接在动作空间）→ MLD（潜空间）
2023  DiT / SiT（Transformer 扩散架构成熟）
2024  MotionBricks（token 化，库内 S 级时序锚点）
2025  MARDM（扩散+掩码自回归）
★2026 本篇（图像兼容表示 + 双流 SiT + 跨骨架共享；"翻译给旧工具"）
```

## Relationships

### Based On

- DiT/SiT（图像扩散 Transformer 架构）+ ImageNet 预训练 ResNet
- 骨架图像表示的识别系前作（SkeleMotion 2019 等——识别→生成的迁移）

### Related

- [[2026-10-10-CAST — Real-Time Motion Capture for Any Skeleton Topology|CAST]] — 跨骨架的捕捉侧样本（同周合流："跨拓扑"在捕捉/生成两端同时出现）
- [[2026-10-08-PriMoRetarget — Humanoid Motion Transfer Through Prioritized Motion Retargeting|PriMoRetarget]] / [[Motion Retargeting]] — retarget 的替代答案："不动数据、动表示"
- [[2026-10-09-Sonic-AMG — Streaming Human Motion Generation from Spatial Audio|Sonic-AMG]] — 同为 2026 动作生成新模态样本（音频模态 vs 文本模态）
- [[2026-10-10-TETRIS-4DGS — Efficient Post-Training Compression of 4D Gaussian Splatting|TETRIS-4DGS]] — "翻译给旧工具"母题同日系（4DGS→codec / 动作→图像）

### Followed By

- 待观察：图像兼容表示是否成为跨骨架生成的标准底座（Watchlist：代码放出）

## Personal Knowledge State

- **user_level: Normal（结论层）**——"翻译给旧工具""双流分工""表示自带先验"三条抽象直接可读；SiT 流匹配细节属 [[Motion Generation]]（Hard）桥上的一格，可后置。

## Learning Value

- 三条可迁移抽象：① **表示兼容性 = 预训练继承权**（换表示的收益不止能跑）；② **全局/局部双流分离 + 条件注入耦合**；③ **跨异构数据共享的另一条路：统一表示而非统一数据**。

## Visualization

> 本篇首图为示意图（表示构造），核心信息可用文字+表格承载；按 §25 自查三问：MD 能表达 → 不做图，仅记 fig 2 结构（关节排序：左腿→左臂→躯干头→右臂→右腿，镜像布局；256×256；root 独立潜流）。待代码/项目页放出后再决定是否补原图。

## Notes

- 昨日（Run #36）记名时的判断"与 CAST 同日的跨骨架第二样本"今日升级：细读后价值高于初判——表示设计与消融完整度是 A- 级。
- 与 [[Fluid Simulation]] 无关；与 [[ReSTIR]] 无关——本篇属动画线，勿混 10-15 光照主线。
