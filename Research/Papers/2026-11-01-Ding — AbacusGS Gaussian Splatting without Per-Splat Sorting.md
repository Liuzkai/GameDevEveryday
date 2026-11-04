---
type: paper
title: "AbacusGS: Gaussian Splatting without Per-Splat Sorting"
authors:
  - Jiacheng Ding
  - Yanliang Qi
  - Cong Guo
  - Peihua Mai
year: 2026
published: 2026-11-01
venue: arXiv preprint（cs.CV，cross-list cs.GR），10 页 5 图 6 表；University of Memphis × National University of Singapore
url: https://arxiv.org/abs/2611.01534
code: "作者称将开源（截至入库未给链接）"
project_page: ""
category:
  - rendering
  - gaussian-splatting
  - real-time
  - mobile
  - data-layout
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: Prototype（OpenGL 查看器实测；真机未测）
user_level: Normal
status: unread
tags:
  - gaussian-splatting
  - sorting
  - order
  - data-layout
  - mobile
  - memory
---

# AbacusGS: Gaussian Splatting without Per-Splat Sorting（Memphis × NUS, 2026）

## TL;DR

**排序键每帧都变，排序结果几乎不变——变的那部分只有一个比特。** AbacusGS 把 3DGS 的混合顺序**存进场景布局**：splat 分进 ≤128 个一盒的自适应八叉树盒子，每盒在写入时按一个轴 $a_b$ 排好序；每帧只排**可见盒子**（不是 splat），再用**每盒一次点积** $s_b=\mathrm{sign}((\bar\mu_b-o)\cdot a_b)$ 决定这盒是正读还是反读。

结果：在一个 **CPU 排序**的 OpenGL 查看器里，排序 24.1 → 4.6 ms，整帧 26.8 → **6.0 ms（4.5×）**；排序条目 2,058k → **35k**（同口径可见集 1,099k → 35k，正好等于"平均每盒 31 个 splat"）。质量：不微调 −1.27 dB，**顺序感知微调后 −0.13 dB**。附带一个 **17 B/splat** 的紧凑格式（3DGS PLY 是 248 B）。

> **一句话定位**：库内"排序线"的**第七个节点，也是换轴的一个**——①–⑥ 全在帧内做文章（排得快 / 不排 / 换基元 / 随机化），⑦ 把**顺序本身搬出帧**，变成资产的一部分。

## Problem

3DGS 每帧把 N 个 splat 按中心深度重排一遍。这笔账**取决于平台**，而论文诚实地把它拆开了：

| 平台 | 排序成本 | 是否瓶颈 |
|---|---|---|
| 桌面 GPU（CUDA / WebGPU radix sort） | 百万级 splat **< 1 ms** | 不是 |
| Web / 移动端查看器（一个 instanced quad + **CPU 线程排序**） | 本文 OpenGL 查看器：**24.1 ms / 26.8 ms 帧** | **是，占 90%** |
| 同上，600 万 splat | Visionary 报 **173 ms** | 是 |

而且每次新顺序都要**上传一份 N 条的索引缓冲**——排序本身之外还有一笔带宽账。

所以论文的问题被限定得很清楚：**不是"3DGS 排序慢"，是"用 CPU 排序的那一类渲染器慢"**。这个限定很重要，它决定了 4.5× 只在特定形态成立（见 Limitations）。

## Historical Context

排序在实时渲染里是一条老线，而"顺序外包给谁"这个问题已经换过四次答案（[[Everitt — Interactive Order-Independent Transparency (2001)|Everitt 2001 的框架]]：应用 / 像素 / 随机 / 权重）。本文给的是**第五个答案：资产**。

- **1984 Porter-Duff `over`**：给顺序一个有定义的算子，但没说顺序从哪来；
- **1980 Fuchs-Kedem-Naylor BSP**：树结构给出**精确**的可见序，且"只在相机穿过分割面时才变"——这是本文思路的直系祖先（论文自引）；
- **2001 Everitt depth peeling**：把顺序外包给**像素**（每层剥一次）；
- **2010s 随机透明**：把顺序外包给**随机数**；
- **2013 WBOIT**：把顺序外包给**权重**，彻底不要顺序；
- **2023 3DGS**：回到"每帧重排"，因为 splat 太多、over 不可交换；
- **2025–2026**：①–⑥ 六个节点（见 [[Gaussian Splatting]] 排序线表）都在帧内优化；
- **2026 本文**：把顺序外包给**布局**——写一次，读时只决定方向。

