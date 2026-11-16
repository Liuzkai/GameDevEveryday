---
type: concept
user_level: Normal
aliases: [Participating Media, Volumetric Rendering, 参与介质, 体积渲染, Radiative Transfer]
prerequisites: [Rendering Equation]
first_introduced: "Chandrasekhar 1960《Radiative Transfer》（天体物理）；1980s 进入图形学"
---

# Participating Media

> **这是你 Easy 域（VFX / OverDraw 预算）与渲染理论之间最大的一个缺口节点。**

## Definition

光在**穿过**一个体积时会被吸收、散射、自发射——这种"会参与光传输的介质"叫参与介质（participating media）。与之相对的是**真空**：光穿过时什么也不发生，所以普通渲染只需要算表面。

控制方程是**辐射传输方程（RTE, Radiative Transfer Equation）**，它是 [[Rendering Equation]] 的体积版：

$$(\omega\cdot\nabla)L(x,\omega) = -\sigma_t L + \sigma_s\int_{S^2} f_p(\omega'\to\omega)\,L(x,\omega')\,d\omega' + \sigma_a L_e$$

三项各管一件事：

| 项 | 系数 | 含义 |
|---|---|---|
| 消光（extinction） | $\sigma_t = \sigma_a + \sigma_s$ | 沿路"离开这条射线"的光（吸收 + 散射走） |
| 内散射（in-scattering） | $\sigma_s$ + 相位函数 $f_p$ | 从别的方向散射进这条射线的光 |
| 自发射（emission） | $\sigma_a L_e$ | 介质自己发光（火焰、等离子） |

**单次散射反照率（albedo）** $\alpha = \sigma_s / \sigma_t$ 决定介质的"性格"：$\alpha\approx 1$ 是云/牛奶（光一直在里面弹），$\alpha\approx 0$ 是黑烟（进来就没了）。

## Core Principle

两个直觉必须建立：

1. **光学厚度 / 透射率是指数衰减的**：$T = e^{-\int \sigma_t\,ds}$。这就是 Beer–Lambert 定律。**"两层半透明叠加 ≠ 两次独立计算"**——透射率相乘，但因为指数，厚度的非线性会让"堆叠顺序"和"密度分布"都以非线性方式影响最终结果。这是体积渲染一切工程麻烦的根源。
2. **散射是各向异性的**：相位函数 $f_p$ 描述"光进来后偏向哪"。常用 **Henyey–Greenstein**，一个参数 $g$：$g>0$ 前向散射（雾、云，逆光有光晕），$g<0$ 后向散射，$g=0$ 各向同性。**$g$ 是烟雾美术手感的最重要一个参数。**

### 为什么它贵

体积渲染的成本 ≈ **采样步数 × 每步的光照查询**：

- 步数不够 → banding / 走样；
- 每步要查阴影（shadowed 体积雾）→ 成本乘以光源数；
- 半透明要按深度排序、要和 deferred 管线打架。

这直接解释了为什么 **OverDraw 是六大实测指标里最"贵"的那个**——OverDraw 高意味着同一像素被多层半透明反复着色，而半透明的着色成本正是体积渲染成本的近亲。

### 第四条便宜路：不算（2026-11-13 补）

上面那笔账 $\text{步数}\times\text{每步查询}$ 给出了三条省钱的路，四十年基本都在这三条里：**减步数**（froxel / 时域重投影）、**减每步**（阴影复用、低阶相位）、**算完再修**（降噪）。第三条隐含一个假设：**那个标量（辐亮度）有一个可估的期望，只是估计量噪声大**。

2026 年出现了第四条：**这个估计量不必存在**。[[2026-11-13-Leonard — Real-Time Volumetric Light Transport Inference without a Path|HaloRT]] 在 1 spp 的噪声图像上抽一组主射线特征（光学厚度、交互深度、朝光透射率、环境光照采样、介质属性），直接回归**解析积分的值**——把"估计"换成"推理"，把六十四次采样换成一次。

**代价不是白拿的，它换了个假设**：

| | 降噪派 | 无路径推断 |
|---|---|---|
| 依赖的假设 | **噪声不变性**（信号不随 spp 变，噪声可分离） | **特征充分性**（这几个量足以定出积分值） |
| 假设坏了会怎样 | 过平滑 / 涂抹 / 鬼影（可见、可调） | **系统性偏差**（不可见、不随 spp 消失） |
| 成本 | $N$ spp × 每 spp 代价 + 降噪网络 | 1 spp + 一次前向（512²：10.66 ms vs 24.76 ms） |

