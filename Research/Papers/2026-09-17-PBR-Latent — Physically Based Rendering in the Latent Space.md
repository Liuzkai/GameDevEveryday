---
type: paper
title: "Physically Based Rendering in the Latent Space"
authors: [Vuk Radovanovic, Vishesh Gupta, Adrien Gruson, Binh-Son Hua]
year: 2026
published: 2026-09-17
venue: "Pacific Graphics 2026, Journal Track（Computer Graphics Forum）；DOI 10.1111/cgf.70633"
url: "https://arxiv.org/abs/2609.21054"
code: "https://github.com/trinity-graphics/latent-rendering（仓库已在线）"
project_page: ""
doi: "10.1111/cgf.70633"
aliases: [Latent Rendering, PBR in the Latent Space]
category: [rendering, neural-rendering, generative-rendering, differentiable-rendering, pbr, vae]
importance: A-
historical_importance: 2
game_relevance: 3
production_readiness: Research
user_level: Normal
status: read
---

# Physically Based Rendering in the Latent Space

> 入库于 **2026-09-21（同日补录）**。**arXiv 9-17 投稿，错过了周五公告切点，直到 9-21（周一）listing 才被 announce** —— 首轮运行（09:00 CST）时它尚未放出，属检索窗口的真实边缘遗漏；10:00 复查周一 listing 时发现并补录（详见 [[2026-09-21]] 的「补录」节）。
>
> **一句话**：把**渲染方程改写成能在 VAE 潜空间里求解的形式**，让路径追踪**直接输出 latent 图**（而不是 RGB），再经解码器还原画面。
> **这是"PBR 进入生成模型表示层"的第一篇可运行工作 —— 渲染方程第一次被当成"可搬进潜空间的工具"，而不是"必须输出像素的合约"。**

## TL;DR

**问题**：扩散模型生成的图像"不守物理"（物理合理性由数据驱动，缺乏原则性框架）；而经典图形学有全套物理正确性技术，两者之间没有桥梁。

**观察**：*"there is a bridge between light transport phenomena and the distribution of latent space values produced by such models."* —— 潜空间的值和光传输现象之间存在桥。

**做法**：在 VAE 学出的特征空间里做**物理渲染**（latent rendering）：
1. **改写渲染方程**（三项修改：带符号辐射 / 平响应项 / 遮挡项），让渲染器能输出符合 VAE 分布的值；
2. 配**可微渲染器**优化场景参数（**单张图**训练），使渲染出的 latent 逼近真实 latent；
3. 加一个**轻量神经精化网络**只预测残差。

**结果**：在 latent 空间估计上比"先 RGB 路径追踪再编码"**快 42–84×**；但把 latent 解回 RGB 输出时比直接 RGB 路径追踪**慢 6–35×**（**瓶颈是解码器**）。

**一句话价值**：

> **"物理先行、网络收尾"的又一次实例（本库"分层收敛"模式）——而且这次是反向的：不是让生成模型学会物理，而是把物理渲染器改写成生成模型的"母语"。**

## 来源与核对

- **全文核对**：arXiv HTML 版（`arxiv.org/html/2609.21054v1`）逐节核对 —— 摘要 / §1 Introduction / §4 Latent Space Analysis / §5.1–5.3 方法 / §6.1–6.7 实验与局限 / Table 1 消融。**本笔记引用的全部引文与数字均出自论文正文。**
- **未核对**：图版（Fig. 1–11）的像素级内容、代码仓库实际代码。
- 性质：**pre-peer-reviewed 版本**（论文页声明 *"will be published in final form at doi.org/10.1111/cgf.70633"*，CGF = Computer Graphics Forum，Pacific Graphics 2026 Journal Track）。
- 作者机构：**Trinity College Dublin**（Radovanovic / Gupta / Hua）+ **ÉTS Montréal**（Gruson，Adrien Gruson 为资深渲染研究者）。

## Problem

两条技术线各自成熟但互不相通：

| 线 | 状态 | 缺口 |
|---|---|---|
| 经典 CG（PBR / Monte Carlo 渲染）| 物理正确性有 40 年积累的完整工具箱 | 表达力 / 可控性之外，**生成能力为零** |
| 生成模型（扩散）| 能生成逼真图像、可用语言控制 | **"not grounded in the laws of physics"** —— 物理合理性只来自数据分布 |

