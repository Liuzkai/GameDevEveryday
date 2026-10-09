---
type: paper
title: "emg2face: Expressive Facial Animation with High-Density Surface EMG"
authors: [Ganidhu Abey, Wendy Greening, Ashika Kamboj, Leonhard Helminger, Abhijeet Ghosh, Karel Petranek, Sergio Orts Escolano, Dinesh K. Pai]
year: 2026
published: "2026-10-07（arXiv v1, 2610.09304；公告 10-08）"
venue: "arXiv Preprint（University of British Columbia × Google × Imperial College London；12 页 + 补充材料）"
url: "https://arxiv.org/abs/2610.09304"
code: ""
project_page: ""
category: [facial-animation, face-capture, emg, blendshape, vr]
importance: B+
historical_importance: 0
game_relevance: 3
production_readiness: "Research（自制纺织电极网格 + 自研采集链路（64 通道 HD-sEMG，2048 Hz）；作者明确：**链路非因果、约 1.3 s 延迟——可用于交互性高的部分场景，不可用于实时动画**；电极接触质量是主要失效源）"
user_level: Normal（结论层）
status: unread
aliases: [emg2face, HD-sEMG, 高密度表面肌电, 非光学面部捕捉, EMG face capture]
tags: [facial-animation, capture, emg, vr, blendshape]
---

# emg2face: Expressive Facial Animation with High-Density Surface EMG（Abey et al. 2026）

> **入库 2026-10-09（Run 31）。** **UBC（Dinesh K. Pai 组）× Google × Imperial College London**（Sergio Orts Escolano / Abhijeet Ghosh / Karel Petranek 等）。
> **一句话定位**：给面部管线补上"**数据层**"的**非光学新通道**——用 **64 通道高密度表面肌电（HD-sEMG）**直接读面部肌肉电活动，驱动 GNM 头模型（253 身份 + 383 表情 blendshape）。**脸被头显遮住、摄像头看不到时，它照样工作**——而光学方法的视线在 HMD 面前恰好是断的。
> **库内位置**：[[Facial Animation]] 概念"数据"层的第一个独立节点（此前只有"记名"）；与 [[2026-10-06-PDB — Point-Based Deformation Blending for Facial Animation Retargeting|PDB]]（重定向层）构成"捕捉 → 重定向"的前后链。

## TL;DR

**把"看脸"换成"读肌肉"。** 64 通道 EMG（额 32 + 颊 32，纺织电极网格，2048 Hz）→ 空间编码器 + 膨胀 TCN → **每秒 100 帧预测 383 个表情 blendshape 权重** → 用**标准实时 blendshape 动画方法**渲染。监督信号仍来自视频（MediaPipe 478 landmark + GNM 分阶段拟合），但**推理时完全不需要摄像头**。

核心数字（25 名参与者，留出重复试次）：

| 指标 | 结果 |
|---|---|
| 与视频拟合的相关系数 | **中位 r = 0.76**（0.48–0.92） |
| 解释的表情运动方差 | 中位 **64%**（20–91%） |
| 网格顶点误差 | **中位 0.43 mm**——约为"保持平均脸"基线（0.87 mm）的**一半** |
| 眼部区域（HMD 遮挡区） | **r = 0.79，比其余区域（0.75）还好** —— 遮挡不是问题 |
| vs 线性基线（脊回归） | 23/25 参与者优于；r 0.67 vs 0.55 |
| 额头网格**单独**工作 | 仍保留 **>4/5** 的方差解释量（r 0.72 / 57% / 0.50 mm） |

两个细节实验值得单独记住：
1. **"戴睡眠眼罩"实验**：眼罩盖住额部网格后，MediaPipe 仍"看到脸"但在**眼罩上画眼睛和眉毛**（眉毛只抬 1.0 mm，真实为 2.3 mm；眼皮从不眨眼）——**光学方法在遮挡下不只是失效，而是自信地幻觉**；而 EMG 不受影响（预测眉毛抬起 1.9–2.7 mm，落在无眼罩范围 1.4–3.1 mm 内；12/12 次闭眼全部预测正确）；
2. **同步方案**：EMG 与视频两个独立时钟——**用"音频脉冲"双录对齐，亚毫秒级（0.2 ms）**。多模态采集里的经典脏活在正文里被认真解决（也点名了可复用性）。

## Problem

VR/XR 的面部捕捉有一个结构性矛盾：

- **HMD 恰好遮住上半脸**——而眼睛/眉毛是表情信息最密集的区域。头显向内的摄像头只能看到斜视、残缺的视角；
- 面向脸的摄像头**有隐私问题**，且需要把相机/光源架在离脸有一段距离的 rig 上（干扰演员与场景）；
- 更深一层：**"视频只见形变，不见肌肉"**——肌肉激活中**几乎不产生可见形变**的部分，光学方法原理上看不到（原文引用 GGS+22）。

**目标**：一条不需要视线、不需要相机、且能读到"形变之前"信号的通道。

## Historical Context

