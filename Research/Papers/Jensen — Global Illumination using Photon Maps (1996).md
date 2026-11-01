---
type: paper
title: "Global Illumination using Photon Maps"
authors: [Henrik Wann Jensen]
year: 1996
published: 1996-06-17
venue: "Rendering Techniques '96 (7th Eurographics Workshop on Rendering), Springer, pp. 21–30"
url: "http://graphics.ucsd.edu/~henrik/papers/photon_map/global_illumination_using_photon_maps_egwr96.pdf"
code: ""
project_page: "https://cseweb.ucsd.edu/~henrik/papers/ewr7"
category: [rendering, global-illumination, monte-carlo]
importance: A
historical_importance: 5
game_relevance: 4
production_readiness: Industry Adopted
user_level: Normal
status: unread
---

# Global Illumination using Photon Maps —— 存证据，不存答案

> Henrik Wann Jensen，Rendering Techniques '96（EGWR 7），pp. 21–30。**库内 166 篇论文里第一次出现光子图**——这是渲染知识图谱上一个被引用了 30 年、却一直没有页面的空洞。
> **一句话**：把"光"从**眼睛这一侧**拉到**光源这一侧**走一遍，把它**落在表面上的样子原样存下来**（位置 + 入射方向 + 功率，20 字节一颗），之后无论谁问"这一点往这个方向出去多少亮"，都只是在这一点附近查最近的 $N$ 颗光子。**存的不是答案，是证据。**

## TL;DR

- **两趟**：第一趟从光源发射能量包，落在漫反射面上就存下来（**light-driven**）；第二趟从眼睛发射线，遇到"精确算太贵"的地方就**查光子图**（**eye-driven**）。原文的定义极准："We simplify the **representation of the illumination** instead of simplifying the geometry"——**简化的是光，不是场景**；
- **两张图分工**：**焦散图**（高密度，直接可视化）与**全局图**（低密度，供估计用）。拆开之后速度更快、内存更省、**精度反而更高**——因为全局图里不再混着"高密度低能量"的焦散光子与"低密度高能量"的普通光子，**能量尺度一致，估计才准**；
- **两条硬数字**：每颗光子 **20 字节**（平衡 kd-tree 的直接结果），$N$ 颗里找 $M$ 最近是 $O(M\log_2N)$；**216 颗 shadow photon ≈ 5 KB，把 shadow ray 数量砍掉 70% 以上**——**这是全文最漂亮的"小预算买大效果"**；
- **一句话点破预算的本质**（原文）："the necessary number of photons **is not directly related to the number of objects in the scene**. It is instead related to **the complexity of the flux** within the scene." **预算量取决于"亮度的变化有多复杂"，不取决于"场景里有多少东西"**；
- **它的立场是"证据优于答案"**：与同期 Ward 的辐照度缓存相比，光子图存的是**入射通量 + 方向**（"simpler and less accurate"），代价是更噪，收益是**任意 BRDF 都能积分、可以做重要性采样、可以把光子当虚拟光源再走一遍**（后者就是 [[Keller — Instant Radiosity (1997)|Keller 1997]]）。

## Problem

1996 年的两条主流路线各有硬伤，而它们的硬伤**正好互补**：

| 路线 | 强项 | 硬伤 |
|---|---|---|
| **Radiosity**（1984 起） | 漫反射 GI 的解析解，无噪声 | **方向信息吃内存**；处理不了镜面；改一次光要重解一次矩阵 |
| **Monte Carlo 光线追踪** | 什么材质都能算 | **噪声**；而且**焦散几乎算不出来**——从眼睛侧采到"先打到镜面再打到漫反射"的路径，概率低到不可用 |

真正的痛点有三个，原文都点了名：

1. **焦散**：eye-driven 采样下"几乎不可能"（"We never compute caustics via Monte Carlo sampling since this is almost impossible in most situations"）；
2. **软间接光**：每一条第一次漫反射之后的路径都要递归采下去，**平均深度不小**；
3. **阴影线**：面光源的软阴影要撒很多 shadow ray，而**大多数区域要么全亮要么全暗**，撒这些线纯属浪费。

