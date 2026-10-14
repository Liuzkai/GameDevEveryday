---
type: paper
title: "Spatiotemporal Variance-Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination"
authors: [Christoph Schied, Anton Kaplanyan, Chris Wyman, Anjul Patney, Chakravarty R. Alla Chaitanya, John Burgess, Shiqiu Liu, Carsten Dachsbacher, Aaron Lefohn, Marco Salvi]
year: 2017
published: "2017-07-28（HPG 2017, Los Angeles；DOI 10.1145/3105762.3105770）"
venue: "High Performance Graphics (HPG) 2017（NVIDIA × KIT × McGill）"
url: "https://research.nvidia.com/publication/2017-07_spatiotemporal-variance-guided-filtering-real-time-reconstruction-path-traced"
code: "Falcor 框架 SVGFPass（NVIDIAGameWorks/Falcor）；原型代码（cwyman.org，未维护）"
project_page: ""
category: [denoising, reconstruction, path-tracing, real-time, spatiotemporal-filtering, classic]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（实时时空降噪的行业基座：后续 NVIDIA 实时降噪器产品线与几乎所有引擎侧时空滤波的参照系）"
user_level: "Normal（结论层 ≈20 分钟：三段管道 + '方差当可信度旋钮' + 三条边缘停止函数）"
status: unread
aliases: [SVGF, Spatiotemporal Variance-Guided Filtering, 时空方差引导滤波]
tags: [denoising, reconstruction, path-tracing, real-time, classic]
---

# Spatiotemporal Variance-Guided Filtering（Schied et al., HPG 2017）

> **入库 2026-10-14（Run 36）。** NVIDIA 官方 pre-print（12 页全文 pypdf 提取核读）。**库内"重建/去噪"端开域第一篇**——
> 读者 10-13 专题报告《[[Modern Real-Time Lighting Framework|现代实时光照的共同设计框架]]》的四阶段框架（采样 → 时空复用 → **重建/去噪** → 更新/失效）里，报告把 SVGF 标为"**重建分支，不是 ReSTIR 本身**"。库内此前没有这篇；与采样端同日入库的 [[Talbot — Importance Resampling for Global Illumination (2005)]] 构成**两端对照**。
> **一句话定位**：**1 spp 输入、10 毫秒输出的"第一个实用实时重建器"**——把"时域累积（补样本数）+ 时空方差估计（可信度）+ 边缘感知小波滤波（空间重建）"三段组合起来；**"方差"第一次成为滤波的控制器**：哪里方差大（不可信）多滤、哪里方差小（可信）保留——"重建误差被钉在方差估计不准的地方"。
> **库内位置**：读者框架第 3 阶段的理论锚点；也是库内所有"降噪/重建"条目的**共同祖先**（[[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]] / [[2026-10-04-SteadySplats — Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering|SteadySplats]]（其作者 Wyman 即本篇共同作者——**血统直接相连**）/ [[2026-10-08-Tabula Rasa — Monte Carlo estimation of unit-variance noise with controlled spatio-temporal correlation|Tabula Rasa]]）。

## TL;DR

**背景**：路径追踪是电影级渲染的标准答案，但实时场景里"物理正确的 Monte Carlo 全局光照达不到 30Hz 最低要求"（2017 年行业现实：约 200–300 Mrays/sec ⇒ 1080p@30Hz 只够每像素几条射线）。**"一个路径每像素（1 spp）+ 重建"被判定为未来若干年的可行上限**——于是"重建质量"成为实时路径追踪能否成立的瓶颈。

**做法（一篇讲清三段管道）**：

