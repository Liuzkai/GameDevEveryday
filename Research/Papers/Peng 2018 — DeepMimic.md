---
type: paper
title: "DeepMimic: Example-Guided Deep Reinforcement Learning of Physics-Based Character Skills"
authors: ["Xue Bin Peng", "Pieter Abbeel", "Sergey Levine", "Michiel van de Panne"]
year: 2018
published: "2018-08-01"
venue: "ACM Transactions on Graphics 37(4), Article 143（SIGGRAPH 2018），18 页"
url: "https://xbpeng.github.io/projects/DeepMimic/index.html"
code: "https://github.com/xbpeng/DeepMimic"
project_page: "https://xbpeng.github.io/projects/DeepMimic/"
category: ["Physics-based Character Animation", "Reinforcement Learning", "Motion Imitation", "Animation"]
importance: A
historical_importance: 5
game_relevance: 5
production_readiness: "Research（2018 年状态：每技能训练数天、MuJoCo 内、无运行时集成；但范式已被工业界吸收为"物理角色 RL"的标准骨架）"
user_level: Normal
status: unread
aliases: [DeepMimic, 深度模仿, 参考状态初始化, RSI, 提前终止, 复合策略]
tags: [physics-based-character, reinforcement-learning, motion-imitation, animation, classical]
---

# DeepMimic — 0.730 分的那个策略，一个后空翻都没做出来

## TL;DR

把一段动作捕捉（或关键帧）当成**奖励的来源**而不是**播放的内容**：策略在物理仿真里执行 PD 目标，奖励是"每个相位上与参考姿态有多像"加上"任务完成得怎么样"，用 PPO 训。学出来的策略**在无扰动时几乎与参考动作无法区分**，但会自己恢复平衡、自己适配不同体型、自己完成任务目标。

它的真正遗产不是"能翻跟头"，是**两个训练技巧**：**参考状态初始化（RSI）** 与 **提前终止（ET）**。没有这两个，动态技能（后空翻、侧翻、旋踢）根本学不出来。

而全篇最该记的一处，是它**自己的消融表里的一次自相矛盾**：

> 表 5 里"只开 ET、不开 RSI"的后空翻得分是 **0.730**（满分为 1，完整方法 0.791）。正文却写着：*without RSI, the policy never learns to perform a full mid-air flip. Instead, it performs a small backwards hop while remaining upright.*
>
> **0.730 分的那个策略，一个后空翻都没做出来。**

**归一化回报差 0.06，技能从"有"变成"没有"。** 这是本库见过的、关于"指标量错对象"最干净的一个样本——而且它是作者自己诚实写下来的，不是被别人抓出来的。

## Problem

2018 年的物理角色动画卡在两件事上，作者开篇就点名：

- **泛化（generalization）**：手工设计的控制器能做出漂亮的动作，但"能做什么"受限于人的洞察——你没法把"我自己怎么走路"写成控制器。
- **可指挥性（directability）**：给仿真角色编排动作"notoriously difficult"，当时的界面没法让用户"要什么给什么"。

而纯深度 RL 给的动作**质量很差**，作者写得毫不客气：*severe (and sometimes humorous) artifacts, such as extraneous upper body motion, peculiar gaits, and unrealistic posture*（脚注还直接给了个 YouTube 链接）。

于是问题变成：**怎么让 RL 学出"像人"的动作，同时保留 RL 的鲁棒性与任务能力？**

## Historical Context

要理解这篇为什么是分水岭，得看它之前的三条路各自缺什么：

| 路线 | 代表 | 强在哪 | 缺什么 |
|---|---|---|---|
| 运动学数据驱动 | Motion Graphs、Motion Matching 前身 | 数据够好时动作质量最高 | 新情境要"有数据"；扰动一来就崩 |
| 基于模型的控制器 | SAMCON、LQR 跟踪、QP 步态 | 可解释、可保证 | 组件多、迭代步骤多、任务目标难加 |
| 无参考的深度 RL | Duan 2016、Heess 2017 | 通用、能探索 | 动作"怪"：多余的上身摆动、不对称步态 |
| 模仿 + RL（半程） | DeepLoco、GAIL | 已经有了"用数据诱导奖励"的想法 | DeepLoco 固定初始状态 ⇒ **做不了高动态动作**；只做过单臂缺失的双足、只用脚落点目标 |

