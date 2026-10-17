---
type: paper
title: "Compression of Uniformly Distributed Unit Quaternions"
authors: [Carsten Faber, Bastian Kuth, Rinaldo Schuster, Daniel Pauls, Quirin Meyer]
year: 2026
published: 2026-10-15
venue: "arXiv 2610.22052 (cs.GR)"
url: "https://arxiv.org/abs/2610.22052"
code: ""
project_page: ""
category: [animation, compression, geometry, gaussian-splatting, engine]
importance: B+
historical_importance: 0
game_relevance: 4
production_readiness: Prototype
user_level: Normal
status: unread
aliases: [LinfCub]
---

# 2026-10-15-LinfCub — Compression of Uniformly Distributed Unit Quaternions

## TL;DR

单位四元数压缩的**系统研究 + 新 SOTA 参数化**：先给常用参数化（丢最大分量/投影立方体/投影八面体/单位球映射）补上**解析最大误差**对比（此前只有经验数据），再提出 **ℓ∞cub**——八面体投影后套一个**单参数立方 warp 函数**（参数闭式优化，最小化最大量化误差），配合**混合基数打包**（chart 索引 + 量化参数打成单个整数，几乎不浪费码字状态）。结果：同位宽下最大误差最小；解压 O(1)、无查找表、不比现有方法贵。实测：骨骼动画 **48 bit/关节关键帧**（99%+ 关键帧误差 ≤1 个 8-bit 层级）、法线贴图切线帧 **32 bit**、3DGS 旋转 **30 bit**（24 bit 的 ℓ∞cub ≈ 30 bit Exp 参数化的质量）。

## Problem

- 四元数无处不在：骨骼动画每关节每关键帧一个、网格切线帧（法线贴图）、3DGS 百万级各向异性高斯的朝向——内存与带宽成为首要约束；
- 现有方法各占一摊（anim 库丢最大分量 / glTF 立方体投影 / 新格式八面体 / 球映射），但**只有经验对比，没有误差理论**——"哪个在给定 bit 下误差最小"没有答案。

## Historical Context

```text
1985  Shoemake 四元数进动画（Compose/插值正统）
1984-98  丢最大分量 + 第三分量重建（最老牌 tricks）
2005  ISO 3DXML 立方体投影入标准
2017  球映射（Rey 2017）
2024-25  八面体投影进 GS 格式（Niantic/PlayCanvas 系）
2026  ★本文：解析误差理论 + ℓ∞cub + 混合基数打包
```

## Previous Work

- 丢最大分量（three.js/anim 库系）：简单，但最大误差与解码分支多；
- 立方体投影（3DXML/glTF KHR_texture_transform 系）：状态利用不足（角落浪费）；
- 八面体投影（2025 GS 格式系）：本文的起点——ℓ∞cub = 八面体 + warp；
- 球映射（Rey 2017/RPWM26）：误差特性中等；
- 3D 单位向量压缩已有大量对比研究（Cigolle 2014 等），四元数侧是空白——本文补齐。

## Core Idea

**两件事**：

1. **ℓ∞cub 参数化**：丢最大分量 → 3D 坐标（八面体 chart）→ 用三次多项式 warp 拉伸中间区域（warp 参数 ω 闭式优化，使量化格点在球面上的**最大**测地误差最小）→ 均匀量化。名字由来：把误差的 L∞ 范数压到最小。
2. **混合基数打包**：chart 索引（6 选 1，部分 bit）+ 三个量化坐标 → 拼成**单个整数**；各参数获得**相同精度**，且任意位宽的码字状态利用率接近 100%（独立位域会在边界浪费状态）。

## Technical Approach

- 理论：对每个参数化推导**一阶闭式最大误差**（量化格点在球面的最大角偏差），再对全部单位四元数给出全局界——排序：ℓ∞cub < 八面体 < 球映射 < 立方体 < 丢最大分量（最大误差口径）；
- 解压：多项式求值 + 归一化，O(1) 无查表，指令数 ≤ 现有方法；
- 应用端（图 1）：骨骼动画关键帧（可丢符号，q 与 −q 同旋转）；切线帧 32 bit；3DGS 每个 Gaussian 128 bit 总预算中旋转只给 30 bit（24/16 bit 档也测）。

## Key Contribution

