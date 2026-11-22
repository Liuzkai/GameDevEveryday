---
type: paper
title: "Sea-thru: A Method For Removing Water From Underwater Images"
authors: ["Derya Akkaynak", "Tali Treibitz"]
year: 2019
published: "2019-06（CVPR 2019）"
venue: "CVPR 2019, pp.1682–1691（IEEE/CVF）"
url: "https://openaccess.thecvf.com/content_CVPR_2019/papers/Akkaynak_Sea-Thru_A_Method_for_Removing_Water_From_Underwater_Images_CVPR_2019_paper.pdf"
code: ""
project_page: "http://csms.haifa.ac.il/profiles/tTreibitz/datasets/sea_thru/index.html"
category: [Underwater Rendering, Participating Media, Inverse Rendering, Computational Photography]
importance: A
historical_importance: 4
game_relevance: 3
production_readiness: "Prototype（需要 RGBD / 已知尺度的距离图 + 逐图优化；作者给出 1,157 张数据与参考实现，但非实时）"
user_level: Normal
status: unread
aliases: [Sea-thru, SeaThru, Akkaynak 2019, 水下图像去水, 修订水下成像模型, 距离相关衰减系数]
tags: [underwater, participating-media, inverse-rendering, image-formation-model, classical]
---

# Akkaynak 2019 — 系数不是水的常数，是距离的函数

## TL;DR

**把大气模型直接搬到水下，错在一个前提：衰减系数被当成常数。** 这篇论文的出发点是 2018 年那篇模型修正（$\beta^D\neq\beta^B$，且 $\beta^D$ 依赖距离与反射率），而它给出的是**第一个能用这个修订模型把水去掉的方法**：

$$I_c=\underbrace{J_c\,e^{-\beta^D_c(z)\,z}}_{\text{直达（被削弱）}}+\underbrace{B^\infty_c\left(1-e^{-\beta^B_c z}\right)}_{\text{幕光（随距离增亮）}}$$

三个决定：

1. **$\beta^D\neq\beta^B$**——直射衰减与幕光增长是两套系数（旧模型假设相同）；
2. **$\beta^D=\beta^D(z)$**——衰减系数随距离**下降**，不是常数；
3. **这个下降用两项指数描述**：$\beta^D_c(z)=a\,e^{bz}+c\,e^{dz}$——四个参数替代"每像素一个系数"，未知量从 $O(\text{像素})$ 降到 $O(1)$。

> **一句话判据：水下颜色不是"按指数衰减"，而是"按一条会变缓的指数衰减"。** 把 $\beta$ 当常数，会在**远处**错，而且错得比"什么都不做"更糟（见下面的 S3）。

## Problem

水下图像的颜色恢复长期做不好，作者把原因归结到一个**模型错误**而不是算法不够强：

- 现有方法几乎都用大气成像模型 $I=J\,t+A(1-t)$，其中 $t=e^{-\beta z}$，$\beta$ 是"这个场景的一个常数"（或每通道一个常数）；
- 这个模型假设：**衰减与散射是同一件事**，且**系数只是水的属性**；
- 但在水里，**直射光的衰减**与**幕光的增长**是两个不同的物理过程，系数不同；而且**直射衰减系数依赖距离与物体反射率**。

后果不是"稍微不准"，而是**系统性偏差**：近处看起来还行，远处颜色被错误地放大或压缩。

## Historical Context

这条线是"把大气模型借过来 → 发现借错了 → 造一个自己的"：

| 年 | 工作 | 借了什么 / 改了什么 |
|---|---|---|
| 1980 | McGlamery 水下相机模型 | 第一个工程可用的正向模型 |
| 1990 | Jaffe 成像系统设计 | 把模型变成设计工具 |
| 2002 | [[Narasimhan 2002 — Vision and the Atmosphere\|Narasimhan & Nayar]] | 两项加法 + 二色平面；**明确假设"前向散射可忽略"** |
| 2018 | Akkaynak & Treibitz（CVPR） | **修订模型**：$\beta^D\neq\beta^B$，且依赖 $z,\rho,E,S_c,\beta$ |
| **2019** | **Sea-thru（本文）** | 第一个用修订模型**做复原**的方法 |
| 2023 | SeaThru-NeRF | 把两项模型塞进体渲染 |
| 2026 | [[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting\|NIRJALA]] | 加第三项（前向散射）让反演增益有界 |

