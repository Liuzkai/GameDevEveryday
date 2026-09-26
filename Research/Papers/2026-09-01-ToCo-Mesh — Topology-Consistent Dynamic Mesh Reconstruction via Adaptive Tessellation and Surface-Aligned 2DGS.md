---
type: paper
title: "ToCo-Mesh: Topology-Consistent Dynamic Mesh Reconstruction via Adaptive Tessellation and Surface-Aligned 2DGS"
authors: [Chuanjin Fan, Wenjie Chang, Aibing Li, Bingzhou Wang, Wenfei Yang, Tianzhu Zhang]
year: 2026
published: "2026-09-01（v1 提交；2026-09-25 公告）"
venue: "ACM SIGGRAPH Asia 2026（Kuala Lumpur, December 2026；to appear）"
url: "https://arxiv.org/abs/2609.29529"
code: ""
project_page: "https://fan-treasure.github.io/ToCo_Mesh_page/"
category: [reconstruction, gaussian-splatting, character, capture]
importance: A-
historical_importance: 2
game_relevance: 4
production_readiness: Research
user_level: Normal
status: unread
aliases: [ToCo-Mesh]
tags: [gaussian-splatting, reconstruction, topology, animation, capture, siggraph-asia]
---

# ToCo-Mesh: Topology-Consistent Dynamic Mesh Reconstruction via Adaptive Tessellation and Surface-Aligned 2DGS

## TL;DR

**动态网格重建一直有个二选一**：逐帧提取细节最好但**顶点对应断掉**（网格闪烁）vs 模板变形**对应一致但分辨率锁死**（细节丢）。ToCo-Mesh 把两者同时拿到——**固定拓扑 + 自适应细分**，并用一层"面片上的高斯"补外观：

1. **双网格表示**：一个**规范模板网格**（canonical template）通过重心坐标绑定时变**粗引导网格**（guide mesh，来自 Deformable-GS 或 SMPL）；引导网格保持固定以"条件化"形变，模板网格按**几何与渲染误差**做 **split-and-merge** 自适应加密；
2. **Surface-Aligned 2DGS**：把压扁的高斯**吸附在网格面片上**，用它们渲染出的法线反过来**引导几何微调**——"几何给高斯提供锚点，高斯给几何提供法线"；
3. **结果**：DG-Mesh 上 CD 0.659（全场最低）、3D-TS 0.05（最稳），**网格只要 13K 顶点**（对比 4D-GS 729K / DG-Mesh 82K），训练 75 min、6.6 GB、**72 FPS**（RTX 3090）。全文声称是**首个在保持严格拓扑一致的同时支持自适应网格细化**的框架。

> **对管线的一句话**：**"能喂进动画系统"比单帧精度更值钱**——拓扑一致性是"好看"与"能用"之间的那道门。重建结果喂入未见过的 pose 参数即可动画（SMPL 驱动）。

## Problem

从多视角时序图像重建**动态网格**（4D reconstruction），工业界要的是"**可动画、可编辑、可下游使用**"的资产，而现有两条路都不合格：

| 路线 | 优点 | 致命伤 |
|---|---|---|
| 逐帧提取（frame-by-frame） | 单帧细节好 | **破坏顶点对应** → 网格抖动/闪烁（flickering meshes），无法做动画与绑定 |
| 模板变形（template-based） | 拓扑一致、可动画 | **优化中无法改分辨率** → 局部细节能力被初始模板锁死 |

**核心矛盾**：细节保真（fine-scale shape）⟷ 拓扑稳定（topological stability）。

## Core Idea

### 1. 双网格绑定（template–guide binding）

- **粗引导网格**（time-varying coarse guide meshes）：逐帧形变的骨架，**固定不细化**——它的职责是"条件化"运动；
- **规范模板网格**（canonical template）：通过**重心坐标参数化**紧绑到引导网格上，随时间被"带着走"；
- **分工**：guide 管运动（不动分辨率），template 管分辨率（不动拓扑连接关系）——**两者通过绑定解耦**。

### 2. 误差驱动 split-and-merge（自适应镶嵌）

- 对模板网格做**顶点分裂与合并**（借鉴 mesh generation 的顶点分裂思想）；
- **判据 = 几何误差 + 渲染误差**，超过阈值 → split（补细节），低于阈值 → merge（去冗余）；
- 全程保持**严格拓扑一致**（vertex correspondence 始终成立）——分裂/合并改变的是分辨率，不是对应关系。