**所以这一篇要做的不是"更准的 GI"，是"把最贵的三种计算，换成一次查表"。**

## Historical Context

把它放回"光源侧 vs 眼睛侧"这条轴上看，位置非常清楚：

```text
1984  Goral        Radiosity：把"面与面之间换多少能量"写成矩阵（烘答案）
1986  Arvo         Backward ray tracing：从光源侧走（概念提出）
1986  Kajiya       Rendering Equation：把问题写成积分方程（容器）
1988  Ward         Irradiance caching：在表面点上存**辐照度**（存答案）
1990  Arvo & Kirk  Particle transport：把"光子"当粒子输运（方法雏形）
1995  Jensen & Christensen  Photon map 初版
1996  ★ 本篇        两张光子图 + shadow photon + 锥形滤波（工程闭合）
1997  Keller       Instant Radiosity：**把同一批光子当虚拟点光源再走一遍**
1998  Jensen & Christensen  参与介质版光子图（体积里的光子）
2001  Jensen       光子映射专著
2008  Hachisuka    Progressive Photon Mapping（去掉"半径必须是常数"）
2011  Hachisuka    SPPM（随机渐进 + 每像素半径）
2013  Hachisuka    VCM（把光子与路径的权重统一起来）
```

**这条轴上有一个反复出现的分岔，值得单独记：**

```text
同一个"光源侧走一遍"的动作，可以落成三种东西——
  ① 落成矩阵（Goral 1984 radiosity）           → 存"面与面之间的关系"
  ② 落成点集（Jensen 1996 光子图）              → 存"能量落在哪、从哪来"
  ③ 落成虚拟光源（Keller 1997 instant radiosity）→ 存"可以再采样的光源"
三者用的是同一次传输，但能回答的问题完全不同。
```

## Previous Work

- **Arvo 1986（backward ray tracing）/ Arvo & Kirk 1990（particle transport）**：本篇的直接前身，提供了"从光源侧走"和"把能量当粒子"两个概念；
- **Ward 1988（[[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)|irradiance cache]]）**：**最重要的对照**。Ward 在表面点上存辐照度并按梯度插值；Jensen 存入射通量。原文自己划清了界线：
  > "Ward uses a comparable strategy storing irradiance values at surface points. Our approach does however differ significantly... **we store incoming flux (photons) which is much simpler and less accurate** than irradiance values. Our motivation for doing so is that we obtain a very flexible environment with a lot of useful information that can be applied in the rendering step."
  **这段是"存答案 vs 存证据"最早的一次自觉表述**；
- **Radiance 程序（Greg Ward）**：本文的对照实现，也是"蒙特卡洛光线追踪的工程上限"；
- **Chen et al. 1991（progressive multi-pass）/ Rushmeier et al.（几何简化 + radiosity）**：两条"先粗后精"的前辈路线，本文明确说自己走的是另一条——**简化光照表示，而不是简化几何**。

## Core Idea

### 一、光子是什么，为什么它比辐照度"便宜且灵活"

一颗光子 = **一包能量**，从光源发出、按表面 BRDF 决定下一次方向、用俄罗斯轮盘赌决定死活；每次打到**漫反射面**就存一条记录。存什么？**位置 + 入射方向 + 功率（$\Delta\Phi$）**。

**关键区别在"信息量 vs 使用成本"**：

| | Ward 存辐照度 | Jensen 存光子 |
|---|---|---|
| 一条记录 | 一个标量（+ 梯度） | 位置 + 方向 + 功率 |
| 能算什么 | 漫反射出射 | **任意 BRDF 的 $L_r(x,\Psi_r)$**（因为方向在） |
| 精度 | 高（已积分过） | 低（要靠 $N$ 近邻估计） |
| 内存 | 每条固定 | **20 字节/颗**（kd-tree 压缩后） |
| 能重加权吗 | 不能 | **能**（改权重就是改光子） |

