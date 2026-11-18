---
type: paper
title: "One Code per Material: Neural Representation and Generation of Measured BTFs"
authors: ["Fazilet Gokbudak", "Canberk Baykal", "Chenliang Zhou", "Rafal Mantiuk", "Cengiz Oztireli"]
year: 2026
published: "2026-11-16"
venue: "arXiv preprint（cs.GR，11 页，University of Cambridge）"
url: "https://arxiv.org/abs/2611.20948"
code: ""
project_page: ""
category: ["Rendering", "Material Representation", "BTF", "Neural Texture", "Generative Material"]
importance: A
historical_importance: 3
game_relevance: 4
production_readiness: "Research（渲染前需一次性解码成 light–view 表，3.7 min/A100）"
user_level: Normal
status: unread
---

# One Code per Material: Neural Representation and Generation of Measured BTFs

## TL;DR

**它把"一个材质"从一个*图像*变成了一个*点*。**

| | 一个材质是什么 | 值多少 | 能不能采样新材质 |
|---|---|---|---|
| UBO2014 测量 BTF | 400² texel × 151² 光视对 × 3 | **1.1 × 10¹⁰** 个数 | ❌ 只能重新测 |
| Rainer 2020（per-texel 隐特征） | 每 texel 32 维隐向量 | 5.44 M 个数 | ❌ 那还是一张图 |
| Xia 2026（多分辨率神经纹理） | 每 texel 特征 + 神经偏移 | 3.32 M 个数 | ❌ |
| **本文（compact）** | **一个 256 维码 + 所有材质共享的解码器** | **256 个数**（解码器摊销后 ≈385 k） | ✅ 在码空间上训一个 flow |

代价写在同一张表里：**LPIPS 0.183，比 Rainer 的 0.132 差**——全局码装不下逐 texel 的细节。论文自己说得很直白：*"trades perceptual sharpness for far smaller per-material storage."*

**一句可带走的**：BTF 那 10¹⁰ 个数不是"压缩问题"，是**"这个函数有几个自变量"**的结果。要让它变小，不能靠"接受更多误差"，只能靠**换掉"一个材质是什么"这个定义**。

## Problem

实测 BTF 抓到的是真实材料的全部外观——**视差、自阴影、texel 之间的互反射**，这三样解析 SVBRDF 一个都表达不了。代价是尺寸：单个 UBO2014 材质 400×400 texel、151×151 光视对，约 **10¹⁰** 个数；而且**每来一个新材质就要重新测一次**（公开的测量 BTF 全世界只有约一百个）。

两条已有的路都堵在一处：

| 路线 | 做到什么 | 没做到什么 |
|---|---|---|
| 神经 BTF 表示（latent maps / feature planes / neural offsets） | 把 10¹⁰ 压到 1.6–5.4 M | **每个材质仍然是一张"特征图"**——是图像，不是空间里的一个点 |
| 生成式材质模型（MaterialGAN / PhotoMat / 各种 diffusion） | 能从照片或合成库生成材质 | **没有一个从"实测 BTF"上学的**——它们学的是合成 SVBRDF 或单张闪光照片 |

**这两条路的缺口正好互补，而缺口的名字叫"共享空间"。** 只要一个材质还是"每 texel 一堆特征"，你就没法问"这两个材质之间的中点是什么"，更没法从分布里采样。

## Historical Context

```
1999  Dana et al.        BTF 被定义：一张按光照/视角参数化的纹理图（6 维函数）
                          └─ 61 个样本 × 205 张，14,000+ 张图，10×10 cm 平面样本
  ↓
2004-2014  线性/稀疏分解   把 6 维函数拆成基 + 系数（PCA、张量分解）
2014  UBO2014            84 个材质 / 7 类 / 151×151 光视对 / rank-101 分解 —— 至今的参照集
  ↓
2019  Rainer            逐 texel 角向 profile 的自编码器（第一次"神经"）
2020  Rainer unified    编码器解码器跨材质共享，但**每 texel 仍存 32 维**
2021  NeuMIP            神经纹理金字塔 + 学习到的视差偏移
2023  Neural Biplanes   通用解码器 + 每材质特征平面
2026  Xia et al.        多分辨率神经纹理 + 每材质神经偏移，并可用扩散扩到更大范围
  ↓
2026 ★ 本文             每材质 **一个 256 维码**，解码器全局共享 ⇒ 材质成为空间里的点
                        └─ 于是可以采样：码空间上训一个 class-conditional rectified flow
```