```text
输入：1 spp 路径追踪颜色 + 无噪声 G-buffer（depth / 世界与物体法线 / mesh ID / motion vector）
       + demodulate albedo（把纹理从光照里除出去，"只滤光照，不糊纹理"）

① 时域累积（Temporal accumulation）
   用 motion vector 回投上一帧；比较 depth / 法线 / mesh ID 判断"同一表面"
   一致 → 指数滑动平均 C'ᵢ = α·Cᵢ + (1−α)·C'ᵢ₋₁（α = 0.2）
   不一致 → 2×2 双线性逐 tap 测试；再失败试 3×3；仍失败 = disocclusion → 丢弃历史
   （目的：把"1 spp"变成"有效样本数 >> 1"）

② 时空方差估计（Variance estimation）
   时间累积颜色亮度的一阶/二阶矩 µ₁, µ₂ → σ'² = µ'₂ − µ'²₁（"可信度旋钮"）
   历史不足 4 帧（disocclusion 后）→ 7×7 双边空间回退

③ 五级 à-trous 小波滤波（Dammertz 2010 的方差引导版）
   5×5 cross-bilateral 核 h = (1/16, 1/4, 3/8, 1/4, 1/16)；迭代 5 级 → 等效 65×65 足迹
   每级把方差按滤波公式一起传播（Var 更新）→ 下一级用更"笃定"的方差
   第一级输出 ⇄ 作为下帧的时域历史（三个缓冲区互喂）

边缘停止函数（决定"哪里不许混"）：
   w(p,q) = w_z · w_n · w_l
   w_z：depth（局部线性平面模型，容忍斜视深度梯度）σ_z = 1
   w_n：法线（cos^σn）σ_n = 128
   w_l：亮度（用"局部标准差归一化"的自适应阈值；驱动用的方差先过 3×3 高斯，只用于此项）
       σ_l = 4——三组常数"全场景通用、不暴露给用户"
```

**结果**：1920×1080 **10 ms（±15%）**；1280×720 平均 4.4 ms（4.1–5.8）；约 **200 MPixels/s**、随分辨率线性缩放；对照先前工作**时域稳定性约 10×**、SSIM 匹配参考图**好 5–47%**；论文自称"**第一个只用每像素一条路径的实用实时重建滤波**"。

## Problem

- **实时的"两条腿"分工**：光栅化（主可见性）+ 光追（次级/阴影射线）；**Monte Carlo 噪声必须被重建掉**才能看；
- **既有滤波的三大不满足**（原文梳理）：① **频域/剪切滤波**（Egan/Mehta 等）需要 4–16 spp 起步、且高频区"缩 footprint"反而涨方差；② **回归/学习滤波**（Rousselle/Moon/Kalantari）在高样本率才好用、**低样本率对离群值敏感**、且不实时；③ **先前的小波滤波（EAW, Dammertz 2010）**用"全局调整的颜色边缘停止"——**丢细节 + 光晕**（后文对照图中 Sponza 青色 inset 即此病灶）；
- **1 spp 的特殊难点**（原文点名）：一个像素只有一个样本，"区分噪声来源"极难——"高频纹理的空间采样噪声与光传输/可见性噪声混在一起"；
- **时域复用的坑**："Color-based temporal filtering 用在噪声输入上会引入伪影"——直接用 TAA 式颜色混合不行，必须**几何感知**（depth/法线/mesh ID 一致性）地累积。

## Historical Context

```text
1990  Lee & Redner：非线性滤波进 CG（离群点剔除）
1994-99 能量保持滤波（Rushmeier-Ward）/ 各向异性扩散（McCool）——"先例存疑"的探索期
1998 ± 双边滤波（Tomasi-Manduchi）→ 交叉/联合双边（Eisemann-Durand 2004）→ "边缘停止函数"语言定型
2005  Durand et al.：光传输的频域分析——"最优滤波足迹"理论
2009-15 剪切/轴对齐滤波（Egan / Mehta / Yan / Hasselgren）——实时化尝试（需 4–16 spp）
2010  Dammertz et al.：edge-avoiding à-trous 小波（**SVGF 的直接结构前身**）
2014  TAA（Karis）：时域复用在游戏里全面普及（"amortized supersampling"）
2015  Kalantari et al. LBF：神经网络控制双边权重（**神经降噪的前声**，本文对照对象之一）
2017  ★ 本篇：时域累积 × 方差估计 × à-trous 三合一 —— 1 spp 实时重建成立
```

## Previous Work（原文视角，两族对照）

- **离线降噪**（tens–hundreds spp，秒/分钟级）：非局部均值、回归、SVD 等——"高样本率可用，低样本率不稳、且不实时"；
- **交互式 MC 降噪**：频域分析族（Durand 2005 谱系）+ 交叉双边族（"用几何/颜色差异做边缘停止"）——SVGF 说自己站**第二族**，并指出：**"把几何信息放进边缘停止函数"是交叉双边鲁棒性的关键**，而"逐像素加权边缘停止分量"（用方差）是下一步的缺口——**本篇即补此缺**；
- **时域滤波族**：ray histogram fusion（Delbracio）、PCA（Meyer-Anderson）等要么需要预计算帧集合、要么离线——**实时的"只回投上一帧"是硬约束**（Nehab 2007 / Yang 2009 / Karis TAA 的血统）。

## Core Idea

**"把'该不该滤、滤多狠'变成一个每像素可估计的量——方差"**：

