---
type: paper
title: "ReMeDi: Single-Image PBR Material Estimation via Reparameterization between Metal and Dielectric Hypotheses"
authors: ["Fan Fei", "Xin Huang", "Yixin Zhu", "Jiawei Lu", "Xin Yang", "Jinglin Liang", "Ruining Tang", "Yifei Feng", "Boxin Shi", "Chunchao Guo"]
year: 2026
published: "2026-11-25"
venue: "arXiv 预印本（cs.CV），27 页 16 图；北大（王选所 / 视频技术国家工程研究中心）+ 腾讯混元3D + 南大"
url: "https://arxiv.org/abs/2611.34284"
code: "未给（项目页有）"
project_page: "https://costrice.github.io/remedi/"
category: ["Inverse Rendering", "PBR", "Material Capture", "Diffusion", "Texture"]
importance: A
historical_importance: 3
game_relevance: 5
production_readiness: "Research（LoRA 微调的 DiT，单卡约 60 s/图；三个模块可并行；无开源权重）"
user_level: Normal
status: unread
aliases: [ReMeDi, intended metalness, 意图金属度, 双假设重参数化]
tags: [inverse-rendering, pbr, material-capture, metallic-roughness, diffusion]
---

# ReMeDi — 一张 base color 不是自描述的，它的含义由另一张图决定

## TL;DR

单图 PBR 材质估计的老做法是**逐通道回归**：网络分别吐出 base color $C$、metalness $M$、roughness $R$，各自算损失。问题是这三张图**不是独立的**——$C$ 的物理含义完全由 $M$ 决定（$M=0$ 时它是漫反射反照率，$M=1$ 时它是镜面反照率）。两个通道各自都"对"，合起来就错。

ReMeDi 把这件事变成**参数化问题**而不是损失问题：把 $C$ 重写成两个完整假设下的变体的混合 $C=\mathrm{lerp}(C_d,C_m;M)$，再用**两个各自只学一个假设的网络**保证两端真的是完整材质。于是 $C$ 的 intended metalness **按构造**等于 $M$。

结果（SYNTEST，365 个 Objaverse-XL 资产）：base color PSNR **21.33**（最强基线 SuperMat 18.47，**+2.86 dB**），重光照 **22.34**（19.60，**+2.74 dB**），metalness RMSE **0.197**（0.266，−25.9%）。

## Problem

**每一张"确定反照率"的贴图，都是带着一个 metalness 决定被生产出来的。** 作者给这个隐式决定起名 **intended metalness** $\mu_X$：贴图的作者"想用它描述的那个材质的 metalness"。

- 一张为"涂装表面"生产的漫反射反照率图，$\mu_D=0$；
- 一张为"金属"生产的镜面反照率图，$\mu_S=1$；
- 而在 MR 工作流里，$C$ 一个通道要同时承担这两种角色。

于是出现一类特定的错误：$C$ 是"以介质身份"画的（偏暗、含漫反射细节），却被 $M=1$ 渲染 ⇒ 渲染结果**太暗**（漫反射没了，只剩 0.5 左右的镜面反射）；反过来（$C$ 以金属身份画的，被当介质）⇒ **太亮且平**（暗的金属 albedo 被当成漫反射 albedo，再叠一个 4% 白高光）。

**这个错误在人工作者身上很少见**——因为他在画 $M$ 的时候知道自己在画什么。**在逐通道回归的网络身上必然出现**——它只被教"base color 该长什么样"，从没被教"这块像素是什么材料"。数据分布只能"鼓励"两者匹配，不能"强制"。

## Historical Context

单图 SVBRDF 估计这条线有两次参数化迁移：

```text
2017  Li et al.      自增强 CNN，SG 工作流（D + S），S 固定 4% ⇒ 根本表示不了金属
2018  Deschaintre    rendering-aware 网络，D + S 联合
2024  IID / RGB↔X    intrinsic decomposition 路线，开始输出 MR
2025  SuperMat       物理一致性 + 交互速率，MR（当前的强基线）
2026  ReMeDi         不换 backbone，换"参数化本身"
```

