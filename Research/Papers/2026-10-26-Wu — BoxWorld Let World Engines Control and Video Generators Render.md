---
type: paper
title: "BoxWorld: Let World Engines Control and Video Generators Render"
authors: ["Zike Wu", "Ahmed Hatem", "Chintan Trivedi", "Wallace Michel Pinto Lira", "Xuebin Qin", "Lele Wang", "Renjie Liao"]
year: 2026
published: 2026-10-26
venue: "arXiv preprint（2610.36371v1，cs.CV；27 页）"
url: "https://arxiv.org/abs/2610.36371"
code: ""
project_page: "https://boxworld-web.github.io"
category: ["world-models", "interactive-video", "state-interface", "controllability", "autoregressive-video"]
importance: A-
historical_importance: 0
game_relevance: 5
production_readiness: "Research（20.7 FPS 单卡可交互，但要一个 coding agent 先生成引擎，且规则是模板化的）"
user_level: Normal
status: unread
---

# BoxWorld: Let World Engines Control and Video Generators Render

## TL;DR

**"生成的世界听不懂规则"这件事，多半不是模型不够强，是规则放错了地方。**

现行两条路都不好：端到端视频世界模型把"规则是否满足、后果是什么"**隐式学进生成器**，于是你没法指定"花开需要浇灌"；把状态做厚的路线（模拟器 / 3D 缓存 / 轨迹）能精细控制，但接口**又重又专**，拿不到视频里。

BoxWorld 的做法是把两件事**按职责切开**，中间只留一个 **Object-Centric State Interface**：

$$S_t=\{(b_{t,i},\ c_i,\ d_{t,i})\}_{i=1}^{N_t}$$

三个分量各管一件事：**布局框** $b$（在哪，兼作 attention mask）、**持久属性** $c$（是什么，外观/身份）、**动态状态** $d$（现在怎样，运动/事件）。规则由 **coding agent 写成可执行代码**（Box World Engine），引擎跑出状态流，视频生成器只负责**把状态画出来**。

**这一篇最值得记的不是分数，而是一个对照实验**：

| | Paired-CF（成对分支都对） |
|---|---|
| LingBot-v1 | 16.2 |
| **LingBot-v1 + Oracle（把正确后果当文本喂给它）** | **16.2** |
| LingBot-v2 † | 10.8 |
| **LingBot-v2 † + Oracle** | **16.2** |
| LongLive（本文的底座） | 13.5 |
| **BoxWorld** | **97.3** |

**把答案直接告诉端到端模型，它还是做不到。** 所以缺的不是信息，是**计算发生的位置**。这一条比任何 SOTA 数字都重要。

![[Research/Files/Pictures/Wu 2026 — 图1 同一规则的两个分支与渲染结果.png]]
**读图**：两行是两条规则（"浇灌才开花" / "有钥匙才开门"），每行两个分支只差**前置条件是否满足**。左起第二列是引擎内部的布局框状态（`Flower: bloom` / `Flower: closed`），最后一列是渲染结果。**看两行的最后一列**：上面的花开了、下面的没开；上面的门开了、下面的锁着。**这不是"生成得像"，是"生成对了"**——而这件"对"由左边那几行 if/else 保证，不由扩散模型保证。（图 1，Wu et al. 2026）

## Problem

要一个"用户可定义行为"的交互视频世界，系统必须做三件事：**追踪对象状态 → 应用交互规则 → 把后果渲染出来**。

端到端模型把三件事都塞进生成器。后果是：

- **规则没法指定**。你想说"钥匙在手里门才开"，只能靠提示词或训练数据分布暗示；
- **后果没法推理**。模型输出的是"最可能的下一帧"，不是"规则蕴含的下一帧"——于是**它倾向于生成那个事件，而不管前置条件**；
- **不可审计**。你不知道它为什么开了门。

另一条路（把状态做厚）解决了可控性，但接口随控制精度变重：轨迹 → 布局 → 物理先验 → 3D 缓存 → 完整模拟器，**越精细越专、越贵、越难从视频里拿到监督**。

原文把这条权衡写成一句话：**generalization vs fine-grained control**。

## Historical Context

