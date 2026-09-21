---
type: paper
title: "Casting Curved Shadows on Curved Surfaces"
authors: [Lance Williams]
year: 1978
published: 1978-08
venue: "SIGGRAPH '78, Computer Graphics 12(3), 270–274"
url: "https://cseweb.ucsd.edu/~ravir/274/15/papers/p270-williams.pdf"
code: ""
project_page: ""
doi: "10.1145/800248.807402"
category: [rendering, shadow, visibility, image-space, classic]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: Easy
status: read
---

# Casting Curved Shadows on Curved Surfaces

> 入库于 2026-09-21。原文 PDF 已下载并**逐节核对**（5 页，含 6 张图版与完整参考文献）。
> **它是"动态灯光预算"这个维度的原始出处** —— 你写的 `动态灯光 ≤3 / ≤2 / ≤1 / 0`，
> 背后那条"每盏灯大概要花掉一倍场景渲染"的成本法则，**第一次被写下来就是在这篇里，1978 年**。
> **它的价值不在机制（机制你早已熟悉），而在成本模型。**

## TL;DR

1978 年，Lance Williams 在 NYIT 用 **Z-buffer**（Catmull 1974）做了一个当时所有人都知道原理、却谁也做不实用的东西：**阴影**。

**一句话价值**：

> **把"物体是否被光源看见"变成"在光源视角下比一次深度"** —— 两次渲染即可，且代价可预测。

**成本法则（原文摘要原句，这是本文最该记住的一句）**：

> *"The cost of determining the shadows associated with each light source is **roughly twice the cost of rendering the scene without shadows**, plus a fixed transformation overhead which depends on the image resolution."*

**⚠️ 一个必须澄清的历史归属（避免常见误传）**：

| 常见说法 | 实际情况（本次逐条核对原文后） |
|---|---|
| "Williams 1978 发明了 shadow map" | **Williams 自己把它叫 "depth map"**（且该词引自 Levine et al. 1973）；**"shadow map" 是后来才通行的叫法** |
| "阴影需要两次渲染是 Williams 提出的" | **原文明确归给 Crow 1977**：*"it has long been suggested that two passes of a visible surface algorithm is sufficient to compute shadowing [5]"*（[5] = Crow, *Shadow Algorithms for Computer Graphics*, SIGGRAPH '77）|
| Williams 的真正贡献 | **让"两次渲染"真的可用** —— 用 Z-buffer 建立了两个视角之间**可线性变换的数据关联**，从而第一次把阴影扩展到**曲面**；并系统处理了随之而来的量化与自阴影问题 |

## Problem

阴影能显著提升场景的可理解性（原文开篇即引电子显微与航拍的先例）——它让物体的**形状与相对位置**变得可读，为动画增加实感。

但在 1978 年：

- **已有的阴影算法只对平面多边形构成**的场景成立（"To date, algorithms for the determination of shadows have been restricted to scenes constructed of planar polygons"）；
- 而当时最新的渲染能力（Catmull 的 Z-buffer）已经能画出**双三次曲面片**的着色图像 —— **能力与算法脱节**；
- 更根本的障碍是：**两次渲染好说，但"把第一次渲染的数据和第二次对起来"极难**。原文结论段说得很清楚：

> *"Although it has long been suggested that two passes of a visible surface algorithm is sufficient to compute shadowing, **relating the data provided by the two passes is very difficult for many algorithms**."*

## Historical Context

```text
1972  Newell/Newell/Sancha —— 隐藏面问题的一种解法
1974  Catmull —— Z-buffer（本文的直接依赖：线性增长成本 + 深度信息副产品）
        ↓
1977  Crow —— 《Shadow Algorithms for Computer Graphics》：提出"两遍可见性即可算阴影"
                （但限定平面多边形；且点光需分扇区，见下）
★ 1978  Williams —— 本文：用 Z-buffer 把"两遍"变成可实用的曲面阴影
        ↓
2006  CSM（Cascaded Shadow Maps）—— 解决"分辨率与距离"的冲突
2008+ PCF / PCSS / VSM —— 解决"硬边与噪声"
        ↓
2021  Virtual Shadow Maps（UE5）—— 逐页虚拟化，把"分辨率预算"从全局改成按需
        ↓
2024–2026  MegaLights（UE 5.5 → 5.8 Production）—— 把"每盏灯一遍"改成"大量灯共享结构化采样"
        ↓
        ★ 这一整条线的成本直觉，全部来自本文开头那句话
```

