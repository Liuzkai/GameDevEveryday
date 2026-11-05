---
type: paper
title: "FlowRig: Unified Skeleton Generation for 3D and 4D Meshes via Flow Matching"
authors:
  - Fengzhi Guo
  - Nate Morrill
  - Shiqi Li
  - Cheng Zhang
year: 2026
published: 2026-11-02
venue: arXiv preprint（cs.CV primary，cross-list cs.GR）
url: https://arxiv.org/abs/2611.03775
code: ""
project_page: https://tamu-visual-ai.github.io/FlowRig/
category:
  - animation
  - rigging
  - skeleton
  - flow-matching
  - generative
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: Research（离线可用：静态 1.1 s，24 帧序列 23.4 s）
user_level: Normal
status: unread
tags:
  - rigging
  - skeleton
  - flow-matching
  - 4d
  - temporal-consistency
  - bone-count
---

# FlowRig: Unified Skeleton Generation for 3D and 4D Meshes via Flow Matching（Texas A&M, 2026）

> ⚠️ 论文自身命名未收口：**标题是 FlowRig，摘要里写的却是 "We propose MoTWorld"**。本笔记按标题记作 FlowRig；引用时若要写方法名，说明原文两处不一致。

## TL;DR

**"时序一致"不是把每帧的骨架磨平，是让整段序列共享同一个离散决策。**

FlowRig 把一整段骨架序列当成**一个样本**：128 个骨架槽 × 7 维（存在位 + 两个端点）× $N$ 帧，用 flow matching 从噪声一次前馈积分出来；**拓扑由一个对时间池化后的图头统一给出**，骨骼数是一个标量条件（CFG 引导，$s=5$，可指定 1–128）。

结果里最值钱的不是精度，是三个**恰好为 0** 的指标：骨长抖动 BDD 0.0038、骨骼数偏差 BND **0**、拓扑距离 GSD **0**、汉明距离 HD **0**（对照 Puppeteer-D 在 $N=24$ 时 0.0150 / 3.18 / 0.265 / 14.19）。**0 不是"抖得小"，是没有自由度可抖**——拓扑和骨骼数在整段上只有一份。

## Problem

自动绑骨（automatic rigging）分成两个几乎不相往来的设定：

| 设定 | 现状 | 病 |
|---|---|---|
| 静态 3D 网格 | Pinocchio 之后一路做到 RigNet / MagicArticulate / UniRig / RigAnything / Puppeteer，质量已经不错 | 多为**自回归**（逐个关节或逐个子物体预测）⇒ **骨骼数不可指定**；套到动态网格上逐帧跑 ⇒ 拓扑闪、骨长飘 |
| 动态 4D 网格 | AKd / RigGS / RAT4D / SPRig 等 | 多为**逐序列优化**（一个 8 帧实例 17 分钟）⇒ 不可批量；或只是静态 rigger 的"一致性扩展" |

作者要的是一件事：**一个前馈模型，3D 与 4D 是同一个任务的两个实例**，而且骨骼数可控。

这里有个常被忽略的前提问题：**"骨骼数"在自回归方法里根本不是一个可指定的量**——它由"什么时候停止预测"决定。想把骨骼数变成输入，就得先把表示改成"固定槽 + 存在位"。

## Historical Context

绑骨这条路有两个哲学，从 2007 年就分岔了（详见 [[Baran 2007 — Automatic Rigging and Animation of 3D Characters|Baran & Popović 2007]]）：

- **提取（extraction）**：从形状里算出一副"属于这个形状"的骨架（中轴、Voronoi、排斥力场）。骨架是形状的函数。
- **嵌入（embedding）**：拿一副**给定的**骨架，把它塞进形状里。骨架是**接口**——它先存在，形状去适配它。

Baran 选了后者，理由很硬：**只有骨架是接口，动捕数据才能复用**。这个判断在 19 年后没有变，变的是接口从哪来——**从"人给的一棵树"变成"数据里的先验"**。FlowRig 是 template-free 的，但它依然在产出"能被动画驱动的东西"，而不是"这个形状自然的中轴"。

中间那一段（RigNet 2020 → MagicArticulate / UniRig / RigAnything / Puppeteer）本库尚未收录，是明确的图谱缺口；本篇的表 1 把它们的静态精度排在一起，可以当作这条线的横向快照。

