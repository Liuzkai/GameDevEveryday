---
type: paper
title: "Light Reflection Functions for Simulation of Clouds and Dusty Surfaces"
authors: ["James F. Blinn"]
year: 1982
published: "1982-07（SIGGRAPH '82）"
venue: "SIGGRAPH '82, Computer Graphics 16(3):21–29"
url: "https://www.microsoft.com/en-us/research/wp-content/uploads/1982/01/p21-blinn.pdf"
code: ""
project_page: ""
category: [Participating Media, Volumetric Rendering, Atmospheric Scattering, Rendering History, Shading Models]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（闭式的形状被此后所有指数雾、软粒子、云 billboard 直接继承；原文本身是 1982 年的 CPU 光栅+扫描线实现）"
user_level: Normal
status: unread
aliases: [Blinn 1982, Light Reflection Functions, 云的光反射函数, 尘埃表面, Hapke-Irvine, Lommel-Seeliger, 单次散射闭式]
tags: [participating-media, volumetric-rendering, single-scattering, phase-function, classical, rendering-history]
---

# Blinn 1982 — 云的第一个闭式

## TL;DR

**把云写成"一层均匀小球 + 只散射一次 + 遮挡是泊松过程"，整条视线上的积分就能积出来。**

积出来的结果只有四个量：

$$B=\underbrace{\omega}_{\text{粒子反照率}}\ \underbrace{\varphi(\alpha)}_{\text{相函数}}\ \underbrace{\frac{\mu_0}{\mu_0+\mu}}_{\text{两段路径}}\ \underbrace{\left[1-e^{-\tau\left(\frac{1}{\mu_0}+\frac{1}{\mu}\right)}\right]}_{\text{沿厚度积一次}}$$

- $\mu_0=\mathbf N\!\cdot\!\mathbf L$、$\mu=\mathbf N\!\cdot\!\mathbf E$、$\alpha$ 是相位角（$\mathbf L$ 与 $\mathbf E$ 的夹角）；
- $\tau=n\pi\rho^2T$ 是**光学厚度**——注意它只由"粒子数密度 × 截面 × 厚度"组成，$n,\rho,T$ 三个量在公式里**只以这个乘积出现**。

> **一句话判据（今天最该记）：你的雾/云 shader 里那个 $1-e^{-\text{density}\cdot d}$，就是这篇 1982 年的式子。** 认出它，就同时认出了它的全部假设——**只散射一次**。

## Problem

1982 年图形学只有两类反射函数：**Lambert**（漫反射，与观察方向无关）与 **Blinn-Phong**（表面高光）。两者都假设"光打到**一个面**然后离开"。

云、雾、火焰、尘埃、行星环不属于这一类：光**进入**一个体积，在里面可能被吸收、被改向、被再散射，然后才从另一个方向出来。当时没有描述这件事的反射函数，只有零散的特例（Max 用解析边界定义云、Voss 用分形密度生成云），而它们都停在"密度怎么造"，没有回答"光怎么出来"。

Blinn 把问题切成两半，并且只做第二半：

| 子问题 | 归谁 |
|---|---|
| 密度场 $\rho(x)$ 怎么来（噪声、分形、模拟） | **不做**（原文明确声明） |
| 光与这个密度场怎么作用 | **本文** |

选择土星环作为第一个目标不是随便挑的：环是**平板状的粒子层**，几何最简，可以把全部注意力放在光传输上，而且有 Voyager 1 的观测数据可以对照。

## Historical Context

这篇的位置很特殊：它比 [[Rendering Equation]]（1986）**早四年**，比体积渲染的正式框架（Max 1995 的光学模型分类）**早十三年**。也就是说——

> **在"渲染方程"这个词出现之前，体积的那一半已经有人用一个闭式解决了。**

它的直系祖先是 Chandrasekhar 1960 的《Radiative Transfer》与行星科学里的 Hapke / Irvine 工作（粉尘覆盖的表面、稠密大气）。Blinn 做的事是把这套天体物理的公式**搬进图形学**，并且**只搬能积出来的那一档**。

## Previous Work（三个邻居，各缺什么）