参数化一侧的迁移是另一条线：**SG（存 $D$ 与 $S$）→ MR（存 $C$ 与 $M$）**。SG 的问题是 $D$ 与 $S$ 的 metalness 互相不约束，可以描述"同时很介质又很金属"的非法材质；MR 把决定显式化成一张 $M$，于是"鼓励"匹配。**但"鼓励"不是"强制"**——ReMeDi 的出发点就是这半句话。

## Previous Work

论文把 MR 工作流的方法按"$M$ 相对 $C$ 什么时候定"分成三档（Table 2）：

| 顺序 | 代表 | 问题 |
|---|---|---|
| **先 $C$ 后 $M$** | CHORD / MuMA / Boosting3D | CHORD 的动机很实在（$C$ 在所有属性里最像输入图），但它**在固定的 $C$ 上搜索 $M\in\{0,1\}$** ⇒ $C$ 永远没被修正过 |
| **$C$ 与 $M$ 联合** | RGB↔X / IDArb / Material Anything / ControlMat | 最常见的做法，也是 ReMeDi 要打的靶子：联合不等于一致 |
| **先 $M$ 后 $C$** | PBR3DGen / Rhee & Lee | 最接近本文。Rhee & Lee 把整张图路由到"金属网络"或"非金属网络"，但**决定是逐图的**——一张图里同时有两种材料就失效 |

ReMeDi 的差别只有一句：**两个假设是逐像素选的**（由 $M$ 决定），所以一张图里混着金属与介质也没问题。

## Core Idea

> **不要问"这张 $C$ 对不对"，要问"这张 $C$ 在它自己的假设下完不完整"。**

把 $C$ 拆成两个**各自完整**的材质：

- $C_d$：把**每个像素**都当介质来描述时，它的漫反射反照率应该是多少（$\mu_C^d=0$，无论真实材料是什么）；
- $C_m$：把**每个像素**都当金属来描述时，它的镜面反照率应该是多少（$\mu_C^m=1$）。

然后按 $M$ 混合。只要两个前提成立（"$C_d$ 确实是全介质的"、"$C_m$ 确实是全金属的"），混合结果的 intended metalness **就等于 $M$**——这是代数结论，不是训练目标。

**关键区分**：$C_d\neq D$、$C_m\neq S$。$D$ 与 $S$ 依赖真实 metalness（金属像素的 $D$ 必须是 0），而 $C_d$ 在金属像素上**也**要给出"如果它是介质，它的漫反射反照率会是多少"——一个非零的数。这正是"假设"两个字的含义，也是为什么它必须由独立的网络来产生。

## Technical Approach

### ① 重参数化（式 2–3、11）

二值情形是硬选择 $C(p)=C_d(p)$ if $M(p)=0$ else $C_m(p)$；中间值用线性混合推广：

$$C(p)=\mathrm{lerp}\big(C_d(p),\,C_m(p);\,M(p)\big),\qquad R(p)=\mathrm{lerp}\big(R_d(p),\,R_m(p);\,M(p)\big)$$

roughness 也这么做，理由较弱但成立：$R$ 对两种材料含义相同，但**典型值分布差很多**（高粗糙度在介质上常见得多），所以它也带一个"弱"的 intended metalness。

中间值的解释沿用 MR 的既定语义：$M$ **不是"半金属"，是金属覆盖率**——于是线性混合不是插值技巧，而是面积加权的正确做法。

### ② 假设专属监督（式 8–10，解"没有 GT"）

困难在于**没有资产同时提供两套 metalness 相反、渲染结果相同的贴图**。所以 GT 的 $C$ 只在"它的假设成立"的像素上是合法目标。

按 GT metalness 分两侧：$M_{GT}<\tau_d=0.2$ 记为介质像素集 $\mathcal T^d$，$>\!\tau_m=0.8$ 记为金属像素集 $\mathcal T^m$。扩散模型以 16×16 token 运作，于是把像素比例折成 token 权重：

