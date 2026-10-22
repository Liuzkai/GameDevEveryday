---
type: paper
title: "Matrix-Game 4.0: Harnessing Video Models for Real-Time Playable Game Worlds"
authors: ["Jihai Zhang", "Runjia Qian", "Kaiqin Kong", "Biao Jiang", "Puyi Wang", "Siru Chen", "Tianle Li", "Guanjie Chen", "Junyi Wu", "Chubin Chen", "Yuanhang Li", "Kaichen Huang", "Haobo Zhang", "Yangguang Li", "Wei Yu", "Guibin Chen", "Yu Cheng", "Yahui Zhou"]
year: 2026
published: 2026-10-20
venue: "arXiv preprint（Technical Report，Skywork AI 等）"
url: "https://arxiv.org/abs/2610.27999"
code: "已开源（代码 + 模型权重 + low-poly 游戏资产，启动即放出；见项目页）"
project_page: "https://matrix-game-v4-0.github.io/page/"
category: ["world-models", "generative-rendering", "game-engine-architecture", "real-time", "video-generation", "interactive"]
importance: A
historical_importance: 3
game_relevance: 5
production_readiness: "Prototype（单张 RTX 5090 上 480p/60FPS 可玩；每个游戏训练一个渲染模型；代码/权重/资产全开源；作者称下一步做 Steam 技术预览）"
user_level: Normal
status: unread
---

# Matrix-Game 4.0: Harnessing Video Models for Real-Time Playable Game Worlds

## TL;DR

**把"世界模型"从"能看的视频"推到"能玩的游戏"，靠的不是更强的视频模型，而是把游戏执行搬出模型。** 系统拆成三个角色：**编码 agent 写规则（scriptwriter/translator）→ 轻量引擎管状态、执行规则、输出低模代理（referee）→ 视频模型只负责把代理画成照片级画面（painter）**。世界状态永远在引擎里，模型不再需要"记住"世界——它只需要画。

于是延迟从"秒级"掉到"毫秒级"：**480×864、60 FPS、按键到出画 112 ms，单张 RTX 5090（32 GB）**；同表对比：Magpie 1.55 s（1×H100）、AgentGarten 878 ms（1×H100）、StreamRender-H3 517 ms（8×GB200）、Code World Model ≈10.2 s（1×H200）——**只有本文跑在消费级单卡上，且代码/权重/资产全部开源**。

> 一句话对读者：**它不是"游戏引擎的替代品"，而是"渲染器的替代品"**——被换掉的是"把三角形画成像素"这一步，不是"世界怎么运转"这一步。

![[Research/Files/Pictures/Zhang 2026 — 图1 系统总览与延迟对比.png]]
**读图**：一张图把全篇讲完。上半是**系统环**（Player 输入 → 编码 agent 改规则 → 引擎保状态 → 代理+字幕 → 视频模型 1–2 步出画）；中段是**留出集上的斜切对照**（左半低模代理，右半生成画面，构图与运动必须对齐）；下段是**用自然语言改玩法**的三例（加护盾耐久、加检查点罚时、把计时赛改成追逐）；底部是**延迟榜**（对数轴）与 **5090 上的生成速率 83/57 FPS**。（图 1，Zhang et al. 2026）

## Problem

纯视频世界模型（Genie 系、GameNGen、Oasis、Matrix-Game 2.0/3.0）在"短时探索"里很惊艳，但要长成"能玩很久的游戏"，有四笔账同时到期：

1. **没有持续的外部纠错**——世界状态只存在于生成的像素与模型上下文里；视野外的区域没有任何显式状态在维护，"回头看一眼"就开始漂；
2. **误差自回归累积**——没有外部校正，小误差逐块复利；即便挂 3D 工具，相机位姿估计误差也会累积到场景崩塌；
3. **按帧计的记忆不压缩**——历史帧越堆越多，记忆成本随 rollout 长度线性增长（[[2026-09-21-WorldCrafter — Consistent Video World Model with Implicit 3D-aware Memory|WorldCrafter]] 那一代已经在用"记忆预算"缓解，但仍是模型内部的事）；
4. **玩法机制单薄**——交互基本停在"走、看、推一下"，没有可执行规则、没有持久后果、没有明确目标。

作者还补了一刀更根本的论证：**扩散模型学的是统计相关，视觉真实不等于因果/物理理解**。按 Jakobson 的三分法，文/图生视频本质是"语际翻译"，而**当前视频生成器的语义智能大量来自上游 MLLM/文本编码器，扩散部分更像一个渲染器**。既然如此，就让渲染器只做渲染。

