---
type: paper
title: "A Median Cut Algorithm for Light Probe Sampling"
authors: ["Paul Debevec"]
year: 2006
published: 2006
venue: "ACM SIGGRAPH 2006 Courses (SIGGRAPH '06), Article 6（一页 sketch/course 短文）"
url: "https://vgl.ict.usc.edu/Research/MedianCut/MedianCutSampling.pdf"
code: "论文未给（LightGen HDRShop 插件 [Cohen and Debevec 2001] 为相关前身工具）"
project_page: "http://www.debevec.org/MedianCut/"
category: ["light-probes", "light-extraction", "image-based-lighting", "sampling", "median-cut"]
importance: B+
historical_importance: 4
game_relevance: 4
production_readiness: "Production（思路被工业界反复复用：pbrt 课程作业标配实现、DNEG iblManager 引其为"程序化 HDRI 分析"源头 [Cieri & Schwank 2019]、多个引擎 HDRI→灯阵工具的事实起点）"
user_level: Easy
status: unread
---

# A Median Cut Algorithm for Light Probe Sampling

## TL;DR

**"HDRI 环境图 → 一簇显式点光源"最经典的一页纸方案。** 把 light probe（经纬展开图）递归切成 $2^n$ 个**等能量**区域，每个区域放一盏灯：位置取区域能量质心、颜色取区域内像素和。64 盏灯已接近昂贵的 Monte Carlo 参考解，256 盏几乎无法区分。

算法四步（原文逐字结构）：

1. 整张 probe 图作为唯一区域加入区域列表；
2. 对列表中每个区域，沿最长维切成能量均等的两半；
3. 迭代次数不足 $n$ 则回到第 2 步；
4. 每个区域在质心放一盏灯，颜色 = 区域内像素值之和。

对「灯光迁移」专题：这是 **① 目标（环境光旧数据）→ ② 显式灯参数** 的最简解析路线——**零优化、零迭代、确定性输出**。与 [[Sloan 2008 — Stupid Spherical Harmonics Tricks\|Sloan 2008]]（SH 频域抽灯）和 [[Gkaravelis 2016 — Inverse Lighting Design using a Coverage Optimization Strategy\|Gkaravelis 2016]]（空间域组合优化选灯）构成"隐式表示 → 显式灯"的三条不同出发点：图像域等能量划分 / 频域峰值检测 / 空间域覆盖聚类。

![[Research/Files/Pictures/Debevec 2006 — 图1 Grace Cathedral 探针 64 等能量区域划分.png]]
**读图**：Grace Cathedral 探针被切成 64 个等能量区域（绿框），小圆圈为 64 盏灯的位置（各区域能量质心）。**亮窗区域被切得很碎、暗部区域很大**——灯自动向高能量区密集，这正是"等能量划分"想要的性质；且每盏灯能量大致相等，重要性天然均衡。（Debevec 2006，图 1）

![[Research/Files/Pictures/Debevec 2006 — 图2 16-64-256 盏灯近似对比.png]]
**读图**：同一漫反射小场景分别用 16 / 64 / 256 盏抽出的灯渲染（a–c），与每像素 4096 条随机光线的 Monte Carlo 参考（d）对比。**16 盏的阴影已显成块、64 盏接近参考、256 盏几乎无法区分**——给"灯数 vs 质量"提供了一个具体的量级锚点。（Debevec 2006，图 2）

## Problem

基于图像的照明（IBL）用一张环境图表示入射光；但很多管线（早期硬件、逐灯阴影、想获得"可编辑的灯"）需要**有限数量的显式点光源**。问题：怎么选灯的位置与颜色，使有限灯阵对环境光的逼近质量最高？先验判断（原文）：灯位跟随入射照度分布时逼近质量最好——此前的分层采样方法（[Cohen and Debevec 2001; Kollig and Keller 2003; Agarwal et al. 2003; Ostromoukhov et al. 2004]）都在追求这一性质。

## Historical Context

2006 年 IBL 已是影视标配（Debevec 本人 1998 起的工作线），但全局光照计算昂贵，"把环境图近似成灯阵"是渲染加速与硬件适配的常见需求。本文借 Paul Heckbert 1982 年的 median-cut 颜色量化算法（把颜色空间切成等像素数区域）——**把"颜色计数"换成"光能量"，就是本文全部灵感**。后续有 Viriyothai & Debevec 2009 的方差最小化改进（见 Relationships）。

## Previous Work

- Heckbert 1982, median-cut 颜色量化（SIGGRAPH '82）——算法骨架来源；
- Crow 1984, summed-area table（SAT）——区域能量 $O(1)$ 查询的加速结构；
- Kollig & Keller 2003（EGSR）、Agarwal et al. 2003（SIGGRAPH）、Ostromoukhov et al. 2004（SIGGRAPH）——此前的分层/结构化重要性采样路线；
- Cohen & Debevec 2001, LightGen HDRShop 插件——同源前身。

## Core Idea