**论文的问题陈述**：*"Physical plausibility in visual content generation is mainly driven by data, leaving physical accuracy and correctness largely underconstrained. While computer graphics has numerous techniques targeting physical correctness, generative models lack a principled framework rooted in a physical basis."*

**此前"混合渲染"路线的共同点**（§2.3）：要么让生成模型去**拟合/预测**渲染方程的解（transformer 直接预测渲染方程近似解，受限于小场景），要么让神经渲染器**精化**物理渲染结果。**共同点：物理管线与生成模型在 RGB 层交换数据。**

**本文的不同**：把交换层从 RGB **下移到 latent** —— 渲染器直接产出 latent 特征，物理约束借此"渗入"生成模型的内部表示。

## 与前作的关系（位置）

- **Latent-NeRF**（2022）：让 NeRF 输出**兼容扩散模型的 latent 特征**，做 text-to-3D。**本文与之同属"渲染到潜空间"大类**，但 Latent-NeRF 是"让神经场对齐 latent"（无物理），本文是"让**物理渲染器**对齐 latent"（有物理）。
- **Splatent**（CVPR 2026）：在 VAE 的潜空间里重建 3DGS，用一步扩散提升 latent 质量以改善新视角渲染。**同路线、不同图元**（高斯 vs 物理光传输）。
- **DLSS 5 / 生成式渲染**：**方向相反** —— 生成模型在 RGB 侧**替代**渲染器出图；本文把物理渲染器**塞进**生成模型的表示空间。
- **Inverse Rendering**：逆渲染从图像反解场景参数；本文场景参数是**已知输入**，要去逼近的是**表示（latent）**——但同用可微渲染器做优化循环。

## Core Idea

### 观察（§4）：latent 与 RGB 的"三个不守物理" + "三个守物理"

对 Stable Diffusion 3.5 的 VAE 潜空间（1024×1024 RGB → **128×128×16** latent）分析 Cornell Box，发现：

**三个"不守物理"（决定了方程必须改哪三处）**：

1. **发光是带符号的**：*"In the latent space, emitted light is signed (Fig. 1, A), in contrast to the strictly non-negative values assumed in physically based rendering."*
2. **强几何结构**：latent 强调几何边缘与角点；高频区域的值**可能看起来就是噪声**。
3. **遮挡区的值可正可负、甚至更强**：*"In occluded regions, latent values can exhibit both positive and negative energy, which can be an even higher intensity than surrounding unoccluded regions. This differs from physically based rendering, where occlusion from a light source can only reduce the amount of energy received by a surface."*

**三个"守物理"（决定了这条路可行）**：

1. 结构对应：不同 BSDF/物体在 latent 通道里有与 RGB 对应的**区分度**；
2. 带符号能量会**被反射**、在理想镜面上**近似守恒**；
3. 反射强度随入射角**平滑变化**（类 BSDF 行为）。

> **直觉提炼**：latent 空间像"一组长曝光的语义特征"，**既保留了光传输的骨架（结构/反射/角度依赖），又长出了物理上不可能的东西（负发光、遮挡增亮、平坦响应）**。所以策略不是"硬套物理"，而是 **"物理骨架 + 显式建模偏离项"**。

### 三项修改 → 潜空间渲染方程（§5.1）

| 修改 | 对应观察 | 内容 |
|---|---|---|
| **① 带符号辐射与反射** | 发光可负 | 允许发射器发出带符号辐射 $\tilde{L}_e$，BSDF 求值也允许带符号值 → **带符号光传输方程**（带符号吞吐 $\tilde{T}$）；**纯镜面材质不改**（仍作理想镜面） |
| **② 平响应项** $\tilde{L}_a$ | 部分通道"平"、不反映材质 | 在表面交互处加一个**显式控制"保留材质差异 vs 塌缩为常数"**的项 |
| **③ 遮挡项** $\tilde{L}_c$ | 遮挡区可增亮/变号 | **只在着色区域额外贡献**的项：在相机子路径的末端漫反射顶点上，采样光源顶点并乘二值可见性 $V$ |

**合成 —— 潜空间渲染方程（式 7）**：

