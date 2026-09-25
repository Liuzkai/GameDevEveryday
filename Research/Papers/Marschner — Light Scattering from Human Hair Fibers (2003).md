---
type: paper
title: "Light Scattering from Human Hair Fibers"
authors: [Stephen R. Marschner, Henrik Wann Jensen, Mike Cammarano, Steve Worley, Pat Hanrahan]
year: 2003
published: "2003-07 (SIGGRAPH 2003)"
venue: "ACM SIGGRAPH 2003 Papers / ACM TOG 22(3): 780–791"
url: "https://doi.org/10.1145/1201775.882345"
code: ""
project_page: "https://www.cs.cornell.edu/~srm/publications/SG03-hair-lr.pdf"
category: [hair, brdf, appearance, scattering, offline-rendering]
importance: S
historical_importance: 5
game_relevance: 4
production_readiness: Industry Adopted
user_level: Normal
status: unread
aliases: [Marschner 2003, Marschner hair, R TT TRT, 毛发散射模型]
tags: [hair, brdf, classic, scattering, pbr-boundary]
---

# Light Scattering from Human Hair Fibers (Marschner 2003)

## TL;DR

**这是毛发着色的物理锚点，也是"微面 BRDF 失效边界"的正式定义。**

一句话：**把一根头发当成"半透明、内部有吸收、表面带倾斜角质鳞片的椭圆柱体"**，光在其中走三种路径，得到三个 lobe：

| lobe | 路径 | 视觉现象（← 这是你库里缺的那半） |
|---|---|---|
| **R**（p=0） | 表面直接反射 | **白色高光**（保偏），沿发丝拉成一条带，**绕发一周都存在** |
| **TT**（p=1） | 折射进、折射出 | **明亮的背光透射**——发丝在逆光下"亮起来"的那道光 |
| **TRT**（p=2） | 折射进、内壁反射一次、折射出 | **有色次级高光**（光穿过了有色素的内部；**黑发上看不到它**），且**离焦发散**（去偏） |

**最有信息量的一条**：角质层鳞片相对柱面**朝发根方向倾斜约 3°**，这个倾斜把 R 与 TRT 的锥面**推向相反方向**——**这就是真实头发上"两条高光"的物理来源**，也是 [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)|Kajiya-Kay]] 式模型永远做不出来的东西。

## Problem

2003 年之前，实时与离线都在用 **Kajiya-Kay 1989** 的经验模型：把头发当圆柱，放一个恒定的镜面锥 + 一个余弦漫反射项。它的三个硬伤：

1. **不保能量**（对基于物理的渲染是致命的）；
2. **当作不透明圆柱**：没有透射，因此**逆光下头发不会亮**——而这是头发最标志性的视觉特征；
3. **预测不出实测里的"第二高光"**。

实测证据（本文复述前人工作）：Bustard & Smith 1991 观察到**双高光**，且**第一峰保偏、第二峰去偏**，**黑发上第二高光不存在**。这两条观测直接指向"第二高光来自穿过色素层的内部反射"。

## Core Idea

**纤维不是面，是细长散射体——所以建模方式必须换。**

| | 微面 BRDF（面） | 纤维散射（本文） |
|---|---|---|
| 度量 | 辐照度/辐射度按**面积**定义 | 曲线辐照度/曲线强度按**长度**定义（$\bar{E}$、$\bar{L}$，单位是"每单位长度"） |
| 积分域 | 上半球 | **整个球面**（光会穿过去从背面出来） |
| 散射函数 | $f_r$ | $S(\omega_i,\omega_r) = d\bar{L}_r / d\bar{E}_i$ |
| 关键几何 | 法线 $n$、半程向量 $h$ | 发丝切向 $u$、法平面 $(v,w)$ |
| 参数 | 粗糙度、$F_0$ | 倾角 $\alpha_R$、纵向宽度 $\beta_R$、折射率 $\eta$、吸收 $\sigma_a$、椭圆度 $a$ |

**这个"换成线度量"的改动，是它无法被塞进标准 PBR 管线的根本原因**——不是公式更复杂，而是**它的输入输出单位都不一样**。

## Technical Approach

### 1. 为什么可以拆成两个 2D 函数（全文的数学骨架）

圆柱的对称性给出两条性质（原文明确引用 Marcuse 1974 / Adler et al. 1998 等前人）：

- 以某个倾角进入介质圆柱的光线，**必然以同一倾角射出**；
- 所以从某个入射方向来的所有光线，**与法平面的倾角都相同**——**3D 问题在法平面内降到 2D**。

于是 4D 散射函数**可分离**：

$$S(\varphi_i,\theta_i;\varphi_r,\theta_r)=\frac{1}{\cos^2\theta_d}\sum_{p} M_p(\theta_h)\,N_p(\varphi)$$