## Historical Context

```text
2018  GameGAN：把 DOOM 当成一个可学习的渲染器（像素里塞世界）
2024  Genie：从无标注视频学隐动作空间 + 自回归动力学
2024  Genie 2 / 2025 Genie 3：可玩 3D 环境；Genie 3 做到 720p/24FPS 实时
2024  GameNGen：单 TPU 上 20 FPS 跑 DOOM——"扩散模型当游戏引擎"的极限样本
2025  Oasis / Hunyuan-GameCraft / Yan / WorldPlay / LingBot-World：更强控制、更长时序、1080p
2026  Matrix-Game 2.0 / 3.0：少步自回归 + 长时记忆，720p 实时
      ↓ 分叉：把"规则"从模型里拿出来
2026  Code World Model：编码 agent 写代码维护 world state，代理视频条件化渲染器
2026  Programmable World Model / Code Video Model / StreamRender-H3 / CoDeR / Magpie / Mora / EdenLive
      ↓ 合流：本文
2026  Matrix-Game 4.0：引擎管规则 + agent 写规则 + 视频模型管画面；112 ms 端到端，单卡 60FPS
```

**这条分叉不是本文首创**——Magpie、Code World Model 都先走了一步（本库 [[World Models for Games]] 概念页把它们归为"层次 2"）。本文的历史位置是**第一个把这套分工做成"可玩、可测、开源、跑在消费卡上"的完整系统**，并且把"分工"写成了一份 roadmap。

## Previous Work

- **交互式视频世界模型**：Genie 1/2/3、GameNGen、DIAMOND、Oasis、Hunyuan-GameCraft、Yan、WorldPlay、LingBot-World、Matrix-Game 2.0/3.0——**网络同时承担世界动力学与画面**；
- **代码定义世界 + 视频模型渲染**：Code World Model（编码 agent 写状态代码）、Streaming Code World Model（4 步 DMD，17 帧块 5.1 s / H200）、Programmable World Model（agent 写实体状态与转移规则，轻引擎维护离屏实体，ControlNet 吃编译出的 3D 包围盒）、Code Video Model（写 Three.js，冻结 MiniMax-H3 只在前几步条件化）、StreamRender-H3（语义图驱动两步流式渲染器）、AgentGarten（模拟器 + 共享神经渲染器训 agent）、CoDeR、Magpie（引擎解算 + 渲染服务分离）、Mora、EdenLive；
- **结构条件化渲染**：Enhancing Photorealism（G-buffer → 照片）、vid2vid / World-Consistent vid2vid、ControlNet / VideoComposer、Cosmos-Transfer1、DiffusionRenderer / FrameDiffuser、AlayaRenderer-Flash（832×448、31.5 FPS / H200）、SolarRenderer（任意结构输入，4 步）；
- **实时自回归生成**：Diffusion Forcing / Rolling Diffusion、StreamDiT、DMD、CausVid、Self Forcing / Self Forcing++、Causal Forcing / Causal Forcing++（本文蒸馏的直接依据）。

**本文相对 Magpie 的增量**：Magpie 是"引擎解算 + 渲染服务"，但**规则由人写、没有 agent、没有开源、延迟 1.55 s**；本文把"改规则"也交给了 agent，并把延迟压到 112 ms。

## Core Idea

**把"世界如何推演"与"世界如何呈现"彻底切开，切到模型连"记住世界"都不需要。**

三个角色各管一件事，接口只有两条：

| 角色 | 职责 | 输入 | 输出 |
|---|---|---|---|
| **编码 agent**（scriptwriter + translator） | 把玩家想法写成游戏代码；把引擎状态翻译成文字描述 | 玩家自然语言 / 引擎状态与事件 | 游戏代码 + 动态字幕 |
| **引擎**（referee） | 维护世界状态、执行规则、渲染低模代理 | 玩家按键 / agent 写的规则 | 低模代理帧 + 位姿/动作/事件元数据 |
| **视频模型**（painter） | 把代理帧画成照片级画面 | 代理帧 + 动态字幕 + 首帧 + 空间记忆帧 | 高清帧 |

关键判断：**视频模型要学的函数，从"世界的动力学"缩小成了"外观的补全"**。这是一个**条件更充分、目标更窄、可蒸馏**的函数——所以 1–2 步去噪就够，所以能跑到 60 FPS。

