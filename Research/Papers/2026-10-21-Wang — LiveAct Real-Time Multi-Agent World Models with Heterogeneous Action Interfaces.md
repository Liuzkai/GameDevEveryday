---
type: paper
title: "LiveAct: Towards Real-Time Multi-Agent World Models with Heterogeneous Action Interfaces"
authors: ["Xinyu Wang", "Huafeng Shi", "Yue Ma", "Zijie Meng", "Teng Xi", "Yuzhuo Chen", "Yucheng Wang", "Ruicheng Zhang", "Guoxin Zhang", "Pengfei Wan", "Xiu Li", "Fangneng Zhan"]
year: 2026
published: 2026-10-21
venue: "arXiv preprint (cs.CV)，23 页（含附录）"
url: "https://arxiv.org/abs/2610.29408"
code: ""
project_page: "https://liveact-ai.github.io"
category: ["world-models", "video-generation", "game-ai", "multi-agent", "real-time", "distillation"]
importance: A-
historical_importance: 1
game_relevance: 4
production_readiness: "Research（512×448 @ 17.7 FPS，单卡研究原型）"
user_level: Normal
status: unread
---

# LiveAct — Real-Time Multi-Agent World Models with Heterogeneous Action Interfaces (2026)

## TL;DR

**世界模型此前的隐含假设是"一个玩家"。** LiveAct 把这个假设拆掉：**同一帧里多个被控实体，各自一路动作流，且这些动作流的语义可以完全不同**（格斗是"出拳/跳跃"的二进制按钮，赛车是"转向"的连续量，机器人是"夹爪开合"）。

三件事：

1. **接口才是瓶颈，不是视频骨干**。多智能体控制过去要么用**数值向量**（值精确、但语义隐含在"第 7 维是什么"里），要么用**语言**（语义清楚、但丢了同步数值）。LiveAct 把两者拼起来：**每个控制维度 = 一个"schema 词"（它是什么意思）+ 一个数值（它有多大）**；
2. **Agent-Aware Spatial Modulation**：用**语言描述**生成每个 agent 的空间门（spatial gate），把这一路动作只注入到"它该管的那个实体"上——**推理时不需要 mask、box、pose 或参考图**（这是与 WanToFight 一线最大的区别）；
3. **三阶段因果转移**：双向教师（50 步去噪）→ 教师强制 AR 调优 → **DMD 流式蒸馏**；最终 **512×448 下 17.70 FPS**（比 Stage 2 的 1.66 FPS 快 **10.7×**）。

**一句话对读者**：这是"**世界模型能不能接住第二个玩家**"这个问题的第一个完整答案——**多人游戏是世界模型进入游戏的必要条件，而不是加分项**。

![[Research/Files/Pictures/Wang 2026 — 图1 多智能体异构控制总览.png]]
**读图**：上半是"多玩家视频游戏"（四款竞技游戏，A1/A2/A3 三门颜色标注各自控制的角色），下半左是"多机器人协同"（同样的 A1/A2/A3，但控制是连续机械臂指令）。**底部绿框是流式 rollout**：首帧 + 两路手柄输入 → 连续生成，标注 **FPS=17.70**。**这张图想说的就一件事：同一套接口，既能吃二进制按钮，也能吃连续指令**。（图 1，Wang et al. 2026）

## Problem

交互式视频世界模型（Genie / Matrix-Game / LingBot-World 一线）已经能"按键生成下一帧"，但**几乎全部假设单一被控体**：把所有输入压成一路"玩家动作"或"相机控制"。

多人环境下这个假设直接失效，因为有两件事同时要成立：

| 挑战 | 表现 | 已有做法的失败方式 |
|---|---|---|
| **独立归因**（谁的动作） | 两路动作流如果只是拼在一起，交叉注意力会把它们**混在一起**——A1 的"右移"会推着 A2 也往右 | 靠推理时给的 mask / box / 身份裁剪："**用几何先验替模型做了归因**" |
| **异构语义**（这个数是什么） | 同一个张量第 7 维，在格斗里是"出拳"、在赛车里是"转向"、在机器人里是"夹爪" | 数值向量：值对、意思不对；语言：意思对、值没了 |

作者的判断很干脆：**短板在动作接口，不在视频骨干**。所以本文不动 diffusion backbone 的主体，只重做"动作怎么进模型"。

## Historical Context