$$f_k^h=\frac{|P_k\cap F\cap\mathcal T^h|}{|P_k\cap F|},\qquad w_k^h=\mathrm{clamp}\!\left(\frac{f_k^h-\tau_p}{1-\tau_p},0,1\right),\ \tau_p=0.9$$

即"这个 token 覆盖的前景像素里，90% 以上属于本假设"才参与训练。损失是加权 flow matching（式 10）。

### ③ 一个网络只学一个假设（解"前提不被保证"）

**只做 ①②是不够的**——这是全文最值得记的一段推理：

> 监督在"另一种材料的像素"上是空的，那部分靠外推。而**"两个变体都输出真实的 $C$"完美满足监督**，却不满足"各自跟随假设"。

于是用三个共享冻结 DiT、各自带 LoRA（rank 64）的模块：$\mathcal G^d$（只学介电假设）、$\mathcal G^m$（只学金属假设）、$\mathcal G^M$（学 metalness，另加一个只在"GT 明确二值"的位置上生效的二值化惩罚 $\mathcal L_{bin}$）。

**"只学一个假设"意味着它在该假设不成立的像素上也按该假设外推**——这正是需要的行为：$C_m$ 在介质像素上给出的是"如果这是金属会怎样"，而不是"这里其实是介质"。

### ④ 训练与推理

1024²、30K 迭代、24×H20、约 95K 资产、LoRA rank 64、AdamW 5e-5、$\lambda=0.3$。推理三个模块各 20 步，**合计约 60 s/图（单卡）**。

## Key Contribution

1. **给一类错误命名并给出判据**：intended metalness $\mu_X$，以及"$\mu_C$ 是否等于 $M$"这个可检查的一致性条件。
2. **把一致性从损失搬到参数化**：$C=\mathrm{lerp}(C_d,C_m;M)$ 使 $\mu_C=M$ 成为代数结论。
3. **证明"只重参数化不够"**：单网络版本（C）在两个假设下输出了**几乎相同**的图（图 4），于是重参数化退化成恒等式。
4. **一个诚实的消融**（Table 4，512²/10K 的缩水设置，只用于内部对比）：

| 变体 | Overall BC PSNR | **Metal 区 BC PSNR** | Overall 重光照 | **Metal 区重光照** |
|---|---|---|---|---|
| (A) 联合回归 $C+M+R$ | 19.98 | 17.07 | 20.46 | 17.62 |
| (B) 先 $M$ 再条件生成 $C$ | 20.10 | 17.81 | 20.72 | 18.02 |
| (C) 重参数化 + **单网络** | **19.37** | 17.37 | **19.81** | **17.18** |
| **(D) 重参数化 + 分网络** | **20.24** | **18.40** | **20.90** | **18.76** |

**(C) 比 (A) 还差 0.61 dB**——重参数化本身不带来收益，**它只在你真的有两个端点时才成立**。而在金属区，(D) 比 (A) 高 **1.33 dB**、重光照高 **1.14 dB**：**收益集中在"最容易错的那一侧"。**

## Why It Works

**因为 $M$ 是一个插值权重，而插值权重只有在两端都是完整对象时才有意义。**

这一句可以拆成三个可验证的推论，论文里各有一个证据：

1. **端点的完整性是可监督的，中间值的正确性不是。** 没有 GT 能覆盖 $M\in(0,1)$ 的像素（没有任何资产同时提供两套对立假设的贴图），所以中间值只能被"算"出来。
2. **端点必须被隔离。** (C) 的失败方式很具体：一个网络同时学两个假设时，"两个变体都输出真实 $C$"是监督的**全局最优**，于是它选了这条捷径。**隔离不是正则化，是消除捷径。**
3. **一致性 ≠ 正确性。** 论文自己在 Limitations 里划了界：如果 $M$ 本身估错，ReMeDi 会把 $C$ 忠实地做成"错的那个材料"的完整材质——**它保证的是内部一致，不是外部正确**。这句话应当贴在每一个"by construction"的宣称旁边。

## Limitations