## Technical Approach

### 1) 数据：像素对齐的"代理–实拍"配对（UE5）

- 同一个场景在 UE5 里**渲两遍**：一遍高清 RGB，一遍低模代理（**同一相机、同一帧号，逐像素对齐**）；外加深度图与逐帧元数据；
- 代理的**样式约定**很硬：天空纯黑、地面灰色棋盘、建筑是纯色盒子/棱柱、树是"圆柱+球"、角色是方块人，**每个图元黑描边、无贴图无反射无阴影**——目的是让代理只携带**空间拓扑 + 刚体位姿 + 运动轨迹**，不携带任何外观线索（否则模型会去抄代理的外观）；
- 规模：**114,334 段 / 1,712 小时 / 1280×720**，四个场景（中世纪城堡、古埃及沙漠、沙漠峡谷空战、CitySample 大都市）；留出集按**整场采集批次**切，杜绝同场景泄漏。

### 2) 语义：动态字幕，且**不用 VLM**

每一帧的字幕由**规则模板**从引擎状态生成：可见物体 ID → 查语义描述表 → 填模板（"第三人称驾驶视角，自车是现代四门轿车…右前方约 4 m 停着一辆小型厢式车…有深肤色女性行人位于左侧约 32 m 处站立"），并单列 **Changes**（新进入视野 / 状态变化）。

![[Research/Files/Pictures/Zhang 2026 — 图4 动态字幕由引擎状态生成.png]]
**读图**：三帧代理（Frame 4 / 53 / 155）与其**同帧字幕**并排。橙色 = 状态与事件（"parked / standing still / walking and getting closer"），蓝色 = 位置与距离（"about 4 m away"）；**Changes** 单列新进入视野的物体。整段文字是模板从引擎状态填出来的，**没有任何 VLM 参与**——这正是"世界状态在引擎里"的直接红利：画面里有什么，引擎本来就知道。（图 4，Zhang et al. 2026）

**为什么这条设计重要**：字幕是**确定性的、可复现的、零推理成本**的语义通道。用 VLM 描述画面会引入幻觉与延迟，而引擎**本来就知道**画面里有什么——这正是"世界状态在引擎里"的直接红利。代理管"结构"（位置/布局/运动），字幕管"语义"（是什么/什么状态），两者互补。

### 3) 渲染器：5B DiT + 三层 KV 缓存

![[Research/Files/Pictures/Zhang 2026 — 图5 渲染器架构与三层 KV 缓存.png]]

- 从 **Wan2.2-Fun-5B Control** 改成 video-to-video：输入 148 通道 = 48 噪声潜变量 + **48 代理控制潜变量** + 52 零通道；1 个 latent = 4 帧；480×864 下 **405 token / latent 帧**；
- 30 个 causal DiT block，宽 3072、FFN 14336、24 头 ×128、3D RoPE 按时/高/宽分 44/42/42 维；文本用 umT5-xxl，**只在字幕变化时重算 text KV**；
- **KV 缓存三部分**：3 帧空间记忆（相机位姿相近的历史帧）+ 参考帧 + **最近 20 个 chunk**；每生成完一个 chunk，**再用 t=0 无控制条件重跑一遍**，把干净 latent 的 KV 写进缓存——保证后续 chunk 看到的是"干净历史"而不是自己的噪声。

![[Research/Files/Pictures/Zhang 2026 — 图7 四十秒长序列代理与生成对照.png]]
**读图**：三个游戏各 40 秒（961 帧 @24fps）的留出 rollout，上排代理、下排生成、四个时间点。要看的是**构图与运动全程跟着代理走**（房子、树、车的位置一致），而材质、植被、光照、枪口火光全是模型补的——且角色外观从第一块到最后一块不变。（图 7，Zhang et al. 2026）

### 4) 训练：SFT → 三阶段因果蒸馏

- **SFT**：每个游戏训一个模型；81 帧窗口 @704×1280；64 卡、batch 64、4,000 步、lr 1e-5；首帧/prompt/代理控制分别以 0.3/0.2/0.2 概率丢弃（支持 CFG 与无首帧生成）；
- **阶段 1 教师强制自回归**：每个 latent 帧独立噪声等级，只能看它之前的干净帧，20,000 步；
- **阶段 2 因果一致性蒸馏**：48 个噪声等级上，教师走一步 Euler（CFG=3），学生匹配 EMA 学生在下一等级上的预测，5,000 步 → **4 步出块**；
- **阶段 3 非对称 DMD**：真分数 = 冻结的双向模型（CFG 3），假分数在线更新；学生每 5 步更新一次；**按部署形态 roll out 21/41/61/121 帧**，且 **80% 的步喂 TAEHV 编码的控制**（部署时就是这么编码的）→ **2 步出块（首块 4 步）**。