```text
传统滤波：全局参数（σ_c、σ_s 固定）→ 要么糊细节、要么留噪声
SVGF：    每像素方差（可信度）→ 驱动滤波强度：
          · 方差大（噪声多）→ 多滤（跨更大 footprint 借样本）
          · 方差小（稳定区）→ 少滤（保住细节）
          且方差本身随滤波迭代下降 → 权重自适应"越滤越保守"（防过糊）
```

配套两条设计公理（原文反复出现）：

- **"时间给样本数"**：1 spp 不够 → 用几何一致的时间累积把有效样本数拉起来；**方差才有得估计**（一帧一个样本算不出方差——必须时间累积矩）；
- **"demodulate 解耦"**：先除 albedo 再滤（"把重建聚焦在光传输上，而不是去噪纹理细节"）——"该滤的"与"该保的"在入口处分家。

## Technical Approach（关键机制细读）

- **时域一致性测试**：depth（回投位置比较）、object-space normal、mesh ID 三项；一致性判据沿用"片段合并启发式"（Jouppi-Chang 1999 / Kerzner-Salvi 2014 一族的经验相似度量）；**2×2 双线性逐 tap 测试、3×3 兜底（挽救树叶类细几何）、全失败判 disocclusion 弃历史**；
- **方差作为"不可靠性的价格"**：原文一句可直接背诵——"**Noise increases variance, but variance can occur without noise**"（空间方差估计只是噪声的**不完美代理**）；因此 ①历史<4 帧时用 7×7 双边空间估计"过渡"；②**用方差归一化亮度差**：w_l = exp(−|l(p)−l(q)| / (σ_l·√(Var)+ε))——"自动适配所有尺度"（亮区大差可容忍、暗区小差即停）；
- **滤波与方差的同步传播**：每级小波把 Var 按 h²w² 公式更新 → "**滤波也滤'可信度'本身**，下一级的判断基于更新后的方差"（"the reliability of the reconstruction grows with each iteration"）；
- **三个"为什么"**：
  - 为什么 5×5 核 + 5 级：à-trous（"带孔"）的非零元恒定 ⇒ 每级成本 O(1)、足迹指数增长（1→65）——**"固定成本换指数足迹"的小波红利**；
  - 为什么分离 direct/indirect：各自局部平滑性不同（阴影边缘 vs 漫反射低频）——分开滤、共享 G-buffer 成本（"看似翻倍、实则大量共享"）；
  - 为什么 α=0.2：原文实测"平衡时域稳定性与滞后"的最佳点（"trading temporal stability for lag"）；
- **性能剖面**：成本"几乎与场景属性无关"；唯一波动源 = disocclusion 后的空间方差回退（**帧耗时随 disocclusion 数量增加**，满屏 disocclusion 时最坏可达 2×）；线性于分辨率。

## Key Contribution

1. **第一个实用的 1 spp 实时重建器**（原文结论句："the first practical real-time reconstruction filter operating on just a single path per pixel"）；
2. **时域方差估计 + 空间回退**：解决"1 spp 无法估计方差"的死结——**方差第一次成为可实时计算的量**；
3. **方差引导的 à-trous 滤波**：滤波强度逐像素自适应 + 方差在迭代中传播（"重建误差被局部化在方差不准处"）；
4. **场景无关的几何边缘停止函数**：depth 用"局部线性平面模型"替代全局深度阈值（应对景观类大尺度变化）；三条 σ 常数全场景通用、**不暴露给用户**（"scene-agnostic"的工程声明）；
5. **质量数字**：10 ms@1080p（TITAN X Pascal）/ 10× 时域稳定性 / SSIM 5–47%（匹配参考）/ 对照表（RMSE、SSIM across Sponza、Glossy Sponza、Red Room、San Miguel、Classroom）。

## Why It Works

- **"误差局部化"原则**：方差估计不准的地方 = 重建误差最高的地方——**把不确定性显式建模，就不会全局地糊**；
- **时间维度的双重作用**：既是"样本数乘器"（EMA 累积）又是"统计量来源"（矩）；**第二条正是它比 TAA 多出来的东西**——"不是把颜色混起来，而是把'颜色的可信度'算出来"；
- **几何一致性是时域复用的安全带**：color-based TAA 在噪声输入上翻车（会糊错面），depth/normal/mesh ID 三件套确保"只混同一表面"；
- **滤波迭代的"自收敛"**：方差随级数下降 → w_l 变紧 → 越往后越保守——**过糊被结构性抑制**。

## Limitations（原文自陈，六条，全列出）

