---
type: paper
title: "Towards a Perceptual Evaluation Framework for Lighting Estimation"
authors: ["Justine Giroux", "Mohammad Reza Karimi Dastjerdi", "Yannick Hold-Geoffroy", "Javier Vazquez-Corral", "Jean-François Lalonde"]
year: 2023
published: 2023-12-07
venue: "arXiv 2312.04334v3（2024-03-20 修订）；Université Laval + Adobe Research + UAB；是否正式会议收录未核"
url: "https://arxiv.org/abs/2312.04334"
code: "https://lvsn.github.io/PerceptionMetric/（感知数据 + 学习指标代码，公开）"
category: ["perceptual-evaluation", "lighting-estimation", "metrics", "user-study", "relighting"]
importance: A-
historical_importance: 0
game_relevance: 4
production_readiness: "Tool（公开数据集 + 可训练的评价指标，可直接用作拟合工具的验收器）"
user_level: Normal
status: unread
---

# Towards a Perceptual Evaluation Framework for Lighting Estimation

## TL;DR

**你用 SSIM 优化出来的"像"，美术可能根本不认——这篇用 49 人的受控实验把这件事量化成了硬数据。**

光照估计领域一直在用 IQA 指标（PSNR / SSIM / LPIPS / FLIP…）排名算法。这篇做了一个心理物理实验：让真人做二选一（2AFC），再把 15 个常用指标逐个对答案。结果是**一盆冷水**：

- 在**高光材质**或**无参照"看着真不真"（plausibility）**任务上，**大多数标准指标与人类判断的一致率≈随机（0.5）**；
- 只有最简单的情况（漫反射 + 有参照的 accuracy 任务）指标才勉强可用；
- 把 15 个指标**学习组合**（SVR）后，一致率拉到 **0.889**（task 2 glossy）——并且能泛化到训练时没见过的光照估计算法。

一句话对读者：**你的"旧观感 → 新灯光"拟合工具，验收那一格（⑤号槽位）不能只用 $L_2$/SSIM——这篇给出了量化证据，也给了一个可以训练出来的替代品。**

![[Research/Files/Pictures/Giroux 2023 — 图1 人类偏好与IQA指标判断分歧.png]]
**读图**：三组对比，每组两张"同一个虚拟球用不同估计光照渲染"的图。`Humans` 行是真人的选择（绿心=偏好），`Metrics` 行是标准 IQA 指标的选择，`Ours` 行是本文学习指标的选择。**三组的 Metrics 全部与 Humans 相反，Ours 全部一致**——整张 teaser 就是结论的微缩：标准指标在"哪个光照更对"上会系统性站错队。（图 1，Giroux et al. 2023）

## Problem

光照估计（从单图恢复场景光照）是 AR/视效的基础件，领域进步一直靠"在标准数据集上算 IQA 指标"来跟踪。论文要问的是一个更根本的问题：**这些数字指标跟人眼觉得"像不像/真不真"到底对不对得上？**

动机里有一个实践视角：实际应用中用户**没有 ground truth**——他们只需要虚拟物体放进照片里"看着可信"。所以评估必须覆盖两种任务：

- **Task 1 — Accuracy**：有参照图，哪个渲染的光照更接近原图；
- **Task 2 — Plausibility**：无参照，哪个渲染在场景里看着更真实。

**这两个任务后来被证明调用的是不同的视觉线索**——这是全文最重要的细分。

## Historical Context

```text
2018     LPIPS：学习式感知指标取代 PSNR/SSIM 成为"像不像"的默认答案
2020s    光照估计算法爆发（学习式为主），排名全靠 IQA 指标
2023  ★  本文：第一次对"光照估计的评估指标"本身做受控心理物理审计
         ↓
      结论：指标与感知脱节，且脱节程度随材质与任务类型变化
```

与本库 [[Fdez-Agüera — A Multiple-Scattering Microfacet Model for Real-Time Image-based Lighting (2019)]] 一类"渲染正确性"工作正交：那一支问"渲染对不对"，这一篇问"**我们说它像的时候，人同不同意**"。

## Previous Work

被审计的指标共 15 个，分三类（论文 §5）：

| 类别 | 指标 |
|---|---|
| 全参照（有 GT 图） | RGB 角误差、PSNR、RMSE、si-RMSE、SSIM、VIF、ΔE、LPIPS、PieAPP、FLIP、HDR-VDP |
| 无参照 | BRISQUE、NIQE、UNIQUE、HyperIQA |
| 本文学习组合 | SVR 聚合上述全部 |

被评估的光照估计算法：室内 5 种、室外 3 种（选自近年文献，含参数化光源模型与 HDR 贴图估计两条路线）。

## Core Idea

### 受控 2AFC 实验设计

