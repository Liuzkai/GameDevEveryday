---
type: concept
user_level: Normal
aliases: ["Secondary Motion", "二次运动", "二级运动", "次级运动", "Follow Through", "Overlapping Action", "余韵", "跟随动作"]
prerequisites: ["[[Skeletal Animation]]", "[[Skinning]]"]
first_introduced: "Thomas & Johnston《The Illusion of Life》(1981) 的十二条原则；Lasseter 1987 将其搬进 CG"
---

# Secondary Motion

## Definition

**主体运动（primary motion）之外的、由主体运动引起的附属响应。**

角色往前走是主体运动；帽檐往后甩、肚子上的肉晚半拍才跟上、尾巴像波一样一节节传过去——这些是二次运动。**它不改变"去哪"，只改变"看起来有多重"。**

判据只有一句：**主体停了，它还在动吗？** 在，就是二次运动。

## Core Principle

### 一、它不是"另一种运动"，是"同一个运动的延迟与残余"

二次运动在数学上是**主体运动信号经过某个算子之后的输出**。所有实现路线，区别只在**这个算子是什么**：

| 路线 | 算子 | 需要什么输入 | 能不能和世界交互 | 成本量级 |
|---|---|---|---|---|
| **全物理仿真**（CPD / CODY） | 解动力学方程（正交子空间） | 初始条件 + 材料参数 | ✅ 能（有碰撞、有体积） | **0.05 ms/面片**（CODY，[[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters\|DS 2026]] 表 1） |
| **降阶/子空间物理** | 少量模态的动力学 | 预计算的模态基 | ⚠️ 部分 | 介于两者之间 |
| **解析形变项**（VS / AS / DS） | 关节运动学量的代数函数 | 当前/过去几秒的关节速度 | ❌ 不能 | **0.00003 ms/面片**（DS 2026） |
| **弹簧骨 / 骨链**（Kawaii Physics、Dynamic Bone、AnimDynamics） | 每根骨一个二阶系统（Verlet 或弹簧） | 父骨的位移 | ⚠️ 只与**自身碰撞体**交互 | 每骨每帧常数 |
| **离线滤波**（Cartoon Animation Filter） | 整条轨迹 ⊗ 一个核 | **整段动画** | ❌ 不能 | 离线 |
| **布料/毛发解算**（Chaos Cloth、Hair） | 每顶点/每段独立自由度 + 约束 | 碰撞体 + 约束 | ✅ 能 | 每顶点每帧数次迭代 |

**这张表的排序轴不是"精度"，是"要不要和世界交互"。** 一旦要碰撞，就必须有自由度；一旦有自由度，就必须解约束；一旦解约束，成本就按顶点数而不是按角色数涨。

### 二、唯一一条硬边界：它需要多长的"时间窗"

同样的分类，换成时间来看会更锋利：

```text
形状类（squash & stretch / arcs）       → 只需要【当下】
余韵类（follow-through / overlapping）  → 只需要【过去 W 秒】
预判类（anticipation）                  → 需要【未来】
```

- **当下**：$\psi(\Omega_j)$，一个代数式，零成本；
- **过去**：$\mathcal F(\Omega_j,t)=\Omega_j(t)+\int_0^W\kappa(u)\dot\Omega_j(t-u)\,du$，需要一个长度为 $W$ 的历史缓冲；
- **未来**：没有因果算子能做到。**这是"实时"对"动画原则"的唯一一处硬裁剪**。

**推论（本页的核心判断）**：游戏里绝大多数"二次运动看起来很假"的问题，不是算子不够好，是**时间窗给错了**——要么太短（余韵还没出来就归零，看起来"轻"），要么把"预判"当成"余韵"来做（起手永远突然）。

### 三、记忆长度不是一个自由参数

DS 的默认值是 $W=5\tau$。这不是随手取的：$\kappa(t)=Ae^{-t/\tau}\sin(2\pi ft)$ 在 $5\tau$ 之后衰减到 $e^{-5}\approx0.7\%$，再往后记也没用。

**所以"要记多久"这件事，是被"衰减多快"决定的，不是独立可调的。** 这条对所有"带余韵"的实现都成立：**你先决定 $\tau$（多快停下来），$W$ 就跟着定了**。反过来设计（先定 3 秒的时长，再问为什么不像）是典型的把结果当输入。

## Prerequisites

