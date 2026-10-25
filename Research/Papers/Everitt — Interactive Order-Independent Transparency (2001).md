---
type: paper
title: "Interactive Order-Independent Transparency"
authors: ["Cass Everitt"]
year: 2001
published: "2001"
venue: "NVIDIA Technical Presentation (GDC / developer.nvidia.com)"
url: "https://developer.download.nvidia.com/assets/gamedev/docs/OrderIndependentTransparency.pdf"
code: ""
project_page: ""
category: [rendering, transparency, sorting, real-time]
importance: A
historical_importance: 4
game_relevance: 4
production_readiness: Prototype
user_level: Normal
status: unread
---

# Interactive Order-Independent Transparency

> **证据形态先说清楚**：这是 NVIDIA 的一份 **14 页技术演示文稿（幻灯片）**，不是同行评审论文，**全文没有任何计时数据**。它的价值在于**把"顺序"从一个应用侧问题改造成一个像素侧问题**，并给出一个能用当时硬件实现的配方。本笔记区分"作者说的"与"我推断的"。

## TL;DR

正确半透明的传统做法是**让应用把多边形按深度排好序再画**——这对动态场景、相交面片、循环遮挡全都不成立。Everitt 的做法是**逐层剥离（depth peeling）**：用两个深度缓冲交替，每一遍只保留"比上一层更远的最近片元"，于是**每个像素自己完成排序**，应用侧一行排序代码都不用写。代价是 **n 层 = n 遍场景遍历**；作者的经验值是 **n = 4 通常够用**。

**一句话**：**把排序从"图元顺序"下推到"像素层号"，用重复遍历换正确性。**

## Problem

幻灯片第 4 页把病根列得很干脆：

- 硬件是 **object-order**（按图元提交顺序画），而正确混合要求 **back-to-front**；
- 想排对，就得让应用把多边形排序——**"not very convenient"**；
- 三条硬伤：**多边形不能相交**（相交就必须切）、**动态场景数据库极难维护**、**排序开销全压在应用侧**；
- 所以 `glEnable(GL_BLEND)` 直接开混，结果是图 1 右边那种"内部结构消失、遮挡关系错乱"。

## Historical Context

- **1984 Carpenter 的 A-buffer** 早就有"每像素存一列片元再合成"的想法，但那是软件光栅化时代的**每像素链表**，没有实时硬件能承载；
- **1990 年代末到 2001**，图形硬件只有固定的"深度测试 + 混合"两个阶段，**没有任何 per-pixel 排序能力**——顺序只能由应用提供；
- 于是行业默认了两条路：**① 逐图元排序（近似对，快）**；**② 干脆用加法混合绕开顺序（但对就不了对）**；
- Everitt 这一篇提供 **③ 用像素层号逼近精确排序**：正确性接近 ①，应用侧负担接近 ②，代价是**重复遍历**。

> **它没有解决的事**：深度剥离给的是**离散层数**下的正确（n 层以外的片元被丢弃/近似）；它不是"免费的正确"，是"用 n 倍几何成本买排序"。

## Previous Work

- **逐图元排序 + alpha blend**：所有实时引擎的默认解；对粒子/头发这类**无序、自相交**的图元系统性失败；
- **加法混合**：顺序无关（交换律），但**不能表达吸收/遮挡**——见 [[Reeves — Particle Systems (1983)]] 的"法则 ③"及其边界；
- **软件 A-buffer / 列表式透明**：正确但当时不可实时。

## Core Idea

**用"深度测试的比较方向"当排序算子。**

第一遍：正常深度测试（LESS），拿到**最近**片元的颜色与深度。
第 i 遍：把第 i−1 遍的深度当**只读深度纹理**，深度测试改成 **GREATER**——于是这一遍画出来的，恰好是**"比上一层更远的最近片元"**，也就是第 i 层。同时把这一遍的深度写进**另一个**深度缓冲，供下一遍当输入。

两个深度缓冲**交替（ping-pong）**，每遍把颜色存成一张 layer 纹理，最后从后往前合成。

## Technical Approach

### 1. 双深度缓冲的伪代码（幻灯片第 8 页原文结构）

```text
for i in 0..num_passes-1:
    clear color
    depth unit 0:  bind depth[(i) % 2]
                   depth writes OFF, depth func = GREATER      # 只读"上一层"
                   (i == 0 时关闭深度测试)
    depth unit 1:  bind depth[(i+1) % 2]
                   clear, depth writes ON, depth func = LESS    # 写"这一层"
    render scene
    save color as layer i
```

**关键在深度测试的方向**：`GREATER` 让"被上一层挡住的片元"变成这一遍的可见片元——**遮挡关系被翻译成了前后两次遍历的顺序关系**。

### 2. 当年的实现技巧：借阴影贴图硬件

2001 年没有"双深度缓冲"扩展，但**只读深度测试正好是 shadow map 的能力**，而且**深度测试的相机可以是任意位置**。于是：