1. **误差理论**：四元数压缩第一次有解析最大误差界（此前只有 Cigolle 2014 式的 3D 向量研究）；
2. **ℓ∞cub**：同位宽最大误差最小 + 存储效率最好 + 解压不贵——三个目标同时占优；
3. **混合基数打包**：通用技巧，任何"索引+多分量"编码都能用。

## Why It Works

- 均匀分布四元数的 3D 投影密度不均匀（角/边密、面心疏）——warp 函数把密度拉平，让量化误差的**最坏情况**（而非平均）最小化；
- 最大误差口径对游戏特别相关：动画数据里最坏关节误差决定可见 pop，平均误差会掩盖它。

## Limitations

- 压缩端要逐资产跑优化与打包（资产管线成本，一次性）；warp 参数对"均匀分布"假设成立——**非均匀分布**（如某关节活动范围受限）需要按数据重优化，论文未展开；
- 误差界是**一阶**的（位深极低时高阶项可见）；
- 无代码放出（AMD 参与者，格式落地可能跟 GS 格式标准走）；16 bit 档图像质量已可见退化（FLIP 图 5）。

## Game Development Relevance

- **直接进 SABC 预算表**：动画数据的内存/带宽账 = 关节数 × 关键帧率 × 每帧位宽——本文给"每帧位宽"这一维一个**有理论依据的档位**（48/32/24 bit 梯度），与贴图尺寸预算（S≤2048/A≤1024/...）同一性质；
- **3DGS 资产线**：GS 进游戏管线的每个 Gaussian 128 bit 预算里旋转占 30 bit——旋转压缩直接放大可加载的 Gaussian 总数；
- **法线贴图切线帧**：32 bit/顶点切线帧 vs 传统存法线+切线向量，顶点流水线的常量级瘦身。

## Unreal Engine Relevance

- UE 动画压缩体系（Anim Compress 系：raw/linear key 减除/量化编码）已有自己的位宽旋钮——ℓ∞cub 属"更优的量化参数化"，可对接 UAnimBoneCompressionSettings 的自定义编码器思路；
- UE5 GS 插件生态（如 XRIE/第三方）导入端位宽选择可参考本文 30 bit 结论。

## Technology Evolution

```text
四元数压缩 = 各家经验 trick 并立（丢分量/立方体/八面体/球映射）
  → ★本文（2026）：解析误差界排出座次 + ℓ∞cub 统一最优 + 单整数打包
  → （可期）动画/GS 资产格式跟进（AMD 系格式、glTF 扩展）
```

## Relationships

### Based On

- 八面体投影（2025 GS 格式系）——ℓ∞cub 的几何起点
- 3D 向量压缩对比研究（Cigolle 2014 系）——方法论模板

### Related

- [[2026-10-10-TETRIS-4DGS — Efficient Post-Training Compression of 4D Gaussian Splatting|TETRIS-4DGS]] — GS 压缩的另一维（4DGS 整体压缩 vs 本文只抠旋转位宽）
- [[Real-Time VFX Performance Budgeting]] — "位宽=预算维度"的资产侧对应物

### Applies

- 骨骼动画关键帧存储 / 切线帧编码 / 3DGS 旋转属性

## Personal Knowledge State

- **Normal（结论层）**：参数化座次、warp 思想、混合基数打包——工程味重，结论可直接搬。

## Learning Value

- "均匀量化的最大误差 = 密度不均匀问题"这个框架可迁移：任何"球面/流形属性压缩"（法线、轴、颜色空间）都适用同一套路（投影→warp→打包）。

## Visualization

![[Research/Files/Pictures/Faber 2026 — 图1 四元数压缩三类应用.png]]

**读图**：ℓ∞cub 的三个落点——左上骨骼动画（龙模型）：上排未压缩参考、下排 48 bit/旋转关键帧，
PSNR 38.3–42.5 dB、99%+ 关键帧误差 ≤1 个 8-bit 层级；右上法线贴图（狮子）：切线帧 32 bit
（对角线分割参考/压缩），PSNR 63.2 dB、max 1 层级；下方 3DGS（自行车场景）：旋转 30 bit，
FLIP 误差图几乎全黄（≈无可见差异）。（Figure 1，Faber et al. 2026）

## Notes

- 提交于 2026-10-15（美东周四），属 10-16 周五公告批次；Coburg 大学 + AMD 德国；**下轮核查代码放出**。
- 关键数字备查：48 bit 动画关键帧 / 32 bit 切线帧 / 30 bit GS 旋转（24 bit ≈ Exp 30 bit 质量 45.9 dB）；解压 O(1) 无查表。
