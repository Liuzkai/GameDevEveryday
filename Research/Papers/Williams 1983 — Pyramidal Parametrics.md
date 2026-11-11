---
type: paper
title: "Pyramidal Parametrics"
authors: [Lance Williams]
year: 1983
published: "1983-07（SIGGRAPH '83）"
venue: "SIGGRAPH '83 / Computer Graphics 17(3), pp. 1–11"
url: "https://doi.org/10.1145/800059.801126"
code: ""
project_page: "https://people.csail.mit.edu/ericchan/bib/pdf/p1-williams.pdf（PDF 镜像）"
category: [texture-mapping, antialiasing, level-of-detail, filtering, reflectance-mapping, classical]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: "Easy（机制）/ Normal（妥协与判据）"
status: unread
aliases: [Pyramidal Parametrics, Williams 1983, mipmap 起源, MIP mapping, 金字塔参数化, multum in parvo]
tags: [texture-mapping, antialiasing, lod, filtering, classical, hardware]
---

# Pyramidal Parametrics（Williams 1983）

## TL;DR

**mipmap 的原始论文。但它的标题不是 mipmap——是"金字塔参数化"。**

Williams 提的不是一个纹理技巧，是一个**框架**：把一个可滤波的量做成**分辨率逐级减半的序列**，用三个坐标 $(U,V,D)$ 索引——$U,V$ 是空间坐标，$D$ 是**层**。层内双线性、层间线性，于是"任意大小的滤波"变成"两次查表 + 七次乘法"。

这个框架他用了三次：**纹理**（§3）、**光照/高光**（§4，反射贴图的金字塔版）、**曲面本身**（§5，标题就叫 *Levels of Detail in Surface Representation*，把顶点坐标当成被滤波的量）。**所以这一篇是纹理 LOD、光照 LOD、几何 LOD 三条分支的共同祖先**，本库此前把它只当成"mipmap 起源"。

而全文最值钱的一句，是作者自己写下的**妥协**：

> *"probably the most severe compromise made by mip filtering is that it is symmetrical. Each of the prefiltered levels of the map is filtered equally in X and Y."*

**这句话就是各向异性过滤问题的原始表述**——它要等 43 年，才在 [[2026-11-11-Fujieda — Explicit LOD-Conditioned Anisotropic Filtering for Neural Texture Compression|AF-NTC 2026]] 里被正面处理。

## Problem

把一张平面图映到曲面上，会同时出现两个方向相反的采样问题：

- **放大**（表面在屏幕上变大）：源图采样不够密，需要**插值**；
- **缩小**（表面变小/倾斜）：一个像素覆盖源图很多样本，需要**多个样本的近似**。

于是"一个不断变化的、形状随视角而变的采样窗口，要在源图上移动"。**每一帧、每个像素都要重新算这个窗口里的积分**——这正是 1983 年做不到实时的那件事（Feibush/Levoy/Cook 1980 的显式卷积，或 Crow 1976 的面积平均）。

Williams 的问题陈述非常精确：**要降的不是"滤波的质量"，是"每次采样的计算量随滤波面积增长"这件事。**

## Historical Context

```text
1976  Blinn & Newell：纹理 + 反射映射
        · 首次把"环境的一张球面投影图"当查找表 —— 脚印问题随之出现
1976  Crow：锯齿问题的博士论文（面积平均的理论）
1978  Catmull：把多张预过滤图用于纹理（双三次面片，论文里没写）
        · Williams 在致谢里明确把"第一个"归给他
1979  NYIT 内部开始用 mip（Williams 原文：since 1979）
  ↓
1983 ★ 本篇：把"预过滤多尺度 + 层间插值"写成一套可索引的数据结构
        · 不只是纹理：光照（§4）与几何（§5）用同一套
  ↓
1989  Heckbert：脚印的椭圆几何（EWA）
1999  Feline（McCormack et al.）：沿主轴 tap 串 ⇒ 硬件 AF
2000s  硬件固化：mip 链 + 三线性 + 各向异性倍数成为 API 的一个整数
  ↓
2026  AF-NTC：在没有硬件过滤的解码器里，把"一个标量 D"拆成三个量
```

