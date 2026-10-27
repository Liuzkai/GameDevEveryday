---
type: paper
title: "RADIANT: Object-Centric Feed-Forward Global Illumination"
authors: ["Sarah Zhang", "Zhiyang Dou", "Minghao Guo", "Benjamin Tod Jones", "Wojciech Matusik"]
year: 2026
published: 2026-10-23
venue: "arXiv preprint（MIT CSAIL；Preprint，未标注会议）"
url: "https://arxiv.org/abs/2610.32075"
code: "未放出（作者称接收后开源代码与数据集）"
project_page: ""
category: ["global-illumination", "neural-rendering", "transformer", "feed-forward-rendering", "point-cloud", "budget"]
importance: A
historical_importance: 3
game_relevance: 4
production_readiness: "Research（单帧 0.83 s @ 500 物体 / 512²，非实时；质量评估只到 50 个实体）"
user_level: Normal
status: unread
---

# RADIANT: Object-Centric Feed-Forward Global Illumination

## TL;DR

**它把"一个物体"当成光传输的计数单位，于是注意力长度不再跟三角形数走。** 每个物体压成 **32 个表面片 token**（2048 个点输入 → 32 个 token），场景传输的序列长度变成 $32N$（$N$ = 场景实体数），复杂度从 $O(T^2)$（$T$ = 三角形数）变成 $O(N^2)$。中位 token 数 **3,205 → 320**。

在同等参数预算（≈206M）和同一份 103 万场景数据下：**32.22 dB（per-image）/ 27.65 dB（pooled）**，对 RenderFormer 是 **+3.3 dB**、对 RenderFormer-V2 是 **+7.6 dB**；达到 25 dB 所需的优化步数是 RenderFormer 的 **1/5**（16k vs 80k）。500 物体场景一次 512² 前向 **0.83 s**，比带 FlashAttention 的 RenderFormer **快 8.5×**。

> 一句话对读者：**"换单位"的收益是 3.3 dB，"在这个单位里调预算"的收益是 0.04 dB**——见下面的 token 预算消融。这条数据比论文的主方法更值得记住（已沉淀进 [[Budget Units]]）。

![[Research/Files/Pictures/Zhang 2026 — 图1 RADIANT 架构总览.png]]
**读图**：上半（绿框，视图无关，每场景算一次）是**点输入 → Point Encoder → 每物体 32 个 patch token → 场景 Transformer 12 层**；左上的紫色窄条 `Invariant form-factor attention` 是唯一注入物理结构的地方（不是模块，是一个加在 attention logit 上的偏置）。下半（橙框，视图相关，每视角算一次）是**相机光线 token 与场景 token 做 cross-attention**，左边三张彩色点云是 point-splat 光栅化出的 G-buffer 通道（物体 id / 深度 / 法线）。右侧 Reference / Rendered 是同一场景的输入与输出。（图 1，Zhang et al. 2026）

## Problem

前馈神经渲染器（RenderFormer 系）把**每个三角形**当成一个 token，做全对全 self-attention。于是：

- 自注意力 $O(T^2)$，$T$ 随网格细分增长——**同一块墙，50 个三角形和 50,000 个三角形在物理上是同一件事，在账上是 1000 倍的差别**；
- 后续变体（RenderFormer-V2 的窗口注意力、RenderFormer++ 的物体池化）都在缓解 $T$，但**编码阶段仍要跑三角形级注意力**（RenderFormer++ 的代价写作 $O(\sum_i T_i^2)$）。

论文的立论就一句：**宏观辐射传输由表面几何与物体排布决定，不由某一个网格离散决定**。既然物理不看细分，计算也不该看。

## Historical Context