## Core Idea

**关键洞察**：**"阴影"和"透视投影"是同一类东西**。

> *"The shadows cast by a point source of light onto a flat surface represent, like a perspective transformation, **a projection of the scene onto a plane**. ... **A scene rendered with shadows contains two views in one image.**"*

而如果两个视角**都是 Z-buffer 视角**，那么它们之间存在一个**线性变换**，可以把观察者视角里的每个点映射到光源视角里 —— **于是"是否在阴影中"就变成了一次比较**。

> **这条"两视图合一"的思路在 1978 年是全新的**，而它恰好与 2026 年的 DLSS 5 同构：**DLSS 5 也是把引擎传出的 G-buffer（几何/深度/运动矢量）当"第二视图"来重建画面**。**48 年后，"引擎多给一张视图，神经层负责合成"成为同一种思路。**

## Technical Approach（逐节核对）

### 算法（原文两步，直接引用）

> *"1. A view of the scene is constructed from the point of view of the light source. **Only the Z values and not the shading values need be computed and stored.**
> 2. A view of the scene is then constructed from the point of view of the observer's eye. **A linear transformation exists which maps X,Y,Z points in the observer's view into X,Y,Z coordinates in the light source view.** As each point is generated in the observer's view, it is transformed into the computed view in the light source space and **tested for visibility to the light source** before computing its shading value. If the point is not visible to the light source, it is in shadow and is shaded accordingly."*

**注意第 1 步里的"只存 Z、不算着色"** —— 这正是今天 shadow map pass 能做到"约 2× 而非 2× 以上"的原因，原文在结论段明确点出：

> *"The rendering cost is only **"roughly"** twice the cost of rendering the scene normally, **since the light source view requires no shading computation**. Depending on the complexity of the shading rules applied, this may represent a substantial savings."*

> **【对今天的直接意义】**：这条解释了一个常见的预算误判 —— **"shadow pass 不贵"是相对于材质复杂度而言的**。材质越贵（Substrate / 多层 / 大量贴图采样），shadow pass 的相对占比就越低；**反之，在一个材质极简的场景里，阴影就会逼近 50% 的渲染开销**。这直接决定你在不同画质档下"该砍灯还是该砍材质"。

### 修正版（后处理式）与它的代价

原文诚实地区分了两个版本：**"正确版"**（逐点生成时就变换并判定）与**"后处理版"**（先完整渲染观察者视角，再把可见点变换到光源空间）。后处理版的**两处错误**（原文自述）：

1. **高光被错误着色** —— *"it incorrectly shades the highlights in the scene... **hilights should not appear in shadowed areas at all**"*；
2. **量化误差更严重** —— 可见点的 Z 已经先被量化到 Z-buffer 分辨率（本文实例是 **16 bit**）才做变换。

**换来的好处**：变换开销**与场景复杂度无关**，只与图像分辨率有关（*"The expense is thus dependent only on the resolution of the image. Like most point-by-point operations, expense increases with **the square of the resolution**."*）。

> **⚠️ 这两处错误的现代对应物**：后处理版"高光出现在阴影里"今天依然会发生（把阴影因子乘到最终颜色上而非在着色前剔除高光）；"Z 先量化再变换"就是今天 shadow bias 与深度精度问题的祖先。

### 自阴影（shadow acne）—— 1978 年的完整判据链

这是本文技术含量最高的一段，**今天所有 shadow bias / PCF 的调参经验都能在这里找到出处**：