**与 [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)|Clark 1976]] 的关系要说清楚**：两篇**互不引用**（Williams 的参考文献里没有 Clark），但它们给的是同一个判据的两种载体——Clark 按"屏幕占比"给几何分细节，Williams 按"投影面积"给纹理/光照/顶点分细节。**所以 LOD 这条线的源头不是一条，是两条平行的 1976/1983。**

## Core Idea

**把"滤波"从"每次采样做积分"改成"预先做好一串尺度，采样时只做插值"。**

三个设计决定，每一个都值四十年：

| 决定 | 内容 | 为什么活下来了 |
|---|---|---|
| **尺度取 2 的幂** | 每层线性尺寸减半（样本数 1/4） | 索引 = **二进制移位**；地址可由 $U,V$ 一次移位得到；硬件实现几乎零成本 |
| **层间也插值** | 不只层内双线性，$D$ 连续 ⇒ 层间线性 | 消除"跳层"的突变（pop），把离散的层变成连续的尺度轴 |
| **滤波固定为方形盒** | 每层在 X/Y 上**同等地**过滤 | 实现最简单、索引最快——**也是它唯一的、致命的妥协** |

Williams 对第三条的解释是全文最诚实的一段：**"all filters are approximated by linearly interpolating a set of square box filters, the sides of which are powers-of-two pixels in length."** 也就是说：**这套系统能用任意核滤波，前提是那个核能被"若干方形盒的线性组合"逼近。** 圆形的、各向同性的脚印可以；椭圆的、长条的不行。

## Technical Approach

### 数据结构

- 图像分成 R/G/B，各自一套金字塔；每层是上一层的一半线性尺寸。
- 五层金字塔首尾相接排进同一块缓冲，**连续四次除以四**正好把缓冲区均分给三个通道，**角落剩一个未用像素**。
- 索引是 $(U,V,D)$：**$U,V$ 空间，$D$ 层**。对应点用一次二进制移位即可跨层寻址。

### 采样成本

每个通道 **8 次像素读 + 7 次乘法**（层内双线性 4 读、层间两图各 4 读）。

> 关键性质：**"mapping entails a fixed overhead, which is independent of the area filtered to compute a sample."**
>
> 这一句是整篇的经济学。**滤波面积从 1 个 texel 涨到 1024 个 texel，成本不变。** 四十年后所有纹理硬件仍然是这个合同。

### 滤波器的选择（以及它为什么是妥协）

- **建层**：box（Fourier）窗；**层间**：tent（Bartlett）窗。作者明说："This choice of filters is **strictly for the sake of speed**."
- **边界**：周期纹理要在极边上用对边的像素做双线性；非周期纹理要 clip 或 scale $U,V$，否则会把不该进的层/通道插进来。

### $D$ 怎么算：Heckbert 的 max

$$D \propto \max\left(\sqrt{(\partial u/\partial x)^2+(\partial v/\partial x)^2},\ \sqrt{(\partial u/\partial y)^2+(\partial v/\partial y)^2}\right)$$

**取两个方向的 max，就是把椭圆压成圆。** 这不是疏忽，是"一个标量只能装一个尺寸"的必然操作：既然只能给一个数，就取"够用的那个"（宁可糊，不要闪）。

### §4 光照：同一个金字塔装反射贴图

把 Blinn–Newell 的反射贴图做成金字塔，$D$ 正比于该像素对应的**立体角**。图 18/19 是 before/after：九个光源在测试物体上"参差地闪"，加金字塔后平滑。

> **一处需要纠正的二手资料**：网上不少引文说 Williams 1983 "提出了反射映射"。**不对**——反射映射是 Blinn & Newell 1976（本文参考文献 [1]）。Williams 做的是**把它放进金字塔**，从而让高光也能抗锯齿。

### §5 几何：把顶点坐标当成被滤波的量

**这是最容易被忽略的一节，也是本篇"框架性"的最强证据。**

