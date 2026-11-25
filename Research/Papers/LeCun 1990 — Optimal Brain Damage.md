---
type: paper
title: "Optimal Brain Damage"
authors: ["Yann LeCun", "John S. Denker", "Sara A. Solla"]
year: 1990
published: "1990"
venue: "NIPS 2（Neural Information Processing Systems 2），pp. 598–605"
url: "https://papers.nips.cc/paper/1989/hash/6c9882bbac1c7093bd25041881277658-Abstract.html"
code: ""
project_page: ""
category: ["Pruning", "Model Compression", "Second-Order Methods", "Classical"]
importance: A
historical_importance: 5
game_relevance: 3
production_readiness: "Industry Adopted（思路）；原文方法本身已被后续几十年取代，但"用损失增量定义重要性"这条至今是默认语言"
user_level: Normal
status: read
aliases: [OBD, Optimal Brain Damage, 最优脑损伤, 显著性剪枝, saliency pruning, 对角近似]
tags: [classical, pruning, compression, second-order, importance, budget]
---

# Optimal Brain Damage — "删掉它有多疼" 这个定义，以及它在哪里失效

## TL;DR

**"一个参数有多重要 = 删掉它之后损失涨多少"——这句话是这篇 1990 年的论文定下来的。** 今天几乎所有剪枝（网络权重、3D 高斯、网格顶点、灯光、样本）都在用它，只是很少人回头读它当时写的三个前提。

三个前提是：**极值**（在极小点，一阶项为零）、**二次**（三阶以上丢掉）、**对角**（**交叉项丢掉**）。前两个是技术性近似，**第三个是立场**：它假设"删一个的代价"可以相加成"删一批的代价"。

**而作者自己在同一页写下了这个立场的失效点：**

> 删掉约 **800 个参数（约 30%）** 之内，预测与实际吻合；**再往后，"非对角项随着删除数量的增加而变得不成比例地重要"**。

这句话在 1990 年是"承认近似有边界"，在 2026 年是**两篇前沿论文的研究前提**（见 [[2026-11-25-Jeong — LMnOP Pruning 3DGS via Leave-Many-Out Importance|LMnOP]]）。**一篇论文最耐读的部分，常常是它自己标注的失效点。**

## Problem

1989–1990 年的处境：网络越做越大，参数越多越容易过拟合；理论上（Denker 1987 / Baum–Haussler 1989）已知"给定训练量，参数太多的网络泛化差"。于是需要一个"**在保持性能的前提下去掉参数**"的办法。

当时的主流办法有两类：

| 办法 | 做法 | 问题 |
|---|---|---|
| **非比例权重衰减 / 门控系数** | 训练时让小权重的衰减更快（Rumelhart 1988 / Chauvin 1989 / Hanson–Pratt 1989 / Mozer–Smolensky 1989） | 要调"剪枝系数"，调不好会灾难性崩塌；且明显拖慢训练 |
| **按幅值删** | 训完，删最小的那些权重 | 隐含假设 **"幅值 = 重要性"** |

本文要打的正是第二条：

> "One of the main points of this paper is to move beyond the approximation that **'magnitude equals saliency'**, and propose a theoretically justified saliency measure."

**注意这句话的形式**：它不是"我们的方法更好"，而是"**你们在用的是一个没有理论依据的代理量**"。这种"把行业默认的代理量点名"的写法，是本库很在意的一类论文价值。

## Historical Context

```text
1986  反向传播普及 ⇒ 网络变大 ⇒ "该留多少参数"成为问题
  ↓
1988–89  权重衰减 / 门控系数：把"删"塞进训练过程（要调系数、拖慢训练）
  ↓
1990  ★ OBD：把"删"拿出来当独立的【选择问题】，用二阶信息定价
  ↓        · 同时给出了"重要性"这个词的现代定义
  ↓        · 同时承认了对角近似的失效点（30%）
1992  OBS（Hassibi–Stork）：用 H⁻¹ 补偿"其余权重会被重新优化" ⇒ 交叉项进闭式
  ↓
2016+  深度网络剪枝复兴（稀疏化 / 结构化剪枝 / 彩票假说）：重要性打分仍是默认入口
  ↓
2024–26  3DGS 剪枝：把同一个词搬到高斯上（LightGaussian / PUP / GaussianPOP / LMnOP）
```

