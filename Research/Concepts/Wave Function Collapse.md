---
type: concept
user_level: Normal
aliases: [WFC, WaveFunctionCollapse, 波函数坍缩, 波形坍缩]
prerequisites:
  - "[[Procedural Content Generation]]"
  - 约束满足 / 弧相容（arc consistency）的基本直觉
  - 格点与邻接关系
first_introduced: "2016（Gumin 的单头文件实现）；算法本体可追到 Merrell 2007 的 model synthesis"
---

# Wave Function Collapse

## Definition

**在格点上做约束满足：每个格子只能取"与邻居相容"的若干候选，从候选最多的状态逐步收缩到唯一取值。** 候选集来自一个**例子**（overlapping 模型）或一组**手工瓦片 + 邻接规则**（tiling 模型）。

"波函数坍缩"是比喻：每个格子初始处于"所有候选的叠加态"，每坍缩一个格子（选定一个取值），相邻格子的叠加态被"观测"（传播约束）而收缩，直到整张图确定。

**它不是一种"生成风格"，是一种"约束求解的搜索顺序"。** 风格完全来自输入（例子或瓦片集 + 规则）。

## Core Principle

### 两种模型（区别很大，常被混为一谈）

| | overlapping 模型 | tiling 模型 |
|---|---|---|
| 输入 | **一张例子位图 + 窗口尺寸 $N$** | **一组瓦片 + 每个瓦片四条边的标号** |
| 约束怎么来 | 自动：例子的所有 $N\times N$ 窗口 + 频率，相邻窗口在重叠处必须一致 | 手写：只有边标号相同的瓦片能相邻 |
| 谁是"格" | 像素（每个格子一个颜色） | 瓦片 |
| 频率 | 例子里数出来 | 人给权重 |
| 控制力 | 弱（给例子就得接受它的统计） | 强（能精确控制每种瓦片出现多少） |
| 失败率 | 高（窗口约束很紧） | 低（约束松） |
| 典型用途 | 快速从美术稿复刻风格 | 关卡 / 地牢 / 建筑构件 |

### 算法循环（四步，顺序很重要）

```
1. 初始化：每个格子的候选集 = 全部模式（按频率加权）
2. 传播（propagate）：弧相容 —— 反复删掉"没有任何邻居能配合"的候选，直到不动点
3. 观测（observe）：挑候选最少的格子（最小熵），按频率加权抽一个，坍缩
4. 检查：若某格候选集为空 ⇒ 矛盾
   - 常见实现：整张画布重开（没有回溯）
   - 改进实现：局部回溯 / 块重解（见 [[Merrell 2007 — Example-Based Model Synthesis|Merrell 2007]]）
   否则回到 2
```

**第 3 步的"最小熵"是启发式，不是定理**。它的意思是"先解决最没得选的格子"，这是 CSP 里的经典 MRV（minimum remaining values）启发式。它有效，但**不能保证不走进死胡同**——WFC 的"生成失败、重开一次"就是这么来的。

### 它到底在解什么

一个**约束满足问题**（CSP）：变量 = 格点，值域 = 模式，约束 = 邻接相容。WFC 的解法是"**弧相容 + 贪心分支，不回退**"。

这个定位解释了三件事：
- **为什么快**：弧相容是多项式时间，传播一次很便宜；
- **为什么会失败**：贪心分支没回溯，走进死角只能重开；
- **为什么"总能解出来"不可能**：一般情形下"能否补完"是 **NP-完全**（[[Merrell 2007 — Example-Based Model Synthesis|Merrell 2007]] 附录，从 Planar 3-SAT 归约）。**所以"失败率"是这类算法的固有属性，不是实现 bug。**

### 三类"局部决定不了"的情况（2026 年的新分层）

