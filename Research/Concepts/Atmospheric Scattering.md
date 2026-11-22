---
type: concept
user_level: Normal
aliases: ["Atmospheric Scattering", "大气散射", "Sky Rendering", "天空渲染", "Aerial Perspective", "空气透视"]
prerequisites: ["[[Participating Media]]", "[[Rendering Equation]]", "[[Linear Transport Theory]]"]
first_introduced: "Nishita et al. 1993（首个物理正确的天空渲染）；理论源头为 Chandrasekhar 1950《Radiative Transfer》"
---

# Atmospheric Scattering

## Definition

**行星大气这层介质把太阳光散射进你眼睛的过程。** 它同时解释了四件在游戏里天天见、却常被当成四件不同事的东西：

| 现象 | 同一套积分里的哪一项 |
|---|---|
| **天空为什么是蓝的** | 分子散射 $\propto \lambda^{-4}$——短波被散得多 |
| **日落为什么是红的** | 光程变长（太阳贴近地平线）⇒ 蓝光在半路被散光，只剩长波直达 |
| **远山为什么发灰发蓝** | **空气透视**：视线穿过的那段介质自己发的光，叠加在物体上 |
| **为什么"看不见太阳"时天还没黑** | 多阶散射：光绕过地球阴影，仍从侧面散过来 |

**一句话判据：只要"光在到达相机之前，在空气里被改过一次方向"，你就在大气散射问题里。**

## Core Principle

### 三个量、两种粒子、一个积分

沿视线 $c \to p$ 的辐亮度是"衰减后的背景 + 沿途自己发的光"：

$$L(c,v) = T(c,p)\,L_o(p,v) + \int_0^{\|p-c\|} L_{scat}(c,\,c-tv,\,v)\,dt,\qquad T(x_a,x_b)=e^{-\int \sigma_t\,ds}$$

$$L_{scat} = \sigma_s \sum_i T(c,x)\,S(x,l_i)\,p(v,l_i)\,E_i,\qquad S = V\!is(l_i)\cdot T(x, x+t_{atmo}l_i)$$

**成本就藏在这个式子里**：每个采样点要乘一次**朝太阳的透射率** $T(x,x+t_{atmo}l_i)$——**这是一个嵌套积分**。所有实时方法的差别，全在"这个嵌套积分怎么省"。

**两种粒子，脾气完全相反**：

| | 空气分子（Rayleigh） | 气溶胶（Mie） |
|---|---|---|
| 波长依赖 | $\lambda^{-4}$（强） | 近似 $\lambda^{-1}$（弱），且随波长**吸收** |
| 相函数 | 平滑的 $1+\cos^2\theta$ | **强前向峰**（$g\approx0.7$） |
| 标高 | **8 km** | **1.2 km**（贴地，随天气剧变） |
| 可变性 | 几乎不变 | **从洁净到沙尘暴，全靠它** |
| 在可见光的吸收 | ~1.5%（臭氧为主） | **不可忽略** |

> **记住这一条：天空的"性格"由分子决定（蓝），天空的"情绪"由气溶胶决定（雾、霞、浑浊）。** 所有把两者合成一个"浑浊度"参数的解析模型，代价就是画不出"情绪"。

### 几何：为什么天空问题比"一团雾"难

大气不是一块平板，是**套在球上的壳**（地球：地面半径 6,360 km、大气顶 6,460 km，即 100 km 厚）。这带来两个后果：