**为什么"二阶"是当时的自然选择**：一阶（梯度）在极小点附近为零，给不出"删了会怎样"的信息；**要判断"离开极小点往哪个方向最便宜"，只有曲率能回答。** 这也解释了为什么"重要性"从一开始就与 Hessian 绑在一起——**后来所有"用一阶代理替代二阶"的做法（梯度幅值、激活值、混合权重），本质都是在省这笔矩阵钱。**

## Core Idea

**定义**：

$$s_k \;=\; \Delta E\big|_{\text{delete }u_k}$$

即"删掉参数 $u_k$ 后目标函数的增量"。**定义很干净，但不可计算**——作者原话："It would be prohibitively laborious to evaluate the saliency directly from this definition, i.e. by temporarily deleting each parameter and reevaluating the objective function."

> **这是全文最重要的一次分裂：定义是"真删一遍"，实现是"局部模型预测"。** 三十六年后的 [[2026-11-25-Jeong — LMnOP Pruning 3DGS via Leave-Many-Out Importance|LMnOP]] 面对的仍是同一次分裂——它真的去积"删一批"的期望，但仍然是在**一个代理损失**上积的。

**三阶泰勒**：

$$\delta E=\sum_i g_i\,\delta u_i+\frac12\sum_{i,j}h_{ij}\,\delta u_i\delta u_j+O(\delta u^3)$$

**三个近似，逐条砍掉一项：**

| 近似 | 假设 | 砍掉什么 | 什么时候会坏 |
|---|---|---|---|
| **极值（extremal）** | 删除发生在训练收敛之后 | 一阶项 $\sum g_i\delta u_i$ | 没训到位 / 学习率还很大时 |
| **二次（quadratic）** | 代价面近似二次 | 三阶及以上 | 删掉**大值**参数时 |
| **对角（diagonal）** | 各参数的 $\delta E$ 可相加 | **交叉项 $h_{ij},i\ne j$** | **删得多的时候**（作者自己标注：~30%） |

于是：

$$\delta E=\frac12\sum_i h_{ii}\,\delta u_i^2 \quad\Longrightarrow\quad s_k=\frac12 h_{kk}u_k^2$$

**这个式子值得多看一眼**：它把"重要性"分解成两个因子的乘积——**曲率 $h_{kk}$（这个方向有多陡）** 与 **位移平方 $u_k^2$（你走多远）**。这也解释了"幅值"为什么是个还不错的代理：**它只取了后一个因子。** 而两者在统计上的相关性，正是"幅值法居然能用"的原因，也是它在大参数上失效的原因（大 $u$ 未必对应大 $h$）。

### 怎么算 $h_{kk}$：和梯度一样便宜

关键是这条工程结论：

> "computing the **diagonal Hessian is of the same order of complexity as computing the gradient**."

做法是把二阶导沿层反传（式 5–8），输出层的边界条件是

$$\frac{\partial^2 E}{\partial a_i^2}=2f'(a_i)^2-2(d_i-z_i)f''(a_i)$$

**并且给了一个"保证正定"的近似**：丢掉含 $f''$ 的第二项，就是 Levenberg–Marquardt 近似（$h_{kk}\approx 2f'^2\sum w^2\cdots$）。**这一步很实用**：极值点附近 Hessian 应半正定，但数值上会出现负的 $h_{kk}$，丢掉第二项就永远为正。

> **可迁移的一条**：**"二阶信息贵"这个印象是错的——贵的是完整 Hessian，不是它的对角。** 对角只要一次改写的反传。本库在 [[Multigrid]] / [[Contact Dynamics]] 里反复遇到"要不要用二阶"，答案在这里有一个历史锚点：**先用对角，再决定要不要逆。**

## Technical Approach（六步配方）

1. 选一个合理的架构；
2. 训到合理解；
3. 算每个参数的 $h_{kk}$；
4. 算显著性 $s_k=\frac12 h_{kk}u_k^2$；
5. 按显著性排序，删掉最低的一批；
6. 回到 2，迭代。

**"删"的定义是"置零并冻结"**（不是"允许它再长回来"）。作者提到几种变体：把小显著性参数**衰减**而不是直接置零、或者允许被删参数**重新适应**——**这两种变体在今天的语言里就是"软剪枝"与"剪枝后微调"。**