- 一份深度用普通深度缓冲（可写）；
- 另一份当阴影贴图纹理（只读）；
- 再用 `DEPTH_REPLACE` 纹理着色器（texture shader）解决**深度精度/不变性（invariance）** 问题——纹理路径要透视校正与投影，与光栅化路径的深度不可能天然逐位一致。

> 这一段的工程含义比算法本身更耐读：**新能力常常不是等来的，是从邻近阶段"借"来的**。

### 3. 合成

- 每遍剥出的 RGBA 存成一层，最后**从后往前**合成；
- 不透明片元**完全覆盖**前面的透明层，所以可以从"全不透明场景"开始，只剥透明层，层数更少；
- **⚠️ 必须剥完全部层**，否则混合顺序错——除非用 `buffer_region` 一类扩展做"整块提前退出"。

## Key Contribution

1. **把"顺序"从图元级降到像素级**：排序不再是应用的数据结构问题，而是**每像素的层号**；
2. **给出一个纯硬件可实现的配方**（双深度缓冲 + ping-pong + 最后合成），不需要改应用内部结构，可"bolt on"到既有 CAD/CAM 程序；
3. **首次把 OIT 的成本明确成"一遍场景 × 层数"**——这个成本表达式此后二十年没变过，只是每一遍越来越便宜。

## Why It Works

因为**"比上一层更远的最近片元"是可以被深度测试直接表达的**：

```text
第 i 层 = argmin{ depth(p) : depth(p) > depth_{i-1}(p) }   （逐像素）
```

`GREATER` 测试 + 上一层的深度值，就是这个 `argmin` 的硬件实现。**没有任何数据结构，只有一个比较方向。** 这也解释了它的两个性质：**逐像素精确**（不是近似排序）、**层数有限**（第 n 层之后的信息被丢弃）。

## Limitations

- **n 遍 = n 次几何遍历**：顶点、图元装配、光栅化全部重跑 n 次；片元数逐层下降，但**几何侧不降**。对顶点密集的场景（角色、植被）代价最高；
- **必须剥完**才能无伪影（否则混合次序错），"提前退出"要靠额外扩展；
- **精度/不变性问题**：深度在"光栅化路径"与"纹理路径"之间必须逐位一致，作者专门用 `DEPTH_REPLACE` 处理；
- **没有计时数据**：全文只有"results are nice"与"n = 4 is often quite satisfactory"两句定性判断——**这是经验值，不是基准**；
- **内存**：n 层颜色 + 两份深度；
- **MSAA 时代的新问题**：深度剥离按像素做，与多样本几何的耦合在原文里不存在（当年无此议题）。

## Game Development Relevance

- **半透明排序是所有 VFX 的老账**：粒子、头发、烟、UI 光晕，都是"无序 + 半透明 + 相互重叠"的图元。深度剥离是**"要正确就必须为最坏层数付费"**的最干净证明；
- **它给出了"加法混合为什么便宜"的反面**：加法混合把顺序维度整个删掉（交换律），代价是不能做吸收与遮挡；**一旦你要吸收，你就买下了"层数"这个成本项**；
- **对预算的直接含义**：任何"正确的半透明"方案，成本都写成 `层数 × 每层单价`。**层数是场景属性（叠了几层），单价是技术属性（一遍多贵）**——两者必须分开谈，这正是你 SABC 表里"同屏粒子数"缺的那一维（见 [[Overdraw]]）；
- **反例价值**：游戏里几乎不用 depth peeling（太贵），用的是逐图元排序 + 加法混合 + 各种近似 OIT。**知道"精确解长什么样、贵在哪"，才能判断近似解在哪儿失守。**

## Unreal Engine Relevance

- UE 的半透明是**逐图元排序 + 逐像素混合**：`Translucency Sort Priority`、以及按距离的排序策略；Niagara 粒子还可在发射器内按 `Sort Mode` 排序；
- **跨发射器 / 跨材质 / 自相交粒子**没有全局顺序——这正是 depth peeling 想绕开的"应用侧排序"问题；
- 若要做真正的 OIT，需要自定义 Pass（MRT 累积 + 合成），不属于 UE 默认管线；**代价模型仍可用本文的 `层数 × 一遍场景` 估算**；
- **实践建议**：把"这个特效是否可以在加法混合下成立"当作第一问；只有必须表达吸收（烟、水、有色玻璃）时才引入排序成本。

## Technology Evolution

```text
1984  Carpenter  A-buffer：每像素片元列表（软件，正确，慢）
        ↓ 硬件只能"深度测试 + 混合"
2001  Everitt    Depth peeling：层号 = 排序（n 遍场景，逐像素正确）
        ↓ 每遍还是太贵
2008  Bavoil & Myers   Dual depth peeling：前后同时剥，约 2×
        ↓ 不想固定层数
2010  Enderton 等  Stochastic transparency：用随机采样换掉排序（噪声换成本）
2010s 逐像素链表（DX11 UAV）/ per-pixel linked list：A-buffer 回到硬件
        ↓ 只要能近似
2013  McGuire & Bavoil  Weighted Blended OIT：一遍、加法式累积、放弃精确顺序
        ↓ 图元本身变了
2023+ 3D Gaussian Splatting：每像素几十~几百层 → overdraw 回到舞台中心
```

