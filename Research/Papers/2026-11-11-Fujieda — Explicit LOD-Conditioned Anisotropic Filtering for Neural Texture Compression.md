---
type: paper
title: "Explicit LOD-Conditioned Anisotropic Filtering for Neural Texture Compression"
authors: [Shin Fujieda, Takahiro Harada]
year: 2026
published: "2026-11-09（arXiv v1）"
venue: "arXiv preprint (cs.GR)"
url: "https://arxiv.org/abs/2611.12650"
code: ""
project_page: ""
category: [texture-compression, neural-texture-compression, texture-filtering, anisotropic-filtering, lod, real-time-rendering]
importance: A-
historical_importance: 2
game_relevance: 5
production_readiness: Prototype
user_level: Normal
status: unread
aliases: [AF-NTC, Anisotropic Filtering NTC, 神经纹理压缩的各向异性过滤, Explicit LOD-Conditioned AF]
tags: [texture-compression, texture-filtering, anisotropic-filtering, lod, neural-rendering, amd]
---

# AF-NTC（Fujieda & Harada 2026）

## TL;DR

**一个标量装不下一个椭圆。**

神经纹理压缩（NTC）的主流设计是"特征金字塔 + 小 MLP 解码器"。它有一个结构性缺陷：**它拿不到 GPU 纹理单元的过滤**——解码器只看到 UV，看不到"这个像素在源图上覆盖了多大一块、什么形状"。业内做法是靠随机纹理过滤（STF）用噪声换抗锯齿，或者干脆不管。

这篇的诊断是全文最值钱的一句：

> 只按"长短轴比"（stretch）设过滤是不够的，因为**单个输入无法同时携带脚印的形状和它的大小**——于是近处和远处糊得一模一样。

修法是把解码器的条件换成**硬件采样器本来就在用的三个有界量**：LOD（次轴尺度）、stretch（长短比）、rot（主轴方向）。三者恰好是椭圆的三自由度，不多不少。

结果：**16.75 MB（BC7 mip 链的 25%）**，在 10 组材质-场景上比"无 AF 的三线性 BC7"高 **+1.31 ~ +7.63 dB**，比 4× 大的"各向异性 BC7"低 **0.03 ~ 3.37 dB（均值 1.57）**。

代价：**全文零运行时数字**。30 个 tap 的软件模拟，没有一次 GPU 采样器上的实测。

## Problem

NTC 要跟 BC7 比"每字节的画质"。但纹理在游戏里不是"贴上去看一次"，是**在缩小和掠射角下被采样几十亿次**。所以 NTC 的评测必须同时回答两件事：**压缩率**，和**过滤质量**。

两族 NTC 对过滤的态度正好相反：

| 族 | 做法 | 过滤从哪来 | 代表 |
|---|---|---|---|
| **latent-in-BC** | 把隐特征塞进 BC 纹理，让硬件采样器过滤隐特征，再小 MLP 解码 | **硬件免费给**（作者叫它 *free AF*） | Belcour & Benyoub 2025（Cooperative Vectors）、Weinreich 2024 |
| **feature-pyramid** | 多分辨率特征网格 + MLP 解码 | **没有**。只看到 UV，一个像素一次解码 ⇒ 必须靠 STF 用噪声换 | Vaidyanathan 2023（NTC）、Fujieda & Harada 2024（NTBC） |

这篇打的是第二族。缺口的形状很清楚：**第一族之所以"免费"，是因为采样器交给解码器的是"已经按真实脚印积过的特征"；第二族只交 UV，于是脚印必须变成显式输入。**

## Historical Context

这条线是 43 年前那句话的延迟回声。

```text
1983  Williams：金字塔的每一层是【正方形盒式滤波】，D 是一个标量
        · 作者自陈："mip 最严重的妥协是它是对称的"
        · 一个 D 同时要回答"多大"和"多细" ⇒ 脚印不圆时只能二选一
  ↓
1989  Heckbert：脚印 = screen→UV 雅可比的椭圆（EWA 的几何）
        · 但"怎么用这个椭圆"仍是一个标量 D = max(两方向导数)
  ↓
1999  McCormack et al.（Feline）：沿主轴多点取样近似椭圆
        ⇒ 硬件各向异性过滤的形状：N 个三线性 tap 沿主轴排一串
  ↓
2023  Vaidyanathan et al.：NTC 把过滤整个拿掉，用 STF 换
  ↓
2025  Belcour & Benyoub：把隐特征放回 BC，过滤又免费了（但另一族）
  ↓
2026 ★ 本篇：在【没有硬件过滤】的解码器里，把椭圆的三个自由度显式喂进去
```