| 工作 | 做了什么 | 缺什么 |
|---|---|---|
| Lambert / Blinn-Phong | 表面反射的经验/半经验模型 | 没有"穿过"这件事；无法表达"越厚越白、越厚越不透明" |
| Max 1983（解析边界云） | 用解析函数定义云的边界 | 光传输仍是经验近似；没有相位函数 |
| Voss 1983（分形密度） | 用分形生成密度，视觉上很像云 | 只对"平面平行层"成立；没有一般的视线积分 |
| Hapke / Irvine（行星科学） | 粉尘表面的反射律（后来称 Lommel-Seeliger） | 只给 $\tau\to\infty$ 的极限，没有有限厚度的闭式 |

**Blinn 的增量**：给有限厚度一个闭式，并且把**相位函数**正式带进图形学的反射模型。

## Core Idea — 三个假设换一个闭式

整篇论文的结构可以压成一张表：**每个假设买一样东西，也欠一笔账**。

| 假设 | 买到 | 欠下 |
|---|---|---|
| **① 单次散射**：光只与一个粒子作用一次 | 亮度 ∝ $\omega$，积分变成线性的 | **只在小反照率 $\omega$ 时成立**；云/雪/牛奶（$\omega\approx1$）**结构性少算** |
| **② 粒子远大于波长**：几何光学，无衍射 | 相函数只由"投影面积随角度变化"决定，可以解析 | 表达不了 Mie 前向尖峰；小颗粒（瑞利）要另换函数 |
| **③ 遮挡是泊松过程**：$P(0;V)=e^{-nV}$ | 两段路径的衰减都是指数，可积 | 忽略粒子体积重叠（$D$ 大时错）；忽略两个圆柱的交叠 |

**第三个假设是这篇论文最值钱的技术动作**，值得单独说：

它不追踪"这条光线撞到了谁"，而是问"**这条光线的通道里，一个粒子都没有的概率是多少**"。答案是泊松分布的零项 $e^{-nV}$，其中 $V$ 是**半径 $\rho$、沿光路的一根圆柱**的体积。于是遮挡从"几何求交"变成了"指数"，积分才有可能。

## Technical Approach

### 1. 几何与符号（图 2）

一层厚 $T$、粒子半径 $\rho$、数密度 $n$ 的平板；体积占比 $D=n\cdot\frac43\pi\rho^3$。

$$\mu_0=\cos i=\mathbf N\!\cdot\!\mathbf L,\qquad \mu=\cos e=\mathbf N\!\cdot\!\mathbf E,\qquad \cos\alpha=\mathbf L\!\cdot\!\mathbf E$$

亮度写成 $B=\dfrac{\omega}{\mu}\,\varphi(\alpha)\,S$。**除以 $\mu$ 不是装饰**：斜着看时，一个像素里投进更多的表面积，所以"单位投影面积的亮度"要乘回 $\mu$ 的倒数。

### 2. 两个圆柱（图 4）

一个位于深度 $T'$ 的粒子要被看见，需要两件事同时成立：光能从 $\mathbf L$ 方向进来，且散射光能从 $\mathbf E$ 方向出去。于是它必须落在**两根半径 $\rho$ 的圆柱**（$V_{\text{in}}$ 与 $V_{\text{out}}$）的并集之外。

原文在这里留了一个明确的近似：**两个圆柱的交叠被忽略了**（同一块体积被算了两次）。这个误差只在 $\mathbf E\approx\mathbf L$（相位角接近 0）时显著，§5.2 专门回来修它。

### 3. 积出闭式

把"投影面积 × 粒子亮度 × 单位面积粒子数 × 未被遮挡概率"沿深度积一次：

$$\tau=n\pi\rho^2T\quad(\text{光学厚度})$$

**顶光**（$\mathbf N\!\cdot\!\mathbf L>0$）：

$$B=\omega\,\varphi(\alpha)\,\frac{\mu_0}{\mu_0+\mu}\left[1-e^{-\tau\left(\frac{1}{\mu_0}+\frac{1}{\mu}\right)}\right]$$

**底光**（$\mathbf N\!\cdot\!\mathbf L<0$）：把"从下面进来"的那一段换成另一段路径，

$$B=\omega\,\varphi(\alpha)\,\frac{\mu_0}{\mu_0+\mu}\left(e^{-\tau/\mu_0}-e^{-\tau/\mu}\right)$$