另有一条非 GS 的直接前身：**Voxlines 2023**（透明流线：每帧排 voxel，voxel 内部沿坐标轴存线序，反向时倒读）。本文与它的三点差别是：自适应盒子（不是均匀 voxel）、轴由训练相机的主特征向量定、以及**用这个序去微调模型**。

## Previous Work

论文把"减少排序成本"分成三族，并指出三族各自缺了什么：

| 族 | 代表 | 做法 | 缺什么 |
|---|---|---|---|
| 少算几次 | Neo、Lumina、splat（视角几乎不变时跳过） | 复用上一帧的有序表，增量更新 | **仍是每 splat 一条记录** |
| 不要顺序 | WSR（学出来的加权和）、Mobile-GS、moment-based OIT、StochasticSplats、[[2026-09-29-Gaussian Stippling — Efficient Sorting-Free 3D Gaussian Rendering\|Stippling]]、SteadySplats | 换掉 `over` 算子 | **换了算子 ⇒ 模型要重训/重拟合**；随机估计在实时采样数下有噪 |
| 给"组"排序 | Duplex-GS（排 cell，组内加权和）、StreamingGS（排 voxel，**每帧还排组内**）、Hierarchical 3DGS / Octree-GS / V3DG（排 LoD 选出的 splat） | 缩小排序对象 | **组内顺序不是存的，是要算的** |

**空白就在这里：没有人把"组内顺序"当成可以写下来的东西。** 本文的全部动作就是把这一格填上。

## Core Idea

一句话：**"盒内顺序"不是每帧量出来的，是写的时候定下来的；每帧只需要知道相机在这个盒子的哪一侧。**

三个部件：

**① 盒子（自适应八叉树）。** 按 48 位 Morton 码排 splat → 建自适应八叉树 → 叶子 = 最大的、≤K 个 splat 的节点（K=128）；深度 16 还超 K 就切块。每盒是 CSR 式的一段连续区间，**平均 31 个 splat**。盒头存：起始偏移、八叉树格、剔除半径、轴 $a_b$。

**② 可逆存序（reversible stored order）。** 写入时按 $a_b\cdot\mu_i$ 升序存。相机沿 $a_b$ 看 → 正读就是前到后；相机沿 $-a_b$ 看 → **倒读**就是前到后。方向由一个符号位给：

$$s_b=\mathrm{sign}\big((\bar\mu_b-o)\cdot a_b\big)$$

轴怎么选是个漂亮的小技巧：取**训练相机方向的协方差主特征向量**

$$a_b=\text{principal eigenvector of}\ \sum_c v_{bc}v_{bc}^{\top},\quad v_{bc}=\text{unit}( \text{box centroid}-\text{camera}_c )$$

理由：训练相机"大体上沿这个轴看这个盒子"。**对 360° 环绕采集**，$\sum_c v_{bc}$ 会因为方向抵消而接近零（平均方向没定义），但 $\sum_c v_{bc}v_{bc}^\top$ **不会**——主轴仍然有定义。这条对"扫描类资产"是关键，因为 360° 采集恰恰是 GS 最常见的采集方式。

**③ 每帧的两级序（two-level order）。**

$$\pi=\big[\rho_{b_1},\rho_{b_2},\dots,\rho_{b_V}\big],\qquad \bar z_{b_1}\le\bar z_{b_2}\le\cdots\le\bar z_{b_V}$$

先按**盒子质心深度**排可见盒子（V 个 key），再按各自的存序（或倒序）读盒内（$N/31$ 个 splat，零排序成本）。

**缺的那一块：微调。** 两级序只是真序的近似，误差有两个来源——相邻盒子在共享面附近**深度区间重叠**、存轴只在部分相机上平行于视轴。所以要把已有的 3DGS 模型**按这个序重新拟合**（见下节）。这一步不是可选的：论文的 −1.27 dB → −0.13 dB 全靠它。

![[Research/Files/Pictures/Ding 2026 — 图2 写一次 每帧只排盒子.png]]
**读图**：左（a）写时——八叉树盒子，每盒里的 splat 沿轴 $a_b$ 排好（编号 1…7）；中（b）每帧——只排可见盒子，**一个点积**给出符号 $s_b$，正读或倒读；右（c）绘制——每盒一条 `glMultiDrawElementsIndirect`，vertex shader 用 `gl_DrawID` 取盒号，硬件混合直接用 3DGS 的 `over`，**没有逐 splat 的索引上传**。出处：Ding et al. 2026, Figure 2。