**这条对判据层的意义**：本页的成本公式现在应该写成"**先问这条路上有没有一个估计量，再问它要采几次**"。而"**这个估计量必须存在吗？**"这个问题，与 [[Differentiable Rendering]] 页的"**它能不能不存？**"是同族问法——**都是把一个被当作前提的东西降级为选项**。

**别急着信**：HaloRT 自己承认 (A2) 是被实测验证的、(A1) 只是"可测"而非"已证"，而它相对降噪器仍有 1.44 dB 的缺口——**那 1.44 dB 很可能就是 (A1) 不成立的证据**。另外它的适用范围只有单一云数据集、HG 相位、非发光、单一介质、无表面——**真实的 UE 体积雾场景（多光源 + 高度雾 + 粒子）还没被试过**。

### 第五条路：换一个方程（2026-11-14 补）

前四条路都在同一个方程里省钱：**减步数 / 减每步 / 算完再修 / 不算**。还有第五种，也是唯一一种能拿到数量级的：**换掉方程本身**。

判据是"这个方程里有哪些自变量"。以 [[Jensen 2001 — A Practical Model for Subsurface Light Transport|Jensen 2001]] 为例：当散射次数足够多时，辐射度分布趋于各向同性，于是**方向这个自变量被抹掉了**——只需保留两个矩（通量 $\phi$ 与通量向量 $\mathbf E$），RTE 就退化成扩散方程 $D\nabla^2\phi=\sigma_a\phi$。**扩散方程里没有"散射次数"这个变量，这就是 250× 的全部来源**：不是把每一步算得更快，是**那个要算很多步的东西不存在了**。

代价随之而来，而且可以精确地说：**扩散只对"多次散射主导 + 半无限均匀"成立**。于是——