### 5) 推理：八招，4.5 → 83.4 FPS

![[Research/Files/Pictures/Zhang 2026 — 图6 推理优化四招与逐核耗时.png]]
**读图**：左侧 (a) 是**逐核账本**——单个 block 在 5090 上的 GPU 时间按 kernel 类别拆开：1 步时 70.5 ms（6.5k 个 kernel）→ 40.1 ms（2.9k 个）；2 步时 100.6 → 58.1 ms。右侧三招是系统侧的"搬数据"优化：(b) 把解码提到 clean-KV 更新之前（帧更早出屏）；(c) KV 窗口用**环形缓冲**（原地覆盖最旧槽位，而不是每步整体搬移）；(d) 自注意力切成 7 段窗口并行算 + LSE 合并（**只有变化的那段重新量化**）。（图 6，Zhang et al. 2026）

| 累加项 | FPS | 延迟 | 倍率 |
|---|---|---|---|
| 基线（4 步 + 全量 Wan VAE） | 4.5 | 1110 ms | — |
| + 轻量 VAE（TAEHV） | 6.5 | 867 ms | 1.4× |
| + RoPE kernel、缓存 text KV | 15.4 | 551 ms | 2.4× |
| + 8-bit 权重 + CUDA Graph | 24.2 | 466 ms | 1.6× |
| + 1 步蒸馏 | 51.2 | 376 ms | 2.1× |
| + 异步 I/O | 57.7 | 367 ms | 1.1× |
| + Triton 融合 kernel | 65.8 | 121 ms | 1.1× |
| + 环形 KV 缓存 | 73.7 | 117 ms | 1.1× |
| + split-KV FlashAttention | **83.4** | **112 ms** | 1.1× |

**这张表是本篇对性能工种最有用的一页**：19× 吞吐、9.9× 延迟不是靠一个魔法，而是**九次"各拿一点"**，且**每一步都标注了它换的是什么**（显存带宽、kernel 启动开销、搬运、量化误差）。注意后三招 FPS 只涨 1.1×，但**延迟从 367 → 112 ms（3.3×）**——**吞吐与延迟是两本账**，短 chunk 场景下后者才是手感。

### 6) 硬件档位

| 卡 | 显存 | 单步吞吐 | 显存占用 |
|---|---|---|---|
| RTX 5090 | 32 GB | 83 FPS | 18.2 GiB |
| RTX 5080 | 16 GB | 67 FPS | 12.8 GiB |
| RTX 5070 Ti | 16 GB | 58 FPS | 10.9 GiB |

（固定 prompt 测吞吐；2 步时 5090 掉到约 57 FPS，显存不变。）

## Key Contribution

1. **提出并实现"可执行 harness"分工**：`code → 规则 / engine → 状态 / video model → 画面`，把世界状态从模型里彻底拿出来；
2. **第一个跑在单张消费卡上的可玩系统**：480p/60FPS、端到端 112 ms，且开源代码 + 权重 + low-poly 资产；
3. **一套可复用的数据–训练–运行时管线**：UE5 像素对齐配对采集 + 规则化动态字幕 + 三阶段因果蒸馏 + 九项推理优化；
4. **把"改玩法"也纳入闭环**：agent 写代码改规则（护盾耐久、检查点罚时、追逐模式），引擎执行，模型只画。

## Why It Works

- **纠错免费**：世界状态每帧由引擎重算，误差不累积；模型只对"当前这一帧的外观"负责，历史由 KV 缓存提供"视觉一致性"，而不是"物理正确性"；
- **代理是强条件**：像素对齐的低模代理把**布局、位姿、遮挡关系**直接喂到输入端（通道拼接，不是 cross-attn），模型不需要"猜几何"；
- **目标函数变窄**：从"预测世界下一帧"变成"给已知结构补外观"——条件充分、歧义小，所以 1–2 步去噪能达到可接受质量（对比：纯视频世界模型必须多步才能保住物理合理性）；
- **字幕是零成本语义**：引擎知道自己有什么，不需要 VLM 看图说话；
- **蒸馏与部署对齐**：阶段 3 用部署时的 TAEHV 编码、部署时的 roll out 长度训练——**训练分布 = 推理分布**，这是少步模型不崩的前提。

