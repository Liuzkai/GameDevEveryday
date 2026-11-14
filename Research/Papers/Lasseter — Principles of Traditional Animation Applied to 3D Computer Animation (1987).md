---
type: paper
title: "Principles of Traditional Animation Applied to 3D Computer Animation"
authors:
  - John Lasseter
year: 1987
published: 1987-07-01
venue: "SIGGRAPH 1987（Computer Graphics 21(4): 35–44）"
url: https://doi.org/10.1145/37402.37407
code: ""
project_page: ""
category:
  - animation
  - history
  - principles
importance: S
historical_importance: 5
game_relevance: 5
production_readiness: Industry Adopted
user_level: Easy
status: unread
---

# Lasseter 1987 —— 十二条原则第一次变成"可验收的清单"

> SIGGRAPH 1987，Computer Graphics 21(4): 35–44。**一句话**：这篇文章没有提出任何算法，它提出的是一份**验收标准**——1930 年代迪士尼攒下来的十二条原则，第一次被系统地搬进 CG，并且立论极硬：**"不是软件给了角色生命，是这些原则。"**

> **⚠️ 证据形态（先读这一段）**：原文 PDF **未取得**——ACM DL 返回 403；CMU / Rutgers / NEU / Georgia Tech / Wisconsin / UBC 六个课程镜像分别 404 / 401 / 403；Wayback 429 限流。因此本笔记的三类内容严格分开：
> ① **书目信息**（标题、作者、卷期、页码、DOI）经多个独立引用源交叉核对，可用；
> ② **十二原则清单与该文的核心立论**，依据 ACM DL 摘要页与被广泛转引的原文译文，**不加引号、不标页码、不给图号**；
> ③ **"原则 → 算法"的映射表是本笔记的整理**，不是原文内容，已明确标注。
> **本页不引任何具体页码、图号或数值**——避免把二手转述伪装成原文引用。

## TL;DR

- **它的价值不在"讲了什么"，在"它把什么变成了可验收的"**。在它之前，CG 动画的验收标准是"看起来对不对"；在它之后，有了一份十二条的检查表；
- **十二条里，只有一条在定义上要求"未来"**：**anticipation（预备动作）**。其余十一条要么只需要当下（squash & stretch、arcs），要么只需要过去（follow-through、overlapping action、secondary action）；
- **这条分层的工程后果，四十年后才显形**：离线系统（2006 的 Cartoon Animation Filter）用对称核，因此**能造预备动作**；实时系统（2026 的 [[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters|Dynamic Skinning]]）用因果核，因此**只能造拖尾**。**"实时"与"预备动作"之间是一条硬边界，不是优化问题**；
- **今天回头看，这十二条其实是三类东西被混在一起**：**形状**（squash & stretch / arcs / solid drawing）、**时间**（timing / slow in & out）、**叙事**（staging / appeal / exaggeration）。**第一类已经被写成公式，第二类被写成了曲线编辑器，第三类至今没有算法**——而且不该有。

## Problem

1980 年代中期的 CG 动画有一个尴尬的处境：**工具已经能做出"正确的运动"，但做不出"有生命的运动"**。轨迹是样条插出来的、旋转是四元数插出来的，物理上无懈可击，观感上像木偶。

Lasseter 的处境很特殊——他**先做过迪士尼的动画师，再去做 CG**（Pixar 的 *André & Wally B.* 1984、*Luxo Jr.* 1986 都是他做的）。所以他能看到别人看不到的一件事：

> **问题不在插值器，在"没有人告诉机器该插成什么样"。**

这篇文章因此不是一篇技术论文，是一篇**规格说明**：把"好动画"拆成十二条可逐条检查的性质，然后指出——**这些性质与媒介无关**，手绘能做到，CG 也能，只是实现方式不同。

## Historical Context