> *"A pyramidal parametric data structure the components of which are spatial coordinates (the X-Y-Z of the vertices of a rectangular mesh, for example, as opposed to the R-G-B of a texture or illumination map) provides a continuously-variable filtered instance of the surface for sampling at any desired degree of resolution."*

于是同一套机器给出 64×64 / 32×32 / 16×16 / 8×8 四级人脸网格（图 20–23）。**"LOD"这个词在这一节里出现，比 mipmap 那三节更接近今天的意思。**

## Key Contribution

1. **一个数据结构的抽象**：金字塔 = "同一内容的多个尺度" + "层内与层间都插值"。**它不关心被滤波的量是颜色、辐照度还是顶点坐标。**
2. **一条成本合同**：每次采样的成本**与滤波面积无关**。这是实时纹理四十年的地基。
3. **一次诚实的自我限定**：各向同性是"最严重的妥协"，且当脚印偏离对称时，"$D$ 的选择就是在锯齿与模糊之间做交易"。

## Why It Works

### 因为预过滤把"可变成本"换成了"一次性成本"

显式卷积的代价 ∝ 滤波面积；金字塔的代价 = 建层的 $O(N)$（一次性）+ 每次采样 $O(1)$。**只要一张纹理被采样超过几次，这笔投资就回来了**——而纹理在游戏里被采样几十亿次。**这就是为什么 mip 链的 33% 额外内存从来没人质疑。**

### 因为"层"这个额外维度是离散且可插值的

金字塔把"尺度"变成了一个可连续索引的坐标 $D$。**一旦尺度成为坐标，"按需选细节"就退化成"查表"**——这也解释了为什么后来所有 LOD 系统（HLOD、虚拟几何、GS 的 LOD）都在复制这个动作：**先把尺度离散化成一串，再让消费点去索引它。**

### 因为它错的那一面，在大多数场景里不出现

各向同性在"正对镜头的平面"上是正确的；只有在**掠射角**（地面、路面、长走廊地板）与**强透视压缩**下才错。**一个近似能活四十年，不是因为它对，是因为它错的那一面很少被踩到。** 而一旦被踩到（1990 年代末的 3D 游戏地面），产业付的账是 **16× 的采样与带宽**——这就是各向异性过滤的全部由来。

> **这条判据可以推广到任何"用一个标量近似一个形状"的设计**：先问"我这个标量的错误方向是什么"，再问"我的使用场景会不会正好沿着那个方向"。

## Limitations

### 五处冷水

1. **全文零定量评估。** 没有一张表、一个 dB、一次计时。说服力全部来自图（NYIT 的青蛙、Emshwiller 的《Sunstone》片段）。按 1983 年的标准这正常，但按今天的读法，它是**"论点 + 演示"**，不是"论点 + 测量"。
2. **两处妥协是作者自己点名的**：各向同性（"最严重"）+ 滤波器选择"**strictly for the sake of speed**"。**质量从来不是这篇的优化目标，速度才是。**
3. **$D$ 的估计把椭圆压成圆**（取两方向导数的 max）。后续所有各向异性方案（EWA / Feline / 硬件 AF）做的第一件事都是**拒绝这个 max**。
4. **层数有限**：金字塔到 1×1 就停，超过这个尺度的滤波只能靠"最粗层 + 各向同性"硬撑。
5. **内存组织绑定 1983 年的帧缓冲**（按 4 的幂均分、角落留一个空像素）。这个约定今天只剩"层是 2 的幂"这一条还在用。

### 一处方法论上的观察

**这篇论文的说服方式是"消除 artifact 的图"**：点采样 vs 双线性（图 3/4）、点采样 vs 金字塔（图 5/6）、旋转到侧面的多边形（图 8–10）。**它没有证明"更好"，它展示了"不再难看"。** 对 1983 年的读者，这够了；对本库的读者，值得记一笔：**"artifact 消失"是一个比 PSNR 更早、也更持久的验收标准。**

## Game Development Relevance

