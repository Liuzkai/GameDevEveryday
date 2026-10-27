---
type: paper
title: "Past as State, Present as Attention: Persistent-State Blockwise Flow Matching for Long-Horizon Co-Speech Motion Generation"
authors: ["Yi Liu", "Xiangyue Zhang", "Jianfang Li", "Jia Ma", "Yisheng He", "Jianqiang Ren"]
year: 2026
published: 2026-10-23
venue: "NeurIPS 2026（同济大学 / 东京大学 / 阿里巴巴）"
url: "https://arxiv.org/abs/2610.32373"
code: "未放出（项目页 ly403.github.io/PASPA）"
project_page: "https://ly403.github.io/PASPA/"
category: ["motion-generation", "co-speech", "long-horizon", "state-space", "flow-matching", "animation"]
importance: A-
historical_importance: 2
game_relevance: 3
production_readiness: "Research（120 s 动作 ≈ 11.4 s 推理、峰值 0.36 GB——离线烘焙可用，非运行时）"
user_level: Normal
status: unread
---

# Past as State, Present as Attention

## TL;DR

**长时程生成漂移的根因不是"历史太短"，是"历史和现在用了同一种表示"。** 论文先量了一件事：在 BEAT2 上算自相关，**风格特征的相关性在 100 帧之后仍然显著，局部运动特征在 20–50 帧内就衰减到 0**（图 3）。既然两者的时间尺度差一个量级，就不该用同一种机制处理。

于是原则一句话：**过去压成状态（past-as-state），现在用全注意力（present-as-attention）**。

- **过去**：块级 Gated DeltaNet（BGDN），把每个块压进一个**固定大小**的状态 $S\in\mathbb{R}^{d_v\times d_k}$；关键是**读只能读块首状态** $S_0^{[n]}$——从数学上保证块内不会偷看未来（block causality 由构造给出，不靠 mask 猜）；
- **现在**：块内双向注意力（BMHSA，块对角 mask），复杂度从 $O((NK)^2d)$ 降到 $O(NK^2d)$。

结果：BEAT2 上 FGD **3.71**（GlobalDiff 4.78）/ 全说话人 **2.33**（2.63）；SHOW 上 FGD **0.00111**，比 GlobalDiff **低 43.9%**。长时程更关键：60 s rollout 的最后 10 s，短种子法 FGD 掉到 **6.721**、token 全历史法 **5.771**、PASPA **5.745**——**开头三种方法几乎一样（4.45 左右），差别全在漂移速度上**。

> 一句话对读者：**最狠的一条证据在第 3 张消融表——"给现在也用状态"（7.652）比"完全不给历史"（7.065）还差。单位用错，不如不用。**

![[Research/Files/Pictures/Liu 2026 — 图1 长时程共同言语动作的风格漂移对照.png]]
**读图**：同一段语音、同一个说话人、52 秒，GT 的风格是"双手抬在胸前"。三行三种历史传播方式，四个时间点（0–2s / 10–12s / 31–33s / 51–52s）。(a) 短种子：开头对，手越走越低（**单调漂移**）；(b) 全历史 token 注意力：历史全留着，但风格**时对时错，与时间距离无关**；(c) PASPA：52 秒里手一直在胸前。**红圈是手部特写**——这是这类论文里少见的"把失败模式画清楚"的图。（图 1，Liu et al. 2026）

## Problem

共同言语动作（co-speech motion）要生成长序列，主流做法是**逐段生成**，段间靠历史传播。两种历史策略各有病：

| 策略 | 做法 | 病 |
|---|---|---|
| **短种子截断** | 只用上一段最后几帧 | 历史太薄 → **风格单调漂移**（图 1a） |
| **全历史 token 注意力** | 把所有历史帧当 token 做全局注意力 | 成本随长度增长；**数据有限时学不动这么长的历史** → 风格"时对时错"（图 1b） |