| 层级 | 谁决定一个格子的值 | 随机访问 | 能承诺什么 | 代价 |
|---|---|---|---|---|
| **全局求解**（WFC 原版） | 整张画布的搜索 | ❌ | 全局合法（若搜索成功） | 矛盾时整张重开；不能流式 |
| **块求解 + 固定周期背景**（2021–23 块求解器） | 块内搜索 + 背景 | ✅（近似） | 块内合法 + 背景强制 | **背景把答案预先钉死了一部分** |
| **局部生成器**（每格只读邻域随机数，含学习型） | 半径 $r$ 内的随机数 | ✅ | **有"沿某条线携带的量"时必然出缝**（密度 ≥ $1/r$ 阶） | 把"合法性"换成"统计相似" |

第三行是 2026 年才被证明的：见 [[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share|IW 2026]]。**这一层解释了为什么块求解器"解不出来就保留背景"不是偷懒——背景就是那个被携带的量本身。**

## Prerequisites

- [[Procedural Content Generation]] 的基本框架（"内容 = 过程 + 参数"）；
- **邻接 / 相容关系**的直觉：什么样的两样东西可以挨着（这是 tiling 模型的全部输入）；
- **弧相容**是什么：一个候选如果没有任何邻居能配合，它就可以被删掉——反复删到不动点；
- （可选）**熵 / 频率**：为什么"按出现频率抽"比"均匀抽"更像例子。

## Historical Evolution

```text
1976  Wang tiles：局部可拼贴的数学原型；难的是造瓦片集，不是拼
        ↓
1999-2001  纹理合成（Efros-Leung / Image Quilting）：局部决定局部
        ↓
2007  ★ Merrell, Example-Based Model Synthesis（I3D'07）
        六向邻接表 + 全局传播 + 切块回滚 + 加权相似度
        附录：能否补完 = NP-完全（Planar 3-SAT 归约）
        自述失效：例子过紧 → 输出会"反复复制例子"
        ↓
2008  Continuous Model Synthesis（脱格点）
2009  博士论文 / 2010 TVCG 期刊版
        ↓
2016  ★ Gumin, WaveFunctionCollapse（GitHub，单头文件 C#）
        同一算法 + 两个模型（overlapping / simple tiling）+ 极低的上手成本
        → 在独立游戏与生成艺术社区爆红，成为"程序化生成"的代名词之一
        作者本人（Merrell）页面：WFC "reimplements" model synthesis，"nearly identical"
        ↓
2019-2023  块求解器（Boris the Brave 2021 / Kleineberg 2023）
        为无限世界：世界铺在固定周期背景上，块解不出来就保留背景
        ↓
2026  ★ 理论收口（Najafgholizadeh & Afsharchi）
        · 被有效性"沿某条线携带"的量 ⇒ 独立随机数生成器的违规密度 ≥ 1/r 阶
        · 半径开到无穷也不行（i.i.d. 的因子 ⇒ 该量处处同值）
        · 要变化就必须"沿该方向共享随机性"；周期必须整除分块周期
        · 只读例子就能算的【证书】：52 个公开例子里 certify 13 个，其余 31 个一个都没有
```

**这条线上最重要的一次"名实分离"**：2016 年之后大家说"WFC"，说的其实是 2007 年的 model synthesis。**名字扩散得比归属快**——这是技术史里常见的一种失真，值得在引用时写清楚。

## Important Papers

- [[Merrell 2007 — Example-Based Model Synthesis|Merrell 2007 — Example-Based Model Synthesis]]——**算法本体**（邻接表 / 全局传播 / 切块回滚 / NP-完全证明）。★ 2026-11-04 入库
- [[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share|Najafgholizadeh 2026 — What Local Generators Must Share]]——**理论**（$1/r$ 地板 / 必须共享什么 / 证书 / 45-of-52 实测）。★ 2026-11-04 入库
- **Gumin 2016（记名，非论文）**：`github.com/mxgmn/WaveFunctionCollapse`，单头文件 C#。**让这条线进入工业界的实际那一步**，但不是同行评议文献；
- **Merrell 2010（记名）**：*Model Synthesis: A General Procedural Modeling Algorithm*，TVCG——期刊完整版；
- **Boris the Brave 2021 / Kleineberg 2023（记名）**：块求解器，无限世界的工程实现；IW 2026 的理论对象就是它们。

## Related Concepts

