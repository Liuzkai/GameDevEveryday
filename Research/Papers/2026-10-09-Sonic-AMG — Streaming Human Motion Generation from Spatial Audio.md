---
type: paper
title: "Sonic-AMG: Streaming Human Motion Generation from Spatial Audio"
authors: [Jingcheng Ni, Zekun Li, Zhiyang Dou, Xiaoyan Cong, Ziyao Zeng, Hongyang Du, Randall Balestriero, Taku Komura, Srinath Sridhar]
year: 2026
published: "2026-10-09（arXiv v1, 2610.13300；cs.CV + cs.GR + cs.LG + cs.RO + cs.SD）"
venue: "arXiv Preprint（Brown University × MIT × Yale University × The University of Hong Kong；项目页 jingchengni.com/sonic-amg/ 已开放，代码与模型待接收后放出；未公告先见：API 通道 2026-10-12 捕获）"
url: "https://arxiv.org/abs/2610.13300"
code: ""
project_page: "https://jingchengni.com/sonic-amg/"
category: [motion-generation, audio-driven-motion, streaming, character-animation, causal-transformer, flow-matching]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: "Research（实时在线生成已演示；代码/模型待放出；SAM benchmark SOTA FID / R-precision）"
user_level: "Normal（结论层：what/where 分工 + rehearsal training + spatial feedback 三件事）"
status: unread
aliases: [Sonic-AMG, 空间音频驱动动作, 流式动作生成, 流式人体运动生成]
tags: [motion-generation, audio-driven, streaming, character-animation]
---

# Sonic-AMG: Streaming Human Motion Generation from Spatial Audio（Ni et al. 2026）

> **入库 2026-10-12（Run 34）。** **Brown University × MIT × Yale × The University of Hong Kong**（Srinath Sridhar 组与 Taku Komura、Randall Balestriero 等；单位经项目页逐人核对）。经 **API"未公告先见"通道**捕获。
> **一句话定位**：**首个"从空间音频流式生成人体动作"的自回归框架**——把"听到了什么（what）"和"从哪儿来（where）"拆成两组 token；用因果 Transformer + flow head 逐个生成动作块；两个训练机制（rehearsal training / spatial feedback）消除"训练用真值历史、推理用生成历史"的错位。
> **库内位置**：音频驱动动画线第三节点（[[2026-09-17-EMODY Flow — Emotion-Aware Audio-Driven Full-Body Motion Generation|EMODY Flow]] 情感 → [[2026-09-18-GestureFAR — Streaming Co-Speech Gesture Generation with Flow Autoregression|GestureFAR]] 流式手势（avatar/video 侧）→ **本篇：流式全身动作 × 空间音频**）；"分解再分工"家族的新样本（**what / where 分工**）；"训练-推理一致性"主题（rehearsal 修 exposure bias；spatial feedback 让条件跟着生成走）。

## TL;DR

**问题一句话**："Prior work generates motion offline from complete acoustic and spatial input sequences. **Humans, however, react to sounds without knowing what they will hear next.**"——人要的是**在线反应**，而此前的工作全是"离线、拿到完整音轨后才生成"。

```text
Sonic-AMG（全因果 / 流式）：

输入流：空间音频（双耳）+ 已知的世界坐标声源位置 + 已生成动作历史
   │
   ├─ what：声学内容 token（听到什么）
   └─ where：body-relative 声源几何 token（相对身体从哪儿来）
   │
   ├─ causal transformer + flow head → 连续动作 latent（下一个动作块）
   ├─ causal decoder → 动作段（扩入动作流，继续滚动）
   └─ spatial feedback：用刚生成的 root pose 重算 body-relative 声源位置
        （让 where 条件与"生成出来的身体"保持一致——不只跟真值一致）

两个训练机制（补上"流式推理"的两个坑）：
① rehearsal training —— 训练时不仅做全序列 teacher-forcing，还在"短自回归
   self-rollout 产生的历史"上做监督（训练姿势 = 推理姿势）
② spatial feedback —— 在 rehearsal 训练与推理中，都用【生成】的 root pose
   重算声源相对位置（条件与生成自洽）
```

**结果**：SAM benchmark 上 FID 与 R-precision **state-of-the-art**；支持**实时在线生成**——在连续的声音事件与声源位置变化中生成连贯反应。代码与模型待接收后开放。

## Problem