论文的关键提问不是"怎么留更多历史"，而是：**"有没有比'注意力扫过原始历史序列'更有效的表示与传播过去的方式？"**

## Historical Context

```text
2002  Motion Graphs：把连续运动切成图上的路径（长动作 = 短片段的拼接）
2020  Learned Motion Matching：把"检索"换成"前馈"，历史压进网络隐状态
2024  Diffusion Forcing / Block Diffusion：逐 token 噪声 + 块内去噪、块间自回归
2025  Self Forcing / 流式动作生成：训练历史与推理历史对齐
2026  GlobalDiff：块级 flow matching 做共同言语动作（本文的直接基线，FGD 4.78）
2026  Motion Mamba：把状态空间模型搬进动作生成（只做长上下文，没做"过去/现在分工"）
2026  PASPA：把"过去/现在的时间尺度不对称"显式写进架构
```

它的位置：**不是"更强的长上下文模型"，而是"承认两种时间尺度、给两种表示"**——这与本库 [[Skinning]] 的四代谱系是同一类判断（不同环节该用不同表示，而不是一个表示打天下）。

## Previous Work

- **共同言语动作**：CaMN / DSG / EMAGE / DiffSHEG / MambaTalk / HoloGest / RAG-GESTURE / SemTalk / GestureLSM / GlobalDiff；VQ-VAE 码本路线（多样性受限）vs 扩散/流匹配路线（当前主流）；
- **长上下文序列建模**：注意力（$O(L^2)$、数据饥渴）vs 状态空间 / 线性注意力（$O(L)$、更强时间衰减归纳偏置）；**Gated DeltaNet** 的门控与 delta 规则更新提供"选择性遗忘 + 定向写入"，正是"风格慢慢变、噪声快速丢"想要的偏置；
- **块式扩散**：Block Diffusion（块内去噪 + 块间自回归）、Diffusion Forcing（每 token 独立噪声等级）。

## Core Idea

**先量时间尺度，再定表示。**

![[Research/Files/Pictures/Liu 2026 — 图3 风格特征与局部运动特征的自相关.png]]
**读图**：BEAT2 上按说话人平均的自相关 $r(\tau)$，横轴是滞后帧数（30 fps，300 帧 = 10 秒）。蓝线（风格特征）在 100 帧后仍保持 0.3–0.5 的相关；橙虚线（局部运动特征）在 **20 帧内**就掉到 0 并转负。内嵌小图是另一组特征（Alt），走势相同。**这张图就是全文的立论**：两个量的相关长度差一个数量级，把它们放进同一个注意力里，模型只能学到折中。（图 3，Liu et al. 2026）

![[Research/Files/Pictures/Liu 2026 — 图2 三种历史传播方式对照.png]]
**读图**：三种"历史怎么进当前块"的接线图。(a) 短种子：只递一小段，其余丢弃（`drop`），风格漂移；(b) 全历史：所有历史 token 与当前块做双向注意力——表达力最强但代价随长度涨，且数据少时学不动；(c) PASPA：历史先汇入一个 **Persistent State**（固定大小），当前块内部仍做双向注意力。**注意 (c) 的箭头方向**：状态只喂给"生成"，当前块内部不读状态——这是"分工"的图形化表达。（图 2，Liu et al. 2026）

## Technical Approach

### 1) BGDN：块级 Gated DeltaNet，读只能读块首

状态更新沿用 GDN：

$$S_i=S_{i-1}\big(\alpha_i(I-\beta_i k_ik_i^\top)\big)+\beta_i v_ik_i^\top$$

$\alpha_i$ 自适应衰减无关波动，$\beta_i$ 控制 delta 规则的定向写入。

**唯一的改动，也是最关键的改动**：把"动态读"改成"**块首读**"：

$$\text{GDN}: o_i=S_iq_i \qquad\text{BGDN}: o_i=S_0^{[n]}q_i\quad\forall\,(n-1)K<i\le nK$$

块内所有 query 只读**块起始状态**（= 上一块的最终状态）。这带来两个后果：