## Previous Work

作者自己划的三类对照：

1. **静态 rigger 逐帧套用**（Puppeteer-D 是他们最主要的 4D 基线）：每帧独立跑一次，再靠后处理抹平。**病根在"每帧独立决策"**——拓扑是一个离散选择，逐帧选就必然闪。
2. **逐序列优化**（RigGS）：把骨架当隐变量对每段序列拟合。质量好、代价高（8 帧 17 分钟），且**不可泛化**（每个新资产都要重跑）。
3. **一致性扩展**：在静态 rigger 上加时序正则。**正则只能压小抖动，不能消掉自由度。**

## Core Idea

### ① 表示：固定槽 + 存在位，把"骨骼数"从架构里拿出来

每帧 128 个槽（$L_s=128$），每槽 7 维：

$$\mathbf{x}_{0,i,j}=\big[e_{i,j},\ \mathbf{p}_{i,j,0},\ \mathbf{p}_{i,j,1}\big],\quad e\in\{0,1\},\ \mathbf{p}\in\mathbb{R}^3\times\mathbb{R}^3$$

**骨骼数 = 有多少个 $e=1$**，于是"1 到 128 根"不需要改网络，只需要改条件。

### ② 生成：在最优传输直线上做 flow matching

$$\mathbf{x}_t=(1-t)\mathbf{x}_0+t\mathbf{x}_1,\qquad \mathbf{v}^{\star}=\mathbf{x}_1-\mathbf{x}_0$$

网络预测 $\hat{\mathbf{v}}=\mathcal{M}(\mathbf{x}_t,t,\mathbf{c};\Theta)$，推理时从 $t=1$ 用 Euler 积分回 $t=0$。**注意它是"一次前馈"而不是"逐帧"**：整个 $N\times128\times7$ 张量一起积分，帧间注意力在积分过程中就生效。

### ③ 拓扑：池化一次，全序列共用一份

这是全篇最关键的一步。图头不逐帧预测 parent，而是先沿时间平均：

$$\bar{\mathbf{x}}_0=\tfrac{1}{N}\sum_i\hat{\mathbf{x}}_{0,i},\qquad \bar{\mathbf{h}}=\tfrac{1}{N}\sum_i\mathbf{h}_i,\qquad \mathbf{z}=\mathrm{concat}(\mathrm{sg}(\bar{\mathbf{h}}),\mathrm{sg}(\bar{\mathbf{x}}_0))$$

再由 $\mathbf{p}=G(\mathbf{z})\in\mathbb{R}^{L_s\times(L_s+2)}$（多两通道：root / 不存在）给出**一套** parent 索引。

**"一套"是 GSD/HD 恒等于 0 的全部原因。** 序列内没有任何一个时刻有机会选一个不同的拓扑。

### ④ 骨骼数：当作"文本提示词"处理

加一个标量条件 $y$，训练时以 $p_{\text{drop}}=0.5$ 随机丢掉换成 $y=-1$，推理时

$$\hat{\mathbf{v}}_{\text{cfg}}=\hat{\mathbf{v}}_{\text{uncond}}+s\cdot(\hat{\mathbf{v}}_{\text{cond}}-\hat{\mathbf{v}}_{\text{uncond}}),\quad s=5$$

这是把 classifier-free guidance 从"文本→图"原样搬到"骨骼数→骨架"。**工程上极省事：不动架构，只在条件里加一个标量。**

### ⑤ 训练：序列级唯一分配，锚在帧 1

用 DETR 式 Hungarian 匹配，但**对整个序列只算一次分配**，且**锚定帧固定为 $i=1$**。

这一条看似细节，实际是课程能成立的原因：$N=1$ 时，序列级匹配退化成静态任务的标准匹配 ⇒ **静态不是"另一个任务"，是同一任务的最小实例**。所以 3D 预训练 → 4D 微调是"同分布外推"，不是"换任务重训"。

总损失：

$$\mathcal{L}=\mathcal{L}_{\text{FM}}+\lambda_{\text{NLL}}\mathcal{L}_{\text{NLL}}+\lambda_{\text{graph}}\mathcal{L}_{\text{graph}}+\lambda_{\text{geo}}\mathcal{L}_{\text{geo}}+\lambda_{\text{smooth}}\mathcal{L}_{\text{smooth}}$$