### 二、辐射估计：把"积分"换成"查最近 $N$ 颗"

$$\boxed{\;L_r(x,\Psi_r)=\int_\Omega f_r(x,\Psi_r,\Psi_i)\frac{d^2\Phi_i(x,\Psi_i)}{dA\,d\omega_i}d\omega_i\;\approx\;\sum_{p=1}^{N}f_r(x,\Psi_r,\Psi_{i,p})\frac{\Delta\Phi_p(x,\Psi_{i,p})}{\pi r^2}\;}$$

其中 $r$ 是"以 $x$ 为心、扩张到装下 $N$ 颗光子"的球半径，$\Delta A\approx\pi r^2$。

**这三个符号就是全部的账**：

- $N$ —— 精度旋钮（$N$ 大 → 噪小、糊多）；
- $r$ —— 由 $N$ 与**局部光子密度**共同决定（**这是它比"固定半径"好的地方**：固定半径在稀疏区全是噪声、在密集区全是糊）；
- $\pi r^2$ —— **面积估计**，也是偏差的来源：它假装最近的 $N$ 颗光子均匀铺在这个圆盘上。

### 三、锥形滤波：给"离得远的邻居"降权

估计的偏差表现为**低频模糊**（图 2 左）。修法不是加光子，是**加权**：

$$w_p=\max\!\left(0,\;1-\frac{d_p}{kr}\right),\qquad L_r\approx\frac{\sum_p f_r\,\Delta\Phi_p\,w_p}{\left(1-\frac{2}{3k}\right)\pi r^2}$$

**归一化因子 $1-\frac{2}{3k}$ 是本文一个容易读漏的细节**：它按"光子分布在**二维**表面上"来算（虽然搜索用的是三维球），因为"面积估计本身也假设光子在表面上"。$k=1$ 一般就够；图 2 用 **12,000 颗光子**渲染心形焦散，未滤波糊成一片，滤波后明显变锐。

### 四、两张图 + shadow photon：三处"用小钱办大事"

| 手段 | 做法 | 买到什么 |
|---|---|---|
| **拆两张图** | 焦散图（只存"经镜面后打到漫反射"的光子，高密度）+ 全局图（所有光子，低密度） | 速度↑ 内存↓ **精度↑**（能量尺度一致）；焦散要"直接可视化"，所以必须高密度，而全局图**不该**被焦散污染 |
| **shadow photon** | 从光源一路打穿场景：第一个交点存普通光子，**之后的交点存 shadow photon** | 渲染时若近邻全是 shadow photon → 必在阴影里，**不用打 shadow ray**；若全是普通光子 → 必全亮。**216 颗 ≈ 5 KB，砍掉 >70% 的 shadow ray** |
| **分类标记** | 光子分 D（直接）/ S（阴影）/ I（间接）三类（图 1） | 让"这一片是直射还是反弹来的"变成可查的元数据 |

### 五、精确算 vs 近似算：这是全文的调度策略

把渲染方程按"光的来源"和"BRDF 的性质"拆成四项：

$$L_r=\underbrace{\int f_rL_{i,l}\cos\theta_i d\omega_i}_{\text{直接}}+\underbrace{\int f_{r,s}(L_{i,c}+L_{i,d})\cos\theta_i d\omega_i}_{\text{镜面/高光}}+\underbrace{\int f_{r,d}L_{i,c}\cos\theta_i d\omega_i}_{\text{焦散}}+\underbrace{\int f_{r,d}L_{i,d}\cos\theta_i d\omega_i}_{\text{软间接}}$$

- **精确算**（走真实光线）：表面**被眼睛直接看到**、或**经少数几次镜面反射看到**、或**距离阈值以内**（"otherwise we might risk inaccurate colour bleeding effects in corners"）；
- **近似算**（查光子图）：**已经漫反射过一次之后**，或**光线权重已经很低**。

