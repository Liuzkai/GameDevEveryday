---
type: paper
title: "An Energy-Conserving Hair Reflectance Model"
authors: [Eugene d'Eon, Guillaume Francois, Martin Hill, Joe Letteri, Jean-Marie Aubry]
year: 2011
published: "2011-06-27（EGSR 2011；Computer Graphics Forum 30(4): 1181–1187）"
venue: "EGSR 2011 — 22nd Eurographics Symposium on Rendering / CGF 30(4): 1181–1187"
url: "https://doi.org/10.1111/j.1467-8659.2011.01976.x"
code: ""
project_page: "https://eugenedeon.com/pdfs/egsrhair.pdf（作者官方 PDF；本次已下载 9.2MB 原文并逐节核对）"
category: [hair, scattering, brdf, energy-conservation, production]
importance: S（经典）
historical_importance: 4
game_relevance: 4
production_readiness: "Industry Adopted（Weta Digital 生产管线 + PRMan 实装；pbrt-v4 的 HairBxDF 直接采用本文的 M_p 与色素 RGB 交叉截面——见下方『行业实装核实』）"
user_level: Normal
status: unread
aliases: [d'Eon 2011, Energy-Conserving Hair, 能量守恒毛发模型, Weta hair, EGSR 2011 hair, 毛发能量守恒]
tags: [hair, scattering, brdf, energy-conservation]
---

# An Energy-Conserving Hair Reflectance Model（d'Eon et al. 2011）

> **入库 2026-10-07（Run 29）。** 作者 = **Weta Digital**（d'Eon / Francois / Hill / **Joe Letteri** / Aubry）。
> **与库内两条线的关系**：① 它是 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 的**直接修正版**——用户当前正深读 Marschner（自制光路图 + 精确勘误），本文回答其遗留的**能量问题**；② 它与表面侧的 [[Multiple Scattering and Energy Compensation]] 构成**同题对偶**——"补能量"的账本从面（Kulla-Conty / Hammon）延伸到纤维（本篇重建守恒散射函数）。③ 与 [[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]] 的对接是原文自己验证的（"integrates seamlessly"）——**修复了双散射的"高有效粗糙度"区间的能量损失**。

## TL;DR

**Marschner 2003 的纵向高斯函数 M_p 不守恒能量——本篇用"球面高斯卷积"重新推导出一个守恒的 M_p（非高斯、带 off-specular peak），并把方位向从"解三次方程"改为"沿纤维宽度积分 + 高斯检测器"。** 结果：对任意粗糙度、任意吸收、任意角度**总反射率恒等于 1**（白环境实验验证）；模型可任意阶展开（补上 TRRT 后白光掠射的近 15% 能量）；Weta 生产 + PRMan 实装，并可无缝接入 Dual Scattering。

## Problem

**把"近十年前为效率做的近似"逐条翻案。**（原文：*"Nearly a decade ago approximations were chosen to yield the most efficient model possible... Today, we find it practical to revisit"*）

Marschner 2003 的模型"widely successful"，但有三类失效场景：

| 失效场景 | 原因 |
|---|---|
| **低吸收发**（浅金 / 白发） | R/TT/TRT 三阶只覆盖部分能量；高阶项（TRRT+）在白发上不可忽略 |
| **掠射角** | 能量被推出有效角域（θ ∈ (-π/2, π/2) 之外），直接丢失 |
| **高表面粗糙度** | 高斯近似在锥压缩/扩张失真下系统性丢能量 |

而此前的扩展（Zinke-Weber 2007 换 g(β; 2θh) 修掉了翻倍、Zinke 2009 加非物理漫反射项、Sadeghi 2010 做美术友好化）**都只治局部——"do not provide a globally energy-conserving model, which is the goal of this paper."**

## Historical Context

```text
Kajiya-Kay 1989（经验）→ Marschner 2003（物理：R/TT/TRT 分解）
        ↓
Zinke-Weber 2007（BCSDF 形式化；但前向散射丢一半——本文图 6 纠正）
        ↓
★ 2011 本文（Weta）：能量守恒重建 + 取消求根 + 任意阶
        ↓
（后续：Chiang 2016 消除宽度积分 → 生产化主流实现；本库记名）
```

## Core Idea

**四个替换**（都不是微调，是重新推导）：

1. **纵向 M_p：用球面高斯卷积替代直接套高斯。** 从"球面上 Dirac 圆 × 球面高斯"的卷积出发，取其在纵向的剖面 —— 得到一个**非高斯、非对称、带 off-specular peak** 的函数，且**对任意 (θi, θr) 精确守恒**（Eq 7）。
2. **方位向 N_p：用"沿宽度积分"替代"解 h"。** Marschner 是"解出光出射到相机的那些离散路径 h"（需要三次方程求根）；本文改为**对整个纤维宽度做积分**，每条路径的光被一个**高斯检测器**（含 2π 周期性求和）展宽——粗糙度由此自然进入，不再需要 caustics 的 ad-hoc 参数。
3. **归一化归位：所有入射/出射因子收进一个互易函数。** 不再需要 Marschner 的 1/cos²θ_d 与 cosθ_i 因子，Bravais Fresnel 项被证明"unnecessary"。
4. **阶数不封顶：TRRT 及更高阶顺手加入。** 白光掠射下 R/TT/TRT 漏掉近 **15%** 能量（原文图 5）。

## Technical Approach

### ① 为什么不守恒（三条原因，原文逐条列出）

| # | 原因 | 后果 |
|---|---|---|
| 1 | 高斯在 θ∈(-∞,∞) 上归一，却在 θ_h∈(-π/2,π/2) 上取值，且用 **θ_h 代替 θ**（分布域翻倍） | **平均能量翻倍**（H∞ ≈ 2；Zinke-Weber 换 2θ_h 修掉这条） |
| 2 | 从锥 -θ_i 偏折到 θ_r 会**压缩/扩张光锥**（图 2），而 1/cos²θ_d 只是近似 | 低粗糙度段的系统性偏差 |
| 3 | 掠射角：大量能量被偏折到**永远不会被接收**的角域之外 | **能量直接丢失** |

**白环境思想实验（本库称"毛发版 furnace test"——个人解释）**：把一根**零吸收**纤维放进**均匀白色环境**：1/η² 的立体角压缩与扩张两两抵消，所有路径都通到白色 → **纤维应与背景不可区分，总反射率必须恒为 1**（H∞(θi) = 1，Eq 5）。实测（原文图 3）：高斯版本小粗糙度 ≈ 2、掠射 >1、高粗糙度 <1；**新 M_p 对全部 θi、全部 v 恒为 1**。

### ② 新纵向函数（Eq 7）

$$M_p(v,\theta_i,\theta_r)=\frac{\mathrm{csch}(1/v)}{2v}\,e^{\,\sin(-\theta_i)\sin\theta_r / v}\,I_0\!\left[\frac{\cos(-\theta_i)\cos\theta_r}{v}\right],\qquad v=\beta^2$$

- $I_0$ = 第一类修正贝塞尔函数；$v$ = 粗糙度方差（论文里 β 的平方）；
- 形态：**非对称、off-specular peak**（对平面 BRDF 的熟知现象，在纤维维度上的对应物）；
- 归一：$\int\int N_R(\varphi)\,M_p\,\cos\theta_i\,d\theta_i\,d\varphi = 1$——**入射与出射因子全在一个互易函数里**。光栅化实现直接 `贡献 ∝ 覆盖C × S`，不用再乘 cosθ_i / 1/cos²θ_d（原文明确指出这是实现上的便利）。

### ③ 方位向：取消求根（Eq 10–11）

$$N_p(\varphi)=\frac{1}{2}\int_{-1}^{1}A(p,h)\,D_p\big(\varphi-\Phi(p,h)\big)\,dh,\qquad D_p(\varphi)=\sum_{k=-\infty}^{\infty} g(\beta_p;\,\varphi-2\pi k)$$

- $A(p,h)$：路径的 Fresnel + 吸收衰减；$\Phi(p,h)$：光滑纤维的精确方位偏折（沿用 Marschner Eq 8 的几何）；
- **高斯检测器 $D_p$ 的 2π 求和是关键**：光可能在纤维内转好几圈才出来；Zinke-Weber 2007 未处理 $\pi-\varepsilon$ 与 $-\pi+\varepsilon$ 的贴边，"transmit approximately half of the expected light in the forward directions"（图 6）；
- 粗糙度进入 $N_p$ 后带来**粗糙度相关色移**（不同粗糙度下，各出射方向混合的吸收长度不同，图 8）——这是"离散路径"框架给不了的；
- 实现：$D_p$ 有第三类椭圆 Theta 函数闭式，实际用有限 k + 高斯求积（**阶 35 对绝大部分毛发够用**）；$N_p$ 可按 (μa, βp) 预计算 2D 表（Nguyen-Donnelly 路线）；可利用对称性只积 h∈{0,1}。

### ④ 其余归一与细节

- R 的衰减：$A(0,h)=F(\eta,\ \tfrac{1}{2}\arccos(\omega_i\cdot\omega_r))$——**用半边角求 Fresnel**（物理一致性）；
- 通用项：$A(p,h)=(1-f)^2 f^{p-1} T(\mu_a',h)^p$，$f=F(\eta,\arccos(\cos\theta_d\cos(\arcsin h)))$；吸收项 $T=\exp(-2\mu_a(1+\cos 2\gamma_t))$，$\mu_a'=\mu_a/\cos\theta_t$；
- 切角鳞片：仍在纵向用移位近似 $M_p(v,\theta_i,\theta_r-\alpha_p)$——**明确承认只近似**（内侧反弹长度变化未计入）；
- **色素 RGB 交叉截面**（从 Donner-Jensen 2006 光谱、40 波段 → sRGB D65）：$\sigma_{a,e}=\{0.419,0.697,1.37\}$（eumelanin）、$\sigma_{a,p}=\{0.187,0.4,1.05\}$（pheomelanin），单位按 R² 计（浓度 ~1/R³ 得黑发）——**pbrt-v4 里用的就是这组值**。

## Key Contribution

1. **一个对全部粗糙度/吸收/角度守恒的解析纤维散射模型**（不是"更准一点"，是"账目闭合"）；
2. **可任意阶**（TRRT+ 顺手加入，白发的近 15% 能量问题由"加阶"解决，而不是加经验项）；
3. **取消三次方程求解** + caustics 进入"一致且稳定"的处理 + 粗糙度色移；
4. 实现了"**解析参数化 + 守恒**"两者兼得——对照物是 Ogaki 2010 的模拟驱动表格法（能守恒，但不可参数化控制）。

## Why It Works

**把"守恒"从约束变成推导起点**（个人解释）：不先写一个形状再补归一化因子，而是**直接构造一个在球面上守恒的分布**（球面高斯卷积天然归一），再投影回纵向剖面——守恒是构造出来的，不是修正出来的。这也解释了为什么新函数"恰好"是非高斯的：球面几何本身就不产生高斯纵向剖面。

## Limitations

1. **切角移位仍是近似**：只改 M_p 的自变量，未计内侧反弹长度变化（原文自述）；
2. **椭圆截面 / 中空 medulla 未建模**（future work 指向 medulla 与 feathers）；
3. **修正了一个流传的猜测**：原文的能量分析显示 **TRRT+ 不解释"漫射有色反射"**——作者认为该现象来自 **inner-core scattering**（内芯散射，同时让主项去饱和）——"需测量验证"（原文自述，未证实）；
4. 高斯检测器实际用有限 k、贝塞尔函数用级数展开——工程近似仍在，只是受控。

## Game Development Relevance

**4/5。给"毛发能量账本"上第二册的钥匙——而且正好接在你正在读的 Marschner 之后。**

- **分档判据（可直接用——推断）**：新 M_p 与高斯 M_p 的差异集中在 **① 高粗糙度 ② 浅色发 ③ 掠射**。含义：**深色发为主 + 中等粗糙的画面，旧近似"够用"**（这是大量实时实现至今不换的原因）；**一旦有白发/浅金发角色 + 强逆光 + 毛躁发型，旧近似的能量账就露馅**——特别是它接在 [[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)|双散射]] 后面时（双散射用"大有效粗糙度"近似多次散射，恰好落进高斯丢能量最狠的区间——**原文图 11 的对照就是"光在暗部扩散不够远"**）。
- **"白环境实验" = 毛发版 furnace test**：与你在表面侧要做的那项实测（粗糙端变暗 / 光滑端亮边）同构——**判据一句话："零吸收 + 均匀白环境 → 应该看不见它"**（工程推断：这是以后评估任何毛发近似实现的第一体检项）。
- **资产/实现侧**：pbrt-v4 直接采用本文模型（见下）→ **如果你用 Blender/Cycles 或 pbrt 做毛发对照渲染，看到的"标准答案"很可能就是这个模型**；Weta 侧它已是生产默认的一部分。
- **与表面的能量线合读**：Kulla-Conty / Hammon 是"**补能量**"（打补丁）；本篇是"**重建函数使能量天然守恒**"——与 [[Multiple Scattering and Energy Compensation]] 里 Dupuy 2026 路线（闭式 + 保能量）同一种哲学：**"要守恒就重推，不要打补丁"**（个人解释）。

## Unreal Engine Relevance

- UE 的头发着色（HairStrands / Groom 路径）属"Marschner 家族的多 lobe + 多散射近似"结构；**但 UE 具体采用的 M_p 形式本次未核实**，不下断言（待核实项）；
- 对材质侧的可迁移判断（推断）：如果你看到 hair shader 参数是**"轴向粗糙度 / 方位向粗糙度"分离**的形式，那就是 2003/2011 这条线的分解结构；本篇的贡献是把其中"轴向"那一半换成守恒版本。

## Technology Evolution

```text
1989 Kajiya-Kay（经验：切向高光 + 圆锥）
        ↓
2003 Marschner（物理：R/TT/TRT + 方位分解 + 高斯近似）
        ↓
2007 Zinke-Weber（形式化 + 修翻倍；前向仍丢一半）
        ↓
2008 Zinke-Yuksel（双散射：多散射近似）★ 库内
        ↓
★ 2011 本文（Weta）：守恒 M_p + 取消求根 + 任意阶 —— 并把双散射"扶正"
        ↓
2016 Chiang et al.（Pixar）：消除宽度积分 → 生产化主流（库外，记名）
2016+ 影视/离线事实标准的一部分
```

## Relationships

### Based On

- [[Marschner — Light Scattering from Human Hair Fibers (2003)]] —— 沿用其 R/TT/TRT 分解、局部坐标系 {u,v,w}、$\Phi(p,h)$ 几何、参数命名；**改掉它的近似**
- "球面高斯卷积取纵向剖面"的数学技法（附录 A 给出完整推导）——**注：这是"先构造守恒分布、再取剖面"的方法来源**

### Improves

- [[Marschner — Light Scattering from Human Hair Fibers (2003)]] —— **修复**：能量不守恒（三条原因）、求根/奇点、离散路径无法表达粗糙度色移
- Zinke-Weber 2007 的方位向处理（贴边丢一半，本文图 6 纠正）

### Productionizes / Provides

- [[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)]] —— 双散射的"单散射输入"换成守恒版本后，高有效粗糙度区不再丢能量（原文图 11）

### Related

- [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]] —— 更早的经验侧；本文的对照物（"第一个物理模型"之前）
- [[Hair Rendering]] —— 本篇补上其"能量"横切面
- [[Multiple Scattering and Energy Compensation]] —— 表面侧的"补能量 / 重推函数"与本篇构成跨域对偶

