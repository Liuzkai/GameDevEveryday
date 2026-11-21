---
type: concept
user_level: Hard
---

# Physics-based Character Animation

## Definition

用物理仿真（刚体动力学 + 接触）驱动的角色动画范式：动画不是播放出来的，而是控制策略在物理世界里"演"出来的。与运动学方法（[[Motion Matching]] 检索、[[Motion Generation]] 生成）的根本区别：动作天然满足动力学约束（平衡、接触、碰撞），不需要后处理修穿模。

## Core Principle

```text
参考动作数据
      ↓
模仿学习目标（DeepMimic 式 RL）
      ↓
物理仿真环境中的控制策略
      ↓
动力学可行的动作输出
```

关键难点从来不在"效果好不好"，而在**训练成本**与**控制策略的泛化**。

## Prerequisites

- [[Motion Matching]]（运动学对照系，先理解检索式为什么不需要物理）
- 刚体动力学基础（Easy 侧工程直觉可覆盖）
- RL 基础词汇（当前缺口）

## Evolution

- DeepMimic（2018，模仿学习范式奠基）
- ASE / AMP 系列（对抗式运动先验，NVIDIA/Jungdam Won 线）
- [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]]（2026，训练成本压到秒级）
- ★ [[2026-09-17-DSD — Diffusion Skill Discovery]]（2026-09-18 入库，Xue Bin Peng 线）——**补上另一半：技能库的"宽度"**
    - 前人（DIAYN/ASE）用互信息学技能库，但**边缘状态熵那一项在高维控制里算不动**，近似的结果是技能堆在初始位置附近；
    - DSD 用扩散模型（score matching）**直接估计熵梯度**，技能库明显变宽（三数据集 FID 最低、Div-R 最高；后空翻 5 个种子里学成 4 个 vs ASE 0 个）；
    - 两阶段规律：**先解决"学得多快"（InstantMimic/Sprinting），再解决"学得多宽"（DSD）**。

## Related Concepts

- [[Neural Animation]]（神经运动学路线，平行阵营）
- [[Motion Generation]]
- [[Neural Physics Simulation]]（更一般的学习式物理，反向问题）
- [[Motion Matching]]（**对偶**：MM 的候选集是"录制的片段"，DSD 零样本控制的候选集是"策略生成的轨迹池"，**两者的查询都是"用代价函数挑"**——见 [[2026-09-17-DSD — Diffusion Skill Discovery]] 的对照表）
- [[Skinning]]（**分层关系**：蒙皮把**骨骼运动**传给表面，物理/肌肉层把**力**传给表面——它们不是替代，是"传运动"与"传力"两件事；RLMuscle 说明后者可以作为前一层的下一级）

## Game Applications

- 物理可信的 NPC 交互动画（推拉、受击、搬运）
- Ragdoll 之上的主动式物理角色
- 体育/格斗类游戏的接触密集场景

## Important Papers

- [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds]]
- [[2026-09-14-Learning Realistic Athletic Sprinting Without Demonstrations]]（零示范路线：肌肉驱动 + 任务奖励，同周与 InstantMimic 双双跨过训练成本门槛）
- [[2026-09-17-DSD — Diffusion Skill Discovery]]（技能库宽度：扩散 score 估熵梯度）
- [[2026-10-21-Li — RLMuscle Muscle-Driven Character Animation via Reinforcement Learning|RLMuscle]]（**肌肉层**：1D Hill 在 RL 环内解激活 + 3D 体积仿真一次前向，两段共用本构律；194 肌肉 18 ms/帧 → 离线真值而非运行时）

## 接触求解：本页此前空着的那一层（2026-11-21 补）

本页把"物理路线 = 用训练成本换动力学正确性"当作主线。这条主线默认了一件事：**动力学那一侧是可靠的**，所以成本全在训练上。

2026-11-21 补上 [[Contact Dynamics]] 之后，这个默认需要修正。物理角色动画是**接触密集**的——脚踩地、手推门、身体被挤过门框——而每一条接触都是"要么/要么"的条件系统。它的求解器不是"可靠的地基"，是**一叠标好价的近似**：

| 求解器欠的账 | 策略会怎么利用它 |
|---|---|
| 法向变软（soft contact） | 学会"贴着地面滑行"——反正穿透会被推出来，不用承担真实摩擦代价 |
| 盒式摩擦锥各向异性（对角方向 $\mu$ 偏大 $\sqrt2$ 倍） | 学会朝**对角线方向蹬地**换取更大的有效摩擦 |
| 静摩擦是 ε 正则（有蠕滑） | 学会用极缓慢的滑动"免费"移动，看起来像滑步 |
| 穿透靠 position projection 事后推 | 学会利用被推出来的那一下速度当额外推进 |

> **这是本页最该记住的一条**：RL 策略是**求解器欠账的放大器**。它不是"在真实物理里学"，它是在**你这台求解器里**学——于是任何近似都会被优化到极限。VFX 里没人会盯着一个箱子看它有没有 0.3 mm 穿透；但**优化器会**。

这解释了一类常被误判的失败：**策略学到了荒谬的动作（贴地滑、抽搐、对角蹬）**，看起来像奖励设计问题，实际上是求解器在给一条捷径。调奖励往往治不好，因为它不是策略的错——**是策略太聪明，找到你没打算提供的免费午餐**。

> **判据**：动作"看起来不像人"时，先问一句**"这个动作在真实物理里会更贵吗？"** 如果答案是"会"，那不是策略问题，是求解器在补贴它。

这也给"怎么选求解器"加了一条**与实时性无关**的理由：**接触越精确，策略学到的动作越能迁移到现实**（这对机器人是硬需求，对游戏是"动作看起来对不对"）。反过来，如果目标只是游戏里看着对，那么一个**故意做得"硬"一点的求解器**（法向硬、摩擦各向同性、无免费蠕滑）可能比一个快的求解器更省训练成本——**因为它不给捷径，策略不会走偏**。

## Personal Knowledge

Current Level: Hard

## Learning Gap

缺 RL 基础词汇与策略学习直觉；工程侧有一个此前没意识到的缺口：**接触求解**（2026-11-21 补，原写"工程侧不缺"，需要修正——见 [[Contact Dynamics]]）。刚体动力学的公式不难，但"接触力是一组要么/要么的条件系统、精确解没有多项式算法、每个引擎都在用近似换可解性"这一层，此前完全空着；而 RL 恰恰会把这些近似利用到极限。

## Next Learning Step

不急。桥顺序：先把 [[Motion Matching]] 推到 Easy（理解运动学阵营的极限），再看物理阵营解决的是哪类运动学解决不了的问题。当前只需记住一句话：**物理路线用训练成本换动力学正确性，InstantMimic 把这个成本打掉了几个数量级。**