$\mu_0=\mu$ 时有 $0/0$ 的奇点，必须在积分前替换，极限是

$$B=\omega\,\varphi(\alpha)\,\frac{\tau}{\mu}\,e^{-\tau/\mu}$$

**透射率**：$\mathrm{Tr}=e^{-\tau/\mu}$（可再加一项前向散射）。

**合成**：

$$B_{\text{new}}=B+\mathrm{Tr}\cdot B_{\text{bkg}}$$

> **这一行就是 alpha compositing 的 `over` 算子**，也是后来所有大气/水下"两项模型"的祖先：**直达项 × 透射率 + 幕光项 × (1 − 透射率)**。1982 年它是云的合成公式，2002 年它变成 [[Narasimhan 2002 — Vision and the Atmosphere|Narasimhan & Nayar]] 的雾模型，2019 年它变成 [[Akkaynak 2019 — Sea-thru A Method For Removing Water From Underwater Images|Sea-thru]] 的水下模型——**同一个形状，换了三次物理**。

### 4. 极限：Hapke–Irvine 与满月

令 $\tau\to\infty$：

$$B=\omega\,\varphi(\alpha)\,\frac{\mathbf N\!\cdot\!\mathbf L}{\mathbf N\!\cdot\!\mathbf L+\mathbf N\!\cdot\!\mathbf E}$$

这就是行星科学里的 **Lommel–Seeliger 律**（Hapke / Irvine 的形式）。代入 $\mathbf L=\mathbf E$（相位角 0，也就是**满月**）：

$$B=\tfrac12\,\omega\,\varphi(0)\quad\text{——与 }\mathbf N\text{ 无关}$$

**满月的整个圆盘亮度均匀**，不是因为月亮是平的，而是因为**尘埃表面的反射律恰好抵消了球面的余弦衰减**。这是这篇论文最漂亮的一次"公式解释观测"：一个 1982 年的闭式，顺手解释了伽利略以来一直有人讨论的"月球视亮度悖论"。

### 5. 相位函数（第 3 节，一张现成的菜单）

| 函数 | 形式 | 适用 |
|---|---|---|
| 常数 | $\varphi=1$ | 粒子远小于波长（各向同性） |
| 各向异性经验式 | $1+x\cos\alpha$ | 想表达"后向多于前向"但不想讲物理 |
| Lambert 球 | $\frac{8}{3\pi}\left(\sin\alpha+(\pi-\alpha)\cos\alpha\right)$ | 每个粒子按 Lambert 反射，对可见圆盘积分 |
| Rayleigh | $\frac34(1+\cos^2\alpha)$ | 粒子 ≪ 波长（衍射主导） |
| Henyey–Greenstein | $\dfrac{1-g^2}{(1+g^2-2g\cos\alpha)^{3/2}}$ | 最常用；$g$ 是椭圆偏心率 |
| 加权和 | $w_1L(\alpha)+w_2F(\alpha)$ / $w_1\mathrm{HG}(g_1)+w_2\mathrm{HG}(g_2)$ | 粒径不均（Hapke 的月面、Esposito–Lumme 的土星环） |

**两个必须记住的细节：**

1. **HG 的 $g$ 符号约定与今天相反**。原文：$g<0$ 前向散射、$g>0$ 后向散射。今天（PBRT、UE）普遍用 $g>0$ 前向。**照抄 1982 的 $g$ 进现代 shader，会把逆光光晕做成背光**。
2. **$g=0.325$ 是拟合出来的**——原文用它拟合"炉渣这类暗粗糙表面"的实测数据。**这个数不是普适常数，是"暗粗糙粉尘"的一个样本。**

### 6. 两个扩展（第 5 节）

- **密度大时**：把数密度换成 $n'=\dfrac{n}{1-D}$，于是 $\tau'=\dfrac{\tau}{1-D}$。$D\to1$ 时 $\tau\to\infty$，层自动变成不透明——**一个分母就把"松散粒子"推到"实心"**。
- **相位角接近 0 时**：补上两个圆柱的交叠（否则同一块体积被算两次，亮度偏高）。

## Key Contribution

