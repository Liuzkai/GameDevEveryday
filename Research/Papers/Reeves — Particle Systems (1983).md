---
type: paper
title: "Particle Systems—A Technique for Modeling a Class of Fuzzy Objects"
authors: [William T. Reeves]
year: 1983
published: 1983-04
venue: "ACM Transactions on Graphics 2(2), 91–108（同一工作亦见 SIGGRAPH '83, Computer Graphics 17(3), 359–375）"
url: "http://www.lri.fr/~mbl/ENS/IG2/devoir2/files/docs/fuzzyParticles.pdf"
code: ""
project_page: ""
doi: "10.1145/357318.357320"
category: [vfx, particles, procedural, animation, simulation, classic]
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: Easy
status: read
---

# Particle Systems—A Technique for Modeling a Class of Fuzzy Objects

> 入库于 2026-09-21。原文 PDF 已下载并**逐节核对**（17 页，含 12 张图版与完整参考文献）。
> **这不是一篇"值得一读的经典"，而是你的五个预算维度里其中四个的原始出处。**
> 本库此前有一个结构性空洞：[[Real-Time VFX Performance Budgeting]] 与 [[Niagara]] 都是 Easy，**但整库没有任何一篇 VFX / 粒子系统的历史锚点**。本文补的就是这个洞。

## TL;DR

1983 年，Reeves 在 Lucasfilm 为《星际迷航 II：可汗怒吼》的「创世纪」段落做火墙特效时，提出了用**大量简单粒子**代替曲面来表示"模糊物体"（火、云、烟、水）的方法，并把它概括成一套**可参数化、可分级、可复现**的生产流程。

**一句话价值**：

> **它把"效果"从"美术手工"变成了"一组带随机数的参数"，并顺手把"要在屏幕上画多少粒子"写成了屏占比的函数。**

43 年后你在做的 SABC × 五档画质矩阵，**本质上就是把这两个动作各做一遍** —— 只是你把"全局参数"细化成了四级分类。

## Problem

1983 年之前，计算机图像合成的表示法只有一类：**曲面**（多边形、双三次片、分形面）。

但"模糊物体"——火、云、烟、水——**没有光滑、明确、有光泽的边界**。它们的表面不规则、复杂、定义不清。**用曲面去逼近它们是方法论上的错配**，不是精度问题：

- 曲面表示困难：边界不存在，无从拟合；
- 手工建模昂贵："获得高细节模型需要大量人工设计时间"；
- 动态无法表达：它们"活着"，随时间改变形态，不是刚体、也不服从仿射变换。

同时代已有零星尝试，但都缺关键要素（原文 §1 逐一列了）：

| 前作 | 做了什么 | 缺什么 |
|---|---|---|
| Roger Wilson（Ohio State）| 用粒子做烟囱冒烟 | **既无随机控制，也无动力学** |
| Alvy Ray Smith & Jim Blinn（Cosmos 系列）| 星系恒星的生与死 | 未成方法论 |
| Alan Norton（SIGGRAPH '82）| 粒子生成三维分形形状 | 静态形状，非动态系统 |
| Jim Blinn（SIGGRAPH '82）| 粒子层的光反射函数（土星环）| **明确不处理"模糊物体建模"** |

> **这正是本文的定位**：不是第一个用粒子的人（历史上"最早的电子游戏用一堆发光小点画飞船爆炸"，Reeves 自己承认），**而是第一个把粒子系统写成完整生产方法论的人**。

## Historical Context