**等能量划分 ⇒ 灯位自动跟随照度分布。** 亮的地方区域小、灯密；暗的地方区域大、灯稀。每盏灯能量近似相等，因此均匀随机选一盏灯采样就是合理的重要性采样。整个划分是**确定性贪心**，不需要优化器，也不依赖渲染——目标域就是 probe 图像本身。

## Technical Approach

- **能量度量**：先在单色图上做划分，单色图按 ITU-R BT.709 加权 $Y = 0.2125R + 0.7154G + 0.0721B$；灯的颜色仍从原 RGB 图的区域像素和计算（划分用亮度、取值用彩色）。
- **经纬图畸变补偿**：lat-long 展开过采样两极，像素先乘 $\cos\varphi$（$\varphi$ 为倾角）；判断区域"最长维"时，区域宽度也要按区域中心倾角的 $\cos\varphi$ 加权。
- **加速**：区域能量查询用 summed-area table（Crow 1984）。
- **输出**：$2^n$ 盏灯（$n$ 为迭代次数），位置 = 区域质心（或中心），颜色 = 区域像素和。

## Key Contribution

1. 把 median-cut 从颜色量化迁移到光探针采样，给出**极小实现成本**的 HDRI→灯阵算法；
2. 明确"等能量区域 ⇒ 等能量灯"这一良好条件性质（well-conditioned static sampling）；
3. 给出 lat-long 畸变补偿与 SAT 加速两个实现要点——**几乎所有工程坑都在这一页里点到了**。

## Why It Works

等能量划分让每盏灯"责任相同"，没有一盏灯独大；灯位跟随能量密度，等价于对入射照度做了一次分层采样，但**无随机性、无方差**——代价是有偏（bias），偏差随灯数反比下降（原文结论节原话："noise-free renderings at the expense of bias inversely proportional to the number of light sources"）。

## Limitations

- 原文自述：**有偏**（偏差 ∝ 1/灯数）；动画光照环境下的稳定性、一般 BRDF（非漫反射）适配留作未来工作；
- 灯数只能是 $2^n$（递归二分），不能任意指定；
- 只输出点光源（方向 + 颜色），**没有灯距、没有尺寸**——影视面光路线（LightCraft、iblManager、Shlyaev）正是补这个缺口；
- 漫反射场景验证，高光材质下"灯位=反射峰"的假设未检验。

## Game Development Relevance

- 旧项目迁移时若旧数据是 **HDRI 探针 / cubemap**，这篇是**零成本第一版**：几十行代码就能出一簇点光/方向光，作为后续连续优化（Lipp 式伴随法）的初值；
- "灯数 vs 质量"量级锚点：64 盏 ≈ 接近参考、256 盏 ≈ 无法区分（漫反射场景、Grace 探针条件下）；
- 引擎内"把天空盒拆成主方向光 + 若干点光"的功能，思想源头多可追溯到这里。

## Unreal Engine Relevance

UE 侧未内置 median-cut，但"HDRI → Directional Light + Sky Light"的拆分实践（如 TrueHDRI 插件，见收集页清单）与该思路同族；若在 UE 工具链里做探针迁移，median-cut 可作为 Python/C++ 几十行的预处理步骤嵌入。

## Technology Evolution

前身：LightGen（2001，HDRShop 插件，按亮度阈值分区）。直系后续：**Viriyothai & Debevec 2009, Variance Minimization Light Probe Sampling**（SIGGRAPH 2009 Sketches，把"等能量贪心"换成方差最小化目标，记名未建页）。工业侧精神续作：DNEG iblManager（[[Cieri 2019 — Streamlining IBL Workflows with Computer Vision and USD]]）明言恢复其"程序化 HDRI 分析"策略，但输出从点光升级为带纹理的面光。

## Relationships

- [[Sloan 2008 — Stupid Spherical Harmonics Tricks]]：同目标（隐式→显式），走 SH 频域路线；median-cut 走图像域；
- [[Cieri 2019 — Streamlining IBL Workflows with Computer Vision and USD]]：引用本文并明言继承其程序化分析思路；
- [[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization]]：median-cut 的输出可当 Lipp 式连续拟合的初始灯阵（比均匀网格更贴合能量分布）；
- [[Global Illumination]]、[[Physically Based Rendering]]：IBL 与灯表示的上游概念。

## Personal Knowledge State

已亲读全文（一页，含全部公式与实现细节）；数字均来自原文。**未核**：LightGen 插件的具体行为（仅引文层面）；Viriyothai 2009 正文未读。

## Learning Value

极低门槛进入"隐式光照 → 显式灯"问题：一页纸、无优化、可立刻实现。适合作为迁移工具的 baseline 与初值生成器。

## Visualization

论文原图两张已提取（见 TL;DR 嵌入）。HTML 图示不必做——算法四步用文字即足够清楚。

## Notes

- 原文作者主页归档：`www.debevec.org/MedianCut/`；PDF 直链可用（2026-11-04 实测）。
- 实现练习标准材料：台大渲染课程作业（pbrt2 MedianCutEnvironmentLight）即按本文实现，可作交叉验证参考。