## Key Contribution

**实验载体**：手写数字识别（邮编数字 + 印刷数字），**105 条连接 / 2,578 个自由参数**，9,300 训练 / 3,350 测试。

| 结果 | 数字 |
|---|---|
| 按显著性删 vs 按幅值删（Fig 1a） | 同样参数数量下，**OBD 的目标函数显著更低**（同一横坐标，OBD 曲线在下方） |
| 随机删 | **"performance was so bad that the curves cannot be shown on the same scale"** |
| 预测 vs 实际（Fig 1b） | **约 800 个参数（~30%）之内吻合**，之后分离 |
| 重训后（Fig 2） | **删掉 1,500 个（60%）**，训练集与测试集性能"几乎不变" |
| 交互式使用 | 直接砍掉倒数第二层 ⇒ 参数减半，训练 MSE ×10，**泛化 MSE 只 +50%**，测试集分类错误率**反而下降** |
| 总计 | 交互式减到 1/4，自动 OBD 再减一半以上；**速度显著提升、精度略升** |

**Fig 1b 的分离原因是全文最值钱的一句解释**（作者给了两条，缺一不可）：

1. **非对角项**随着删除数量增加而"不成比例地重要"——因为对角近似把"同时删掉互相补偿的一群"当成了"独立地删每一个"；
2. **三阶以上项**在删掉**大值**参数时变重要——因为 $\delta u$ 大了，泰勒展开的高阶项不再可忽略。

> **两条原因对应两种不同的修法**：第 1 条要靠"考虑交互"（OBS 的 $H^{-1}$、LMnOP 的联合期望），第 2 条要靠"减小步长"（多阶段、每次少删一点）。**今天所有剪枝方法，基本都落在这两个修法里，或者两者都用。**

## Why It Works

**① 在极小点附近，损失面确实是二次的。** 这不是假设的胜利，是"训练收敛"这个前提的胜利——**OBD 的第一条前提（extremal）是整个方法的地基**，一旦你在训练中途剪枝，一阶项就回来了。

**② 对角近似在"稀疏删"时几乎无害。** 删一个参数时，交叉项 $h_{ij}\delta u_i\delta u_j$ 里只有一个 $\delta u$ 非零 ⇒ **交叉项严格为零**。所以 LOO 的重要性在"只删一个"时是**精确到二阶**的。**这不是近似，是恒等式。**

**③ 它给了"重要性"一个可比较的单位：损失。** 幅值、激活、梯度都只是代理量，彼此不可比；**"损失增量"可以跨层、跨类型相加**——这正是它能活三十六年的原因。**代价是它把"预算"绑在了"损失"上，而损失不一定是你要交付的东西**（见 [[2026-11-25-Oh — DiffSimp Differentiable Mesh Simplification|DiffSimp]] 的 raw vs deployed）。

## Limitations

- **对角近似的失效点是实测的 30%，不是理论的。** 作者没有给"为什么是 30%"的解释——它取决于具体网络的曲率结构。**换个任务，这个数字会变。**
- **只处理"置零"这一种删除。** 结构化删除（整通道 / 整层）不在框架里；后来的结构化剪枝要另建重要性（group saliency）。
- **需要收敛 + 需要重训。** 六步配方里第 6 步是"回到 2"——**没有重训的 OBD 是不完整的**（Fig 2 的两条曲线差得很远）。
- **规模**：2,578 个参数。作者自己提到完整 Hessian 是 $6.5\times10^6$ 项——**"矩阵太大"这件事从第一天就是障碍**，也是后来所有近似（对角、Kronecker、低秩）的动机。
- **"最优"是名义上的。** 名字叫 Optimal Brain Damage，实际是"贪心 + 对角近似 + 迭代"，没有任何最优性保证。**作者没有过度声称，是后来的引用把它当成了"最优"。**
- **分类错误率下降这件事不能当普遍结论。** 作者自己标注了"which indicates that MSE is not the optimal objective function for this task"——**它说明的是目标函数选错了，不是剪枝有益。**

## Game Development Relevance

**它不是图形学论文，但它是"预算下选子集"这件事的语法书。** 今天的游戏管线里，同一句话被反复使用：