> **这条调度的本质是**：**只有"第一眼看到的那一层"值得精确算，其余都是"背景光"**。所以平均值射线深度 ≈ **2**——这个数字解释了一切加速比（Diffuse Cornell：8 min vs Radiance 60 min）。

## Key Contribution

1. **确立"光源侧预计算 + 眼睛侧查询"这个两趟范式**，并给出可工程化的三件套（kd-tree 存储 / 锥形滤波 / 两张图分工）；
2. **把"光子"变成一种可复用的中间表示**：带方向的能量样本，能积分任意 BRDF、能做重要性采样、能当虚拟光源；
3. **shadow photon 这个"元数据"设计**：不是更准的估计，是**更少的计算**；
4. **给出"预算取决于什么"的第一条正确表述**（见下）。

**原文那句最该背下来的话**：

> "the necessary number of photons **is not directly related to the number of objects in the scenes**. It is instead related to **the complexity of the flux within the scene**. We would probably be able to render the Cornell box with detailed stone walls made of millions of triangles using the same number of photons as we did with the simple Cornell box."

以及紧随其后的一句自嘲：

> "The current strategy is just to use enough photons (**what the available memory permits**)."

**两句合起来是一个完整的预算观：预算量由"变化复杂度"决定，而实际取值由"内存上限"倒推。** 30 年后这句话在 [[Overdraw]]、[[Hexels]]、[[2026-10-26-Kim — RLSplat Resource-Linear Budget Control for Feed-forward 3D Gaussian Splatting|RLSplat]] 里以不同形式重演。

## Why It Works

1. **为什么密度估计是对的**：$L_r$ 是一个**在表面上对入射通量做卷积**的量；光子就是 $d^2\Phi/dA$ 的离散样本。用"固定 $N$ 颗、半径自适应"去估计，等于**让核宽度跟着样本密度走**——这是所有非参数密度估计的标准做法（kd-tree 的 $k$-NN）；
2. **为什么"糊"在这里是优点**：原文自己说了——"The blur in the photon map is actually an advantage since it **reduces noise in the final gathering step**"。因为最终图像的第一层是精确采样的，**光子图只需要提供"低频的正确背景"**；把高频留给采样，把低频留给估计，是这套方法的核心分工；
3. **为什么拆两张图能提精度**：估计量的方差强烈依赖样本的**能量分布**。把"高密度低能量"的焦散光子混进全局图，会让近邻里出现能量差异巨大的样本，**估计的方差被放大**；拆开之后每组内部能量尺度一致。

## Limitations

1. **有偏**（density estimation 的固有偏差）：$N$ 固定则 $r$ 随密度变，稠密处糊、稀疏处噪；**PPM（2008）就是为了去掉这个"半径必须收敛"的枷锁**；
2. **内存随光子数线性涨**：~9 MB 在 1996 年是"能接受"，在 2026 年是"太便宜"——**但这条限制从未消失，只是尺度变了**；
3. **多光源会爆炸**：原文诚实写了 "naive emission of photons from every light source will generate a very large number of photons"，而且"目前只处理了少于 10 个光源的场景"。**这直接催生了后来的"光源重要性采样 / 三趟法"**；
4. **高光表面不适合查表**：需要的光子数太多（"a very large number of photons is needed"），所以**高光仍然走光线追踪**——"in order to limit the memory requirements"。**这不是能力边界，是内存边界的直接后果**；
5. **当年没有参与介质**：原文把它列为 future work（"The photon map could also be used to represent flux within participating media. This should be straightforward..."）。**这一句在 1998 年被作者自己兑现**（Jensen & Christensen 的介质版光子图）；
6. **时间**：1996 年的数在今天没有比较意义（100 MHz Pentium / 32 MB），**但比例有意义**：建图 ≈ 渲染的 1/10（Museum：5 min vs 51 min）。

## Game Development Relevance

**这一篇是所有"烘焙"的总源头**——不是比喻，是字面意义上的谱系：