$$\tilde{I}_i(\pi)=\tilde{I}_i^e(\pi)+\tilde{I}_i^a(\pi)+\tilde{I}_i^c(\pi)$$

**估计方式**：单条光路（BSDF 采样），前两项在**第一个漫反射面**处估计，第三项继续延伸路径估计；$\tilde{I}^e$ 用 NEE + MIS（显式采样带符号发射器）。**保号 gamma 校正**（式 8）：

$$g(x,\gamma)=\max(x,0)^{\gamma}-\max(-x,0)^{\gamma}$$

> **注意这个结构的哲学**：后两项是**"物理之外的偏离"被显式建模**——不是交给网络去隐式吸收，而是**写成物理方程的同级项**。这是"物理先行"路线的关键设计选择：**把'不像物理的部分'也变成有名字、有参数、可优化的一等公民。**

### 优化与精化（§5.2–5.3）

- **可微渲染器优化**：Mitsuba + **path replay backpropagation**（无重参数化），扩展支持 16 通道带符号渲染；每个路径顶点的 $\pi=\{\tilde{L}_e,\tilde{L}_a,\tilde{L}_c\}$ 是 $\mathbb{R}^{16}$ 向量，同时优化 BSDF 反射率与粗糙度 $\alpha$（支持 diffuse / conductor / dielectric / plastic 及其粗糙变体）。
- **损失直接建在 latent 空间**（Huber，$\delta=1$）：*"we compute the loss directly in the latent space, which empirically improves training convergence, stability, and efficiency. This also helps avoid inefficient gradient backpropagation through the decoder."*
- **神经精化网络** $R_\theta$：输入 = 渲染 latent + AOV（着色法线 + 深度，latent 分辨率），输出**残差 latent**；三层卷积块（3×3 / 5×5 / 3×3），隐藏宽度 $16\times d=256$；损失 = latent 域 MSE + SSIM + 解码 RGB 的 LPIPS。
- **训练数据：单张渲染图**（每场景训练）。

## 关键数据（均出自论文正文）

| 项 | 数值 |
|---|---|
| 表示压缩 | 1024×1024×3 RGB → **128×128×16** latent（**12×** 降维，通道 d=16） |
| 估计 latent 值 | 比"先 RGB 路径追踪再编码" **快 42–84×** |
| 输出 RGB（解码后） | 比直接 RGB 路径追踪 **慢 6–35×** —— *"the single decoder evaluation dominates the runtime"* |
| 训练成本 | 单图训练；场景参数 3000 迭代 + 精化网络 3000 迭代（Adam，lr $10^{-4}$，cosine + restart + early stop）；**Cornell Box 在 RTX 4090 上各 5 分钟** |
| 场景集 | Cornell Box / Lamp / Veach-Bidir / Dining Room / Living Room（5 个，复杂度递增） |
| 消融（Cornell Box · 相机变化）| LPIPS：仅带符号发射器 **0.502** → +平响应 **0.228** → +遮挡项 **0.213** → +精化 **0.075** |
| 消融（Living Room · 物体变化）| 最终 **0.008 / MSE 0.013**（×100） |

**消融的关键结论**（§6.4，原文）：

> *"Using only signed emitters (Eq. 4) leads to poor performance and unstable training. In contrast, the flat-response term (Eq. 5) and occlusion term (Eq. 6) have a significant impact on image quality."*
>
> *"If the rendered latents are noisy (e.g., produced using only signed emitters), the refiner does not yield significant improvement."*

**两条可带走的**：

1. **"物理 + 偏离项"缺一不可**：只做"带符号化"的纯物理版本效果很差且训练不稳定 —— **潜空间的"非物理结构"不是噪声，是必须建模的一等结构**；
2. **"网络不能救烂输入"**：渲染 latent 质量差时，精化网络几乎补不回来（它只预测残差，不做从零生成）。

## Key Contribution

1. **首次把物理渲染的求解目标从 RGB 换成 VAE 潜特征**（latent rendering 范式），并给出完整、可运行的实例（Mitsuba + PRB）；
2. **为潜空间改写渲染方程**：带符号辐射/反射、平响应项、遮挡项 —— 把"latent 不像物理的部分"变成方程内的显式项；
3. **证明可行性边界**：单图训练即可泛化到**场景内**几何/光照/相机变化；并诚实给出成本结论（latent 侧快、RGB 侧受解码器拖累）。