1. **色度过糊（chrominance）**：只估亮度方差 ⇒ 彩色反射被糊（Glossy Sponza 彩色 inset）；修法（逐通道方差）代价更高；
2. **运动脱影（detached shadows）**：近光快速运动物体会产生"脱开的影子"——时域累积不知道照明已变；
3. **极暗区不稳定**：路径贡献极少的暗部（间接光角落）——"样本密度趋零时方差估计越来越不可靠"；
4. **镜面反射被糊**：采样噪声被滤掉、但反射本身也糊了（"noise 与 signal 在同一尺度"的新样本）；
5. **运动下的特征过糊**：回投本身引入模糊 + 非零方差 → 额外空间偏差（相机运动时硬阴影被糊）；
6. **不兼容随机主射线效果**：需要**无噪声 G-buffer** ⇒ 随机透明、随机景深、随机运动模糊全不支持；高度走样的亚像素几何也会让边缘停止找不到样本。

## Game Development Relevance

- **你所用的所有"降噪"的谱系源头**：实时 GI/反射/阴影的降噪器（引擎内置、NVIDIA 产品线、乃至神经重建）在结构上都是"**时域累积 + 可信度启发 + 边缘感知空间滤波**"三件套的后代；库内既有条目 [[2026-10-04-SteadySplats — Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering|SteadySplats]]（共同作者 Wyman）与 [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]] 是本谱系在 GS 域的直系应用；
- **判据（对特效/管线）**：*"我的降噪/滤波哪里'一刀切'？能不能把'该不该滤'变成每像素可估计的量？"*——方差只是其中一种"可信度"；同族还有历史长度、置信度（RTXDI Confidence）、动作残差（CAST 的 reliability-gate）——**"可信度驱动"是今日跨域复现的母题**（见 Daily）；
- **对"分档"的直接映射**：SVGF 的降级路径（历史不足 → 空间回退）是**实时系统的标准降级形态**——"高质量来源不可用时退回廉价来源、恢复后再切回"；
- **成本口径**："10 ms@1080p、与场景复杂度几乎无关、随 disocclusion 波动"——降噪是**关于分辨率线性、关于场景近常数**的成本项（对照 1978 灯光"∝ 灯数"、粒子"∝ 屏占比"——各自不同的成本函数族）。

## Unreal Engine Relevance

- **不通过源码绑定**：读者报告已定口径——"这条解释对应 SVGF 研究的时空重建思想……它**不表示 UE 使用 NVIDIA denoiser**"；本篇入库支撑的是**思想来源层**（"时域累积 + 边缘停止 + 方差引导"的模式在 Lumen 的屏幕追踪降噪、各引擎的 TSR/TAA 系管线中同构存在）；
- **实践接口**：UE 项目内可观察的最直接后代形态 = ①Lumen/RTGI 的降噪行为（读者报告 §9 排查表的"哪一层"方法同样适用）；②GS/自定义后端的去噪（若做自研管线，Falcor 的 SVGFPass 是公开参考实现）；
- 与 [[Temporal Stability and Artistic Intent]] 的接口：**"时域稳定"从 TAA 的图像层问题，在本篇扩展为"光照重建"问题**——概念页已互链。

## Technology Evolution

```text
重建/去噪端谱系（本篇为分水岭）：
  1990s 非线性滤波萌芽（离群点剔除 / 能量保持 / 各向异性扩散）
  1998-2004 双边 → 交叉双边（"边缘停止函数"语言）
  2005 Durand：频域分析（最优足迹理论）→ 2009-15 剪切/轴对齐滤波（需 4–16 spp）
  2010 Dammertz à-trous 小波（结构前身）
  2014 TAA 普及（时域复用的游戏化）
  2015 LBF：学习式权重（神经降噪前声）
  ★ 2017 SVGF：1 spp + 时域 + 方差引导 → 实时（10 ms）
    ├─ 2018+ A-SVGF（自适应采样版）等衍生
    ├─ NVIDIA 实时降噪器产品线（时空降噪的工程化延续）——对照库内 E-Day"主动放弃 Ray Reconstruction"的适用边界样本
    └─ 2020s 神经重建全面接管（DLSS RR / 各类神经降噪 / GS 去噪）
       库内节点：SteadySplats（2026，GS"生干净"）/ Stochastic GS Denoising（2026）/ Tabula Rasa（2026，"噪声合同"）
  ——"重建"正在从"滤波"走向"学习"，但三件套骨架（时域/可信度/边缘）仍是共同底盘
```

## Relationships

### Based On