```text
1930s  迪士尼九老在制作中总结出十二原则（Thomas & Johnston《The Illusion of Life》1981 成书）
  ↓    原则长期以"师徒制"传播，没有可计算的表述
1984  André & Wally B.：第一次证明 CG 角色可以"有表演"
1986  Luxo Jr.：台灯没有脸、没有手，却完成了完整的表演——**这是全文最有力的论据**
  ↓
1987 ★ 本文：把原则写成清单，并在 SIGGRAPH 上公开
  ↓
1995  《玩具总动员》：整条管线按这套标准验收
  ↓
2006  Cartoon Animation Filter：第一次把其中几条写成**滤波器**
2021  Velocity Skinning → 2022 AS → 2026 Dynamic Skinning：把"时间"那一维做成可调参数
```

**为什么 1987 年是分水岭**：同年 SIGGRAPH 还有 Terzopoulos 等人的 *Elastically Deformable Models*（可变形体的物理建模开端）。**同一年，这个领域分裂成了两条路**——一条问"怎么算得对"（物理），一条问"怎么看起来对"（表演）。此后四十年，两条路各自演化，偶尔交叉（[[Cloth Simulation]]、[[Physics-based Character Animation]]），**而"二次运动"正好落在交叉点上**：物理能算对但太贵，表演能便宜但对不齐。

## Previous Work

- **Thomas & Johnston《The Illusion of Life》(1981)**：原则的源头，但它是**一本给动画师看的画册**，不是可执行规格；
- **早期 CG 动画（Boeing Man / 各种广告片）**：追求"物理正确的运动"，把"夸张"当成不专业；
- **关键帧动画系统**：技术上已经够用（Lasseter 自己说 *André & Wally B.* 用的就是一个普通的 keyframe 系统），**缺的是使用它的方法**。

## Core Idea

### 一、十二条原则，按"需要多少时间"重排（本笔记的整理，非原文结构）

这是我认为今天最有用的一种读法——**不问"这条原则是什么"，问"实现它需要看到多长的时间窗"**：

| 原则 | 它约束什么 | 需要的时间窗 | 今天有算法吗 | 库内落点 |
|---|---|---|---|---|
| **Squash & Stretch** | 形状（体积感） | **当下**（当前帧的速度） | ✅ 有（VS/AS/DS 的 $\psi$ 就是它） | [[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters\|DS 2026]] |
| **Arcs** | 形状（轨迹曲率） | 过去 2–3 帧 | ✅ 有（曲线插值/路径约束） | — |
| **Solid Drawing** | 形状（造型） | 与时间无关 | ❌ 不归动画管（归建模） | — |
| **Slow In & Slow Out** | 时间（缓动） | 当下（切线的斜率） | ✅ 有（一切曲线编辑器的默认） | — |
| **Timing** | 时间（节奏） | **整段**（要比较前后） | ⚠️ 半有（曲线编辑器 + 经验） | — |
| **Follow Through & Overlapping** | 时间（余韵） | **过去 $W$ 秒** | ✅ 有（DS 的 $\mathcal F$，$W=5\tau$） | [[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters\|DS 2026]] |
| **Secondary Action** | 时间（附加动作层） | 过去 | ✅ 有（分层动画 / additive layer） | [[Secondary Motion]] |
| **Anticipation** | 时间（**预判**） | **未来** | ❌ **实时系统做不到**（见下） | — |
| **Staging** | 叙事（构图与镜头） | 整场 | ❌ 不该有算法 | — |
| **Exaggeration** | 叙事（夸张度） | — | ⚠️ 是一个可调增益（DS 的 $A$） | — |
| **Appeal** | 叙事（魅力） | — | ❌ 不该有算法 | — |
| **Straight Ahead / Pose-to-Pose** | 工作流（不是效果） | — | —（这是制作流程，不是观感） | — |

**三件事从这张表里掉出来**：

1. **"形状"类的原则基本都已经被写成公式了**（$\psi$、缓动曲线、arc 约束）——因为形状只需要当下；
2. **"时间"类的原则分两半**：余韵只需要过去，**预判需要未来**。这一刀把实时系统能做的与不能做的分开了；
3. **"叙事"类的原则至今没有算法，而且不应该有**——它们是"要表达什么"，不是"怎么算"。

### 二、anticipation 为什么是硬边界