1. **问题的根源**：把观察者视角上的一点变换到光源空间，**它理论上应该正好落在自己所在的曲面上**；但受机器精度与 Z 量化影响，**它会落在曲面上方或下方**；
2. **修法一：偏置（bias）** —— *"we **subtract a bias** from the Z value of the point after it has been transformed"*，并且"这个 bias 实际上是**并入那个通用线性变换**里的"（→ 今天 bias 是投影矩阵的一部分，同源）；
3. **修法一带来的副作用** —— bias 会**移动阴影线**（peter-panning 的祖先）；
4. **光滑曲面的临界情况** —— *"As a surface curves smoothly away from the light... **it may switch back and forth as the quantizing error beats with the sampling grid, producing a vivid moire**"*（**这就是今天的 shadow acne / 摩尔纹**，原文用 Fig. 3 刻意调小 bias 来展示它）；
5. **量化误差最大的两个位置** —— ① 光源视角下的物体边缘；② 观察者视角下的物体边缘。原文的解释很关键：**这两处本来就已被走样（非带限图像被离散采样）**；
6. **一个反直觉的结论** —— **"对 Z 做带限预处理没用"**：边缘处的 Z 会是"边缘深度与远裁剪面深度的混合"，**对场景而言毫无意义**。所以走样是**局部**的，只需在边缘外接受它；
7. **修法二：双线性插值光源 Z** —— 不用最近邻比较，而是在精确 X,Y 处**插值**光源图 Z 值 → *"reduces shadow noise in the form of isolated pixels"*；
8. **修法三：把噪声当量化问题处理（抖动 dither）** —— *"Addition of random noise **in the range of a single quantum** breaks up this correlation... and **whitens the spectrum of the error**"*（Fig. 4 用了 ±0.5 正态随机数）；
9. **修法四：边缘去量化滤波 + 低通滤波** → **这就是 PCF 的原始形态**，而且原文给出了它的**物理意义**：

> *"**low pass filtering of the shadows before they are applied to the image subjectively approximates the soft penumbra cast by real light sources.**"*
> —— **"对阴影做低通滤波，主观上近似了真实光源的软半影。"** 1978 年就已经说清楚 PCF 为什么看起来对。

10. **一条极其实用的产业判断**（在结论前）：

> *"As a final, not unimportant observation, ... **the problem of self shadowing surfaces may not be terribly significant in practice.** ... Shading the spheres according to the position of the light source casting the shadows causes a **smooth shadow transition which obscures the quantization error**. In practice, **translucent shadows** (their translucency corresponding to the additive "ambient" term in most surface shading formulations) **generally look better than deep black shadows**."*

> **【对今天】**：**"环境光项 + 不完全黑化的阴影"既好看、又能掩盖精度问题** —— 这是一条从 1978 到今天都没变的分档技巧：**低档画质不要试图做"精确的硬阴影"，而应做"偏亮、偏软、带环境项"的阴影**，观感更好且更便宜。

## Key Contribution

1. **首次让阴影在曲面场景上可用**（把阴影从"平面多边形"解放出来）；
2. **用 Z-buffer 建立两视图之间的线性数据关联** —— 结论段称这是本文相对 Crow 1977 的关键跨越；
3. **开销可预测**：每个光源约 2× 场景渲染 + 分辨率相关的固定变换开销，且**与场景复杂度无关**（后处理版）；
4. **系统刻画了图像空间阴影的失效模式**：量化、摩尔纹、自阴影、bias、边缘走样；
5. **给出"低通滤波 ≈ 软半影"的物理解释** —— PCF 的思想源头。

## Why It Works

因为它把**一个"几何/可见性问题"降维成了"图像比较问题"**。原文对 Z-buffer 这一选择的论证自成一段，值得记下来（它是**今天所有"屏幕空间技术"的共同辩护**）：