1. **不修正 metalness 本身的错误。** 一致性是相对 $M$ 的，$M$ 错则全错，只是"错得很自洽"。
2. **三个网络 = 三倍算力。** 作者说可并行、不增延迟，但显存与总 FLOPs 是三倍。60 s/图对离线入库可以，对交互不行。
3. **合成训练、合成测试。** 95K 资产全是渲染的，SYNTEST 也是渲染的（Objaverse-XL）；WILDTEST 只做定性。**"真实照片上的 intended metalness 是否也这么好"没有数字。**
4. **两个表的数不能直接比。** Table 3（主表）是 1024²/30K，Table 4（消融）是 512²/10K——主表的 +2.86 dB 里包含了 backbone、数据量、分辨率的全部差异；**只归因于方法的增量是 +0.26 dB（overall）/ +1.33 dB（金属区）**。这不是造假，是读法问题，但不读消融就会高估方法。
5. **没有 per-image 计时分解**（三个模块各多少秒），也没有失败案例的定量统计。
6. **512² 消融里，(D-a)/(D-b) 在各自范围上反而更强**（介电假设单独跑 Dielectric BC 22.72 > 22.08；金属假设单独跑 Metal 20.15 > 18.40）——**混合是有代价的**，只是这个代价小于"把假设分配到正确像素"的收益。

## Game Development Relevance

**这是"贴图不是自描述的"这条工程常识第一次被写成可训练的约束。**

1. **扫描 / 摄影测量入库**：最痛的两件事——高光烘进 albedo、$C$ 与 $M$ 不同源——正好是本文的两个靶子。60 s/图的离线成本完全可以塞进导入管线。
2. **AI 生成材质的上线验收**：逐通道指标（每张图的 PSNR）会**系统性放过** intended metalness 错配，因为两张图"各自都很像"。**验收必须看重光照**，这是本文最可直接搬用的流程结论。
3. **材质库审计**：$\mu_C=M$ 是一个可以在已有资产上跑的检查——把 $C$ 当介质与当金属各渲一次，差异过大而 $M$ 又落在中间值附近，就是可疑资产。
4. **纹理管线的位深决策**：既然 $M$ 是覆盖率，那么 $M$ 的量化误差就是**材质比例误差**——8 bit 在金属边界上够不够，取决于边界像素的密度。
5. **对风格化材质的意义有限**：MR 的"物理"在参数化层面；卡通/手绘材质用 MR 只是为了管线统一，本文的一致性约束对它们不构成约束。

## Unreal Engine Relevance

- **输出直接可用**：$C$ / $M$ / $R$ 就是 UE Lit 的 BaseColor / Metallic / Roughness 三个输入，无需转换。
- **检查点做在材质编辑器里**：用 `lerp` 手搭 $C=\mathrm{lerp}(C_d,C_m;M)$ 是 3 个节点的事——**这就是本文的全部数学**，可以在引擎里 5 分钟复现它的核心断言。
- **不涉及运行时**：三个 DiT 模块是离线工具，与 Nanite / Lumen 无关。
- **对 Substrate 的影响**：分层材质里每一层都有自己的 $C$ 与 $M$，**intended metalness 的问题会按层数翻倍**。

## Technology Evolution

```text
参数化一侧
2012  Burley / Disney     SG → MR；metallic 明确写成"两个模型的线性混合"
2013  Karis / UE4         MR 进实时引擎，成为默认
2015+ glTF / USD          MR 成为交换格式（单通道 M 省 1/3 贴图）

估计一侧
2017  Li et al.           SG + 固定 4% 镜面 ⇒ 表示不了金属
2018  Deschaintre         rendering-aware，D + S 联合
2024  IID / RGB↔X         intrinsic decomposition，开始出 MR
2025  SuperMat / Material Anything   物理一致性 + 交互速率（强基线）
2026  ReMeDi              不换 backbone，把"一致"写进参数化 ⇒ +2.86 dB / +2.74 dB
        └ 遗留：M 错时全错；三网络；只在合成数据上有数字
```

