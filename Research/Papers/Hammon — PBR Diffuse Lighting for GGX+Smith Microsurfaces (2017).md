---
type: paper
title: "PBR Diffuse Lighting for GGX+Smith Microsurfaces"
authors: [Earl Hammon, Jr.]
year: 2017
published: "2017-03"
venue: "GDC 2017 (Game Developers Conference), Programming Track — slides, 193 pp."
url: "https://media.gdcvault.com/gdc2017/Presentations/Hammon_Earl_PBR_Diffuse_Lighting.pdf"
code: ""
project_page: "https://www.gdcvault.com/play/1024063/"
category: [rendering, brdf, pbr, energy-conservation, gdc]
importance: S
historical_importance: 4
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal
status: unread
---

# PBR Diffuse Lighting for GGX+Smith Microsurfaces

> **入库 2026-10-05（Run 27）。** 本库"经典队列"挂账**最长**的一篇：自 2026-09-19 进入队列（PBR 线），9-21/9-22/9-23/10-4 四次尝试原文均因 GDC Vault 旧 CDN（`twvideo01.ubm-us.net`）403 而失败，**阻塞 16 天**。今日经搜索命中**新 CDN 域名 `media.gdcvault.com`**，193 页幻灯片全文一次下载成功、逐页核对。**原文阻塞至此解除。**
> 它是 [[Multiple Scattering and Energy Compensation]]「能量账本」的**漫反射侧**：此前五条补法全部是 **specular 侧**的账，本篇给出 diffuse 侧同题答案，并顺手回答了本库两个长期悬置的问题——**"PBR specular 里的 4 是哪来的"** 与 **"UE 的 Smith G1 近似（k=α/2）从哪来"**。

## TL;DR

- **问题**：Titanfall 2 的材质系统用 Oren-Nayar 漫反射 + GGX+Smith 镜面，但两套模型**假设完全不同**（V-cavities vs Smith、球面高斯 vs GGX、参数 s∈[0,∞) vs α∈[0,1]——连标准差都无法换算），"在同一微面上用两种互不相关的粗糙度"这件事在物理上说不通；
- **答案**：从与 GGX+Smith **完全相同的微面假设**出发，**数值求解** GGX+Smith 下的漫反射 BRDF（微面间多次反弹的路径追踪）→ 发现单次散射**最多丢掉一半的光**（"Up to half the light was missing!"）→ 拟合出一个新漫反射模型；
- **副产品（对工程师最重要）**：① 新 **G2 近似**（比 G1·G1 贵 ~2 cycles、质量显著更好；**与 UE 的 k=α/2 近似在数学上同源**）；② **"4"的完整推导**（specular BRDF 分母的 4 = (L+V=2H·V) 的平方——它不是凑出来的常数，而是**从立体角测度变换 dV/dm 里掉出来的**）；③ **"理想 Lambert 缺失 5%"**（加上出射 Fresnel 与内部多次反弹机会后，归一化常数 k=21/20π=1.05/π，比纯 Lambert 大 5%）；④ 一套 shader 恒等式（省 7-10 cycles/light）。

## Problem

PBR 的 specular 半边有完整的微面模型（GGX+Smith）；**diffuse 半边却在用一套异质模型**（Oren-Nayar / Lambert / Disney），它与 specular 假设互不兼容：

| | Oren-Nayar | GGX+Smith |
|---|---|---|
| 遮挡/遮蔽假设 | V-cavities | Smith |
| 法线分布 | 球面高斯（Spherical Gaussian） | GGX |
| 粗糙度参数 | s ∈ [0, ∞)（法线斜率标准差） | α ∈ [0, 1] |
| 完美平面 | s = 0 | α = 0 |
| 斜率标准差 | s | **α² → ∞**（≠ s 的对应量） |

**连标准差都对不上**——原文结论："Oren-Nayar and Smith+GGX don't match! Can't even match standard deviations"。**"如何把 Oren-Nayar 的粗糙度 s 从 GGX 的 α 换算过来"这个问题本身无解**。且"两个 GGX 分布之和不是 GGX，所以不能顺着 mipmap 滤波 α²"——贴图管线侧的推论。

正确的问法是：**在同一个微面模型里，直接求解漫反射**。

## Historical Context

微面漫反射的能量问题此前有两个理论锚点，但都**没有被工程化**：