1. **第一个可用于图形的体积反射闭式**：把"云"从几何问题变成**四个标量**（$\omega,\varphi,\tau,\mu$）的问题；
2. **把相位函数带进图形学**：此后"云好不好看"第一次有了一个物理旋钮；
3. **Hapke–Irvine 极限**：给"尘埃表面"一个物理基础——漫反射**不是** Lambert；顺带解释满月的均匀亮度；
4. **$B+\mathrm{Tr}\cdot B_{\text{bkg}}$**：合成算子的原型，也是后来所有"两项介质模型"的骨架。

## Why It Works

三个因子各有出处，缺一不可：

- $e^{-\tau/\mu}$：来自"通道里没有粒子"的泊松概率——**指数不是拟合出来的，是组合计数**；
- $\dfrac{\mu_0}{\mu_0+\mu}$：来自两段路径的长度比（入射段 $\propto1/\mu_0$、出射段 $\propto1/\mu$）——**斜着照、斜着看，两段都变长**；
- $1-e^{-\tau(\cdot)}$：来自沿厚度积一次——**厚度越大越接近饱和**，极限就是 Hapke–Irvine。

> **可迁移的读法**：看到任何"$1-e^{-\text{某物}}$"的雾公式，先问那个指数里是**几段路径**。一段是"穿过"，两段是"进 + 出"。**这正是"体积"与"表面"在公式上的唯一区别。**

## Limitations（六条，前两条是结构性的）

1. **单次散射只在 $\omega$ 小时成立**。云、雪、牛奶的 $\omega\approx1$，光会在里面弹很多次。**这不是精度问题，是模型少了一整类路径**——所以云会被画暗、画"薄"。**这正是 Kajiya & Von Herzen 1984 的出发点**（其摘要明写：给 Blinn 模型一个"高反照率下多次散射"的替代）。
2. **均匀平板、单一粒径、球粒子**。真实的云有空间变化的密度与粒径分布；这篇只有一层常数 $\tau$。**形状必须靠贴图或程序噪声另加**——而一旦密度沿视线变化，闭式就没了（要回到数值积分）。
3. **忽略圆柱交叠**（相位角小的时候偏亮），§5.2 才补。
4. **忽略粒子体积重叠**（$D$ 大时），用 $n/(1-D)$ 近似。
5. **相位函数是"投影面积"的**，没有衍射；小颗粒要换 Rayleigh，大颗粒的强前向尖峰表达不了。
6. **没有偏振、没有各向异性介质、没有自阴影以外的相互照明**（"云不会给云投影"要靠多层近似）。

## Game Development Relevance

**这一页的公式，你每天都在用，只是它换了名字。**

| 引擎里的东西 | 与本文的关系 |
|---|---|
| `Exponential Height Fog` | $\mathrm{Tr}=e^{-\rho d}$ 就是 $e^{-\tau/\mu}$；fog color 乘 $(1-\mathrm{Tr})$ 就是 $B$ |
| 软粒子（soft particles） | 用深度差做指数淡出——同一个 $1-e^{-\cdot}$ 形状 |
| 云 billboard / 粒子云 | $B=\omega\varphi(\alpha)(1-\mathrm{Tr})$，$\varphi$ 用 HG，$g$ 是"云的厚度感"旋钮 |
| 体积光 / god rays | 单次散射 + 沿视线的透射率估计 |
| 水下后处理 | 两项模型（直达 + 幕光），见 [[Underwater Rendering]] |

**三条可以直接用的判据：**

1. **你的雾是"单次散射模型"。** 它对薄雾（$\tau$ 小）几乎精确，对厚云（$\omega$ 高、$\tau$ 大）系统性偏暗。**"云看起来像纸片"不是美术没调好，是少了一整类光路。**
2. **相函数的 $g$ 是手感旋钮，但符号要看实现。** 1982 的 $g>0$ 是后向，今天的 $g>0$ 是前向——**逆光光晕做反了，先查这一条**。
3. **"厚度"与"密度"在闭式里不可分。** 公式只认 $\tau=n\pi\rho^2T$ 这一个乘积。所以"把雾调厚一点"与"把粒子调大一点"在画面上**完全等价**——**想让远处更朦胧但近处不变，改 $\tau$ 没用，得改空间变化**（而那就出闭式了）。

## Unreal Engine Relevance

