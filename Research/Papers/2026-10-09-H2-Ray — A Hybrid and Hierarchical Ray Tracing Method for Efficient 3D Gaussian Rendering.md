---
type: paper
title: "A Hybrid and Hierarchical Ray Tracing Method for Efficient 3D Gaussian Rendering"
authors: [Tiancheng Zhao, Sixu Li, Yuzhou Chen, Hengyu Ding, Zhiming Pan, Chaojian Li]
year: 2026
published: "2026-10-09（arXiv v1, 2610.13498；cs.AR + cs.GR）"
venue: "arXiv Preprint（架构提案；周期精确 RT Core 模型 + 实测 CUDA 时序；单位待核——提交后 PDF 尚未放出；未公告先见：API 通道 2026-10-12 捕获）"
url: "https://arxiv.org/abs/2610.13498"
code: ""
project_page: ""
category: [gaussian-splatting, ray-tracing, hardware-architecture, load-balancing, hybrid-rendering]
importance: B+
historical_importance: 0
game_relevance: 3
production_readiness: "Research（硬件架构提案；RTX A5000 上对照 RaySplats 的模拟+实测混合评估）"
user_level: "Normal（结论层：GS 光追的三个难点 + 三招对策；架构细节可跳过）"
status: unread
aliases: [H2-Ray, 混合层级光追, GS 光追架构]
tags: [gaussian-splatting, ray-tracing, hardware, load-balancing]
---

# A Hybrid and Hierarchical Ray Tracing Method for Efficient 3D Gaussian Rendering（Zhao et al. 2026）

> **入库 2026-10-12（Run 34）。** 经 **API"未公告先见"通道**捕获（cs.GR 窗口 5 条之一）。
> **一句话定位**：**GS 光追侧的第一个硬件架构节点**——光栅化是 3DGS 的效率基石，光追能解锁二次光照/畸变相机/景深，但带来三个新难题（**负载不均 / 遍历冗余 / 异构求交**）；H2-Ray 用三招对应（**LB-SBVH / RCPR / UIE**），相对已有 GS 光追方案 RaySplats 几何平均 3.77×、最高 9.26×。
> **库内位置**：GS"渲染执行"侧开线——此前库内 GS 渲染线全是**光栅化**视角（排序线六节点、overdraw、tile 分箱）；本篇给出"**另一条执行路线（光追）的困难清单与硬件对策**"，且"Gaussian + triangle 混合求交"与 [[Magpie — Real-Time World Renderer for Interactive Games|Magpie]] 的混合渲染器互为呼应。

## TL;DR

**背景（原文）**：3DGS 系统靠**光栅化**拿效率；近期工作开始探索**光追**——为了二次光照（secondary lighting）、高度畸变相机、景深。但"光追版 3DGS"比光栅化多了三个结构性问题：

```text
三个难点（光追侧特有）：
① 负载不均：海量高斯的空间分布高度不均匀 → 全局负载失衡
② 遍历冗余：相邻像素的射线路径高度相似 → 重复遍历开销大
③ 异构求交：高斯 + 三角形两种 primitives 混合 → 求交负载与现有 GPU
   架构不匹配（"not well matched to existing GPU architectures"）

三招对策（H2-Ray）：
① LB-SBVH（Load-Balanced Spatial BVH）：
   均匀网格划分 + density-aware 表示 → 把"不平衡的全局负载"
   转成"平衡的局部遍历单元"
② RCPR（Ray-Cluster Path Reuse）：
   让"代理射线"与邻居射线共享遍历状态 → 摊销重复遍历
③ UIE（Unified Intersection Engine）：
   模式复用数据通路（mode-multiplexed datapath）支持高斯/三角形混合求交，
   两类 primitive 共享 40% 的浮点单元

结果：8 个场景，对照 RaySplats：
  几何平均 3.77× / 最高 9.26×（NVIDIA RTX A5000；周期精确 RT Core 模型 + 实测 CUDA 时序）
```

## Problem

- **动机侧**：光栅化 GS 做不了/做不好"二次光照、畸变相机、DOF"——光追是解法方向；但**光追 GS 的成本结构完全是新的**；
- **三个结构性困难**
  1. **不均匀性**：Gaussian 分布天然非均匀（物体处密集、空旷处稀疏）——BVH 遍历的负载在 warp/线程间失衡；
  2. **空间相干性利用不足**：光栅化利用"像素邻域"的规律性；光追里相邻像素发射的射线 traversal path 高度相似，**逐射线遍历 = 系统性重复劳动**；
  3. **混合场景**：[[Magpie — Real-Time World Renderer for Interactive Games|Magpie]] 式混合渲染（三角形 + 高斯）在求交层是两种完全不同的数学（ray-triangle vs ray-Gaussian）——现有 RT Core 只擅长前者。

## Core Idea

**"把不均匀转成平衡、把重复变成共享、把异构装进同一数据通路"**——三招各治一症：

```text
症状                          对策                  机制
① 全局负载不均   →   LB-SBVH     均匀网格 + 密度感知表示，
                                  全局不平衡 → 局部平衡单元
② 相邻射线重复   →   RCPR        代理射线共享 traversal state
                                  （复用"这条路怎么走"的中间结果）
③ 两种 primitive →   UIE         模式复用数据通路，
                                  40% FPU 在高斯/三角形间共享
```

本库视角：这是"**复用/摊销**"家族在硬件层的样本（对照：DDGI 的每帧光线摊销、[[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]] 的重用、"取消式优化"的反面——**不去掉，改为共享**）。