- **数学上保证块因果**：块内没有 token 能读到同块更早 token 写入的状态，所以不会"偷看未来"；
- **读退化成一次矩阵乘**：$O^{[n]}_{\text{BGDN}}=Q^{[n]}(S_0^{[n]})^\top$，可直接吃 Tensor Core（作者写了 Triton kernel，基于 FLA）。

### 2) BMHSA：块内双向，块间绝对禁止

$$O=\text{Softmax}\!\left(\frac{QK^\top}{\sqrt d}+B_{\text{blk}}\right)V,\qquad B_{\text{blk},ij}=\begin{cases}0&\lceil i/K\rceil=\lceil j/K\rceil\\-\infty&\text{otherwise}\end{cases}$$

实现上不物化 $B$：把 $X\in\mathbb{R}^{NK\times d}$ reshape 成 $\mathbb{R}^{N\times K\times d}$，**把块索引当 batch** 跑标准双向注意力——复杂度从 $O((NK)^2d)$ 降到 $O(NK^2d)$。

### 3) 训练：多块监督 + 弱单调时间步

- 每个块有自己的时间步 $t_n$，一次前向并行去噪整条序列（教师强制喂干净历史）；
- 但**必须约束 $t_1\le t_2\le\cdots\le t_N$**，否则 FGD 反而变差（表 4：单块 7.410 → 直接多块 8.456，**变糟**；加单调 6.346；再加自适应权重 5.614；+stop-gradient 5.216；+logit-normal 时间采样 4.450）。**这条"多块监督不是免费的"值得单独记**：更密的监督若时间步无序，等于给模型看不可能出现的噪声剖面。

### 4) 推理：prefill-decode + 混合 CFG

- **prefill** 只编码干净块更新状态（跳过去噪子层）；**decode** 把状态当只读条件，迭代去噪当前块；
- 状态是**唯一的持久缓存**，固定大小 → **每步查询 $O(1)$，与历史长度无关**（对比线性增长的 KV cache）；
- **Hybrid CFG**：四次预测（无条件 / 只条件 / 只历史 / 全条件）做多线性组合，把"历史依赖强度"变成可调旋钮；只在中间噪声区间 $t\in[0.1,0.6]$ 生效。代价：延迟 +4.9–9.1%，显存 +0.01 GB。

## Key Contribution

| # | 贡献 | 可迁移性 |
|---|---|---|
| 1 | **命名了"过去/现在不对称"并用自相关实测** | 高——任何长序列生成都可以先量一下"哪些量慢、哪些量快" |
| 2 | **块首读**（block-start read）保证块因果 | 高——把"不许偷看未来"从 mask 约定变成代数性质 |
| 3 | 固定大小状态 ⇒ $O(1)$ 查询 | 中高——与 KV cache 的账完全不同 |
| 4 | 多块监督的两个稳定器（单调时间步 / 自适应权重） | 中——块式训练的通用配方 |

## Why It Works

- **单位匹配时间尺度**：慢变量（风格/幅度/空间偏好）只需要"几个持久的成分"，token 级注意力会把容量浪费在瞬时波动上；快变量（速度/精细关节角）必须在块内做稠密交互。
- **状态是"选择性遗忘"**：$\alpha$ 衰减 + $\beta$ 定向写入，正好是"风格慢慢累积、抖动快速丢弃"的算子形式——**归纳偏置与目标同构**。
- **块首读同时解决两件事**：因果性（正确）与并行性（快）。这两件在朴素 RNN 里是矛盾的。

## Limitations

**论文自陈/实验可见**：