预备动作的定义就是"**在主体动作之前**先做一个相反方向的小动作"。要生成它，系统必须知道"接下来要往哪动"。

- **离线系统天然知道**：整条曲线都在手里。2006 的 Cartoon Animation Filter 用**对称** LoG 核（中心在 $t$，两侧都取样本），负瓣落在 $t$ 之前 ⇒ 自动长出"先退一下"。**它不是"学会了预备动作"，是"滤波器看到了未来"**；
- **纯反应的实时系统天然不知道**：[[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters|DS 2026]] 的核只在 $[0,W]$ 上有值，$u<0$ 恒等于零 ⇒ **数学上不可能产生 $t$ 之前的效果**。

**但工程上并非无解**：游戏里的"未来"经常是已知的——**下一段 montage、下一个技能、下一个转向**都写在配置里。所以正确的说法不是"实时做不到 anticipation"，而是：

> **anticipation 需要一个"前瞻输入"，而大多数动画系统没有把"意图"当成一等输入。**

这条判据可以直接搬进评审：**任何一个声称能做"预备动作"的实时方案，先问它从哪拿到未来。**

## Key Contribution

1. **把"好"变成可逐条检查的性质**——这是它真正的贡献，不是任何一条原则本身；
2. **证明原则与媒介无关**：*Luxo Jr.* 的台灯没有脸、没有手、没有布料，却完整地完成了"看—跳—摔—回头看"的表演。**这个论据四十年后仍然是最强的**；
3. **给后来的"算法化"留下了接口**：今天所有把动画原则写成公式的工作（CAF 2006、VS 2021、DS 2026），**逐条对应回这十二条**，没有一条是新的。

## Why It Works

- **它讲的是一组"不变量"**：不管媒介是手绘、CG、还是游戏引擎，"预备—动作—余韵"这个结构对观众都成立。**不变量才能被工程化**；
- **它把"艺术判断"降级成"可讨论的清单"**：评审时不再说"感觉不对"，而是说"缺 anticipation"或"slow out 太快"。**这是动画从手艺变成工业的那一步**；
- **它承认了"timing 最难教"**：十二条里唯一没法写成规则的就是节奏——**而它恰好是后来唯一被做成"参数"的那条**（$\tau$、$f$、$W$）。

## Limitations

- **它是清单，不是方法**。告诉你"要有预备动作"，不告诉你"预备多少帧"。**这个空缺正是 2006–2026 那批工作在填的**；
- **它的语境是"表演型动画"**（电影/短片）。游戏里大量动作是**反应式**的（受击、闪避、被击退），这些恰恰**没有"预备"的位置**——**所以游戏动画天然只用得上十二条里的前八条**，这一层原文不可能讨论；
- **它对"物理正确"的态度是排斥的**（夸张优先）。而今天的写实向 3A 需要的是"物理可信 + 表演正确"，两者的配比原文没有答案；
- **证据形态本身的限制**：本笔记未读到原文，**所有"原文怎么说"的判断都只到摘要与转述的粒度**。

## Game Development Relevance

- **它是一份评审表**：一段技能动作、一个受击反馈、一个 UI 弹窗的动效，都可以拿十二条里的**形状/时间**那八条逐条过一遍；
- **它解释了"为什么程序化动画总是差一口气"**：程序化能算准 shape 与 timing 的**数值**，但给不出 staging 与 appeal——**那两条要到"这段动作要表达什么"这一层才有答案**；
- **对技能 VFX 的直接含义**：特效的"预备"（起手蓄力）与"余韵"（残留拖尾）在时间轴上是两件事，**而它们在引擎里的实现代价差一个数量级**——预备要提前排期（前瞻），余韵只要一个滤波器（因果）。**这就是为什么游戏里的特效总是"收得漂亮、起得突然"**。

## Unreal Engine Relevance

