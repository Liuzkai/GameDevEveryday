---
type: paper
title: "Real Shading in Unreal Engine 4"
authors: [Brian Karis]
year: 2013
published: "2013 (course notes v2; changelog 2013-08-05)"
venue: "SIGGRAPH 2013 Course — Physically Based Shading in Theory and Practice"
url: "https://blog.selfshadow.com/publications/s2013-shading-course/karis/s2013_pbs_epic_notes_v2.pdf"
code: "UE4 源码（Public/Shading 与 ReflectionEnvironment 相关）"
project_page: "https://blog.selfshadow.com/publications/s2013-shading-course/"
category: [pbr, real-time-rendering, ibl, split-sum, unreal-engine]
importance: S
historical_importance: 4
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal
status: unread
aliases: [Karis 2013, Real Shading in Unreal Engine 4, UE4 PBR 课程笔记]
tags: [pbr, real-time-rendering, unreal-engine, split-sum, classic]
---

# Real Shading in Unreal Engine 4 (Karis 2013)

## TL;DR

**这是把你前四天学的 D · G · F 真正装进引擎的那一篇。**

它一个新技术都没有。**它全部的贡献是"在正确的层级上做便宜的近似"**——这句话就是实时 PBR 的全部方法论，而这篇是它的规范文本。

| 你已经知道的 | 这篇告诉你实时里用哪个 | 一句原话 |
|---|---|---|
| D（法线分布） | **GGX / Trowbridge-Reitz**，并采纳 Disney 的重参数化 $\alpha=\text{Roughness}^2$ | "longer *tail* appealed to our artists" |
| G（几何遮蔽） | **Schlick 形式 + $k=\alpha/2$ 去拟合 Smith** | "exactly matches Smith for $\alpha = 1$" |
| F（菲涅尔） | **Schlick + 球面高斯改进版** | "slightly more efficient… difference is imperceptible" |
| 漫反射 | **Lambert**（评估过 Burley，差异太小，不值那个成本） | "couldn't justify the extra cost" |
| **环境光镜面** | **split-sum：预滤波 cubemap + 2D BRDF LUT** | "we can only practically afford a single sample" |

> **一句话定位：你的环境光（IBL）那一半，今天补齐。** [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]] 给了漫反射侧（SH 投影），**Karis 2013 给了镜面侧**。两篇合起来，[[Physically Based Rendering]] 的"IBL 大半边"才算完整。

## Problem

2013 年 Epic 要做的事有两件，第二件容易被忽略但决定了全文的技术取舍：

1. **把材质参数化到美术能用**（去学 Disney 2012），且要**一次着色**支持材质分层（material layering）；
2. **解析光源与 IBL 必须能互换**，参数行为一致；延迟渲染下 G-Buffer 空间、贴图存储、pixel shader 里的分层混合成本都要压住。

于是本文的"目标函数"不是画质，是这串约束：**实时 · 参数少 · 感觉线性 · 易掌握 · 稳健 · 可表达 · 可分层**。

## Historical Context

```text
Cook-Torrance 1981 —— D·G·F 框架（离线）
        ↓
Schlick 1994 —— 廉价的 F（[[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]]）
        ↓
Walter 2007 —— GGX + Smith，D 与 G 的现代标准（[[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]）
        ↓
Disney 2012（Burley）—— 原则化参数化 + 生产验证
        ↓
★ Karis 2013 —— 进引擎：G 的廉价拟合 + 环境光 split-sum + 材质模型定型（本文）
        ↓
全工业收敛到金属度工作流（UE / Unity / 自研引擎）
        ↓
UE5 Substrate —— 分层 lobe 框架
        ↓
2026 —— 神经渲染（DLSS 5）从画面侧绕过这条管线（见 [[DLSS 5 — Generative Neural Rendering]]）
```

一个值得记的历史细节：**本文致谢里点名 Sébastien Lagarde 在他 importance sampling 数学里找出了错误，修正后"lead to the development of our environment BRDF solution"。** 环境 BRDF LUT 这个今天每个引擎都有的东西，起点是一次代码审查。

## Core Idea

**没有新公式，只有新取舍。** 三处取舍各自对症：