```text
1984  Goral 辐射度：把间接光写成"面片之间传能量"的矩阵（本库 [[Goral — Modeling the Interaction of Light Between Diffuse Surfaces (1984)]]）
1985  hemi-cube：把 form factor 的积分投影到固定分辨率的立方体面（可见性进 F）
1986  Kajiya 渲染方程：一切 GI 的容器（[[Kajiya — The Rendering Equation (1986)]]）
1991  层次辐射度：传输算子按重要性细分（"不是所有面片对都值得算"）
2008  Lehtinen 无网格层次表示：光照与曲面细分解耦（本库未入库，本文引用）
2019  Hermosilla：点云上的 direct-to-indirect 算子
2021  Neural Radiosity：把渲染方程残差当损失去拟合单个场景
2025  RenderFormer：三角形 token + 全对全注意力，前馈 GI 的第一个可扩展样本
2026  RenderFormer-V2（窗口注意力 + 参与介质）/ RenderFormer++（物体查询池化，但仍先跑三角形注意力）
2026  RADIANT：直接把 token 定义在"物体"上，点输入，32 token/物体
```

**它的历史位置不是"又一个 transformer 渲染器"，而是"把 1984 年辐射度的记账单位搬进 2026 年的注意力"**：辐射度的能量交换本来就发生在**有限表面区域之间**，与三角化无关——RADIANT 只是照这句话去定义 token。

## Previous Work

| 方法 | 输入 | 三角形级注意力 | 传输 token 数 | 传输复杂度 | 主可见性 |
|---|---|---|---|---|---|
| Neural Radiosity | 网格（+ 光追） | — | — | — | 需光线追踪 |
| RenderFormer | 三角形 | **全对全** | $T$ | $O(T^2)$ | 学出来 |
| RenderFormer-V2 | 三角形 | 窗口化 | $T$ | $O(T^2)$ | 学出来 |
| RenderFormer++ | 三角形 | **每物体内全对全** | $8N$ | $O(N^2)$ | G-buffer |
| **RADIANT** | **点** | **无** | $32N$ | $O(N^2)$ | **point splat** |

关键区别在最后一列之外还有一列：**编码阶段**。RenderFormer++ 已经把传输压到 $O(N^2)$，但它**仍要先对每个物体的三角形跑一遍全对全注意力**才池化成物体查询；RADIANT 从点集抽象直接得到物体 token，**一次三角形级注意力都不跑**。作者用一句话点破了"池化"与"重定义"的差别：把 RenderFormer 的三角形 token 池化成 32 个/物体，**反而让它的 PSNR 掉 0.38 dB**——说明起作用的是表示与架构，不是 token 预算。

## Core Idea

**让"计数单位"对齐"物理交换单位"。**

| 层 | 传统做法 | RADIANT |
|---|---|---|
| 表示 | 三角形（离散化产物） | 物体（物理实体）→ 32 个表面片 token |
| 传输 | token 间自注意力，几何信息靠位置编码 | token 间自注意力，**几何不变量直接进点积** |
| 可见性 | 学出来 / 光线求交 | **点云泼溅出 G-buffer**（显式、免学习） |
| 渲染 | 一次前向出整图 | 光线 token × 场景 token 的 cross-attention + DPT 上采样 |

## Technical Approach

### 1) 物体 tokenization：2048 点 → 32 个 patch token

- 每物体 **$P=2,048$ 个表面采样点**，每点 16 维（位置 3 + 法线 3 + 材质 10：漫反射/镜面/粗糙度/自发光…）；
- PointNet 风格编码器（10.0M）在**物体居中的归一化坐标系**里做 PointNet++ 集合抽象：**最远点采样选 32 个质心 + 每个 128 近邻**，局部 max-pool + 线性投影 → 32 个宽 1,024 的 patch token；
- 16 个可学习 register token 吸收多余注意力（去掉掉 0.13 dB）；
- **物体置换等变**由构造保证（Prop. 1）→ 输入顺序不影响输出；
- 位姿用 **OBB-RoPE**：每个 token 带 12 维有向包围盒描述（质心 + 三个主轴 × 标准差），query/key 的旋转合成为相对描述的函数。

### 2) Invariant form-factor attention：把 form factor 塞进点积

这是全文唯一"有物理"的地方，也是它最值得学的地方。

**目标**：把辐射传输里的**不变量**（距离、两面元朝向）直接变成 attention logit 的一部分，**但不物化 $O(N^2)$ 的 bias 矩阵**。

**做法**：构造 4 维几何不变量向量