```text
2023  Genie：从无标注视频学 latent action（单智能体、离散）
2024  GameNGen / DIAMOND：单玩家、单视角的游戏世界模型
2025  Matrix-Game / Magpie 一线：键盘鼠标级控制 + 长序列记忆（仍是单玩家）
2025  MultiWorld：多智能体，但用"同步多视角观测"喂模型 —— 评测时退化为固定视角
2026  WanToFight 一线：参考图引导，靠 identity crop 做归因（Attr-Acc 0.886，但要额外输入）
★2026 本文：把"归因"从**推理时输入**改成**模型内的空间门**；把"语义"从**任务专用维度布局**改成**每个维度一个 schema 词**
```

**这条线的核心变化**：早期世界模型解决"画得像"，中期解决"控得住"，现在解决"**同时控多个还不串**"。**每往前一步，靠的都是把某个外部先验搬进模型结构里**（这里是"agent 描述"的语言先验）。

## Previous Work

- **单智能体交互世界模型**：Genie（latent action）、GameNGen、DIAMOND、Matrix-Game 4.0（可执行 harness + 规则状态）；
- **多智能体**：MultiWorld（多视角同步，评测时退化）、ActionParty（逐 agent 数值控制）、Gamma-World（置换等变 agent 编码）；
- **语言化动作**：LingBot-World 2.0、Incantation（文本描述动作，语义通、数值丢）；
- **参考引导**：WanToFight 一线（identity crop + 参考图，归因强但要额外输入）。

## Core Idea

**把"动作"拆成两个正交的东西：它是什么意思（schema）与它有多大（value）；再把"归因"做成一个由语言生成的空间门。**

```text
每帧每个 agent i 的原始动作向量 u_i
   ↓ 逐维拆开
[ schema 文本 s_j : "steer right" ]  +  [ 数值 u_ij : 0.7 ]
   ↓ T5 编码 + MLP                         ↓
   语义 token h_sem                       仿射调制 h = u ⊙ (1+γ) + β
   ↓
   Agent-Aware Spatial Modulation：由该 agent 的**文字描述**生成空间门
   ↓（只对这一路动作做注意力，只更新属于它的视觉 token）
   X_out = X + Σ_i ΔX^i        （零初始化投影，初始不动预训练先验）
```

**两个设计的关键都在"不要外部输入"**：归因不再是推理时给的 mask，而是**从"这个 agent 长什么样/穿什么"的文字里推出来的空间门**；语义不再是"第 7 维在这个任务里是什么"，而是**跟着 schema 走的 token**。

## Technical Approach

### ① 异质动作接口（HAI）

- **Schema token**：每个控制维度配一句 schema 描述（"move left/right"、"jump"、"kick"、"gripper open/close"），过 T5 → 语义 token；
- **数值注入**：不是拼接，而是**仿射调制** $h_{i,j}=u_j\odot(1+\gamma_{i,j})+\beta_{i,j}$，其中 $(\gamma,\beta)$ 由 MLP 从数值算出。**语义决定"往哪调"，数值决定"调多少"**；
- **逐维 token**：一个动作向量拆成若干 token（不是压成一个），保证"第 7 维"始终对应"转向"这件事。

### ② Agent-Aware Spatial Modulation

- 由 agent 的文字描述（"穿黄绿衣服的战士"）生成 **agent embedding**，再分两路：**AgentGate**（空间门 λ^i，图里是热力图）与 **AgentDelta**（动作残差 Δ^i）；
- 视觉 token 的更新写成 $X^i=X+\lambda^i\odot\Delta^i$——**只在这一路该管的区域上生效**；
- 每个 agent 的注意力**只看自己那一路动作**（per-agent cross-attention），从结构上杜绝"A1 的动作污染 A2"。

### ③ 从双向教师到流式因果

| 阶段 | 做什么 | 结果 |
|---|---|---|
| Stage 1 | 双向 DiT 教师，21 个 latent 联合去噪、50 步 + CFG | 质量上限 |
| Stage 2 | 教师强制 + AR 调优（blockwise causal，4 latent/chunk，逐 chunk 刷 KV） | **1.66 FPS**（比教师还慢——大量小的串行前向，GPU 吃不满） |
| Stage 3 | **DMD 流式蒸馏**：每 chunk 4 步、无 CFG | **17.70 FPS**（10.7×） |

配套两个"对齐"细节：**causal temporal alignment**（原始动作对齐到压缩后的 latent 时间轴）+ **global latent offset**（动作窗口随缓存 rollout 一起前移）——否则流式生成里"按键"与"画面"会错开半拍。

![[Research/Files/Pictures/Wang 2026 — 图3 LiveAct三阶段架构.png]]
**读图**：右下 (c) 是 DMD 流式蒸馏（生成器 4 步 vs 假/真评分器）；左上 (a) 是主训练路径——**注意中间紫色框**：动作 token 与 schema token 在上方汇合，经 cross-attention 出来的是 $\Delta X^i$，再由左下橙色框的 **AgentGate（热力图 λ^i）** 门控后加回视觉 token。**"谁能动这块像素"是一个学出来的软门，不是画出来的框**——这是全文最值得记的结构。（图 3，Wang et al. 2026）

