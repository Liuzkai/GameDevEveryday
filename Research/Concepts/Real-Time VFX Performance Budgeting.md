---
type: concept
title: "Real-Time VFX Performance Budgeting"
user_level: Easy
tags: [vfx, performance]
---

# Real-Time VFX Performance Budgeting

> 这是**用户本人的专业领域**，不是从论文学来的知识。
> 本文件的目的不是教你，而是把你的实践抽象成可与外部研究对接的模型。

## Definition

为游戏中的视觉特效分配**可度量的资源上限**，使得在目标硬件与画质档位下，帧时间、内存与带宽同时不越界的一套方法论。

在 NGR 淬炼系统的语境下，它以**技能**为单位，以 **SABC 分级**为纵轴，以**实测指标 × 五档画质**为约束维度。

## Core Principle

```
技能 → SABC 分级（触发频率 × 难度 × 伤害）
     → 分配资源预算（与分级占比倒挂）
     → 五档画质实测校验
     → 不达标则降级或重构
```

**倒挂配比原则**：S/A/B/C 的资源分配约为 **35-40% / 30% / 20% / 10%**，与各级数量占比（约 10% / 13% / 42% / 34%）相反。即：少数高价值技能吃掉大部分预算，大量低价值技能保持极低成本。

## Prerequisites

- [[Real-Time Rendering]]
- [[GPU Architecture]]
- [[Niagara]]
- [[Profiling and Measurement]]

## 五个量化维度（NGR 现行模板）

| 维度 | S | A | B | C |
|---|---|---|---|---|
| Niagara 发射器数 | ≤12 | ≤8 | ≤4 | ≤2 |
| 同屏粒子数 | 3000 | 1200 | 400 | 100 |
| 贴图尺寸 | 2048 | 1024 | 512 | 256 |
| 动态灯光 | ≤3 | ≤2 | ≤1 | 0 |
| VFX 时长 | ≤3s | ≤2s | ≤1s | ≤0.5s |

## 五档画质

`PC_High` / `PC_Low` / `Android_High` / `Android_Mid` / `Android_Low`

## 实测指标

`Particles` / `DrawCalls` / `Primitives` / `OverDraw` / `CPUTime` / `GPUTime`

> **2026-10-25 补：`OverDraw` 这一项终于有自己的口径页了** → [[Overdraw]]。三条关键结论：
> ① **它有三种口径**（Coverage / Shaded / Blended），混用会得出矛盾结论；
> ② **它的成本 = 层数 × 屏占比 × 单价**，而"同屏粒子数"只是层数的一个上界；
> ③ **"同屏粒子数 ≤ 3000"单独存在时没有性能含义**——按平均屏占比换算，同样的 3000 可以是 **5.9× 全屏**（64²）也可以是 **94.8× 全屏**（256²）。**建议补一行"像素预算"（$\sum A_i / A_{\text{screen}} \le k$，S/A/B/C 各一个 $k$）。**

## 第六维该有的形状：预算不是"总量"，是"沿时间的分布"（2026-10-28 补）

现有五维全是**总量指标**：一个 S 档技能 = 发射器 ≤12、粒子 ≤3000、贴图 ≤2048、灯 ≤3、时长 ≤3 s。这五行有一个共同的隐含假设——**这 3 秒里预算是均匀花的**。

**它不成立。** 一个 3 秒技能的能量分布通常是：起手 0.2 s（几乎空）、**爆发 0.2–0.5 s（全部粒子/灯/屏占比都在这）**、余波 1.5–2 s（少量拖尾）。均摊口径下，"总量达标"完全可能与"峰值帧掉到 38 fps"同时成立——因为**总量守恒，峰值不受约束**。

两种可落地的补法（不改 SABC 分级）：

| 改法 | 写法 | 解决什么 | 成本 |
|---|---|---|---|
| **峰值窗口 + 峰值预算** | 在 3 s 内标出 ≤0.4 s 的峰值窗口，窗口内给"总预算的 60–70% 上限 + 峰值帧的粒子/屏占比上限" | 峰值帧掉帧 | 一次标注，无需新工具 |
| **按变化率重采样** | 曲线/粒子发射率按"每秒变化量"重采样，而不是按帧率 | 曲线资产冗余、GPU 端 curve LUT 过大 | 与动画压缩同一套工具 |

**证据来自今天入库的两篇**：[[2026-10-26-Yachil — GaussianSpline Adaptive Bezier Splines for 4D Gaussian Splatting|GaussianSpline]] 的消融显示"**加时间采样点没有收益（$K$=4 与 $K$=2 打平甚至更差），把点放到变化快的地方才值 +0.37~+0.91 dB**"；[[2026-10-24-Truong — Explicit Frictional Contact Modeling in Neural Garment Simulation|Truong 2026]] 的成本结构则是另一种"不均摊"——**GNN 前向 58% + 物理算子 42%，算子只在接触点上花，不接触的地方一分不花**。