1. **流式设定缺失**：空间音频下的动作生成（"对声音的全身反应"）此前全是**离线**——需要完整声学+空间序列。而真实交互（游戏 NPC、机器人、VR avatar）里，声音是**边听边来**的；
2. **空间信息复杂**："understands both **what** is heard and **where** it comes from"——动作既取决于声音内容（什么声音），也取决于声源相对身体的位置（从哪个方向来）；
3. **流式推理的两个固有错位**（本文真正的技术贡献所在）：
   - **历史错位**：teacher-forcing 用真值动作历史，流式推理只有自己生成的历史（**exposure bias / 误差累积**的传统问题，在长时动作流上更严重）；
   - **空间条件错位**：训练时 body-relative 声源位置由**真值**身体位姿算；推理时身体是生成的——若不修正，条件描述的是"真值的身体"而不是"正在动的身体"。

## Core Idea

**"what / where 分开 + 两条自洽回路"**：

- **分开**：声学内容与身体相对声源几何，**作为两组独立 token 编码**（内容学内容、几何算几何——where 不需要网络去"学"，它是可计算的量）；
- **两条回路**（本库"反馈修正"家族的新样本）：
  ```text
  ① 历史回路：rehearsal training —— "让训练见过的历史，等于推理会产生的那种历史"
       （全序列 TF 监督 + 短自回归 self-rollout 上的监督，混合配比）
  ② 空间回路：spatial feedback —— "让条件描述的身体，等于生成出来的那种身体"
       （rehearsal 与推理阶段都用【生成】root pose 重算 body-relative 声源位置）
  ```

**架构**：causal transformer（自回归）+ **flow head**（连续 motion latent 的生成头）+ causal decoder（还原动作段）——与 [[2026-09-18-GestureFAR — Streaming Co-Speech Gesture Generation with Flow Autoregression|GestureFAR]] 的"flow 自回归"同族，但对象从手势/画像升级到**全身动作 + 空间声源**。

## Technical Approach（按摘要与项目页可核部分）

- **编码**：acoustic content（双耳音频）与 body-relative source geometry（已知世界坐标 + 身体位姿 → 相对几何）为两组 token；
- **生成**：causal transformer 条件于两组 token + 动作历史 → flow head 出连续动作 latent → causal decoder 出动作段 → 扩流（extend the motion stream）；
- **rehearsal training**：full-sequence teacher-forced 监督 + **短自回归 self-rollout（自滚）历史上的监督**——把"推理时自己产生的历史"提前放进训练；
- **spatial feedback**：rehearsal 与推理**都用生成 root pose 重算 body-relative 声源位置**；
- **评测**：SAM（空间音频→动作 benchmark）上 FID、R-precision 双 SOTA；演示**实时在线**连贯反应（跨声音事件与声源位置变化）。

## Key Contribution

- **首个流式（自回归）空间音频→人体动作框架**——把"离线条件生成"改成"在线反应"；
- **两个错位问题的显式拆解与修法**：历史错位 → rehearsal；条件错位 → spatial feedback。**两者都是"让训练世界与推理世界对齐"**（本库已有家族：[[2026-10-08-Tabula Rasa — Monte Carlo estimation of unit-variance noise with controlled spatio-temporal correlation|Tabula Rasa]] 的"接口契约"、PDB 的"算一次×每帧"背后的 train/infer 拆账）——**本篇给出"条件本身也要随生成滚动"的样本**；
- **what / where 分工**：可计算的物理量（相对几何）从"需要学习的表示"里拿出来，与内容分开承载。

## Why It Works

- **自回归流的误差累积**（长时动作流上尤其致命）被"训练时先体验自己的历史"压制——rehearsal 让模型见过"带噪声的历史→下一步"的映射；
- **空间条件是闭环的**：声源相对位置依赖身体——身体生成后不更新条件，等于用"错误的自我参照"生成下一步；**spatial feedback 把这一环补上**，避免"身体与空间条件各说各话"。

## Limitations（以现有信息可判的部分）

- 需要**已知的世界坐标声源位置**（游戏引擎里天然有；真实录音场景需要声源定位前置）；
- 实时性指标（延迟、帧率）与模型规模的细节待代码/论文放出；
- SAM 之外的泛化（多角色、遮挡声、混响）未展开；
- 与"文本条件动作生成"（[[2026-09-15-UniMo — Unifying Human and Animal Motion Generation|UniMo]] 等）的组合未涉及。

## Game Development Relevance