**注意这条线里"变"和"没变"的分界**：从 1983 到 2026，**脚印的几何一点没变**（还是那个雅可比的椭圆），变的只是"谁来积它"——先是预过滤的层，再是硬件的 tap 串，现在是解码器。**所以这篇的输入设计不是新发明，是把 1983 年就存在、1999 年被硬件固化的那三个量，第一次接到一个可微解码器上。**

与 [[2026-11-05-Fujieda — Patch-Based Generalized Neural Texture Upscaling|Fujieda 2026 纹理上采样]] 的关系是同一组的连续两步：那一篇把"层存不存在"交给采样时刻（取消金字塔），这一篇把"层怎么被读"交给解码器（金字塔还在，读法变了）。**两篇都零运行时数据**——这条后面要单独记一笔。

## Core Idea

**把"过滤"从后处理搬进解码器的条件里，而条件的参数化必须与物理量同构。**

三步：

1. **诊断**：ratio-only 的失败不是因为网络不够大，是因为 $\text{length}=\text{stretch}\cdot k$ 里 $k$ 是一个常数——**它只能在一个距离上正确**。"近处小但强拉长的脚印"（近处掠射）这个组合，ratio-only 表达不出来。
2. **参数化**：用硬件 AF 的三元组 $(\text{rot},\text{stretch},\text{LOD})$，三者都是**有界**的（$[0,\pi)$、$[1,16]$、$[0,12]$）。有界很重要——它让采样空间可控，也让网络不用外推。
3. **双轴**：椭圆有两条轴，用两种不同的手段处理——**主轴用空间 tap 线**（这条轴上信息是高频的，值得多采），**次轴用 mip 层间混合**（这条轴上信息已经被金字塔预过滤掉了，混合就够）。这是全文最省的一处设计：**一条轴花钱采样，另一条轴白拿金字塔。**

## Technical Approach

### 脚印三元组：从雅可比到三个有界标量

$J$ = screen→UV 的雅可比，$A=JJ^\top$，特征值 $\sigma^2_{\max}\ge\sigma^2_{\min}$：

$$\text{LOD}=\log_2(\sigma_{\min}\cdot\text{tex\_res})\in[0,12]\quad(\text{4K 时})$$
$$\text{stretch}=\sigma_{\max}/\sigma_{\min}\in[1,16],\qquad \text{rot}=\angle(\text{主轴})\in[0,\pi)$$

**一处容易漏的实现约定**：$\sigma_{\min}$ 不单独传，而是从 LOD 反解

$$\sigma_{\min}=2^{\text{LOD}}/\text{tex\_res}$$

**同一个反解值同时用于两处**——放 tap（式 6）和造训练目标（§3.4）。这是"**一个量两种读法**"的又一例（对照 11-08 DC-SQEM 的 SVD 秩亏：同一个数既是尺度又是索引）。如果两处各算一遍，训练目标和推理就会差半个 texel，而这种偏差不会报错，只会让 PSNR 少一点点。

### 双轴滤波：tap 线 + 层混合

- **主轴**：tap 线长度 $=\sigma_{\min}\cdot\text{stretch}=\sigma_{\max}$（**绝对长度**，不是比例）。四个层各一份预算 $[16,8,4,2]$，共 **30 tap**。作者的理由很直白：**对齐硬件的 tap 数、压低解码成本，不是为了峰值质量**。
- **次轴**：四个层各有一个 LOD 中心 $c_l=\log_2(\text{tex\_res}/\text{res}_l)$（本例 = 2,3,4,5），每层特征按 $c_l$ 处的高斯（$\sigma=0.7$）加权后拼接，**保持 48 维宽度不变**。
- **只有 LOD 进 MLP**：$48+1=49$ 维输入。rot 和 stretch 只通过 tap 几何起作用。附录 B 加了 18 维位置编码把二者也喂给 MLP，**只差 0.05 dB**——所以砍掉。

> **这条消融值得单独记**：把 (rot, stretch) 当"特征"喂给网络没用，把它们当"几何"用才有用。**方向与比例是作用在采样位置上的量，不是作用在特征上的量。**

### 架构与预算

四层 2D 特征网格 1024/512/256/128，每层 12 维 ⇒ 48 维；MLP $49\to192\to192\to9$（albedo 3 / ARM 3 / normal 3），sigmoid 输出；int8 QAT。

**16.75 MB vs 67 MB**（同三张 4K 图的完整 BC7 mip 链）= **25%**。

### 训练目标本身是一个近似