## Technical Approach

### 顺序感知微调

把可微 tile 光栅器的**深度键**换掉，换成"盒秩 + 盒内位次"编码：

$$\kappa_i = 2^{12}\,r(b_i) + \ell_i$$

$r(b)$ 是盒子在盒序里的秩，$\ell_i$ 是 splat 在自己盒子遍历里的位次。这样每个 tile 的列表顺序**就是** $\pi$，前向与反向都用它。

三个约束条件（很重要，决定了这是"重拟合"而不是"微调"）：
- **盒子划分与存序在微调期间冻结**——因为它们已经是存储格式的一部分；
- **关闭 densification**（不能增删 splat，否则盒子边界全变）；
- 位置学习率取 3DGS 调度的**终值**，其余学习率用 gsplat 常数。

步数：float 模型 3,000 步，compact 模型 10,000 步。

### 紧凑格式：13 B/splat + 16 B 盒头 = 17.3 B

| 字段 | 位数 | 说明 |
|---|---|---|
| 位置 | 10 bit/轴 | **相对八叉树格**（不是相对全局包围盒——这是它比 PlayCanvas 16 B 更省的原因） |
| opacity、3 个 DC 色、平均 log-scale | 各 8 bit | |
| 形状 | 16 bit 索引 | 查 65,536 项**形状码本**（log-Euclidean 特征 k-means；各向同性 splat 共用一项，与旋转无关） |
| SH 1–3 阶 | 16 bit 索引 | 查 65,536 项**颜色码本** |
| 盒头 | 16 B | 起始偏移 / 格 / 剔除半径 / 轴 |

码本的 k-means 权重用**每个 splat 的训练损失敏感度**（Niedermayr 2024 用的是"像素值之和"）。论文报了两处消融：16k 码本 −3.64 dB → 64k −3.08 → 加敏感度 −2.52 → 换 log-Euclidean + 损失敏感度 **−1.92 dB（不微调）**。也就是说**码本这一层是"三个决定叠起来才够用"**。

### 两个渲染器

- **OpenGL / 硬件光栅化**：每 splat 一个 instanced quad，vertex shader 做 EWA 投影；**每盒一条间接绘制命令**，shader 里 `b = gl_DrawID`，`i = s[b]>0 ? k : n[b]-1-k`，`j = start[b]+i`。**每帧不建、不上传索引缓冲**。硬件从后往前混合，所以渲染时按 $\pi$ **反向**遍历。
- **CUDA tile renderer**：把 (tile, splat) 对的全局排序换成 **(tile, box) 两级索引**——排序对象从"tile×splat 对"降到"tile×box 对"，每个 tile 顺流读取自己那些盒子的 splat，屏幕矩形不交的直接跳过。

## Key Contribution

1. **把"混合顺序"定义成一个可存储的量**，并给出写入时定轴、运行时只需一个符号位的完整方案；
2. **顺序感知微调**：证明"顺序换掉之后模型可以适应回来"（−1.27 → −0.13 dB），而不是"顺序换了就掉点"；
3. **一份布局同时是三样东西**：排序结构、压缩格式（17 B）、绘制命令组织（每盒一条 indirect draw）——这三件事在别的方案里是三套数据结构；
4. 一组诚实的横向数字：**只有 CPU 排序时才有 4.5×**，GPU 排序时 1.46×，4K 下 13 个场景里 4 个反而更慢。

## Results

**帧时间与排序时间**（13 场景中位数，RTX 6000 Ada；1× 指基础分辨率）：

| 方法 | 排序 ms | 帧 ms | 每帧排序条目 |
|---|---|---|---|
| 3DGS，深度排序 | 24.1 | 26.8 | 2,058k |
| 3DGS + 盒剔除 + 深度排序 | **33.1**（更慢） | 35.0 | 1,099k |
| 每帧排盒内（StreamingGS 式） | 48.9 | 50.7 | 1,099k |
| **AbacusGS，展开索引** | 8.8 | 10.7 | 1,099k |
| **AbacusGS，multi-draw** | **4.6** | **6.0** | **35k** |
| AbacusGS，multi-draw，K=64 | 8.5 | 10.1 | 66k |