### Followed By

- Chiang et al. 2016（Pixar，记名未入库：消除宽度积分的高效生产实现）；d'Eon et al. 2014（non-separable 模型，记名）

## Personal Knowledge State

- **user_level: Normal（推断）**。你是做角色 VFX 与分档的，毛发"表示层"（发片/发丝）在你 Easy 区；本篇属于**着色模型的能量账**层——与你正在读的 Marschner（R/TT/TRT）无缝衔接，读完 Marschner 后本篇是自然的下一站（+30–60 分钟）。
- **读法建议**：① 先读 §4.1 的三条不守恒原因 + 图 3（白环境对照）；② 再看图 4（新函数形状）；③ 再看 §5.1 高斯检测器 + 图 6（前向丢一半）；④ 其余（Eq 7 推导 / 色素截面 / 实现细节）按需。**不需要跟进附录 A 的完整推导**。

## Learning Value

- **3 条新自测（并入 [[Hair Rendering]]，总数 12 → 15 → 18 条）**：
  1. 高斯 M_p 不守恒的**至少两条**原因？（θ_h 翻倍 / 锥压缩近似 / 掠射溢出）
  2. "白环境实验"（零吸收纤维 + 均匀白环境）为什么要求总反射率为 1？它相当于什么测试？
  3. d'Eon 为什么取消"解 h"？换成了什么？代价与收益各是什么？
