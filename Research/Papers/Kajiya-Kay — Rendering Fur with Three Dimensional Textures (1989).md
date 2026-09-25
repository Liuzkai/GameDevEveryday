---
type: paper
title: "Rendering Fur with Three Dimensional Textures"
authors: [James T. Kajiya, Timothy L. Kay]
year: 1989
published: "1989-07 (SIGGRAPH '89)"
venue: "ACM SIGGRAPH '89 / Computer Graphics 23(3): 271–280"
url: "https://doi.org/10.1145/74333.74361"
code: ""
project_page: "https://www.cs.drexel.edu/~david/Classes/CS586/Papers/p271-kajiya.pdf"
category: [hair, rendering, texture, volume-rendering, classical]
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: Normal
status: unread
aliases: [Kajiya-Kay 1989, Kajiya-Kay hair, texel, 三维纹理渲染毛发]
tags: [hair, classic, rendering, texture, lod, pbr-boundary]
---

# Rendering Fur with Three Dimensional Textures (Kajiya-Kay 1989)

## TL;DR

**这是毛发实时着色的源头，也是本库 [[Hair Rendering]] 缺的最后一块——"实时侧那一半"。**

一句话：**把"细到不该用几何画的表面"整体改画成一个三维纹理——texel**；texel 里存的不是几何，而是**密度场 + 帧场 + BRDF 场**。渲染时沿光线把"每个微表面"的贡献**求和**（不是积分）出来。

三个必须记住的点：