**注意这条线的刻度在变**：2017–2024 比的是"网络能看多懂"，2026 开始比"**参数化允许你错在哪**"。ReMeDi 的 +2.86 dB 里有多少来自 DiT backbone、有多少来自参数化，论文没有拆开——这是本页最大的读法风险。

## Relationships

### Based On

- [[Metallic-Roughness Workflow]] —— 式 (1) 的两行（$D=\mathrm{lerp}(C,0;M)$、$S=\mathrm{lerp}(0.04,C;M)$）直接来自这套参数化；本文做的就是"让 $C$ 配得上它"。
- [[Burley 2012 — Physically Based Shading at Disney|Burley 2012]] —— "metallic 是两个模型的线性混合"这句话是本文的地基；"中间值必须合法"（§5.7）是本文要解决的问题的原始表述。
- [[Inverse Rendering]] —— 单图 SVBRDF 是逆渲染最成熟的子问题；本文的新意全在"输出该怎么参数化"。

### Extends

- [[2026-11-26-Pentapati — SplatTok Data-Driven Compression of 3D Gaussian Splats|SplatTok 2026]] 的方法论同构：**换被优化的对象**（那里从"图元属性"换成"渲染图"，这里从"逐通道贴图"换成"两个完整假设"）。两篇都在说：**你优化的那个东西，不是你要的那个东西。**

### Related

- [[BRDF]]、[[Microfacet Theory]] —— $D$ 与 $S$ 最终进入 $D\cdot G\cdot F$；$F_0=S$。
- [[Split-Sum Approximation]] —— 换 $F_0$ 在实时管线里只是换 LUT 的查表坐标，所以"$C$ 是镜面反照率"这件事在引擎里是免费成立的。
- [[Texture Filtering]] —— mip 平均 $M$ 之后，中间值语义必须仍然成立；这是本页"覆盖率"解释的直接后果。
- [[Bidirectional Texture Function]] —— 同一族"中间值"问题（空间维可滤波、角向维不可）。
- [[2026-11-18-Gokbudak — One Code per Material Neural Representation and Generation of Measured BTFs|Gokbudak 2026]] —— 另一种"把材质变成码"的路；两者都在压缩材质的自由度，但一个压维度、一个压一致性。
- [[Subset Selection]] —— 远亲：都在"给定预算下，哪些自由度可以交给参数化、哪些必须留给优化"。

### Contrasts

- [[2026-10-30-Liu — Texture Consolidation of Generated Multi-View Renderings|Liu 2026]] —— 那篇把多视角渲染"烘"成一张贴图（从图像到纹理），本篇从一张图像反推物理参数。**一个把光烘进去，一个把光剥出来。**

## Personal Knowledge State

- **Easy**：MR 工作流的操作层（怎么画、怎么打包、UE 里三个滑块在哪）；为什么金属没有漫反射。
- **Normal**：0.04 的出处（$[0,0.08]$ 中点 = IOR 1.5 的精确值）；metalness 是**覆盖率**而不是材料状态；intended metalness 这个失效模式；"一致性 ≠ 正确性"这条边界。
- **Hard**：LoRA 分模块训练时的梯度隔离（为什么共享 backbone 不会让两个假设互相污染）；token 权重 $\tau_p$ 的取值对金属边界的影响（论文没扫）。

## Learning Value

**三个可迁移动作：**

1. **看多通道输出时，先问"这些通道之间有没有一致性约束"**。有 ⇒ 逐通道指标会系统性放过联合错误；没有 ⇒ 逐通道指标就够。**这是一条 30 秒能做完的判断，能省掉一次上线事故。**
2. **"by construction" 要立刻追问"那什么没被构造住"**。这里没被构造住的是 $M$ 本身；MESSI 里没被构造住的是"退出之后还能不能继续"；DiffSimp 里没被构造住的是"换一次 $\sigma$ 之后的梯度还有效吗"。**每个构造性保证都有一个它管不到的邻居。**
3. **消融表与主表要在同一设置下读**。这里主表 1024²/30K、消融 512²/10K，两个表的 dB 不可相减——**先找"缩水设置"那一行，再看增益**。