```text
2018  Ha & Schmidhuber：World Models —— "世界可以学出来"的起点
2021  Menapace：从视频学可玩的动态（早期 playable world）
2024  Genie / GameNGen：tokenized latent world model + 大规模视频语料
2024  StreamingLLM / Diffusion Forcing：让自回归视频生成"能一直生成下去"
2025  Matrix-Game / WorldKV / ReWorld：动作条件 + 检索记忆（[[World Models for Games]] 的第二层）
2025  LongLive / Self-Forcing：少步自回归 + rollout 蒸馏 —— **实时**这条路打通
2026  WorldStream：把"规划"和"实现"拆开，中间留一份**记录**
2026  BoxWorld：把"规则"和"渲染"拆开，中间留一份**状态**
```

**它的历史位置**：如果把 [[2026-10-20-Ma — WorldStream Streaming Interactive 3D Worlds with Hierarchical Planning and Persistent State|WorldStream]] 看作"**决策/实现分离**"，BoxWorld 就是"**规则/渲染分离**"。两者都在回答同一个问题——**生成式系统里，"什么由代码决定"的边界画在哪**——但画的位置不同：WorldStream 把边界画在**几何**上（模型出语义规格，代码编译出地形），BoxWorld 把边界画在**状态**上（代码出对象状态，模型画出像素）。

## Previous Work

| 路线 | 代表 | 状态在哪 | 控制粒度 | 代价 |
|---|---|---|---|---|
| 隐式状态 | Genie / GameNGen / LingBot / Matrix-Game | **在生成器权重里** | 粗（动作/相机） | 规则不可指定 |
| 稠密状态 | 驾驶模拟器 / 游戏引擎 / 3D 缓存 | 在模拟器里 | 细 | 专、重、难从视频学 |
| 轨迹/布局条件 | 轨迹条件生成、BlobGEN-Vid | 在外部给定 | 中 | 多为**离线**、需整段条件 |
| **本文** | — | **对象级稀疏状态** | **中（对象级）** | 需要一个 coding agent |

关键分野是"**状态是否可由现成感知模型从视频里抽出来**"。如果状态必须由模拟器产生，生成器就只能在那几个模拟器的域里训练；如果状态能从视频里标注（框 + 属性 + 状态位），生成器就能吃开放域视频——**这是本文能同时做到"可控"和"通用"的唯一理由**。

## Core Idea

**把"发生了什么"写成代码，把"看起来怎样"交给模型；两者之间只传三个字段。**

| 常规做法 | 本文做法 | 换来什么 |
|---|---|---|
| 规则学进权重 | **规则由 coding agent 写成引擎代码** | 可读、可改、可审计；改规则不用重训 |
| 状态是稠密的（几何/物理场） | **状态是对象级的三个字段** | 能从视频里自动标注 ⇒ 能吃开放域数据 |
| 生成器同时"想"和"画" | 生成器只"画" | 模型容量全部用来保真，不用来推理规则 |
| 条件一次性给全（离线） | **状态缓冲异步流式喂** | 20.7 FPS 实时、状态更新立即生效 |

## Technical Approach

### 1) 引擎与接口

交互开始**之前**，coding agent 依据用户写的规则、从一个共享引擎模板生成 **Box World Engine**。交互开始后，引擎：

- 维护内部场景变量；
- 每个用户动作到来时，按规则判断"某个对象是否/何时改变状态"；
- 把结果**投影**成 $S_t$，写进一个 **state buffer**；
- 渲染器**异步**读这个 buffer。

**这个"异步 + 缓冲"的设计有个直接好处**：引擎可以跑得比渲染器快（也可以慢），二者解耦；状态更新体现在**下一批生成的帧**上，而引擎同时还在继续模拟。**这就是游戏服务器和渲染进程的关系，只是这里的"服务器"是几十行 Python。**

### 2) State Grounding Module：把状态路由到它该管的像素

每帧只有**它自己那个时刻**的状态是对的（对象在动、状态在变），所以状态必须**逐帧注入**，不能整段一次性条件化。

对第 $t$ 帧：

$$Y_t=\mathrm{Softmax}\!\left(\frac{(\tilde h_t W_Q)(E_t W_K)^\top}{\sqrt D}+\Phi(M_t)\right)E_t W_V \tag{1}$$

$$h'=h+\lambda\tanh(\gamma)\,\bar Y \tag{2}$$

其中 $E_t\in\mathbb R^{(N_tL)\times D}$ 是每个对象 $L$ 个 token 的接地表示，$M_t\in\{0,1\}^{HW\times(N_tL)}$ 是**由布局框生成的掩码**：**每个视觉 token 只能 attend 到覆盖它的那些对象**。$\Phi$ 把不允许的位置置 $-\infty$。$\gamma$ 是初始化为 0 的可学门，$\lambda$ 是控制尺度——**训练初期这一支完全关闭，不会破坏底座**。