GPU 排序（对照）：3DGS 1.3 / 3.8 ms；AbacusGS 1.1 / 2.6 ms ——**帧只快 1.46×**。4K：3DGS 30.6 ms，AbacusGS 9.3 ms（但论文正文说 4K 下 13 个场景有 4 个**略慢于** 3DGS，说明中位数掩盖了分布）。

**质量**（13 场景，对比"同样微调步数的 3DGS 对照"）：

| 配置 | M360 PSNR | T&T | DB | 平均差 |
|---|---|---|---|---|
| 3DGS +3k 步（对照） | 27.43 | 23.71 | 29.64 | — |
| AbacusGS 不微调 | 26.05 | 22.56 | 28.74 | **−1.27 dB** |
| AbacusGS +3k 步 | 27.29 | 23.56 | 29.56 | **−0.126 dB** |
| AbacusGS compact +10k 步 | 27.26 | 23.55 | 29.66 | **−0.172 dB** |
| StochasticSplats 64 spp | 24.06 | 20.65 | 27.33 | −1.25 / −1.11 / −0.75 |
| StochasticSplats 1024 spp | 27.33 | 23.56 | 29.72 | −0.16 / −0.19 / −0.03 |

**存储**：

| 方法 | 文件 B/splat | 显存 B/splat | 首帧加载 |
|---|---|---|---|
| 3DGS PLY | 248.0 | 236.5 | 293 ms |
| AbacusGS（本文 loader） | **19.3** | 36.8 | 69 ms |
| AbacusGS（16 B 盒头，按格式计算） | **17.1** | **17.3** | — |

**浏览器模拟**（headless Chrome，手机视口，CPU 降频 4×）：3DGS 排序 45.2 ms / 帧 65.7 ms / 26–51 FPS；AbacusGS 4.2 / **21.6** / 60 FPS（四个场景全部到 60）。

![[Research/Files/Pictures/Ding 2026 — 图1 八叉树盒子与前后向读取.png]]
**读图**：一张图讲完机制。左：原图。中：按八叉树盒子着色——同一颜色的 splat 在空间上是一小片，所以它们的相对前后关系在整个环绕视角里基本不变。右：按盒子质心深度从近到远着色，蓝盒正读、橙盒反读。**颜色块是"局部"的证据，明暗是"顺序"的证据**——两者都只需要盒子这一层。出处：Ding et al. 2026, Figure 1。

## Why It Works

**① 变化的自由度只有一个比特。** 一个盒子里的 splat 通常覆盖一小片表面，深度序随相机变化很小——**除了相机转到这片表面的另一侧时整体反号**。所以"每帧重排 31 个 splat"这件事的真实信息量是 1 bit。这是全文最锋利的一刀，也是可以迁移到别处的判据。

**② 两级序什么时候等于真序，条件写得很干净**：可见盒子的深度区间不重叠，且每盒存轴平行于视轴。两条都不满足时，误差**被局限**（盒对之间 / 盒内部），所以可微调吸收——这解释了为什么 −1.27 dB 能被 3,000 步拉回 −0.13 dB。

**③ 剔除的账要看"按谁计费"。** 一个反直觉的实测：**同样做盒剔除，加在 3DGS 的逐 splat 排序上反而更慢**（24.1 → 33.1 ms），加在 AbacusGS 上却是收益的一部分。原文只报数、未解释；一个可能原因是剔除后 splat 列表不再连续，排序与上传都退化成 gather（**推测，未验证**）。可用的教训是：**剔除的收益按 splat 计，剔除的成本按盒计——把它放在盒那一层才划算。**

## Limitations

