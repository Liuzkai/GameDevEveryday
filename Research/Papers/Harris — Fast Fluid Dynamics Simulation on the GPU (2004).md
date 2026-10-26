---
type: paper
title: "Chapter 38: Fast Fluid Dynamics Simulation on the GPU"
authors: ["Mark J. Harris"]
year: 2004
published: 2004
venue: "GPU Gems 1（Addison-Wesley，第 38 章；非同行评审书章）"
url: "https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-38-fast-fluid-dynamics-simulation-gpu"
doi: ""
code: "随书 CD（GPU Gems 1 配套源码）"
project_page: ""
category: ["fluid-simulation", "gpgpu", "real-time", "graphics-hardware", "game-engine"]
importance: A
historical_importance: 4
game_relevance: 5
production_readiness: "Industry Adopted（思想层：纹理当数组、双缓冲反馈、pass 序列 = 一个求解器——此后所有 GPU 模拟都沿用这套骨架；原始实现已被 compute shader 取代）"
user_level: Normal
status: unread
---

# Harris — Fast Fluid Dynamics Simulation on the GPU (2004)

## TL;DR

**把 [[Stam — Stable Fluids (1999)|Stam 1999]] 的求解器整体搬进像素管线的那一章**——它教的不是新流体算法，而是**"如何用一个只能「写自己位置」的渲染器去跑一个需要「读任意位置、写任意位置」的迭代求解器"**。

三个翻译是全篇的骨架：

```text
数组        → 纹理（RGBA 四个通道 = 四个标量）
下标        → 纹理坐标
for 循环    → 一个片元程序（对每个片元同时执行 = SIMD）
迭代反馈    → 纹理更新（渲染到纹理 / 拷贝到纹理）+ 双缓冲交换
```

于是**一次时间步 = 一串 slabop（全屏 pass 序列）**：对流 → 扩散 → 加外力 → 解压力（40~80 次 Jacobi）→ 减压力梯度。论文自报：**GeForce FX 上比同算法 CPU 实现快最多 6×**；**128×128 的云模拟在 FX 5950 Ultra 上 >80 次迭代/秒**。

> 一句话对读者：**这一章的价值不在"流体"，在"GPU 的编程模型"**——它把"为什么 GPU 上的模拟必须是 pass 序列、必须有双缓冲、分辨率上不去"讲清楚了，而这些约束今天仍然以 compute shader 的形式活着（见 [[GPGPU]]）。

![[Research/Files/Pictures/Harris 2004 — 图1 染料被湍流卷曲的两种配色.png]]
**读图**：同一套 2D 求解器、两种染料配色。画面里的"墨"是被速度场搬着走的被动标量（dye），**右上小窗是真正在算的东西**——一张几百像素见方的低分辨率场。**视觉上的丰富度与计算量几乎无关**：分辨率决定成本，"好看"来自配色与卷曲结构。（图 1，Harris 2004）

## Problem

2004 年的可编程管线只有**顶点着色器 + 片元着色器**两个可编程阶段，且都绑定在"画三角形"这件事上。要用它做通用计算，必须先回答四个"没有"：

1. **没有数组**——显存里没有"可随机寻址的缓冲区"，只有纹理；
2. **没有循环**——着色器不知道"网格有多大"，它只知道"我这一片元在哪个坐标"；
3. **没有读-写**——片元程序能读纹理，但**输出只能写到光栅化指定的那个位置**，不能写回自己读的那张图；
4. **没有共享内存/原子操作**——片元之间不能通信（这条论文没明说，但整套 slab 设计就是它的后果）。

流体求解恰好把这四条全踩了一遍：它是**网格上的迭代求解器**——读邻居、写自己、拿上一轮的结果当下一轮的输入。所以这一章实质是**一份"如何在缺这四样东西的机器上跑迭代求解器"的工程报告**。

## Historical Context