$$\text{单次散射自由程}=\frac{1}{\sigma_t}\quad(\text{短})\qquad\text{扩散长度}=\frac{1}{\sigma_{tr}}=\frac{1}{\sqrt{3\sigma_a\sigma'_t}}\quad(\text{长，}\sim10\times)$$

**两个长度差一个量级，这是本页与 [[Subsurface Scattering]] 的分界点**：体积渲染里没有"出去"这件事（光在介质里一直走到被吸收或逃逸到无穷远），而次表面散射里**光必须从一个有界的表面出去**——**边界条件是这两个域的全部差别**。同一条方程，一个有边界、一个没有。

**顺手一条工程判据**：$\sigma'_s=\sigma_s(1-g)$ 里的 $(1-g)$ 说的是"**前向散射不算散射**"。所以调体积雾时把 $g$ 从 0 拉到 0.8，有效散射掉到 1/5、雾"变透"，**但消光一点没变**。**"变透"与"变亮"是两件事，混淆它们的人多半栽在这个因子上。**

### 第六条路：把"无穷阶"折成一个公比（2026-11-16 补）

前五条路都在同一个维度上省钱：**少采样**（步数）、**少查询**（每步）、**事后修**（降噪）、**不估**（推断）、**换方程**（扩散）。第六条不在采样维度上，它**换掉的是"要算多少阶散射"**。

[[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 的做法可以一句话说完：

$$\Psi_{ms} = L_{2nd}\cdot\frac{1}{1-f_{ms}}$$

**二阶散射的答案 × 一个几何级数的和。** 前提是三条假设：二阶以上各向同性；**"邻域内所有点收到的照度与我相同"**（平均自由程 ≫ 邻域尺度）；忽略可见性。$f_{ms}\in[0,1]$ 是被积出来的无量纲转移因子，级数因此收敛。

**它和前五条的分工很清楚**：

| 路 | 动的是 | 典型收益 | 代价 |
|---|---|---|---|
| ①~④ | **采样**（spp / 步数 / 噪声） | 几倍到几十倍 | 噪声、偏差、鬼影 |
| ⑤ 扩散 | **方程**（抹掉方向维度） | **250×**（Jensen） | 只对多次散射主导 + 均匀介质成立 |
| **⑥ 级数** | **阶数**（$O(n)\to O(1)$） | **约 1,470× 的表重建**（250 ms → 0.17 ms） | 各向异性相函数失效（$g{=}0.8$ 时误差 6.7 倍） |

**最容易忽略的一点：这条路的先例不在大气，在毛发。** Zinke 2008 的 dual scattering 用同一个"二阶 × 公比"顶掉毛发的无穷阶散射；Hillaire 2020 明写灵感来自它。**同一个数学结构，在毛发、皮肤（偶极子扩散）、大气三处被独立发现——说明它不是某个域的技巧，是"无穷阶近似"的通用形状。**

### 边界形状：平板、有界体积、球壳（2026-11-16 补）

同一个 RTE，三种边界，工程上是三套完全不同的东西：

| 边界 | 典型场景 | 关键几何量 | 工程后果 |
|---|---|---|---|
| **平板/半空间** | 高度雾、贴地雾 | 视线与水平面的夹角 | 解析式（Exp/Exp2 雾）直接可用 |
| **有界体积** | 云、烟、局部雾 | 进入/离开距离 | 需要稀疏表示 + 排序（见 [[Sparse Volumetric Representation]]） |
| **球壳** | 行星大气 | **地平线方向的斜程** | 纹素必须非线性分配，且多一个"太阳天顶角"维度 |

球壳这一格值得单独记两个数：

1. **地平线方向的斜程约为天顶的 $\sqrt{\pi R/2H}$ 倍**——取 $R=6360$ km、标高 $H=8$ km 得**约 35 倍**。**这就是"日落变红"的定量来源，也是为什么天空的 LUT 必须把纹素往地平线压**（[[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 用 $v=0.5+0.5\,\mathrm{sign}(l)\sqrt{|l|/(\pi/2)}$）。
2. **太阳可以落到地平线以下，大气仍被照亮**（地球投在大气上的阴影、晨昏线）。**这一条把 LUT 的维度从 3 抬到 4**——[[Bruneton 2008 — Precomputed Atmospheric Scattering|Bruneton 2008]] 的全部复杂度都来自这个决定。

> **判据：如果介质是"无限大且只随高度变化"，它就是球壳问题，别用有界体积那套（froxel / 稀疏网格）；如果它有明确的进出面，它就是有界体积问题，别硬套大气公式。** 两者共用 RTE，但**预计算的结构、维度和更新代价完全不同**。

## Prerequisites

- [[Rendering Equation]]（表面版；RTE 是它的体积推广）
- 概率与指数分布直觉（自由程采样：距离按 $e^{-\sigma_t s}$ 分布抽样）
- 蒙特卡洛积分（透射率估计、俄罗斯轮盘赌）

## Historical Evolution

```text
Chandrasekhar 1960 —— RTE 在天体物理中成型
        ↓
Blinn 1982 —— 把参与介质引入计算机图形学（"Light Reflection Functions for Simulation of Clouds and Dusty Surfaces"）
        ↓
Kajiya & Von Herzen 1984 —— 体积光线步进（Ray Marching）
        ↓
1992  Farrell et al.（医学物理）—— 偶极子模型测组织光学参数（图形学之外）
        ↓
1995  Stam —— 扩散近似做云的多次散射（★ 扩散第一次进图形学，是云不是皮肤）
        ↓
1999  Dorsey et al. —— 光子图做完整次表面（风化石头）
        ↓
2001  Jensen et al. —— ★ 偶极子进图形学（属 [[Subsurface Scattering]]，**不是本页**）
        · 2026-11-14 修正：此前这一行笼统写作 "Jensen / Premože 1990s —— 光子图、扩散近似"，
          把三件不同的事（1992 医学偶极子 / 1995 云扩散 / 2001 图形学 BSSRDF）压成了一个十年
        ↓
2000s —— 实时近似：fog volumes、light shafts、解析雾（Exp/Exp2）
        ↓
2010s —— 生产级：Froxel / 体积雾（[[]]  clustered volume）、Temporal reprojection 降噪
        ↓
Wronski / Siggraph 2014-2019 —— UE4 体积雾、寒霜的 froxel volumetric；TAA 时序重投影把步数降下来
        ↓
2020s —— 神经参与介质：神经辐射场/GS 表示体积、神经降噪、神经超分补偿
        ↓
★ 2026 —— 随机几何统一：表面与介质不再是两套模型
         （Seyb et al. SIGGRAPH 2024 → [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]]）
              ↓
★ 2026 —— 无路径推断：把"估一个标量"换成"从特征推一个标量"
         （[[2026-11-13-Leonard — Real-Time Volumetric Light Transport Inference without a Path|HaloRT]]）—— 降噪的前提被取消
```

## Important Papers

| 论文 | 角色 |
|---|---|
| [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]] | 把表面与介质打通：从 GPIS 统计量解析出 RTE 参数，并把 GGX / SGGX / Beckmann 统一为特例 |
| Seyb, d'Eon, Bitterli, Jarosz — SIGGRAPH 2024 | 统一理论的起点（微面 ↔ 介质连续体） |
| [[2026-11-13-Leonard — Real-Time Volumetric Light Transport Inference without a Path\|HaloRT 2026]] | **"取消估计量"路线**：1 spp 特征 → 直接回归积分值；把降噪的前提（噪声不变性）换成特征充分性 |
| [[2026-09-15-Grid-Free Monte Carlo for Time-Dependent Diffusion]] | 求解器侧：把时间依赖热方程也纳入无网格 MC |
| [[2026-09-14-Gaussian Light Transport]] | 同周"删掉一个求解器中间产物"的另一例 |
| [[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique\|Hillaire 2020]] | **第六条路（级数）**的代表：$O(1)$ 多阶散射，让 LUT 每帧可重建；代价是各向异性相函数 |
| [[Bruneton 2008 — Precomputed Atmospheric Scattering\|Bruneton 2008]] | 球壳边界的标准解：4D LUT，$O(1)$ 运行时，$O(n)$ 更新（250 ms） |
| [[Bruneton 2017 — A Qualitative and Quantitative Evaluation of 8 Clear Sky Models\|Bruneton 2017]] | **量尺**：证明最准的 CG 模型已撞到"参数地板"，精度不再是积分问题 |

## Related Concepts

- [[Rendering Equation]]（表面版 ↔ RTE 体积版）
- [[Subsurface Scattering]]（★ **同一条方程，一个有边界一个没有**——那是本页与它的全部差别；2026-11-14 建页）
- [[Microfacet Theory]]（**对偶关系**：微面 = 随机表面的统计；参与介质 = 随机体积的统计；2026 的工作证明它们是同一套随机几何的两端）
- [[Neural Rendering]]（神经体积表示是当下主流路线）
- [[Gaussian Splatting]]（同属体积/半透明渲染，同样受排序与 OverDraw 支配）

## Technologies

- **UE Volumetric Fog**（froxel grid + 时序重投影 + 降噪）
- **Niagara 流体/烟雾**（你的日常工具：粒子驱动的体积近似）
- **Exponential Height Fog / Local Fog Volume**（便宜的解析近似）
- **OpenVDB**（影视侧标准体积资产格式；2026 的 GPIS 工作可以让它直接当概率表面渲染）

## Game Applications

- **VFX**：烟、火、尘、体积光、能量场——Niagara 的日常，也是 OverDraw 预算的主要来源；
- **氛围**：体积雾是"空间感"最便宜的来源之一；
- **玩法**：体积雾可被用于遮挡与视线控制（Arm 在移动端 MegaLights 演示里把"光=可交互提示"当作核心玩法语言）。

## Personal Knowledge

**Normal**。判断依据：你每天都在做 Niagara 粒子与半透明，**工程侧直觉是 Easy 的**（知道 OverDraw 会炸、知道排序会出错、知道移动端要砍步数），但**RTE 的数学形式、散射参数（σ/α/g）的物理含义、以及"表面与介质统一"这条理论线是新的**。

本次（2026-09-16）建库的直接理由：补齐你 Easy 域（VFX/OverDraw）与渲染理论之间的缺口节点，并让今日的 GPIS 论文有地方挂。

## Learning Gap

1. **RTE 三项的物理含义与量纲**（σ 的单位是 1/长度——"每米发生多少次相互作用"）；
2. **自由程采样**：为什么距离按指数分布抽样，以及为什么这是无偏的；
3. **α 与 g 对观感的具体影响**（这两个参数就是烟雾美术的"手感旋钮"）。

## Next Step

不要先读 RTE 推导。建议的顺序：

1. 打开 UE 的 Exponential Height Fog / Volumetric Fog，把 scattering、albedo、extinction 三个参数各拉一遍，看画面往哪走；
2. 再回来看 $\sigma_s$、$\alpha$、$g$ 分别对应哪个滑杆——**把公式和手感对上**；
3. 这一步做完，本笔记可从 Normal 标 Easy。

## Mastery Criteria

1. 说出 RTE 三项（extinction / in-scattering / emission）各自对应什么物理过程；
2. 解释为什么透射率是指数的、以及为什么"两层叠加"不能简单相加；
3. 说出 albedo α 与相位参数 g 分别控制什么观感；
4. 指出 UE 体积雾里哪几个参数对应 $\sigma_s$ / $\sigma_a$ / $g$；
5. 说明为什么体积渲染成本高，并把成本拆成"步数 × 每步光照查询"，再映射到 OverDraw。