训练目标不是精确 EWA 积分，是**模拟硬件 AF**：沿主轴放 $\lceil\text{stretch}\rceil$ 个 tap，长度 $(2^{\text{LOD}}/\text{tex\_res})\cdot\text{stretch}$，每个 tap 在 mip=LOD 上三线性取样。作者自己标了它的成色：**与 supersampled EWA 参考相差 37–41 dB**。

采样策略也值得抄：$(\text{rot},\text{stretch},\text{LOD})$ **三个量独立采样**，LOD 按面积比例 $p(\text{LOD})\propto4^{-\text{LOD}}$。理由：独立采样让解码器看到"任何单一视角都给不出的组合"——**这等于把"数据增广"做在了物理参数空间里，而不是图像空间里。**

## Key Contribution

1. **一个诊断**：ratio-only 失败的根因是"一个输入无法同时携带形状与大小"，而不是容量不足。
2. **一个最小修复**：$(\text{rot},\text{stretch},\text{LOD})$ 三元组 + 双轴滤波，把 AF 变成解码器的显式操作。
3. **一次干净的误差分账**：模型放大（2× 特征 / ~4× 参数）只换来 ≤0.4 dB ⇒ **在这个预算下，剩下的误差是压缩造成的，不是模型容量造成的**。作者那句总结很锋利：更大的模型**无法把过滤已经平均掉的东西找回来**。

## Why It Works

### 因为椭圆恰好三个自由度，而硬件那三个量是它的完备参数化

不是"三个量效果更好"，是"**三个量刚好够、两个量必然不够**"：

| 输入 | 能表达 | 不能表达 |
|---|---|---|
| 无（free-AF） | 各向同性 | 任何拉长 |
| stretch 一个标量 | 形状 | **大小**（⇒ 近处恒糊） |
| (LOD, stretch) | 形状 + 大小（轴对齐） | **方向** |
| (LOD, stretch, rot) | 全部 | — |

**这条推理可以直接搬到别的"要喂几何量给网络"的场合**：先数清楚这个几何对象有几个自由度，再决定喂几个数。喂少了不是"学得慢"，是**结构上不可能**。

### 因为两条轴的性质不同，所以用两种手段

主轴上的信息是**高频的**（沿主轴方向纹理变化快），必须真采样；次轴上的信息在金字塔里**已经被预过滤**，混合层就等价于低通。所以：

- 去掉层混合 ⇒ **−5.3 ~ −9.3 dB**（最大的单项损失，且损失集中在近处小脚印）；
- 只留 tap 线 ⇒ 相当于"沿一条线的超采样"，次轴完全没滤波。

**"哪条轴该花采样、哪条轴该花预过滤"是这篇真正的工程判断**，比三元组本身更难想到。它的一般形式是：**一个各向异性核，可以拆成"沿长轴的空间积分 + 沿短轴的频率截断"**，前者贵、后者便宜，而便宜的那一半恰好是 mip 一直在做的事。

### 因为 LOD 是"索引"也是"尺度"

经典 mip 里 $D$ 只用来选层；这里 LOD 同时决定 tap 的绝对间距（经 $\sigma_{\min}$ 反解）和层权重中心。**同一个数的两种读法让"几何"和"存储"保持同步**——这也是为什么它能做到"只在 25% 体积下不糊"：它没有额外存任何东西，只是把已有的层级结构按脚印重新读了一遍。

## Limitations

### 四处冷水（数据自己说的）

1. **全文没有任何运行时数字。** 原话：*"We report no runtime evaluation: our AF is a PyTorch software emulation, not an optimised GPU implementation on a texture sampler."* 30 tap/像素的解码成本 vs 硬件 16× AF 的 16 tap——**这个比较是全文的核心卖点，也是唯一没被测量的一项**。而且这是同一组（Fujieda & Harada）连续第二篇零运行时（11-05 的上采样那篇同样如此）。**两篇合起来形成一个模式：算法讲清楚了，硬件账没算。**
2. **它从未赢过各向异性 BC7。** 全 10 行都低，均值 1.57 dB，最差 3.37 dB（pavement plane）。作者自己划了范围：*"We claim correct filtering at a fixed compression budget, not a general size–quality win, and we do not claim to match BC7."* **它的卖点是"1/4 的体积做到接近"，不是"更好"。**
3. **等字节比较只在"脚印大"时成立。** 两个网格资产上 AF-NTC 与 2K aniso-BC7 打平（cat +0.28 / barrel −0.07），但作者明说那里的脚印 LOD≈1.4–2.4、细节早被 AF 抹掉了；**在表 1 的细脚印上，2K 链反而好 0.4–3.3 dB**。这条自我限定很诚实，也说明"25% 体积"这个数字**依赖于视角分布**。
4. **诊断在图里，消融表里没有它。** "Naive (ratio-only)" 只出现在图 1，表 1 的消融是从 iso（完全无脚印输入）直接跳到 +aniso（有各向异性真值、无 LOD）再到 AF-NTC。**全文最核心的那个失败案例（ratio-only 近处恒糊）没有被量化成一行数字。** 图能说明方向，但说不清"ratio-only 到底差多少"。