> 一句话判据：**"这个预算数的是长度，还是形状？"** 只数长度的（时长、总粒子数、总灯数）都会在高动态段失守；数形状的（峰值窗口、分布、层数 × 屏占比）才能预测掉帧。


## Related Concepts

- [[Scalability and Quality Tiers]]
- [[Overdraw]]
- [[Budget Units]]
- [[Tile-Based Rendering]]
- [[Niagara]]

## Game Applications

- [[AAA Real-Time VFX]]

## External Research Touchpoints

外部研究对这个模型的冲击点：

1. **[[Reeves — Particle Systems (1983)]]** → **你的五个维度里有四个的原始出处**（屏占比 LOD / 发射器层级 / 寿命 / 加法混合免排序）。**含义：这份预算表可以引用 40 余年前的成本法则来论证，而不必只靠实测经验值。** 见 [[预算五维_1978-1983_源头图解]]
2. **[[Williams — Casting Curved Shadows on Curved Surfaces (1978)]]** → **"动态灯光"维度的成本依据**：每盏灯的阴影代价 ≈ +1× 场景渲染（单位是"一遍渲染"，不是"一个图元"）→ **这才解释了为什么它的上限是 0–3 而同屏粒子是 100–3000**
3. **[[LightOpt — Lights Optimization for Real-Time Rendering]]** → 把"动态灯光上限"从经验值变成可推导值
4. **[[DLSS 5 — Generative Neural Rendering]]** → 部分表达力外包给神经层，预算的自变量集合会变（但代价是 -50% 帧率，且只覆盖 PC）。**2026-09-21 补：神经层已确认消耗三个预算维度 —— 功耗（9-19）、帧时间（9-20）、显存/设备缓冲（9-21，Intel Xe2 重实现在 720p 占约 2.3 GiB）**
5. **[[Motion Style Slider — Continuous Style Control for Human Motion Diffusion]]** → "端点监督换连续控制"的思路可迁移：S/A/B/C 是 4 个离散端点，还是 1 个强度滑杆？
6. **[[MotionBricks — Scalable Real-Time Motions]]** → 若技能动作变成运行时生成，VFX 时序锚点必须从"固定帧"改为"语义事件"（**2026-09-21 由 [[2026-09-18-GestureFAR — Streaming Co-Speech Gesture Generation with Flow Autoregression]] 独立佐证第二次**）
7. **🔴 一条硬约束（2026-09-21，二手待核实）** → 二手来源称 **UE 5.8 MegaLights 不支持半透明物体 / 流体 / 云 / 发丝，也不支持前向渲染** → **特效打光不在 MegaLights 覆盖范围内**，仍走 [[Shadow Mapping]] 的成本法则。**"动态灯光变便宜"不能直接推到 VFX 侧。**
8. **[[Everitt — Interactive Order-Independent Transparency (2001)]]**（2026-10-25）→ **"顺序"这一项的成本边界**：不想用加法混合、又要正确的半透明，账单就是 `层数 × 一遍场景`（depth peeling）。**对你的意义**：加法混合买的是"无限层免排序"，卖的是"不能吸收/遮挡"——**每一次"这个特效要不要更真实的烟/水"，都是在给层数付钱**。
9. **[[Overdraw]] 的换算表**（2026-10-25）→ 把"粒子数"翻译成"像素预算"的那张表（64²/128²/256²/512² 各占全屏多少、多少粒子铺满 1×），是**唯一一条不改分级、只改度量**的修订建议。
10. **[[Budget Units]]**（2026-10-27）→ **对整张五维表的一次元层审计**：五维里有六个互不相同的计价对象（容量 / 成本 / 精度 / 时间 / 状态-记忆 / 像素覆盖），而**"降一档"在每个维度上的含义完全不同**——"发射器数 ≤12→≤8"是砍结构，"贴图 2048→1024"是砍分辨率，"VFX 时长 3s→2s"是砍内容长度。同日三篇前沿论文各自给出了一条量化证据：**换单位比调额度收益高两个数量级**（RADIANT 换 token 单位 +3.3 dB vs 调 token 预算 ±0.04 dB；PASPA 用错记忆单位比不给历史更差；UniWorld 把单位从"帧数"换成"窗口"后显存上界与时长解耦）。**落到本表的具体动作只有一个：给"同屏粒子数"补一行像素预算**（与第 9 条的结论同一条，但从"换算"升级为"为什么必须这么改"）。

## Personal Knowledge

Current Level: **Easy**

## Next Step

你的 Normal 区不在这里，而在：
- 把这些经验上限**形式化**（为什么是 12 个发射器？）
- 与 [[Differentiable Rendering]] 结合，把"上限"变成"优化解"
- 预判神经渲染普及后预算模型的重构