1. **texel = 三元组 $(\rho, B, \Psi)$**：投影面积密度 + 帧束$[n,t,b]$ + 双向反射函数场。它是 [[Reeves — Particle Systems (1983)]] 粒子系统在光线追踪里的对偶（原文自述："this paper represents the extension of particle systems to ray tracing"）；
2. **毛发着色两条公式（今天的 shader 里还在用）**：
   - 漫反射：$\tilde{I}_{diffuse} = K_d\,\sin(t, l)$ —— **正比于光方向与发丝切线的夹角正弦**。发丝切线直指光源处是暗的（反向是亮的）；
   - 高光：$\tilde{I}_{specular} = k_s\cos^p(\theta - \theta')$ —— 一个 **Phong 式的圆锥高光**（因为圆柱你从法线方向看高光是一个"锥"，不是点）；
3. **"渲染时间与它代表的几何复杂度无关"**（原文关键句）—— 这是**预算与几何解耦**的最早正式表述，也是今天"发片代替发丝"的哲学起点。

以及一条被 2026 年重新点亮的原话：**"近距离时应从 texel 切回真实几何"** —— **这就是 37 年后的"发片 ↔ 发丝"分档原则**。

## Problem

1989 年，两根硬骨头：

1. **毛皮不能靠几何画**。用几何（上千根圆柱）画毛发会遭遇"intractable aliasing problem"（不可解的走样）："hairs tend to look like spines"（像刺不像毛）。原因：**细节的尺度用错了工具**——"the aliasing problem arises because geometry is used to define surfaces at an inappropriate scale"；
2. **体积密度也不够用**。Blinn 1982 的 volume density（云雾模型）看起来是最自然的候选——把毛发看作密度场——但本文给出了一个漂亮的**反证**（见下）。

## Historical Context

- **前史**：Csuri 1979（几千个多边形画烟雾/毛皮）、Weil 1986（几千根 Lambert 圆柱画布料）、**Reeves 1983 粒子系统**（"成功的原因是它'没有几何'"）、**Blinn 1982 体积密度**（云、尘埃、土星环）、**Kajiya & Von Herzen 1984**（非均匀介质的体积光追）、Miller 1988（几何 + 复杂光照画毛茸茸的动物，但复杂度仍依赖毛发数）；
- **本文的问题意识**：作者认为"画家幻觉"（painter's illusion）是对的方向——**细节应该由纹理与光照模型给出，而不是几何**；
- **直接后代**：实时渲染中流行三十余年的 "Kajiya-Kay shading model"；GPU Gems 1 的毛发章节；直到 2003 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 用物理模型替代它的经验公式——**但它在低配档从未退场**。

## Previous Work（与三个前驱的关系）

| 前驱 | 关系 |
|---|---|
| [[Reeves — Particle Systems (1983)]] | **对偶**：粒子系统"没有几何地渲染"启发本文；本文是它在 ray tracing 中的版本。原文："particle systems and texels are complementary, e.g. particle systems could be used to generate texel models" |
| Blinn 1982 体积密度 | **母体**：texel 从 volume density 推广而来，但推翻了"直接复用"的可能性（见 Core Idea） |
| Kajiya & Von Herzen 1984 | **技术底座**：非均匀介质的光追方程，本文以其为基础改造 |

## Core Idea

### 1. 为什么不能直接把体积密度"改名"当表面用（全文最漂亮的论证）

拿一个**无限薄的理想表面**放进体积密度：

- **透明度计算**：光程 $\int \rho\, ds$ 只在单点非零 → 积分为 0 → 表面完全透明；
- **亮度计算**：被积函数同样只在单点非零 → 贡献为 0。

**结论：一个完全反光、完全不透明的表面，在体积密度里是隐形的。** 因为体积密度的物理量是"相对体积"，而**表面的不透明度与亮度根本不取决于体积**（可以体积为零、却 100% 反射）。

修正方式：**密度换成"投影面积密度"（Dirac delta 式），积分换成"遍历微表面的求和"**。这就是 texel 的本质：

$$T = \sum_{\text{microsurfaces}} \exp\!\big(-\tau \textstyle\sum \rho\big), \qquad B = \sum_{\text{microsurfaces}} \Big[\text{attenuation} \times \sum_i I_i \cdot \Psi \Big]\times \rho$$

（式 3 / 4：把体积渲染的**积分**换成**形式求和**——每个微表面贡献一个离散项。）

### 2. texel 三元组的定义

| 成分 | 符号 | 含义 |
|---|---|---|
| 密度 | $\rho(x,y,z)$ | **投影面积**密度（该体积单元内微表面覆盖的投影单位面积比例；本文取各向同性近似——本应依赖视向） |
| 帧束 | $B = [n(x,y,z), t(x,y,z), b(x,y,z)]$ | 微表面的代表朝向（法线/切线/副法线场） |
| 反射函数场 | $\Psi$ | 微表面的 BRDF（毛发中简化为常数——所有发丝同色） |

### 3. 毛发的特化

- 每根头发当作**无限细的圆柱面** → 帧束里**只有切线 $t$ 参与光照**（法线绕切线一圈全都有）；
- texel 只需存 $(\rho, t)$ 两个场。

## Technical Approach

### 1. 渲染算法（蒙特卡洛分层采样，5 步）

1. 求光线与所有 texel 边界交点，得到 $[T_{near}, T_{far}]$；
2. 按参考长度 $L$ 把光线分段；
3. 透明度初始化为 1；
4. 对每段：随机取点 → 向每盏灯发 shadow ray 求入射 → 由光照模型算亮度 × 当前透明度累加 → 透明度乘以该段透射系数；
5. 末段按**实际长度**归一化（原文明确：不这样做会引入偏差，让体积边缘"显得比实际更不透明"）。

### 2. 毛发光照模型的推导（本文最被引用的部分）

**几何**：发丝 = 位置 $x_0$ + 切线 $t$；光方向 $l$；视方向 $e$。把 $l$ 投影到垂直 $t$ 的平面得到 $l'$，再取 $b = l \times t$ 构成正交基——**这一步等价于说"所有光照计算发生在与发丝垂直的平面上"**。

**(a) 漫反射 = 对可见半圆柱积分 Lambert**

$$\tilde{I}_{diffuse} = K_d\, \frac{l - (t\cdot l)t}{\|l-(t\cdot l)t\|}(\text{积分后}) = K_d\,\sqrt{1-(t\cdot l)^2} = K_d\,\sin(t,l)$$

原文注释很直白："Thus the diffuse lighting component is proportional to the sine between the light and tangent vectors. Thus **if the tangent of the hair is pointing straight at the light, the hair is dark. This is readily observed in real hair.**"

**(b) 高光 = 圆锥 + Phong 衰减（ad hoc）**

物理直觉：圆柱面的法线绕切线一圈都有，所以**镜面反射光形成一个圆锥**（顶角 = 入射角），高光强度应**与视向量的方位角无关**。

$$\tilde{I}_{specular} = k_s\cos^p(\theta-\theta') = k_s\big[(t\cdot l)(t\cdot e) + \sin(t,l)\sin(t,e)\big]^p$$

其中 $\theta,\theta'$ 为光/视相对切线的倾角。

**作者自评**："我们是 ad hoc 的，但图像质量对此不敏感"（"the exact form of the details of the lighting model not to be particularly critical to the quality of the images"）——**这句话预判了后来二十年的工程现实：实时毛发一直用近似，直到 2003 年才有物理模型。**（对照：Marschner 2003 用 Bessel 函数与 2D 降维给出物理版本。）

### 3. 工程细节（对今天仍有直接参考价值的三条）

| 细节 | 内容 | 为什么重要 |
|---|---|---|
| **双层毛皮** | undercoat（密集短毛）+ overcoat（稀疏长毛） | 原文："this is an important feature for **avoiding a brushlike appearance**"——**"避免刷子感"的原始出处** |
| **环面拓扑分布** | 泊松盘 + torus 拓扑 | 单 texel 无缝平铺整张皮——**可平铺噪声/纹理的 1989 版本** |
| **texel 规格与成本** | 40×40×10 数组；1280×1024 图像；12×IBM 3090 + 4×3081，约 2 小时 | 1989 年的"预算现实"：离实时还差得远——所以它注定先成为"离线参考"，公式则靠简化版流入实时 |

## Key Contribution

1. **texel 概念**：把"亚像素细节"从几何问题重构为**密度场 + 帧场 + BRDF 场**的采样问题；
2. **毛发解析着色模型**（后世称 Kajiya-Kay model）：$\sin(t,l)$ 漫反射 + 圆锥 Phong 高光——**极其便宜，实时可用**；
3. **证明了"渲染成本可以与几何复杂度解耦"**：一个 texel 的渲染时间固定，不论它内部"装"了多少根毛。

## Why It Works

1. **按尺度选表示**：细到几何画不动时，改画纹理/密度——"painter's illusion"的工程化；
2. **求和替代积分**：把连续性假设（体积）换成离散性假设（一堆微表面），绕开了"薄表面在体积里不可见"的数学陷阱；
3. **对称性降维**：圆柱对称 → 光照只依赖切线——与 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 的"倾角守恒 → 3D 降 2D"是**同一种手法**（先找守恒量，再降维）。

## Limitations（原文明说的 + 后世暴露的）

1. **"从几何自动生成 texel"未解决**——原文自述："how to turn geometry into texture has not yet been solved… we speculate that geometric measure theory may provide the key"（这个坑后来由其他路线填上）；
2. **只做了短毛皮（fur），没做长发/卷发**——原文自述；
3. **ad hoc 高光**：不保能量、无透射（TT）、无次级高光（TRT）——**这三个硬伤正是 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 的立项理由**；
4. **依赖各向同性密度近似**（密度本应依赖视向）。

## Game Development Relevance

**5/5 —— 对你是"三个维度的双重历史锚点"（着色 + 资产表示 + 预算哲学）。**

### 1. 它是 [[Hair Rendering]] 的"实时侧那一半"（本库最后一块）

库里此前只有 Marschner 2003（物理侧）。装上 Kajiya-Kay 后，毛发谱系两端齐全：

| | Kajiya-Kay 1989（经验侧） | Marschner 2003（物理侧） |
|---|---|---|
| 模型 | 圆柱，ad hoc | 半透明圆柱，R/TT/TRT |
| 成本 | 极低，**实时默认 30 年** | 高，先影视后高端档 |
| 高光 | 单圆锥 + Phong | 双高光（鳞片倾角）+ caustic |
| 透射 | 无（逆光死板） | 有（TT 逆光亮边） |
| 今天在哪 | **低配档 / 大量存量 shader** | 高端档 / UE Groom 系 |

### 2. "texel ↔ 几何"切换 = 分档阶梯的 1989 版本

原文原话：**"We should switch from the texel representation to actual geometry when viewing the model at this resolution."**（近看时切回真实几何。）

对照你库里 2026 年的同一条原则：
- [[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards]]：发片 → 发丝**自动升档**（"分档阶梯首次可上可下"）；
- 本文：texel → 几何**手动切换**。

**两条相隔 37 年的话说的是同一件事：细节表示要跟着观察尺度走。** 你的 S/A/B/C × 五档矩阵里"毛发/草/远景植被"的档位逻辑，源头就在这里。

### 3. 对五维预算体系的意义（第三条历史锚点）

- **"渲染时间与几何复杂度无关"**——这就是"发片档用贴图表示毛发"能进预算的必要前提。你给"贴图尺寸"设上限（2048/1024/512/256）时，本质是在给这类"用纹理换几何"的资产**标定信息量上限**；
- 与 [[Reeves — Particle Systems (1983)]] 合看：**你的五个预算维度，两个源头（1978 阴影、1983 粒子）已在库**，Kajiya-Kay 1989 补的是"**当细节小到不该用几何时，预算从几何维度迁移到纹理维度**"这条迁移律；
- 一个尖锐的当代对照（今日产业信号）：**《巫师 3》重制版（9-29）用 LSS 路径追踪毛发增强 HairWorks**——毛发从"texel 密度场"（1989）→"物理 lobe"（2003）→"**GPU 光追的原生曲线基元**"（2026）。三代表示，同一个目的。

## Unreal Engine Relevance

- **UE 的 Hair 资产两种表示**（发片 mesh / Groom 发丝）**分别对应本文的"texel 侧"与"几何侧"**；
- 早期大量 UE 毛发 / 卡通毛发材质仍在用 Kajiya-Kay 系思路（切线各向异性高光 + 沿发丝的高光带）；**具体引擎默认模型的归属以官方文档/源码为准，本笔记不做断言**；
- 与 [[Niagara]]：Groom 物理走 Niagara——那是"几何侧"的代价，恰好也是"什么时候该切回几何"的现代判据之一。

## Technology Evolution

```text
Csuri 1979 / Weil 1986 —— 暴力几何（几千多边形/圆柱）→ 走样
        ↓
Reeves 1983 —— 粒子系统（"没有几何地渲染"）★ 本库已入
        ↓
Blinn 1982 / Kajiya-Von Herzen 1984 —— 体积密度 / 非均匀介质光追
        ↓
★ Kajiya-Kay 1989 —— texel：密度+帧+BRDF 场；毛发 sin(t,l) + 圆锥高光（本文）
        ↓
1990s-2000s —— "Kajiya-Kay model" 成为实时毛发默认（GPU Gems 等）；同时被指出不保能量、无透射
        ↓
★ Marschner 2003 —— R/TT/TRT 物理模型（2003 年提出的"替代者"）★ 本库已入
        ↓
2004-2015 —— 实时化近似（Scheuermann 等）、TressFX/HairWorks、UE Groom
        ↓
2026 —— 三条线同时推进：
        · 资产侧：HairCS 发片→发丝自动升档
        · 画面侧：DLSS 5 把 hair 列为神经增强对象
        · 光追侧：《巫师 3》重制版 LSS 路径追踪毛发（HairWorks 增强）
```

## Relationships

### Based On

- [[Reeves — Particle Systems (1983)]] —— 思想前驱（"没有几何的渲染"）；本文是其 ray tracing 对偶
- Blinn 1982《Light Reflection Functions for Simulation of Clouds and Dusty Surfaces》—— volume density 母体
- Kajiya & Von Herzen 1984《Ray Tracing Volume Densities》—— 非均匀介质光追技术底座

### Replaced By / Contrasts

- [[Marschner — Light Scattering from Human Hair Fibers (2003)]] —— **物理侧替代者**：本文不保能量、无透射、无双高光；Marschner 用半透明椭圆柱 + R/TT/TRT 补齐
- [[Microfacet Theory]] —— **另一个方向的边界**：毛发既不走微面 D·G·F（细长散射体 + 透射），本文选择的是"圆柱抛物线高光"这条 ad hoc 路线

### Related

- [[Hair Rendering]] —— 本笔记补上该概念的**实时侧源头**（其 Next Step #1 的直接兑现）
- [[Participating Media]] —— texel 是 volume density 的"表面化"推广：从"球状粒子"到"微表面"（求和取代积分）
- [[Particle Systems]]（概念）—— 同一个"按尺度选表示"的思想在两个渲染管线里的落地
- [[Real-Time VFX Performance Budgeting]] —— "渲染时间与几何复杂度解耦"是"用表示换预算"的最早表述
- [[Scalability and Quality Tiers]] —— "texel ↔ 几何"切换原则

## Personal Knowledge State

- **user_level: Normal**。判断依据：你的日常工作（发片、预算、分档）与"texel 密度场"的直觉一致（Easy 侧）；新的是**两条着色公式本身**与"**为什么体积密度不能直接画表面**"的论证——这两块是纯新增；
- 与 PBR 线的关系：**这是你"D·G·F 失效边界"研究的第三个锚点**——Marschner 给了"物理上怎么失效"，Kajiya-Kay 给了"**失效后工程上用的是什么**"（答案是：一个 ad hoc 圆锥高光，用了 30 年）。

## Mastery 自测（4 条，与 Marschner 的 5 条合并即毛发着色收口）

1. **texel 的三要素是什么？** 为什么密度必须是"投影**面积**密度"而不是体积密度？（答：无限薄表面体积为 0 但完全不透明、100% 反射——体积叙事下它是隐形的。）
2. **毛发的 Kajiya-Kay 漫反射为什么是 $\sin(t,l)$？** 什么时候毛发最暗？（答：对可见半圆柱积分 Lambert；切线直指光源时最暗。）
3. **高光为什么是"圆锥"而不是"点"？**（答：法线绕切线一圈都有，反射光形成顶角=入射角的锥；高光强度与视向量方位角无关。）
4. **"渲染时间与几何复杂度无关"这句话对你的预算体系意味着什么？** 举一个你正在用的例子。（答：发片/贴图档的成本由屏占比×密度×像素决定，不由"实际毛量"决定——这是"用表示换预算"的前提。）

## Learning Value

1. **补齐 [[Hair Rendering]] 的 Next Step #1**（实时侧源头），毛发概念下一步只剩 **Scheuermann 2004 / dual scattering** 这类"实时化工程文"；
2. **一个可迁移判据**：**"这个量该用积分还是求和？"取决于对象是连续的还是离散的**——texel 的全部数学就在这一步切换；
3. **一个跨 37 年的原则**：**表示跟着观察尺度走**（texel↔几何），其当代形态就是你的发片↔发丝分档。

## Visualization

![[Texel 与毛发_Kajiya-Kay 1989 图解.html]]

含：texel 三要素、体积密度"看不见的表面"反证图示、两条毛发公式的几何推导（$t/l/e$ 与圆锥高光）、texel↔几何切换与分档阶梯的对照。

## Notes

- **原文已下载并逐页核对**（Caltech 系课程镜像，10 页扫描件 + OCR 文本层；本笔记所有公式、数字、引语均出自原文逐条确认）：texel 定义（p.272-273）、式 3/4 的"积分→求和"、5 步渲染算法（p.274）、$\sin(t,l)$ 推导（式 13→14）、圆锥高光（式 15→16）、40×40×10 texel / 双层毛皮 / torus 泊松盘（p.274）、渲染成本 12×3090+4×3081 ≈ 2 小时（p.276）、"independent of the geometric complexity"（p.271）、"switch back to actual geometry"（p.276）、"geometric measure theory"推测（p.276）；
- 引用信息对照：SIGGRAPH '89 Proceedings, pp. 271–280；ACM DL 页面同时可见 10.1145/74333.74361 与 10.1145/74334.74361 两个 DOI 记录（本笔记取前者）；
- 入库时机：2026-09-25。触发来源有两处——[[Hair Rendering]] 的 Next Step #1（"Kajiya-Kay 1989 优先"）与 [[Marschner — Light Scattering from Human Hair Fibers (2003)]] 的 Replaces 条目（"Kajiya-Kay 1989 尚未入库，是本概念的下一个经典候选"）。**两处挂账同日结清**；
- 产业巧合（同日记录）：NVIDIA 发布《巫师 3》重制版（9-29）细节，**LSS 路径追踪毛发**（HairWorks 增强）——1989 的 texel → 2003 的物理 lobe → 2026 的光追曲线基元，三代表示在同一天的笔记里会合。