- **刺激**：50 张 HDR 全景图（25 室内 + 25 室外）——用一阶球谐系数 k-means（k=25）聚类选场景，保证光照多样性；每张全景渲出**漫反射球与高光球**两个版本；
- **配对**：室内 5 算法 × 25 场景 = 250 图；室外 3 算法 = 75 图；顺序与左右位置随机化；
- **被试**：**49 人**（33 男 / 16 女，24–63 岁），全部过石原色盲测试；室内实验实际完成 30/31 人（diffuse/glossy）；
- **环境**：受控实验室，哑光黑房间，显示器 sRGB，显示器为唯一光源；观察者距屏约 70 cm（视角 11.5°/17°）；
- **分析**：Thurstone Case V 比较判断模型把 2AFC 计数转成 z-score 排名；指标得分与人类偏好算**一致率（agreement）**，随机水平为 0.5。

### 学习指标

把 15 个指标的输出当特征，用支持向量回归（SVR）拟合人类偏好。用**留一法**验证泛化：训练时扣掉某一类算法，测试时看它能否预测该类算法上的人类判断。

## Technical Approach

主要结果（图 5 / Table 1，数字均来自原文）：

![[Research/Files/Pictures/Giroux 2023 — 图5 各指标与人类判断一致率.png]]
**读图**：四个面板 = 任务（task 1 accuracy / task 2 plausibility）× 材质（diffuse / glossy）。橙柱室内、蓝柱室外，y 轴是与人类判断的一致率，0.5 即随机。三条参考线：Perfect=1、Random=0.5、Expected Observer（人类自身的预期一致性）。**读法：左边 15 组（RGB Ang. Err. 到 HyperIQA）大多贴着 0.5 上下，尤其 task 2 glossy 几乎全军覆没；最右两组 Ours / Ours Holdout（带星号）在全部四个面板里都显著高于其他指标。**（图 5，Giroux et al. 2023）

- 学习指标一致率：task 1 漫反射/高光 **0.786 / 0.856**；task 2 **0.787 / 0.889**；
- 最好的单个标准指标（VIF、NIQE、RGB 角误差、BRISQUE，因任务而异）只有 **0.670/0.809**（task 1）与 **0.686/0.691**（task 2）；
- "metrics on average performing around chance level"——标准指标平均在随机水平附近；
- **泛化成立**：holdout（训练集不含的算法）上一致率仍然最高，包括 Stable Diffusion outpainting 类的新算法；
- 任务分解结论：**参数化光源模型对漫反射球够用，对高光球失败**——高光需要高频 HDR 贴图，而这个差别恰好是简单指标抓不住的。

## Key Contribution

1. **第一次用受控心理物理实验系统审计光照估计的 IQA 指标**（49 人、双任务、双材质、双环境）；
2. **量化证明标准指标与人类感知脱节**，且脱节模式有结构（材质 × 任务），不是均匀噪声；
3. **一个可复用的学习指标**（SVR 聚合 15 个 IQA），一致率最高 0.889，且对未见算法泛化；
4. **公开全部感知数据与代码**（lvsn.github.io/PerceptionMetric/）——别人可以直接拿它当验收器用；
5. 方法上把"评估"从**算一个数**变成**定义一个任务**（accuracy vs plausibility 分开测）。

## Why It Works

**因为"像不像"不是一个数，是一个决策——而决策依赖任务上下文。**

Accuracy 任务里人可以用"逐块对比参照"的策略，这类策略与全参照指标（SSIM 等）部分同构，所以指标在漫反射+accuracy 这个最简单格子里尚可。Plausibility 任务里没有参照，人只能调用"这个世界通常长什么样"的先验——高光反射的内容合理性、阴影方向一致性这类**全局语义线索**，而逐像素/逐块的 IQA 指标在构造上就摸不到这些线索。

学习指标之所以work，不是 SVR 聪明，而是**15 个指标的联合特征里碰巧拼出了足够的线索覆盖**——每个指标各捕获一小片感知差异，组合后逼近人的判断函数。这也意味着它的上限受特征池限制（论文没有声称超过人类自身一致性上限）。

## Limitations

- **刺激形态单一**：虚拟球（漫反射/高光两种）置于真实背景——不是完整场景重打光，结论外推到"整个关卡换光照"需谨慎；
- **被试规模** 49 人、每格实际 30/31 人，够发论文不够定国标；
- **学习指标只在"光照估计算法"的分布上验证过**——把它拿去评估"优化拟合出的灯"属于分布外使用，应重新校验（至少小规模重标定）；
- **只评估最终渲染图**，不管参数可解释性——对"迁移工具"来说，灯参数是否可编辑是指标看不到的维度；
- 场景选择用一阶 SH 聚类，**只保证低频光照多样性**，高频布光模式（多盏硬光）覆盖不明；
- 是否被正式会议收录：**未核**（arXiv v3 止于 2024-03）。

## Game Development Relevance

**这篇直接命中收集页的 ⑤ 号槽位（评估），并给出三条可执行结论：**