1. **只在共同言语动作上验证**（BEAT2 / SHOW），没有全身通用动作、没有物体交互、没有游戏动作（跑跳打）；
2. **风格的定义是隐含的**：论文用"特征自相关"论证风格是慢变量，但没有给出可解释的风格参数，也不能像 [[Motion Style Slider — Continuous Style Control for Human Motion Diffusion|风格滑杆]] 那样连续控制；
3. **状态大小是超参**（$d_v\times d_k$），容量上限固定——极长序列里"风格演变"（例如角色从紧张到放松）会被状态抹平，因为状态只保留"持续成分"；
4. **数据依赖**：多块监督 + 单调时间步的收益建立在 BEAT2 这种规模上，换到更小的数据集是否仍成立未知。

**我补的两条**：

5. **它优化的是"不漂移"，不是"更生动"**：盲测 39.8% top-1 是四选一（随机 25%），提升实在但不压倒；BC（beat consistency）在部分设置下并非最优（0.746 vs HoloGest 0.795）——**节奏对齐让位给了风格稳定**；
6. **"实时"不成立但"可烘焙"成立**：120 s 动作 ≈ 11.4 s（无 CFG）/ 12.4 s（有 CFG），峰值显存 0.36–0.48 GB。**这是一个能进离线管线的成本结构，不是运行时**。

## Game Development Relevance

- **对 VFX / 角色最有用的是那条"慢变量 vs 快变量"判据**，可直接搬去审预算表：

| 你的预算维度 | 它更像"风格"还是"局部运动" | 含义 |
|---|---|---|
| VFX 时长 / 整体强度 | 慢变量 | 可以按"技能级"给一个状态（你的 SABC 就是这个状态） |
| 粒子发射曲线 / 爆发峰值 | 快变量 | **必须全分辨率**，压成统计量一定失真 |
| 动态灯光数 | 慢变量（场景级） | 可缓存、可滞后更新（DDGI 的 $\alpha$ 就是同一手法） |
| Overdraw 峰值 | 快变量 | 平均值无意义，必须看峰值帧 |

- **"用错单位比不用更糟"是这条线最强的一句**：如果哪天有人提议"用平均粒子数代替峰值"或"用平均帧时间做预算"，这篇的 7.652 vs 7.065 就是反例；
- **0.4 GB / 11 s 的离线成本**意味着：NPC 手势、过场动作、对话表演可以做成"每次对话现生成"，但**不能每帧生成**。

## Unreal Engine Relevance

- **可落点**：`Control Rig` 之上的**离线动作生成**（对话/演出），或作为 `Motion Matching` 数据库的**扩充器**（生成候选片段再入库）；
- **不可落点**：运行时逐帧生成（延迟量级不对）；
- **可借的机制**：**块首读**这种"只读块起始状态"的写法，与 UE 动画图里的 `State Alias` / 状态机过渡是同一类结构——**"块"就是状态机里的状态节点，"块首读"就是"转移只发生在边界"**。

## Technology Evolution

```text
"历史怎么带进下一步"
2002  Motion Graphs：历史 = 图上的位置（离散、精确、不可泛化）
2020  Learned Motion Matching：历史 = 网络隐状态（连续、可泛化、不可解释）
2024  Diffusion Forcing：历史 = 带噪的 token 序列（可并行、成本随长度涨）
2026  MG4.0 / UniWorld：历史 = 外部引擎里的状态（最可靠、需要接口）
2026  PASPA：历史 = 固定大小的持久状态 + 块内注意力（O(1) 查询）
      ↑ 四条路的账：精确 / 泛化 / 并行 / 成本，没有一条全占
```

## Relationships

### Based On

- **Gated DeltaNet**（门控 + delta 规则 + chunkwise 并行形式）——状态更新的全部数学来自它，本文只改"读"；
- **Block Diffusion / 半自回归分解**：$p(x\mid a,e)=\prod_n p(x_n\mid x_{<n},a_n,e_n)$；
- **GlobalDiff**（本文的直接骨架与基线）。

### Extends

- **Motion Mamba 一类"用状态空间做长动作"的工作**：从"用 SSM 换掉注意力"推进到"**只用 SSM 管过去、注意力仍管现在**"——这是一个更精确的取舍，而不是替换。

### Related