- $M_p$：**纵向**散射函数（管 $\theta$）；
- $N_p$：**方位**散射函数（管 $\varphi$）；
- $\theta_h=(\theta_i+\theta_r)/2$ 为半角，$\theta_d=(\theta_r-\theta_i)/2$；
- $1/\cos^2\theta_d$ 是**锥面的投影立体角修正**——一束光被散到一个圆锥上，观者看到的亮度与几何有关。

**这条"对称性 → 降维"的思路和 [[Microfacet Theory]] 里"只有 $h=m$ 的微面参与着色"是同一种手法：先找守恒量，再降维。**

### 2. 三个 lobe 的纵向项：一个高斯 + 一个倾角

$$M_R(\theta_h)=g(\beta_R;\ \theta_h-\alpha_R),\quad
M_{TT}=g(\beta_{TT};\ \theta_h-\alpha_{TT}),\quad
M_{TRT}=g(\beta_{TRT};\ \theta_h-\alpha_{TRT})$$

$g$ 是单位积分、零均值的 lobe（实现用归一化高斯，标准差 $\beta$）。**每个 lobe 只差一个中心偏移与一个宽度**：

$$\alpha_{TT}=-\alpha_R/2,\qquad \alpha_{TRT}=-3\alpha_R/2$$

**正负号就是"双高光"的全部秘密**：R 往发根偏，TT/TRT 往发梢偏，TRT 偏得最多。

### 3. 方位项：绕着彩虹问题解路径

$N_p$ 来自对介质圆柱的射线追踪：入射光按 Snell 折射进圆截面，第 $p$ 次内部事件后出来。路径的偏折角是

$$\varphi(p,h)=2p\gamma-2\gamma+p\pi$$

其中 $h$ 是入射高度（碰撞参数）、$\gamma$ 是折射角，$p$ 为内部路径段数：**R=$p$0，TT=$p$1，TRT=$p$2**。

**要算某个出射方位 $\varphi$ 有多少光，就要解 $\varphi(p,h)-\varphi=0$ 的根 $h(p,r,\varphi)$**：

- $p=0,1$：**单根**（一条路径）；
- $p=2$（TRT）：**可能有一根，也可能三根**——函数 $\varphi(p,h)$ 在 $h$ 上是光滑的，**单根到三根的过渡是一个 fold（折返）**，而"折返处"正是 Descartes 当年用来解释彩虹的那条条件。

> **这就是物理来源：TRT 的第二高光有时候是一个 caustic（焦散）**，因为多条路径汇聚到同一方向。黑色头发看不出来，只是因为它被内部吸收吃掉了——**R 与 TRT 的差别归根到底是一个玻尔兹曼式的"光在里面走了多远"**。

**吸收怎么进来的**：内部吸收用 Beer 定律式的指数衰减（吸收系数 $\sigma_a$），路径越长衰减越大 → **TRT 走的路最长，因此最先消失**。灰发/白发（$\sigma_a\to$ 很小）反而把这些效应全暴露出来。

**椭圆截面**：论文不走几何，而是**把椭圆度折算成"有效折射率"**并用圆形结果近似：

$$\eta^*_1=2(\eta-1)a^{2}-\eta+2,\qquad \eta^*_2=2(\eta-1)a^{-2}-\eta+2,\qquad
\eta^*(\varphi_h)=\tfrac{1}{2}\Big[(\eta^*_1+\eta^*_2)+\cos(2\varphi_h)(\eta^*_1-\eta^*_2)\Big]$$

即在 $N_{TRT}$ 的计算里用与方位相关的 $\eta^*$ 代 $\eta$。

### 4. 完整参数表（Table 1 原表，典型值区间）

| 参数 | 含义 | 典型值 |
|---|---|---|
| $\eta$ | 折射率 | **1.55** |
| $\sigma_a$ | 吸收系数（R,G,B） | 0.2 → ∞ |
| $a$ | 椭圆度（轴比） | 0.85 – 1 |
| $\alpha_R$ | R lobe 纵向偏移 | **−10° ～ −5°** |
| $\alpha_{TT}$ / $\alpha_{TRT}$ | 分别为 $-\alpha_R/2$ / $-3\alpha_R/2$ | +2.5°～+5° / +7.5°～+15° |
| $\beta_R$ | R lobe 纵向宽度（标准差） | 5° – 10° |
| $\beta_{TT}$ / $\beta_{TRT}$ | $\beta_R/2$ / $2\beta_R$ | 2.5°–5° / 10°–20° |
| $k_G$ | glint 缩放 | 0.5 – 5 |
| $w_c$ | caustic 方位宽度 | 10° – 25° |
| $\Delta\eta'$ / $\Delta h_M$ | caustic 合并的过渡区间 / 强度上限 | 0.2–0.4 / 0.5 |