**这条线的每一次进步都不是"把水建模得更像水"，而是"让反演更可解"。** Sea-thru 的贡献是把"每像素一个未知系数"压成"每通道四个参数"，**用参数化换可解性**。

## Previous Work（三类，各缺什么）

| 路线 | 做法 | 缺什么 |
|---|---|---|
| 单图 dehaze（DCP 及其变体） | 用暗通道先验估幕光与透射率 | 假设系数恒定；水下会把幕光估高（能见度变好但颜色被编造） |
| 多图 / 立体 | 用已知距离拟合模型 | 仍用旧模型（$\beta^D=\beta^B=$ 常数） |
| 深海网络 | 端到端回归 | 训练数据是合成的，模型错了学不回来 |

**Sea-thru 的位置**：它不反对学习，而是先把**物理模型**改对，再用一个轻量的估计流程（没有神经网络）把参数解出来。

## Core Idea — 三个偏离

| 偏离 | 换成什么 | 代价 |
|---|---|---|
| ① 两套系数 | $\beta^D$（直达）与 $\beta^B$（幕光）分开 | 多一个未知量；需要知道"幕光在无穷远处"的 $B^\infty$ |
| ② 系数随距离变 | $\beta^D=\beta^D(z)$，随 $z$ **下降** | 必须知道每个像素的距离 $z$（RGBD / SfM） |
| ③ 用两项指数描述这个变化 | $\beta^D_c(z)=a e^{bz}+c e^{dz}$ | 四个参数是**经验拟合**，不是解析推导 |

**第 ② 条为什么必然**：$\beta^D_c$ 不是一个物理常数，而是**波段平均**的结果（Eq. 3）：

$$\beta^D_c=\frac{1}{\Delta z}\ln\!\left[\frac{\int_{\lambda_1}^{\lambda_2}S_c(\lambda)\rho(\lambda)E(d,\lambda)e^{-\beta(\lambda)z}\,d\lambda}{\int_{\lambda_1}^{\lambda_2}S_c(\lambda)\rho(\lambda)E(d,\lambda)e^{-\beta(\lambda)(z+\Delta z)}\,d\lambda}\right]$$

看这个式子：**相机的光谱响应 $S_c$、物体的反射率 $\rho$、环境光 $E$、水的光束衰减 $\beta(\lambda)$ 全被卷进去了**。所以"红通道的衰减系数"必然随 $z$ 变——**红光先被吃掉，剩下的光谱形状就变了，平均值自然变**。

> **这一条是本篇最重要的观念**：水下的"红通道系数"不是水的属性，是**水 × 相机 × 物体 × 距离**的联合属性。所以任何"水下预设"都只是一次标定的快照。

## Technical Approach

### 1. 距离图

- SfM（Photoscan）建 3D 模型 ⇒ 每像素距离 $z$；SfM 只给相对尺度，所以场景里要放**已知尺寸的物体**（色卡）定标；
- 或者立体相机（需要同步与水下标定）。

### 2. 幕光（backscatter）

暗像素 + 已知距离：

- 把距离图分成 **10 个等距簇**；每簇取 RGB 三元组最暗的 **1%**（记作 $\Omega$）；
- 用非线性最小二乘拟合

$$\hat B_c=B^\infty_c\left(1-e^{-\beta^B_c z}\right)+J'_c\,e^{-\beta^{D\prime}_c z}$$

- 第二项是**残差项**（"暗像素不是全黑"时的直接信号），参数约束：$B^\infty_c\in[0,1]$、$\beta^B_c\in[0,5]$、$J'_c\in[0,1]$、$\beta^{D\prime}_c\in[0,5]$；
- 残差项什么时候可以丢：暗像素是真黑 / 在阴影里 / $z$ 很大 / 水极浑（$B_c\gg D_c$）——**否则必须保留**。

### 3. 衰减系数（两步）

**粗估**：先把幕光减掉得 $D_c$，用**局部空间平均色**（LSAC）估局部光照图 $\hat E_c$，再

$$\hat\beta^D_c(z)=-\frac{\log\hat E_c(z)}{z}$$

LSAC 的邻域被**距离门限**限制：只平均"深度差小于 $\epsilon$"的 4-邻居（否则会把远处的暗像素混进近处的光照估计）。