```text
1982  Blinn —— 粒子层的光反射（土星环）          ← 只管"怎么被照亮"
1982  Fournier/Fussel/Carpenter —— 分形随机建模  ← 同一套"过程化"哲学
        ↓
★ 1983  Reeves —— 本文：把"模糊物体"当成一个随时间演化的随机系统
        ↓
1985  Reeves & Blau —— 结构化粒子系统（树/草/发丝；整条轨迹当静态形状）
        ↓
1987  Reynolds —— Boids：粒子之上加"外部状态交互"（这是另一个分枝，见 [[UE5Steering]] 方向）
        ↓
1990  Sims —— 数据并行计算做粒子动画（GPU 粒子的概念前身）
        ↓
2006+ GPGPU → 2016+ UE Niagara / Unity VFX Graph —— 计算着色器粒子
        ↓
1983 的"发射率 / 寿命 / 初速度均值+方差"直接变成 Niagara 的 Spawn Rate / Lifetime / Initial Velocity 模块
```

> **注意一个历史细节**：Reeves 1983 的引擎是**逐帧、逐粒子的标量循环**，写在 PDP-10 级机器上。而他在结论里已经写下：
> *"Because they are so simple, they lend themselves to a hardware or firmware implementation. With a hardware antialiased line-drawing routine, the computation of our wall-of-fire element would have been **two to three times faster**."*
> **"粒子足够简单，所以它天然适合硬件实现"** —— 这句话在 2006 年之后的 GPU 粒子上完整兑现。

## Core Idea

### 三处与曲面表示的根本不同（原文 §1）

1. **体积而非边界**：物体不由定义边界的图元集合表示，而由**定义其体积的粒子云**表示；
2. **非静态**：粒子"出生"、"移动改变形态"、"死亡"；
3. **非确定性**：形状与形态**不被完全指定**，而是用**随机过程**生成与改变。

### 一帧的五步循环（原文 §2，这是 Niagara Emitter 生命周期的原型）

```text
(1) 新粒子生成进系统          → Spawn
(2) 每个新粒子被赋予独立属性   → Initialize Particle
(3) 超过寿命的粒子被熄灭       → Lifetime / Kill
(4) 剩余粒子按动力学属性移动   → Update（速度累加 + 加速度）
(5) 渲染存活粒子到帧缓冲       → Render
```

**Reeves 明确说这五步是"可编程的"**：
> *"The particle system can be programmed to execute any set of instructions at each step. Because it is procedural, this approach can incorporate any computational model that describes the appearance or dynamics of the object. For example, the motions and transformations of particles could be tied to the solution of a system of partial differential equations, or particle attributes could be assigned on the basis of statistical mechanics."*

> **一句话**：**五步骨架 + 每步任意算子** —— 这就是 Niagara 的 Simulation Stage 与 Data Interface 的设计空间。**1983 年就已经预留了。**

## Technical Approach（逐节核对）

### §2.1 粒子生成 —— **两种控制方式，这就是"分档"的祖宗**

**方式一：绝对数量（均值和方差）**

$$NParts_f = MeanParts_f + Rand() \times VarParts_f,\quad Rand()\in[-1,+1]$$

**方式二：按屏占比 × 单位面积密度**

$$NParts_f = \big(MeanParts_{sar} + Rand()\times VarParts_{sar}\big)\times ScreenArea$$

Reeves 对方式二的说明，**几乎是今天所有粒子 LOD 文档的原始版本**：

> *"This method controls the **level of detail** of the particle system and, therefore, **the time required to render its image**. For example, **there is no need to generate 100,000 particles in an object that covers 4 pixels on the screen**."*

并且生成率本身可以是**时间的线性函数**（用来做"渐强/渐弱"）：

$$MeanParts_f = InitialMeanParts + DeltaMeanParts \times (f - f_0)$$

**其中 `DeltaMeanParts` 就是今天 Niagara 里 Emission Rate 的曲线（Curve）参数。**

### §2.2 粒子的七个属性（生成时一次性决定）

`初始位置` / `初始速度（速率+方向）` / `初始尺寸` / `初始颜色` / `初始透明度` / `形状` / `寿命`