## Key Contribution

1. **HAI 接口**：语义（schema token）+ 数值（仿射调制）解耦，一套接口覆盖二进制按钮与连续机器人指令；
2. **无外部先验的归因**：Agent-Aware Spatial Modulation 把归因变成模型内的空间门，**推理时不需要 mask / box / pose / 参考图**；
3. **实时化**：三阶段因果转移 + DMD，512×448 **17.70 FPS**；在 6 个多智能体源、108 段留出片段上取得最好结果（FVD 338.5 / 270.2，游戏 Act-F1 0.594，机器人 NRMSE 0.558，归因准确率 **0.898 / 0.942**，随机为 0.5）。

## Why It Works

- **为什么"逐维 schema token"比"整帧一个动作 token"强**：动作的语义是**逐维**的（第 7 维是转向，第 12 维是刹车），压成一个 token 后模型必须自己学"这个向量的哪个分量对应什么"——那是**任务专用**的知识，换个游戏就废。逐维 + schema 把这个知识**外化**了；
- **为什么"门"比"框"更适合多智能体**：框是**几何**先验（要人给、要给对），门是**语义 → 空间**的映射（可以学）。代价是门必须靠"文字描述"来区分 agent——**描述写得含糊，门就糊**；
- **为什么 Stage 2 反而更慢**：因果注意力省的是"重复算历史"，但 Stage 2 每 chunk 仍跑 50 步 + CFG + 刷 KV，**大量小的串行前向把 GPU 利用率打崩**。Stage 3 把每 chunk 压到 4 步且去 CFG，才是真正的吞吐来源——**"因果"省的是计算量，"少步"省的才是时间**。

## Limitations

- **分辨率与吞吐**：512×448 @ 17.7 FPS，是"研究原型可交互"，不是"能上 1080p 的运行时"；
- **归因依赖文字描述**：agent 描述是输入的一部分，描述歧义（两个"红衣"）会直接伤 Attr-Acc；
- **没有 limitations 章节**（CV 论文常态，但读者要自己补）：FVD 338/270 距真实视频仍有明显差距；Act-F1 0.594 意味着**约四成的离散事件没被正确执行**；
- **指标是代理**：Act-F1 / NRMSE / Attr-Acc 全部由**在真实片段上训练的逆动力学模型**打分——它衡量的是"生成视频是否可被读回动作"，不等于"玩家觉得跟手"；
- **训练数据自建**：17,951 条轨迹 / 119.54 小时 / 4 款竞技游戏 + 机器人协同，未提开源。

## Game Development Relevance

- **多人是世界模型的入场券**：绝大多数商业游戏是多人/多单位同屏。**不能接住第二个被控体的世界模型，在游戏里只能当"过场生成器"**。LiveAct 把这个门槛显式化了；
- **"schema 化动作"对引擎侧的直接启发**：把技能/输入抽象成 **"语义 + 数值"两段**，是引擎里本来就有的思路（Input Action + Value），**世界模型需要的正是引擎已有的这层抽象**——这解释了为什么 MG4.0 那条"引擎管状态"的路线与本文能接上；
- **17.7 FPS @512×448 的定位**：这不是"替代渲染器"，是"**替代原型**"——关卡白模阶段的玩法验证、AI agent 的训练环境、过场预演。它的价值在**把"做出来看"的成本压到零**；
- **对你的预算表**：又一条"时间计价"的样本——**分辨率 × 帧率 × 智能体数** 三者互斥，且**智能体数增加的代价不是线性的**（注意力与门控都要按 agent 复制）。

## Unreal Engine Relevance

- **无直接 UE 路径**（研究原型、无代码）。但有两个可借鉴的工程点：
  1. **语义 + 数值的输入抽象**：UE 的 Enhanced Input（Action + Value）几乎就是 HAI 的工程形态，做 AI 驱动或录制回放时可以直接复用这层；
  2. **"门"而非"框"**：如果要在引擎里做"多角色注意力/影响范围"，**用角色标签生成软权重**比手画范围框更可维护。
- **不要期待**：它不会变成 UE 的渲染替代；真正的落点是**离线内容生成 / agent 训练环境**。

## Technology Evolution

```text
单智能体 latent action（Genie 2023）
  ↓  加控制（键盘鼠标级）
单玩家可玩世界模型（GameNGen / Matrix-Game 2024–2025）
  ↓  加状态（引擎管规则，MG4.0 2026）
可执行、可长时、可实时
  ↓  加"第二个玩家"（本文）
多智能体 + 异构接口 + 实时（512×448 @ 17.7 FPS）
  ↓  下一步（可预期）
更高分辨率 / 更强归因（不依赖文字描述）/ 与引擎状态的正式耦合
```