其中两处值得单拎：

- **图头损失按 $(1-t)^{\gamma_p}$ 加权**：$t$ 小（接近干净样本）时拓扑目标可靠，权重大；$t$ 大时噪声主导，降权避免"用噪声去学拓扑"。
- **几何损失** $\mathrm{ReLU}(\mathrm{SD}(\hat{\mathbf{x}}_0,M_i))$：关节掉到网格外面就罚。**这是 Baran 2007 "embedding 必须在体内"那条硬约束的软版本**，同一条约束在 19 年后换了实现方式。

## Technical Approach

```text
4D 网格序列 M_{1:N}（点云 + 法线，每帧）
   ↓ 冻结的 Michelangelo 点云 VAE
网格 token c ∈ R^{N×L_m×H}
   ↓ 与 128 个骨架槽 token 拼接（每条 (L_s+L_m)，共 N 条）
DiT-L（depth 24，H=1152，16 头）
   · 帧内注意力 ↔ 全局注意力交替
   · 时间 RoPE 用帧索引
   ↓
三个头：Skeleton(S) 头 / Graph 头（池化后出 1 套拓扑）/ Uncertainty(U) 头
   ↓ 5 次采样 + 用 U 头当置信分数选最优
骨架序列 S_{1:N}
```

- **条件**：骨骼数 $y$ 经 AdaIN 注入（CFG）；
- **课程**：① 静态 3D 预训练（$N=1$，lr 1e-4）→ ② 4D 重建（全帧网格条件，lr 5e-5）→ ③ 4D 预测（mask 掉部分帧的 token；偏结构化掩码：前一半 / 后一半 / 只有两端 / 全观测，lr 3e-5）；
- **数据**：静态用 Articulation-XL 2.0（33,000 个带绑定模型）；4D 从 TexVerse 筛出约 2,500 段（2,000 训 / 500 测），筛掉的包括准静态运动、时序错位、骨架标注错误。

## Key Contribution

1. **统一的 3D/4D 前馈绑骨**：不是"先做静态再做一致性"，而是把 $N$ 当维度。
2. **时序一致性由结构保证，不由损失保证**：整段共享一套拓扑 + 一个骨骼数 ⇒ 离散量零方差。
3. **骨骼数成为可指定的输入**（1–128），且能无条件推断合理值。
4. **3D→4D 课程**：在只有约 2,500 段配对 4D 数据的条件下，把 33,000 个静态绑定资产用起来。

## Results

**静态（CD 三变体，越低越好；Articulation-XL 2.0 / ModelsResource / Diverse-Pose）**

| 方法 | Art-XL 2.0 J2J | ModelsResource J2J | 备注 |
|---|---|---|---|
| Pinocchio (2007) | 0.0832 | 0.0685 | 表里最差，但**必须报** |
| RigNet (2020) | 0.0762 | 0.0722 | |
| MagicArticulate | 0.0326 | 0.0411 | |
| UniRig | 0.0331 | 0.0396 | |
| RigAnything | 0.0489 | **0.0343** | **在 ModelsResource 上最好** |
| Puppeteer | 0.0311 | 0.0377 | 4D 基线 |
| **FlowRig** | **0.0264** | 0.0358 | 5 次采样选优 0.0259 |

**4D（TexVerse，$N=24$）**

| | J2J-W | BDD | BND | GSD | HD |
|---|---|---|---|---|---|
| Puppeteer-D* | 0.0163 | 0.0150 | 3.1763 | 0.2654 | 14.1894 |
| **FlowRig** | **0.0142** | **0.0038** | **0** | **0** | **0** |

**开销**：静态单网格 **1.1 s**（Pinocchio 3.9 / RigNet 4.5 / UniRig 2.9 / MagicArticulate 2.4 / Puppeteer 1.5，同表口径）；序列 1 / 2 / 4 / 8 / 16 / 24 帧 → 1.1 / 1.7 / 2.2 / 4.4 / 11.9 / **23.4 s**。对照 RigGS 逐序列优化 8 帧 17 分钟，**约 230×**；1,656 个实例全跑 RigGS 需 469 H200·h，FlowRig 约 2 GPU·h。