DeepMimic 的位置：**把"模仿"做成奖励里的一项，而不是做成一个跟踪控制器**。跟踪控制器要求参考动作"物理可行"；奖励不要求——你只要"越像越好"，物理上做不到的部分策略自己会找替代。

**RSI 也不是它发明的**（作者引了 Sharon & van de Panne 2005 的平面双足、Nair 2017 的机械臂），但**是它把 RSI 与 ET 从"实现细节"提升为"方法的核心主张"**——这在写作上也是一次示范：把两个没人当回事的训练设置拿出来做消融，证明它们是"能不能学会"的分水岭。

## Core Idea

三句话：

1. **参考动作只提供"像不像"，不提供"怎么动"。** 策略负责在物理里找到能产生相似轨迹的力矩序列。
2. **相位变量 $\phi\in[0,1]$ 把"时间"接进状态。** 参考姿态是随时间变的，所以状态里必须有一个"现在该做第几帧"的指针。$\phi$ 从 0 线性走到 1，循环动作到 1 归零。
3. **训练时的初始状态分布与终止条件是一等公民。** 它们不是环境的固定属性，是**引导探索的手段**。这一条是全文最有迁移价值的判断。

## Technical Approach

### 状态、动作、网络

状态 $s$：各连杆相对根（骨盆）的位置、四元数旋转、线速度与角速度，全部在角色局部坐标系里算，**加上相位 $\phi$**；有任务时再加目标 $g$。人形是 **197 维状态 / 36 维动作**。

动作 $a$：**每个关节 PD 控制器的目标角度**（球关节用轴角，铰链用标量角）。策略 **30 Hz** 查询。作者明确说这是相对"直接出力矩"的改进——PD 把阻尼与局部反馈这些低层细节抽象掉了。

网络：两层全连接 **1024 / 512**，ReLU，输出高斯均值，**协方差固定为对角阵、当超参**。价值网络同构、单输出。**就这么小。**（对比今天的 Transformer 策略，这个规模本身就是一条信息：2018 年的瓶颈不在容量。）

### 模仿奖励：四项，各有各的尺度

$$r^I_t=w_p r^p_t+w_v r^v_t+w_e r^e_t+w_c r^c_t,\qquad w_p{=}0.65,\ w_v{=}0.1,\ w_e{=}0.15,\ w_c{=}0.1$$

$$r^p_t=\exp\Big(-2\sum_j\|\hat q^j_t\ominus q^j_t\|^2\Big)\qquad r^v_t=\exp\Big(-0.1\sum_j\|\hat{\dot q}^j_t-\dot q^j_t\|^2\Big)$$

$$r^e_t=\exp\Big(-40\sum_{e}\|\hat p^e_t-p^e_t\|^2\Big)\qquad r^c_t=\exp\big(-10\|\hat p^c_t-p^c_t\|^2\big)$$

**四个系数不是随手调的，它们在做"单位换算"**：姿态用弧度（量级 ~0.1），速度用 rad/s（量级 ~1），末端位置用米（量级 ~0.1），质心用米。系数把四种不同量纲的平方误差拉到可比的量级，再让 $w$ 去分配注意力。

而 $w_p=0.65$ 说明作者的判断：**姿态对了，其余大多会对**。末端位置只给 0.15——因为手的位置由肩肘腕的旋转决定，姿态项已经隐含了它。

最终奖励 $r_t=\omega_I r^I_t+\omega_G r^G_t$，任务项举例：

- **目标朝向**：$r^G_t=\exp[-2.5\max(0,v^*-v_t^\top d^*)^2]$——**只罚"慢了"，不罚"快了"**。
- **击打目标**：命中给 1，否则 $\exp(-4\|p^{\text{tar}}-p^e\|^2)$，命中判定是连杆中心落在目标 0.2 m 内。

### RSI：把参考动作当成初始状态分布

固定初始状态的害处，作者讲得很直白：策略必须**按时间顺序**先学会前半段才能碰后半段；而后空翻这种动作，**"学会落地"是"敢跳"的前提**——跳了但落不了地，回报反而更差。于是策略永远不去探索"跳"。