**注意这张表的形状本身就是一条工程结论**：**TRT 的宽度是 R 的两倍**（$\beta_{TRT}=2\beta_R$），所以**次级高光天然比主高光"糊"**——这和"hair cards 上用一张贴图糊弄"的做法方向一致，也解释了为什么只做 R 的近似模型在近景会立刻露馅。

## Key Contribution

1. **模型**：把头发建模为"半透明椭圆柱 + 倾斜鳞片 + 内部吸收"，得到 R/TT/TRT 三 lobe 的解析散射函数；
2. **测量**：首次给出**全 3D 半球散射测量**，并发现两个新现象——主高光**绕着发丝一圈都存在**，而次级高光**只存在于朝向光源的一侧**；以及**一对离面尖峰（glints）**；
3. **降维与近似**：给出 4D → 2×2D 的可分离形式，以及椭圆截面、caustic 平滑等可直接实现的近似。

## Why It Works

1. **对称性守恒**：圆柱的倾角守恒使 3D 降为 2D，这是模型能被工程实现的前提；
2. **把"颜色"交给物理**：色素不再是一个美术参数，而是 $\sigma_a$（三个通道），路径越长吸收越多——**"TRT 有色、R 无色"是自动出来的，不需要分两条 shader**；
3. **把双高光交给几何**：两条高光的分离量等于倾角 $\alpha_R$，**而这个角度是可测量的物理量（约 3° 的鳞片倾斜）**。

## Limitations

1. **单散射**：不含纤维之间的多次散射（人发束的"体积感"主要来自它）→ 工程上要靠 **dual scattering 一类近似**补（后续工作，非本文）；
2. **只算到 p=2**：显式忽略 p>2；
3. **caustic 需要人为平滑**：TRT 的三根分支会产生无限亮度的焦散，论文专门做了 caustic 合并（参数 $\Delta\eta'$、$\Delta h_M$、$w_c$），**这是一处明确的"物理妥协"**；
4. **参数要手调**：$\alpha_R$、$\beta_R$ 不是从扫描数据自动来的；
5. **单位不同**：曲线度量使它**无法直接塞进以面积为度量的标准材质管线**（这就是为什么引擎需要一个**独立的 Hair shading model**）。

## Game Development Relevance

**4/5：不是"能不能用"，而是"分档时哪些 lobe 可以砍"。**

| 档位 | 可承受的近似 | 砍掉什么会立刻看出问题 |
|---|---|---|
| 发丝档（PC_High） | R + TT + TRT（可含多次散射近似） | — |
| 发片档（PC_Low / Android_High） | R + 简化 TT | **砍 TT → 逆光死板**（角色轮廓光消失，这是最显眼的一刀） |
| 移动低档 / 远景 | 只留 R（等价退化成各向异性高光） | 保留"沿发丝的带子"即可，观众看不出缺 TT/TRT |

**给你的三个直接结论：**

1. **TT 是最不能砍的那一项**：它是头发"活起来"的唯一来源，成本却只是一个前向 lobe；
2. **TRT 是最适合砍的那一项**：宽、暗、只在近景可辨（$\beta_{TRT}=2\beta_R$）；
3. **$\sigma_a$ 是三通道的**——**这给了你一个"用颜色换预算"的位置**：低档把 $\sigma_a$ 饱和度压低，可以在同亮度下少算一个 lobe。

另外，本模型解释了 [[Real-Time VFX Performance Budgeting]] 的一个老朋友：**毛发与 VFX 都以 OverDraw 为主要成本**，但原因不同——VFX 是因为叠层，毛发是因为**每根发丝都是细长几何 + 大量 alpha 覆盖**。

## Unreal Engine Relevance

- UE 有**独立的 Hair Shading Model**（不是 Lit 的变体），走的就是 Marschner 系路线：R/TT/TRT 三项 + 多次散射近似；**具体系数与近似细节以引擎文档/源码为准，本笔记不做断言**；
- **Groom 资产**用发丝几何 + 该 shading model；**发片**用 masked 材质，通常只能拿到"类 R + 贴图伪造的 TT"；
- 与 [[Niagara]] 的交界面在**发丝物理**（groom 的物理求解），不在着色。

## Technology Evolution

```text
Kajiya-Kay 1989 —— 各向异性经验模型（圆柱 + 恒定镜面锥），实时默认 30 年（**2026-09-25 已入库**：[[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]]）
        ↓
★ Marschner et al. 2003 —— R/TT/TRT 半透明椭圆柱物理模型（本文）
        ↓
Scheuermann 2004 / Weta 一路 —— 把 TRT 做成可实时计算的近似
        ↓
Dual Scattering（次表面/多次散射近似）—— 补单散射缺失的体积感（后续工作）
        ↓
UE Groom / TressFX / HairWorks —— 进引擎，只服务高端档
        ↓
★ 2026 —— 两条新路同时施压：
        · 资产侧：[[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]] 让发片能自动升档到发丝
        · 画面侧：[[DLSS 5 — Generative Neural Rendering]] 官方把 hair 列为神经增强对象
```