```text
1996 光子图（本篇）
  ├→ 2001 光子映射专著 → 离线渲染的默认 GI 方案
  ├→ 2008/2011/2013 PPM / SPPM / VCM → 离线渲染的现代形态
  └→ 烘焙线：把"光子落在哪"变成"贴图/探针上的一个数"
       ├→ Lightmap（UE Lightmass 的静态 GI）
       ├→ Light Probe / DDGI（把"存证据"退化成"存答案"）
       └→ 移动端烘焙 GI（同一套账，只是预算小两个数量级）
```

**为什么它对今天的项目仍然有用**（三条可直接搬的）：

1. **"只精确算第一层"这条调度策略**，是所有实时 GI 方案共同的骨架：Lumen 的 surface cache、DDGI 的探针、ReSTIR 的重采样，**都在做同一件事——把"看得见的那一层"和"背景光"分开付账**；
2. **"shadow photon 用 5 KB 换 70%"是一个可复制的模板**：**不要用更好的估计去省计算，用一个极小的元数据去跳过计算**。这条在今天的 VFX 里对应的是"用一张 8×8 的 tile 占用图跳过整块"，在 [[Overdraw]] 里对应的是"用一次 early-Z 查询跳过一片"；
3. **"预算 ∝ 变化复杂度"这条判据可以直接拿来审自己的 SABC 表**：同屏 3000 颗粒子，**如果它们全在同一块区域均匀分布**，与**分成 10 团各自集中**，成本完全不同（后者 overdraw 集中、批次更碎）。**"数量"是弱代理，"分布"才是真变量**——这条在 10-28 那天的"分布 > 总量"里已经验证过一次，**它的源头在这里**。

## Unreal Engine Relevance

- **UE 的静态 GI（Lightmass）就是这条线的工业后代**：光子式的间接光收集 + 辐照度缓存 + 存进 lightmap/体积纹理。**区别在于它把"证据"又压回了"答案"**（每个 texel 存一个辐照度），所以**改灯要重烘**——这正是 1996 年 Ward 路线与 Jensen 路线的分野，在 2026 年的引擎里仍然存在；
- **一个可以直接做的对照实验**（30 分钟）：拿一张已经烘好的 lightmap，问三个问题——① 这张图里存的是"答案"还是"证据"？② 如果我把某盏灯的能量乘 0.8，这张图还能用吗（线性缩放成立吗）？③ 如果我把灯**挪** 20 cm，哪些部分必须重烘？**第三问的答案就是"编辑面积"，也就是 10-30 与 11-01 两篇前沿论文在算的那笔账**；
- **不建议**把光子图当实时方案读：它是**离线/烘焙**语境的产物，实时对应物是探针与缓存，**两者共享的是"预算哲学"，不是算法**。

## Technology Evolution

```text
"光从哪一侧走，走完存成什么"
1984  Radiosity          光源侧 → 存成**矩阵**（面与面的换能比）
1986  Arvo               光源侧 → 概念
1988  Ward               眼睛侧 → 存成**辐照度缓存**（答案）
1990  Arvo & Kirk        光源侧 → 存成**粒子**
1996  ★ 本篇             光源侧 → 存成**带方向的点集**（证据）+ 眼睛侧查询
1997  Keller             同一批点 → 存成**虚拟光源**（可再采样）
1998  Jensen             同一套方法 → 进**参与介质**
2005  Dachsbacher        RSM：把第一次反弹存进 **G-buffer**（证据，但只有一层）
2008  PPM                去掉"半径必须固定"
2013  VCM                光子与路径权重统一
2019  Majercik           DDGI：把证据又压回**探针里的辐照度**（答案，但可实时更新）
2020s ReSTIR             把"复用"从光源侧搬到**像素侧的蓄水池**
        ↓
      存储形态：矩阵 → 标量缓存 → 点集 → 虚拟光源 → G-buffer → 探针 → 蓄水池
        ↓
      "每一种形态都在回答同一个问题：哪些量值得留下来？"
```

## Relationships

### Based On