## Why It Works

1. **因为挑了正确的"接口层"**：不要求生成模型懂物理，也不要求渲染器懂扩散 —— 双方在 **VAE 的 latent 特征空间**握手（一个双方都有定义的公共语言）。
2. **因为 12× 的空间压缩让渲染目标变小**：1024²像素 → 128² 特征图，路径追踪的"输出带宽"降了一个量级。
3. **因为分工沿"可学习性"切**：**渲染器负责决定性的物理骨架**（*"most of the work is done by the rendered latents"*），**网络只负责残差**（边界/高频）。消融证明这个顺序不可交换（烂 latent 上网络无能为力）。
4. **因为训练损失建在 latent 域**：避开了"经解码器反传"的低效路径。

## Limitations

- **锐利细节走样**：*"as the latent space is in low spatial resolutions, aliasing tends to dominate when rendering highly detailed objects."*；
- **精化网络逐场景训练，不能跨场景泛化**（*"does not generalize across scenes"*）—— 作者列为最重要的下一步（universal refiner）；
- **RGB 输出性能落后**：解码器是瓶颈（6–35× 慢），画质与耗时都还没达到传统方法水平；
- **论文自称 "a very first step"** —— 未验证大规模场景/材质，未接 SOTA 扩散模型；
- **依赖特定 VAE**（SD 3.5 的 VAE）—— 换模型需重新分析 latent 结构（论文的"三项观察"是经验性的，非通用定理）。

## Game Development Relevance

### 对"运行时"：不构成近期落点

- 它的目标场景是**内容生成 / 场景编辑**（SDS 文本驱动改材质、潜空间内插删物体等），**不是实时渲染**：训练以分钟计、单帧推理以解码器为主导，离每帧 16.7 ms 有两个量级。
- **不要把它读成"又一个实时渲染技术"** —— 本库把它入库的理由是**结构性的**（见下）。

### 🔴 对本库/对你：三个结构性意义

**1. 它是"分层收敛"模式在渲染侧的又一个实例（而且是最"纯"的一个）**

| 实例 | 显式规则侧 | 学习侧 | 交换层 |
|---|---|---|---|
| [[Magpie — Real-Time World Renderer for Interactive Games]]（9-8）| 引擎管规则 | 生成模型只重画白模 | 图像 |
| [[DLSS 5 — Generative Neural Rendering]]（9-18）| G-buffer 提供结构 | 扩散生成最终外观 | 缓冲 + 图像 |
| **本篇**（9-21 补录）| **物理渲染器做承重（决定性的物理骨架）** | **网络只预测残差** | **latent 特征** |

> **"分层收敛"此前在 世界模型 / 渲染管线 / 仿真 三处确认；本篇把"交换层"第一次压到了"表示层"** —— 两层不再在像素上交换，而是在**生成模型的内部语言**里交换。**这是这个模式目前走得最深的一个样本。**

**2. 它改写的是 [[Rendering Equation]]（Kajiya 1986 是它的参考文献 [22]）—— 对你的"PBR 学习线"是新的一站**

你正在收口的 PBR（来源/工程/能量三侧已闭合）在这里显示出**下一站的方向**：**渲染方程作为"可迁移的求解框架"，其输出物（辐射度）可以被换成任意表示。** 具体地，这项研究指出：**要让渲染方程在 latent 空间成立，必须补三项"非物理"修正**（带符号 / 平响应 / 遮挡）—— 这是对"物理约束边界"的一次清晰刻画。

**3. 一个新的成本账本（与你熟悉的分档思维同构）**

| 指标 | 数值 | 同构对照 |
|---|---|---|
| 潜空间内估计 | **42–84× 快** | "在低维表示里干活更快" |
| 解回 RGB | **6–35× 慢** | **解码器成为新的成本单位** —— 类比"每盏灯 ≈ 一遍场景渲染"：**成本不会消失，只会转移** |
| 压缩率 | 12× | 与"屏占比×密度"同类的**表示效率账** |