- **无需预排序** → 场景复杂度可无限；
- **成本随平均深度复杂度线性增长** —— 原文甚至给了算法层面的解释：Z-buffer 在 X/Y 上是**桶排序**（基数覆盖键值范围，免比较），在 Z 上退化为一次比较；因此**它是唯一成本随深度复杂度线性增长的可见性算法**；
- **副产品是"深度图"** —— *"Such algorithms are noteworthy because **their expense does not vary with the size or complexity of the environment, but depends only on the image resolution**."*

> **【对你的意义】**：这就是"**屏幕空间技术 = 成本与画质解耦于场景复杂度**"这条原则的原始表述。**同一句话在 2026 年解释了为什么 Lumen / SSR / MegaLights 都走屏幕空间路线** —— 因为它们的成本只看分辨率，不看你的场景有多复杂。**这也是"档位可以按分辨率定义"的根本理由。**

## Limitations（原文自述，且几乎每条都活着）

| # | 限制（原文） | 今天的形态 |
|---|---|---|
| 1 | **只能在自己的视锥内投影阴影**：*"The assumption is that points transformed into the light source space which lie outside the viewing volume of the light source are **eliminated**. Shadows may only be cast within the viewing volume of the light source."* | **阴影贴图只覆盖一个区域** → CSM 分级、per-light 分辨率、caster 剔除 |
| 2 | **全向光需要分扇区**：*"if a light source ... is to cast shadows in all directions, its sphere of illumination must be **sectored into multiple views** as suggested by Crow"* | **Cube shadow map / 6 面**；point light 比 spot light 贵数倍 |
| 3 | **多视图的直接后果是内存**：*"**The major difficulty with this method is the increased memory required.**"* | **每盏灯一张图 → 显存/带宽预算 → 灯数与分辨率双向受限** |
| 4 | **量化与走样是图像空间算法的主要缺点**：*"quantization and aliasing are the chief drawback of image space algorithms"* | bias、acne、peter-panning、边缘锯齿 |
| 5 | **横向分辨率是平方级代价**：*"extra lateral resolution is purchased at **square law** expense"* | 阴影分辨率与显存/带宽的平方律 |
| 6 | **大透视会加剧量化问题**（观察者视角或光源离场景很近时）| **近距离光的精度问题**；需要更大的 near/far 比值容忍度 |

## 🔴 Cost Model（本文最值钱的部分，单独成节）

把原文的成本表述整理成**可直接用于分档的成本表**：

| 成本项 | 量级 | 与什么成正比 | 原文依据 |
|---|---|---|---|
| **阴影图渲染（每盏灯一遍）** | **≈ 1× 场景渲染**（无着色） | **每盏灯 × 场景复杂度** | *"roughly twice the cost... since the light source view requires no shading computation"* |
| **主渲染（含判定）** | 1× 场景渲染 | 场景复杂度 | 同上 |
| **变换开销（后处理版）** | 常数 | **图像分辨率²** | *"expense increases with the square of the resolution"* |
| **变换开销（逐点版）** | 变量 | **场景深度复杂度** | *"increases linearly with the depth complexity of the scene"* |
| **内存** | **× 光源数 × 面数** | 光源数 × 每光视图数 × 分辨率² | *"The major difficulty with this method is the increased memory required."* |

> **【对你的预算表，直接可用的三条推论】**
>
> **① 为什么动态灯光上限必须是 0–3 这种小整数？**
> 因为单位是"**又一遍完整场景渲染**"。一盏灯 ≈ +1× 场景成本。
> **对照**：同屏粒子多 100 个的成本单位是"100 个点"。
> **所以两个维度的上限数量级天生不同（3 vs 3000），这不是拍脑袋，是成本单位的差别。**
>
> **② 为什么灯光档位通常靠"关灯"而不是"降精度"？**
> 因为**降分辨率只省图像空间那部分（分辨率相关项），省不掉那 1× 的场景渲染**。
> **这条解释了 MegaLights 存在的理由**：MegaLights 的核心是**让大量灯共享一次结构化采样**，从而绕开"每盏灯一遍"的固定倍数（见下）。
>
> **③ 为什么"全向光"比"聚光"贵？**
> 因为需要分扇区（6 面）→ **内存与渲染遍数同时乘以面数**。**这就是"动态灯光预算"里最容易被忽略的隐性乘数。**