- **它是唯一一个"每个游戏开发者每天都在用、但几乎没人读过原文"的算法。** 读它的收益不是学会 mip（谁都会），是**知道它的边界写在哪一行**。
- **纹理预算的直接依据**：mip 链 = 原图 × 4/3。这个 33% 是 Williams 的数据结构的直接结果，不是经验值。
- **各向异性倍数的选择**：本篇说明"各向同性是一个 max 操作"，所以 AF 的本质是**把被 max 丢掉的那个方向找回来**。16× AF 的代价（16 次三线性）与收益（掠射角清晰度）都能从这条推出来。
- **虚拟纹理 / streaming**：所有"按 $D$ 只驻留部分层"的方案（稀疏虚拟纹理、mip tail）都是这篇的内存组织在现代的重演。

## Unreal Engine Relevance

- UE 的 `r.MaxAnisotropy`、mip 生成（`Texture2D` 的 `MipGenSettings`）、虚拟纹理的 tile 分级，全部是这篇的直接后裔。
- **一个容易踩的坑**：UE 的 mip 生成默认用 box 下采样（与 Williams 的 box 窗一致），所以"斜面上的远处纹理发糊/发闪"的根因**不是采样器设置，是金字塔的层本身就是各向同性盒滤波的结果**。调 `MaxAnisotropy` 只能减轻，不能消除——**这正好是 AF-NTC 想解决的那一层**。

## Technology Evolution

```text
1976  Blinn & Newell（纹理 + 反射映射）
1978  Catmull（多张预过滤图，未发表细节）
  ↓
1983 ★ Williams：金字塔 + $(U,V,D)$ 索引 + 层间插值
        · 一份机器，三处使用：纹理 / 光照 / 几何
  ↓
1989  Heckbert：脚印椭圆（EWA 的几何基础）
1999  Feline：沿主轴 tap 串（硬件 AF 的形状）
  ↓
2000s  GPU 固化：mip 链 + 三线性 + AF 倍数
2010s  虚拟纹理：把"层"变成"页"，按需驻留
  ↓
2023  NTC：把"层"压进网络（特征金字塔）
2026  AF-NTC：把"一个标量 D"拆成三元组
```

## Relationships

### Based On

- Blinn & Newell 1976（纹理与反射映射）—— §4 的被金字塔化的对象。
- Crow 1976（锯齿问题）/ Feibush-Levoy-Cook 1980（显式数字滤波）—— §2 要取代的那条贵路线。
- Catmull 1978（多张预过滤图用于纹理，未写细节）—— 本篇致谢里明确承认的先例。

### Extends

- [[Level of Detail]] —— **纹理分支的源头**，而且 §5 说明它同时是几何分支的一个独立源头（与 [[Clark — Hierarchical Geometric Models for Visible Surface Algorithms (1976)|Clark 1976]] 平行、互不引用）。
- [[Texture Filtering]] —— 这一概念的历史起点：$(U,V,D)$ 三元组就是今天"脚印三元组"的雏形，只不过 $D$ 只有一个自由度。

### Related

- [[Mesh Parameterization]] —— 纹理坐标从哪来；脚印的雅可比正是定义在参数化之上的。
- [[Gaussian Splatting]] —— 2020 年代的"预过滤"版本：同样是把"每次采样的滤波"换成"预先算好的多尺度"，只是尺度轴变成了协方差。

### Contrasts

- **EWA（Heckbert 1989）**：把脚印当椭圆积，质量对但成本随面积增长；本篇把成本压成常数但假设圆。**这是"对但慢"与"快但错一个方向"的经典分工。**
- **STF（Pharr 2024）**：同样放弃"精确滤波"，但换成**随机采样 + 时域收敛**；本篇是**确定性预过滤**。两条路的分界是"你愿意把误差放到空间还是时间"。

### Followed By

- [[2026-11-11-Fujieda — Explicit LOD-Conditioned Anisotropic Filtering for Neural Texture Compression|AF-NTC 2026]] —— 直接回应本篇自陈的那个妥协：把 $D$ 拆成 $(\text{LOD},\text{stretch},\text{rot})$。