1. **地平线方向的斜程远大于天顶**。按 Chapman 近似，斜程/垂直程 $\approx \sqrt{\pi R/2H}$；取 $H=8$ km、$R=6360$ km 得**约 35 倍**（本文按公式估算，非论文数字）。**这就是日落变红的定量来源，也是"地平线附近的纹素最贵"的原因**——所以 [[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 的 Sky-View 表要用 $\sqrt{|l|}$ 把纹素往地平线压。
2. **太阳可以落到地平线以下**，而大气仍然被照亮——地球投在大气里的阴影、晨昏线、以及"日落后天还亮着"全在这一件事里。**这也是 4D LUT 必须把太阳天顶角当维度的原因**（[[Bruneton 2008 — Precomputed Atmospheric Scattering|Bruneton 2008]]）。

### 成本结构：三族，三种"省法"

| 族 | 省的是 | 每像素/每帧的账 | 天花板 |
|---|---|---|---|
| **ray march** | 不省，直接积 | $O(n)$ 采样 × 每采样一次朝日透射率 | 分辨率和步数直接买画质 |
| **解析拟合** | 把整条积分拟合成函数 | $O(1)$，零内存 | 只有地面视角、只有一种大气 |
| **LUT** | 把"贵的那部分"预存 | $O(1)$ 查表（多张表） | **更新贵**；伪影 |
| **解析 + 级数**（2020） | 把多阶散射变成公比 | $O(1)$ 查表 + 0.17 ms 重建 | 各向异性相函数、稠密大气色相 |

## Prerequisites

- [[Participating Media]]：吸收/散射系数、消光、相函数、透射率——本篇是它在"球壳 + 两种粒子"上的特化。
- [[Rendering Equation]]：沿视线的积分形式（辐射传输方程）是同一个东西。
- [[Linear Transport Theory]]：多阶散射的级数结构（今天在 [[Subsurface Scattering]] 与本文各见一次）。

## Historical Evolution

| 年 | 工作 | 突破 | 遗留 |
|---|---|---|---|
| 1950 | Chandrasekhar《Radiative Transfer》 | 理论完备 | 不实时 |
| **1982** | **[[Blinn 1982 — Light Reflection Functions for Simulation of Clouds and Dusty Surfaces\|Blinn（平板云）]]** | **两项形状第一次出现**：$B+\mathrm{Tr}\cdot B_{\rm bkg}$（单次散射 + 透射率）；衬着 Hapke–Irvine 极限 | 平板、常数密度、无球壳几何 |
| **1993** | **Nishita et al.** | 首个物理正确的天空渲染（球壳 + ray march + 单散射） | 少算约 1/3 辐照度 |
| 1996 | Nishita et al. | 加二阶散射（8 方向体素） | 换太阳角要重烘 |
| 1999 | Preetham et al. | 解析拟合，$O(1)$ | 实测高估 2×；只有地面 |
| 2005 | O'Neal / Haber | GPU 逐顶点 / 球坐标全阶 | 单散射 / 一个太阳角一张表 |
| **2008** | **Bruneton & Neyret** | **4D LUT（含太阳角），$O(1)$ 运行时** | **改一次 250 ms** |
| 2009–13 | Elek / Yusov | 任意大气、光谱、参数化 | 仍 $O(n)$ 更新 |
| 2016 | Hillaire（Frostbite） | 时间切片 | 更新延迟 |
| **2020** | **Hillaire** | **几何级数替迭代：0.17 ms/帧** | 各向异性、稠密色相 |
| 2020+ | 路径追踪 + 降噪 | 直接用 PT 出天空 | 帧预算 |

**一条被反复验证的规律**：**这个领域的每一次跃迁，都不是"算得更准"，而是"把某个量的更新变便宜"**——从"每帧重算"到"预计算一次"，再到"每帧可重建"。**精度在 2017 年就被量到了参数地板（见下）。**

## 另一半：把它倒过来解（2026-11-20 补）

上表全在记**正向**——给定几何与大气参数，把天画出来。所以这张表读起来像一条"越算越省"的下降曲线。但同一套积分还有另一半，2002 年才被正面处理：**给定像素值，能不能把介质和场景反推出来？**

[[Narasimhan 2002 — Vision and the Atmosphere|Narasimhan & Nayar 2002]] 做的是这件事：把上面的积分项整体打包成一个"大气光" $A$（airlight），写成 $A=A_\infty(1-e^{-\beta d})$，于是每个像素变成两个机制的线性组合

$$E=p\,\hat D+q\,\hat A$$

$p$ 走直射（衰减后的物体色），$q$ 走散射（介质自己发的光）。**这一步的价值不在公式，在它与上面正向路线的三处结构性差异**：

| | 正向（本页上半） | 反向（2002） |
|---|---|---|
| 未知量 | 全是给定的 | 表面 $L_o$ + 介质 $\beta$ + 距离 $d$，**比方程多** |
| 为什么能解 | 不需要解，只要积出来 | 靠**两个机制颜色方向不同**：二色平面是 2 维子空间，$p,q$ 恰好 2 个未知 |
| 均匀假设 | 可有可无（球壳、指数剖面都行） | **必须有**：$\beta$ 要能提出积分号，$1-e^{-\beta d}$ 才成立 |

**能解的条件窄得惊人**：$\beta$ 一旦随高度变化（真实大气就是这样），$A=A_\infty(1-e^{-\beta d})$ 立刻失效；一旦距离 $d$ 未知（单图），$p,q$ 解出来也只是"某个 $d$ 下的解"。2002 年的对策是**多图**——用两幅不同天气/不同距离的图做 DOT（光学厚度差）比值，把 $d$ 消掉。这是"用拍摄条件换可辨识性"的最早范式。

### 它明确丢掉的那一项，十六年后成了主项

论文在推导 Allard 定律时写了这样一句（§2，原文）：

> "We have ignored the flux scattered in the forward direction (towards the observer) by each particle. Fortunately, this component is small in vision applications since the solid angles subtended by the source and the observer with respect to each other are small (see Middleton (1949))."

中文：**"我们忽略了每个粒子朝观察者方向前向散射的光通量。幸运的是，在视觉应用中这一项很小，因为光源与观察者相对彼此张开的立体角很小。"**

同一节还补了一条：**"在重雾等情况下，由于大气粒子的显著多阶散射，指数律可能不成立……本文不建模多阶散射造成的模糊。"**

这两句话是**整个去雾/去水文献的地基，也是它最薄弱的地方**。它们在大气里成立的理由是一个几何量——光源与观察者的张角。搬到水下，这个理由**当场失效**：

| 条件 | 大气（2002 的假设） | 水下（2026 的现实） |
|---|---|---|
| 光源张角 | 太阳 ≈ 0.5° | 太阳 + **整个天空穹顶**，等效半球的量级 |
| 视场 | 窄（长焦、小孔） | 广角相机 90°–180°；**头显更宽** |
| 介质密度 | 气溶胶 1.2 km 标高 | 水 3–10 m 能见度，**稠密两个数量级** |
| 前向散射占比 | 小量，可丢 | **与直射项同阶，甚至主导**（[[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting|NIRJALA 2026]] 的三项模型里它是独立一项） |

**所以"把去雾算法搬到水下"不是参数没调好，是前提被抽掉了。** 这条链在 2026 年才被正式补上：[[Underwater Rendering]] 记的三项模型（直达 + 后向 + **前向**）就是把 2002 年丢掉的这一项放回积分，代价是"两个未知标量"变成"三个机制、且前向项与直射项都乘 $J$"——**可辨识性从"靠颜色方向不同"退化成"图像只约束两者之和"**（详见该页）。

### 对游戏研发的两点意义

1. **正向廉价、反向病态，这个不对称是结构性的**。天空一张 LUT 就够（0.17 ms），因为正向只需"积出来"；而任何"从画面反推大气"的需求（拍照模式复位、HDR 环境光估计、截图转场景）都撞在同一堵墙上：**加法项与乘法项混在一起，且衰减在 $1/T=e^{+\beta r}$ 上指数放大**。
2. **"忽略小量"是这份文献里最贵的四个字**。今天的实时体积雾、参与介质渲染都在吃同一个亏的变体：单散射假设在浓雾/体积云里崩、各向同性假设在太阳附近崩（见本页 Learning Gap 1–2）。**判据：任何"某一项很小可以丢"的论证，先问它小是靠几何还是靠参数——靠几何的，换个场景就没了。**

## Important Papers

- [[Bruneton 2008 — Precomputed Atmospheric Scattering|Bruneton & Neyret 2008]] — 方法基石，4D LUT，十多年的事实标准。
- [[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] — 生产级换代，$O(1)$ 多阶散射，UE 的 `SkyAtmosphere`。
- [[Bruneton 2017 — A Qualitative and Quantitative Evaluation of 8 Clear Sky Models|Bruneton 2017]] — **量尺**：八模型同参数对比，给出"精度已到参数地板"的结论。
- [[Narasimhan 2002 — Vision and the Atmosphere|Narasimhan & Nayar 2002]] — **逆问题的那一半**：二色大气散射模型、DOT 消距离、明确丢掉前向散射（后被水下场景推翻）。
- [[2026-11-13-Leonard — Real-Time Volumetric Light Transport Inference without a Path|Leonard 2026]] — 体渲染的神经路线（可视为"把整条积分交给网络"的极端版本）。

## Related Concepts

- [[Participating Media]] — 上位概念；本篇补它缺的"球壳 + 双粒子 + 高度剖面"。
- [[Subsurface Scattering]] — 同一族近似的另一个域（**二阶 + 级数顶无穷阶**，毛发/皮肤/大气三处独立发现）。
- [[Sparse Volumetric Representation]] — 云与雾的表示（大气本身通常不需要稀疏结构，它是解析的指数剖面）。
- [[Temporal Stability and Artistic Intent]] — 时间切片造成的"太阳动了天没动"，是本页在工程上的主要敌人。
- [[Underwater Rendering]] — 同一套积分换介质：水把"前向散射可忽略"这条前提抽掉，三方模型由此成立。
- [[Inverse Rendering]] — 反向路线的总纲；本页"另一半"是它的介质支线（第四条支线）。

## Technologies

- **UE `SkyAtmosphere`**：[[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 的直接产品化，四张表 + 体积阴影 ray march。
- **Unity HDRP `Physically Based Sky`**：公开文档声明基于 [[Bruneton 2008 — Precomputed Atmospheric Scattering|Bruneton & Neyret 2008]]。
- **Frostbite / Decima / RDR2**：Hillaire 2016、de Carpentier 2017、Bauer 2019 三条 GDC/SIGGRAPH 课程路线。

## Game Applications

- **ToD 系统**：太阳角度一维扫描，天空、空气透视、间接光全跟着走——**这是"一张 LUT 服务全部时刻"的最典型用法**。
- **空气透视做距离提示**：开放世界里"远山发蓝"是最便宜的空间深度线索，成本是两个查表。
- **高度雾**：本质就是"把大气散射限制在近地几十米"，同一套公式换剖面。
- **飞行/太空游戏**：唯一必须"从地面到太空连续"的品类，也是 4D LUT 与 2020 方法的分水岭场景。

## Personal Knowledge

- **Normal**（2026-11-16 建立）：三个量、两种粒子、一个嵌套积分——结构清楚；三族算法的成本结构清楚。
- **Easy**：为什么蓝、为什么红、为什么远山发灰（都能用"光程 + $\lambda^{-4}$"一句话解释）。
- **Hard**：多阶散射的**方向性**（一旦要求"前向峰也参与多阶"，各向同性假设就失效，误差 6.7 倍）；以及 $f_{ms}$ 的解析积分与 $[0,1]$ 归一化。

## Learning Gap

1. **气溶胶相函数**：所有模型在太阳 20° 以内都不准，根子在这里；而游戏里根本拿不到实测相函数——**这块是"知道但做不到"的典型**。
2. **多阶散射的方向性**：现在的近似都是"二阶以上各向同性"，而太阳附近恰恰最各向异性。**补法（前两阶各向异性 + 之后各向同性）在公开文献里没看到干净实现。**
3. **体积阴影与 LUT 的结构冲突**：LUT 不支持沿视线的积分采样，所以山影只能 ray march。**这是结构问题不是实现问题。**
4. **空间变化**（云、局地雾、火山灰）——所有 LUT 方法都假设"只随高度变化"。
5. **反向的可辨识性**（2026-11-20 补）：正向可以随便加阶数，反向连"两项相加"都分不开。二色平面靠颜色方向救了一次，但靠的是"介质单色 + 均匀"；一旦介质有前向散射、有波长相关吸收，这个 2 维子空间就塌了。**我还没读到任何一篇把"正向越算越准"和"反向越算越不准"放在同一个框架里解释的文献。**

## Next Step

1. **反读代码**：`ebruneton/precomputed_atmospheric_scattering`（2017 重写版）的 `model.h` 与预计算 pass——比找 2008 年的 PDF 更快，且注释齐全。
2. **做一次可验证的小实验**：把 UE `SkyAtmosphere` 的多阶散射关掉（或把阶数从全阶降到 1），在日落场景里截图对比——**亲眼确认"多阶散射改的是色相不只是亮度"**（对应 [[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 图 11）。
3. **接到既有预算体系**：把"天空 pass"作为固定成本项写进 [[Budget Units]]（0.14 / 0.31 / 1.0 ms 三档），并注明"体积阴影是唯一的 3 倍开关"。