模块插在 backbone 每一块的 **self-attention 与 cross-attention 之间**。时间一致性**不由它负责**，交给 backbone 的因果 self-attention。

**这个掩码是全文最重要的一个结构决定**：它把"对象身份"和"像素位置"绑在一起，而**不需要任何分割监督**。消融里去掉掩码，YouTube-VIS 的 mIOU 从 **0.572 掉到 0.291**——**掉了一半**。

### 3) 三段训练，每段解决一个不同的问题

| 阶段 | 数据 | 训什么 | 用哪招 | 为什么必须单独一段 |
|---|---|---|---|---|
| ① Masked Layout Grounding | 开放域视频（VideoGen-1M / 4DNeX） | 只训 SGM，冻结 backbone | **Diffusion Forcing**（每帧独立噪声） | 要让"接地"与时间上下文的噪声水平无关 |
| ② State Dynamics Adaptation | **游戏视频** | 学状态转移 | teacher forcing（干净历史 = 前一状态） | 要让模型"只在状态变化时改对象，其余继续" |
| ③ Rollout Distillation | **自生成的历史** | 恢复少步生成 | Self-Forcing 式分布匹配 | 推理时见到的是自己的历史，不是真视频 |

**第 ② 段的效果最能说明问题**：Suppress 从 **60.3 → 85.7**、Task 从 **49.4 → 64.4**。也就是说，**光有"接地"不够，还得学"什么时候不该动"**——而这恰好是端到端模型最缺的能力（它总是生成那个事件）。

第 ③ 段把 Paired-CF 从 **67.6 → 97.3**，且 VBench-Long 总分 **81.82 → 83.00**（**长程收益最大**）——**自生成历史上的蒸馏，本质是在修"误差累积"**。

### 4) 规格：一个 20.7 FPS 的实时系统

底座是 LongLive（从 Wan2.1 蒸馏出的少步自回归视频生成器），**保持同样的 20.7 FPS**，VBench Semantic 反而从 76.47 涨到 **79.54**（VBench-Long 75.82 → **77.67**）。**对象级状态不只是控制手段，它还是个更好的条件信号。**

## Key Contribution

1. **提出并验证了"规则/渲染分离"这个架构**，中间接口只有三个字段，且三个字段都能从视频里自动标注；
2. **PairedStateBench**：成对分支的评测协议——同一初始场景、两个分支要产生**不同**的指定后果，同时测"该发生的发生"（Trigger）与"不该发生的不发生"（Suppress）；
3. **三段训练各自的必要性被拆开证明**（见上表）；
4. **那条 oracle 对照**——把正确后果当文本喂给端到端模型，Paired-CF 仍是 16.2。

## Why It Works

**因为"规则"和"外观"是两种完全不同的函数。**

- 规则的输入是**离散状态 + 动作**，输出是**离散状态**，定义域小、可枚举、可写成 if/else，正确性可判定；
- 外观的输入是**状态 + 历史像素**，输出是**高维连续信号**，定义域巨大、只能靠先验。

把它们放进同一个网络，等于让一个函数同时做"离散逻辑"和"连续回归"——**前者要求精确，后者要求泛化，梯度会把前者拉向"平均而言最像"的答案**。这正是"模型倾向于生成那个事件，而不看前置条件"的机制解释。

**而 oracle 实验证明了这不是"信息不足"**：即使把正确答案告诉它，它仍然做不到 16.2% 以上——因为**它没有"条件分支"这个计算原语**，只有"下一个 token 的分布"。

## Limitations

- **判断"对不对"依赖规则的形式化程度**。引擎是由 coding agent 从模板生成的，**规则必须能写成状态机**；"这朵花看起来该不该开"这种模糊判断接不进去。
- **接口的空间分辨率是"框"**。作者自己承认：YouTube-VIS 上 mIOU **0.572 输给 BlobGEN-Vid 的 0.598**，因为 "blob representation offers finer spatial support than boxes"。**框是粗的。**
- **只在"对象级"成立**。流体、烟、破碎、布料这类**没有对象身份**的现象，进不了这个接口——它们被显式交给"生成式渲染"去补（原文说 leave "smoke and splashing water" to rendering）。**这是聪明的一刀，也是明确的边界。**
- **基准是自己提的**。PairedStateBench 的分数优势极大（97.3 vs 16.2），但这是新基准上的第一次测量，**没有第三方复现**；公开基准上（VBench / YouTube-VIS）它只是"有竞争力"，不是碾压。
- **依赖一个 coding agent 在交互前生成引擎**。这一步的失败率、延迟、可调试性都没报。**工程上这是最脆弱的一环。**