- [[Kajiya — The Rendering Equation (1986)]] — 被积的那个积分；本篇的式 (3) 就是它的离散化；
- [[Goral — Modeling the Interaction of Light Between Diffuse Surfaces (1984)]] — **同一个问题的另一种存储形态**（矩阵 vs 点集）。两者都是"光源侧预计算"，但 Goral 只能做漫反射、改灯要重解；Jensen 能带方向、改灯只改查询权重。

### Extends / Improves

- **Arvo & Kirk 1990（particle transport）**：把"粒子"从概念做成可查询的数据结构（平衡 kd-tree + 20 字节/颗）；
- **Ward 1988（irradiance cache）**：把"存答案"改成"存证据"，换取 BRDF 无关性与可重加权。

### Contrasts

- [[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)|Ward 1988]] — **存答案 vs 存证据**的第一组对照。**判据**：如果下游要问的问题**只有一种**（这个点的漫反射出射是多少），存答案是压倒性的；如果下游要问**很多种**（换 BRDF / 换视角 / 当光源 / 加权），存证据才划算；
- [[Keller — Instant Radiosity (1997)|Keller 1997]] — **同一批数据的第二种用法**。光子图把它当"密度估计的样本"，Instant Radiosity 把它当"可以直接再采样的虚拟点光源"。**同一个 $d^2\Phi$，两种问法**——这正是"证据优于答案"的最强证据；
- [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)|RSM 2005]] — **证据的"只有一层"版本**：把第一次反弹存进光源视角的 G-buffer。**比光子图便宜、比光子图窄**（只能做一次反弹）；
- [[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)|PRT 2002]] — **同一笔账的另一端**：PRT 存"算子"（换灯一次点积），光子图存"一次传输的实现"。**两者都在回答"预计算什么"，答案不同：一个预计算关系，一个预计算结果**。

### Followed By

- **PPM / SPPM / VCM**（库外）：去掉半径约束、统一权重；
- **烘焙 GI 全家**（Lightmass / 移动端烘焙）：把证据压回答案，换取运行时零成本；
- **[[2026-10-30-Liao — From Walks to Skeletons A Reusable Transport Architecture|Walks to Skeletons]] / [[2026-10-30-Liao — Walk on Multiscale Media Compiling Editable and Composable Transport Medium Prefabs|WoMM]]**：**30 年后同一个立场的回归**——都选择存"证据"（轨迹图 / 事件记录）而不是"答案"（辐照度 / 首达概率），理由也一模一样：**答案只有一个，证据能回答很多问题**。

## Personal Knowledge State

- **Normal**：两趟结构、密度估计、锥形滤波、shadow photon 四点都无需新数学；$1-\frac{2}{3k}$ 的归一化需要想一下"二维分布"这个前提；
- **与你已有认知的接口**：你天天在用的 **lightmap 就是这篇的"退化版"**（证据 → 答案）。**看懂这一篇，就看清了"为什么改一盏灯要重烘一片"**——不是因为引擎笨，是因为当年为了省运行时，主动把"证据"烧成了"答案"。

## Learning Value

1. **"存证据 vs 存答案"是本库目前最通用的一条表示判据**。它同时解释了：为什么 lightmap 便宜但不能改、为什么光子图贵但能换 BRDF、为什么 WoMM 存事件记录而不是首达概率、为什么 PRT 存算子而不是存图；
2. **"预算 ∝ 变化复杂度，不 ∝ 物体数量"**——这条直接可以拿去审 SABC 表里的"同屏粒子数"。**数量是弱代理，分布才是真变量**；
3. **"用 5 KB 的元数据砍掉 70% 的计算"**：这是所有性能优化的原型动作——**不是算得更快，是知道哪些不用算**。

## Visualization