## Technical Approach（按摘要可核部分）

- **LB-SBVH**：场景 → 均匀网格；density-aware 表示；"convert an imbalanced global workload into balanced local traversal units"；
- **RCPR**：ray cluster 内代理射线（proxy ray）代表邻居共享遍历状态——摊销冗余遍历；
- **UIE**：mode-multiplexed datapath；高斯/三角形求交 **共享 40% FPU**；
- **评估方法**：cycle-accurate model of the modified RT Core（周期精确的 RT Core 模型）+ measured CUDA timings（实测 CUDA 时序）——**模拟 + 实测混合**（架构提案论文的标准姿态）；
- **基线**：RaySplats（已有 GS 光追方案）；平台 RTX A5000；8 个场景。

## Key Contribution

- **GS 光追的"困难清单"**（对理解 GS 渲染的完整图景最有用）：负载不均 / 遍历冗余 / 异构求交——**三条都源于"高斯的空间统计特性 ≠ 三角形的空间统计特性"**；
- **"Gaussian + 三角形"的混合求交硬件**：与混合渲染器（Magpie 类）的需求闭环——如果混合表示成为常态，"求交单位"必须杂食；
- 相对 RaySplats 的 3.77×/9.26×（模拟+实测混合口径）。

## Why It Works

- 均匀网格 + 密度感知 = **用空间划分的均匀性对冲数据分布的不均匀性**（把统计学问题变成工程问题）；
- 射线相干性（neighboring rays similar）是光追算法里的既有宝藏——RCPR 把它显式变现；
- 两种求交共享 FPU：在硅面积约束下换吞吐（**共享 40% 是实现成本与灵活性的折中**）。

## Limitations（以现有信息可判的部分）

- 架构提案（research），非产品化硬件；与 NVIDIA 官方 RT Core 路线的关系待观察；
- 3.77× 是对 RaySplats 的口径（RaySplats 本身不是成熟最优基线）；
- 训练侧/表示侧不变——本方案只动"执行"（渲染管线层）；
- 移动端场景未讨论。

## Game Development Relevance

- **GS 渲染的"执行路线分叉"**：光栅化（当前主流，排序/overdraw 问题系）vs 光追（本篇困难清单系）——对理解"为什么 GS 的实时化这么难"提供第二条完整视角；
- **二次光照/DOF 的 GS 侧入口**：与 UE 光追管线（Lumen/MegaLights 生态）的对照——GS 内容要进高端管线，"光追能力"迟早要补；
- 混合求交 = "杂食渲染器"的硬件化前兆（对照 [[Magpie — Real-Time World Renderer for Interactive Games|Magpie]]"三角形+高斯同屏"）。

## Unreal Engine Relevance

- 无直接集成；间接意义：UE 若在 GS/Nanite 混合场景走光追加速（RTX Mega Geometry 生态），"高斯求交单元"可能成为未来硬件话题——记入 Watchlist。

## Technology Evolution

```text
GS 渲染执行线（本库账本，首次成对）：
  光栅化轨（2023→2026）：
    3DGS → 排序问题（六节点"取消排序"线）→ tile 分箱 → 预算/LOD → 随机化
  光追轨（2026-10 开线 ★）：
    RaySplats 等探索（二次光照/畸变相机/DOF）
    → H2-Ray（本篇）：负载均衡 + 遍历复用 + 混合求交的架构三件套
  ——两轨的共同命题：高斯不是三角形，"按三角形的硬件直觉"全部要重写
```

## Relationships

### Based On

- 光追 GS 的近期探索（RaySplats 等——具体谱系待 PDF）；RT Core 架构惯例。

### Extends

- **RT Core**：从"ray-triangle 专用"向"高斯/混合求交"扩展的架构提案；
- **负载均衡技术**（均匀网格 + density-aware）从通用 GPU 计算迁移到 GS 光追。

### Related

- [[Magpie — Real-Time World Renderer for Interactive Games|Magpie]]——混合表示（三角形+高斯）的渲染器侧样本；本篇是其"求交硬件"侧；
- GS 排序线（[[Gaussian Splatting]] 概念笔记）：光栅化轨的六节点 vs 本篇光追轨——**两条执行路线的困难清单对照**。

### Followed By

- 待观察：单位核对；与 GPU 厂商路线的对照；DOF/二次光照的实用收益。

## Personal Knowledge State

- Normal（结论层 15 分钟：三个难点 + 三招名字级别的记忆；架构细节跳过）。

## Learning Value

- **"不均匀分布 → 均匀划分 + 密度表示"**的通用手法（H2-Ray 的 LB-SBVH）——与"分配三连"的其余两篇共构今日主题；
- **反问句**：*"我的瓶颈是'做得慢'还是'重复做'？"*——光追 GS 里"相邻射线重复遍历"的问题与 RCPR 的"代理共享"提醒：**先找系统性重复，再谈优化**（与"取消式优化"家族互补：找不到重复就先问"能不能不存在"，找到重复就问"能不能共享"）。

## Visualization

![[分配三连_SplatQ、CompactSplat、H2-Ray 图解]]

## Notes

- 单位未能核对（PDF 未放出）；作者列表中 Chaojian Li 为（香港中文大学（深圳））学者——**待 PDF 放出后核实**，本笔记暂不写入单位；
- **未公告先见记录**：API 通道 2026-10-12 捕获（提交 10-09，listing 不可见）。
