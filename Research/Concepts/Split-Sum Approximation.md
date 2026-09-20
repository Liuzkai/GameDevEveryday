---
type: concept
user_level: Normal
aliases: [Split-Sum, Prefiltered Environment Map, EnvBRDF LUT, 环境 BRDF 查表, IBL 近似]
prerequisites: [BRDF, Physically Based Rendering, Microfacet Theory]
first_introduced: "Karis 2013（SIGGRAPH 2013 Course, Real Shading in Unreal Engine 4）；同期独立发现 Gotanda / Drobot / Lazarov"
---

# Split-Sum Approximation

> 建立于 2026-09-18。它是 [[Physically Based Rendering]] 的 Learning Gap 里最后一条"工程侧"缺口：**漫反射侧由 [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]] 的 SH 投影解决，镜面侧就靠这个。**

## Definition

环境光（IBL）下的镜面反射要对**整个半球**做一次卷积：

$$L_o=\int_H L_i(l)\,f(l,v)\,(\text{shading terms})\,dl$$

split-sum 把它近似成**两个可以离线预计算、运行时只查表的项**：

$$L_o\;\approx\;\underbrace{\text{PrefilteredColor}(R,\text{Roughness})}_{\text{预滤波环境贴图}}\;\times\;\underbrace{\Big(F_0\cdot A(\text{Roughness},n\cdot v)+B(\text{Roughness},n\cdot v)\Big)}_{\text{环境 BRDF LUT}}$$

- **第一项**：只有环境、没有 BRDF。用 GGX 重要性采样卷积环境图，按粗糙度存进 **cubemap 的 mip 层级**；
- **第二项**：只有 BRDF、把环境当作"纯白"。因为代入 Schlick 的 F 之后 $F_0$ 可以提出积分，剩下的只依赖 $(\text{Roughness},\,n\cdot v)$，于是存成一张 **2D LUT**（UE 用 **R16G16**，原文注明"精度重要"）。

**一个可复用的理解**：split-sum 不是"化简公式"，而是**把"随视角变化的部分"从积分里剥离出来，让剩下的部分可以脱离场景预先算死**。

## Core Principle

### 1. 为什么可以拆

原文的判词很短：**"对常数 $L_i$ 精确，对常见环境相当准。"**

直觉版：如果环境在各个方向亮度相同，那么"环境 × BRDF × 余弦"的积分就等于"环境（常数） × BRDF 积分"，拆分无损。真实环境是低频到中频的（天空、地面、墙面），所以**低频近似**成立。

**这也解释了它的失效场景**：环境里有**高频亮源**（太阳、窗户、灯带）时，反射的方向性由这张环境图的高频细节决定，拆分后的乘积会把"光源不该出现在这里"的错误放大成可见的错位高光。

### 2. 两个近似，哪个更要紧（这是本篇最容易被搞反的地方）

| 近似 | 内容 | 代价 | 严重程度 |
|---|---|---|---|
| **① split-sum 本身** | 把一次积分拆成两个独立积分相乘 | 高频环境 + 粗糙表面的组合下不准 | **较小** |
| **② $n=v=r$** | 预滤波时假设法线 = 视线 = 反射方向 | **掠射角得不到拉长的反射**；预滤波结果无法随视角变化 | **较大**（Karis 原文原话："actually the larger source of error for our IBL solution"） |

> **第二条的物理后果值得单独记住：真实世界掠射时反射会被"抻长"（Fresnel 让掠射反射增强，且可见的微面方向被压扁到一个窄带）。$n=v=r$ 把它抹平了。** 这就是为什么很多游戏的湿地面/车漆在极掠视角下"不够长"。

### 3. 为什么非得查表不可

- 直接可用的是重要性采样：原文用 **1024 采样**（Hammersley + GGX 分布）能算出接近参考的结果；
- 但**每像素要在多张环境贴图之间混合（局部反射）**，实际只负担得起 **1 个采样**。

→ **1024 次采样 vs 1 次 cubemap 采样 + 1 次 LUT 采样。** 这就是实时 IBL 的全部秘密：**把时间预算换成显存预算**。

## Prerequisites