**这条线的每一次进展都在动同一个旋钮：把"材质特有的东西"从"通用的东西"里剥出来多少。**
- Rainer 2020 剥出了**解码器**（共享），留下**逐 texel 隐向量**（特有）。
- 本文把**逐 texel 的部分也剥掉了**，只留一个 256 维码——于是"通用"那一半涨到 32.3 M 参数，"特有"那一半缩到 256 个数。

这不是新招（auto-decoder / functa 那一族早就在做"一个数据点 = 一个码"），**新的是把它搬到"空间变化 + 角向变化"的 6 维函数上**——而这一搬，解码器必须同时表示**空间布局**，这是此前所有 BRDF 侧的码空间工作不必面对的。

## Previous Work

四族，各自的天花板：

1. **经典 BTF 压缩**：线性/稀疏分解（SparseBTF 2023）。压缩率高，但重建靠线性基，镜面与阴影细节会钝。
2. **神经 BTF**：latent maps（Rainer 2019/2020）、neural offsets（NeuMIP 2021 / Xia 2026）、feature planes（Neural Biplanes 2023）、guidance-conditioned（NeuBTF 2023）。**共同点：每 texel 存特征 ⇒ 每材质仍是百万级。**
3. **函数空间里的码**：Sztrajman 2021（神经 BRDF）、DeepBRDF 2020（可编辑隐空间）、hypernetwork 系（Gokbudak 2024 / Li 2024）、physically-based neural BRDF 2026。**共同点：对象是"空间均匀"的 BRDF——没有空间维度。**
4. **生成式材质**：MaterialGAN / TileGen（合成 SVBRDF）、MatFuse / ControlMat / StableMaterials（diffusion on MatSynth）、PhotoMat（12,000 张真实闪光照片）、Generative Neural Materials 2025。**共同点：学的是合成数据或单张照片，学不到视差/自阴影/互反射。**

## Core Idea

两个工作点，一条主线：

```
① per-material 坐标网络        —— 精度上限（36.44 dB）
       ↓ 把"每材质一个网络"换成"每材质一个码 + 共享解码器"
② compact 模型（256 维码）     —— 32.56 dB，但材质成了空间里的一个点
       ↓ 码空间有分布了，就可以采样
③ class-conditional flow      —— 生成的新材质仍是完整 BTF（可重打光）
```

**关键的一步是 ②，而它成立的理由不是"压缩得更好"，是"压缩的对象换了"。**
per-texel 隐特征压的是"这张图有多少冗余"；256 维码压的是"这个材质在这个共享空间里的坐标"。前者压到头也还是一张图（图就有 texel 数），后者压到头是**一个数**（材质的个数）。

## Technical Approach

### 表示

$$\hat B_m(x,\omega_i,\omega_o)=\exp\!\big(g_\theta([\gamma(x),h,d,z_m])\big)-1$$

- $x\in[-1,1]^2$ 是 texel 位置；方向对用 **Rusinkiewicz 的 half/difference 向量** $(h,d)\in\mathbb{R}^6$，**保留方位角**——因为 BTF 的 texel 是各向异性的（这一点与多数 BRDF 参数化不同）。
- $\gamma(x)$ 是随机 Fourier 特征：per-material 用 $F\in\mathbb{R}^{128\times2}\sim\mathcal N(0,16^2)$，compact 用 512 个频率 $\sim\mathcal N(0,32^2)$。
- 输出走 $\exp(\cdot)-1$（沿用神经 BRDF 的做法），把预测限制在实测反射率的动态范围内。
- 损失是**对数空间的 L1**（余弦加权）：$\mathcal L=\frac1n\sum|\log(1+\cos\theta_i\hat B)-\log(1+\cos\theta_i B)|$。**取对数是为了让镜面高光和暗部阴影区在损失里等权**——否则高光会主导梯度。

| | per-material | compact |
|---|---|---|
| 隐层 | 4 × 1024 | 4 × 3072 |
| 参数量 | 3.42 M（每材质一份） | 32.3 M（**全局共享**） |
| 每材质存多少 | 3.42 M | **256**（+ 摊销的解码器 ≈385 k） |

### 编码

集合编码器（DeepSets 式，置换不变）：从 $k$ 个实测样本 $\{(x_j,h_j,d_j,b_j)\}$ 出发，逐样本过 MLP 再平均，得到 $z_m\in\mathbb{R}^{256}$。训练时每步每材质取 $k=16{,}384$ 个随机上下文，测试时用全部样本。