## Game Development Relevance

**这篇的架构，就是游戏引擎的架构——只不过把"渲染"换成了"生成"。**

| BoxWorld | 游戏引擎里的对应物 |
|---|---|
| Box World Engine（规则代码） | Gameplay / 逻辑层（Blueprint、状态机、GAS） |
| Object-Centric State Interface | **渲染线程拿到的可见性/实例数据** |
| Layout box | 屏幕空间包围盒（也是 GPU culling 的输入） |
| Persistent attribute | 材质/资产引用 |
| Dynamic status | 动画状态机 / 参数 |
| Box-State Renderer | 渲染器（这里是生成式的） |

**这条对应关系给出的判断是**：生成式渲染要进游戏，最可能的位置不是"替代渲染器"，而是"**接管那些逻辑已经算清楚、但美术成本太高的部分**"——远景、氛围、过场、NPC 的非关键动作。**因为逻辑层本来就在引擎里，不需要重新发明。**

对你的具体意义：**技能 VFX 的"规则"部分（什么时候播、播多久、命中判定在哪一帧）本来就在技能配置里**，不在 Niagara 里。BoxWorld 提示的分工是——**别指望特效系统自己去"理解"技能语义；把语义留在配置里，把像素交给生成**。

## Unreal Engine Relevance

- **Mass / MassEntity**：Object-Centric State Interface 的三个字段（transform 框、fragment 属性、状态位）与 Mass 的 fragment 模型几乎同构；**BoxWorld 相当于给 Mass 的输出接了一个生成式渲染器**。
- **State buffer 的异步读**：对应 UE 的"游戏线程写、渲染线程读"——只是 UE 用 proxy 做双缓冲，本文用一个未加锁的 state buffer。
- **不适用**：本文的渲染器是 DiT，UE 的渲染器是光栅化 + 光追。这套接口对 UE **有意义的只有"接口设计"，不是实现**。

## Technology Evolution

```text
"交互世界的状态放在哪"（= [[World Models for Games]] 的接口厚度轴）
2024  隐式：状态在生成器权重里          —— 通用、但只能粗控（Genie / GameNGen）
2025  稠密：状态在模拟器里              —— 精细、但专（驾驶/游戏引擎直出）
2025  图像对齐：状态是深度/几何         —— 薄、通用（UniWorld）
2026  语义记录：状态是一份可编译的记录   —— 最厚（WorldStream）
2026  ★ 对象级稀疏：状态是三个字段，**规则是可执行代码** —— 薄而可控（BoxWorld）
```

**注意这条轴不是一条直线**：WorldStream 往"厚"走（语义级规划），BoxWorld 往"薄"走（对象级字段）——但**两者都比隐式状态"厚"**，因为它们都把某样东西**从权重里搬出来了**。真正的方向是"**把不可审计的东西搬出网络**"，厚度只是副产品。

## Relationships

### Based On

- [[World Models for Games]] — 本文是该页"接口厚度"轴的**第五档**：不是最厚也不是最薄，而是**第一个"规则是代码"的样本**；
- [[Procedural Content Generation]] — "规则可执行"这条与 PCG 同源：**规则不是提示词，是程序**。

### Extends

- LongLive / Self-Forcing（少步自回归 + rollout 蒸馏）：本文给它们加了一个**逐帧状态注入通道**，且保持 20.7 FPS 不变；
- Diffusion Forcing：用它实现"接地与噪声水平无关"，这是第 ① 段能只训 SGM 的前提。

### Related