## Game Development Relevance

### 🔴 与 MegaLights 的对照 —— **正好扣上你挂了一周多的待办**

你在 [[Personal Knowledge Model]] 里的待办第 4 条是"**动态灯光维度复审（UE 5.8 MegaLights 转 Production 的影响）**"。**本文给出了这个复审所需的对照基线**：

| | Williams 1978 的成本结构 | MegaLights（UE 5.5→5.8） | 差异的本质 |
|---|---|---|---|
| 灯数增长 | **线性**（每灯一遍） | **近似固定开销**（大量灯共享结构化采样） | 从"**每盏灯一个乘数**"变成"**一个固定的采样预算，灯数与质量互换**" |
| 内存 | 每灯 × 每视图 × 分辨率² | 结构化采样缓冲，与灯数弱相关 | 内存从"灯数 ×"变成"采样数 ×" |
| 分档旋钮 | **灯数**（只能关） | **采样数 / 灯的选择与裁剪** | 从"**二元（开/关）**"变成"**连续（采样预算）**" |

> **⚠️ 但 1978 的那条"分档旋钮直觉"仍然有效**：MegaLights 把成本从"灯数线性"改成了"采样预算"，**可你把灯开满时，采样预算本身就会被灯数撑大** —— 成本守恒没有消失，只是换了个变量名。**复审时该问的问题不是"MegaLights 快不快"，而是"它的成本函数现在是关于哪个变量线性的"。**

### 🔴 一条硬约束（与你的 VFX 领域直接冲突，**二手来源，待官方核实**）

多个二手来源（含百度百科条目）记 UE 5.8 MegaLights 的限制为：**不支持半透明物体、流体、云、发丝；不支持前向渲染**；并给出"~70 盏灯以上才显著收益""RTX 4080 上 1440p/4K 提升可达 50%"等数据。

**⚠️ 来源分级**：以上**全部为二手**（百科/聚合站）。**"不支持半透明 / 前向渲染"与社区已知的 MegaLights 限制一致，方向可信；但具体数字（70 盏、50%）未经官方文档核对，标为待核实。**

> **【为什么这条对你极其重要】**：**你的领域（[[AAA Real-Time VFX]]、Niagara 粒子、半透明特效）恰好在 MegaLights 覆盖范围之外。**
> 含义是：**动态灯光的 MegaLights 路径，与 VFX 半透明物体是两套互不相通的账。** 所以"MegaLights 让动态灯光变便宜了"**不能**直接推出"VFX 可以多带动态灯" —— **特效打光的成本结构仍归 1978 那条法则管（每盏灯 +1× 场景，或至少 +1 遍受光物体）。**
> **这是本次入库对你最直接的一条实践结论。**

### 其他可迁移结论

- **不透明物体与半透明物体的阴影账不能混算**（本文的"点光源假设""无排序"与半透明渲染顺序是两套体系）；
- **低档画质应做"亮而软的阴影"而非"精确硬阴影"**（原文 1978 年就有此判断，见 Technical Approach 第 10 条）；
- **阴影分辨率是平方级成本** → 与"贴图尺寸"维度共享同一条法则（见 [[Reeves — Particle Systems (1983)]] 的对照表）。

## Unreal Engine Relevance

- **历史形态**：UE 的 CSM（Cascaded Shadow Maps）正是对"限制 1"（只能在自己的视锥内投影）的直接工程解 —— **用多个级联把有限的分辨率按距离分配**；
- **现形态**：UE5 的 **Virtual Shadow Maps** 把"分辨率预算"从"全局定一张"改成"**逐页按需**"，本质上是对"限制 5（横向分辨率平方级代价）"的绕行；
- **最新形态**：**MegaLights** 绕行的是"限制：每盏灯一遍"—— 你没在跟 1978 的算法竞争，你在跟 1978 的**成本结构**竞争；
- **可动手的验证（30 分钟）**：在一个只有材质极简几何的场景里，抓一次 GPU 剖析，**分别看 Shadow Depths / Shadow Projection 与 Base Pass 的占比** —— 用实测验证"材质越贵、阴影占比越低；材质越简、阴影越接近 50%"这条 1978 年就写明的规律。**这条实测对你的分档表是可直接落地的输入。**