RSI 的做法：**每个 episode 开始时，从参考动作里随机采一个状态初始化角色。**

> 效果是把参考动作变成"**另一条信息通道**"：不只是通过奖励告诉策略"该像什么"，还通过初始分布直接把它**放到那些好状态上**。

### ET：把"摔倒之后在地上乱蹬"的数据删掉

终止条件：躯干或头触地即终止，之后回报为零。作者给了两层理由：

- **塑形**：摔倒后零回报 ⇒ 摔倒这件事本身被罚；
- **数据配比**（这个更本质）：不终止的话，训练早期的样本会被"在地上挣扎"占满，网络容量被浪费在建模无望的状态上——**和类别不平衡是同一个问题**。

### 三条多技能路线（这是 2026 年那条线的起点）

1. **Multi-clip reward**：$r^I_t=\max_j r^j_t$。一个策略吃多个片段，自己挑当下最像哪个。
2. **Skill Selector**：给一个 one-hot 目标 $g_t$，$r^I_t=r^i_t$（$g_{t,i}=1$）。每个循环开始时随机采样 $g_t$ ⇒ **"the policy is therefore required to learn to transition between all skills within the set of clips."**
3. **Composite Policy**：各技能独立训，运行时按值函数做 Boltzmann 加权

$$\Pi(a|s)=\sum_i p_i(s)\pi_i(a|s),\qquad p_i(s)=\frac{\exp(V_i(s)/T)}{\sum_j\exp(V_j(s)/T)}$$

**不需要额外训练。** 而且作者提到一个很妙的现象：角色摔倒时，复合策略**自动**激活起身策略，"without requiring any manual scripting"。

## Key Contribution

### ① 三十个技能、四种角色（Table 2）

人形 24 个技能，归一化回报从 **Frontflip 0.485** 到 **Walk 0.985**；另加 Atlas 4 个、T-Rex 1 个、Dragon 1 个。

| 技能 | 回报 | 训练样本 |
|---|---|---|
| Walk | **0.985** | 61 M |
| Dragon: Walk | 0.990 | 139 M |
| Run / Jog | 0.951 | 53 / 51 M |
| Crawl | 0.932 | 68 M |
| Jump | 0.947 | 86 M |
| Cartwheel | 0.804 | 51 M |
| Spinkick | 0.748 | 67 M |
| Backflip | 0.729 | 72 M |
| Headspin | 0.640 | 112 M |
| Landing | 0.590 | 66 M |
| **Frontflip** | **0.485** | 81 M |
| **Atlas: Spinkick** | **0.477** | 66 M |

**读法**：**周期性、落地稳的技能接近满分；有腾空、要求精确落地的技能分数低。** Frontflip 0.485 与 Backflip 0.729 的差不是"前空翻更难"，是**前空翻的落地容错更小**——参考动作里那 1.65 s 内没有第二次机会。

### ② 消融：RSI 与 ET 各自管什么（Table 5）——**全篇最值钱**

| 技能 | RSI + ET | 只 ET | 只 RSI |
|---|---|---|---|
| Backflip | **0.791** | 0.730 | **0.379** |
| Sideflip | **0.823** | 0.717 | **0.355** |
| Spinkick | 0.848 | **0.858** | **0.358** |
| Walk | 0.980 | 0.981 | 0.974 |

**这张表的形状比数值更重要**：

- **"只 RSI"那一列在三个动态技能上全部塌到 0.35–0.38**——比"只 ET"低一半。没有 ET，策略学到的是"躺在地上模仿"这个局部最优（作者原话：*the character falls and mimes the motions as it lies on the ground*）。
- **"只 ET"那一列几乎追平完整方法**（0.730 / 0.717 / **0.858**）。旋踢上它甚至更高。
- 也就是说：**ET 是承重的，RSI 是锦上添花的**——如果你只看这张表。

**但正文推翻了这张表。** 作者写：没有 RSI，后空翻"**从未学会完整腾空翻转**，只是原地向后小跳一下、保持直立"。而它的得分是 **0.730**。

> **0.730 与 0.791 差 0.06，而其中一个根本不会后空翻。**
>
> 归一化回报把"做了个像样的动作"和"做了个不相干的动作但没摔倒"混在同一个尺度上——因为它的分母是"这个 episode 内的最低/最高回报"，而**"原地小跳"在奖励塑形下也能拿到七成**。