- **`ExponentialHeightFog`**：解析、单次散射、无相位函数——本文的最简特化（$\varphi=1$，一层常数密度）。
- **`VolumetricFog`（froxel）**：仍是**单次散射**，只是把"一层"换成"一格一格"，并用 shadow map 做每步的透射率。**它的精度上限由本文的假设决定，不由步数决定。**
- **`SkyAtmosphere`**：[[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]] 用"二阶 × 几何级数"补多阶散射——**这是对本文第一条限制的正面回答**（云的多次散射不能靠加步数解决，只能换结构）。
- **Niagara 的 sprite 云 / 体积材质**：几乎都是本文公式的逐粒子版；`Lightness`/`Density`/`Phase` 三个参数与 $\omega,\tau,\varphi$ 一一对应。

## Technology Evolution

```text
1960  Chandrasekhar《Radiative Transfer》    RTE 成型（天体物理）
        ↓
1982  ★ Blinn  平板 + 单次散射 + 泊松遮挡 ⇒ 第一个闭式
                + 相位函数进图形学
                + Hapke–Irvine 极限（满月为什么是均匀圆盘）
        ↓
1984  Kajiya & Von Herzen   任意三维密度 + 沿视线数值积分
                            （明确针对"高反照率多次散射"）
        ↓
1986  Kajiya   渲染方程：把体积写成积分项，表面与介质统一
        ↓
1995  Max 光学模型分类（吸收 / 发射-吸收 / 单次 / 多次）
        ↓
2000s 实时近似：fog volumes、light shafts、Exp/Exp2 雾
        ↓
2010s froxel 体积雾 + 时域重投影（仍是单次散射，靠降噪补样本）
        ↓
2020  Hillaire  级数近似多阶散射（O(1) 重建 LUT）
        ↓
2026  反解：把同一个两项模型倒过来解（Sea-thru / NIRJALA）
```

**一条可以看的规律**：这条线上每次真正的跃迁，都不是"把积分算得更准"，而是**"换一个能积出来（或能解出来）的模型"**。1982 换的是"平板 + 单次"，2020 换的是"二阶 × 公比"，2019/2026 换的是"两项 + 距离函数"。**精度是买不到的，结构才是买得到的。**

## Relationships

### Based On

- Chandrasekhar 1960《Radiative Transfer》——RTE 与单次散射项；
- Hapke / Irvine 的行星表面反射律——$\tau\to\infty$ 的极限（Lommel–Seeliger）。

### Extends

- [[Blinn 1978 — Simulation of Wrinkled Surfaces|Blinn 1978]]（同一作者：把"丢掉几何、只留一项影响"的手法从法线搬到密度）；
- Lambert / Blinn-Phong——从"一个面"扩展到"一层体积"。

### Related

- [[Participating Media]]——上位概念；本文是它的"路径 0：让积分有闭式"；
- [[BRDF]]——Hapke–Irvine 给"尘埃表面"一个非 Lambert 的漫反射律；
- [[Narasimhan 2002 — Vision and the Atmosphere|Narasimhan & Nayar 2002]]——同一个两项形状，从"渲染"翻到"反解"；
- [[Underwater Rendering]]——同一个两项形状，换了系数谱与边界；
- [[Atmospheric Scattering]]——同一个 $1-e^{-\tau}$ 结构，边界从平板换成球壳。

### Followed By

- **Kajiya & Von Herzen 1984**（Ray Tracing Volume Densities）——任意密度 + 高反照率（**本库仍缺全文，见"Notes"**）；
- Max 1986 / Nishita 1996——大气与云的多次散射；
- [[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique|Hillaire 2020]]——级数近似；
- [[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting|NIRJALA 2026]] / [[Akkaynak 2019 — Sea-thru A Method For Removing Water From Underwater Images|Sea-thru 2019]]——把同一个模型倒过来解。

## Personal Knowledge State

- **Normal。** 工程侧（雾、半透明、OverDraw）是 Easy；本文给的是**这套工程直觉的公式来源**，以及**它在哪里失效**。
- 已能用自己的话说清：为什么透射率是指数、为什么除以 $\mu$、为什么 $\tau$ 只有一个乘积。
- 仍需练：相函数的归一化约定（$4\pi$ 从哪来）、HG 的 $g$ 符号、以及"$D\to1$"时 $\tau'=\tau/(1-D)$ 的推导。