## Technology Evolution

```text
1972  Newell et al. —— 隐藏面的一种解法
1974  Catmull —— Z-buffer（成本线性增长 + 深度图副产品）
1977  Crow —— 提出"两遍可见性即可算阴影"（平面多边形；点光需分扇区）
★ 1978  Williams —— 本文：Z-buffer 两视图 + 线性变换关联 → 曲面阴影首次可用
        ↓                    （"depth map" 一词引自 Levine et al. 1973；"shadow map" 是后起叫法）
2006  CSM —— 分辨率按距离分级（解"限制 1"）
2008+ PCF / PCSS / VSM —— 解"限制 4"（硬边与噪声），PCF 的思想源头即本文的低通滤波段
2021  Virtual Shadow Maps（UE5）—— 逐页虚拟化（解"限制 5"）
2024  MegaLights（UE 5.5 首发）—— 大量灯共享结构化采样（解"每盏灯一遍"）
2025-11  UE 5.7 —— MegaLights 转 Beta
2026-06  UE 5.8 —— MegaLights 转 Production-Ready；Lumen Lite（2× 速度）
        ↓
        ★ 整条线上的每一个技术，都是在绕开本文列出的六个限制之一
```

## Relationships

### Based On

- **Catmull 1974（Z-buffer 细分曲面）** —— 本文的全部基础；原文明确"Z-buffer 提供了关联不同视图数据的直接手段"；
- **Levine / O'Handley / Yagi 1973** —— "depth map" 一词的引文来源（原文参考文献 [4]）。

### Extends

- **Crow 1977（*Shadow Algorithms for Computer Graphics*）** —— **本文是它的实用化**：Crow 提出"两遍"，本文给出"怎么把两遍对起来"；**Crow 还提出了点光分扇区（限制 2），本文把它继承并指出其内存代价**。

### Improves

- **所有"只对平面多边形成立"的阴影算法** —— 本文把它们扩展到曲面。

### Related

- [[Reeves — Particle Systems (1983)]] —— **同一天入库的另一半**：Reeves 给"粒子侧的预算法则"（屏占比 × 密度），Williams 给"灯光侧的预算法则"（每灯 2× + 分辨率²）；
- [[Real-Time Global Illumination]] —— **两篇的"做不到"部分是同一件事**：Williams 的阴影不含间接光（只判直接可见性），Reeves 的火墙不能照亮星球。**两者都靠"没做的东西用环境项/手工技巧补"，直到实时 GI 才真正还账**；
- [[Scalability and Quality Tiers]] —— 本文的"成本只看分辨率"是"档位可以按分辨率定义"的根本理由；
- [[Real-Time VFX Performance Budgeting]] —— 动态灯光维度的原始出处。

### Followed By

- **CSM（2006）→ PCF/PCSS/VSM（2008+）→ Virtual Shadow Maps（2021）→ MegaLights（2024–2026）**：见上 Time Line。

## Personal Knowledge State

- **user_level: Easy** —— **机制层是你的日常工作，本笔记不做教学。**
- **按本库 9-20 建立的分层模板拆两层**：

| 层 | 内容 | 状态 |
|---|---|---|
| **机制层** | 两遍渲染、光源视角深度图、bias、PCF、级联、cube map | **Easy —— 不再复述** |
| **成本模型层** | **① 每盏灯 ≈ +1× 场景渲染（无着色，故"约"2×）② 横向分辨率是平方级 ③ 全向光按面数乘 ④ 阴影占比取决于材质复杂度** | **Normal —— 本文唯一能给你新东西的地方** |