| 场景 | 元素 | "损失"是什么 |
|---|---|---|
| 网格简化（[[Mesh Simplification]]） | 边 / 顶点 | 二次型误差（QEM 就是"到一组平面的距离平方和"——**和 $s_k=\frac12h_{kk}u_k^2$ 是同构的**） |
| 3DGS 剪枝（[[Gaussian Splatting]]） | 高斯 | 渲染损失（[[2026-11-25-Jeong — LMnOP Pruning 3DGS via Leave-Many-Out Importance\|LMnOP]] 就是 LOO 的现代版） |
| 灯光预算（[[Many-Light Rendering]]） | 灯 | 辐照度误差（Lightcuts 的 2% 界） |
| 采样预算（[[Reservoir Sampling]]） | 样本 | 方差 |
| LOD 选择（[[Level of Detail]]） | 表示 | 屏幕空间误差 |

**四条可搬到引擎里的判断：**

1. **"幅值 = 重要性"这个代理，在任何域里都只在"曲率与幅值统计相关"时成立。** 换成高斯：**不透明度大 ≠ 重要**（一片大而不透明的高斯可能完全被遮挡）。这也是 [[2026-11-25-Jeong — LMnOP Pruning 3DGS via Leave-Many-Out Importance|LMnOP]] 里 `Opacity` 基线只有 15.36 dB 的原因。
2. **二阶对角很便宜，别怕它。** "要算曲率"不等于"要算 Hessian"。
3. **预算越紧，独立打分越错**——30% 是 1990 年那台网络的数字，但它给的是一个**可检验的预言**：任何"逐个打分 + 取 top-K"的管线，都应该在**高压缩比**下重新验证一次。
4. **"删了再重训"是配方的一部分**，不是可选项。**没有重训的剪枝结果，不能代表这条方法的真实水平**（这条对今天的 GS 剪枝同样成立——LMnOP 也做了"剪后微调 5,000 步"的对照）。

## Unreal Engine Relevance

**间接，但有一条很实在**：

- **任何"预算器"（LOD 生成、粒子上限、灯光上限、贴图流送）都在隐式地用一个"重要性"函数。** 这篇论文的价值是提醒你：**那个函数是"损失增量"还是"某个幅值代理"？** 如果是后者，**在预算收紧时要重新标定。**
- **不要用"数量"当验收指标**（与 [[Budget Units]] 同一条）：OBD 的 Fig 1a 就是"同样的参数数量，两种选法差出一大截"的最早图示。

## Technology Evolution

```text
1990  OBD：定义 + 对角近似 + 30% 失效点
  ↓
1992  OBS：H⁻¹ 补偿交互（二次型世界里的闭式 leave-many-out）
  ↓
2015+  深度剪枝：importance 变成工程流水线的一环（稀疏训练、结构化剪枝）
  ↓
2020s  "数据归因 / Shapley"：把"一个样本值多少"也变成同一个问题
  ↓
2026  LMnOP：在乘性遮挡世界里重写 —— 没有 H⁻¹，改用"随机掩码的期望 + 闭式方差"
```

**这条线的方向很清楚：从"假设独立"走向"承认交互"，而每走一步，代价都从"一次反传"涨到"多次全模型评估"。** 这是 [[Budget Units]] 里"换单位要付转换费"的另一种形态：**换精度也要付转换费。**

## Relationships

### Based On

- **反向传播（Rumelhart–Hinton–Williams 1986，库外）** —— 式 5–8 的"二阶反传"是反向传播的直接改写。
- **Levenberg–Marquardt 近似（库外）** —— 丢掉 $f''$ 项换"保证正定"。

### Extends

- **权重衰减 / 门控系数那一族（1988–89，库外）** —— 把"删"从训练过程里拿出来，变成一个独立的、可定价的选择问题。

### Related