```text
1999  ★ Stam《Stable Fluids》：无条件稳定的四步分解（对流/扩散/外力/投影）——CPU 上的答案
2001  NVIDIA GeForce 3：第一代可编程顶点/片元着色器（DX8）——"能编程"但只能画图
2001  Larsen & McAllister：矩阵乘法跑在图形硬件上——GPGPU 这个词的史前史
2002  "GPGPU" 社区成形（Mark Harris 等人办的 GPGPU 课程/邮件列表）
2003  Bolz 等《Sparse Matrix Solvers on the GPU》：共轭梯度 + 多重网格搬上 GPU（本文引用）
2003  Krüger & Westermann：线性代数算子（本文引用）／Goodnight 等：多重网格边界值问题（本文引用）
2003  Harris 等《Simulation of Cloud Dynamics on Graphics Hardware》（本文的云模拟前作）
  ↓
★ 2004  本篇：把 Stam 的四步完整搬上 GPU，并写成"翻译手册"（GPU Gems 1 第 38 章）
  ↓
2006  NVIDIA CUDA 公布（2007 发布）：GPGPU 从"借渲染管线"变成"正规编程模型"
2009  DirectX 11 Compute Shader / OpenCL 1.0：scatter、共享内存、原子操作、组内同步全部到位
2014+ 异步计算队列（主机与计算并行）成为主机/PC 的常规预算手段
2020s+ 计算着色器成为模拟、剔除、GPU 驱动管线的默认执行环境（[[GPU-Driven Rendering]]）
```

**这一章的历史位置**：它是**"借"时代的教科书**——在 CUDA 出现前两年，把"用渲染管线做通用计算"的全部技巧（打包、双缓冲、slab、边界图元）写成了一份可照抄的说明书。它同时也是**最后一代必须这样写代码的人**的教材：两年后，同样的算法用 CUDA 写只需要一个 `__global__` 函数。

## Previous Work

- **[[Stam — Stable Fluids (1999)]]**：算法来源。半拉格朗日对流（回溯 + 插值）+ 隐式扩散 + 投影，无条件稳定；本文**原样继承四步分解**，只换执行单元。
- **Fedkiw, Stam & Jensen 2001（Visual Simulation of Smoke）**：加了**涡量约束**（把数值耗散掉的小涡补回来）与交错网格。本文把涡量约束作为"扩展"给出（38.5.1），并明确说明自己用的是**同位网格**（cell-centered）。
- **Bolz / Krüger / Goodnight 2003**：把稀疏线性求解器搬上 GPU 的三篇同代工作——它们证明了"GPU 能解 PDE"，本文证明的是"**GPU 能解一个完整的、每帧都要重算的时变系统**"。
- **Stam 2003（GDC，Real-Time Fluid Dynamics for Games）**：游戏向的实时版；本文的贡献不是"更快"，是"**在 GPU 上**"。

## Core Idea

**把"数据并行"这件事从"数据结构"改写成"渲染"**。一句更狠的概括：

> **在 2004 年的 GPU 上，你唯一能做的写操作是"画一个覆盖某块像素的图元"。所以算法必须被重写成"一串覆盖全屏的绘制"。**

由此推出来的三个设计决定，构成了此后十五年 GPU 模拟的标准骨架：

| CPU 写法 | GPU 写法（2004） | 代价 |
|---|---|---|
| `for each cell: read neighbors, write cell` | 画一个覆盖全屏的 quad，片元程序读 4 邻居、写自己 | 每步一次全屏 pass；**无法原地更新** |
| `u = f(u)` 原地赋值 | 读纹理 A → 写纹理 B → 交换 A/B（ping-pong） | **显存 ×2**；每步额外一次交换（RTT 时只是换 ID，近乎免费） |
| 边界是一个 `if` | 内部画 quad、边界画 4 条线，用**两个不同的片元程序** | 每个 slabop 变成"两个 draw call" |

## Technical Approach

### 1) 状态表示：场 = 纹理

至少两张纹理：**速度**（RG = u,v，或 RGB 三分量）与**压力**（单通道）。可视化再加一张"染料"（dye）当被动标量。图 2 是这张网格的原始样子——**每个格子一个速度矢量，格子中心采样**（同位网格）。

![[Research/Files/Pictures/Harris 2004 — 图2 速度场网格即纹理.png]]
**读图**：$M\times N$ 个格子，每格一个 $(u,v)$；格子间距 $\delta x,\delta y$。整张图就是一张 $M\times N$ 的纹理，`tex2D` 的坐标就是格子下标——**"数组下标 → 纹理坐标"这条翻译的全部内容**。（图 2，Harris 2004）