## Visualization

![[Research/Files/Pictures/Fei 2026 — 图1 同一张base color配错metalness的后果.png]]

**读图**：上半是**同一块龙头雕饰**的两条生产路径。左：$C$ 以金属身份生产（$\mu_C=1$，偏亮的银灰），配 $M=1$ 渲染正确（右下绿勾），被当介电渲染就"太亮且平"（红叉）。右：$C$ 以介质身份生产（$\mu_C=0$，偏暗的深灰），配 $M=0$ 是对的（"plausible plastic"），被当金属渲染就"太暗"。**两张 $C$ 各自都像一张正常贴图——错的是配对。** 下半是真人盔甲：SuperMat 给出"暗 $C$ + 高 $M$"（白框区域），渲染整体偏暗；ReMeDi 的两个变体在高金属处**同样是暗的**（$C^m$ 那格），但混合后被正确解读。（图 1，Fei et al. 2026）

![[Research/Files/Pictures/Fei 2026 — 图2 双假设重参数化与三网络.png]]

**读图**：三块正好对应三节。(a) 是**代数**：$C^d$、$C^m$ 各自带固定的 $\mu_C=0/1$，$M$ 从 0 扫到 1，中间那条蓝→橙的渐变就是 $\mathrm{lerp}$。(b) 是**监督**：GT metalness 分出蓝（$M_{GT}<\tau_d$）与橙（$>\tau_m$）两块，每个 token 的权重由它覆盖的 16×16 像素里"本假设像素"的比例决定。(c) 是**隔离**：两路共享冻结 DiT，各带 LoRA，各自只对一种假设的 token 负责；下面那条"Frozen / Trainable"图例是整个设计的支点——**共享的是能力，隔离的是假设**。（图 2，Fei et al. 2026）

![[Research/Files/Pictures/Fei 2026 — 图4 消融单网络把两个假设输成同一张图.png]]

**读图**：全篇最值钱的一张。四行是四个变体，最后两行（(C) 单网络 / (D) 分网络）每格下面还带着**"金属假设"与"介电假设"两张小图**（橙/蓝底）。看 (C)：两张小图**几乎一模一样**——单网络根本不管自己被要求跟哪个假设，它只是把真实 $C$ 输出两遍，于是 $\mathrm{lerp}$ 退化成恒等。(D) 的两张小图差别明显（金属假设那一侧明显更暗、更纯色）。**这就是"重参数化必须配隔离"的视觉证据**——没有它，(C) 在 Overall 上比联合回归的 (A) 还差 0.61 dB。（图 4，Fei et al. 2026）

## Notes

原文 PDF 已归档：`Research/Files/PDF/Fei 2026 - ReMeDi.pdf`（32.6 MB，27 页）。

- **Table 3 vs Table 4 的设置差异是读这篇的第一道坎**：主表 1024²/30K iters、消融 512²/10K iters + 4× batch。所以"ReMeDi 比 SuperMat 高 2.86 dB"与"(D) 比 (A) 高 0.26 dB"**不矛盾也不可相加**。
- **一个容易漏的细节**：metalness 模块的输出在 B/C/D 三个变体里完全相同（RMSE 0.237 / Acc 0.874），因为它们共用同一个 $\mathcal G^M$——**消融的差异全部来自 $C$ 与 $R$ 的生成方式**，这也是为什么能干净地归因。
- 作者栏里 **Boxin Shi（北大）** 是逆渲染 / 光度学老面孔，**Chunchao Guo（腾讯混元3D）** 是 3D 生成侧的常客；这篇是"学术逆渲染 + 工业 3D 资产管线"的典型合流。
- 与库内 [[2026-09-29-Texture Space Material Diffusion]] 的分工：那篇生成材质（无输入或弱条件），这篇从一张图反解。**一个 forward、一个 inverse，共用同一套 MR 参数化。**

**相关每日记录**：[[2026-11-27]]
