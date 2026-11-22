---
type: concept
title: "Single Scattering"
user_level: Normal
aliases: [Single Scattering, 单次散射, Single-Scattering Albedo, 单次散射反照率, 一次散射近似]
prerequisites: ["[[Participating Media]]", "[[BRDF]]", "[[Rendering Equation]]"]
first_introduced: "介质侧：Blinn 1982（闭式）；表面侧：Cook-Torrance 1981（微面单次散射）"
---

# Single Scattering

> **一个近似，两个域，同一条判据：光只与介质/微面作用一次。**
> 它在**雾**上几乎精确，在**云**上结构性错误——**区别不在"厚不厚"，在 $\omega\tau$。**

## Definition

**单次散射 = 只保留"光与散射体作用一次"的那一项。**

在参与介质里，它把辐射传输方程的内散射积分替换成"**直接光 × 相函数 × 透射率**"：

$$L(x,\omega)\;\approx\;\underbrace{\int_0^d T(0,s)\,\sigma_s(s)\,\varphi(\omega_l,\omega)\,L_{\text{light}}(s)\,e^{-\tau_{\text{light}}(s)}\,ds}_{\text{一次散射}}+\underbrace{T(0,d)L_{\text{bg}}}_{\text{透射}}$$

在微面表面里，它就是 Cook-Torrance 的 $D\,G\,F$ 那一项——**光打到一个微面、弹一次、离开**。

> **两个域的差别只有一句**：介质里没有"出去"这个边界（光一直走到被吸收或逃逸），表面里**必须从一个有界的面出去**。公式的形状一样，边界条件不同。

## Core Principle

### 1. 它只由三个数决定

| 量 | 含义 | 典型值 |
|---|---|---|
| $\omega=\sigma_s/\sigma_t$ | **单次散射反照率**：一次作用里"被散射"而非"被吸收"的比例 | 雾 ≈0.9–1；黑烟 ≈0.1；云 ≈1 |
| $\tau=\int\sigma_t\,ds$ | **光学厚度**：一条视线上"平均撞几次" | 薄雾 0.1；浓雾 5；云 50–1000 |
| $\varphi(\theta)$ | **相函数**：散射后往哪偏 | HG 的 $g$：雾 +0.6~0.9（前向），尘 −0.3 |

**三者合起来给出闭式的形状**（[[Blinn 1982 — Light Reflection Functions for Simulation of Clouds and Dusty Surfaces|Blinn 1982]] 的平板解）：

$$B=\omega\,\varphi(\alpha)\,\frac{\mu_0}{\mu_0+\mu}\left[1-e^{-\tau\left(\frac{1}{\mu_0}+\frac{1}{\mu}\right)}\right]$$

**结构的读法**：

- $\omega$：**振幅**（有多少光参与）；
- $\varphi$：**方向**（往哪去）；
- $1-e^{-\tau(\cdot)}$：**饱和**（越厚越接近"无限厚"的极限，即 Hapke–Irvine / Lommel–Seeliger）。

### 2. 它什么时候对：$\omega\tau\ll1$

**二阶项（两次散射）需要"路径上有两个粒子"，其份额随 $\omega\tau$ 增长。** 于是：

| 情形 | 单次散射 | 原因 |
|---|---|---|
| 薄雾、轻霾、体积光 | ✅ 几乎精确 | $\tau$ 小 ⇒ 撞第二次的概率低 |
| 暗烟、煤灰 | ✅ 可接受 | $\omega$ 小 ⇒ 撞到了也多半被吸收 |
| **云、雪、牛奶** | ❌ **结构性错误** | $\omega\approx1$ **且** $\tau$ 大 ⇒ 多次散射主导 |
| 皮肤、大理石 | ❌ 不适用 | 必须用扩散/BSSRDF（见 [[Subsurface Scattering]]） |

> **判据（本页最该记）：不要问"厚不厚"，问 $\omega\tau$。** 一层很厚的黑烟与一层很薄的云可以有相同的 $\tau$，但误差来源完全不同：前者是"吸收主导，单次够用"，后者是"散射主导，单次少算"。