- **生成形状**（generation shape）：球、圆（x-y 平面）、矩形 —— 决定新粒子被随机放置在哪个区域；
- **抛射角**（ejection angle）：粒子偏离垂直方向的范围（**这是 Niagara "Cone Velocity" 的前身**）；
- **初始速率**：$InitialSpeed = MeanSpeed + Rand()\times VarSpeed$；
- **颜色/透明度/尺寸**：同样的"均值 + 最大偏差"形式；
- **粒子形状**：球形 / 矩形 / **streaked spherical（拉长的球形，用于运动模糊）**。

> **一条贯穿全文的设计原则**：**所有属性都用"均值 + 最大方差"的二元组描述。** 这在生产上是决定性的 —— 美术只需要调两个数就能控制一个维度，而不是去设计分布。

### §2.3–2.4 动力学与熄灭

- 位置更新 = **位置向量 + 速度向量**；再给一个**加速度因子**就能模拟重力，让粒子走抛物线而非直线；
- 颜色/透明度/尺寸各有"变化率"参数（原文自承是**全局的**，但"很容易做成随机的"）；
- **寿命以帧为单位倒数，归零即杀**；
- 另有两条**"零贡献即杀"**的规则 —— 这是最早的性能优化：
  1. 粒子强度（由颜色与透明度算出）跌破阈值 → 杀；
  2. 粒子偏离父系统原点超过给定距离 → 杀（用于裁剪兴趣区域之外）。

### §2.5 渲染 —— **两个假设换来"不需要排序"**

本文的两个简化解假设：

1. **粒子系统不与曲面物体求交**（需要交叠就按裁剪面切成子图，事后合成）；
2. **每个粒子可以当作点光源**显示。

第 2 条带来一连串后果，**几乎定义了整个实时 VFX 管线的形态**：

| 后果 | 原文表述 | 今天的对应 |
|---|---|---|
| 不需要做隐藏面 | "determining hidden surfaces is no longer a problem" | **加法混合（Additive）** |
| 后面的粒子不被遮住，而是"加更多光" | "A particle behind another particle is not obscured but rather adds more light" | Additive / Translucent 混合模式 |
| **不需要排序** | "no sorting of the particles is needed... rendered in whatever order they are generated" | **—— 注意：这是 Additive 模式的专属红利** |
| **没有阴影问题** | "Shadows are no longer a problem, since particles do not reflect but emit light" | 粒子投影需要额外系统（Niagara 的 Light 模块） |
| 颜色加法会溢出 → **钳制而非回绕** | "clamps the individual RGB intensities at the maximum... instead of letting them wrap around" | HDR 缓冲出现前的做法；今天用浮点缓冲绕开了 |
| 全部抗锯齿绘制 | "All particle shapes are drawn antialiased" | 防止时间走样与频闪 |

> **⚠️ 一处历史的关键分岔点，值得记下来**：Reeves 的"不用排序"是**因为用了加法混合**。**一旦改成 Alpha 混合（半透明叠加），排序立刻变成必须** —— 这正是本库 [[GS 半透明顺序依赖]] 与 [[GS 排序成本三乘数_TileGS]] 两个图解在解决的问题。**同一个约束（顺序无关性）在 1983 与 2026 两次成为管线设计的核心，第一次靠"加法"，第二次靠"排序"。**

### §2.6 粒子层级 —— **"发射器数"这个预算维度的原始出处**

> *"The model designer creates a particle system **in which the particles are themselves particle systems**. When the parent particle system is transformed, so are all of its descendant particle systems... The parent's mean color and its variance are used to select the mean color and variance of the offspring... The number of new particle systems generated at a frame is based on the parent's particle generation rate... **The data structure used to represent the hierarchy is a tree.**"*

**这就是 Niagara 里 "Spawn Particles from other Emitters" / Emitter Stage 的原型**：父系统的参数（颜色、速率、尺寸、寿命…）**随机继承**给子系统，实现"大量相似的爆炸，但每个都不一样"。

**同时它也是"发射器数"必须被预算的原始理由**：层级一旦可以递归，"数量"就不再是美术随手拉的滑杆 —— 而是**指数**。

## Key Contribution