### 论文自陈的边界

- 一个材质训一个模型（per-material），训练与评测共用同一套 4K 贴图。
- 只做 albedo / ARM / normal 三类贴图，没有做 displacement、没有做 mip 之外的各向异性反射。
- 没有报告种子方差，但附录说每个数字是多次训练的均值——**0.03 dB 这种差距（stone_floor sphere vs aniso-BC7）落在未报告的方差里，不该被当成结论**。

### 我自己的两处待核

1. **30 tap 的真实成本**：硬件 16× AF 是 16 次三线性取样（约 128 次 texel 读），AF-NTC 是 30 次网格取样 + 一次 49→192→192→9 的 MLP。**"对齐 tap 数"不等于"对齐成本"**——MLP 的 2×192 矩阵乘在每像素都要跑一遍，这笔账只有实测能给。
2. **训练目标 37–41 dB 的天花板**：模型学的是一个近似器，评测却对精确参考。**报告的 PSNR 里有多少是"学不像目标"贡献的？** 论文没有把"目标自身的误差"和"模型的误差"分开。

## Game Development Relevance

- **纹理内存是游戏里最大的一笔静态预算**（一个 3A 关卡常常 4–8 GB 贴图）。NTC 的 4× 压缩如果能上生产，换来的不是画质而是**驻留量**——直接决定同屏能放多少 unique 材质。
- **各向异性过滤等级是玩家可见的画质选项**（2×/4×/8×/16×），成本在带宽不在算力。这篇说明：**如果纹理是解码出来的，AF 就不再是"采样器的一个开关"，而是"解码器必须内建的能力"**——这是一个引擎架构层面的变化。
- **判断"这条路值不值"的一个实用尺子**：它赢的是"等体积下的近景清晰度"，输的是"绝对画质"。对**近景主导**的游戏（室内、角色特写）比对**远景主导**的（开放世界飞行）更有意义——因为后者脚印大，2K 各向异性 BC7 就够了。

## Unreal Engine Relevance

- UE 的纹理栈（虚拟纹理 / mip 生成 / `r.MaxAnisotropy`）默认假设"采样器负责过滤"。NTC 类方案要落地，必须替换的不只是采样函数，还有 **mip 生成器**（金字塔要按解码器的语义重建）与 **streaming 的页粒度**（16.75 MB 是整个材质，不是一页）。
- 真要做，最小可跑的形状是：一个自定义 shader 里的解码器 + 一张"脚印三元组"常量 buffer（可由 `ddx/ddy` 在 shader 里现算，正是 §3.2 的 $J$）。**难点不在解码，在于把 30 个 tap 塞进一个像素着色器的指令预算里。**

## Technology Evolution

```text
1976  Blinn & Newell：纹理 + 反射映射（脚印问题首次出现）
  ↓
1983  Williams：金字塔 + 三线性 = "预过滤多个尺度" 的范式
  ↓
1989  Heckbert：脚印的椭圆几何（EWA）
  ↓
1999  McCormack（Feline）：沿主轴的 tap 串 ⇒ 硬件 AF 的形状
  ↓
2000s 硬件固化：2×/4×/8×/16× AF 成为 API 里的一个整数
  ↓
2023  NTC（Vaidyanathan）：把过滤整个移出硬件，用 STF 顶
  ↓
2025  Cooperative Vectors：把隐特征放回 BC，AF 重新免费
  ↓
2026 ★ 本篇：在无硬件过滤的解码器里把三元组显式化
```

## Relationships

### Based On

- [[Williams 1983 — Pyramidal Parametrics|Williams 1983]] —— 金字塔 + 层间插值的基本形状；本篇的层混合就是它的"inter-level interpolation"，只是权重从线性换成高斯。
- Vaidyanathan et al. 2023（NTC）—— 特征金字塔 + 小 MLP 的架构骨架（本库暂无笔记）。
- Heckbert 1989（EWA）/ McCormack 1999（Feline）—— 脚印椭圆与"沿主轴 tap 串"的两个来源。

### Extends