- **Dammertz et al. 2010**（edge-avoiding à-trous 小波——结构直接前身，"我们扩展它"）；
- **时域复用三源**：Nehab 2007（反向重投影缓存）、Yang 2009（摊销超采样）、Karis 2014（TAA）；
- **交叉双边/边缘停止**（Tomasi-Manduchi 1998；Eisemann-Durand 2004）；**Durand 2005** 频域分析的问题设定。

### Extends

- **EAW（Dammertz 2010）**：全局参数 → **逐像素方差引导的滤波强度**；"3×3 高斯预滤方差以驱动亮度项"是其工程细节补丁（原文图 5）。

### Contrasts

- **LBF（Kalantari 2015）**："学习式权重"线条——SVGF 用**可解释统计量**（方差）达到更快的速度；两者对照 = 日后"解析 vs 学习"在重建域的第一次同框；
- **Mara et al. 2017（HPG 同期）**：按完整路径去噪（不分 direct/indirect）——"独立同期开发"，原文列为未来对比工作。

### Related

- [[Talbot — Importance Resampling for Global Illumination (2005)]]——**同日入库的另一端**（采样端：把噪声的"来源"变便宜）；本篇管"噪声的后果"；
- [[Temporal Stability and Artistic Intent]]——概念层：时域稳定问题从图像层扩展到光照重建；
- [[2026-10-04-SteadySplats — Resampling of Low-Variance Gaussians for High-Fidelity Stochastic Rendering|SteadySplats]]——"共同作者 Wyman"的直接血统；`库内 GS 排序线（第 6 节点）`的"生干净"与 SVGF"滤干净"互补；
- [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]]——GS 域去噪的"取消式"变体；[[2026-10-08-Tabula Rasa — Monte Carlo estimation of unit-variance noise with controlled spatio-temporal correlation|Tabula Rasa]]——"噪声 = 接口契约"的 2026 样本；
- [[Modern Real-Time Lighting Framework]]——读者报告的"重建/去噪"阶段指认。

### Followed By

- A-SVGF 等演变；NVIDIA 实时降噪产品线；**神经重建**（DLSS Ray Reconstruction 系、"从滤波到学习"的过渡）；GS 时代的两条续线（SteadySplats / Stochastic GS Denoising）。

## Personal Knowledge State

- Normal（结论层 ≈20 分钟：三段管道 + "方差 = 可信度旋钮" + 三条边缘停止 + 降级路径；数值细节（σ 常数、核权重）可查不背）；
- 与读者的接口：读者报告第 2.5 节（重建）+ §9 排查表（"噪声 vs 偏差""缓存没变 vs 历史滞后"）在本篇都有原型——本篇补的是**原文级的机制与口径**（"10 ms、1 spp、10× 稳定"的出处）。

## Learning Value

- **三句可背**：① "**方差是可信度旋钮**——滤波强度不是全局常数，是每像素估计出来的"；② "**时间给样本数，方差给分寸**"（前半句是 TAA 的遗产，后半句是 SVGF 的新增）；③ "**历史断了就退回空间估计**——降级路径是实时系统的标配"；
- **判据**：*"哪里'一刀切'？把它换成每像素可估计的可信度。"*——今日与 CAST 的 reliability-gate、读者报告的"重要度 vs 可信度两条轴"合流；
- **对排查的映射**：读者报告 §9 表的"拖影 / 缓慢翻动的噪斑 / 光照变化滞后"三类现象，在本篇分别对应"一致性测试失败/方差误判""时间累积与方差的统计极限""缓存/历史更新预算"——**本库自此有了"排查语言"的半个出处**。

## Visualization

![[重建端_SVGF 2017 时空方差引导滤波图解]]

## Notes

- **核证记录**：HPG 2017（Los Angeles，7-28~30；DOI 10.1145/3105762.3105770）；NVIDIA Research 页 + 官方 pre-print（svgf_preprint.pdf，12 页全文提取核读）；作者单位（NVIDIA / KIT / McGill / U Montreal）；性能数字出自 §5.3–5.5（TITAN X Pascal）；限制六条出自 §6；Falcor 参考实现（NVIDIAGameWorks/Falcor `SVGFPass`）；
- **同期旁证**：Mara et al.《An Efficient Denoising Algorithm for Global Illumination》为 HPG 2017 同期独立工作（原文致谢/对照提及）；
- 同日入库采样端经典 [[Talbot — Importance Resampling for Global Illumination (2005)]]；相关概念页 [[Temporal Stability and Artistic Intent]] 已同步互链。