Fig 11 把这件事画出来了：左上角 Humanoid: Backflip 的面板里，橙线（只 ET）稳稳爬升到 **0.73 平台**——一条看起来非常健康的曲线，对应的是**零个后空翻**。

**这就是"指标量错对象"**：数字量的是"奖励的累积"，被检验的是"技能是否存在"。两者在奖励工程的重叠区里可以脱钩。

**还有一条更硬的方法学冷水**：作者自陈 *the majority of performance statistics are collected from one run of the training process*——**表 5 基本是单种子**。0.791 vs 0.730 在单种子上根本不足以谈显著；**"ET 单独就够"这个结论，不能从这张表里读出来**，只能从正文的定性观察里读出来。**表格与文字给出的结论方向相反，而以文字为准**——这在读论文时是必须习惯的一件事。

### ③ 鲁棒性：训练时从没加过扰动（Table 6）

在动作周期中点对骨盆施加 0.2 s 的推力，逐级加大直到摔倒：

| 技能 | 前向 (N) | 侧向 (N) |
|---|---|---|
| Run | **720** | 300 |
| Spinkick | 690 | **600** |
| Backflip | 440 | 100 |
| Cartwheel | 200 | 470 |
| Walk | 240 | 300 |

（人形体重 45 kg。720 N × 0.2 s ≈ 144 N·s 冲量。）作者把鲁棒性归因于**训练时随机策略的探索噪声**——**没有做 domain randomization，只是 PPO 的高斯策略本身就在扰动**。

### ④ 重定向：策略不能跨体型搬，但同一套方法可以（§10.3）

把人形训练好的策略**直接**用到 Atlas（169.8 kg，近 4 倍质量）上：Run 归一化回报 **0.013**、Backflip **0.014**。而用同样参考片段为 Atlas 重新训练：**0.846 / 0.630**。

> **差两个数量级。** 这解释了为什么"物理角色"这条线的每一个新角色都要重训——**而 2026 年的 MESSI 之所以有价值，正是因为它在同一个角色上把"加一条技能"从"重训"变成"训一个 0.39M 的补丁"**。

另外两个重定向实验：从平地落地片段学"从 2 m 高台跳下"（成功）；**月球重力 $1.622\ \text{m/s}^2$** 下旋踢 0.792、侧手翻 0.688（也成功）。**同一个奖励，换环境照样能用**——这是"奖励而非跟踪"的直接好处。

## Why It Works

三件事，缺一不可：

1. **参考动作定义了"像"，但没定义"怎么动"。** 这让策略有自由去用物理上可行的方式逼近——于是"落地时多迈半步"这种参考动作里没有的修正会自己长出来。
2. **RSI 把"探索"从时间轴上解耦。** 没有它，策略必须按顺序征服时间；有了它，**每个相位都是"起点"**，梯度可以并行地作用在整条轨迹上。
3. **ET 让"失败"变成可辨识的信号。** 摔倒后零回报 + 立刻重开，等于给策略一个廉价的"这一步错了"的标签；不终止则这个标签被淹没在"在地上挣扎的几百帧"里。

## Limitations

作者自己列的（§11），每一条八年后看都还在：

- **相位变量线性推进**：$\phi$ 随时间匀速走，策略**不能调整动作的快慢**。这直接限制了"被打断后重新对齐"的能力——也是后来 motion matching 与 phase-free 方法（如 ASE）的切入点。
- **多技能只在小数量上验证**：*has not yet been demonstrated on large motion libraries*。**2026 年 MESSI 的 19.8% 就是这句话的实证。**
- **PD 增益要按角色手调**：换一个形态就要重新给增益，作者说这仍然需要洞察。
- **训练极慢**：*several days per skill*，且**每个策略独立训练**。41–191 M 样本/技能。
- **奖励仍是手工定义的状态相似度**，四项权重"needs to be defined with some care"。

补充两条本文没强调但很硬的：

- **表 5 基本是单种子**（见上）。
- **同期视频里的动作都是在 MuJoCo 里、无渲染质量可言**。它的说服力全部来自"物理正确"这一侧，视觉保真度不在讨论范围内。