1. **D 用 GGX 但接受它贵一点**：GGX 比 Blinn-Phong 贵，但长尾"自然"，美术认；
2. **G 不用 Smith 原始式，改写成 Schlick 形式配 $k$**：同一个效果，更便宜，且在 $\alpha=1$ 处与 Smith 精确相等；
3. **环境光不采样，改查表**：把"1024 次重要性采样"换成 **1 次 cubemap 采样 + 1 次 2D 查表**。

第 3 条是整个实时 IBL 的地基，值得单独看。

## Technical Approach

### 1. D — GGX + $\alpha=\text{Roughness}^2$

$$D(h)=\frac{\alpha^2}{\pi\left((n\cdot h)^2(\alpha^2-1)+1\right)^2},\qquad \alpha=\text{Roughness}^2$$

**$\alpha=\text{Roughness}^2$ 是 Disney 的重参数化**，不是 GGX 原文的东西。它的作用是让美术滑杆中段的手感更线性——**这是一个纯 UX 决定，藏在数学符号里**。看懂这件事，你就知道 UE 材质面板上`Roughness`为什么和"镜面宽度"不是线性关系。

### 2. G — Schlick 形式拟合 Smith

原文做法：用 Schlick 的几何项，**把 $k$ 取成 $\alpha/2$ 以更好拟合 GGX 对应的 Smith**，并采纳 Disney 对粗糙度的重映射：

$$k=\frac{(\text{Roughness}+1)^2}{8},\qquad G_1(v)=\frac{n\cdot v}{(n\cdot v)(1-k)+k},\qquad G(l,v,h)=G_1(l)\,G_1(v)$$

**这里有一条最容易踩的坑，原文专门警告**：

> Disney 那个重映射**只用于解析光源**；如果用在 IBL 上，**掠射角会暗得不成样子**。

也就是说：**UE 里直接光和 IBL 用的是两套 $k$。** 你在看 shader 时如果发现同一份 BRDF 代码有两处参数，根因就在这里。

### 3. F — Schlick + 球面高斯（去 pow）

$$F(v,h)=F_0+(1-F_0)\,2^{\,(-5.55473\,(v\cdot h)\,-6.98316)(v\cdot h)}$$

**用 $2^{\text{线性函数}}$ 替代 $(1-\cos\theta)^5$**：少一次 `pow`，视觉上察觉不到。这是"优化到最后一个指令"的典型样本——顺便说明**[[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]] 的形式在引擎里早已被改写，只是形状不变**。

### 4. 环境光 —— 这一节才是全文的真正贡献

要算的积分（重要性采样形式）：

$$\int_H L_i(l)\,f(l,v)\cos\theta_l\,dl \;\approx\; \frac{1}{N}\sum_{k=1}^{N}\frac{L_i(l_k)\,f(l_k,v)\cos\theta_{l_k}}{p(l_k,v)}$$

原文给出的运行时现实：**因为每像素要在多张环境贴图之间混合（局部反射），实际只负担得起"1 个采样"。** 于是：

**Split sum（式 7）——把求和拆成两项的乘积：**

$$\frac{1}{N}\sum \frac{L_i f\cos\theta}{p}\;\approx\;\left(\frac{1}{N}\sum L_i\right)\left(\frac{1}{N}\sum \frac{f\cos\theta}{p}\right)$$

| 项 | 物理含义 | 怎么预计算 | 运行时怎么查 |
|---|---|---|---|
| 第一项 | **预滤波环境贴图**（只有环境，无 BRDF） | 用 GGX 重要性采样卷积环境图（1024 采样），按粗糙度存进 **cubemap 的 mip 层级** | 用反射向量 $R$ 与粗糙度采样一次 |
| 第二项 | **环境 BRDF**（纯 BRDF，把 $L_i$ 当作纯白） | 代入 Schlick 后 **$F_0$ 可提出积分**（式 8），得到两个只依赖 $(\text{Roughness},\ \cos\theta_v)$ 的输出 $A,B$，存 **2D R16G16 LUT** | 用 $(\text{Roughness}, n\cdot v)$ 查一次 |

$$L_o \;\approx\; \text{PrefilteredColor}\times\Big(F_0\cdot A+B\Big)$$

**两个近似的诚实排序（原文自己排的，请照抄这个判断）：**