## Relationships

### Based On

- **DiT / 双向视频扩散教师**：Stage 1 的质量来源
- **DMD（Distribution Matching Distillation）**：把多步教师蒸馏成 4 步流式生成器
- **T5 文本编码**：schema 与 agent 描述的语义来源

### Extends

- [[2026-10-21-Zhang — Matrix-Game 4.0 Harnessing Video Models for Real-Time Playable Game Worlds|Matrix-Game 4.0]]：**同一问题的下一个约束**——MG4.0 解决"规则由谁执行"，本文解决"多个玩家怎么不串"。两者合起来才是"可玩的世界模型"：**状态可读 + 多体可归因**
- **MultiWorld / Gamma-World**：把"多智能体"从"多视角同步观测"推进到"多路异构动作"

### Related

- [[World Models for Games]]：本文把"多智能体/多玩家"这条维度显式加进了世界模型的评价表
- [[2026-10-19-InfiniDream — Open-Ended Photorealistic 3D Worlds Through Procedural Generation|InfiniDream]]：**相反的分工**——InfiniDream 用程序化保证可编辑与确定性，本文用生成保证画面；**一个要"可改"，一个要"可玩"**

### Followed By

- 更高分辨率 / 更长 rollout；把 agent 描述从"文字"换成"引擎里的身份 ID"（更可靠）

## Personal Knowledge State

`Normal`（推断，结论层 ≈20 分钟）：只需读 TL;DR + 图 3 的结构 + 主表数字，**不需要碰 DMD 推导**。

- **Easy**：为什么单玩家假设不够、为什么数值向量与语言各有缺陷——工程直觉即可；
- **Normal**：schema token + 仿射调制的分工、"门 vs 框"的取舍、Stage 2 为什么反而慢；
- **Hard**：DMD 蒸馏的分布匹配目标、blockwise causal 的 KV 管理——**不必读**。

## Learning Value

1. **"接口即瓶颈"是一个可迁移的判断**：很多系统卡住不是因为模型不够强，而是因为**输入的表达方式把某类信息扔掉了**（这里是"语义"或"数值"二选一）。遇到"模型学不会"时，先问**输入里有没有这个东西**；
2. **归因从"外部标注"变成"内部结构"**：mask/box 是人替模型做的决定，门是模型自己学的决定。**凡是需要人在推理时给先验的系统，都值得问一句"这个先验能不能学"**；
3. **实时化的两笔账要分开算**：因果注意力省**计算量**，少步蒸馏省**时间**——Stage 2 比教师还慢就是最好的反例。**"结构更省"不等于"跑得更快"**。

## Visualization

![[Research/Files/Pictures/Wang 2026 — 图5 与五个基线定性对比.png]]
**读图**：同一初始帧、同一动作流，六列分别是 GT、五个基线、本文。前两行是游戏（格斗/竞技），后两行是机器人协同。**看两件事**：① 语言序列化方法（LingBot-World 2.0 / Incantation）在机器人行**画面直接漂掉**（语义通了，数值没通）；② ActionParty 画面稳但**动作不跟随**（数值通了，语义没通）。**这一列对比就是"语义与数值必须同时在场"的实证**。（图 5，Wang et al. 2026）

## Notes

**主表要点（Tab. 2）**

| 方法 | 游戏 FVD ↓ | 游戏 Act-F1 ↑ | 机器人 Attr-Acc ↑ | 备注 |
|---|---|---|---|---|
| LingBot-World 2.0 | **1256.6**（最好） | 0.146（很弱） | — | 画质好、几乎不跟动作 |
| MultiWorld（单视角） | 2661.4 | 0.163 | 0.548 | 归因接近随机 |
| Incantation | 1989.5 | 0.162 | — | 机器人画面漂移严重 |
| ActionParty | 1672.5 | — | — | 机器人重建最好、动作跟随最弱 |
| WanToFight-style（复现） | — | — | 0.886（**用了 identity crop**） | 最强基线，但依赖外部输入 |
| **LiveAct** | **338.5 / 270.2** | **0.594** | **0.898 / 0.942** | 不用 mask/box/pose/参考图 |

**诚实点**

- **"最好"要看清口径**：WanToFight 在机器人上 0.886 是**带身份裁剪**的成绩，本文 0.942 是**不带**的成绩——这才是可比的那一对；
- **Act-F1 0.594** 意味着**四成离散事件没执行对**。"实时多智能体"目前是"能看、能大致跟"，不是"能玩"。