- **NPC / 角色对 3D 音效的实时反应**：引擎里的枪声、脚步、载具位置天然是"空间音频事件 + 世界坐标"——**恰好是 Sonic-AMG 的输入形态**；"流式"意味着不需要预生成、可随声音实时滚动；
- **与"音频驱动"游戏技术的对照**：当前游戏的角色音频反应多为状态机/单帧动画触发；本篇给出"连续动作生成"形态（未来 NPC 反应层的一个候选）；
- **where 的分工给引擎架构的启示**：把"可计算的几何量"从网络条件里外置（引擎里就是把 transfrom/声源位置直接喂进去，不需要"学"）——与用户熟悉的"引擎擅长什么、模型别抢"分工原则一致。

## Unreal Engine Relevance

- 可想象路径：MetaHuman / Control Rig 角色 + 生成式反应层（**实验性**）；输入对接 UE 的音频空间化（Audio Spatialization / MetaSounds）与声源 transform；产物对接 AnimSequence/Control Rig 的流式播放。**生产上更现实的近程形态 = 离线蒸馏为有限的反应动画库**（对应"神经 → 传统资产形态翻译"家族：[[2026-10-01-GALA — Gaussian Blendshape Distillation for Real-Time Avatars|GALA]]）。

## Technology Evolution

```text
音频驱动角色动画线（本库账本）：
  2026-09 EMODY Flow：情感感知的全身动作（离线、音频条件）
  2026-09 GestureFAR：流式共语手势（auto-regressive flow；avatar/video 侧）
  2026-10 ★ Sonic-AMG（本篇）：流式全身动作 × 空间音频（what/where 分工 +
    rehearsal + spatial feedback；实时在线反应）
       ↓
  演化主线一：离线 → 流式（"不等完整输入"）
  演化主线二：单条件（音频）→ 空间条件（声源位置）
  演化主线三：训练-推理对齐成为显式设计环节（rehearsal / feedback）
```

## Relationships

### Based On

- 自回归生成 + **flow matching 动作表示**（[[2026-09-18-GestureFAR — Streaming Co-Speech Gesture Generation with Flow Autoregression|GestureFAR]] 同族的 flow 自回归工具链）；因果 Transformer。

### Extends

- **音频驱动动作生成**：从"离线 + 单一音频流"扩展到"**流式 + 空间音频（what+where）**"；
- **exposure bias 修法谱系**：teacher-forcing 修补（scheduled sampling 一族的近代变体 = rehearsal on self-rollouts）。

### Related

- [[2026-09-17-EMODY Flow — Emotion-Aware Audio-Driven Full-Body Motion Generation|EMODY Flow]]——同一问题域（音频→全身），离线侧样本；
- [[2026-09-18-GestureFAR — Streaming Co-Speech Gesture Generation with Flow Autoregression|GestureFAR]]——流式侧先驱（手势/画像），**"冻结-蒸馏"配方的第 3 例出处**；
- **"条件随生成滚动"家族**：本篇 spatial feedback + 动作历史自洽（可与"数字双胞胎"式的闭环条件对照）。

### Followed By

- 待观察：code/model 放出；SpatialAudio→Motion 任务在游戏引擎内的实用延迟；Robotics 侧应用（项目页有独立栏目）。

## Personal Knowledge State

- Normal（结论层 25 分钟：what/where 分工 + 两个错位 + 两种修法；不需要进入自回归/flow 训练细节）。

## Learning Value

- **一句话背下来**："**流式生成的两个错位——历史错位用'训练时先自滚'修，条件错位用'条件跟着生成走'修**"——评估任何流式/在线 AI 系统时的通用检查单；
- **"可计算量外置"**：where 是几何计算，不是学习内容——"引擎/物理擅长什么，就别让网络去猜"（与 [[2026-10-03-Neuroll — Real-Time Neural Strand-Based Hair Simulation via Simulator-in-the-Loop Unrolling|Neuroll]] 的"镜像 I/O"、DDGI 的"光追补充光栅化"同族）。

## Visualization

（本日无独立图解——机制以正文 TL;DR 的 text 图为准）

## Notes

- 单位核对来源：项目页（Brown × MIT × Yale × HKU 逐人核对——Brown: Ni/Z. Li/Cong/Du/Balestriero/Sridhar；MIT: Dou；Yale: Zeng；HKU: Komura）；
- **未公告先见记录**：API 通道 2026-10-12 捕获（提交 10-09，listing 不可见）。