- **一条可迁移判据**：**"守恒是构造出来的，不是修正出来的"**——先构造一个在正确域上归一的分布，再回推形状；而不是先形状后补因子。（与 Dupuy 2026、Ward 误差容限同一族：**先定约束，后定实现**）

## Visualization

![[毛发能量守恒_d'Eon 2011 与高斯函数对照图解.html]]

## Notes

- **原文核对（本次已下载官方 PDF 9.2MB、逐页抽取核对）**：白环境实验与 H∞(θi)（§4.1 / Eq 5 / 图 3）、新 M_p 公式（Eq 7 / 附录 A）、高斯检测器与 2π 求和（Eq 11 / 图 6）、TRRT 15%（图 5）、色移（图 8）、色素截面数值（§6.1）、求积阶 35（§6）、双散射对接（§7 / 图 11）——**均为原文逐条确认**。
- **行业实装核实（pbrt-v4 官方文档《Further Reading》，2026-10-07 核实）**：*"They showed that their M_p term was not actually energy conserving and derived a new one that was; **this is the model from Equation (9.49) that our implementation uses**."* + 色素 RGB *"were computed by d'Eon et al. (2011)"*——**pbrt-v4 的 HairBxDF 直接采用本篇**。另：d'Eon 2013 给出低粗糙度的数值稳定版 M_p（记名）。
- **引用格式化**：EGSR 2011 / CGF 30(4): 1181–1187；DOI 10.1111/j.1467-8659.2011.01976.x；作者单位 = Weta Digital（全部五人）。
- **与用户既有笔记的接口**：用户在 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 补充的"路径权重 A 不能直接当作完整散射函数 S"——**本篇的 A(p,h) 归一与 S 分解正是对该问题的一条标准答案**（推断：可作为用户那张自制图的下一层）。