```text
面部捕捉的通道演化（库内视角）：
  marker/光学（影视传统）
        ↓
  头戴相机 rig（2017+ 消费级：ARKit 52 维 blendshape 流 → 平民化）
        ↓
  HMD 内摄像头（Quest 系；但上半脸被遮 → 只能"部分看"）
        ↓
★ 2026 本篇：HD-sEMG（非光学）——"读肌肉电活动"，
        遮挡免疫 + 隐私友好 + 能读到不可见激活
```

**历史落点**：这是"捕捉通道"维度的又一跳——**上一次是"从标记点到相机"（更便宜），这次是"从光子到电位"（换物理量）**。

## Previous Work

- **光学捕捉 / HMD 内摄像头（BWL+24, CWV+24 等）**：主流通路；本篇要解决它们**在 HMD 场景下的结构性缺口**；
- **ARFaceGeometry / MediaPipe Face Landmarker**：本篇的**监督来源**（478 landmarks）——注意角色：**光学方法从"主方法"退位为"训练时的标注器"**；
- **GNM（Google，arXiv 2607.23687）**：高分辨率参数头模型（253 身份 + 383 表情 blendshape）——本篇的输出空间；
- **EMG 面部研究（临床/神经科学侧，如 HFNA22）**：证明面部肌肉的 sEMG 可测；本篇把它推进到"驱动高分辨率 3D 动画"的工程标准。

## Core Idea

**三条主线的合流**：

```
① 传感器：高密度表面 EMG（64 通道，纺织网格）
        +
② 标签：视频侧的高质量参数拟合（GNM 253+383；MediaPipe 478 landmark）
        +
③ 映射：网络只学"电信号 → 表情权重"
   （空间编码器（每网格独立） + 膨胀 TCN（时间上下文）→ 100 Hz 输出 383 维）
        =
   推理时无需摄像头的面部动画链
```

**关键设计判断**：
- **"每网格一个空间编码器"**：额网格与颊网格分别编码后再融合——为"只用额网格"的降级形态留了结构（§4.3 直接做消融）；
- **输出层对齐行业接口**：预测的是 **blendshape 权重**——"can be rendered using standard real-time blendshape animation methods"。**不挑战动画管线，只喂它一个它认识的信号**；
- **训练侧用最优，推理侧用最简**：训练时不惜重工（GNM 分阶段拟合 + 双时钟同步），推理时只有一个网络。

## Technical Approach

1. **采集**：两块纺织 EMG 网格（额 32 + 颊 32 = 64 通道），2048 Hz 数字滤波；同步视频用于监督与验证；
2. **同步**：双设备独立时钟 → **模拟音频脉冲**同时被两套系统录到 → 建模时钟漂移 → **onset 对齐误差 0.2 ms**；
3. **标签生成（分阶段拟合）**：GNM 拟合 MediaPipe 478 landmark——**把头姿、眼睑闭合/注视、其余表情分开解算**（分阶段 = 把难问题拆成三个较易问题）；
4. **网络**：每网格空间编码器 + 膨胀时间卷积（TCN）→ 383 个表情权重 @100 Hz；
5. **渲染与后处理**：
   - 眼睑穿插修正（补充材料）；**次级效果**：基于**皮肤压缩量**（变形梯度 $\mathbf{F}$ 沿垂直于皱纹方向的投影）驱动 Bando 式皱纹（体积守恒：$\int S = 0$），额部三条横纹 + 鼻唇沟——**"皱纹只在皮肤被横向压缩时加深，沿纹路拉伸时不加深"**（对比：网格张力标量在各项异性形变下会互相抵消）；实测幅度 d = 1.3–1.7 mm / w = 4.5–5.0 mm。

## Key Contribution

1. **首次把 HD-sEMG → 高分辨率参数头模型（GNM，383 维）端到端打通**，给出 25 人规模的系统评估（r=0.76 / 0.43 mm / 遮挡免疫）；
2. **三项可复用的工程件**：音频脉冲同步（0.2 ms）；GNM 分阶段拟合；"空间编码器 + TCN"的通道布局；
3. **诚实的失效面**：电极接触质量是主要误差源（7 名参与者信号差 → r 0.65 vs 0.79）——**"采集时的信号质检，和网络选型一样重要"**；
4. **把限制摆在摘要外**：非因果（双向滤波 + ±1.27 s 上下文 → 动作后约 1.3 s 出结果）。

## Why It Works

- **肌肉电活动是"形变的原因"，不是"形变的结果"**——信息论上更靠近源头：不可见激活（微表情、肌肉预热）也能被读到；
- **遮挡不改变电位**：传感器贴在皮肤上，光路问题整体绕开；
- **监督来自光学、推理脱离光学**——用最成熟的光学拟合做"教师"，训练一个对光免疫的"学生"。

## Limitations

- **非因果 + 1.3 s 延迟**（原文明确："acceptable for many interactive applications, but not for real-time animation"）——要实时需因果滤波链 + 因果网络（列为 future work）；
- **需佩戴纺织电极网格**（自研硬件）——产品化路径长；脸颊网格削弱后精度下降（额头单独用：57% vs 64%）；
- **语音（唇/颌）是最弱项**：说话口型的唇颌运动被低估（与 Büchner et al. 一致）——**当前强项在眼部/眉毛（恰是 HMD 遮挡区），恰是其设计目标**；
- 评价相对视频拟合，非绝对真值（拟合本身是估计）。