1. **split sum 本身误差较小**：原文措辞是"对常数 $L_i$ 精确，对常见环境相当准"；
2. **更大的误差来源是 $n=v=r$ 假设**：预滤波时假设视角与法线、反射方向重合（各向同性），否则预滤波结果要随视角变化、无法预存一张 mip 链。**代价是：掠射角下得不到拉长的反射。** 原文原话是"**compared with the split sum approximation, this is actually the larger source of error**"。

> 这条是今天最值得带走的判断：**人们通常担心 split-sum 准不准，而作者说真正的误差在它旁边那一刀。**

### 5. 材质模型与灯光（顺带定型的工业标准）

- 基础参数：**BaseColor / Metallic / Roughness / Cavity**；
- **砍掉 `Specular`**：原文说这个名字"有害"，且美术常误以为默认是 1（实际默认 0.5 对应 4% 反射率）；**非金属 $F_0$ 现在直接固定为常数 0.04**。→ **这就是你今天在 UE 里看到"Specular 默认 0.5 / $F_0$=0.04"的来源。**
- 金属度是二值的（原文建议直接回答美术"是"），中间值用材质分层表达；
- 逆平方衰减 + 窗函数软截断：`saturate(1-(d/r)^4)^2 / (d^2+1)`，光单位改 **lumens**；
- **面光源用 representative point + 能量守恒修正**（球光乘 $(\alpha/\alpha')^2$，线光乘 $\alpha/\alpha'$）——这是 UE 面光源的实现路线。

## Limitations

1. **$n=v=r$**：掠射反射被抹平（上表第 2 条）；
2. **未采纳 Burley 漫反射**（缺 retroreflection / fresnel shadowing），掠射的漫反射行为不如 Disney；
3. **单散射能量损失**：GGX + 单次散射在高粗糙度下会丢能量（后续由多散射补偿工作修正，如 Kulla-Conty 2017 / Fdez-Agüera 2019——**这两条是我的时间线补充，不在本文中**）；
4. **面光源的失败尝试被完整记录**：2012 那套"按光源立体角修正 D"的做法**在 GGX 上不可用**（高光材质被大面积光源照得发糊），原文明确说"for our chosen shading model (based on GGX), it isn't viable"；
5. **管线级副作用（对你最有用的一条）**：材质分层受 shader 成本限制，**美术的绕法是把网格切成多段 → DrawCall 变多**。原文原话是"an area that is cause for concern"。

## Game Development Relevance

**5/5。这是"你每天的材质工作"的规范文本。**

- **[[Real-Time VFX Performance Budgeting]] 的直接接口**：$F_0=0.04$、Roughness 重参数化、$k$ 的两套取值——**特效材质上的"发光边"是不是白高光，取决于这三处常量**，而不取决于你调 alpha；
- **DrawCall 维度**：材质分层 → 拆网格 → DrawCall 上涨，是**材质预算向 DrawCall 预算溢出**的一条已知路径（本文 2013 年就点了出来）；
- **分档旋钮（可直接用）**：IBL 侧真正随档位变的是 **cubemap 分辨率/mip 数 + LUT 精度**（原文强调 LUT 用了 R16G16，因为"精度重要"）。**这两项是你可以在 Android 三档里动手的地方，而不是改 F 或 D 的形式**；
- **材质数量 = 隐藏预算**：Shading Model 与 lobe 数直接决定 G-Buffer 与 IBL 采样次数（Clearcoat 要双份 IBL 采样，原文明确）。

## Unreal Engine Relevance

- **对照 shader 自测的最佳素材**：`Roughness²`、`k=(Roughness+1)²/8`、`2^(−5.55473x−6.98316)x`、`PrefilteredColor * (F0*A + B)` 这四句在 UE 源码/材质函数里都能找到对应物；
- **Cubemap / Sky Light**：预滤波 mip 链就是 Sky Light 的镜面部分，`r.SkyLight.RealTimeReflectionCapture` 一类开关改的是它的生成方式；
- **面光源**：UE 的 `Rect Light` / `Point Light (Source Radius)` 走 representative point 路线；
- **Substrate（UE 5.2+）**：分层 lobe 正是本文"材质分层"目标的下一代方案，但**性能风险的形状没变**（lobe 组合数爆炸）。

## Technology Evolution

```text
理论（1981-2007）：Cook-Torrance → Schlick → Walter
        ↓
参数化（2012）：Disney Principled
        ↓
★ 工程化（2013）：本文 —— 三因子换廉价形式 + split-sum 环境光 + 材质模型定型
        ↓
多散射补能量（2017-2019，后续工作）：Kulla-Conty / Fdez-Agüera
        ↓
分层化（UE5 Substrate）
        ↓
神经化（2026）：DLSS 5 从画面侧接管外观生成
                 → 值得问的问题：当画面由生成模型补齐，IBL 的精度还值多少预算？
```

## Relationships

### Based On

- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]（框架）
- [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]]（F）
- [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]（D 与 G）
- Disney Principled 2012（参数化）