- [[2026-10-20-Ma — WorldStream Streaming Interactive 3D Worlds with Hierarchical Planning and Persistent State|WorldStream]] — **同一刀切在两个地方**：WorldStream 切在"决策/实现"之间（边界是**记录**），BoxWorld 切在"规则/渲染"之间（边界是**状态**）。两者的边界都小到可以手写、大到足以决定结果；
- [[2026-10-26-Zhang — UniWorld A Unified World Model for Real-Time Visual World Simulation|UniWorld]] — 接口是图像 + 深度（连续、稠密、无对象身份）；BoxWorld 是框 + 属性 + 状态位（离散、稀疏、有对象身份）。**这两种接口的差别不是精度，是"能不能表达 if"**；
- [[2026-10-26-Liu — WorldPager Zero-Stall Tiered Memory for Long-Horizon World Models|WorldPager]] — 那篇关心**记忆的容量**（KV 搬到盘上），这篇关心**记忆的语义**（对象状态只有几个标量）。**接口厚度与记忆厚度是两条独立的轴**：BoxWorld 的接口极薄，但它照样需要长时程记忆来保持画面一致（只是论文没做）。

## Personal Knowledge State

`Normal`。**结论层约 20 分钟**：抓住"规则是代码、状态是三个字段、oracle 也没用"这三条即可。

值得看的两处：

1. **Table 1 的 oracle 两行**——这是全文最有信息量的 6 个数字；
2. **图 2 的三联图**——它把"接口厚度"画成了一条从左到右的谱，比任何文字定义都清楚。

## Learning Value

- **对 TA**：把特效的"语义"和"表现"分开记账——**语义在配置里，表现才在 Niagara 里**。这一刀能省掉大量"让特效自己判条件"的奇怪需求。
- **对架构**：这是"**把不可审计的部分搬出网络**"的又一个样本（和 WorldStream 同族）。判据：**如果一个量需要被人改、被人查、被人复现，它就不该在权重里。**
- **对评审**：看到"我们的世界模型支持用户自定义规则"时，先问一句"**规则写在哪？**"——写在提示词里、写在训练数据里、还是写在代码里，是三件完全不同的事。

## Visualization

![[Research/Files/Pictures/Wu 2026 — 图2 三种交互世界的状态接口厚度.png]]
**读图**：三联从左到右是三种接口。(a) **隐式**：用户动作 → 视频生成器 → 下一帧，小字写着 `state hidden inside generator`，代价是 "Generalizable, but coarse control"；(b) **稠密**：动作 → 完整模拟器（游戏引擎/驾驶模拟器）→ **稠密场景状态** → 生成器，代价是 "Dense control, but domain-specific"；(c) **本文**：动作 → Box World Engine → **一个只有三格的接口**（Layout / Attributes / Status & events）→ 生成器。**看 (c) 中间那个黄框的宽度**——它比 (b) 的"稠密场景状态"窄得多，但比 (a) 的"什么都没有"宽。**这一张图就是"接口厚度"这个说法的定义。**（图 2，Wu et al. 2026）

![[Research/Files/Pictures/Wu 2026 — 图3 状态接地模块与掩码交叉注意力.png]]
**读图**：左边是**掩码交叉注意力**的矩阵——行是视觉 token（按对象分色：cat 红 / boat 蓝 / buoy 橙），列是对象 token，**每个 patch 只被允许 attend 到自己被覆盖的那个对象的那些 token**，其余全黑。右下角那句话是全文的结构核心：`Each patch attends only to its covering-box state condition`。右边是插入位置——**在 self-attention 与 cross-attention 之间**，即"先想清楚帧内关系，再接受状态条件，再接受文本条件"。去掉这个掩码，mIOU 从 0.572 掉到 0.291。（图 3，Wu et al. 2026）

## Notes

- **证据形态**：一手 PDF（arXiv 2610.36371v1，27 页，含 12 页附录），**未经同行评审**。所有数字取自 Table 1 / Table 2 / Table 3 与正文。
- **一处必须小心的读法**：PairedStateBench 是**本文自己提的**基准，且难度的三档（Easy/Medium/Hard）里 Easy 档 baseline 的 Suppress 是 `–`（无定义，因为该档没有"不该发生"的分支）。**所以 97.3 vs 16.2 这个比值应当读成"同一把尺子下的巨大差距"，不是"通用能力提升 6 倍"。**
- **"oracle 也没用"这条要限定范围**：oracle 是**把后果当文本**给模型，不是**把后果当条件**（比如给一张目标帧）。前者证明"没有条件分支原语"，后者没测。**这个区分很重要，别把结论扩大成"条件生成没用"。**
- 图 4–图 9（定性对比、多引擎 rollout）已看过但未入库；如需"同一交互在两个视觉世界里的渲染"作为迁移性证据，可补图 5。