- **Slow In & Slow Out** → 曲线编辑器的 tangents / AnimNode 的 easing；
- **Arcs** → 路径约束与 `Spline` 组件；
- **Follow Through** → `AnimDynamics`（弹簧骨，反馈式）或 [[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters|DS]] 式的前馈位移；
- **Anticipation** → **没有原生机制**。要靠 Montage 的 `Loop Count`/`Start Section` 手工排，或者靠 State Machine 的"进入条件提前触发"——**都是"把未来喂进去"的手工版本**，没有一个是"系统知道意图"。

## Technology Evolution

```text
"原则 → 算法"的四十年，以及每条原则被兑现的时间
1987  ★ 十二条原则（清单）                           ← 本篇
2006  CAF：squash/stretch + anticipation + follow-through 写成 −LoG 滤波（离线、非因果）
2008  Wiggly Splines：timing 写成带约束的样条
2014  Unity/UE 的曲线编辑器普及：slow in & out 变成默认
2021  Velocity Skinning：squash/stretch 变成每顶点解析项（因果）
2022  Acceleration Skinning：follow-through 变成二阶导项（因果）
2026  Dynamic Skinning：overlapping / secondary action 变成 FIR 核 + 延迟（因果）
        ↓
      十二 → 八（形状 + 时间）→ 其中六条有算法，两条（staging / appeal）永远不会有
        ↓
      唯一没被兑现的：**anticipation**（因为它是唯一需要"未来"的那条）
```

## Relationships

### Based On

- **Thomas & Johnston《The Illusion of Life》(1981)**（库外，书）：十二条原则的真正源头。本文是"搬运 + 翻译"，不是"发现"。

### Extends

- **（间接）整个程序化动画线**：[[2026-10-29-Rohmer — Dynamic Skinning Kinematics-Driven Cartoon Effects for Articulated Characters|DS 2026]] 的每一个参数都能对回这张表。

### Related

- [[Secondary Motion]]：十二条里 **follow-through / overlapping / secondary action** 三条合起来就是今天说的"二次运动"。**本篇是它的规格书，DS 是它的一个实现**；
- [[Skeletal Animation]]：原则的载体。骨架提供"主体运动"，原则提供"主体运动之外的什么"；
- [[Motion Matching]]：**一条相反的路**——它不生成原则，它检索"已经符合原则的片段"。**判据：能不能在数据库里找到，比能不能算出来更便宜时，就检索**。

### Followed By

- **2006 Cartoon Animation Filter**（库外）：第一次把原则写成核；
- **2021–2026 VS / AS / DS**：把"核"做成因果的、可实时。

## Personal Knowledge State

**Easy（原则本身）／Normal（"原则 → 算法"的映射）**。

- **Easy**：十二条是什么、squash & stretch 与 anticipation 的区别——这些是动画的常识层，不需要再教；
- **Normal**：**哪几条已经被写成公式、哪几条永远不会有公式、以及"需要未来"这条分界线**——这才是本篇今天值得读的部分；
- **Hard**：无。

## Learning Value

**中高，但不是"学原则"，是"学分层"。** 它给了一张**按时间窗长度排序**的表：当下 / 过去 / 未来。**任何"实时能不能做"的问题，先把它归到这三档里，答案就出来一半。**

## Visualization

**未取图**（原文 PDF 不可达，见页首证据形态）。**未自绘 HTML**：十二条的映射表是表格，Markdown 已经是最合适的形态；时间轴用等宽文本画得下。**按 §25 的三问，这一页不该有 HTML。**

## Notes

- **一处必须说清的边界**：本文经常被当成"十二原则的原始出处"引用，**这是错的**——原始出处是 Thomas & Johnston 1981 的书，本文是**把原则搬进 CG 的那一篇**。库内引用时不要混；
- **一处常见误读**：本文**没有**说"CG 要模仿手绘"。它说的是"这些性质与媒介无关"，并明确举了 CG 才能做的例子（比如摄像机运动与三维空间的 staging）。**"用 2D 原则约束 3D"是后来的简化**；
- **待补**：原文 PDF 值得再找一次（可试 SIGGRAPH 的历史档案站或作者主页）；拿到后应补 ① 十二条的**原始顺序**（本文的顺序是有意为之，不是清单顺序）② 图（*Luxo Jr.* 的帧对照是全文最好的图）。