## Limitations

- **480p**：对比系统多在 768p；作者把"高分辨率精修阶段"列为未来工作；
- **每个游戏一个模型**：没有跨游戏泛化，换游戏要重新采集 + SFT + 蒸馏；数据采集仍依赖 UE5 手工场景；
- **玩法深度有限**：演示是射击/驾驶两类；"agent 写规则"的天花板 = 编码 agent 的能力，没有规则正确性验证环节（写错了就是错的游戏）；
- **不解决"生成即作弊"**：模型画出来的东西可能与引擎状态不一致（视觉上像打中了，规则上没中）——**规则以引擎为准**，这既是优点也是体验风险；
- **延迟口径**：112 ms 是"按键→第一帧反映该按键"；60 FPS 游戏循环下相当于 6–7 帧的输入延迟，手感等级接近云游戏而非本地原生；
- **成本**：单卡 18.2 GiB + 5B 参数模型常驻，对同时跑游戏逻辑的机器是实打实的预算占用。

## Game Development Relevance

**对读者（VFX/性能预算）最直接的三条：**

1. **"渲染预算"变成"推理预算"，且预算单位换了**：不再是 draw call / 粒子数 / overdraw，而是**token 数 × 去噪步数 × 输出分辨率 × KV 窗口长度**。本文的九项优化表就是一份"新预算维度"的清单——每一项都在某个维度上做减法（VAE 换成 TAEHV = 解码预算；1 步蒸馏 = 步数预算；环形 KV = 历史预算；split-KV = 注意力预算）；
2. **"档位"语言可以直接迁移**：1 步 83 FPS / 2 步 57 FPS、480p / 768p、5090 / 5080 / 5070Ti——**这三个旋钮就是生成式渲染的 S/A/B/C**，而且**画质与延迟的兑换率是可测的**（本文给了表）；
3. **对 VFX 的定位判断**：这类系统里**没有粒子系统、没有材质图、没有后处理链**——特效是"模型顺手画出来的像素"。短期内它替代不了技能 VFX 的可控性与可预算性（你无法给"模型画出来的火花"定 particle 上限）；它的现实用途是**原型期卖相**与**2D/轻量玩法**，不是 3A 战斗特效管线。

**对项目管理的意义**：如果这套分工成立，"关卡/玩法迭代"与"美术迭代"可以真正并行——规则改动不进美术流程，美术改动不进代码流程。这与 Magpie 的"白模先行"是同一个赌注，本文把它推到了"可玩 + 可开源"。

## Unreal Engine Relevance

- **数据管线直接长在 UE5 上**：CitySample + 城堡/沙漠场景，双通道采集（RGB + 低模代理 + 深度 + 元数据），**这套采集脚本本身就是"用 UE 造训练数据"的模板**；
- **低模代理 = 引擎侧的"结构渲染通道"**：可以用 `SceneCapture` 的自定义 Show Flags（纯色材质、关阴影/后处理、黑背景）+ 自定义 depth 输出复现；
- **对引擎团队的信号**：如果未来引擎要"接一个神经渲染后端"，需要的正是本文那三样输出——**代理帧 / 位姿与事件元数据 / 语义描述**；这三样现在都不是引擎的标准导出物；
- **Niagara 视角**：与本篇无接口（模型不吃粒子数据）；但"引擎出结构、模型补外观"的分工可以类比为"Niagara 出运动矢量、材质层补外观"——**分层负责**是同一思路。

## Technology Evolution

```text
"世界状态放哪"的四代答案：
2018-2023  放像素里      GameGAN / 早期视频预测        → 世界=生成的画面
2024-2025  放模型隐状态  Genie / GameNGen / Oasis      → 世界=上下文 + KV
2026       放外部记忆    WorldCrafter 类             → 世界=可检索的隐式记忆（记忆当预算）
2026       放引擎/代码    Magpie → Code World Model → ★ Matrix-Game 4.0  → 世界=可执行程序
```

**每一代都在回答同一个问题：模型需要"记住"多少世界？** 答案从"全部"退到"零"——**退得越狠，模型越轻、越快、越可纠错**。这条轴与 [[Global Illumination]] 那条"冻结哪个变量"的轴是同构的：**都是把复杂度从最难的地方搬走**。

## Relationships

### Based On