- [[BRDF]] / [[Microfacet Theory]]：要明白拆的是 $D\cdot G\cdot F/(4(n\cdot l)(n\cdot v))$ 里的哪部分
- [[Physically Based Rendering]]：split-sum 是"实时 PBR 三件套"（微面 BRDF + 金属度工作流 + IBL）里的 IBL 半边
- [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]]：**$F_0$ 能提出积分的唯一原因是 F 采用了 Schlick 的线性形式**（$F=F_0+(1-F_0)(1-v\cdot h)^5$）。如果 F 是精确 Fresnel（含偏振与复折射率），这一步做不了——**"用一个近似换来的可分解性"，这是 Schlick 的第二个红利**

## Historical Evolution

```text
半球积分（无法实时）
        ↓
预滤波环境贴图 / cubemap mip 链（GPU Gems 时代，含 CubeMapGen）
        ↓
★ Karis 2013 —— split-sum 分解 + 环境 BRDF LUT（本文概念入库）
   · 同期独立发现：Gotanda（3D LUT）、Drobot（2D LUT）、Lazarov（解析拟合）
        ↓
全引擎默认（UE / Unity / 自研）
        ↓
★ 2019  Fdez-Agüera —— 发现 "缺口就在这张表里"：
   EnvBRDF LUT 的两个通道相加 = 单次散射方向 albedo
   → E_ms = 1 − E_ss，零新增资源补回多次散射
   （见 [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]]）
        ↓
移动端：Analytic EnvBRDF（纯解析拟合，省掉 LUT 采样）
        ↓
2026 —— 神经路线绕开它：DLSS 5 从画面侧补光照与材质真实感
        （见 [[DLSS 5 — Generative Neural Rendering]]）
```

**"同期四个人独立做出几乎一样的方案"**这件事本身是个信号：**2013 年不是谁灵光一闪，而是"实时 IBL 必须查表"这个结论到了该出现的时候。**

## Important Papers

| 论文 | 角色 |
|---|---|
| [[Karis — Real Shading in Unreal Engine 4 (2013)]] | **提出者**（本文概念的来源） |
| [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]] | **对偶的另一半**：漫反射侧用 SH 投影（9 个系数）而不是查表；两者合起来才是完整 IBL |
| [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] / [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] | 被拆的那个积分里 D 与 G 的来源 |
| [[Schlick — An Inexpensive BRDF Model for Physically-based Rendering (1994)]] | F 的可分解性来源 |

## Related Concepts

- [[Physically Based Rendering]] — split-sum 是它的 IBL 半边
- [[BRDF]] — 拆的对象
- [[Microfacet Theory]] — $D\cdot G$ 的积分在 LUT 里被预积分掉了
- [[Global Illumination]] — **边界**：split-sum 只算**环境光**的镜面反射（天空盒 / 反射捕获），**不含场景间间接光**；真正的动态 GI（Lumen / MegaLights / 神经 GI）是另一条线
- [[Neural Global Illumination]] — 对偶的技术路线：一边是"预积分查表"，一边是"学习/推理解算"
- [[Multiple Scattering and Energy Compensation]] — **本概念的免费搭车项**：缺口 $1-E_{ss}$ 所需的 $E_{ss}$ 就是本概念的 LUT 两个通道之和（[[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]]）

## Technologies

- **Sky Light / Reflection Capture**（UE）：预滤波 cubemap 的生产者
- **Analytic EnvBRDF**：把 LUT 拟合为解析式（Lagarde 一派），移动端省一次贴图采样
- **Substrate（UE 5.2+）**：每个 lobe 都需要自己的 IBL 采样 —— **lobe 数 × IBL 采样 = 隐藏预算乘法**

## Game Applications

| 用途 | 依赖 |
|---|---|
| 金属/车漆/湿地面 | 预滤波镜面 IBL + LUT；**掠射"抻长"缺失在这里最明显** |
| 天空光下的室外场景 | Sky Light 漫反射（SH）+ 镜面（split-sum） |
| 移动端 | cubemap mip 数 + LUT 分辨率是**分档旋钮**；再低档改解析 EnvBRDF |

**给你的预算结论（三条，可直接写进分档表）：**