### 3. Surface-Aligned 2DGS（面片吸附的高斯）

- 把**压扁的高斯锚定到网格面片上**（anchoring flattened Gaussians to mesh faces）；
- 高斯的渲染法线质量高 → **反哺几何微调**（inverse geometric fine-tuning），消除表面噪点；
- 三方约束：网格提供"结构化中间表示"，高斯提供"高频外观与法线"，**光度和法线一致性损失**把二者绑在一起。

## Technical Approach

- **两阶段优化**（Stage I 拓扑自适应几何 → Stage II Surface-Aligned 2DGS 微调），三个基于网格姿态的神经模块（MPE-Net 姿态特征 / RSGD-Net 驱动 2DGS 相对运动 / NDMR-Net 几何精修）——**丢弃显式时间戳输入**，以引导网格几何状态为条件 → 可泛化到新 pose；
- PyTorch，单卡 RTX 3090；数据集：**DG-Mesh / D-NeRF / HuMMan**（HuMMan-Recon 取官方测试集前 8 个序列，10 同步相机）。

**DG-Mesh 定量对比（Table 2）**：

| 方法 | CD↓ | EMD↓ | PSNR↑ | 3D-TS↓（时序平滑） | 顶点数 | 训练 |
|---|---|---|---|---|---|---|
| 4D-GS | 1.396 | 0.209 | 32.48 | 1.47 | 729K | 11 min |
| SC-GS | 1.521 | 0.169 | 38.17 | 1.53 | 709K | 58 min |
| DG-Mesh | 0.697 | 0.130 | 30.67 | 0.91 | 82K | 127 min |
| MaGS | 1.719 | 0.108 | 39.69 | 0.08 | 0.5K | 66 min |
| **ToCo-Mesh** | **0.659** | **0.107** | 38.75 | **0.05** | **13K** | 75 min |
| GT | — | — | — | — | 72K | — |

**效率对比（Table 9，D-NeRF）**：ToCo-Mesh 75 min / 6.6 GB / **72 FPS**（4D-GS 11/1.5/82；SC-GS 58/3.6/78；DG-Mesh 127/9.5/45；D-2DGS 186/4.1/71）。

**HuMMan（真拍人体）**：几何精度优于/持平同类，训练量级对比——Anim-NeRF >12h、MMLPHuman >24h、SplattingAvatar 38 min（ToCo-Mesh 与 DG-Mesh 同级 ~75 min 量级）。

## Key Contribution

1. **模板-引导绑定机制**：时间拓扑一致性 + 自适应网格优化的同时成立；
2. **误差驱动的分裂/合并策略**：按几何与渲染误差自适应调整顶点密度；
3. **Surface-Aligned 2DGS 模块**：2DGS 与网格的双向紧耦合（高斯锚定几何、法线反哺几何）。

## Why It Works

1. **把"对应关系"与"分辨率"拆成两个自由度**：guide 冻结 → 对应稳；template 误差驱动 → 分辨率活。**一个表示同时被两个互相矛盾的指标（一致性/细节）约束时，拆开比折中更好**；
2. **让每个表示做它擅长的事**：网格管拓扑/运动（结构化、可动画），高斯管外观/高频（渲染质量）——**"结构锚定"母题的第 N 例**（Web：[[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction|Mira-Scene]] 的"有界对应"、[[2026-09-22-PartLLM — A Unified Multimodal Foundation for 3D Part Segmentation|PartLLM]] 的"意图粒度"、[[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]] 的"面/边拓扑"）；
3. **误差阈值 = 什么时候加密**："让分辨率跟着误差走"与 LOD/自适应细分同源。

## Limitations

- **超参数负担重**（正则项平衡、拓扑阈值）：过强正则 → 抹平真实高频细节；过松 → 拓扑噪声（原文 E.2 自述）；
- 训练 75 min 级（远非秒级）；依赖粗引导网格的质量；
- 适用域为**人物/刚性-近刚性动态**（SMPL 类先验）；开放场景的通用动态重建未覆盖。

## Game Development Relevance

**4/5 —— 对你有三层**（且都属于"管线语言"而非"研究细节"）：