## Personal Knowledge State

- **机制层 Easy**：mip、三线性、各向异性——日常在用，不复述。
- **判据层 Normal（今天新增的三条）**：
  1. **成本合同**："每次采样成本与滤波面积无关"是 mip 的全部价值，也是为什么 33% 内存从来不是问题。
  2. **$D$ 是一个 max**：各向同性不是"近似"，是**主动丢掉一个方向**。所有 AF 方案都在把这个方向找回来。
  3. **同一套金字塔能装颜色、辐照度、顶点坐标**——所以它是"LOD 框架"，不是"纹理技巧"。

## Learning Value

**最高（历史 5 / 当下 5）。** 它是 [[Level of Detail]] 与 [[Texture Filtering]] 两页的共同地基，也是"读完原文才发现二手资料漏了什么"的最好样本（三处：它是三个分支的共同祖先、它没有提反射映射、它自己承认各向同性是最严重的妥协）。

## Visualization

**图 1｜$(U,V,D)$ 三坐标索引。** 左上角是逐级缩小的金字塔（"multum in parvo"——把很多东西塞进一个小地方），右下角那条对角线就是 $D$ 轴。

![[Research/Files/Pictures/Williams 1983 — 图1 mip 金字塔的 UV D 三坐标索引.png]]

**读图**：看每个子图左上角的 $U$/$V$ 箭头——**每一层的 $U,V$ 方向相同、尺度不同**，所以"跨层访问同一点"只需要对坐标做一次移位。**这张图解释的不是"mip 长什么样"，是"为什么它能用一次移位寻址"**：层的排布与坐标的二进制结构是对齐的。这是它能进硬件的真正原因。（出处：Williams 1983，图 1）

**图 2｜点采样 vs 金字塔下采样。** 上图（图 5）是点采样缩小的青蛙：右下的放大细节满是彩色噪点与断裂；下图（图 6）是金字塔：细节糊但干净。

![[Research/Files/Pictures/Williams 1983 — 图5-6 点采样与金字塔下采样对照.png]]

**读图**：**这张图是"artifact 消失"式论证的教科书样本**——没有数字，但有对比。注意上图右边那些**彩色的孤立像素**（红/蓝/黄点），那是把高频信号采到低频网格上的典型症状（混叠折回）；下图把这些颜色换成了"糊"。**"用模糊换掉闪烁"就是 mip 的全部交易**，四十年来没变过。（出处：Williams 1983，图 5、图 6）

**图 3｜同一套金字塔用在顶点坐标上（§5）。** 64×64 → 32×32 → 16×16 → 8×8 的人脸网格，全部由同一套"层内 + 层间插值"生成。

![[Research/Files/Pictures/Williams 1983 — 图20-23 曲面本身的四级金字塔.png]]

**读图**：**这四张图是本篇"框架性"的证据**——被滤波的量从 RGB 换成了 XYZ，机器一行没改。所以"LOD"在 1983 年就已经是**表示无关**的了。对照本库 [[Level of Detail]] 页把 1976（Clark）当唯一起源，**这里应该补一个平行的 1983 节点**。（出处：Williams 1983，图 20–23）

## Notes

- 11 页，图多表零。作者单位是 NYIT（Computer Graphics Laboratory）——1983 年纽约理工是 CG 的重镇（Catmull、Blinn 都待过）。
- **"mip" = multum in parvo**（拉丁语，"许多东西在一个小地方"）。三个字母是缩写，不是"mipmap"的 mip。
- 致谢里有半部 CG 史：Catmull（第一个用多张预过滤图）、Duff（保精度的递归建层）、Lundin（第一版汇编访问例程）、Heckbert（优化建层与访问、研究 $D$ 的各种估计）、Chou（高光抗锯齿与高分辨率反射贴图的实现）。
- **PDF 来源**：`people.csail.mit.edu/ericchan/bib/pdf/p1-williams.pdf`（ACM DL 需订阅，此镜像一次 200，1.39 MB / 11 页，扫描件）。**作者主页/学术镜像第 9 次胜出。**