1. **只在"CPU 排序"的形态里成立。** GPU radix sort 时排序只有 1.3 ms，帧 3.8 → 2.6 ms（1.46×）；4K 下 13 个场景有 4 个**比 3DGS 慢**——因为绘制占比上升，省下的排序变成小数。**"4.5×" 属于 web / 移动查看器这一类渲染器，不属于桌面 CUDA 管线。**
2. **已有资产不能直接吃。** 每个模型都要顺序感知微调（3k / 10k 步），且微调期间**盒子划分与存序冻结、densification 关闭**。所以这是"按新顺序重拟合一个模型"，不是"转一次格式"。分布成本 = 每个资产 × 3k 步。
3. **有损是结构性的。** 微调恢复的 1.14 dB 是**适应**（模型学会在错的序下也画对），不是**修正**（序还是错的）。两个误差源（盒间深度重叠、轴不平行视轴）在定义上就消不掉。
4. **"精确"的 BSP 序反而更差。** 用八叉树 BSP 遍历（对格子是**精确**可见序）时：不微调 −2.89 dB、微调后 −0.246 dB，是质心深度序的两倍。原因一句话：**splat 会跨出格面**——BSP 对**格子**精确，对**格子里的 splat** 不精确。**当"保证的对象"和"你要的对象"差一层时，精确性是负资产。**
5. **排序没有被消灭，缩小了 31×。** V ≈ N/31，帧只快 4.5×——Amdahl：绘制仍是大头。论文标题里的 "without Per-Splat Sorting" 精确说法应是 "without per-splat sorting, still with per-box sorting"。
6. **真机未测。** 浏览器结果是"降频 4× 的桌面 CPU + 桌面 GPU"，作者自述"we have not measured real phones"。
7. **splat 大于它自己的盒子时无特殊处理**——这正是 BSP 失效的同一种情况的另一面。

## Game Development Relevance

- **排序线的第七个节点，且是换轴的一个**：①–⑥ 都在问"怎么让每帧的排序更便宜/不存在"，⑦ 问的是"**排序该不该发生在帧里**"。答案是：不该——它属于资产。
- **和烘焙同构**。[[Gaussian Splatting]] 的 GI 谱系那条判据是"**每代冻结一个变得慢的变量**"（视角 → 光照 → 几何）。本文冻结的是**顺序**——顺序在帧间变得最慢（只翻转一个符号），却被每帧重算。**这条轴现在有第五格。**
- **对 VFX / 半透明的直接判据**：任何"每帧重排一遍"的列表（半透明粒子、毛发片、decal 队列、贴花排序），先问一句——**"这个排列里有多少真的每帧都变？"** 如果答案是"只有方向"，那它就该被拆成"存好的序 + 一个符号位"。
- **对预算表**：**17 B/splat** 是一个新的硬数字（对照 3DGS PLY 248 B）。它同时是"流式下载量"和"显存驻留量"，对移动端档位讨论比 FPS 更好用——因为它不依赖硬件。
- **一个反面教材**：顺序感知微调意味着"**换了表示就要重训**"。这在管线里是真实成本：一次格式升级 = 全库资产重跑。选型时要把这笔算进去，而不是只看 4.5×。

## Unreal Engine Relevance

GS 仍不是 UE 官方管线成员，所以映射只能落在"形态借鉴"上：

- **一份布局三用**（排序结构 / 压缩格式 / draw 命令组织）是 UE 资产管线值得学的形态——Nanite 的 cluster 页、VT 的页表都在做同一件事：**让一份数据同时回答"怎么找、怎么压、怎么画"**；
- **对 Niagara 半透明排序的借鉴是有限度的**：粒子每帧生成、没有稳定身份，存不了序。但**静态半透明几何（玻璃、贴花队列、植被卡片）**是有稳定身份的，可以套这个模式——按空间盒子分组、组内沿轴存序、每帧只定方向；
- **对 `gl_DrawID` 那一段的引擎等价物**：UE 的 GPU Scene / instanced stereo 里有类似的"用 draw id 反查实例"机制，`MultiDrawIndirect` 在 UE5 的 RDG 里也支持。**这条不是新能力，是新用法**。

## Technology Evolution

```text
1980  Fuchs-Kedem-Naylor BSP：树给精确可见序，"只在穿面时变"
        ↓（同样的观察，四十年后换了个对象）
1984  Porter-Duff over：算子有了，顺序没有
        ↓
2001  Everitt depth peeling：顺序外包给像素
        ↓
2010s 随机透明：顺序外包给随机数
        ↓
2013  WBOIT：顺序外包给权重（不要顺序）
        ↓
2023  3DGS：回到"每帧重排 N 个"——因为 splat 太多、over 不可交换
        ↓
2025-26 ①–⑥：排得快（TileGS/FlashGS）/ 不排（StochasticSplats/Stippling/SteadySplats）/ 换基元
        ↓
【本文】⑦ 顺序外包给【资产】：写一次（轴 + 盒内序），每帧一个符号位 ★ 入库
        —— "每帧"退化成"一次"，代价是 0.13 dB + 每个资产 3k 步重拟合
```

## Relationships

### Based On