## Game Development Relevance

这篇是**"物理角色"这条线在游戏语境下的原点**，价值不在"能翻跟头"，在三件事：

1. **它给了"动画数据"一个新的用法。** 以前 mocap 是**要播的内容**；这里它是**奖励函数的一部分**。这条转向后来长出了整个"神经动画"分支（[[Neural Animation]]、ASE、Motion Matching 的学习版）。
2. **它把"多技能"拆成三条路，而三条路今天都还在用**：
   - max 奖励 → 今天的"统一策略吃混合数据"；
   - skill selector → 今天的"技能条件化策略"（MESSI 的学生就是它的直系后代）；
   - 值函数复合 → 今天的"运行时策略选择 / 混合"（也是 MESSI 那个 exit head 的远亲）。
3. **它的失败模式是可复用的诊断表。** 学不出来时依次问：初始状态分布对不对（RSI）？失败样本有没有被终止掉（ET）？相位对齐了没有？奖励量纲配平了没有？

## Unreal Engine Relevance

- **和 Motion Matching 的分工**：MM 负责"常见情况下的高质量"，物理策略负责"扰动、交互、非预定义情形"。**DeepMimic 提供的是后者的训练范式**；工程上常见的做法是 MM 为主、物理策略在受击/失衡时接管。
- **Control Rig / Physics Asset**：动作是 PD 目标角度 ⇒ UE 里就是 `Set Angular Drive Target`，**接口天然对齐**。
- **不适用**：需要逐帧确定性复现的场合（联机同步）。RL 策略是随机的、对仿真步长敏感的，**回放一致性是硬障碍**。
- **Chaos 与 MuJoCo 的差异**：本文的接触与摩擦模型是 MuJoCo 的；换到 Chaos 上，同一套奖励大概率要重调——参见 [[Physics-based Character Animation]] 那条"策略是求解器欠账的放大器"。

## Technology Evolution

```text
2018  DeepMimic：参考动作 → 奖励；RSI + ET 让高动态技能可学；三条多技能路线
      瓶颈：几天/技能、单技能单策略、相位锁死时间
  ↓
2021  ASE / AMP：对抗式运动先验取代逐帧姿态匹配 ⇒ 动作更像人、对参考的依赖减弱
  ↓
2024  SuperPADL：专家 → 学生的规模化蒸馏，"技能库"成为工程对象
  ↓
2026  SkillX / PhysicsFC / Joga / MESSI：统一策略装多技能、用前驱状态微调、只在接缝上挂适配器
      其中 MESSI 量出的 19.8% 正是 DeepMimic「skill selector」那条路的天花板
```

**八年，问题从"能不能学会一个技能"变成了"技能之间接不接得上"。** 而 DeepMimic 在这两个问题里各留了一半：它证明了可以学会，也顺手把"接不上"写进了 Limitations。

## Relationships

### Based On

- **Sharon & van de Panne 2005 / Nair 2017** — RSI 的前身（平面双足、机械臂操作）。
- **DeepLoco（Peng et al. 2017）** — "模仿项 + 任务项"的第一次尝试，但固定初始状态 ⇒ 做不了高动态动作。
- **GAIL（Ho & Ermon 2016）** — "用数据诱导奖励"的同代思路。
- **PPO / GAE(λ) / TD(λ)** — 优化器骨架。

### Extends

- **SAMCON（Liu et al. 2010/2016）** — 它此前是唯一能做出大量高动态动作的系统，但组件多、要人工定义低维状态表示、任务目标难加。DeepMimic 用"一个奖励 + 一个 RL"换掉了那套复杂性。

### Related

- [[Physics-based Character Animation]] — 本页是该概念的**根节点**；此前那一页把 DeepMimic 写成纯文本、没有链接，今天补上。
- [[Motion Matching]] — **对偶**：MM 把"动作"当检索问题（要从数据里找得到），DeepMimic 把"动作"当控制问题（只要物理上做得到）。**MM 的边界是数据覆盖，DeepMimic 的边界是训练成本。**
- [[Motion Retargeting]] — 本文的"角色重定向"是**最粗暴的版本**（直接拷局部旋转），也正因如此暴露了策略不能跨体型搬（0.013 vs 0.846）。
- [[Amortization Gap]] — **本文是"不摊销"的极端样本**：每个技能一个策略、几天训练、零共享。它后来所有工作的动机都能写成"能不能少摊销一点损失"。