- [[2026-11-25-Jeong — LMnOP Pruning 3DGS via Leave-Many-Out Importance|LMnOP 2026]] —— **同一条定义在 3DGS 上的重写**；也是"对角近似失效"这件事的现代解法。
- [[2026-11-25-Oh — DiffSimp Differentiable Mesh Simplification|DiffSimp 2026]] —— 反方向：不问"删谁"，问"删完之后怎么把位置调好"。**两者合起来才是完整的选择问题：选子集 + 调剩下的。**
- [[Budget Units]] —— "损失增量"是一种可以跨模块相加的预算单位，这是它最大的优点。
- [[Mesh Simplification]] —— QEM 的代价 $E_e(v)=\tilde v^\top Q_e\tilde v$ 与 $s_k=\frac12h_{kk}u_k^2$ 是同一个东西：**二次型 + 位移平方**。
- [[Subset Selection]] —— 本文是该概念的起点。

### Contrasts

- **"幅值法"** —— 本文的直接对手，也是它自己的"零阶"版本：只取 $u_k^2$，不要 $h_{kk}$。**Fig 1a 是这两者的第一次正面对比。**

### Followed By

OBS（1992）→ 稀疏训练 / 结构化剪枝（2015+）→ GS 剪枝（2024–26）。

## Personal Knowledge State

- **Normal**：定义、三个近似、六步配方、$h_{kk}$ 与梯度同阶——这些今天读一遍就能讲清。
- **需要记住的是"作者自己标注的失效点"**：30%，以及那两条原因（非对角项 / 高阶项）。**这是本库"判据要带适用域"这条规矩的最早样本之一。**
- **Easy（已有）**：QEM 的二次型代价、剪枝后重训——这两块在 [[Mesh Simplification]] 早已覆盖，不重复教学。

## Learning Value

**三个可迁移动作：**

1. **任何"重要性/优先级"函数，先问它用的是"损失增量"还是"幅值代理"。** 若是代理，**在高压缩比下重新标定一次**。
2. **"对角近似"是一个可以主动使用的工具**：它在"只删一个"时**精确**，在"删一批"时有偏——**所以它天然适合"多阶段、每阶段少删"的流程**（这正是 GaussianPOP 的 8 阶段做法）。
3. **读经典要看它标注的失效点，而不是它宣称的结论。** 这篇的 30% 比它的 60% 更有用：**前者是"什么时候要换方法"，后者只是"这个方法能走多远"。**

## Visualization

![[Research/Files/Pictures/LeCun 1990 — 图1 显著性删除与幅值删除对比.png]]

**读图**：横轴是**剩余参数个数**（从右往左删），纵轴是 $\log$ MSE（dB）。**(a)** 两条曲线：上面是"按幅值删"，下面是"按显著性删"——**同样的剩余参数个数，按幅值删的损失高出一大截**，而且差距在删到 1,000 附近最大（约 4 dB）。这就是"幅值 ≠ 重要性"的第一次图示。**(b)** 两条线：带点的是**实测**，直的是**预测**（各被删参数显著性之和）。**看它们在 1,500 左右分开**——**预测低估了损伤**，作者给的两条原因是"非对角项"与"高阶项"。**这张图的右半边，就是 2026 年 LMnOP 要修的那个洞。**（图 1，LeCun et al. 1990）

## Notes

- 原文 PDF 已归档：`Research/Files/PDF/LeCun 1990 - Optimal Brain Damage.pdf`（1.7 MB，8 页，扫描件 + OCR 层，公式排版有噪声，读数时以图为准）。
- **引用格式**：NIPS 2 (1989 conference, published 1990), pp. 598–605。库内统一记作 **1990**。
- **名字的来源**：删权重会"损伤"网络，OBD 求的是"**最优的损伤方式**"——不是"无损伤"。这个名字在后来被大量误读成"最优剪枝"。
- **与 LMnOP 的一处精确对应**：OBD 的 $s_k=\frac12h_{kk}u_k^2$ 只在 $\delta u_k=-u_k$（一次删到零）且**其他都不动**时成立——这就是 leave-one-out。**LMnOP 的贡献可以一句话概括：把"其他都不动"换成"其他按概率动"。**
- **⚠️ 未入库（下一步值得补）**：Hassibi & Stork 1992 *Second Order Derivatives for Network Pruning: Optimal Brain Surgeon*（NIPS 5）——它是"交互"这一支的起点，公式 $L_q=\frac12 w_q^2/[H^{-1}]_{qq}$ 我**没有读原文核对**，本文中按 LMnOP 的转述引用。

**相关每日记录**：[[2026-11-25]] ｜ **概念**：[[Subset Selection]]