**加 $\lambda\|z_m\|_2^2$（$\lambda=10^{-2}$）把码压小、压到同一量级**——这一项不是为了正则化重建，是为了**让码空间紧凑到能被一个 flow 建模**。这是"表示"与"生成"两个目标在同一个损失里握手的地方。

### 色系（colourways）——不加结构的增广

UBO2014 每类只有 12 个材质，学不出分布。做法：同类的 A、B 两个材质，把 B 的颜色搬到 A 上——**一个 3×3 矩阵 $M_{AB}$ 作用到 A 的每一个 RGB 反射值上**（每个 texel、每个光视对），负值截零、大于 1 保留。

$M_{AB}$ 由线性色彩迁移（Pitié & Kokaram 2007）拟合：把 A 正面图的颜色均值与协方差映到 B 的。

**这一步的性质值得单独记**：结构、起伏、阴影全部来自 A 的实测值，**每一个训练值仍然是一个实测值（在一个 3×3 线性变换下）**。它对应真实生产——同一种织法/纹理/印花，换染料。84 → 1,141 个训练材质（flow 实际用 886，剔掉了纹理太淡的 255 个色系与色相旋转）。

| | 只用 84 个实测 | 加色系 |
|---|---|---|
| KID ↓ | 656 | **466** |
| 覆盖率 COV ↑ | 10.8% | **22.1%** |

**不加色系时，flow 会直接复制几个训练材质**（原文图 9）。这不是模型不够大，是 12 个样本/类撑不起一个分布——**增广在这里买的不是精度，是"别抄"**。

### 生成

类条件 rectified flow：码 $z_1$、类别 $c\in\{1..7\}$、噪声 $z_0$，速度网络 $v_\phi$ 训 $|v_\phi((1-t)z_0+tz_1,t,c)-(z_1-z_0)|^2$，以 0.1 概率丢类别做 classifier-free guidance。$v_\phi$ 是宽 512、4 个残差块的 MLP（4.1 M），adaLN 调制。推理 100 步 Euler，guidance $w=2$。

## Key Contribution

1. **per-material 神经 BTF**：同等规模（3.42 M vs 3.32 M）下比 Xia 2026 高 **+3.2 dB**（36.44 vs 33.24），九个材质上 SSIM / LPIPS / $\Delta E$ 全胜。**比逐 texel 各向异性 GGX 高 14 dB**（22.10）——那 14 dB 就是"视差 + 自阴影 + 互反射"的价格。
2. **compact 模型**：256 个数/材质，PSNR **32.56**，高于 Rainer 2020 $d{=}32$ 的 31.37（5.44 M 个数）+1.2 dB，色彩误差也更低；**每材质存储少 14×**（摊销后 ≈385 k vs 5.44 M）。
3. **可采样的码空间**：KID **466**（对比 Henzler 698 / PhotoMat 863 / 同码空间的高斯 1222），且样本**留在自己的类里**（图 7 的 t-SNE：高斯与 PCA 采样会落到类与类之间、混出别的类的结构）。

## Why It Works

三件事各自成立：

**① 为什么共享解码器不会"串味"。**
32.3 M 的解码器学的是**"介观结构如何与光作用"这个算子**，256 维码只带"是哪一个材质"。这与 split-sum 是同一种分解：一部分只依赖材质身份，一部分只依赖几何/光照。**只要码的维度足够低而解码器足够宽，身份与算子就能分开**——反过来（窄解码器 + 宽码）就是 Rainer 那条路，身份被摊进了一张图。

**② 为什么 +3.2 dB 不是调参调出来的。**
per-material 网络与 Xia 2026 规模相当，差别在**输入**：本文喂 $(x,h,d)$ 给一个 Fourier MLP，Xia 喂的是多分辨率特征网格 + 神经偏移。前者在空间上是**连续**的（Fourier 特征没有网格分辨率），后者受特征图分辨率限制。BTF 的视差与阴影本质是**亚 texel 的位移**——连续表示天然占优。

**③ 为什么码空间能采样。**
因为 $z$ 是被 $\lambda\|z\|^2$ **显式压到同一量级**的，而且解码器对所有材质是同一个。于是"材质"这件事在这个空间里是**连续**的：两个材质的中间点解码出来仍是一个合法的 BTF（不是两张图的混合）。

## Limitations