**四代其实在换同一个变量**：**"顺序"由谁负责**——应用（排序）/ 像素（剥离）/ 随机（采样）/ 权重（近似）。

## Relationships

### Based On

- **固定功能深度测试与混合阶段**（2001 年的硬件能力边界）；
- **Carpenter 1984 A-buffer** 的"逐像素合成"思想（本文用硬件比较替代链表）。

### Extends

- [[Reeves — Particle Systems (1983)]] 的"顺序无关"讨论：Reeves 用**加法混合**绕过顺序，Everitt 用**层号**换回顺序——**同一约束的两次不同解法**。

### Related

- [[Overdraw]] —— 层数就是 overdraw 的上界；本文是"层数即成本"的最早清晰表述之一；
- [[Particle Systems]] / [[Niagara]] —— 半透明粒子的排序问题；
- [[Gaussian Splatting]] —— 同一问题在 2023 年后以"每像素数百层"的规模复现；
- [[Tile-Based Rendering]] —— TBDR 上"层"还会额外触发 tile 内存与 flush。

### Followed By

- Bavoil & Myers, *Order Independent Transparency with Dual Depth Peeling*（NVIDIA, 2008）——前后双向剥离；
- Enderton 等, *Stochastic Transparency*（I3D 2010）——用随机采样替代排序；
- McGuire & Bavoil, *Weighted Blended Order-Independent Transparency*（JCGT 2013）——一遍近似，工业界最常用；
- 逐像素链表 / A-buffer 的硬件回归（DX11 时代）。

## Personal Knowledge State

**Normal** —— 混合与排序的工程直觉你已经有了（加法混合免排序、alpha 混合要排序），本文补的是**"精确顺序的代价长什么样"**这一格：`层数 × 一遍场景`，以及"GREATER 测试当排序算子"这个技巧。

## Learning Value

- **可迁移的判断**：任何"顺序很重要但我们不想排序"的方案，都可以问"**我把顺序外包给了谁**"（像素层号 / 随机 / 权重 / 时间累积）；
- **可迁移的成本式**：`层数 × 每层单价`——今天算 GS overdraw、算粒子叠层、算烟雾体积切片，都是这个式子；
- **不必学的部分**：2001 年的 `DEPTH_REPLACE` 与 shadow map 借用技巧已过时，读它是为了看清"约束如何塑造方案"。

## Visualization

![[Research/Files/Pictures/Everitt 2001 — 图1 有无OIT对照.png]]

**读图**：左 = 有 OIT，杯壁与内部球体的前后关系正确；右 = 直接开混合，内部结构糊成一团、遮挡关系错乱。**注意右边不是"更透明"，是"顺序错了"**——同样的 alpha，顺序错就变成另一种材质。（图 1，Everitt 2001，幻灯片第 3 页）

![[Research/Files/Pictures/Everitt 2001 — 图3 深度剥离剖面示意.png]]

**读图**：剖面视角下每一遍剥掉一层。**粗黑线 = 这一遍的可见面（离相机最近的未剥面）**，细黑线 = 被挡住的，浅灰 = 已剥走。三幅图连起来看：**每剥一次，"最近的未剥面"就后退一层**——这就是 `GREATER` 测试在做的事。（图 3，Everitt 2001，幻灯片第 7 页）

![[Research/Files/Pictures/Everitt 2001 — 图2 逐层剥离Layer0-3.png]]

**读图**：同一场景的第 0/1/2/3 层。第 0 层是不透明外壳；第 1 层露出内壁与把手内侧；到第 3 层只剩零星碎片。**层数越往后，覆盖率越低**——所以每多剥一层的**片元成本在下降，几何成本不变**。（图 2，Everitt 2001，幻灯片第 6 页）

![[Research/Files/Pictures/Everitt 2001 — 图4 1至4层画质对照.png]]

**读图**：这是全文唯一"定量"的东西——1 层基本是错的（内部全丢），2 层开始有内部结构，3 层接近正确，4 层与最终结果几乎无差别。作者的结论是 **n = 4 通常够用**。**注意这只是单场景的目测**，没有误差度量，也没有计时。（图 4，Everitt 2001，幻灯片第 11 页）

## Notes

- **一句话记住它**：深度剥离不是"更好的透明"，是**"把排序这件事变成可数的层"**——一旦可数，就能定价；
- **与 [[2026-10-21-Beche — KoalaGS Complexity-Aware Continuous Level of Detail for Gaussian Splatting|KoalaGS]] 的对照**：GS 的排序键在优化"哪一层先画"，depth peeling 在优化"层从哪来"——**同一个"层"字，一个当预算单位，一个当排序单位**；
- **写作时的自我提醒**：本文所有"n 遍"的成本表述都是**结构性推断**（n 次几何遍历是伪代码的直接后果），不是作者给的测量；作者的定性结论只有 "results are nice" 与 "n = 4 is often quite satisfactory"。
