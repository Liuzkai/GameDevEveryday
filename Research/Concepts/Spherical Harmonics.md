---
type: concept
user_level: Normal
aliases: [Spherical Harmonics, SH, 球谐函数, 球谐, SH9, SH25]
prerequisites: ["[[Rendering Equation]]"]
first_introduced: "1782（Laplace 球谐）；2001 起成为实时图形学的标准低频基（Ramamoorthi & Hanrahan）"
---

# Spherical Harmonics

## Definition

**球面 $S^2$ 上的一组正交归一基函数**——把"方向上的函数"（环境光、BRDF、可见性、辐照度）展开成有限个系数，就像傅里叶级数把一维信号展开成正弦波。

一个方向上的函数 $f(s)$ 可以写成：

$$f(s)=\sum_{l=0}^{n-1}\sum_{m=-l}^{l} f_l^m\,Y_l^m(s)$$

其中 $l$ 叫**带（band）**，$m$ 是带内索引；带 $l$ 的基函数是 $x,y,z$ 的 $l$ 次多项式。截断到 $l_{\max}$ 时**总共 $(l_{\max}+1)^2$ 个系数**（若按"带数 $n=l_{\max}+1$"计，就是常说的 $n^2$）。

| 最高带 $l_{\max}$ | 系数个数 $(l_{\max}+1)^2$ | 工程叫法 | 能表达什么 |
|---|---|---|---|
| 2 | **9** | SH9 / 二阶 | 余弦核辐照度、方向性环境光——**无阴影漫反射的极限** |
| 3 | 16 | SH16 / 三阶 | 稍锐一点的方向性 |
| 4 | **25** | SH25 / 四阶（原文称 quartic） | 带自遮挡的漫反射（阴影边缘开始像样） |
| 5+ | 36+ | SH36 | 更锐的阴影，但开始 ringing |

## Core Principle

**为什么图形学爱它：四条性质，每条对应一个工程收益。**

| 性质 | 数学形式 | 工程收益 |
|---|---|---|
| **正交归一** | $\int Y_iY_j\,ds=\delta_{ij}$ | 投影就是积分、重建就是求和——**两个函数乘积的积分 = 系数点积**（$\int ab=\sum a_ib_i$） |
| **旋转不变** | $g(s)=f(Q s)\Rightarrow$ 系数按阶线性变换 | 物体转、光不转时**只转少量光照系数**；旋转采样**不产生走样/时域抖动**（这是选 SH 而非球面小波的关键理由） |
| **卷积定理** | 圆对称核 $h$ 的卷积 = 系数**逐项相乘**（乘一个带内标量） | 余弦核（辐照度）、Phong 瓣（光泽）都能"一次乘法"完成，且**核只需 $m=0$ 的系数**（zonal harmonics） |
| **乘积投影** | 两函数相乘 = 系数经一个对称矩阵变换（Clebsch-Gordan / 三积积分） | 把"可见性 × 余弦"这类乘积烘成一个算子（[[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)\|PRT]] 的 `DS` 档就靠这条） |

**代价与边界**：

- **带限 = 低通**：截断到 $n$ 阶就等于把信号强行抹平到该频带；小光源、锐利阴影、高频细节全部丢失；
- **ringing / Gibbs**：信号在高阶仍有能量时，截断会在重建中出现振铃（[[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)|PRT]] 图 10 的 $n=26$ 列是标准样本）；工程上用 **windowing**（对高阶系数加衰减窗）压制；
- **不是万能压缩**：SH 对**低频、全方向**的信号最优；对局部集中的信号（一个亮点、一条窄高光）效率极低——那是球面小波/球面高斯的领域。

## Prerequisites

- 球坐标与立体角（$d\omega=\sin\theta\,d\theta\,d\varphi$）——见 [[Rendering Equation]] 的积分形式；
- 正交基与投影的直觉（傅里叶级数是最好的一维类比）；
- 勒让德多项式（只需知道"zonal harmonic = 只依赖 $\cos\theta$ 的那一族"）。

## Historical Evolution

```text
1782  Laplace：球谐函数（位势理论，纯数学）
1880s-1900s  量子力学角动量（Clebsch-Gordan 系数、旋转矩阵——图形学直接借用了这套公式）
  ↓
1992  Westin / Arvo / Torrance：用 SH 表示 BRDF（离线）
1995  Dobashi 等：SH 基 + 表面转移向量场（交互式光照设计）
  ↓
★ 2001  Ramamoorthi & Hanrahan：**9 个系数就够了**——无遮挡漫反射辐照度是余弦核卷积，
        解析给出 9 个带内标量；等价于"光照的 2 阶 SH"。移动端 GI 的数学根源
  ↓
★ 2002  Sloan / Kautz / Snyder：把**可见性**塞进同一个基 → PRT（9 → 25 系数，向量 → 矩阵）
2003  CPCA：transfer 场聚类 + 主成分压缩（SH 系数太多的问题）
2005  LDPRT：可变形物体的局部 transfer
  ↓
2006-2010s  引擎工业化：SH 探针 / 辐照度体 / Lightmap 里的 SH（UE 的 ILC、Volumetric Lightmap）
  ↓
2019  DDGI：探针里存的就是 SH 系数（8×8 或 16×16 八面体编码 + SH 投影），但**不再是 PRT 的 transfer**
2020s  3DGS 每个高斯存一组 SH 系数表示视角相关颜色（**SH 从"光照表示"变成"外观表示"**）
2026  神经表示兴起，但 SH 仍是"必须知道的基线"——几乎所有神经场/高斯方案都用它做低频兜底
```