### 2) 对流：回溯 + 双线性（图 3）

```hlsl
pos  = coords - dt * rdx * tex2D(u, coords);   // 沿速度场"倒着走"
xNew = bilerp(x, pos);                          // 在四个邻居间双线性插值
```

**这一步是整个 GPU 化的关键**：半拉格朗日对流**天然是 gather 而不是 scatter**——"我这个格子该取谁的值"是一个**读**问题，正好落在片元程序的能力范围内。如果用前向欧拉（"我这个格子的值该送给谁"），就是 scatter，2004 年的硬件根本写不出来。

> **判据（可迁移）**：把算法改写成 GPU 友好，第一步永远是**把 scatter 改写成 gather**。这条判据从 2004 年的流体一直活到今天的 GS 光栅化与粒子排序。

![[Research/Files/Pictures/Harris 2004 — 图3 半拉格朗日回溯与双线性采样.png]]
**读图**：黄圈是当前格子（双圈标记），箭头是速度 $u(x,t)$；沿 $-u\,\delta t$ 回退到位置 $X$，再用周围四个黄点的值双线性插值。**一次 pass 里完成"搬运"，不需要知道谁搬给了我**。（图 3，Harris 2004）

### 3) 投影：40~80 次 Jacobi

压力泊松方程用最笨的迭代解：$\nabla^2 p = \nabla\cdot u$ 离散成五点差分，Jacobi 迭代

$$p_{i,j}^{k+1}=\frac{p_{i-1,j}^k+p_{i+1,j}^k+p_{i,j-1}^k+p_{i,j+1}^k-\Delta x^2\,(\nabla\cdot u)_{i,j}}{4}$$

**这就是 40~80 个连续的全屏 pass**——也是整个求解器里最贵的一段。论文给的经验值很直白：**低于 20 次精度就明显不够**。收敛慢是 Jacobi 的固有性质（谱半径趋近 1），论文没有回避，只是说"便宜到可以多跑"。

> **成本结构从此定型**：`每帧 pass 数 ≈ 5 + N_jacobi`，$N_{jacobi}\in[40,80]$。**压力求解占总 pass 数的 90% 以上**——这就是后来所有人去做多重网格（Goodnight 2003）、共轭梯度（Bolz 2003）、以及今天直接上 compute + 更快的求解器的原因。

### 4) 边界：一个 quad + 四条线（图 5）

网格预留**一圈格子**存边界条件（速度无滑移、压力纯 Neumann）。每个 slabop 因此变成两次绘制：内部画一个**收缩一像素的 quad**，四条边画**四条线图元**，用不同的片元程序。

![[Research/Files/Pictures/Harris 2004 — 图5 内部四边形边界线段.png]]
**读图**：蓝区 = 内部（一个 quad 覆盖），黄边 = 一圈边界（四条 line primitive 覆盖），小方块 = 像素位置。**"一个求解步骤 = 两个 draw call"**——这个模式在 compute shader 里被 `if (inside) ... else ...` 直接取代。（图 5，Harris 2004）

### 5) 反馈：RTT 还是 CTT

这是全文最"工程"的一段，也是最能说明时代局限的一段：

- **RTT（render to texture）**：把纹理当 framebuffer 直接渲染进去。**必须两张纹理轮换**（同一张既读又写是未定义行为），交换只是换 ID，**成本恒定**；
- **CTT（copy to texture）**：渲染到 framebuffer，再拷进纹理。**只要一张纹理**，但**拷贝成本 ∝ 纹理尺寸**。

论文说两者"功能等价"，但把成本差异写得清清楚楚。**十五年后 compute shader 的"双缓冲 + 屏障"仍是同一件事**，只是不再需要伪装成绘制。

## Key Contribution

1. **一份完整的"CPU 算法 → GPU pass 序列"翻译手册**：数组/下标/循环/反馈四个类比，至今仍是理解 GPU 编程模型最干净的入口；
2. **证明了时变系统可以在 GPU 上每帧重算**（不是离线烘焙）——这是"实时模拟"与"实时渲染"合流的起点；
3. **把成本结构写成可数的东西**：pass 数、纹理数、迭代次数。**这一章的预算语言是"pass"，不是"毫秒"**。