- [[Procedural Content Generation]]——上位概念。**WFC 是"约束求解线"的代表，与"文法线"（形状文法 / CGA）并列**；
- [[2026-10-20-Ma — WorldStream Streaming Interactive 3D Worlds with Hierarchical Planning and Persistent State|WorldStream]]——"边界承诺"：相邻瓦片共享一条 boundary record。**本文补的是它的能力上限**：边界能承诺不变量，承诺不了相位（相位要靠周期相容）；
- [[2026-09-27-Plate-Local River Generation]]——"确定性域划分"路线：靠"读位置"逃出地板，代价是只有一个世界；
- 图搜索 / CSP——WFC 在算法层面就是"弧相容 + MRV 贪心分支"。

## Technologies

- **参考实现**：`mxgmn/WaveFunctionCollapse`（C#）；社区移植（JS / Python / Rust / Unity / Godot）大量存在；
- **引擎内**：Unity 的 tilemap + WFC 插件、Godot 的 WFC 插件；**UE 没有官方 WFC**——UE 的 `PCG` 是逐点独立求值，**按 IW 2026 的分类属于"独立随机数生成器"，对有被携带的量的图案结构性出缝**；
- **Houdini**：可用 solver / for-each 手工实现"传播 + 回滚"，`for each` + 沿线的 ID 属性就是"共享随机性"的手工版；
- **块求解器**（无限世界）：把世界铺在固定周期背景上，按块重解——工程实现见 Kleineberg 2023 一类。

## Game Applications

- **关卡 / 地牢 / 室内**：tiling 模型的主力场景（约束松、可控性强）；
- **建筑与城市填充**：用 modular kit 当瓦片，邻接规则当"什么能接什么"；
- **纹理 / 贴图合成**：overlapping 模型的本行（它就是纹理合成的 3D 推广）；
- **大世界流式生成**：块求解器 + 固定背景——**这是 WFC 进开放世界的现有形态，也是它的能力边界**；
- **编辑器内的"再生成一块"**：注意 [[2026-10-31-Najafgholizadeh — Infinite Worlds from One Example What Local Generators Must Share|IW 2026]] 的 reach = 804 格——**"再生成一块"在数学上成立，在"只影响这一块"的意义上不成立**。

## Personal Knowledge

- `user_level: Normal`（推断）：PCG 概念本身 Normal，模块化资产 / Houdini / 关卡生成的工作经验在，**但"约束求解"这一层是新的**——库里此前只有文法线，没有约束线；
- **库内真实交叉点**：NGR 这类开放世界的**分块流式**与 WFC 的"块求解"是同一个问题；本文给的"周期必须整除分块周期"是**今天就能拿去核对**的一条；
- **另一个交叉点**：你在 VFX 里熟悉的"随机感 vs 可预算"张力，在 PCG 里有严格版本——**"多个世界"和"每格独立"在数学上不能兼得**（IW 2026 Theorem 2）。

## Learning Gap

- **无硬缺口**（结论层可读）。要深入的话，两个方向：
  - **形式化层**：$K$-valid、transported quantity、i.i.d. 的因子 → IW 2026 §2–§3（Hard，需测度论直觉）；
  - **算法层**：弧相容 / MRV / 回溯的经典 CSP 教材章节 → 不需要读论文，任何一本 AI 教材的 CSP 章就够。

## Next Step

1. **（10 分钟，可立刻做）** 把项目里任意一套 modular kit 的"邻接表"写出来——**如果写不出来，说明这套 kit 的生成不可能自动化**（Merrell 的 $T$ 是唯一输入）；
2. **（20 分钟）** 如果手上有分块生成的地图：量一下**图案周期**（砖缝 / 栅栏 / 铺装的重复长度）与**chunk 尺寸**，看前者是否整除后者。**不整除就是必然出缝**，与算法无关；
3. **（观察项）** 若考虑引入学习型生成器做无限世界：先跑"局部有效斑块的重叠图"检查（IW 2026 §4 的证书）——**certified 就别指望独立噪声**，那笔钱会白花。