**这不是精度问题，是项数问题。** 单次散射不是"多次散射的粗糙版"，它是**级数的一阶截断**——把 $\omega^2,\omega^3,\dots$ 全扔了。所以云被画暗、画"薄"、画成纸片，**调参数救不回来**。

### 3. 表面侧的同一个洞

微面 BRDF 的 $D\,G\,F$ 也是单次散射：$G$ 把"被挡住"的光当作**消失**，但真实微结构里它会弹到别的微面上。**"被挡住" ≠ "被吸收"**——这就是高粗糙度材质发闷的全部来源（见 [[Multiple Scattering and Energy Compensation]]）。

**两个域的对照：**

| | 介质（本页） | 表面（微面） |
|---|---|---|
| 丢掉的是什么 | 二阶以上的散射路径 | 微面之间的互反射 |
| 失效判据 | $\omega\tau$ | 粗糙度 ↑、掠射角 |
| 表现 | 云偏暗、缺"内部发光" | 高粗糙度发闷 |
| 补法 | 扩散 / 级数 / 路径追踪 | 能量补偿 / 随机游走 |

### 4. 反解侧的后果（今天新增的一条）

**单次散射模型是"两项的线性组合"，所以它可解——但正因如此，它不可辨识。**

$$I=\underbrace{J\,T^D}_{\text{直达}}+\underbrace{B^\infty(1-T^B)}_{\text{幕光}}$$

- **可解**：两个未知量、两个以上的观测（多距离/多颜色）就能拟合；
- **不可辨识**：$T^D$ 与 $B^\infty$ 的形状都由同一个 $e^{-\beta z}$ 控制，数据只能约束它们的**和**，不能约束"各占多少"。分离靠先验，不靠数据。

**这条在 [[Underwater Rendering]] 与 [[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting|NIRJALA]] 里被反复撞到**：任何"从图里去掉介质"的方案，只要它用的是单次散射模型，就必然面对"分量图只是模型的分配，不是光的测量"。

> **一条可迁移的判据：正向的单次散射近似越"干净"，反向的分解就越不唯一。** 因为干净来自"两项形状相同"，而可辨识需要"两项形状不同"。

## Prerequisites

- [[Participating Media]]——$\sigma_a/\sigma_s/\sigma_t$、透射率、相函数；
- [[Rendering Equation]]——知道"沿视线的积分"长什么样；
- 概率论基础（指数分布、泊松过程的零项）。

## Historical Evolution

```text
1960  Chandrasekhar        RTE + 单次散射项（天体物理）
        ↓
1982  ★ Blinn             平板 + 单次散射 ⇒ 第一个闭式
                          （明确写：只在粒子反照率 ω 小时成立）
        ↓
1984  Kajiya & Von Herzen 用"高反照率多次散射"作为替代目标
        ↓
1986  Kajiya              渲染方程：单次散射成为级数的第一项
        ↓
1995  Max                 光学模型分类：吸收 / 发射-吸收 / 单次 / 多次
        ↓
2000s 实时雾与体积光       单次散射 + 阴影贴图（工业默认）
        ↓
2020  Hillaire            二阶 × 几何级数（用结构补多阶）
        ↓
2026  反解                单次散射模型的两项结构 ⇒ 可解但不可辨识
```

## Important Papers

| 论文 | 角色 |
|---|---|
| [[Blinn 1982 — Light Reflection Functions for Simulation of Clouds and Dusty Surfaces\|Blinn 1982]] | **介质侧闭式的源头**；三个假设（单次、几何光学、泊松遮挡）与 Hapke–Irvine 极限 |
| [[Narasimhan 2002 — Vision and the Atmosphere\|Narasimhan & Nayar 2002]] | 把同一个两项结构翻到**反解**侧；明确丢掉前向散射 |
| [[Akkaynak 2019 — Sea-thru A Method For Removing Water From Underwater Images\|Sea-thru 2019]] | 反解侧的第一个可执行版本；演示"半个正确模型比不处理更糟" |
| [[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting\|NIRJALA 2026]] | 给单次散射加第三项，让反演增益有界 |
| [[Hillaire 2020 — A Scalable and Production Ready Sky and Atmosphere Rendering Technique\|Hillaire 2020]] | 用"二阶 × 公比"近似多阶——单次散射之外最便宜的补法 |