## Relationships

### Based On

- 前人测量：Stamm et al. 1977（$\eta\approx1.55$）、Bustard & Smith 1991（双高光、偏振）、Robbins 1994（鳞片结构）
- 介质圆柱散射与彩虹理论：Descartes → Humphreys 1964；Marcuse 1974 / Adler et al. 1998

### Replaces / Contrasts

- [[Kajiya-Kay — Rendering Fur with Three Dimensional Textures (1989)]]（各向异性经验模型）：本文明确以"它预测不出实测现象"为立项理由 —— **已入库（2026-09-25）**：它给出的是"失效后工程上用的是什么"（$\sin(t,l)$ 漫反射 + 圆锥 Phong 高光，实时用了 30 年）；与本文构成毛发着色的"经验侧 ⟷ 物理侧"两端
- [[Microfacet Theory]]：**失效边界**——微面假设表面由面组成，而纤维是细长散射体且有透射

### Related

- [[Hair Rendering]]（本笔记是它的理论锚点，已从"最大缺口"变为"已补齐"）
- [[BRDF]]：纤维散射函数 $S$ 与 $f_r$ **单位不同**（曲线度量 vs 面积度量）
- [[Participating Media]]：**同类思路**——都是"光在体内走"，都用 Beer 式吸收（一个是体积，一个是纤维内部）
- [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]：本文是**它的对立面**——不建立在微面分布上

## Personal Knowledge State

- **user_level: Normal**。判据：你已掌握"发片便宜 / 发丝贵"与分档判断（Easy 侧），**新的是三个 lobe 分别对应什么视觉现象**——这正是 9-17 建立的 [[Hair Rendering]] 里 Learning Gap 第 2 条，今天补上。
- 与你 PBR 主线的关系：**它是 D·G·F 的失效边界**。学到"一个理论在哪失效"与学到"它怎么用"同等重要。

## Mastery 自测（5 条，过了即可把毛发着色标 Easy）

1. 说出 **R / TT / TRT 三条路径**，以及各自对应的**一个视觉现象**。
2. **为什么次级高光在黑色头发上看不见？**（答：TRT 路径最长，被内部吸收 $\sigma_a$ 吃掉——吸收是路径长度的函数。）
3. **两条高光为什么会分开？** 用 $\alpha_R$、$\alpha_{TT}$、$\alpha_{TRT}$ 解释（答：鳞片朝发根倾斜，三个 lobe 的中心偏移不同号）。
4. 为什么毛发**不能**直接用标准微面 BRDF？（答：细长散射体 + 透射，且**度量单位是"每单位长度"而非"每单位面积"**。）
5. 给你一个移动端角色，**哪一项先砍、哪一项最后砍**？说明理由（答：先砍 TRT——宽、暗、远处不可辨；TT 最后留——逆光轮廓光靠它）。

## Learning Value

1. **补齐 [[Hair Rendering]] 的最大缺口**，该概念从"有尾无头"变为"头尾俱全"；
2. **一个可迁移的判据**：**一个模型生效的范围，等于它假设的适用范围**——微面假设"表面由面组成"，纤维违反它，于是整套路数失效；
3. **一条分档判据**：**按"视觉可辨性"排优先级，而不是按"物理完备性"**（TRT 物理上很漂亮，但在你的预算里它排最后）。

## Visualization

![[Hair R·TT·TRT 三叶散射图解.html]]

含：纤维截面与三条光路、倾角如何把两个高光推开、参数表、lobe↔视觉现象对照表、以及"为什么微面 BRDF 不适用"的度量对比。

## Notes

- 引用信息已核实：ACM SIGGRAPH 2003 Papers，pp. 780–791；ACM TOG 22(3)；DOI 10.1145/1201775.882345。作者单位：Cornell（Marschner）、UCSD（Jensen）、Stanford（Cammarano、Hanrahan）、Worley Laboratories（Worley）。
- 本次运行**已下载原文 PDF 并核对**：$\varphi(p,h)=2p\gamma-2\gamma+p\pi$、$M_p$ 高斯形式、$\alpha_{TT}=-\alpha_R/2$、$\alpha_{TRT}=-3\alpha_R/2$、Table 1 全部典型值、$\eta^*$ 椭圆度近似、"R 白 / TRT 有色 / 黑发无次高光"、"out-of-plane glints"、"Descartes rainbow" 均在原文中逐条确认。