1. **它是"固定开销"型预算**：一次 cubemap 采样 + 一次 LUT 采样，与实际光源数无关 —— 但**每多一个 lobe 就多一份**（Clearcoat / Anisotropy / Hair 都要自己那份）；
2. **它的分档变量不是公式，而是资源**：**cubemap 分辨率/mip 数 + LUT 精度/分辨率 + 是否改用解析拟合**；
3. **它和动态灯光的预算是此消彼长的**：环境光做得好，动态光源的需求可以降；这正是 MegaLights / 神经 GI 路线要挑战的位置。

## Personal Knowledge

- **user_level: Normal**。判据：你在 UE 侧天天用 Sky Light 与反射捕获（Easy 侧工程直觉已有），新的是**"拆成什么、代价在哪、误差在哪"**。
- 与 9-14~9-17 的 PBR 主线的衔接：**来源侧（D·G·F）已在 9-17 闭合，工程侧（怎么进引擎）由 2026-09-18 的 [[Karis — Real Shading in Unreal Engine 4 (2013)]] 补上，本概念是其中最核心的一块。**

## Learning Gap

- ✅ **残留一条已闭环（2026-09-19 → 09-20）**：**多次散射能量补偿**（高粗糙度下单散射 GGX 丢能量）已由 [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]]（新增查表的通用解）、[[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]]（精确真值）、[[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]]（**复用本概念的 LUT**，零新增资源）三篇补齐。概念汇总见 [[Multiple Scattering and Energy Compensation]]；
- **🔴 本次最有价值的一条（2026-09-20）**：**split-sum 的产物不只是"一个近似"，而是"三样东西"** —— 预滤波 cubemap、EnvBRDF LUT、**以及一个免费附带的能量补偿项**（因为 $E_{ss}=f_a+f_b$ 已经在 LUT 里）。**这意味着 IBL 镜面半边的"固定开销表"要比原先记的多一行，而这一行成本 ≈ 0**；
- **一条实践差距**：你库里没有"同一场景 IBL 开关 + 精度分档"的实测数据。**这一条只能自己测**，见 Next Step。

## Next Step

1. **可做的一次实测（不需读论文）**：在 NGR 里取一个代表性场景，测 **Sky Light 镜面开/关**、以及 **LUT 精度从 R16G16 降到 R8G8** 两组的 GPUTime 与显存差。这一组数字能直接支撑你的"反射类特效/材质分档"判断；
2. 视觉验证 **$n=v=r$**：找一个高粗糙度金属球 + 强环境（天空盒有太阳），对比掠射角下的反射"长度"。**看得见就说明这条近似在你项目里是可感的**；
3. ~~经典补缺：Kulla-Conty 2017~~ ✅ 已入库（9-19）；~~Fdez-Agüera 2019~~ ✅ 已入库（9-20）。**下一步剩下的相关经典是 **Hammon 2017**（漫反射侧）与 **d'Eon 的 Hitchhiker's Guide**；**
4. **新增（与上面第 1 条合并做）**：顺手确认**引擎当前的多次散射补偿状态** —— 用同一组球做 furnace test（做法见 [[多次散射_五条补法路线与实时落地图解]] 第五节）。**如果发现没开或没有，这一篇的 GLSL 就是现成的补法。**

## Notes

- 2026-09-18 由 [[Karis — Real Shading in Unreal Engine 4 (2013)]] 建立，来源为该文第 3 节（Image-Based Lighting / Split Sum Approximation 式 7 / Environment BRDF 式 8）与实现代码片段。
- 与 [[Microfacet Theory]]、[[BRDF]] 一起，构成"微面 BRDF 三件套"的完整链条：**理论 → 三个因子的实现形式 → 成像（IBL）的最后一步**。
- **2026-09-20 的重要补充**：本概念的 LUT **不只是为单次散射服务的**。它的两个通道相加就是 $E_{ss}$，因此**它同时是多次散射能量补偿的输入**。
  → **它从"一个近似"升级为"一个可以重复提取信息的固定资源"**。这改变了它在预算表里的定位：**不是"为 IBL 付一次钱"，而是"付一次钱、拿到两件事"**。
- ⚠️ **勘误提醒**：本概念的后续实现 [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] **有官方勘误**（JCGT 论文页，2018-02-01 修订版）。**引用与照抄该文时请走修订版。**