## Learning Value

1. **"闭式"是一种设计选择，不是数学运气。** 想清楚"我愿意为了可积性删掉什么"，比会推公式更有用。
2. **指数来自组合计数。** 以后再看到 $e^{-\tau}$，先想"这是哪一个'没有东西挡路'的概率"。
3. **单次散射的失效边界是 $\omega\tau$，不是 $\tau$。** 薄而亮的介质（云）与厚而暗的介质（浓烟）可以一样"看起来厚"，但误差来源完全不同。

## Visualization

![[Research/Files/Pictures/Blinn 1982 — 图2 云层模型的几何.png]]
**读图**：一层厚度 $T$、数密度 $n$、粒子半径 $\rho$ 的平板；$\mu_0=\mathbf N\!\cdot\!\mathbf L$、$\mu=\mathbf N\!\cdot\!\mathbf E$、$\alpha$ 是 $\mathbf L$ 与 $\mathbf E$ 的夹角（相位角）。**注意 $\mathbf N,\mathbf L,\mathbf E$ 不要求共面**——这正是相位角可以独立变化的原因。出处：Blinn 1982, Fig. 2。

![[Research/Files/Pictures/Blinn 1982 — 图4 入射与出射圆柱.png]]
**读图**：位于深度 $T'$ 的粒子要被看见，必须让"入射圆柱 $V_{\text{in}}$"与"出射圆柱 $V_{\text{out}}$"的并集里**没有别的粒子**。阴影区必须为空——这就是 $P(0;V)=e^{-nV}$ 里那个 $V$ 的几何来源。原文在此明确忽略了两个圆柱的交叠。出处：Blinn 1982, Fig. 4。

![[Research/Files/Pictures/Blinn 1982 — 图8 光学厚度与入射角扫描.png]]
**读图**：四行是 $\tau=\{1000,\,2,\,0.5,\,0.1\}$，四列是入射角 $i=\{36^\circ,72^\circ,108^\circ,144^\circ\}$；矩形代表云层，从入射点向各方向伸出的长度就是该方向的亮度（已省略 $\mu$ 的除法、相函数取 1）。**三件事一眼可见**：①亮度沿**垂直于层面**的方向最大（出射路径最短、最不容易被挡）；②$\tau$ 增大时顶面反射变亮并趋于饱和（Hapke–Irvine 极限）；③$\tau$ 小时底面漏光明显，$\tau$ 大时底面全黑。出处：Blinn 1982, Fig. 8。

![[Research/Files/Pictures/Blinn 1982 — 图9 土星环的两面.png]]
**读图**：用上面的闭式渲染的土星环——左为受照面、右为背照面。环的 $\omega_1(r),\omega_2(r),\tau(r)$ 是从 Voyager 1 在几个已知几何下的照片反解出来的（$w_1L(\alpha)+w_2\mathrm{HG}(-0.5,\alpha)$）。**这张图的意义**：闭式不只是"能画云"，它**能被观测数据标定**，所以它是一个物理模型而不只是一个好看的经验式。出处：Blinn 1982, Fig. 9。

## Notes

- 原文 PDF 已归档：`Research/Files/PDF/Blinn 1982 - Light Reflection Functions for Simulation of Clouds and Dusty Surfaces.pdf`（Microsoft Research 公开副本，9 页扫描件 + OCR 文本层，逐页核对）。
- **扫描件 OCR 有明显错字**：作者名在正文写作 "Von Herren"、`φ` 被识别成 `~`、`τ` 被识别成 `"~`——引用时以 ACM 元数据为准（Kajiya 那篇同理）。
- **Kajiya & Von Herzen 1984 仍未取到全文**（ACM 封闭、Caltech 记录仅元数据、RG 反爬、多所大学镜像 404）。本文与它的关系只写到摘要级可核实的部分：**Kajiya 明确针对"高反照率下的多次散射"提出替代**。
- ⚠️ **符号陷阱**：本文 HG 用 $g<0$ 表示前向散射，与现代实现相反；引用 $g=0.325$ 时务必注明这是"暗粗糙粉尘"的拟合值。