> **一条对你的直接含义**：**"把计算搬到低维表示"能省的是表示内计算，省不掉"还原到消费端"的成本。** 你的特效分档里同样存在这类账：**粒子模拟可以便宜，但"合成到画面"（OverDraw / 排序 / 混合）的成本不随模拟端降低** —— 两笔账要分开记。

## Unreal Engine Relevance

**诚实说明：本篇没有 UE 侧的近期落点**（Mitsuba 离线研究原型，无引擎路径），**不强行映射**。唯一真实的接口是一个**方向性判断**：

- UE 侧的"神经层"目前都是**在像素侧收尾**（[[Neural Upscaling and Frame Generation]] / MegaLights 等）；
- 本篇提示的另一个方向是**在材质/表示的中间层交换**（例如把光照的部分计算搬到某种学习表示里、再解码回着色输入）—— **目前无任何引擎实现，仅作为雷达项跟踪**；
- 若未来出现"latent 侧编辑 → 导出资产"的生产管线（如 DCC 侧的材质/光照编辑），它的 SDS 应用模式（文本驱动改 BSDF，跳过编码器）可能先于实时侧落地。

## Technology Evolution

```text
1986  Kajiya 渲染方程（求解目标 = RGB 辐射度）        ← 本库经典锚点，本文引用 [22]
        ↓
2018+ 神经渲染（NeRF / 3DGS）：表示可学，但物理仍是"重建的对象"
        ↓
2022  Latent-NeRF：神经场 → 对齐扩散模型 latent（无物理）
        ↓
2024  SDS / DreamFusion 范式：扩散先验反哺 3D（像素/latent 层交换）
        ↓
2026  Splatent（CVPR）：3DGS 在 VAE latent 里重建（图元 ≠ 物理光传输）
        ↓
★ 2026-09-17  PBR in the Latent Space —— 首次把"物理渲染器"改写进 VAE 潜空间：
              渲染方程加三项修正（带符号 / 平响应 / 遮挡），单图训练
        ↓
        （下一步：通用精化网络、latent 空间超分、接 SOTA 扩散模型）
```

## Relationships

### Based On

- **[[Kajiya — The Rendering Equation (1986)]]** —— 被改写的对象（本文参考文献 [22] 即 Kajiya 1986）；
- **Mitsuba 3 / Dr.Jit（Jakob et al. 2022）+ Path Replay Backpropagation（Vicini et al. 2021）** —— 可微渲染与优化的引擎（§6.1 明确 [19] + [48]，且"无重参数化"[64]）；
- **Stable Diffusion 3.5 的 VAE**（Esser et al. 2024）—— 潜空间的定义者（三项观察全部在该 VAE 上做出）；
- **DreamFusion / SDS（Poole et al. 2023）** —— 文本驱动场景编辑的应用侧机制（§6.6）。

### Extends

- **PBR 的可求解域** —— 从"输出 RGB"扩展到"输出任意可微表示"；
- **"渲染到潜空间"路线**（Latent-NeRF / Splatent）—— 从神经表示扩展到**物理光传输**。

### Related

- [[Generative Rendering]] —— **反方向的同题工作**：DLSS 5 让生成模型出最终外观；本篇把物理渲染器搬进生成模型的表示空间。**两者构成"物理 ↔ 生成"的两端**；
- [[Magpie — Real-Time World Renderer for Interactive Games]] —— **"分层收敛"的同模式**（显式规则 + 生成收尾），交换层不同（图像 vs latent）；
- [[Inverse Rendering]] —— 共享可微渲染优化循环；区别：逆渲染解场景参数，本篇解"表示对齐"；
- [[Differentiable Rendering]] —— 本篇的使能技术（可微路径追踪是全部优化的前提）；
- [[Neural Rendering]] —— 属于其"渲染目标换成神经表示"的分支。

### Contrasts

- **传统 RGB 渲染（baseline）** —— 论文的对照实验：同一场景，RGB 路径追踪 vs 潜空间渲染。**结论是"各有快慢一侧"**：latent 侧快 42–84×、RGB 侧慢 6–35×，**说明对比必须声明"在哪个空间里算账"**。

## Personal Knowledge State

- **user_level: Normal**（按本库 9-20 分层模板拆读）：