## Why It Works

**因为流体求解的并行粒度恰好是"一个格子"，而 GPU 的最小并行粒度恰好是"一个片元"。** 两者都是"同一段代码作用在大量独立元素上"，且**元素之间的通信只发生在相邻格**——邻居读取用纹理采样天然完成，不需要共享内存。

**它为什么只能做到这个规模**：一旦需要"任意位置写"（粒子散射、排序、图遍历），gather 模型就撑不住了。这不是实现问题，是**模型**问题——所以 2009 年的 compute shader 才是一次真正的分水岭，而不是一次性能优化。

## Limitations

**论文自述的**：

- **只有 2D**（3D 需要 6 邻居 + 平铺纹理，见 38.5.2）；
- **没有自由表面**（不能做水面/水花，只能做"盒子里的连续介质"）；
- **边界只能是矩形盒**（任意障碍物要每格判断方向，更慢更复杂）；
- **同位网格**（精度与振荡不如交错网格）。

**我补的三条（读者应知道）**：

- **精度天花板**：fp16 纹理在长迭代（40~80 次 Jacobi）下会累积误差；2004 年的硬件上 fp32 渲染目标既慢又常不支持，这是"分辨率上不去"的隐形原因之一；
- **没有原子/散射**：粒子类现象（烟灰、飞溅、泡沫）无法在这一套里表达——它们要等到 compute shader 或 transform feedback；
- **性能口径含糊**：**"6× 快于 CPU"没有给 CPU 基线、没有给网格分辨率、没有给计时方法**。这是书章而非论文的典型特征——**数字要当"量级"读，不能当"结论"读**。

## Game Development Relevance

**这一章是"实时 VFX 模拟"这条线的分水岭**：

- **在那之前**：游戏里的烟/火/水花几乎全是**烘焙**（Houdini → 翻页图 → 运行时播放）——成本在 DCC，运行时只付采样；
- **在那之后**：出现"运行时解算"的选项，但**预算语言变了**——不再是"多少张贴图、多少粒子"，而是 **分辨率 × pass 数 × 每 pass 带宽**。

对 [[Real-Time VFX Performance Budgeting|SABC 预算表]]的直接含义：

$$\text{GPU 模拟成本} \approx \underbrace{W\times H}_{\text{网格像素}} \times \underbrace{(5+N_{jacobi})}_{\text{每帧 pass 数}} \times \underbrace{B_{\text{每像素}}}_{\text{读 4~5 邻居 + 写 1}}$$

**代入一组量级**（本页估算，非论文数据）：$W=H=256$、$N_{jacobi}=60$、fp16 RGBA（8 B）、每 pass 读 5 写 1 → 每像素 48 B：

$$256^2 \times 65 \times 48\ \text{B} \approx 204\ \text{MB/帧} \;\Rightarrow\; 60\ \text{fps 时} \approx 12\ \text{GB/s}$$

2004 年的 GeForce FX 5950 Ultra 显存带宽约 30 GB/s——**这个规模的模拟在当年就已经吃掉约四成带宽**。所以论文把分辨率定在 **128×128**（约 51 MB/帧），不是审美选择，是**带宽选择**。

> **给你的判据**：GPU 模拟永远不会"算不动"，它先"**读不动/写不动**"。问一个模拟能不能上，先算 `分辨率 × pass 数 × 48 B`，再看它占多少带宽预算——**这比问"GPU 算力够不够"准确得多**。

## Unreal Engine Relevance

- **Niagara GPU 模拟**是这一章的直系后代（只是执行环境从片元程序换成 compute shader）：**Simulation Stage 的每一次迭代 = 一个 pass**，"迭代次数"参数直接乘在成本上；
- **双缓冲**在 Niagara 里表现为 `Previous Frame` 数据的可用性——那不是魔法，是 ping-pong；
- **`Grid2D` / `Grid3D` 数据接口**就是"场 = 纹理"的现代形态（见 [[Niagara]]）；
- **压力求解的 40~80 次迭代**在引擎里通常被砍到个位数 + 视觉补偿——这就是"降档"在模拟线上的真实含义：**不是降分辨率，是降迭代次数**（先把 Jacobi 从 60 砍到 10，再把网格从 256 砍到 128）。