### Productionizes

- 上面四篇 → 本文是它们**进入引擎的那一步**；[[Physically Based Rendering]] 的"实时化"含义就是本文

### Enables

- [[Split-Sum Approximation]]（★ 本文提出的环境光近似，已单独建概念）
- 全工业金属度工作流、材质分层、UE 面光源

### Contrasts

- [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]]：2026 的工作在**重新推导** D 与 G 的理论来源；本文在**回避**理论、只求便宜——**同一套公式的两个极端用法**，都值得看

## Personal Knowledge State

- **user_level: Normal（正在收口）**。你是 UE 侧工程师，本文的每个结论都在你日常范围内；新的是**"为什么是这些常数、这些形式"**。
- [[Physically Based Rendering]] 的 Learning Gap 里原有一条"split-sum 预积分的推导直觉"，**今天这条可以划掉**（见 [[Split-Sum Approximation]]）。

## Mastery 自测（5 条，过了即可把 PBR 标 Easy）

1. 写出 UE 的 D 与 G，**并说明 $\alpha=\text{Roughness}^2$ 与 $k=(\text{Roughness}+1)^2/8$ 这两个重映射各自的目的是什么**（一个是美术手感，一个是压"热"）。
2. **为什么 $k$ 在解析光源与 IBL 上不同？** 用错会看到什么？（答：IBL 用解析光那套 → 掠射过暗。）
3. 写出 split-sum 的两项，并说明**每一项各自忽略掉了什么**。
4. **$n=v=r$ 假设带来的具体视觉后果是什么？**（答：掠射角得不到拉长的反射——这是本文自己排的第一误差源。）
5. 在 UE 里指出：`Specular` 默认值与非金属 $F_0$ 的关系；环境 BRDF LUT 的输入是哪两个量。

> 这 5 条 + Cook-Torrance 5 条 + Kajiya 5 条 + Walter 5 条 + Schlick 5 条 = **PBR 收口 25 条**（比 9-17 预估的 20 条多出本篇的 5 条，因为实时化路径本身也是知识，不是噪音）。

## Learning Value

1. **PBR 研读线今天在"工程侧"闭合**：来源侧 9-17 已闭合（D·G·F 全部有出处），本篇补上"怎么进引擎"；
2. **一套可复用的判断法**：评估实时方案，先问"它换了哪个等价物、代价落在哪一层"（本文答案是：精度换查表、视角依赖换预存）；
3. **一条团队级的工程教训**：材质分层的成本最终会从 shader 预算溢出到 DrawCall 预算——**预算维度之间是会串的**，你在做 SABC 分级时应当假定这种串扰存在。

## Visualization

![[Split-Sum 环境光近似图解.html]]

含：原积分 → 两项拆分 → 两条查表路径（cubemap mip 链 / R16G16 LUT）→ 两个近似的误差排序 → 成本的量级对比（1024 采样 → 2 次查表）。

## Notes

- 原文：SIGGRAPH 2013 Course notes，作者 Brian Karis（Epic Games），v2，changelog 2013-08-05。本次运行**已下载原文 PDF 并逐节核对**（D/G/F 公式、split-sum 式 7、$F_0$ 提出积分的式 8、R16G16 LUT、1024 采样、$n=v=r$ 的原话、$F_0=0.04$、材质分层→DrawCall 的警告、致谢中 Lagarde 的细节）。
- 本文同时在当年课程中被其他作者并行覆盖：Gotanda（3D LUT）、Drobot（2D LUT）、Lazarov（解析拟合）——**原文自己列出了这个"同期独立发现"的名单**，说明 split-sum 是那个时间点的必然产物而非个人灵感。