**泛化（Objaverse 1,656 实例、10 类动物、未参与训练，$T=8$）**：相对 RigGS / Puppeteer-D 的 J2J 分别低 19.3% / 19.2%，30 个"类别×指标"单元里 28 个领先。

**消融**：去掉静态预训练 J2J 0.0213 → 0.0142（**3D 先验是主力**）；把图头换成 1-NN 建树，GSD 0 → 0.0375、HD 0 → 2.28（**拓扑的稳定性完全来自那个池化头**）。

## Why It Works

**一句话：它没有去"约束"时序一致性，而是把产生不一致的那个自由度删掉了。**

拆成三层看：

1. **连续量 vs 离散量要分开治理**。骨长、速度是连续量，用 $\mathcal{L}_{\text{smooth}}$ 罚；拓扑、骨骼数是离散量，**罚不了，只能一次定**。所有"逐帧独立预测 + 平滑后处理"的方法都在用连续手段治离散病——所以 BDD 能压下去（Puppeteer-D 0.0150），BND/GSD/HD 压不下去（3.18 / 0.265 / 14.19）。
2. **池化是"在时间上投票"，而投票对离散量是良定义的**。对一个 parent 分类做时间平均再 argmax，语义上就是"整段序列认为谁是谁的父"。对连续端点做平均则会把姿态信息抹掉——所以池化只给图头用、不给骨架头用。
3. **课程之所以能迁移，是因为锚定帧让两个任务共用同一个目标函数**。不是"预训练帮我初始化"，是"静态就是 $N=1$ 的同一个问题"。这一点在消融里非常干净（0.0213 → 0.0142）。

## Limitations

- **4D 数据只有动物，没有人形**。4D 评测集是 Objaverse 派生的 10 类（双足哺乳 27.4%、鱼 18.2%、鸟 12.3%、恐龙 11.7%、四足 11.1%、昆虫 9.3%…），**游戏最需要的双足人形角色恰好不在 4D 训练/评测分布里**。这不是小瑕疵，是迁移风险。
- **鱼类仍输给逐序列优化**（J2J/J2B 上 RigGS 领先 0.4%/0.5%）。理由作者自己给了：鱼的运动平滑、自由度低，**逐序列拟合在这种情形下本来就够用**。
- **ModelsResource 上不是最优**：零样本 J2J 0.0358 输给 RigAnything 的 0.0343，J2B / B2B 同样输。"专业设计骨架"这一档，学习方法的优势不明显。
- **23.4 s / 24 帧**，随帧数近似线性，**离实时很远**；这是离线资产工具，不是运行时。
- **$L_s=128$ 是硬上限**；骨骼数再多要换表示。
- **4D 监督依赖重筛**：TexVerse 原始数据要过 Qwen3.5 预筛 + 人工核验才留下约 2,500 段，**数据成本被藏在附录里**。
- 摘要与标题的方法名不一致（FlowRig / MoTWorld），**preprint 尚未收口**。

## Game Development Relevance

**它是"资产入库"那一步的自动化，不是运行时技术。** 三个可直接用的落点：

1. **扫描 / 外包 / UGC 资产的自动绑骨**：静态 1.1 s 意味着可以在导入管线里跑，失败就退回手工。
2. **LOD 骨架**（图 6 演示的最好）：同一份网格，远处要 12 根、中景 20 根、特写 27 根。过去这是"三份绑定资产"，现在是**同一个模型换一个条件标量**。**这是"骨骼数成为输入"真正的产线价值**——它把 LOD 从"资产数量"变成"参数"。
3. **4D → 骨骼动画**：把顶点动画缓存（Alembic / 解算缓存 / 商店买的带动画资产）反推成骨骼动画。这类资产在管线里通常是"死的"，转成骨骼才能重定向、才能压体积、才能进动画蓝图。

## Unreal Engine Relevance