**精修**：把两项指数代回去，最小化距离残差

$$\min_{a,b,c,d}\ \|z-\hat z\|,\qquad \hat z=-\frac{\log\hat E_c}{\beta^D_c(z)}$$

参数上下界保证"衰减"（$a,c>0$、$b,d<0$）。

### 4. 白平衡

用 Gray World（色彩丰富场景）或空间域方法（单色场景，如 D4 的峡谷）。

### 5. 评测指标

六个灰块的**角误差**：

$$\bar\psi=\frac16\sum_{i=1}^{6}\cos^{-1}\!\left(\frac{I_c}{\sqrt3\,\|I_c\|}\right)$$

越小越好——但作者自己指出它**不是**最优指标（见下面 S2 的反例）。

## Key Contribution

1. **第一个用修订模型做复原的方法**：$\beta^D\neq\beta^B$ 与 $\beta^D(z)$ 同时进流程；
2. **把"每像素系数"压成"每通道四个参数"**：两项指数是这篇论文的"可解性装置"；
3. **一个可复现的数据集**：5 组、**1,157 张**（559 / 318 / 68 / 153 / 59），两台相机（Sony α7R III、Nikon D810），两类水体（清澈礁石 / 浑浊峡谷），全部带 SfM 距离图与色卡；
4. **一个诚实的负结果**：**S3 比 raw 更差**（见下）。

## Why It Works

**因为"除以透射率"这件事的误差形状被改对了。**

旧模型下，去水 = 除以 $e^{-\beta z}$，$z$ 越大增益越大；如果 $\beta$ 取大了，远处被过度提亮；取小了，远处仍然发灰。**而真实的 $\beta^D(z)$ 是下降的**——远处每米的衰减比近处小。用一个常数去拟合整段距离，必然在**两端之间**取一个折中，于是近处过校正、远处欠校正（或反之）。

两项指数给了曲线一个"先陡后缓"的形状，正好对上"红光在近处被吃得快、远处剩下的光谱已经没多少红可吃"这件事。

## 关键数字（这一节是全文最值钱的部分）

**D1–D5 五组数据的平均角误差 $\bar\psi$（度）：**

| 方案 | 做法 | 平均误差 |
|---|---|---|
| raw | 原图 | **20.57** |
| S1 | 全局对比度拉伸 | 12.49 |
| S2 | DCP（旧模型 + 错估幕光） | 14.38 |
| **S3** | **旧模型 + 正确幕光 + 常数 $\beta$** | **21.77** |
| S4 | 修订模型，但不显式求 $\beta^D(z)$（只除光照图） | **4.13** |
| S5 | **Sea-thru**（修订模型 + $\beta^D(z)$） | 6.33 |

**立体数据集（Berman 等）**：raw 22.28 / S1 6.83 / S2 10.03 / S3 12.04 / S4 4.46 / **S5 4.94**。

**三个必须记住的读数：**

1. **S3 比 raw 更差（21.77 > 20.57）。** 把幕光减对了、却用常数 $\beta$ 去除——**半个正确模型比不处理更糟**。原因很清楚：减幕光让远处的暗信号露出来，而常数 $\beta$ 把这个信号放大错了倍数。**"做对一半"在反演里会主动制造误差。**
2. **S4 在色卡上赢 S5（4.13 < 6.33），但在整幅场景上输。** 色卡是平面、近距离、颜色已知——恰好是"局部光照估计"最容易的地方；整幅场景有深度变化与阴影，S5 的 $\beta(z)$ 才发挥。**指标选在哪里，结论就长在哪里。**
3. **S1（一个全局对比度拉伸）在距离均匀的场景里能赢 S2。** 这不是"简单方法更好"，而是**当场景没有距离变化时，距离相关的模型没有发挥空间**——**方法的价值与场景的距离动态范围成正比**。

## Limitations（七条）