1. **"拓扑一致性 = 可动画的货币"**：录制/重建资产进入游戏管线的第一道门不是"像不像"，而是"**能不能绑定、能不能喂动画**"。ToCo-Mesh 的卖点正是把这道门拆掉：输出直接喂 pose 参数即可动画；
2. **"13K 顶点 vs 729K 高斯"**：一个**可动画的紧凑网格**可以在几何精度上超过几万个高斯——**表示的大小与精度不是一回事**（对"资产预算"的含义：重建类资产的成本要按表示类型分开记）；
3. **误差驱动细分**：与你的分档/自适应体系同构——**"何时升档"可以用误差阈值量化**，而不是拍脑袋。

## Unreal Engine Relevance

- 若进入生产：对应"**4D 捕获 → 可动画资产**"的离线管线（比 UE 现成的体积视频/神经渲染路径更贴近现有骨骼动画工具链）；
- 与 [[Gaussian Splatting]] 的引擎侧落点同族（2DGS 家族），但**本条路线的产物是网格**——对 UE 的意义是"资产而不是渲染技术"；
- 关联观察：[[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards|HairCS]] 是"同一条哲学在毛发资产上的版本"（跨表示转换/升档），可并列参考。

## Technology Evolution

```text
动态场景重建：
NeRF 系（D-NeRF 等）—— 隐式、无显式表面
        ↓
3DGS 系（4D-GS / Deformable-GS / SC-GS）—— 快，但网格需事后提取（噪声/伪影）
        ↓
Mesh-GS 混搭系（DG-Mesh / D-2DGS / MaGS）—— 网格与高斯联合，但拓扑恒定或对应断裂
        ↓
★ ToCo-Mesh 2026 —— 双网格绑定 + 误差驱动细分 + 2DGS 吸附（**固定拓扑与自适应分辨率首次兼得**）
```

## Relationships

### Based On
- 2DGS / SuGaR 类"面片吸附高斯"路线；Deformable-GS / SMPL 提供粗引导网格

### Contrasts
- **逐帧提取**（对应断裂、闪烁）与**纯模板变形**（分辨率锁死）——本文同时否定了两者的"妥协版"

### Related
- [[Gaussian Splatting]] —— 2DGS 家族；本库"GS 之上的新研究"又一例
- [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]] —— 同为 GS 工程化（一个管渲染顺序，一个管与网格耦合）
- [[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction|Mira-Scene]] / [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies|ProxyBuild]] / [[2026-09-22-PartLLM — A Unified Multimodal Foundation for 3D Part Segmentation|PartLLM]] —— "结构锚定"母题
- [[Hair Rendering]] 的资产线（HairCS）—— 跨表示转换的姊妹问题

## Personal Knowledge State

- **user_level: Normal**。判断依据：**"拓扑"与"动画"对你显然在 Easy 区**（角色管线日常）；**新的是"误差驱动细分"与"网格-高斯绑定"这套重建侧的语言**——只需取三条抽象（见下），不需要读消融。

### 三条可直接拿走的抽象

1. **"固定拓扑 + 自适应分辨率"是把一对矛盾拆成两个自由度** —— 遇到"既要 A 又要 B"的问题，先问"能不能把 A、B 拆到两个可分别调节的表示层上"；
2. **"可动画性 > 单帧精度"**（管线视角）—— 重建/生成类资产进管线的门槛是拓扑与绑定，不是截图；
3. **"误差阈值决定密度"** —— 与 LOD/自适应细分同源，可作为"何时升档"的量化判据。

## Learning Value

- 本库**首个"4D 捕获 → 可动画网格"**的完整样本（此前 GS 线偏渲染、动画线偏生成，缺"捕获侧"）；
- 与 W39 的"结构锚定"母题、GS 工程化线、资产升档线（HairCS）三处交汇。

## Notes

- 原文已核对：2026-09-25 公告（v1 提交 2026-09-01，**属"早提交、晚公告"条目**——API `submittedDate` 窗口查不到，只能靠 recent 页公告分组捕获）；arXiv HTML 全文 + 项目页 + API 元数据三方核对；表 2/表 9 数字逐项比对（含 13K 顶点、75 min、72 FPS、6.6 GB）；
- 机构：**中国科学技术大学（USTC，合肥）**；venue：**ACM SIGGRAPH Asia 2026**（页面标注 Conference: KLCC, Kuala Lumpur, December 2026）；
- 项目页：`fan-treasure.github.io/ToCo_Mesh_page/`；入库时机：2026-09-26（Run #18）。