1. **LPIPS 输给 per-texel**（0.183 vs Rainer $d{=}32$ 的 0.132）。全局码装不下逐 texel 的尖锐细节，**这是"点 vs 图"的必然，不是训练不足**。作者把它写成"用感知锐度换每材质存储"——诚实，但也意味着**这条路的终点是"材质库/传输"，不是"最终画质"**。
2. **"重建"不是"泛化"**：*"the evaluated materials are part of every method's training set"*（沿用 Rainer 2020 的协议）。表 1 衡量的是拟合能力，不是对未见材质的推断能力。
3. **生成的比较是在图像空间做的**：所有生成器都被渲染成"闪光照片"再比 KID/MMD/COV。**这恰好丢掉了 BTF 唯一的意义——角向行为。** 一个 SVBRDF 生成器可以在这张榜上很好看，却完全不会视差。反过来，本文的样本"能重打光"这件事，在这张榜上是**看不见的**（只能靠图 8 定性看）。
4. **结构新意有限**：每类 12 个实测样本，生成的结构基本是"两三个训练材质的混合"。作者自陈 *"structural novelty is limited to a few blends of two training materials"*。
5. **运行时并不便宜**：渲染一个采样出来的材质要**先一次性解码成 light–view 表，3.7 min / A100**（之后与查表的实测 BTF 等价）。**所以省下的是存储与传输，不是运行时。**
6. **覆盖范围 = 测量过的那块布**：*"Our decoder covers the measured patch."* BTF 不是可平铺材质，不能像 texture 那样重复铺满一面墙（Xia 2026 用扩散扩范围，本文列为 future work）。
7. `COV` 22.1 vs Henzler 23.0 —— 多样性只是"接近"，没有超过。

## Game Development Relevance

**直接抄不了，但有三件事值得抄。**

**① "材质 = 空间里的一个点"这件事，对*工具链*比对*运行时*更有价值。**
游戏的材质系统是"参数化 + 贴图"，不是"查表"。但一个项目里几百个材质的**组织方式**可以是码空间：同一族材质（比如 17 种皮革）落在一簇，簇内的插值就是"这个皮革再软一点/再旧一点"。这比在 metallic/roughness 两个旋钮上手调要**多一个可编辑维度**，而且不会破坏物理合法性。

**② 那 14 dB 是"游戏放弃了什么"的定价。**
逐 texel GGX 22.10 dB vs 实测 BTF 36.44 dB。游戏里所有"看起来像塑料"的抱怨，价格就在这 14 dB 里。**但游戏不是不知道，是买不起**——10¹⁰ 个数。所以真正的进步不是"更准的 BRDF"，是"更便宜地拿到那 14 dB 里的一部分"：视差贴图（拿视差那一项）、detail normal（拿自阴影的一部分）、预计算 AO（拿互反射的平均）。**每一项都是这 14 dB 的一个切片，而切片之间不能相加。**

**③ 这篇的真正对手不是 SVBRDF，是"扫描资产工作流"。**
现实中的 3A 材质来自扫描 → 去光 → 参数化 → 手修。BTF 那条路（保留全部角向信息）一直因为 10¹⁰ 被排除。**256 维码把"传输/存储"这一段的成本打到 KB 级**，这意味着：扫描现场可以把 BTF 原样带走（而不是当场拟合成 SVBRDF 丢掉视差），回到工作站再解码。**它改变的是"哪一步做决定"，不是"哪一步更快"。**

## Unreal Engine Relevance

- **不能直接进材质图**：UE 的材质是 shader 图 + 纹理采样，没有"按 (x, ωi, ωo) 查一个共享解码器"的节点。要试只能走 Custom node / Compute shader，且解码器 32.3 M 参数（约 130 MB fp32 / 65 MB fp16）**放不进常量缓冲，只能放 Structured Buffer 或 UAV**。
- **可借鉴的是"解码一次，之后当表用"这条**：3.7 min 的离线解码 → 一张 light–view 表 → 运行时等价于一个巨大的 texture array。这条与 UE 的 **Virtual Texture / Texture Array + 稀疏驻留**是对得上的，只是表的角向维度（151×151）远超任何现有采样器能寻址的范围。
- **最现实的落点是 LOD/变体生成**：把"这个材质在斜视角下该长什么样"交给一个训练好的解码器，离线烘出**视角相关的 detail 层**（类似 UE 的 Parallax Occlusion Mapping 但由实测数据驱动），而不是运行时跑网络。
- **与 Nanite / 虚拟几何无关**——这条线解决的是"表面着色"，不是"几何提交"。

## Technology Evolution