![[Research/Files/Pictures/Jensen 1996 — 图3 博物馆场景最终渲染.png]]
**读图**：这是 1996 年的"最终答案"。场景里有 5000 个普通物体 + 1 个程序化物体（那个"球鳞"，用 Schlick 反射模型），两盏小的球形面光源。**该看的四处**：① 玻璃球在粗糙地面上的**焦散**（亮斑）；② 金属茶壶在**光泽圆柱**上的反射；③ 墙面之间的**颜色渗透**（红墙把地板染红）；④ 球鳞表面的**软间接光**。**这张图的意义在于它同时包含了"眼睛侧能采到的"和"眼睛侧采不到的"**——焦散就是后者，而它之所以出现，全靠第一趟的光子。（图 3，Jensen 1996）

![[Research/Files/Pictures/Jensen 1996 — 图4 全局光子图直接可视化.png]]
**读图**：**同一场景，只显示全局光子图的辐射估计**（165,000 颗，平均每次求交用 80 颗近邻），不做任何精确采样。**这幅图是本篇的灵魂**：它是"证据"长什么样——**糊、但有正确的低频结构**；焦散还在（玻璃球下方那块亮斑）、颜色渗透还在、茶壶的高光反射也在（因为它本来就是光泽面，光子带着方向）。**注意它有多"脏"**：这正是"存证据"的代价。而作者的判断是——**这点脏不要紧，因为最终图像的第一层会被精确采样覆盖，光子图只负责背景**。（图 4，Jensen 1996）

![[Research/Files/Pictures/Jensen 1996 — 图2 锥形滤波前后对比.png]]
**读图**：同一个心形焦散，**只用 12,000 颗光子**（故意给得很少）。**左**是标准辐射估计——圆盘半径 $r$ 内所有光子等权，于是边缘糊、细节被抹平；**右**是锥形滤波（$k=1$），离中心越远权重越低，**锐度明显回来**。**这张图讲的是一个通用道理**：当样本不够时，**"怎么加权"比"加多少样本"更有效**——因为加权不花钱，加样本要花钱。（图 2，Jensen 1996）

## Notes

**证据形态**：EGWR '96 正式版（Springer，pp. 21–30）的**扩展版 PDF**，作者主页镜像（`graphics.ucsd.edu/~henrik/`，1.35 MB，17 页含彩页）。**正文 12 页 + 5 页彩图**（图 3–7 在彩页上）。

**本次的"老经典可达性"经验（第 N 次验证）**：
- `graphics.stanford.edu/papers/photon_map/global_illum.pdf` → **HTTP 000**（连接被掐，与前几日 Stanford 站的表现一致）；
- Cornell 课程镜像 → **404**；UCSB 课程页 → 返回 HTML；
- ✅ **作者旧域名 `graphics.ucsd.edu/~henrik/papers/...` 与 Rose-Hulman 课程镜像 `rose-hulman.edu/class/csse/csse451/Papers-global illumination/` 均 200，且字节数完全一致（1,350,018）**。
> **规律再确认：2000 年前后的经典，优先找"作者个人主页的旧域名"和"本科课程阅读清单"，不要先试出版社与名校主页。**

**三处必须跟着数字走的口径**：
1. **6× / 7× 是相对 Radiance 的**，且**两张图的像素数不同**（Glossy Cornell：MIRO 2560×1920 vs Radiance 5120×3840）——原文自己提醒"像素数不能直接比，应该比**在光泽面上撒的样本数**"（Radiance 64 spp vs MIRO 最多 96）；引用加速比时必须带上这句；
2. **9 MB 是"光子图"的内存**（289k 焦散 + 165k 全局 × 20 B ≈ 9.1 MB），**不是场景总内存**；
3. **"20 字节/颗"依赖平衡 kd-tree 的紧凑存储**，换成别的结构就变——这是**数据结构决定预算**的早期样本。

**一处应当纠正的常见误传**：本篇**不是**"光子映射的发明"（1995 年作者与 Christensen 已有初版），**它的贡献是"两张图 + shadow photon + 锥形滤波 + 完整的两趟调度"**——即**把方法做成工程**。写历史时不要把 1995 抹掉。

**未能核实**：原文引用的 [15]（Jensen & Christensen 1995）与 [16][17][18]（同组的三篇扩展）均不在库内，**其具体贡献按本篇的转述记录，未做独立验证**。