- **不是运行时**：23 s/24 帧排除掉任何"每帧调用"的想法；连"导入时同步调用"都偏慢，应该是**后台任务 + 结果缓存**。
- 真正对应的是 **Skeleton Asset 的自动生成 + Physics Asset 的初始胶囊**，以及 Control Rig 的起始骨架。当前 UE 的自动绑骨仍是第三方（Maya / Blender 的 bone heat、或 Houdini 的 rig 生成），FlowRig 这一档如果工程化，位置是 **DCC 侧插件**而不是引擎内。
- **骨骼数条件**在 UE 里对应的是 Skeleton LOD / 骨骼精简（`Skeletal Mesh LOD Settings` 的 bone removal）。现在的做法是"删骨 + 重算权重"，FlowRig 的做法是"重新生成一副"，**前者保拓扑、后者保形状**——这两条路的差别在多动画共享同一套 Skeleton 时会暴露。

## Technology Evolution

```text
1988  LBS 表述（权重是输入，没人管它从哪来）
  ↓
2007  Baran & Popović：把"绑骨"整个自动化
        · 嵌入（不是提取）⇒ 骨架是接口
        · 热平衡解权重 ⇒ 权重第一次有"算法"
        · max-margin 学罚项权重 ⇒ 把"品味"变成可学的向量
  ↓
2011  bounded biharmonic weights（同一组作者，把热扩散换成有界调和）
  ↓
2020  RigNet：神经网络直接从点云回归骨架 + 权重（库外）
  ↓
2023-25  MagicArticulate / UniRig / RigAnything / Puppeteer
        · 自回归、逐子物体、大模型化
        · 共同病：骨骼数不可指定、逐帧套用会闪
  ↓
2026  FlowRig：把序列当一个样本 + 骨骼数当条件 + 拓扑池化一次
```

**这条线的形状，是"接口"越来越显式**：2007 骨架是人给的模板，2020 之后骨架是模型输出，2026 骨架数和拓扑变成**可以点的旋钮**。**自动化走到了"可控"这一步，才真的能用进产线**——不可控的自动化只能当黑盒，可控的才能当工具。

## Relationships

### Based On

- [[Baran 2007 — Automatic Rigging and Animation of 3D Characters|Baran & Popović 2007]]（**问题定义与"接口"立场**；连"关节必须在网格内"这条约束都继承了，只是从硬约束变 SDF 软罚项）
- flow matching / OT 直线路径（Lipman 等；本篇是标准用法，没有改生成框架）
- DETR 式集合预测 + Hungarian 匹配（序列级唯一分配的来源）
- classifier-free guidance（骨骼数条件的来源）

### Extends

- **静态 rigger 的自回归范式 → 集合预测范式**：自回归的停止条件就是骨骼数，所以"可控骨骼数"必须先废掉自回归。**这是"表示决定能力"的一个清晰案例。**
- **Puppeteer 的 slot 表示 → 序列级 slot 表示**：槽不变，但槽的语义从"每个子部件一个槽"变成"整段序列共享一组槽"。

### Related

- [[Skinning]]（下游：FlowRig 附录 F 另训了一个 SkinningTransformer 出权重，性能"competitive"，**没有成为主贡献**——说明"出骨架"和"出权重"仍是两件事）
- [[Skeletal Animation]]（骨架是运行时的接口；本篇的"骨骼数条件"直接对应骨架 LOD）
- [[Motion Retargeting]]（**逻辑上的下游**：骨架生成得越"标准"，重定向越省事；反过来说，**生成模型的输出拓扑不固定，反而给重定向出难题**——这是这条路尚未解决的问题）
- [[Automatic Rigging]]（本页所属概念）

### Followed By

（待观察）三个方向：把骨骼数从"标量"升级成"语义规格"（作者自己列的未来工作：文本引导的 rig 规格）；把权重和形变联合预测；把 4D 从"物体"推到"人形 + 衣物"。

## Personal Knowledge State

**Normal（结构层可读，数学层不必碰）**

- **Easy**：为什么要自动绑骨、骨骼数影响什么、LOD 骨架的产线价值；
- **Normal**：为什么"逐帧独立决策"的拓扑必然闪、池化为什么对离散量良定义、槽表示如何让骨骼数变可控、课程为什么靠"锚定帧 1"成立；
- **Hard**：flow matching 的 OT 路径推导、DiT 的帧内/全局交替注意力细节——**要用再读，读结论层足够**。

## Learning Value

**能迁移走的是一条判据，不是一个方法**：