## Important Papers

| 节点 | 角色 | 状态 |
|---|---|---|
| [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]] | **9 系数定理**：无遮挡漫反射辐照度 = 光照的 2 阶 SH（解析带内标量） | ✅ 入库 |
| [[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)]] | **把可见性与互反射纳入基**：9 → 25 系数，向量 → 矩阵；三档价目表 | ✅ **2026-10-21 入库** |
| [[Majercik — Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields (2019)]] | 探针场用 SH 存辐照度（运行时更新，非预计算转移） | ✅ 入库 |
| [[2026-09-14-Gaussian Light Transport]] / [[Gaussian Splatting]] | 3DGS 用 SH 存视角相关颜色——SH 的"外观表示"用法 | ✅ 入库 |
| CPCA（Sloan 2003）/ LDPRT（2005） | transfer 场的压缩与可变形扩展 | 记名 |

## Related Concepts

- [[Rendering Equation]] —— 全部推导的容器；SH 只是给它的方向积分换了一套坐标；
- [[Global Illumination]] —— 谱系表第 2 行（SH 表示）与第 4 行（PRT）都以 SH 为语言；
- [[Split-Sum Approximation]] —— 同族思路：把积分拆成两张可预计算的表（BRDF 项 + 环境项），但拆的是"BRDF × 环境"，不涉及可见性；
- [[Physically Based Rendering]] —— IBL 的 split-sum 与 SH 辐照度是环境光的两条并行路线（前者保镜面细节、后者保方向性低频）；
- [[Gaussian Splatting]] —— SH 的最大规模民用场景：几百万个高斯 × 每色 16/25 个系数。

## Technologies

- **SH 光照探针**（Unity Light Probes / UE Indirect Lighting Cache / Volumetric Lightmap）——引擎里最常见、也最容易被误解的 SH 用法（**只存辐照度，不存可见性**，所以"漏光"是设计使然）；
- **PRT / CPCA**——预计算转移（已退出实时主流，思想仍在）；
- **DDGI / RTXGI**——探针 + 光追增量更新 + SH 编码；
- **3DGS 的视角相关颜色**——每个高斯一组 SH 系数（通常 16 个/通道）。

## Game Applications

- **静态场景的间接光**：Lightmass / Volumetric Lightmap 烘 SH 辐照度，移动物体查询；
- **角色/动态物体的环境光**：SH 探针插值，避免"角色比环境亮/暗"；
- **天空光/时间变化的低成本近似**：只更新 9–25 个系数，不重烘贴图；
- **与你的预算工作的接口**：SH 阶数是**极少数"质量档位有明确数学刻度"的参数**——9 / 25 / 36 个系数直接对应"有没有阴影""阴影锐不锐"，比"高/中/低"这种命名精确得多。[[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)|PRT]] 图 10 是这张刻度表的原始证据。

## Personal Knowledge

- `user_level: Normal`（推断：你在 PBR/IBL 语境里已经用过 SH9，但**没有把它当"一套有代价的表示"系统看过**）；
- 已知的连接：[[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]]（9 系数定理）→ 本笔记（四条性质）→ [[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)|PRT]]（25 系数 + 可见性）→ 探针/DDGI（运行时更新）；
- **一句话检验**：能说清"**为什么阴影要比无阴影多两个阶**"（因为 $V$ 是 0/1 的阶跃函数，在局部造出高频；余弦核本身是天然低通）——即算掌握。

## Learning Gap

- **需要**（≈15 分钟）：**卷积定理为什么让辐照度变成 9 个数**——余弦核的 SH 投影只有 $m=0$ 项（$h_l^0$），所以卷积 = 逐带乘一个标量，[$l=0,1,2$ 三条带共 9 个数]；这是 [[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)|RH 2001]] 的全部内容；
- **不需要**：Clebsch-Gordan 系数的具体递推、SH 旋转矩阵的显式形式（工程上用现成库；只需知道"按阶分块、不跨阶混合"）；
- **可选**：ringing 的成因（带限截断的 Gibbs 现象）与 windowing 的三种常用窗（Hann / Lanczos / Sloan 的余弦幂窗）。

## Next Step

1. 打开引擎的 SH 探针可视化（UE: Volumetric Lightmap / ILC），找一个"探针网格尺寸 >> 几何细节"的位置，观察漏光——这是"9 系数 + 无可见性"的直接后果；
2. 把 [[Sloan — Precomputed Radiance Transfer for Real-Time Rendering (2002)|PRT]] 图 10 的八列，与自己项目的画质档位表并排看一次：**"低/中/高"能不能翻译成 "9 / 16 / 25 系数"**——能翻译，档位就有了物理依据。