- [[Skeletal Animation]]：主体运动的来源。没有骨架量（关节位置/速度/层级），就没有驱动二次运动的信号；
- [[Skinning]]：二次运动的载体。**蒙皮负责"骨动皮动"，二次运动负责"骨停了皮还在动"**——两者是加法关系，不是替代关系；
- **线性时不变系统与卷积**：理解"为什么 FIR 核比二阶递推好调"，只需要知道 FIR 没有反馈、因而没有稳定域。

## Historical Evolution

```text
1930s  迪士尼：follow-through / overlapping / secondary action 作为"表演技巧"口传
1981   Thomas & Johnston《The Illusion of Life》：写成文字
1987   Lasseter：搬进 CG，成为验收清单                    ← [[Lasseter — Principles of Traditional Animation Applied to 3D Computer Animation (1987)|Lasseter 1987]]
  ↓   以下四十年，是"算子"的四次换代
1987   Terzopoulos et al.：可变形体物理建模 ⇒ 二次运动可以"算"出来（但贵、且要初始条件）
2006   Cartoon Animation Filter：整条轨迹 ⊗ (−LoG) ⇒ 第一次"不求导、只滤波"（离线）
2008   Wiggly Splines：把振荡写成带约束的样条 ⇒ 好看，但要解全局系统
2010s  弹簧骨普及（Dynamic Bone / Kawaii Physics / AnimDynamics）⇒ 每根骨一个二阶系统，便宜、会炸、要和碰撞体打交道
2021   Velocity Skinning：每顶点、当前帧的关节角速度 ⇒ 因果，能进顶点着色器
2022   Acceleration Skinning：加一阶导 ⇒ 有了一次"反转"
2026 ★ Dynamic Skinning：因果 FIR 核 + 沿骨架延迟 ⇒ 有余韵，且仍然因果   ← [[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters|DS 2026]]
```

**四十年的方向是单向的：时间窗越来越短。** 1987 要整个表演，2006 要整条曲线，2008 要整段解，2010s 只要父骨的当前位移，2026 只要最近 5 秒。**被交换掉的东西始终是同一个：预判。**

## Important Papers

| 论文 | 它给这条线添了什么 |
|---|---|
| [[Lasseter — Principles of Traditional Animation Applied to 3D Computer Animation (1987)\|Lasseter 1987]] | **规格书**：把"余韵"变成可验收的性质 |
| [[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters\|DS 2026]] | **第一个"有时间维"的解析实现**：$\mathcal F$ 是 FIR 核，$\beta$ 免费传播，dLBS 让波动沿骨架走 |
| [[2026-10-30-He — HairDeck Animation-Ready Stylized Hair Cards from a Single Image\|HairDeck 2026]] | **另一条答案**：生成真的骨链（48 条弹簧骨），会受力、会和头碰撞——**代价是要 substep、要调参** |
| [[Cloth Simulation]]（概念页） | 极端的一端：每顶点独立自由度。**它是二次运动的"全保真"版本** |
| （库外）Cartoon Animation Filter, Wang et al. 2006 | **第一次把原则写成滤波器**；对称核 ⇒ 能造预判，但只能离线 |

## Related Concepts

- [[Skinning]]：**必须分清**——蒙皮是"骨架 → 皮肤的映射"（零时间维），二次运动是"骨架的导数 → 皮肤的附加位移"（有时间维）。**DS 的两项是相加的，不是二选一**；
- [[Cloth Simulation]]：同样是"骨停了还在动"，但布料有**自己的自由度**（会穿模、会自碰撞），二次运动没有。**判据：衣服穿进大腿是碰撞体问题，帽檐甩得假是算子问题**；
- [[Hair Rendering]] / [[Cloth Simulation]]：两条"高保真"分支。它们和本页的关系是"同一件事的不同预算档"；
- [[Physics-based Character Animation]]：把二次运动做成"真的动力学"，最贵的一档；
- [[Motion Matching]]：**相反的策略**——不生成余韵，检索"已经带余韵的片段"。**当"数据库里有"比"算得出"更便宜时，检索赢**。

## Technologies