## Learning Value

**给你的分档表补上"为什么"**：

1. **把动态灯光上限从"经验值"变成"可推导值"**：`≤3 / ≤2 / ≤1 / 0` 现在有了 1978 年的成本依据（单位是"完整渲染遍"而非"小图元"）；
2. **给出一条反直觉的实测题**（见 Unreal Engine Relevance 的 30 分钟验证）：**材质复杂度决定阴影占比**；
3. **给 MegaLights 复审提供对照基线**（见 Cost Model 推论 ② 与 Game Development Relevance）：**该问的不是"快不快"，而是"成本函数现在关于哪个变量线性"**；
4. **提醒一条 VFX 侧硬约束**：MegaLights 不覆盖半透明 → **特效打光仍走 1978 的账**。

## Mastery Criteria（4 条，纸面自测）

1. 说出 **"约 2×" 里那个"约"字为什么存在**（答案：光源视角不需要着色计算，所以省下的比例取决于材质复杂度）；
2. 说出 **为什么"降阴影分辨率"无法解决灯数过多的问题**（答案：分辨率相关项只占图像空间那部分，省不掉每盏灯的 1× 场景渲染）；
3. 说出 **PCF 为什么"看起来是对的"** —— 用原文的物理表述回答（答案：低通滤波主观上近似了真实光源的软半影）；
4. 说出 **本文列出的六个限制里，CSM / VSM / MegaLights 分别绕开的是哪一个**。

> **一句话检验**：能说出 **"动态灯光的上限之所以是 0–3 而粒子是 100–3000，是因为前者的成本单位是'又一遍场景渲染'"**，即算抓住了本文。

## Visualization

![[预算五维_1978-1983_源头图解.html]]

见 [[预算五维_1978-1983_源头图解]] —— 本文的每灯成本法则与 Reeves 1983 的屏占比法则并列在同一张图上，右侧直接对到你当前的 SABC × 五档矩阵。

## Notes

- 2026-09-21 入库。**原文 PDF 已下载并逐节核对**（5 页，`https://cseweb.ucsd.edu/~ravir/274/15/papers/p270-williams.pdf`）。**所有成本表述、原文引语、作者自述均来自抽取文本，非记忆。**
- **发表信息核对**：SIGGRAPH '78，Computer Graphics 12(3), pp. 270–274；DOI `10.1145/800248.807402`（与 ACM DL 条目一致）。**本次核对的是该 PDF 版**（含完整"References"段，共 10 条）。
- **作者与单位**：Lance Williams，**Computer Graphics Lab, New York Institute of Technology（NYIT）**。同段落提到 NYIT 当时正计划把 **Floating Point Systems AP120-B 阵列处理机**用于坐标变换的流水化 —— **这是"把变换交给专用硬件"的最早表述之一，也正是后来 GPU 顶点变换的雏形。**
- **⚠️ 两处归属澄清（本次逐条核对后写下，用以纠正常见误传）**：
  1. **"阴影需要两次渲染"归 Crow 1977**，本文原文明确引用并致谢；
  2. **本文把光源视角的深度缓冲叫 "depth map"**（引 Levine et al. 1973），**不是 "shadow map"**；"shadow map" 是后起叫法。**引用本文时请用 "depth map"。**
- **本文的两条"预言"**：① 结论段预测"**内存技术的乐观前景**"会让这类质量-内存权衡的高质量算法成为主流（已被验证）；② 摘要式的判断"**速度不等于计算开销，当可以施加专用硬件时**"（已被 GPU 完整验证）。
- **相关性判断（为什么它与 Reeves 1983 同日入库）**：两篇合起来正好覆盖用户五个预算维度的**全部历史出处**（见 [[Reeves — Particle Systems (1983)]] 的对照表）。**这不是提前安排的计划，而是核对本文成本段时发现它同时命中"动态灯光"与"贴图尺寸"两个维度后的即时补入。**