1. **把"模糊物体"从曲面表示的错配中解放出来**，给出适用整个类别的表示法；
2. **提出"过程化 + 随机参数"的生产范式** —— 极高细节不需要极高人工时间；
3. **把渲染开销写成可控变量的函数**（屏占比 × 密度），**首次让"该画多少粒子"有据可依**；
4. **给出可复现的生产流程**（见下"随机数检查点"）；
5. 在结论中预言了**硬件实现**与**自适应细节**两条后来的主线。

## Why It Works

因为它**同时降低了三件不同事情的难度**，这是任何单一技术优化做不到的：

| 层面 | 传统曲面路线 | 粒子系统路线 |
|---|---|---|
| **图元复杂度** | 多边形/双三次片 | 点/线段（**简单得多 → 同等算力下能处理更多图元**） |
| **人工设计时间** | 与细节量成正比 | **与细节量脱钩**（细节由随机数产生） |
| **动态表达** | 需显式动画曲面 | **动力学直接给出**（速度 + 加速度） |
| **时间走样** | 难以运动模糊曲面片 | **粒子太简单，容易做运动模糊** |

> **本文最值钱的一句判断**（在结论段，也是整篇的落点）：
> *"The most important aspect of particle systems is that **they move**: good dynamics are quite often the key to making objects look real."*
> **【对你的直接意义】**：这句话给出了一条**分档原则** —— **砍预算时，优先砍"静态细节"（粒子尺寸/贴图/颜色分层），最后才砍"动态"（发射率曲线、寿命、速度散布）**。因为"动得好"才是可信度的来源，静态细节只影响精致度。

## Limitations（原文自述，§5 与结论）

1. **点光源假设不普适**：爆炸与火符合得很好（原文脚注明说），**云与水不符合**；
2. **不支持粒子被打亮**：下一代要做"粒子必须被渲染成独立的光反射物体"；
3. **云的自阴影做不到** —— 而原文明确指出：**"这一点非常重要，因为它是让云看起来像云的关键"**（→ 通向 [[Participating Media]]）；
4. **云所需的粒子数会极大** ——*"This will require an efficient rendering algorithm."*（→ 通向体积渲染与今天的 OverDraw 预算）；
5. **光照不参与全局照明**：火墙本该照亮星球表面，实现里做不到 —— 团队 **Tom Duff 手工在火环中心上方加了一盏强点光**来伪造那个光晕（原文坦率记录了这个 workaround）。

> **Limitation 5 是一条值得记住的产业史细节**：**渲染器做不到的，就用手工技巧补** —— 直到 2026 年 [[Real-Time Global Illumination]] 才把这笔账真正还上。

## 生产案例：Genesis Demo 的火墙（原文 §3，含全部粒子数）

**这是 1983 年"百万级粒子"的实测账本**，逐图核对：

| 图 | 内容 | 粒子系统数 | 粒子数 |
|---|---|---|---|
| Fig. 4 | 创世纪炸弹初始撞击 | 1 大 + 约 20 小 | **约 25,000** |
| Fig. 5 | 扩散中的火墙 | 约 200 | **75,000** |
| Fig. 6 | 越过星球边缘的火环 | 约 200 个爆炸 | **85,000** |
| Fig. 7–8 | 火墙即将/已经吞没镜头 | **约 400** | **超过 750,000** |

**结构**：**两层粒子系统层级**（顶层系统生成"粒子本身是粒子系统"）→ 第二层粒子系统按**距撞击点的距离**错开起始时间 → 造出"火墙扩散"的时间差。每个第二层系统的参数**从父系统随机继承**，所以每个爆炸高度不同、颜色不同。

**颜色是加法混合的副产品**（原文明确）：粒子主导为红 + 一点绿 → 大量粒子覆盖同一像素时**红通道先被钳到饱和**、绿继续上涨 → 中心出现**橙黄热核**，边缘退回红色，极密处**加一点蓝**得到白心。"颜色变化率"模拟发光材料的冷却（**绿蓝先掉，红最后掉**）。