- **Wan2.2-Fun-5B Control**（渲染骨干）+ **TAEHV**（轻量 VAE）+ **Causal Forcing++**（因果蒸馏）+ **非对称 DMD**（分布匹配）——四块现成组件，本文的组合方式是新的；
- [[World Models for Games]] 的"层次 2"路线（引擎管规则、模型管画面）。

### Extends

- [[Magpie — Real-Time World Renderer for Interactive Games|Magpie]]：同一个分工，但把"改规则"也交给 agent，且延迟从 1.55 s 压到 112 ms、全开源；
- Code World Model / StreamRender-H3 一系：把"代码定义世界"从 demo 推到可玩系统。

### Related

- [[DLSS 5 — Generative Neural Rendering|DLSS 5]]：同为"生成模型只管外观"，但 DLSS 5 在**已渲染帧**上做增强（层次 3），本文替换整条光栅化；
- [[2026-10-19-Wan-Alpha++ — Animating a Single RGBA Asset into Controllable Transparent Video Effects|Wan-Alpha++]]：同为"素材/画面侧生成"，但一个进运行时（本文），一个只做素材；
- **SolarRenderer**（同日 arXiv 2610.28630，记名未入库）：**同一天两篇独立工作撞在同一句话上**——"引擎给结构，模型补外观"（它做的是 4 步、12.3× 加速的 reference renderer）。

### Followed By

- 待观察：多人共享世界状态（作者列为未来工作）、更高分辨率精修阶段、Steam 技术预览。

## Personal Knowledge State

- `user_level: Normal`（系统层 ≈25 分钟可读完；渲染器内部 ≈45 分钟）
- **机制层（Easy）**：游戏循环、状态机、渲染与逻辑分离——这是你的日常工作；
- **增量层（Normal）**：**"延迟账本"的拆法**（九项优化各换什么）+ **生成式渲染的预算维度**（token × 步数 × 分辨率 × KV 长度）；
- **不需要**：DiT 内部结构、RoPE 维度分配、DMD 的数学——知道"它是把 4 步蒸馏成 2 步、把 VAE 换成 TAEHV"即可。

## Learning Value

1. **一份现成的"性能优化拆解模板"**：4.5 → 83.4 FPS 的九行表，每一行都写清"改了什么、换到多少、倍率多少"——这是你写预算/优化报告可以直接照抄的体裁；
2. **两个容易混的账被分开**：**吞吐（FPS）与延迟（ms）**——后三招吞吐只涨 1.1×，延迟却掉 3.3×；做手感评估时别只看 FPS；
3. **"档位 = 三个正交旋钮"**：步数 / 分辨率 / 显卡档——比"低中高"三档更能解释"为什么这一档长这样"；
4. **一个反面校准**：这类系统**不产生可预算的特效**。看到"生成式游戏"时先问：**它的特效可控吗？能定上限吗？** 不能，就不该进你的 VFX 预算表。

## Visualization

- 图 1 系统总览与延迟对比（已嵌入 TL;DR）
- 图 2 三角色分工（已嵌入 Core Idea 前文）
- 图 5 渲染器架构与三层 KV 缓存（已嵌入）
- 图 6 推理优化四招与逐核耗时（已嵌入）
- 图 7 四十秒长序列代理与生成对照（已嵌入）

![[Research/Files/Pictures/Zhang 2026 — 图2 三角色分工引擎管规则模型管画面.png]]
**读图**：三角色的接口全在这张图上——agent 与引擎**双向**通信（向下写代码/场景，向上把状态翻译成文字）；引擎向模型只送**结构渲染 + 控制信号**；模型回给玩家**画面**。注意 ④ 那一路（控制信号）是"单向"的：**模型无法回写世界状态**，这是整个架构的安全边界。（图 2，Zhang et al. 2026）

## Notes

- 论文自述的定位很克制：**"对研究者是可复用的技术底座，对开发者是未来工作流的一次预览"**——它没有宣称"游戏引擎要死了"；
- 一个值得记住的措辞：**"code keeps the rules, the video model draws every frame"**（图 1 标语）——这句话本身就是这套架构的定义；
- 数据规模对照：1,712 小时 / 11.4 万段，**全部来自 UE5 合成场景**——这也意味着它的"真实感"上限由 UE5 资产决定；
- 与 2026-09 的 [[Magpie — Real-Time World Renderer for Interactive Games|Magpie]] 相隔约两个月，延迟降了一个数量级（1.55 s → 112 ms），**这条曲线的斜率值得每季度复核一次**。