$$G_{ij}=\Big(\underbrace{-\lVert\Delta_{ij}\rVert^2}_{\text{距离核}},\ \underbrace{m_i^\top\Delta_{ij}}_{\text{接收面朝向}},\ \underbrace{-m_j^\top\Delta_{ij}}_{\text{发送面朝向}},\ \underbrace{m_i^\top m_j}_{\text{相互朝向}}\Big)$$

其中 $\Delta_{ij}=c_j-c_i$，$m$ 是 patch 的平均法线。**每一项都能写成"接收方特征"与"发送方特征"的精确内积**——例如 $-\lVert\Delta_{ij}\rVert^2=\big\langle[2c_i,-\lVert c_i\rVert^2,-1],\,[c_j,1,\lVert c_j\rVert^2]\big\rangle$。把这四组因子化特征直接拼进 query/key，attention logit 里就多出一项 $\sum_{t=1}^{4}a_t(i)\,b_t(j)\,G_{ij,t}$，**不需要任何两两 bias 表**。

**它到底覆盖了什么**：这四项正好是微分 form factor $dF_{ij}=\cos\theta_i\cos\theta_j\,dA_j/(\pi r_{ij}^2)$ 的**二次型自变量**（Prop. 4）。**没覆盖的三项交给网络去学**：非负余弦截断 $[\cos\theta]_+$、$1/r^2$ 衰减、相互可见性（遮挡）。作者自己写得很直白——这不是物理求解器，是**给注意力注入一个学出来的 form-factor 先验**。

### 3) 主可见性与视角解码：把"看见"从注意力里拿出去

- **point-splat 光栅化**（7×7 膨胀）给出免网格的 G-buffer：每个像素的可见物体 id、深度、法线、材质；
- 光线按 4×4 像素块打包成 token（CamRay 参数化 + 球坐标 RoPE），与场景 token 做 cross-attention；
- **物体 id 变成 routing bias**：某个光线 patch 里有多少比例的像素光栅化到了该物体 → 给对应的 attention logit 加一个可学习的带符号门（初始化为 0）；
- 消融很说明问题：**只给 G-buffer、不给 3D 场景 token，掉 5.66 dB**；**给 3D token 但不给 G-buffer 通道，掉 3.12 dB**。两者是分工不是替代——G-buffer 管"这条光线先撞到谁"，场景 token 管"这个点收到多少间接光"。

## Key Contribution

1. **物体级 tokenization**：传输序列长度与多边形数解耦（$32N$ vs $T$），中位 token 数 3,205 → 320，且**完全免网格**（点云直入）；
2. **invariant form-factor attention**：把辐射传输的 $O(3)$ 不变量精确因子化成内积，**不物化两两 bias 矩阵**；
3. **rasterizer-guided 解码**：用点泼溅解决主可见性，让 cross-attention 只负责"多次弹射 / 软接触阴影 / 颜色渗透"这些真正需要全局的事。

## Why It Works

- **物理量纲对齐**：物体的表面片是"有限区域"，正是辐射度能量交换的自然粒度；三角形是渲染管线的产物，不是物理量。
- **归纳偏置进结构而非损失**：距离与朝向不是靠数据学出来的，是**写进点积的**——所以样本效率高（1/5 步数到 25 dB）。
- **把难学的部分外包给确定性算子**：主可见性用光栅化（零学习、零幻觉），学习只负责"光怎么弹"。
- **register token + routing bias 是工程胶水**：前者防注意力沉没，后者把离散的"物体 id"变成连续的注意力偏置。

## Limitations

**作者自述**：

1. **32 是均匀配额**——给平面墙分多了，给几何复杂的物体分少了；按面积/曲率自适应分配留作未来工作。**这是"换单位"之后留下的下一个问题，也是最该被下游研究接住的一条**；
2. 视角解码器对光线 token 做**全自注意力**，分辨率超过 1024² 就撑不住（可用窗口/层次化光线注意力缓解）；
3. point-splat 光栅化**只处理直接表面求交**，烟、雾等体积介质未覆盖；
4. 输入法线的高频噪声会渗进平面渲染结果（表 24）。

**我补的三条**：