```text
图景位置（本库已建立）：
  微面框架 Cook-Torrance 1981 → 单次散射形态定型
  Oren-Nayar 1994 → 微面漫反射第一次系统化（但假设与 GGX 不同源）
  ├─ 本库路线 ③ Kulla-Conty 2017：specular 侧能量补偿（查表补标量）★ 已入库
  └─ ★ 本篇 2017：diffuse 侧同题 —— 用与 GGX+Smith 同源假设重做漫反射
Shirley 1997 （本文引用：解决"Lambert + Fresnel 对称化"的归一化）
```

- 本文与 [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] 是**同年（2017）、同一问题的两侧**：Kulla-Conty 补 **镜面**丢的能量，Hammon 补**漫反射**的账；
- 它同时也是 **Titanfall 2（Respawn，2016）的生产研究成果**——"GGX diffuse research done during Titanfall 2 development"——是 [[Multiple Scattering and Energy Compensation]] 谱系里**工程动机最直接**的一篇（不是为了论文而论文，是"材质系统的两套假设对不上"逼出来的）。

## Previous Work

- **Oren-Nayar 1994**（原文引用其完整版含 second bounce）：微面 + 漫反射的第一次系统化；但 NDF / 遮蔽假设与 GGX+Smith 不同源；
- **Walter 2007**（[[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]）：Smith G2 的精确形式与推导框架——**Hammon 附录逐句说"follows Appendix A in Walter's 2007, with many missing details filled in"**（本笔记的推导链即以 Walter 附录为底本补全）；
- **Shirley 1997**（*A Practitioners' Assessment of Light Reflection Models*）：给出"Lambert 漫反射 + Fresnel 入射/出射对称化"的归一化处理——本文 diffuse 的 smooth 项直接源自此；
- **Heitz 2014**（*Understanding the Masking-Shadowing Function*）：Smith 遮蔽理论的现代理解框架（本文引用）；
- Disney / Karis：`*_diffuse` 项在引擎里的现状。

## Core Idea

**一句话**：把漫反射写成"微面上的 λ=1/π 反弹 + 微面间多次弹射的完整路径追踪"，求数值解 → 得到"单次散射最多丢一半"的量化结论 → 用数学上简洁、实现上便宜的形式拟合它。

三个论证支柱：

1. **归一化与测度**：所有 PBR 的推导根基是"BRDF 在出射半球的余弦加权积分 = 1"。**为了对 δ 函数积分必须换测度（dV → dm），"4"就是这次换测度的雅可比系数里掉出来的**；
2. **多次反弹不能忽略**：漫反射光进入微结构后会在微面间反复弹射——完整 Oren-Nayar 有 second bounce，GGX+Smith 下的完整解必须**路径追踪**（不能闭式）——忽略它 = 丢掉最多 50% 的能量；
3. **对称性修复**：Lambert 漫反射与 specular 混合时 BRDF 不对称（Fresnel 只按入射方向插值）→ **出射也必须有 Fresnel**；内部反射光"被弹回表面还会再有机会逃逸"→ 需要**归一化常数**——恰好 1.05/π。

## Technical Approach

### 1. 微面 BRDF 通用形式与"4"的推导（本库从此可直接引用）

$$
\rho = \int_\Omega \rho_m(L,V,m)\, D(m)\, G_2(L,V,m)\, \frac{m\cdot L}{N\cdot L}\,\frac{m\cdot V}{N\cdot V}\, dm
$$

镜面：单个微面是完美镜面 → $\rho_m = F(L,m)\,\delta_m(H,m)/(4\,H\cdot L\, H\cdot V)$，δ 把积分消掉（m=H）：

$$
\rho_{spec} = \frac{F(L,H)\,D(H)\,G_2(L,V,H)}{4\,(N\cdot L)(N\cdot V)}
$$

**"4"的推导链**（规范化 ∫ρcosθ_V dV = 1 → 换测度到 dm）：

```text
∫ k·δ_m(H,m)·cosθ_V·dV = 1
  必须对 dm 积分 → 需要 dV/dm
  把 dV 移到 (L+V) 方向球（half-vector 的几何）：
    dV = (H·V / |L+V|²) dm
    |L+V|² = 2 + 2(L·V)  且 L+V = 2(H·V) → |L+V|² = 4(H·V)²
    ⇒ dV/dm = 4 (H·V)
  ⇒ k·(H·V)·4(H·V) = 1 → k = 1/(4 H·L H·V)
```

> **一句话**：**"4" 来自两次 L+V 的几何——|L+V| = 2(H·V)，平方出来 2²=4**。原文标题句："This 2 squared is specular BRDF's 4!" 与 "why isn't it π" 的答案：**因为它是测度变换的系数，不是归一化面积的商**。

### 2. 漫反射的求解与"丢一半"的量化

- $\rho_m = 1/\pi$（Lambertian），**没有 δ 可消积分、没有闭式解** → 数值求解（同 Oren-Nayar 论文的方法）；
- **发现**："Up to half the light was missing!"——**单次散射忽略的正是微面间二次以上的弹射**；Oren-Nayar 完整版也含 second bounce，方向一致；
- 该发现与 specular 侧的故事完全平行：[[Multiple Scattering and Energy Compensation]] 里"被挡住 ≠ 被吸收"在漫反射侧的版本是——**"光进了表面还会被弹回来"，而单次散射模型把它当成了终结**；
- 求解方式（附录）：**自研微面路径追踪器**——对数学上定义的 heightfield 模型（Smith 的"处处不可微但处处连续"怪异模型）直接路径追踪，Fresnel 事件用 Schlick 近似（$F_0=0.02$，η=1.33 电介质）、能量阈值分裂 + Russian Roulette；64 zenith 角 × 16 α 采样；把逃逸射线按视角分桶统计。

### 3. 新 G2 近似（工程核心）

Smith+GGX 的精确式：

$$
G_1 = \frac{2\,N\cdot V}{\alpha^2 + (1-\alpha^2)(N\cdot V)^2 + N\cdot V},\qquad
G_2 = \frac{2\,(N\cdot L)(N\cdot V)}{(N\cdot V)\sqrt{\alpha^2+(1-\alpha^2)(N\cdot L)^2} + (N\cdot L)\sqrt{\alpha^2+(1-\alpha^2)(N\cdot V)^2}}
$$

近似（把分母里的 $N\cdot V^2$ 放松为 lerp）：

$$
G_1 \approx \frac{2\,N\cdot V}{\mathrm{lerp}(N\cdot V,\ 1,\ \alpha) + N\cdot V}
\;=\; \frac{2\,N\cdot V}{N\cdot V(2-\alpha)+\alpha}
$$

> **🔴 最重要的发现（原文原话）**：*"Turns out, identical to Unreal's Smith: $G_1 \approx \frac{N\cdot V}{N\cdot V(1-k)+k}$, $k=\alpha/2$."*
> **即：UE 里那个 $k=\alpha/2$（Karis 记作 $k=(\mathrm{Roughness}+1)^2/8$ 的变体）不是随手拟合——它恰好是 Smith G1 的一个 lerp 形式近似。** 本库 [[Karis — Real Shading in Unreal Engine 4 (2013)]] 的"k 从哪来"问题自此有第二个答案（第一答案是 Karis 自述的拟合）。

Height-correlated 的 G2 近似（**分子在完整 specular BRDF 里与分母抵消**）：

$$
G_2(L,V) = \frac{2\,(N\cdot L)(N\cdot V)}{\mathrm{lerp}\big(2(N\cdot L)(N\cdot V),\ (N\cdot L)+(N\cdot V),\ \alpha\big)}
\quad\Rightarrow\quad
\mathrm{BRDF} = \frac{F\,D}{2\cdot \mathrm{lerp}(2\,N\cdot L\,N\cdot V,\ N\cdot L+N\cdot V,\ \alpha)}
$$

| 形式 | 成本 | 质量 |
|---|---|---|
| $G_1(L)G_1(V)$（uncorrelated） | ~4 cycles | 掠射 + 高粗糙度端偏暗 |
| **新 $G_2$ 近似**（correlated） | ~6 cycles | **粗糙电介质的掠射端显著改善；"height-correlated 只有可忽略的额外成本"** |

### 4. Lambert 的物理解释与 5% 修正（smooth 项）

- **余弦衰减的真相**：不是"能量随角度变少"，而是**几何**——内部光束各向同性，表面按 $\cos\theta$ 切割光束、又按 $1/\cos\theta$ 缩放面积，两账相抵只在 BRDF 形态上留下 $\cos$；
- **出射 Fresnel**（对称性要求）：$1-F = (1-F_0)\big(1-(1-N\cdot V)^5\big)$；
- **内部多次反弹机会** → 归一化常数：
  $$2\pi k\int_0^{\pi/2}\big[1-(1-\cos\theta)^5\big]\cos\theta\sin\theta\,d\theta = 1 \;\Rightarrow\; k = \frac{21}{20\pi} = \frac{1.05}{\pi}$$
- **"理想漫反射比 Lambert 大 5%"**——这是把"出射 Fresnel + 内部弹回几何"算进去后的**正确平滑极限**。源流：Shirley 1997。

### 5. 最终模型（三版本）

```
facing = 0.5 + 0.5·(L·V)
rough  = facing^(0.9 − 0.4·facing) · (0.5 + N·H)/(N·H)
smooth = 1.05·(1−(1−N·L)^5)·(1−(1−N·V)^5)
single = (1/π) · lerp(smooth, rough, α)
multi  = 0.1159·α
diffuse = albedo·single + albedo·multi
```

- `single` 在 α=0 时退化为 **5% 修正的 Lambert**（smooth），α=1 时退化为粗糙极限（rough）；
- `multi = 0.1159α` 是**多次弹射的能量补偿项**（直接乘 albedo 的线性修正——注意它的形态：与 [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] 一样是"算出缺多少、加回来"，但这里是**闭式线性项**而非查表）；
- 两个零售版本：**hybrid**（smooth 换 Disney 的 fd90=0.5，在 α=0 处与 Disney 完全一致）、**cheaper**（smooth 换 Lambert，省两个 pow）。

### 6. Smith 的"伟大与怪异"（理论附赠）

- **伟大**：Smith 是**唯一**满足"所有面法线可见比例相同（G 不依赖 m）"的遮蔽函数 → 也是**唯一能量守恒**的 G。任何其他 G 都会在某个方向把可见总面积算错：**偏大 → 反射过多 → 凭空造能量**；**偏小 → 反射过少 → 变相吸收**；
- **怪异**（射线追踪推导的两个自相矛盾假设）：
  1. 步进 $d\tau$ 假设"高度独立"（**处处不连续**），但又假设"heightfield 是可微函数"（**处处连续**）——"continuous nowhere, yet differentiable everywhere"；
  2. **Λ 不对称**：$\Lambda(-\mu) = -\Lambda(\mu) - 1$——**下行射线比上行射线更容易被挡**（因为更多法线朝向"迎面"方向）。这直接导致路径追踪的 BRDF **不对称**；
- **工程修法**：重定义 $\Lambda$ 使 $\Lambda(-\mu)=-\Lambda(\mu)$（"把下行的表面积重新归一化到上行"），并让**出射也有 Fresnel 事件**——两处修复后才能得到对称的、可混合的 merged BRDF（$\rho = F\rho_{spec} + (1-F)\rho_{diff}$）。

### 7. Shader 恒等式（省 7-10 cycles/light）

```
|L+V|² = 2 + 2·(L·V)
0.5 + 0.5·(L·V) = |L+V|²/4
N·H = (N·L + N·V)/|L+V|
L·H = V·H = (1 + L·V)/|L+V|
```

**可以不构造 H 就得到 N·H、L·H**：取 H 再点乘 = 19 cycles → 用恒等式 = **7 cycles**（省 12；原文口径"7-10 pixel shader cycles per light"）。

## Key Contribution

1. **同源漫反射模型**：GGX+Smith 假设下的漫反射解——single（smooth→rough 的 lerp）+ multi（0.1159α），全部闭式；
2. **新 G2 近似**：height-correlated，~6 cycles，**与 UE 的 G1 数学同源**（为"引擎里正在跑的东西"提供理论解释）；
3. **两个经典问题的答案**：「specular 的 4 从哪来」（测度变换）+「Lambert 缺的 5%」（出射 Fresnel + 内部弹射归一化）；
4. **Smith 遮蔽的完整射线追踪推导**（193 页中约 45 页是附录推导，"填上了 Walter 附录里缺失的细节"）+ 对其两个内部矛盾的诚实披露。

## Why It Works

- **"同源"是硬约束**：材质系统里 diffuse 与 specular 若假设不同源，参数就失去了共同语义（美术调一个参数会同时改变两个说不通的东西）——同源后，**同一 α 同时驱动两半，能量账才可能真正闭合**；
- **数值解先于拟合**：先用路径追踪求出"真相"（含最多丢一半的量化），再拟合便宜的闭式——这是本库"测量先于设计"家族的又一样本；
- **修复点是"对称性"而非"补丁"**：出射 Fresnel 与 Λ 归一化都是让模型**满足 BRDF 的数学公理**（互易性）——不是事后加能量，而是先让模型合法。

## Limitations

- **仍需拟合**：single 项的 rough 部分（facing 的幂与 0.9-0.4·facing 指数）与 multi=0.1159α 都是**数值拟合**产物（原文自述"tried tons of random equations… until I saw ones that I liked based on tradeoff between computation cost and fidelity"）；
- **忽略 Snell 折射对方向的影响**（附录自述）：假设表面内外光线都"各向同性均匀分布"——出射角在 $\pi/2$ 附近比实际"少"（但巧的是那里透射最弱，误差被部分抵消）；
- **仅限于不透明电介质+金属语境**（η=1.33、F0=0.02 的路径追踪设定），未覆盖 SSS 真实次表面；
- 光照上只讨论 direct lighting 的 BRDF 形态（IBL 侧的漫反射能量耦合是另一件事）；
- **是 GDC 演讲而非期刊论文**：无 formal proof、实验以视觉对比与 lit-sphere/BRDF slices 为主（无错误度量表格）；推导细节需与 Walter 2007 附录互读。

## Game Development Relevance

- **直接回答"引擎里在跑的东西是什么"**：UE 的 G1（k=α/2）与"5% 修正的 Lambert"在本文都有出处；**如果哪天要把 diffuse 换成 GGX 同源版，本文是现成的实现参考**（三个复杂度档：full / hybrid / cheaper）；
- **与 [[Multiple Scattering and Energy Compensation]] 合读**：specular 侧（Kulla-Conty，查表补）+ diffuse 侧（Hammon，闭式 lerp）**两个半场都补齐了**——"能量账本"家族现在全谱系在库；
- **性能口径**：G2 近似 ~6 cycles、"省 7-10 cycles/light"的恒等式——**多光源场景（如 MegaLights 的每像素采样预算世界）里每 light 的固定成本又有一份可参考的账**；
- **材料验收**：漫反射能量问题的验证同样可进 furnace test 家族（smooth 项 = 纯电介质球在均匀环境中的正确亮度）。

## Unreal Engine Relevance

- **G1 同源**：`k = α/2` 那个近似即本文 $G_1 \approx 2N\cdot V/(N\cdot V(2-\alpha)+\alpha)$——**UE 的遮蔽项与 Smith 推导只差一步 lerp 放松**；
- **新 G2 的集成位**：替换 `Vis_SmithJointApprox` 的 height-correlated 形态（分子抵消后 BRDF 只剩 2·lerp(...) 分母）——**改动面小、有论文同款代码结构**；
- **diffuse 模型的集成位**：UE 默认 diffuse 是 Lambert（`Diffuse_Lambert`）——换成本文模型属于 **Shading Model 级改动**（Substrate 下则是 lobe 级），可作为技术预研项；
- 与 [[Karis — Real Shading in Unreal Engine 4 (2013)]] 对照读：两篇合起来 = "UE 的 G 与 diffuse 的今天与可能的明天"。

## Technology Evolution

```text
1994  Oren-Nayar —— 微面漫反射第一次系统化（假设与 GGX 不同源）
1997  Shirley —— Lambert + Fresnel 对称化（本文 smooth 项源头，记名）
        ↓
2007  Walter —— Smith G2 精确式 + 推导框架（本文附录的底本）★ 本库已入库
        ↓
2016  Titanfall 2 生产 —— "两套假设对不上"的实际问题浮出
        ↓
★ 2017  Hammon（GDC，本篇）—— 同源求解 + 新 G2 + "4"的推导 + 5% 修正
2017  Kulla-Conty（specular 侧同题）—— 两半场同年闭合 ★ 本库已入库
        ↓
2026  现代状态：UE 的 G1 已被证明与其同源；diffuse 侧同源化仍是"未采用的技术预研"
```

## Relationships

### Based On
- [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] — Smith G2 推导框架（附录逐节沿 Walter Appendix A 补全）
- Shirley 1997 — Lambert+Fresnel 对称化与归一化（smooth 项直接来源，**记名未入库**）

### Extends
- Oren-Nayar 1994 — 首次把"微面漫反射"的假设**换成与 specular 同源**的 Smith+GGX

### Related
- [[Kulla-Conty — Revisiting Physically Based Shading at Imageworks (2017)]] — 同年、同题（能量账本）、**对偶侧**：镜面补法（查表）
- [[Multiple Scattering and Energy Compensation]] — 本文是其**漫反射侧**的账本条目
- [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] — 同一条"Smith 理论现代化"线；Heitz 2014 为本文引用
- [[Karis — Real Shading in Unreal Engine 4 (2013)]] — G1 同源确认；"4"的分母也在 Karis 的公式里
- [[Physically Based Rendering]] / [[BRDF]] — PBR 线收口材料

### Followed By
- （截至 2026-10 未见直接后继把 diffuse 侧全面工程化——**这是一个"公开答案 + 未被广泛采用"的样本**，可作技术预研观察项）

## Personal Knowledge State

- **user_level: Normal** —— 你在 PBR 线的收口窗口正好覆盖此篇：它的前置（[[Microfacet Theory]] D/G/F、[[BRDF]]、[[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]、[[Karis — Real Shading in Unreal Engine 4 (2013)]]）全部 Easy/Normal 区间；
- **读法建议（15 分钟级）**：
  1. **只读 4 个黄金点**：p.28-43（"4"的推导链）→ p.76-85（新 G2 近似 + 成本表）→ p.100-108（Lambert 物理解释 + 5% 常数）→ p.113（最终模型公式）。附录（p.148-193）**跳过**（是给推导爱好者的）；
  2. **一个人能带走的检查**："**我引擎里的漫反射与镜面，是同一套微面假设吗？**"——如果不是，能量账本两边永远对不上。

## Learning Value

- **对 [[Multiple Scattering and Energy Compensation]]**：补上"漫反射侧"的第三条一句话检验——*"被挡住的光不是被吸收了"是镜面侧的表述；漫反射侧的表述是"进了表面的光还会带着 Fresnel 与弹射几何回来，理想极限比 Lambert 大 5%"*；
- **对 [[Scalability and Quality Tiers]]**：`multi=0.1159α` 这种"一行线性补偿"的形态提供了"能量补偿当分档项"的新样本（vs Kulla-Conty 的 4KB 表现金价）；
- **方法论**：又一篇"**把引擎里已经在跑的东西追溯到原始推导**"的样本（与 Karis 的 k、GGX 的 α=Roughness² 同类）——"追溯"本身就是本库 PBR 线的主学习方法。

## Visualization

![[Hammon 2017_漫反射能量账本与那个4图解.html]]

## Notes

- **原文获取（方法经验，务必记录）**：GDC Vault 的 CDN **换域名**了 —— 旧 `twvideo01.ubm-us.net/o1/vault/...`（全 403），新 **`media.gdcvault.com/gdc2017/Presentations/...`**（直接可下，6.8MB）。**下次取 GDC 旧演讲直链：先 WebSearch 搜幻灯片标题，命中的媒体域名即新 CDN。**
- **与 Titanfall 2 的生产关系**：原文明确"研究在 Titanfall 2 开发期间完成"，致谢含 Respawn 引擎组与 GDC mentor Mark Cerny——**"研究 → 出货作品"距离最短的一类文献**；
- 本笔记的页码引用基于 193 页官方幻灯片（p.1-147 正文 + p.148-193 附录）；
- 🔴 **对"漫反射侧"一词的精确化**：本库此前把这篇记作"漫反射侧的同类问题"（[[Multiple Scattering and Energy Compensation]] 队列）——精确说法是：**"同一微面模型下漫反射的能量与分布问题"**。它与 specular 侧的能量补偿不是"同一个洞的两半"，而是**两个独立账本**：specular 的洞（G 遮挡的能量）与 diffuse 的洞（多次弹射 + 出射 Fresnel）成因不同、修法不同。