```
"材质该被记成什么" 的六次改写：

1970  Nicodemus        BRDF：一个 4 维函数（空间均匀）
1999  Dana             BTF：加 2 维空间 ⇒ 6 维，实测而非建模        ← 本文的起点
2004+ 张量分解          6 维 → 基 + 系数（压，但仍是"一个材质的表"）
2019  Rainer          神经解码器：表 → 隐特征图（每 texel 一份）
2020  Rainer unified  解码器共享，隐特征仍逐 texel
2026 ★ 本文            隐特征也共享掉 ⇒ 每材质 **256 个数**，材质成为空间里的点
                      └─ 于是"生成新材质"变成"在这个空间里采样"
```

**这条线的方向一直是"把'材质特有'的部分缩小、把'通用'的部分做大"。** 而它的终点有一个天然上限：**通用部分不能无限大**（32.3 M 已经是每材质 256 个数的 12 万倍）。**当通用部分涨到"所有材质共享一个大模型"时，这个框架就变成了"一个世界模型 + 每个材质一句 prompt"**——那正是 2026 年另一条线在做的事。两条线会在那里汇合。

## Relationships

### Based On

- [[Dana 1999 — Reflectance and Texture of Real-World Surfaces|Dana 1999]] —— BTF 的定义与第一个数据库。本文的全部训练数据（UBO2014）是它的直系后代。
- [[Microgeometry]] —— BTF 抓的正是"编译不进通道"的那部分：视差、自遮挡、互反射。
- functa / auto-decoder 那一族（库外）—— "一个数据点 = 一个码"的范式。本文把它从"形状/BRDF"搬到"空间变化 + 角向变化"的 6 维函数。

### Extends

- **Rainer 2020（unified neural encoding of BTFs）** —— 把它的"共享解码器 + 逐 texel 隐向量"改成"共享解码器 + 一个全局码"。
- **Xia 2026（EGSR）** —— 同一批数据、同一套协议，用连续 Fourier 表示换掉多分辨率特征网格，+3.2 dB。

### Related

- [[Texture Filtering]] —— 同一个函数，两种失败：BTF 说"这张图有 6 个自变量"，过滤说"这个像素盖住了多少"。**两者都指向同一件事：texel 不是最小单位。**
- [[Decoupling Geometry and Appearance]] —— 本文把"外观"整个搬进了一个码，连 texel 网格都不要了；那一页讲的是"属性住哪"，这里是"属性根本不落地"。
- [[Budget Units]] —— 一次教科书式的"换单位"：从"每材质的 texel 数"换成"材质的个数"。**但换完要把转换费报出来**（3.7 min 解码 / 32.3 M 解码器）。

### Followed By（可预期）

- 扩范围（tiling / 更大 extent）：Xia 的扩散扩范围与本文的码空间是互补的，接起来就是"可采样的、可平铺的 BTF"。
- 条件化（文本/图像而非类别）：作者列为下一步。
- 更大的实测集（RoboCloth 的 500 个布料）：本文的瓶颈是 84 个材质，数据一多，码空间的分布会立刻变厚。

## Personal Knowledge State

- **Normal**。BTF 这个概念本身在 [[Microgeometry]] 里已经反复出现（"编译不进通道的那部分"），但**"一个材质可以是一个点"这个表示论层面的转向是新的**。
- 已有的 [[Texture Filtering]] / [[Budget Units]] 两条判据可以直接挂上去，不需要新建框架。

## Learning Value

1. **"10¹⁰ 不是压缩问题"** —— 这句话可以搬去任何"表太大"的场合：先问"这个表有几个自变量"，再问"能不能压"。**自变量数不降，压缩只能拿精度换。**
2. **换单位的转换费必须一起报** —— 256 个数听起来是 4 万倍压缩，但配上 32.3 M 解码器和 3.7 min 解码，**摊销后是 14×，端到端是"离线一次 + 运行时查表"**。这与 [[Budget Units]] 的 ⑦ 是同一族陷阱。
3. **"重建"与"泛化"必须分开报** —— 表 1 的训练集包含被测材质，这是该子领域的惯例，但**惯例不等于可以不说**。

## Visualization