- [[2026-11-05-Fujieda — Patch-Based Generalized Neural Texture Upscaling|Fujieda 2026 纹理上采样]] —— 同一组的前一步（取消层级）；本篇把层级留下、改读法。
- [[Texture Filtering]] —— 本页把"过滤"从后处理搬进解码器条件，是这一概念在神经表示下的当前形态。

### Related

- [[Level of Detail]] —— **纹理侧的 LOD 分支**：本页的 LOD 是"输入量"而非"选择量"，这是与几何 LOD 最大的差别（几何侧的误差要投影估计，纹理侧的脚印是解析算出来的）。
- [[Decoupling Geometry and Appearance]] —— 纹理被压进解码器之后，"分辨率"不再是资产属性而是解码器属性。
- [[Gaussian Splatting]] —— 同样是"表示的采样率与消费点解耦"，但那边解耦的是几何。

### Contrasts

- **STF（Pharr 2024）**：一次解码 + 随机 tap，用**噪声**换抗锯齿，靠时域后处理收；本篇是**确定性**的，一次解码出无噪值，代价是更多 tap。**两条路的分界是"你愿不愿意把成本放到时域去摊"。**
- **latent-in-BC（free AF）**：过滤免费但隐特征必须住在 BC 里（体积有下限）；本篇自由但过滤要自己实现。**"免费"的代价是"形状被 BC 的块结构钉死"。**

## Personal Knowledge State

- **机制层 Easy**：mip、三线性、各向异性过滤是什么，日常在用。
- **判据层 Normal（今天新增的两条）**：
  1. **一个标量装不下一个椭圆**——这条在 1983 年就被作者自己写下来了（"最严重的妥协是它是对称的"），但直到 2026 年才在 NTC 里被正面处理。**凡是"用一个数描述一个形状"的地方，都该问一句：它是尺寸还是形状，还是两个都要？**
  2. **一个各向异性核 = 沿长轴的空间积分 + 沿短轴的频率截断**。前者贵、后者便宜；便宜的这半正好是 mip 在干的事。**这就是为什么"mip + 沿主轴几个 tap"能撑起整个硬件 AF 产业。**

## Learning Value

**最高。** 它是本库 [[Texture Filtering]] 一页的"当代形态"锚点，也是 [[Level of Detail]] 纹理分支的第一篇当代论文。更实际的是：**它的诊断句式可以直接搬去审自己的系统**——"我这个方案里，哪一个标量其实承担了两个独立自由度？"

## Visualization

**图 1｜四种做法的近景/远景对照（这篇的论点全在这一张图里）。** 四列从左到右：ratio-only / 三线性 BC7（无 AF）/ AF-NTC / 超采样参考；上排全景、下排放大远景部分。

![[Research/Files/Pictures/Fujieda 2026 — 图1 AF-NTC 四方法近景远景对照.png]]

**读图**：看**第一列**（ratio-only）——近处的地砖纹路整片糊掉，而且**四个距离上糊的程度一样**（这正是"一个输入装不下形状+大小"的可视化）。看**第二列**（三线性 BC7，体积是本篇的 4 倍）——近处清楚、**远处糊成一片**。看**第三列**（AF-NTC）——近处锐、远处糊，两个方向都对。**这张图的说服力在于：错误的方向不同，而只有第三列两个方向都对。**（出处：Fujieda & Harada 2026，图 1）

**图 2｜双轴滤波的管线形状。** 左侧是脚印三元组如何同时驱动"四个层上的 tap 几何"与"层间的 LOD 加权混合"，右侧是 49 维输入进 MLP 再出 9 通道。

![[Research/Files/Pictures/Fujieda 2026 — 图2 AF-NTC 双轴滤波管线.png]]

**读图**：注意图里有**两条并行的路径**——下面那条（rot/stretch/LOD → tap 位置）是**几何**路径，上面那条（LOD → 高斯权重 → 拼接）是**混合**路径，只有 LOD 分流到 MLP。**"哪个量进几何、哪个量进特征"在这张图上是可读的**，而附录 B 证明这个分配是对的（把 rot/stretch 也喂 MLP 只差 0.05 dB）。（出处：Fujieda & Harada 2026，图 2）

## Notes

- 6 页，2 图 4 表，无附录图表之外的补充材料。AMD（东京 / 圣克拉拉），作者是 NTBC（2024）与纹理上采样（2026-11-05）的同一组。
- **写作上值得学的一句**：`a single input cannot carry both the shape of the footprint and its size` ——14 个词把整篇的问题定义完了，且不含任何术语。**好的问题陈述是"不可能性陈述"，不是"挑战性陈述"。**
- 待核：30 tap 的真实 GPU 成本（见 Limitations 第 1 条）、训练目标的误差是否该从报告里扣掉。