1. **两项指数是经验拟合**，不是从 Eq. 3 推出来的闭式；作者只说"短距离可近似为直线"。
2. **只恢复了 $z$ 依赖，没有恢复 $\rho$ 依赖**（作者在结论里明确留作未来工作）。
3. **仍是两项模型**：没有前向散射的"模糊"（光不从一个像素搬到另一个像素）——与 2002 年同一笔欠账，直到 2026 年的 NIRJALA 才补第三项。
4. **需要 RGBD**：SfM 或立体，且要已知尺度；单张图不行。
5. **距离图在远处最不准**，而远处恰好是误差最大的地方——**两个误差源在同一端叠加**。
6. **水平成像假设**：Eq. 2 是对水平方向写的，不同朝向的场景只是"偏差小"的近似（作者自己标注）。
7. **指标只看色卡**：作者给出反例——S2 在某个色卡上 $\bar\psi$ 比 S5 低，但灰块上有明显紫色伪影。**角误差不惩罚"编造的颜色"。**

## Game Development Relevance

- **水下关卡的正向渲染**：游戏里几乎都用"一个 $\beta$ + 两项"。Sea-thru 说的是——**这个 $\beta$ 在 30 m 尺度上不是常数**。如果你的水下关卡有 20–40 m 的可见距离，"远处的红"会被常数模型低估（因为真实衰减在远处变慢）。
- **可落地形式**：$\beta^D(z)$ 是一条一维曲线 ⇒ 引擎里可以用**一条 1D LUT 或一个两项指数的解析式**实现，**成本 O(1)**，不需要体渲染。这是本页对工程最有用的一个转换。
- **3DGS 水下扫描**：颜色漂移不是单一指数；用常数 $\beta$ 的 pipeline 会把远处颜色"压平"，并且在优化时把误差推回几何。
- **验收方式**（承 [[Underwater Rendering]]）：让模型渲染"无水"图，看**远处**的颜色是否还在原地；再看近处——**近处对、远处错，说明 $\beta$ 被当成了常数**。

## Unreal Engine Relevance

没有直接的 UE 系统对应（它不是渲染技术，是复原/标定方法）。可迁移的只有两点：

- **`ExponentialHeightFog` 的 `FogDensity` 是常数**；若要做"物理正确的水下"，需要把密度换成"距离的函数"——在材质里用一条 1D 曲线或两项指数即可；
- **水下后处理**里的"颜色随距离偏移"，本质是 $\beta^D(z)$ 与 $\beta^B$ 两套系数的差；用一套参数（一个颜色 + 一个密度）做不出"近处偏青、远处偏蓝"的正确过渡。

## Technology Evolution

```text
1980  McGlamery        水下相机模型（正向，两项）
1990  Jaffe            成像系统设计
2002  Narasimhan       大气两项 + 二色平面；明确丢掉前向散射
2018  Akkaynak         ★ 修订模型：βD ≠ βB，且依赖 z / ρ / E / S
2019  ★ Sea-thru       第一个用修订模型复原的方法
                       + βD(z) 的两项指数
                       + 1,157 张带距离图的公开数据
                       + S3 < raw 的负结果
2023  SeaThru-NeRF     两项进体渲染（NIRJALA 的真实数据基准）
2026  NIRJALA          加第三项（前向散射）⇒ 反演增益有界
```

## Relationships

### Based On

- [[Narasimhan 2002 — Vision and the Atmosphere|Narasimhan & Nayar 2002]]——两项加法与"介质是编码"的世界观；
- Akkaynak & Treibitz 2018（CVPR）——修订模型本身（本文的方法基础）。

### Extends

- 大气成像模型：从"常数系数"扩展到"距离相关系数"；
- 旧的水下复原方法（DCP 系）：从"先验估透射率"扩展到"已知距离 + 物理模型"。

### Related

- [[Participating Media]]——上位概念；本文是"第七条路（反解）"的第一个可执行版本；
- [[Underwater Rendering]]——本页补上了那条线 2018/2019 的原文节点；
- [[Inverse Rendering]]——水是"几何/材质/光照"之外的第四类未知量；
- [[Atmospheric Scattering]]——同一族方程，边界与系数谱不同。

### Followed By

- SeaThru-NeRF 2023（两项进体渲染）；
- WaterSplatting / SeaSplat / 3D-UIR / AquaGS 等 GS 族；
- [[2026-11-20-Chhetri — NIRJALA Draining the Sea out of Gaussian Splatting|NIRJALA 2026]]——三项 + 有界增益证明。

## Personal Knowledge State