## Related Concepts

- [[Participating Media]]——上位；本页是它的"路径 0"：**选一个能积出来的模型**；
- [[Multiple Scattering and Energy Compensation]]——**表面侧的同一个洞**，以及五条补法；
- [[BRDF]]——Lommel–Seeliger 是单次散射在"粉尘表面"上的极限；
- [[Atmospheric Scattering]]——单次散射 + 球壳边界；
- [[Underwater Rendering]]——单次散射 + 逐通道吸收 + 反解；
- [[Subsurface Scattering]]——单次散射彻底失效的地方（必须多阶）。

## Technologies

- **指数高度雾 / Exp2 雾**：单次散射的最简特化；
- **Froxel 体积雾**：单次散射 + 每步阴影（步数换精度，但项数不变）；
- **体积光 / god rays**：单次散射 + 屏幕空间径向模糊；
- **粒子云 / billboard 云**：逐粒子的单次散射。

## Game Applications

- **正向**：雾、沙尘、水下后处理——**单次散射就够**，因为玩家只需要"看起来对"；
- **分档**：单次散射是"低档"，多次散射（Hillaire 级数 / 路径追踪）是"高档"；**中间没有连续过渡**——这是"换结构"而不是"调参数"；
- **验收**：拿一个"无限厚"的参照（例如 $\tau=1000$ 的平板），看你的云在厚度增加时是否**趋于一个有限亮度**。如果它随厚度一直变亮或一直变暗，你的实现里少了饱和项。

## Personal Knowledge

- **Normal。** 工程直觉（雾、半透明、OverDraw）是 Easy；本页要建立的是**"项数"这个维度**——以前我只按"步数/分辨率"分档，现在多了"算几阶散射"。
- 已能说清：为什么 $\omega\tau$ 是判据、为什么闭式里有 $1-e^{-\tau}$、为什么云不能靠调参救。
- 仍需练：相函数的归一化（$4\pi$ vs $1/4\pi$ 的约定）、HG 的 $g$ 符号、以及"二阶项份额 ∝ $\omega\tau$"的量级估算。

## Learning Gap

1. **$\omega\tau$ 的定量门槛**：本文只给"$\omega$ 小"的定性说法；"多次散射占多少"需要解 RTE 的二阶项（或查表）。**这是本页最大的空白。**
2. **相函数归一化约定**：不同文献对 $\varphi$ 是否含 $1/4\pi$ 处理不同，导致"$\omega$"与"反照率"差一个常数——**读公式前先看归一化**。
3. **单次散射与"前向散射"的区别**：前向散射是**方向**问题（$\varphi$ 的尖峰），单次散射是**阶数**问题。两者常被混为一谈。

## Next Step

1. **做一次 10 分钟的对照**：在你的引擎里把雾密度翻倍，观察远处的亮度是否趋于饱和。**趋于饱和 = 有 $1-e^{-\tau}$；线性变亮 = 实现里把 $1-e^{-\tau}$ 近似成了 $\tau$**。
2. **把"阶数"加进分档表**：低档 = 单次散射；中档 = 单次 + 环境项；高档 = 多阶（级数/路径追踪）。见 [[Scalability and Quality Tiers]]。
3. **读 [[Blinn 1982 — Light Reflection Functions for Simulation of Clouds and Dusty Surfaces|Blinn 1982]] 的 §4.2**：图 8 的 $\tau$ 扫描是"饱和"最直观的一次演示。

## Mastery Criteria

1. 说出单次散射丢掉的是哪一阶，以及为什么它"不是粗糙版而是截断"；
2. 用 $\omega$ 与 $\tau$ 判断一个场景该不该用单次散射；
3. 解释闭式里三个因子各自的来源（振幅 / 方向 / 饱和）；
4. 说出表面侧与介质侧"同一个洞"的对应关系；
5. 解释为什么单次散射模型可解但不可辨识。