> 当一个量是**离散**的（拓扑、骨骼数、材质槽、LOD 档位、排序分组），**不要试图用损失函数让它"稳定"**——它没有梯度意义上的"稳定"，只有"选了几次"。要么一次定，要么接受它闪。

把它挂到自己的管线上问一遍：哪些东西现在是"每帧/每个资产独立决定"的？其中哪些其实是离散的？

## Visualization

![[Research/Files/Pictures/Guo 2026 — 图1 3D与4D绑骨能力雷达.png]]

**读图**：左半是静态 3D 绑骨结果（人形 / 鹿 / 蟹 / 蛙…，蓝线是生成的骨架），右半是 4D 序列（每一列是同一资产的不同时刻，骨架沿时间保持同一套拓扑）。中间的雷达图是作者自评的五个轴——**3D Rigging / Controllable # Bone / 4D Consistency / Prediction / Speed**，FlowRig（蓝）在"可控骨骼数"和"4D 一致性"两轴上把 Puppeteer（粉）与 UniRig（绿）压到中心。**注意雷达图是自评口径，不是第三方基准。**（图 1，Guo et al. 2026）

![[Research/Files/Pictures/Guo 2026 — 图2 骨架流匹配架构.png]]

**读图**：左边两路输入——**噪声 4D 骨架**（$h^t_0..h^t_N$）与 **4D 网格**（经 $\mathcal{E}_m$ 编码成 $c_0..c_N$），按帧交错打包成一条长序列；中间是 DiT 的"帧内注意力（F_i.Attn）↔ 全局注意力（Global Attn）"交替块，条件 $n,\phi$ 经 AdaIN 注入；右边三个头，**Graph Head / Skeleton Head 的输出汇到 Selection**，再由 U 头（Uncertainty）当分数挑一个样本。**看这张图只抓一件事：Graph Head 的输入是先 pool 过的**——拓扑不是逐帧输出再合并的。（图 2，Guo et al. 2026）

![[Research/Files/Pictures/Guo 2026 — 图6 骨骼数可控对照.png]]

**读图**：上排是 FlowRig——GT、无条件（Auto）、指定 $n=n_{GT}=27$、$n=20$、$n=12$；下排是五个基线（Anymate / MagicArticulate / UniRig / RigAnything / Puppeteer），**它们都只能给一个结果**。这张图是全篇最实用的一张：**$n=12$ 时手指骨全部消失、脊柱只剩三节，但整体姿态仍然成立**——这正是"远景 LOD"要的东西，也是"删骨 + 重算权重"很难做到的（删完权重往往崩）。顺带一个观察：Anymate 那条腿骨一路穿到脚底外侧、RigAnything 的脊柱直接连到骨盆，都是"拓扑合法但结构不合理"的典型。（图 6，Guo et al. 2026）

**没有做 HTML**：三张原图已经把"能力对比 / 架构 / 可控性"讲完；本页的信息结构是"一个表示 + 三个头"，Markdown 表格足够。按 §25 自查三问，第 ②③ 条都不满足。

## Notes

- **窗口**：提交 2026-11-02，公告在 `Wed, 4 Nov 2026` 段（当日仅 2 条）；
- **同批的另一篇**是 [[2026-11-03-Jeon — Phase-DGS Dynamic Gaussian Splatting from Unsynchronized Multi-view Video|Phase-DGS]]，两篇在"换参照系"上是同构的，见 [[2026-11-05]]；
- **机构**：Texas A&M（Cheng Zhang 组，tamu-visual-ai）；代码截至入库未放；
- **口径提醒**：表 1 的推理时间来自同一张表，但**没有逐项注明硬件**；Baran 2007 的 3.9 s 显然是复跑值（原文是 1.73 GHz Core Duo 上 31.3 s 均值），两者不可直接比；
- **4D 评测的采样**：每个实例采样 5 次取平均以降方差，选优版本（Ours*）额外用 U 头当分数——**"5 次采样"这个成本没有进 1.1 s / 23.4 s 的表**；
- 原文附录 F 的 SkinningTransformer 用了 Michelangelo + PartField 两个冻结编码器 + 拓扑感知关节注意力偏置（按图距离 $\mathbf{D}$ 加 bias）——**这部分自成一篇文章，本文只到 "competitive"**。