5. **"实时"这个词在这里不成立**：0.83 s / 帧（500 物体、512²、runtime-only、随机权重基准）离 16.6 ms 差两个数量级；论文也从未声称实时；
6. **小场景反而更慢**：20 物体以内 RenderFormer 更快，因为 RADIANT 的耗时被 **16,384 个光线 token 的解码**支配——**交叉点在 25 物体（开 FlashAttention 是 35）**。这条对"什么时候值得换单位"是硬约束：**单位换对了，只有在规模足够大时才回本**；
7. **质量评估只到 50 个实体**（训练分布 3–18），25–50 实体的拥挤场景三种模型一起掉（25.32 dB vs 32.2 dB）。"$O(N^2)$ 可扩展"目前是**算力上的可扩展，不是质量上的可扩展**。

## Game Development Relevance

- **它不是运行时方案，是"光照设计工具 / 真值工厂"**：0.83 s 一帧的 GI 预览，对打光迭代（换灯位、换材质、看颜色渗透）已经够用，而且**免烘焙、免光追**；
- **对预算框架的直接价值不是性能，是记账方式**：它证明了"**换计数单位**"能带来数量级的复杂度变化，而在新单位内部调额度几乎不敏感（$\pm8\times$ 预算 → 0.04 dB）。这条已经写进 [[Budget Units]]，并可反向检验你的 SABC 表——**你的五个维度里，哪些是"按内容增长"的单位，哪些是"按输出增长"的单位**；
- **与虚拟几何同构**：Nanite 把"几何成本"从物体复杂度改写成可见像素复杂度；RADIANT 把"GI 成本"从三角形数改写成物体数。**两者都是"让细分变成内容细节，而不是成本乘数"**；
- **点云直入**对扫描资产 / 摄影测量管线友好——不需要先重建网格。

## Unreal Engine Relevance

- **短期（研究/工具向）**：不能进运行时。可考虑的落点是**编辑器内的光照预览**或**离线 GI 校验**（对比 Lumen 的结果，找漏光/色渗）；
- **中期**：`invariant form-factor attention` 这个手法与 UE 无关，但**"把几何不变量塞进点积"** 这件事对任何 transformer 化的渲染组件都适用（例如神经后处理的空间偏置）；
- **对照物**：Lumen 的表面缓存 + 屏幕追踪，与 RADIANT 的"物体 token + 全局注意力"是同一问题的两种预算形态——**前者按探针/像素记账，后者按物体记账**。

## Technology Evolution

```text
"GI 的记账单位怎么选"
1984  面片（patch）——辐射度矩阵，$O(n^2)$ 个 form factor
1985  hemi-cube——把 form factor 的积分换成固定分辨率立方体的加权和
1991  层次辐射度——按重要性决定"哪些面片对值得算"
2005  RSM——按像素记账（每像素固定样本）
2019  DDGI——按探针记账（每探针固定射线）
2021  Neural Radiosity——按场景记账（一个网络拟合一个场景）
2025  RenderFormer——按三角形记账（$O(T^2)$）
2026  RADIANT——按物体记账（$O(N^2)$，$32N$ token）
      ↑ 每一次换单位，都是把"随内容增长的量"换成"随场景结构增长的量"
```

## Relationships

### Based On

- [[Goral — Modeling the Interaction of Light Between Diffuse Surfaces (1984)]] —— 微分 form factor $dF_{ij}$ 的定义就是它给的（论文 §3.4 直接引用）；
- [[Kajiya — The Rendering Equation (1986)]] —— 问题的容器；
- PointNet / PointNet++（集合抽象）、RoPE / CamRay（相对位置编码）、FiLM、register token —— 四个工程积木。

### Extends

- **RenderFormer / RenderFormer-V2 / RenderFormer++**（本库未入库，仅在 Watchlist 记名）：把 token 的定义从"三角形"上移到"物体"，并且**取消了三角形级注意力这一步**（RenderFormer++ 保留）。

### Related