**随机数检查点（本节最被低估的一段）**：

> *"To checkpoint a production, all that need be saved is this random number table — **we do not save all the parameters of 750,000 particles**. To restart a computation at frame n, the closest preceding frame p is found that cannot contribute particles to frame n... Frame p+1's random number table is then read, and particle generation can begin from there."*

**配套的关键不变式**（原文明确）：

> *"**all stochastic decisions concerning a particle are performed when it is generated. After that, its motion is deterministic.**"*
> —— **随机性只存在于生成时刻；生成之后的运动是确定性的。**

> **这是一条可直接迁移的架构原则**：**随机数与动力学解耦，换来"可回放、可断点续算"**。Reeves 也诚实标注了它的边界：一旦用随机数扰动**动力学**（如模拟湍流），检查点机制就必须重做、并需要**更确定性的可复现随机数生成器**。
> **【对今天的映射】**：引擎里"确定性粒子"（Deterministic / Seeded）能不能做到逐帧一致，取决于**随机数是否只在 Spawn 阶段被消费**。这一条在联机同步、回放、性能回归对比里都是硬需求。

## Game Development Relevance

### 🔴 **你的五个预算维度，五个全部能在 1978–1983 找到祖先**

这是本次入库最有价值的发现，**逐条给出原文出处**：

| 你的维度 | S/A/B/C 上限 | 祖先（年份） | 原始机制 |
|---|---|---|---|
| **Niagara 发射器数** | 12 / 8 / 4 / 2 | **Reeves 1983 §2.6** | 粒子系统的**树状层级**（粒子本身是粒子系统）→ 数量可递归 → 必须设上限 |
| **同屏粒子数** | 3000/1200/400/100 | **Reeves 1983 §2.1** | **屏占比 × 单位面积密度** —— 原文原句即"控制细节层次与渲染时间" |
| **贴图尺寸** | 2048/1024/512/256 | **Williams 1978**（见 [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]） | *"extra lateral resolution is purchased at **square law** expense"* —— 分辨率/内存的平方律 |
| **动态灯光** | ≤3 / ≤2 / ≤1 / 0 | **Williams 1978** | **每盏灯的阴影代价 ≈ 2× 场景渲染 + 分辨率相关固定开销** |
| **VFX 时长** | 3s / 2s / 1s / 0.5s | **Reeves 1983 §2.4 / §2.2** | **寿命以帧计、生成时决定、归零即杀** |

> **结论（可以直接写进你的分档文档）**：
> **你的预算体系不是"从经验里总结出来的规则"，而是两类 1978–1983 就已存在的成本结构的重新参数化。**
> 两条基本法则：
> 1. **数量类维度（粒子/发射器）→ 成本随数量线性**，所以用"屏占比或分级数量"封顶；
> 2. **质量类维度（分辨率/贴图）→ 成本随边长平方**，所以每上一档只能翻倍，不能线性加；
> 3. **每盏动态灯的代价是一个"固定倍数"而非"增量"** —— 所以它必须是极小的整数上限（≤3 / ≤2 / ≤1 / **0**）。
> **这解释了为什么"动态灯光"的上限是 0–3 而"同屏粒子"可以是 100–3000**：**前者单位是"又一个完整渲染遍"（2×），后者单位是"又一个点"。**

### 分档启示（从原文直接推出）

- **Reeves 的"屏占比密度"是比"固定数量上限"更好的分档变量** —— 它对性能的预测性是**几何的**（面积变化 → 数量按比例变化），而固定上限对性能的预测性是**统计的**。**如果 NGR 当前的 3000/1200/400/100 是"绝对上限"，可以考虑补一层"每千像素粒子数"的约束**，用于极端镜头（如镜头贴脸时）。
- **"砍预算的优先级"由本文给出**（见 Why It Works）：**先砍静态细节、后砍动态**。
- **发射器数的上限本质是"层级深度 × 每层分支数"的函数**：如果 NGR 允许嵌套发射器，那么"≤12"必须在**树的总节点数**意义下解释，而不只是顶层发射器数。