### Followed By

- [[InstantMimic — A High Performance System for Learning Physics-based Skills in Seconds|InstantMimic 2026]] — 把"几天/技能"压到秒级。
- [[2026-09-17-DSD — Diffusion Skill Discovery|DSD 2026]] — 补技能库的"宽度"。
- [[2026-09-14-Learning Realistic Athletic Sprinting Without Demonstrations|Sprinting 2026]] — 打掉"必须有参考动作"这个前提。
- [[2026-11-26-Peng — MESSI Multi-Expert Soccer Skill Integration|MESSI 2026]] — 正面回答"多技能怎么接"。

## Personal Knowledge State

`Normal`。奖励四项与 RSI/ET 的动机都能讲清；**没吃透的是"为什么旋踢在只 ET 时反而更高（0.858 > 0.848）"**——大概率是单种子噪声，但也可能是"RSI 让策略过度依赖参考的入场姿态"。这条待后续论文验。

## Learning Value

三条判据，都能搬到非角色动画的地方：

1. **"训练时的初始状态分布"是方法的一部分，不是环境的配置。** 想让模型学会"从任意状态接住"，就得让它从任意状态开始——这一条对任何序列生成/控制问题都成立。
2. **"失败样本要能被廉价地终止掉"。** 不终止，它们会以数量优势淹没梯度（与类别不平衡同构）。
3. **归一化回报不是技能存在性的证据。** 判据：**换一个"技能是否真的发生"的二值指标重测一遍**（有没有翻转？有没有触球？有没有命中？）。DeepMimic 自己是靠看视频发现的，不是靠看表。

## Visualization

![[Research/Files/Pictures/Peng 2018 — 图11 RSI与ET的消融学习曲线.png]]
**读图**：四个技能的学习曲线（横轴样本数 ×10⁷，纵轴归一化回报）。蓝 = RSI+ET，橙 = 只 ET，绿 = 只 RSI。**看绿线**：Backflip / Sideflip / Spinkick 三个动态技能上，绿线全部卡在 **0.35–0.4** 一条水平线上不再上升——这就是"没有 ET 会收敛到躺在地上模仿"的图形。**再看左上角橙线的平台：0.73。** 那就是正文说的"从未学会完整腾空翻转、只是原地向后小跳"的那个策略。**一条健康的曲线，一个不存在的技能。** 出处：Peng et al. 2018, Figure 11。

![[Research/Files/Pictures/Peng 2018 — 图3 四种角色形态.png]]
**读图**：同一套框架下四个角色——(a) Humanoid（13 连杆 / 45 kg / 34 DoF）、(b) Atlas（12 / 169.8 kg / 31 DoF）、(c) T-Rex（20 / 54.5 kg / 55 DoF）、(d) Dragon（32 / 72.5 kg / 79 DoF）。**这张图是"参考动作 + 奖励"这个范式的适用边界声明**：状态维度从 197 到 418、动作维度从 36 到 94，方法一个字没改。**但注意——每个角色都是重新训练的一整套策略，不是迁移。** 出处：Peng et al. 2018, Fig. 3。

## Notes

- **作者构成**：Xue Bin Peng（UC Berkeley，一作）、Pieter Abbeel、Sergey Levine、Michiel van de Panne（UBC）。这四个人后来分别主导了这条线的不同分支（Peng → ASE/AMP/InstantMimic 生态；van de Panne → 物理角色与运动控制的教学传统）。
- **注意别和 [[2026-11-26-Peng — MESSI Multi-Expert Soccer Skill Integration|MESSI]] 的作者搞混**：那是 **Xiaogang Peng**（Northeastern），这是 **Xue Bin Peng**（Berkeley）。同姓不同人。
- 本文是 **SIGGRAPH 2018** 论文，ACM TOG 37(4) Article 143，18 页。项目页与代码都在，是这条线里少见的**可复现**工作（`xbpeng/DeepMimic`，C++/Python，依赖 MuJoCo + Bullet）。
- 一句话记住它：**"参考动作不是拿来播的，是拿来当奖励的。"**

★ 2026-11-26
