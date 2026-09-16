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
Jensen / Premože 1990s —— 光子图、扩散近似（diffusion / dipole）
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
```

## Important Papers

| 论文 | 角色 |
|---|---|
| [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]] | 把表面与介质打通：从 GPIS 统计量解析出 RTE 参数，并把 GGX / SGGX / Beckmann 统一为特例 |
| Seyb, d'Eon, Bitterli, Jarosz — SIGGRAPH 2024 | 统一理论的起点（微面 ↔ 介质连续体） |
| [[2026-09-15-Grid-Free Monte Carlo for Time-Dependent Diffusion]] | 求解器侧：把时间依赖热方程也纳入无网格 MC |
| [[2026-09-14-Gaussian Light Transport]] | 同周"删掉一个求解器中间产物"的另一例 |

## Related Concepts

- [[Rendering Equation]]（表面版 ↔ RTE 体积版）
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