- [[Kerbl — 3D Gaussian Splatting (2023)]]（记名）——被转换的宿主表示；`over` 算子原封不动；
- [[Zwicker — Surface Splatting (2001)]]——EWA 投影，两个渲染器都用它；
- Porter-Duff 1984 `over`；Fuchs-Kedem-Naylor 1980 BSP（作为"精确序"的对照与祖先）；
- Voxlines 2023——"voxel 内沿轴存线序、反向倒读"的最接近前身。

### Extends

- StreamingGS（DAC 2025）——同样排 voxel，但**每帧还要排 voxel 内部**；本文把那一步也搬到写时；
- Duplex-GS——同样按组，但组内用加权和（换算子）；本文组内保留 `over`。

### Related

- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]——同题的另一端：把排序**做便宜**（tile 内分箱）vs 把排序**搬走**；
- [[2026-09-29-Gaussian Stippling — Efficient Sorting-Free 3D Gaussian Rendering|Gaussian Stippling]] / [[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]]——③④ 家族：删掉顺序，用随机+神经还债；本文**不删顺序，只是不再每帧算**；
- [[Everitt — Interactive Order-Independent Transparency (2001)|Everitt 2001]]——"顺序外包给谁"这个提问方式的出处，本文是第五个答案；
- [[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)|Sloan 2002 PRT]]——同一条方法论：**冻结变得最慢的那个变量**（PRT 冻几何、本文冻顺序）。

### Followed By

待观察三件事：① 4D / 动态 GS 上盒子会不会每帧变形（那时"盒内序"还稳定吗）；② 真机（不是降频桌面）的实测；③ 盒内"splat 跨出盒面"的处理——目前无特殊处理，而 BSP 的失败正是这一条。

## Personal Knowledge State

Current Level: **Normal**（GS 基础已 Easy，本篇不复述；三处是新的）

1. **"每帧重算"是最贵的默认值。** 判据：当输入每帧变时，问**"输出真的每帧都变吗？变的是全部还是其中一点点？"** 本文的答案是"31 个 splat 里只有 1 bit 在变"。这是可直接搬到任何逐帧排序/重建流程上的一句话；
2. **"精确"可能作用在错的对象上**（BSP 对格子精确、对 splat 不精确 → 更差）。这条与你库里已有的"归一化指标只能比算法不能做预算"同族：**度量/保证的粒度错一层，结论就反号**；
3. **一份数据回答三个问题**（排序 + 压缩 + draw 命令）——是资产格式设计的正面样本。

## Learning Value

- **对分档工作**：17 B/splat 是**不依赖硬件**的档位数字。五档画质里 Android_Low 的 GS 预算讨论，用"每 splat 多少字节"比用 FPS 更稳；
- **对"取消式优化"这条线**：本篇补的是**换轴**样本——六篇都在同一个平面内优化（更快的排序、没有排序），第七篇换了一个平面（顺序放在哪）。**当一条线上连续六个方案都在做同一件事时，下一个有效动作通常是换轴**；
- **对管线成本**：顺序感知微调把"格式升级"变成"全库重训"，这是真实的分布成本，值得在选型时单独列一行。

## Visualization

三张原图已嵌（图 1 机制全景 / 图 2 三阶段）。**没有做 HTML**：机制是"盒子 + 一个符号位"，图 2 已经把它画完了；排序线七节点的全景是 Markdown 表格能表达的（在 [[Gaussian Splatting]] 排序线表里）。按 §25 自查三问，三条都不满足。

## Notes

- 窗口：**提交 2026-11-01，公告在 `Tue, 3 Nov 2026` 段**（19 条，含大量 cross-list）；
- 机构：University of Memphis（Cong Guo 组）× NUS；LLM Usage Statement 明写"用了 Claude 做代码、实验、文献检索、图与文字，作者核验全部结果"——**这条声明本身值得记一笔**，它是 2026 年论文里越来越常见的格式；
- 质量口径：13 场景（Mip-NeRF 360 / Tanks&Temples / Deep Blending），每 8 张留 1 张测试；基础模型是官方 3DGS 30k 步（1.07–4.88 M splat）；对照组是**同基础模型、同步数、用逐 splat 深度序**继续训练——这个对照设计是本文最扎实的部分；
- 指标口径警告：作者自述他们的 LPIPS 实现报的数比官方脚本高，所以**只在本文管线内做纵向比较**；
- 遗留在 `.workbuddy/tmp/` 的裁剪脚本与页面渲染不进库。