- [[Motion Generation]] —— 本篇归入其"长时程一致性"支线；
- [[Motion Tokenization]] —— **本文是对"一切皆 token"的一次明确反对**：过去不该被 token 化保留，该被压成状态；
- [[2026-09-21-WorldCrafter — Consistent Video World Model with Implicit 3D-aware Memory|WorldCrafter]] / [[2026-10-26-Zhang — UniWorld A Unified World Model for Real-Time Visual World Simulation|UniWorld]] —— 同一个月里，"历史"被处理成三种账：**token 预算**（WorldCrafter）/ **逐比特潜变量**（UniWorld）/ **固定状态**（PASPA）；
- [[Motion Matching]] —— **检索 vs 状态的对照**：MM 的历史是"数据库里的位置"（精确但需存储），PASPA 的历史是"一个矩阵"（有损但 O(1)）；
- [[Budget Units]] —— "状态大小与时长解耦"是内存维度的换单位。

### Followed By

- 待观察：状态容量与"风格演变"的取舍、能否扩展到全身游戏动作、代码放出。

## Personal Knowledge State

**Normal（结论层）**。四条：

1. 先量时间尺度（自相关），再决定"压成状态还是留全分辨率"；
2. **块首读**：把因果性从"mask 约定"变成"代数性质"；
3. **多块监督需要单调时间步**，否则更密的监督更差（8.456 vs 7.410）；
4. **用错单位比不用更糟**（State+State 7.652 > 无历史 7.065）。

## Learning Value

- 对你最直接的用处不是动作生成，而是**"慢变量/快变量"这张判据表**（见上）——它能把你的五维预算分成"可按技能级给状态"和"必须按帧给"两类；
- 次之：**"读只能读块首"** 这个技巧可以迁移到任何"分块 + 有状态"的系统（例如分帧计算的粒子模拟：块内不许读同块中间状态，否则结果依赖调度顺序）。

## Visualization

![[Research/Files/Pictures/Liu 2026 — 图1 长时程共同言语动作的风格漂移对照.png]]
（图 1，Liu et al. 2026 —— 52 秒漂移对照；读图见文首）

![[Research/Files/Pictures/Liu 2026 — 图3 风格特征与局部运动特征的自相关.png]]
（图 3，Liu et al. 2026 —— 时间尺度实测；读图见"Core Idea"）

## Notes

- **数据**：BEAT2（官方划分，Scott 单人子集 + 全说话人）、SHOW（按 DiffSHEG 协议）；动作 = SMPL-X 55 关节 rot6d（333 维）+ 全局平移，表情 = 100 维 FLAME 系数；音频用 WavLM 特征 + 冻结的 audio-to-expression 网络。
- **消融全表（表 3，FGD 越低越好）**：

| 过去建模 | 现在建模 | FGD | 读法 |
|---|---|---|---|
| 无 | 注意力 | 7.065 | 不给历史 |
| 注意力（8 帧） | 注意力 | 6.051 | 短种子 |
| 注意力（全历史） | 注意力 | 5.710 | token 全历史 |
| 状态 | 状态 | **7.652** | **用错单位，比不给更差** |
| 状态 | 注意力 | **5.216** | PASPA |

- **长时程（表 6，终端 10 s 窗口的 FGD）**：$[0,10]$ s 三种方法 4.448 / 4.448 / 4.731；$[20,30]$ s 5.119 / 5.388 / 5.886；$[50,60]$ s 5.745 / 5.771 / 6.721。**短种子法的崩溃是"加速"的，状态法几乎是平的**——这才是"持久状态"的真正卖点（不是更低，是**更平**）。
- **效率（表 7）**：120 s = 3,600 帧 → 11,411 ms（无 CFG）/ 12,439 ms（有）；300 s → 28.6 s / 30.0 s；600 s → 56.0 s / 61.1 s。**延迟随长度近似线性**（不是常数），因为块数线性增长；常数的是**每步的状态查询**与**显存**。