## Unreal Engine Relevance

**映射关系逐条对应（不是类比，是同一条设计）**：

| Reeves 1983 | Niagara（今日） |
|---|---|
| 一帧五步循环（生成/赋属性/熄灭/移动/渲染）| Emitter 生命周期：Spawn → Initialize → Update → Render → Kill |
| 属性"均值 + 最大方差" | Random Range 节点 / Curve + Uniform 分布 |
| 生成形状（球/圆/矩形）| Shape Location 模块 |
| 抛射角 | Cone Velocity / Add Velocity in Cone |
| 速率是时间的线性函数 | Emission Rate 的 Curve 参数 |
| 粒子形状 = streaked spherical | Sprite Renderer 的 Velocity Alignment 拉伸 |
| **粒子层级（粒子本身是粒子系统）** | **Emitter Stage 内 Spawn 粒子 / Spawn Particles from other Emitters** |
| "强度低于阈值即杀" | Kill Particles + 透明度阈值 |
| **无排序（因加法混合）** | Renderer 的 Sort Mode = **None**（仅 Additive 安全）|

**⚠️ 一处必须澄清的映射边界**：Reeves 的"无排序"是**加法混合的专属结论**，**不能推广到 Alpha Blend**。Niagara 里 Renderer 的 Sort Mode 与 Facing Mode 选择，直接决定这句话成立与否 —— 这也是本库 [[GS 半透明顺序依赖]] 的同一问题。

## Technology Evolution

见新建概念笔记 [[Particle Systems]] 的 Historical Evolution 段（本文对应其中 **1983** 一行）。

## Relationships

### Based On

- **Fournier / Fussel / Carpenter 1982（分形随机建模）** —— 原文自述"我们的工作与分形建模的进展相关"，共享"**过程化 + 随机**"的哲学；
- **Catmull 的帧缓冲 / 隐藏面工作** —— 粒子最终仍需写进帧缓冲。

### Extends

- **Blinn 1982（粒子层的光反射）** —— 原文明确区分：Blinn 解决"粒子怎么被照亮"，**本文解决"模糊物体怎么被建模"**。

### Related

- [[Participating Media]] —— **本文 §5 的"下一步"就是它的入口**：云的**自阴影**与**粒子被照亮**两个做不到的点，正是体积渲染要解决的；
- [[Real-Time VFX Performance Budgeting]] —— **本文是它的历史源头**（屏占比 LOD、发射器数、寿命）；
- [[Niagara]] —— 五步循环与层级结构的直接继承者；
- [[Williams — Casting Curved Shadows on Curved Surfaces (1978)]] —— **同一天入库的另一半**：本文给"粒子侧的预算法则"，Williams 给"灯光侧的预算法则"。

### Followed By

- **Reeves & Blau 1985** —— 结构化粒子系统：把**整条轨迹**当静态形状画出来 → 头发、毛皮、草（**这是"static particles / strands"的源头，也是本库 [[Hair Rendering]] 资产表示的最早祖先**）；
- **Reynolds 1987（Boids）** —— 在粒子之上加"外部状态交互"（目标追寻、碰撞避免、群体中心、有限感知）；
- **Müller 2003（SPH）** —— 粒子 + 流体力学 → 有表面的水；
- **GPU 粒子（2006+）→ Niagara（2016+）** —— 结论里那句"适合硬件实现"在 23 年后兑现。

### Contrasts

- **纯 Eulerian 网格流体**（Niagara Fluids 的另一条路）：粒子是 **Lagrangian** 的，每个粒子有独立历史；网格是**场**的。**本文的"每粒子七属性"在网格法里不存在** —— 这也是为什么 Niagara 的两种路径不能互换。

## Personal Knowledge State

- **user_level: Easy** —— **这是你的专业基础，不做教学**（Easy Policy 第 4 条例外：经典文献填补重要历史缺口）。
- **按本库 9-20 建立的"分层模板"拆两层**（可复用）：