- [[Neural Global Illumination]] —— 本库把"用网络估间接光"归在这一页；RADIANT 属于其中的 **scene-wide / 前馈** 分支（与 per-scene 拟合的 NRC/Neural Radiosity 相对）；
- [[Global Illumination]] —— 谱系表第 8 行（神经 GI）的 2026 新增样本；
- [[GPU Architecture]] —— "点泼溅 G-buffer"是典型的**先可见性后着色**结构，与 Reyes 1987 的"先着色后可见性"正好相反（见 [[Reyes — The Reyes Image Rendering Architecture (1987)]]）；
- [[Budget Units]] —— 本文最强的可迁移结论在第 4.3 节的 token 预算消融，不在主方法。

### Followed By

- 待观察：自适应 token 分配（按面积/曲率）、窗口化光线注意力、体积介质扩展、代码与数据集放出。

## Personal Knowledge State

**Normal（结论层）/ Hard（Transformer 内部）**。

- 结论层要拿走的只有三件事：① 换单位（三角形 → 物体）；② 换单位值 3.3 dB、调预算值 0.04 dB；③ 交叉点在 25 物体；
- `invariant form-factor attention` 的推导（4 维不变量的因子化）属 Hard，**但它的思想一句话就够**：把物理里"只依赖相对位姿的量"写成内积，就不需要两两表；
- 与你的已有知识接口：你在 [[Microfacet Theory]] 里熟悉的"BRDF 是四维函数、可因子化"是同一类操作——**因子化是为了让计算可分离**。

## Learning Value

- **最高价值不在 GI，在"成本模型怎么换单位"**：这套"识别随内容增长的量 → 找它的物理对应物 → 在新单位上重写"的三步，可以直接搬到 VFX 预算（例如把"粒子数"换成"屏幕覆盖面积"，把"发射器数"换成"材质/混合模式类别数"）；
- 次高价值：**"显式可见性 + 学习传输"的分工**，与 [[World Models for Games]] 里"引擎管状态、模型管画面"是同一条哲学在渲染内部的一次复现。

## Visualization

![[Research/Files/Pictures/Zhang 2026 — 图1 RADIANT 架构总览.png]]
（图 1，Zhang et al. 2026 —— 架构总览；读图见文首）

![[Research/Files/Pictures/Zhang 2026 — 图3 RADIANT 与两个基线在拥挤场景下的定性对比.png]]
**读图**：三组、每组 8 个场景，每场景 4 行——GT / RADIANT / RenderFormer / RenderFormer-V2（第 600k 步，同参数预算）。要看的是**接触阴影与颜色渗透**：GT 里物体底部那圈暗、白墙被彩色物体染上的那层色，RADIANT 保住了，两个基线在平面与曲面上都出现模糊与块状伪影。**注意 GT 本身也不"干净"**（Cycles 512spp 去噪后的结果），所以这类对比要看的是"错在哪一类"，不是"谁更锐"。（图 3，Zhang et al. 2026）

## Notes

- **数据规模**：103 万场景（1–6 个 Objaverse 物体，18,085 个网格，模板房间 + 随机墙面裁剪，1–8 个发光体，均值 4.5），每场景 4 视角，Cycles 512² / 512spp / OIDN 去噪；留出集 1,028 场景 / 4,112 图。
- **模型规模**：205.9M = 点编码器 10.0M + 场景 Transformer 97.0M（12 层、宽 1024、8 头、FFN 1890）+ 视角解码器 96.1M（6 层交叉/自注意力交替）+ DPT 头。
- **训练**：AdamW、峰值 lr $5\times10^{-5}$、8k warmup、cosine 衰减到 $10^{-6}$、全局 batch 24、bfloat16 + FP32 主权重；8×A100，1.28 s/step（中位）。
- **一个容易被忽略的诚实之处**：作者用同一份 103 万场景、同一套指标、同一参数量**重训了两个基线**（RenderFormer-V2 用它自己发布的训练循环）——所以这不是"打别人的论文数字"，是**同预算复现**。这类对照在本库里不常见，值得记一笔。
- **`RenderFormer-V2` 在同等预算下反而最差**（24.60 dB，600k 步都没到 25 dB）：与它原论文的设定（不同数据/不同规模）不可直接比较，作者也标了 n/r。