![[Research/Files/Pictures/Gokbudak 2026 — 图1 测量BTF重建与生成的整场景.png]]
**读图**：作者把重建出来的 BTF 与生成出来的 BTF 一起铺进同一个客厅——沙发织物、三只靠垫（红／金／蓝格纹）、墙纸、地毯、花盆与三颗球（奶白／深石／陶土橙），**没有一块是手调的 SVBRDF**。看那五个圆圈放大镜：沙发扶手处在斜光下出现**织物自身的绒毛阴影**，蓝靠垫的凹凸随视角错位（视差），石球表面的颗粒有**互反射造成的暗部色偏**。**这三样东西正是 BRDF 的三条假设排除掉的**——所以这张图的卖点不是"画质好"，是"这些效果居然来自一张表"。（图 1，Gokbudak et al. 2026）

![[Research/Files/Pictures/Gokbudak 2026 — 图2 紧凑模型与生成流程总览.png]]
**读图**：上半支是表示——84 个实测 BTF 加色系增广 → 集合编码器 → 每个材质一个码 $z_m$ → 所有材质共享的解码器 $g_\theta$，输入 $(x,\omega_i,\omega_o)$ 输出 RGB。下半支是生成——噪声 + 类别 → rectified flow → 采样码 $\hat z$ → 同一个解码器 → 完整 BTF。**"reconstructed / rendered on objects / samples / relit" 四个输出框是同一套解码器的四种用法。**（图 2，Gokbudak et al. 2026）

![[Research/Files/Pictures/Gokbudak 2026 — 图4 率失真曲线.png]]
**读图**：横轴是每材质存的浮点数（对数），纵轴 PSNR。**红星有两个位置**——左边那颗是纯码（256 个数 / 32.56 dB），右边那颗是"把解码器按 84 个材质摊销后"的位置（≈385 k / 32.56 dB）。**同一份画质，两个记账口径差三个数量级**，这就是"转换费"在图上长什么样。注意 Rainer 2020 (32) 用 5.44 M 个数只到 31.37 dB——在它右边的每一颗点，都是"更贵但更差"。（图 4，Gokbudak et al. 2026）

![[Research/Files/Pictures/Gokbudak 2026 — 图5 逐材质重建对比.png]]
**读图**：每行一个光视对，每列一种方法，第一列是实测原图。**看地毯（carpet）与皮革（leather）的斜视行**——GGX 那一列（最右）在斜视角下变成一片均匀的糊，因为它逐 texel 独立着色，拿不到邻域的自阴影；越往左，起伏与阴影越接近实测。（图 5，Gokbudak et al. 2026）

![[Research/Files/Pictures/Gokbudak 2026 — 图8 生成材质重打光.png]]
**读图**：五列是五个**生成出来**的材质（不是实测），五行是点光源在相机位 / 左 / 上 / 右 / 下。**关键的观察不是"像"，是"跟着动"**——皮革的高光、地毯的阴影、毛毡的绒面朝向都随光移动，而这是任何 SVBRDF 生成器做不到的（它生成的是一张 albedo+法线+粗糙度，斜光下不会长出新阴影）。（图 8，Gokbudak et al. 2026）

## Notes

- **建笔记依据**：全库 `grep "BTF|Bidirectional Texture Function|Dana"` = **0 命中**（唯一一处出现在 `LightOpt-中文全译.pdf` 里，是参考文献列表）。而 [[Microgeometry]] 与 [[Texture Filtering]] 两页反复在讲"异质贴图 / 编译不进通道的效果"，**它们描述的那个对象一直没有名字、没有源头、没有当代端点**。今天一次补齐：源头 [[Dana 1999 — Reflectance and Texture of Real-World Surfaces|Dana 1999]] + 端点本篇 + 概念页 [[Bidirectional Texture Function]]。
- **本文与 [[2026-11-12-Silaghi — Deformable Texture Maps Compiling Dynamic Microgeometry into Neural Textures|DTM 2026]] 是同一个问题的两种答案**：DTM 把微观几何**编译成通道**（$\alpha,n,h,c$ + 变形 $d$），本文**拒绝编译**、直接存测量结果。前者便宜但只能表达"局部效果"，后者准确但贵 10¹⁰ 倍。**中间那条路（编译一部分、查表一部分）至今没人画出来。**
- **那个 3×3 色系矩阵值得单独记住**：它是"增广但不发明结构"的模板——所有新样本的几何、阴影、视差都是实测的，只有颜色是合成的。**在数据稀缺的领域，这种"只改一个可分离因子"的增广比 GAN 增广安全得多。**
- 图 5 的 GGX 列（22.10 dB）与本文（36.44 dB）之间的 14 dB，**不是"更好的模型"的胜利，是"更多自变量"的胜利**。这一点在库里值得反复引用：它给 [[BRDF]] 页的"BRDF 是一个假设"提供了最贵的那个反例。