| 层 | 内容 | 状态 |
|---|---|---|
| **操作层** | 发射率/寿命/初速度/形状/混合模式/层级 | **Easy —— 你每天都在用，无需复述** |
| **成本模型层** | **"屏占比 × 密度"作为分档变量**、**发射器数是树节点数**、**随机数只在 Spawn 消费以换确定性** | **Normal —— 这三条是本文真正能给你新东西的地方** |

## Learning Value

**不是"学一个新知识"，而是"给你的经验找到坐标"**：

1. **省掉一次重复发明**：你的 SABC 分级与五档矩阵，可以在文档里直接引用 1978/1983 的两条成本法则，而不用从零论证"为什么动态灯光上限是 0–3"；
2. **提供一条可反驳的判据**：**"绝对数量上限" vs "屏占比密度"** —— 前者是统计的，后者是几何的。这值得你在 NGR 里试一次；
3. **给出历史坐标用于对外沟通**：跟美术解释"为什么要限制发射器嵌套"时，**"1983 年 Lucasfilm 就要靠树结构管这个"**比"性能不够"有效得多。

## Mastery Criteria（4 条，纸面自测）

1. 说出 **Reeves 为什么不需要给粒子排序**，并指出这个结论在**改成 Alpha 混合后是否还成立**（答案：不成立）；
2. 说出 **"屏占比 × 密度"为什么比"绝对数量上限"是更好的分档变量**（提示：几何 vs 统计）；
3. 说出 **随机数检查点机制依赖的那条不变式**，并指出它什么时候会失效（提示：随机性一旦用于动力学扰动）；
4. 说出本文结论里那句"**它们是可变的，这才是关键**"对应的**分档优先级**（答案：先砍静态细节、后砍动态）。

> **一句话检验**：能说出 **"我的五个维度里有四个的祖先在 1983 年就已经写好了"**，即算抓住了本文。

## Visualization

![[预算五维_1978-1983_源头图解.html]]

见 [[预算五维_1978-1983_源头图解]] —— 把 1978/1983 的两条成本法则与你当前的 SABC × 五档矩阵直接叠在一起。

## Notes

- 2026-09-21 入库。**原文 PDF 已下载并逐节核对**（17 页，`http://www.lri.fr/~mbl/ENS/IG2/devoir2/files/docs/fuzzyParticles.pdf`）。**所有粒子数、成本公式、原文引语均来自抽取文本，非记忆。**
- **发表信息核对**：ACM TOG 2(2), April 1983, pp. 91–108, DOI `10.1145/357318.357320`（与 ACM DL 页一致）；SIGGRAPH '83 版本为 Computer Graphics 17(3), pp. 359–375（DOI `10.1145/964967.801167`）。**本次核对的是 TOG 版的再印本**（页脚标明 "Reprinted From acm Transactions On Graphics--April 1983--Vol. 2, No. 2"）。
- **作者单位**：Lucasfilm Ltd, San Rafael, CA。致谢名单包含 **Loren Carpenter / Ed Catmull / Pat Cole / Rob Cook / Tom Duff / Rob Poor / Tom Porter / Alvy Ray Smith** —— 即 Pixar 前身的整套班底。
- **⚠️ 一处需要避免的常见误传**：Reeves 1983 **不是**"第一个提出粒子系统的人"。原文 §1 自己澄清："**建模对象为粒子集合并不是新想法**"，并列出 Wilson / Smith & Blinn / Norton / Blinn 四条前史。**本文的贡献是把粒子系统写成完整方法论 + 完成一次大规模生产验证。**
- **相关性判断（为什么它值得进库而不是只当背景）**：原文 §2.1 那句 *"there is no need to generate 100,000 particles in an object that covers 4 pixels on the screen"* **就是用户当前工作的 43 年前版本**。这一类"同一句话在两个时代分别被写下"的匹配，是本库判定经典价值的主要依据。