## Game Development Relevance

- **VR 社交/化身**：HMD 内上半脸捕捉的**直接候选解法**（对比"向内摄像头只能看下半脸"）；与 [[2026-10-01-GALA — Gaussian Blendshape Distillation for Real-Time Avatars|GALA]]（化身表示）拼起来是"**捕捉 → 表示**"的完整 VR 化身链想象；
- **隐私友好捕捉**：无摄像头 = 合规优势（对直播/会议/化身类产品）；
- **"输出对齐标准 rig"的启示**：预测 blendshape 权重而非网格顶点——**动画管线本身不用改**；这条"接到既有接口上"与今日 [[Turquin — Practical Multiple Scattering Compensation for Microfacet Models (2019)|Turquin 的"gain to the closure"]] 同构；
- **数据层的价值不被"当前延迟"否定**：延迟在滤波与网络因果性，**是工程问题不是物理问题**——与其他"传感器换维度"的历史（RGB→深度→IMU）同一条曲线；
- ⚠️ 与 [[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards|同类"从低维代理升级"]] 的差别：那条是**管线内**升级；这条是**换物理量**。

## Unreal Engine Relevance

- 无直接 UE 映射（硬件级新通道）；若成立，落点是 **Live Link 类输入源**：把"EMG 网络输出"当作第三类面部数据源（与视频/音频驱动并列）接入 MetaHuman / ARKit 兼容 rig；
- **对分档的含义（推断）**：面部捕捉的"通道分档"可能出现：摄像头（缺遮挡区）↔ EMG 补丁（补遮挡区）——**多源融合**是比单一方案更现实的近期形态。

## Technology Evolution

```text
1970s  FACS / 参数化面部（"脸可以被编码"）
        ↓
2008+  光学捕捉 + blendshape 工业管线（影视 → 游戏）
        ↓
2017   ARKit：52 维 blendshape 流平民化（消费级捕捉）
        ↓
2020s  神经重定向（NeuralCage → … → PDB 2026）——"怎么换脸"问题
        ↓
★ 2026  捕捉通道本身开始换物理量：
        emg2face（EMG）｜同时段：PDB（重定向）已入库
        → 面部域首次呈现"捕捉 / 表示 / 重定向"三层都有节点的形态
```

## Relationships

### Based On

- **GNM（Google 参数头模型）** —— 输出空间（253 身份 + 383 表情）；
- **MediaPipe Face Landmarker** —— 训练期监督（478 landmark）；
- EMG 面部肌肉研究（临床侧文献）—— 可行性来自它们。

### Related

- [[2026-10-06-PDB — Point-Based Deformation Blending for Facial Animation Retargeting]] —— **上下游**：emg2face 产出表情参数流 → PDB 把表情"换头"（重定向）。**两者共享"身份 × 表情"分解的世界观**（GNM 的 253/383 本身就是这个分解；PDB 把同一分解做成重定向算子）；
- [[2026-10-01-GALA — Gaussian Blendshape Distillation for Real-Time Avatars]] —— 表示侧（"一套基代表全部身份"）；面部三节点（捕捉/表示/重定向）齐；
- [[Facial Animation]] —— 本节点即其"数据"层首个独立条目。

### ⚠️ 归属更正

- 库内 2026-10-08 记名时写作 "**Meta/UBC**"（Run 30 记名表）——**按 PDF 首页核实：UBC × Google × Imperial College London**；Meta 未出现（文中 Meta Quest Pro 仅作引用）。**已更正。**

## Personal Knowledge State

- **user_level: Normal（结论层）**——系统性论文、结论无门槛；推导（TCN/拟合细节）不需要；
- **读法建议（≈15 分钟）**：§4.1 主表（r/方差/顶点误差）→ §4.2 眼罩实验（最生动）→ §4.3 额网格单独 → §5 限制段。

## Learning Value

1. **"换物理量"型创新的样本**：同一任务（面部捕捉），当旧通道的系统性缺陷（遮挡 × 隐私 × 不可见激活）无法在旧物理量内解决时，**换通道**比继续优化通道更值；
2. **"训练用最优、推理用最简"的链路设计**（监督器可以很重，推理必须很轻）；
3. **实验设计的可取处**："眼罩实验"把"光学失败 vs EMG 不失败"做成了一次直观对照（还顺带暴露"幻觉式失败"这个光学方法的隐藏违规模式）。

## Visualization

（本节点暂不新增图解——其"通道对照"（光学 vs EMG 的失效模式）与 GNM 分解已在文中表格与 [[Facial Animation]] 概念中表达。）

## Notes

- **采集质量即上限**：7/25 参与者因电极接触差而精度显著下降——"quality check at capture time is as important as the choice of network"（原句大意），与"管住输入"家族同调；
- **对遮挡的精确表述**：遮挡下光学方法并非"没输出"，而是**继续输出看似合理的错误**——评估任何 HMD 面部方案时，值得把这个失败模式当成检查项。