| 层 | 内容 | 读法 |
|---|---|---|
| **渲染方程改写层** | 三项修正的动机与形式（带符号 / 平响应 / 遮挡）| **Normal 可直接读** —— 你能看懂每项在补什么 |
| **可微渲染/优化层** | Mitsuba + PRB、Huber loss、AOV 精化网络 | 只需知道"它用可微渲染做梯度优化"（[[Differentiable Rendering]] 的内容，Hard 可暂跳过细节） |
| **VAE/扩散机制层** | SD 3.5、latent 分布、SDS | **可全部跳过** —— 只取"latent 是特征图、可解码"这一个事实 |
| **结论层（最值钱）** | ① 分层收敛的表示层实例；② 渲染方程输出物可替换；③ 成本转移账本 | **只读这一段也值得** |

- **为什么不是 Hard**：它不要求你推任何公式 —— 三项修改每个都能用一句话说清"补的是什么"。**它难的部分（可微渲染/扩散）在你需要动手时才需要，而当前没有动手场景。**

## Learning Value

1. **"分层收敛"的最深样本**：交换层从图像压到表示层 —— 为观察该模式是否继续复现提供关键判据；
2. **"渲染方程的守恒量可以换"这条元认知**：帮你把 Kajiya 方程从"渲染的公式"升级为"约束可迁移的框架"；
3. **成本转移账本**：42–84× / 6–35× 的双向数字，是"计算搬家不消灭成本"的干净案例（直接可对 NGR 分档迁移）；
4. **"非物理偏离也必须显式建模"的工程判据**：任何"把物理塞进学习表示"的工作都会遇到同一问题（只保留物理 → 差且不稳）；
5. **消融方法示范**：三项修改逐项叠加 + 精化网络 —— 干净的"每项在补什么"分解实验。

## Mastery Criteria（3 条，纸面自测）

1. 说出 **latent 与 RGB 的三处"不守物理"**，以及论文为每处改写的是方程的哪一项；
2. 说出 **为什么"只把物理搬过去"（仅带符号发射器）会又差又不稳**（提示：偏离项不是噪声、是结构；网络不能救烂输入）；
3. 说出 **"在 latent 里算快 42–84×"与"解回 RGB 慢 6–35×"为什么可以同时成立**，以及它对你分档思维的含义（成本转移）。

> **一句话检验**：能说出 **"物理渲染器提供决定性的骨架，网络只补残差；而要让物理在别的表示里成立，必须先显式建模那个表示'不像物理'的部分"**，即算抓住本篇。

## Visualization

![[潜空间渲染_管线与三项修改图解.html]]

含：两条管线对照（传统"RGB + 编码器" vs 本文"渲染方程直接产出 latent"）→ "三个不守物理"与方程三项修改的对应表（含消融数字）→ 成本账本四卡（12× / 快 42–84× / 慢 6–35× / 5+5 分钟）。

## Notes

- **入库理由（须记录）**：① 它是 **9-17 投稿、9-21 公告的窗口边缘条目**（首轮运行因公告未放出而不可见，非重复、非凑数）；② 其知识图谱价值明确：**"分层收敛"表示层实例 + 渲染方程输出物可替换 + 成本转移账本**，三条均为本库主线所缺的样本。
- **一条检索方法记录（新增，务必沿用）**：**"API `submittedDate` 窗口查询"与"listing 日期分组"各有盲区** —— 本次它两者都躲过了：公告前 API 不可见（尚未 announce），公告后 submittedDate（9-17）又落在窗口（9-18~9-22）之外。**下次运行对"周末/节假日积压窗口"应直接抓 `recent` 页的日期分组逐组核对**，不能只信 API 窗口。
- **2026-09-21 核对边界**：全文 HTML 逐节核对（抽象/方法/实验/消融数字均有出处）；**图版细节与代码仓库内容未核对**。论文为 CGF 录用版预印本。
- **与 [[2026-09-18-GestureFAR — Streaming Co-Speech Gesture Generation with Flow Autoregression]] 的关系**：两者同日（9-21）出现在同一 listing 分组，但 GestureFAR 是 9-18 已覆盖条目（周一仅重公告），本篇是新条目 —— **去重时按"公告分组"而非"同组"处理**。