- **AnimDynamics（UE）**：每根骨一个弹簧+阻尼，带球/胶囊碰撞体。**反馈系统 ⇒ 参数跨过稳定域会抖炸，需要 substep 或 clamp**；
- **Kawaii Physics / Dynamic Bone（Unity/VRM）**：Verlet + 距离约束，面向发丝与飘带。**便宜、稳、但不能表达"波动沿骨架传播"**（每根骨只看自己的父骨）；
- **Chaos Cloth / Chaos Flesh**：全保真一档，用于披风、肌肉；
- **顶点着色器位移项**（DS 式）：把"每关节滤波 + 每顶点加权"塞进 VS。**关键工程改动是把滤波 hoist 到关节级**，否则复杂度是 $O(V\cdot J\cdot W/\Delta t)$。

## Game Applications

| 场景 | 推荐档位 | 理由 |
|---|---|---|
| 二次元角色的帽穗 / 发梢 / 飘带 | **解析项**或弹簧骨 | 只要"看起来轻"，不需要碰撞；解析项永不炸 |
| 写实角色的受击颤动 / 重甲质量感 | 解析项（$\tau$ 调大） | 需要"有质量"，不需要"守恒" |
| 长发 / 长裙 / 披风 | 弹簧骨 → 布料 | 需要和身体碰撞，必须上自由度 |
| 主角特写的一次性镜头 | 离线滤波或全物理 | 可以预读整段，能买回 anticipation |
| **技能 VFX 的"起手"与"余韵"** | **两件事，两套价** | 余韵 = 一个滤波器（因果、便宜）；起手 = 一次前瞻排期（要意图、贵）。**这就是为什么游戏特效总是"收得漂亮、起得突然"** |

## Personal Knowledge

**Normal（判据层已清楚，缺"手感 → 参数"的映射）。**

- **Easy**：什么叫余韵、为什么它让角色"有重量"、弹簧骨与布料的区别；
- **Normal**：三条路线的时间窗差别、FIR 与二阶递推在可调性/稳定性上的分野、$W=5\tau$ 的来历；
- **Hard**：不需要。这条线上没有需要啃的数学，难的是"调到好看"。

## Learning Gap

**缺的不是知识，是一次对照实验。** 库内现在有三篇能凑成一个受控对比（DS 的解析项 / HairDeck 的骨链 / Cloth 的解算），但没有人在**同一个资产、同一个动作**上并排跑过。

具体缺三件事：

1. **同一根飘带、三档实现，并排看**：解析项 / 弹簧骨 / 布料。**要看的不是"谁更像"，是"在什么镜头距离上开始看不出差别"**；
2. **"时间窗"的可感知阈值**：$\tau$ 从 0.1 s 调到 2 s，人眼在第几档开始觉得"这是另一个材质"——**这个阈值决定了要开哪一档**；
3. **预判的替代方案**：游戏里有没有"提前 0.2 s 知道下一段动作"的现成接口（Montage 的下一段、State Machine 的转移条件）——**如果有，anticipation 就不是做不到，是没接上**。

## Next Step

1. **30 分钟**：在引擎里给一个现有角色加一档解析项余韵（$\tau=1\,\mathrm{s}$、$f=1.5\,\mathrm{Hz}$、$A=0.3$，DS 的默认值），**只调 $\tau$**，找出"开始像布料"的那个值；
2. **一次评审动作**：把现有特效的"起手"与"余韵"分开列两栏，**看有没有哪一条被当成同一件事在做**；
3. **按需**：若要和世界交互（碰撞、风），**不要扩解析项**，直接换弹簧骨或布料——**这条线的成本不是连续的，是跳档的**。

---

## Notes

- 本页与 [[Skinning]] 的分工：蒙皮页回答"骨动皮怎么动"，本页回答"**骨不动了皮为什么还在动**"。两页共享同一套输入（骨架 + 权重），但**一个是映射，一个是算子**。
- **一个容易混淆的历史事实**：`secondary action`（次要动作）与 `secondary motion`（二次运动）在中文里都译作"次级/二次"，但**不是一回事**——前者是"表演层再加一个小动作"（比如说话时转笔），后者是"物理层的一个延迟响应"。本页讲的是后者，但两者在实现上经常共用同一套系统（additive layer）。
- **2026-11-03 建页**：建页前**库内零覆盖、零引用**——175 篇论文里没有一篇以它为主题，概念页也没有一个提到它（`grep` 全库为 0）。原因大概是它一直"看起来像美术的事"而不是技术的事；**但它是表现层唯一没有被计价的维度**（五维表管发光、管粒子、管贴图，没有一格管"动得好不好看"）。