1. **别用单一指标验收迁移结果。** 特别是你的场景里有高光材质、或验收方式是"美术看着行不行"（=plausibility 任务）时，SSIM/PSNR 的说服力≈抛硬币。本文图 5 是可以直接拿去说服评审的证据。
2. **验收指标可以"学"出来。** 在迁移工具的场景里，可以让美术对"旧版 vs 新版"渲染对做几十组二选一，用同样的 SVR 配方训一个**项目专用的感知指标**——本文开源了数据与代码，配方可复制。这比争论"该用哪个现成指标"便宜得多。
3. **accuracy 与 plausibility 要分开验收。** "和旧版一模一样"与"在新管线里看着对"是两个任务，人类用的线索都不同。迁移工具应当同时报告两个数，否则会出现"指标过了但美术不买账"的扯皮。

与 [[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization|Lipp 2024]] 的关系：Lipp 的目标函数是辐射场系数的加权 $L_2$——按本文结论，这个 $L_2$ 做**优化目标**可以（梯度方向大致对就行），但做**验收指标**不够。优化目标与验收指标分离，正是收集页 ⑤ 号槽位预判的"最容易被忽略的一条"。

## Unreal Engine Relevance

- UE 里做 A/B 验收时，可用本文开源代码训练项目内指标；双球探针（漫反射球 + 高光球）是现成的低成本的"光照指纹"渲染件——在 UE 里渲一对材质球比渲整关便宜得多，适合放进 CI 式的自动对比；
- 注意 HDR-VDP、FLIP 等指标需要 HDR 输入，UE 侧截图要关色调映射或用 EXR 输出。

## Technology Evolution

```text
"渲染结果像不像，谁来裁决"
2004-     PSNR/SSIM：信号保真度时代
2018      LPIPS：深度特征时代——"感知"第一次被显式建模
2020-     FLIP/HDR-VDP：把显示与视觉模型做进指标
2023   ★  本文：审计时代——先问指标与人对不对得上，再谈排名
          ↓
       下一步：把"指标组合"换成"直接学习判断函数"（大模型偏好打分？
       本文的 GPT 时代后继者已在路上——见收集页 2026-11-02 记名条目 Ad-Relight 的 GPT-4o 评审）
```

## Relationships

### Related

- **[[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization]]** —— 目标函数（$L_2$ 系数差）与验收指标的分离点；Lipp 自己在 Followed By 里也猜"感知指标可能是迁移像不像的正解"，本文给了证据；
- **[[Schwengerer 2022 — Inverse Method for Baked Lighting]]** —— 其验收止步于"visually close"，本文说明这类定性验收的风险；
- **[[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]]** —— 本文选场景用的一阶 SH 聚类，正是这条谱系的工具；
- **[[Physically Based Rendering]]** —— 漫反射/高光双球设计是 BRDF 语言下的最小材质探针。

### Followed By

- 待观察：把 2AFC 人类标注换成多模态大模型评审（已有工作用 GPT-4o 做光照一致性偏好判断）是否保真；
- 待观察：感知指标作为**可微优化目标**（而不只是验收器）——LPIPS 已经可微，本文的学习指标若可微化，可直接替换 Lipp 框架里的 $L_2$。

## Personal Knowledge State

`Normal`。实验设计（2AFC + Thurstone Case V）是心理物理学标准件，读结论不需要推导；需要拿住的只有两点：① 一致率的定义与随机基线 0.5；② "指标平均≈随机"这个结论的适用边界（高光材质与 plausibility 任务，不是所有情况）。

## Learning Value

- **对工具验收**：指标选型前先做小规模人类标注校准——哪怕只有 30 组二选一，也能告诉你现成指标在你的分布上是不是随机水平；
- **对评审**：看到"我们的方法 SSIM 更高"时，追问"SSIM 在这类材质/任务上与人类判断的一致率是多少"——本文证明这个问题经常没有好答案；
- **一句话**：**优化目标和验收指标是两件事；前者要梯度良态，后者要和人一致——用一个数同时充当两者，是照明拟合工具最容易埋的雷。**

## Visualization

![[Research/Files/Pictures/Giroux 2023 — 图1 人类偏好与IQA指标判断分歧.png]]
（图 1：teaser——三组对比中标准指标全部与人类相反，学习指标全部一致）

![[Research/Files/Pictures/Giroux 2023 — 图5 各指标与人类判断一致率.png]]
（图 5：15 个标准指标 vs 学习指标在 任务×材质 四格中的一致率）

## Notes

- **数字来源**：49 名被试（33M/16F，24–63 岁）；室内 30/31 人（diffuse/glossy）；250/75 张图（室内/室外）；25 场景由一阶 SH + k-means（k=25）选取；70 cm 视距、11.5°/17° 视角；学习指标一致率 0.786/0.856（task 1）、0.787/0.889（task 2）；最佳单指标 0.670/0.809、0.686/0.691。以上均亲见原文文本层。
- **venue 未核**：arXiv v3（2024-03-20）为止，未确认正式会议/期刊收录。
- **素材版权**：图 1/图 5 提取自 arXiv PDF，仅作学习引用，版权属原作者。