- **Normal（2026-11-22 建页）**。此前本库对 2018/2019 两个节点的描述全部来自 NIRJALA 的二手转述；今天补上原文后，**"$\beta^D$ 为什么依赖距离"第一次有了可核对的推导（Eq. 3 的波段平均）**。
- **Easy**：水下偏青、远处发灰、红光先没——做场景的人都见过。
- **Hard**：Eq. 3/4 的波段积分与相机光谱响应的耦合；以及"两项指数"为什么是这个形状（作者没有给推导，只有拟合）。

## Learning Value

1. **"系数是波段平均"这个观念可以迁移到任何逐通道建模**（雾、皮肤、胶片）：**只要通道是宽的，"系数"就不是介质的属性。**
2. **负结果比正结果有用**：S3 < raw 说明"模型错一半"会主动制造误差。**反演里没有"部分正确"这回事。**
3. **参数化的形状决定了可解性**：两项指数不是物理，是**把未知量从每像素压到每通道四个**的手段。**看到"可解"，先问"它为此假设了什么形状"。**

## Visualization

![[Research/Files/Pictures/Akkaynak 2019 — 图2 直达与幕光随距离的交叉.png]]
**读图**：一个像素的亮度是两项之和——直达项 $D_c$（虚线，随距离指数衰减）与幕光项 $B_c$（实线，随距离指数饱和）。**两条曲线在 $z\approx27$ m 处相等**：比这更远，幕光主导；比这更近，物体本身主导。**这就是"水下能见度"的定量含义**，也是为什么远距离的颜色恢复必然病态（除以一个接近 0 的 $D_c$）。出处：Akkaynak & Treibitz 2019, Fig. 2。

![[Research/Files/Pictures/Akkaynak 2019 — 图4 色卡随距离的幕光估计.png]]
**读图**：同一张色卡在 8.0 m 到 1.3 m 的不同距离被拍（上排 raw、下排去幕光），下排显示**幕光被减掉后色卡颜色基本回到一致**。右下是用"暗像素 + 已知距离"估出的 $B_c$（×）与色卡实测的真值（○）——**两组点几乎重合，且不需要任何水体类型输入**。出处：Akkaynak & Treibitz 2019, Fig. 4。

![[Research/Files/Pictures/Akkaynak 2019 — 图5d 衰减系数随距离下降.png]]
**读图**：三个通道的 $\beta^D_c$ 都随距离**下降**，而且三种独立估计方法（两张不同距离的照片、白点法、Eq. 3 的光谱模拟）给出同一形状——**这是"系数不是常数"的直接证据**。红通道的绝对值最大（红先死），蓝通道最小；三条曲线都呈"先陡后缓"，这正是两项指数要拟合的形状。出处：Akkaynak & Treibitz 2019, Fig. 5d。

![[Research/Files/Pictures/Akkaynak 2019 — 图6 五个场景的方法对照.png]]
**读图**：五行是五个场景（D1–D5），列依次为 raw、距离图、S1 对比度拉伸、S2 DCP、S3 常数 $\beta$、S4 只除光照图、S5（Sea-thru）。**最该看的是 S3 那一列**：它已经减掉了幕光，但颜色仍然错——有的场景（D3/D4）甚至比 raw 更花。**"减对幕光"不等于"除对透射率"**。出处：Akkaynak & Treibitz 2019, Fig. 6。

## Notes

- 原文 PDF 已归档：`Research/Files/PDF/Akkaynak 2019 - Sea-thru A Method For Removing Water From Underwater Images.pdf`（CVF Open Access，10 页，含 7 图 1 表，逐页核对）。
- ⚠️ **一处旧记载需要更正**：本库 [[Underwater Rendering]] 此前把本文写作"CVPR 2019 最佳论文"，**未找到官方依据**（CVPR 2019 最佳论文为 *A Theory of Fermat Paths for Non-Line-of-Sight Shape Reconstruction*）。已改为"CVPR 2019 论文（pp.1682–1691）"。
- 数据集：5 组共 **1,157 张**（559 / 318 / 68 / 153 / 59），Sony α7R III + FE 16-35mm f/2.8 GM、Nikon D810 + Nikkor 35mm f/1.8；深度 4–10 m；清澈礁石与浑浊峡谷两类水体。
- 作者自陈的"未来工作"：恢复 $\rho$ 依赖、用深度网络替代当前的估计器、以及**用正确的模型做仿真来生成训练数据**（因为没有真值）。