## Technology Evolution

```text
1999  Stam：算法（CPU，四步分解）
2003  Bolz/Krüger/Goodnight：线性求解器上 GPU（"能解 PDE"）
2004  ★ 本篇：完整时变系统上 GPU（"每帧重算"）——借渲染管线
2006  CUDA：正规编程模型（scatter / 共享内存 / 原子）
2009  Compute Shader (DX11) / OpenCL：GPGPU 变成"标准件"
2012+  异步计算：模拟与渲染可并行排队
2015+  GPU 粒子（Niagara GPU / VFX Graph）：本章骨架的引擎化
2020s  神经模拟（[[Neural Physics Simulation]]）——"不求解，学求解"
```

**一条主线**：**每一代都在把"缺的能力"补上**——先补 scatter（compute），再补并发（async），再补"要不要解"（神经）。但**"一次 pass 一遍全屏"的成本形状从来没变**。

## Relationships

### Based On

- [[Stam — Stable Fluids (1999)]] —— 算法与四步分解的直接来源；本文只换执行单元，不改数学。

### Extends

- [[Particle Systems]] —— 同一个"实时模拟"问题的另一条路（离散 vs 场）：粒子散射**恰好是** GPU 在 2004 年做不了的那件事，所以粒子长期留在 CPU，直到 compute。

### Related

- [[GPGPU]] —— 本页的"概念侧"：三条锁（输出位置固定 / 不可读写 / 精度与打包）与后代形态；
- [[GPU Architecture]] —— "为什么会这样"的硬件侧：带宽层级、固定功能单元、pass 的固定开销；
- [[Overdraw]] —— **每个 slabop 就是一次全屏写入**：GPU 模拟的 overdraw 是**结构性**的（by construction），不是美术叠出来的；
- [[Fluid Simulation]] —— 概念页；本页补上其时间线里缺失的"GPU 化"节点。

## Personal Knowledge State

`user_level: Normal`

**你已经会的**：Niagara 的 GPU 粒子、模拟 stage、带宽/ALU 的分野——这套语言你日常在用。

**这一页补的是"为什么"**：为什么模拟要双缓冲、为什么迭代次数直接等于成本、为什么 2004 年只能做 128×128。**你把这三条和 Niagara 的参数面板对上，这一页就吃透了。**

## Learning Value

- **读法（≈20 分钟）**：只读 38.3.1（CPU–GPU 类比）与 38.3.3 的 advect 片段——**四张类比表 + 八行 Cg 就是全章精华**；
- **带回工作的一个问题**：*我这条 GPU 模拟，pass 数是多少？* 大多数人说得出分辨率，说不出 pass 数——**说不出 pass 数就等于没算过成本**。

## Visualization

![[Research/Files/Pictures/Harris 2004 — 图7 二维云模拟十二帧.png]]
**读图**：2D 云模拟的十二帧演化（热力学 + 凝结/蒸发叠加在流体求解上）。**128×128 在此图上已经"够看"**——这是书里最诚实的一处：**分辨率不是画质旋钮，是成本旋钮**；画质靠的是标量场设计与配色。（图 7，Harris 2004）

## Notes

- **证据形态**：GPU Gems 是**书章**，非同行评审；影响力来自教学与工程实践，不来自实验。全章唯一的定量结果是"6× 快于 CPU"与"128×128 云模拟 >80 迭代/秒"，**两者都没有给基线细节**。
- **原文可达性**：NVIDIA 官方页面可直接读全文与取图（本页 5 张原图均来自该页面）。注意 GPU Gems 1 的图注与公式在网页版是 **JPG/GIF 图片**，不是文本。
- **术语校准**：论文里的 "slab operation (slabop)" 是作者自造词，指"一次全屏片元处理 + 一次纹理更新"；本库统一称 **pass**。
- **一个容易记错的点**：本章做的是**同位网格**（cell-centered），**交错网格**（staggered）在 38.5.3 只是作为"扩展"介绍——别把 Fedkiw 2001 的交错网格算到这一章头上。
