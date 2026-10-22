---
type: concept
user_level: Normal
aliases: [Fluid Simulation, 流体模拟, 流体仿真, 流体求解器]
prerequisites: [向量场直觉（散度/梯度）, 数值时间积分基础概念, 粒子系统（对照侧）]
first_introduced: "1965（CFD 差分法进入 CG 前史 / Harlow-Welch）；1999（CG 界首个无条件稳定求解器 / Stam）"
---

# Fluid Simulation

## Definition

**用数值方法求解流体的运动方程（Navier-Stokes），让烟、火、水、气体"自己动起来"**——而不是靠关键帧或手工动画。与 [[Particle Systems]] 的区别：粒子系统把现象离散成独立粒子（Reeves 1983），流体模拟求解"**场**"（速度场 + 密度/温度等标量场）的演化，天然带**守恒性与旋转结构**（涡旋）。

## Core Principle

**四大步骤（本项目前最通用的骨架，Stam 1999 的四项分解）：**

```text
每个时间步：
① 外力（add force）—— 重力/风/鼠标施力/浮力
② 对流（advect）—— "物体被自己的速度搬运"
   网格线：回溯粒子 + 插值（半拉格朗日）
   粒子线：粒子直接跟随速度场移动（FLIP/SPH）
③ 扩散（diffuse）—— 粘性/扩散（隐式或显式）
④ 投影（project）—— 强制"无散"（质量守恒）→ 涡旋被逼出来
```

**三条路线**（选型的核心分野）：

```text
网格法（Eulerian）：Stam 1999 一系；成本 ∝ 网格单元数 × 步数；
  优点：稳定、省粒子、易交互；缺点：数值耗散、难做自由表面（水花/飞溅）
粒子法（Lagrangian）：SPH 一系；成本 ∝ 粒子数；适合飞溅/破碎，稳定性与调参难
混合法（FLIP）：2005 前后进入图形学；粒子携带量、网格解压力——
  影视管线主流（Houdini FLIP）；兼得"体积保持"与"网格稳定"
```

**核心权衡（本域第一账）**：**精度 ↔ 稳定性/速度**——Stam 1999 的名言是"CG 要的是形状与行为，物理精度次要甚至无关"；降档语言 = 降网格分辨率、降步数、换更省的表示。

### 水的第二条路：不求解，采样统计（2026-10-22 落位）

流体的三条路线都在**解方程**；水面还有一条**不解方程**的路——**谱合成**（[[Tessendorf 2001 — Simulating Ocean Water|Tessendorf 2001]]）：

```text
按实测海浪谱（Phillips: k⁻⁴ 衰减 + 风向余弦）随机生成频域系数
    → 每帧只乘相位 e^{iωt}（色散 ω=√gk，帧间无状态）
    → 一次 2D IFFT 变回高度场
    → 水平位移造尖峰（Creamer）+ Jacobian J<0 处起泡沫
```

**它值钱的地方是成本模型**：每帧 $O(N^2\log N)$，**只与网格分辨率有关，与"水面上有什么/镜头看到多少"完全无关**——是**固定成本型**（像后处理），不是**按元素计费型**（像粒子、像网格求解）。代价是**不可剔除、不可局部加密、无拓扑变化**：想让船扰动水面就得换 iWave 一类的显式波动方程，想做"近细远粗"只能多级 patch。

> **判据（放进预算工具箱）**：*这个水面项的成本，随"画面里有多少水"变、还是随"分辨率"变？* 前者可剔除、可降档；后者一旦开就全帧付账——**游戏最终普遍选 Gerstner 叠加（顶点可控、可剔除）而非 FFT，原因在此**。

## Prerequisites

- **向量场直觉**：梯度、散度（"流体是否在源/汇"）、无散场的含义（不可压）；
- **数值积分基础概念**：显式 vs 隐式时间步、稳定性直觉（何时会"爆炸"）；
- **对照侧**：[[Particle Systems]]（离散 vs 场的两种建模），[[Rendering Equation]] 的"场"语言（同为"求积分/求场"的两条数值路线）。

## Historical Evolution

```text
1965  Harlow-Welch：CFD 经典差分法（CG 的数学前史）
1986  Yaeger-Upson：《2010》木星 2D 涡方法
1990  Kass-Miller：浅水方程线性化（快、稳，但无旋转结构）
1993-95  Stam-Fiume：湍流风场 / 火与气体（非动力学速度场）
1996/97  Foster-Metaxas：完整 3D NS 首次进 CG（液体 / 热湍流气体）——显式、会爆炸
1999  ★ [[Stam — Stable Fluids (1999)|Stam《Stable Fluids》]]：无条件稳定求解器
      （拉格朗日对流 + 隐式扩散 + 投影；"动画师续命"的耗散取舍）
2001  ★ [[Tessendorf 2001 — Simulating Ocean Water|Tessendorf《Simulating Ocean Water》]]：**水的第二条路——谱合成**
      （不解 NS，按 Phillips 谱随机造频域系数 + 一次 IFFT；色散 ω=√gk 只管相位；
       尖锐位移 + Jacobian 折叠判据→泡沫；"全有或全无"的固定成本模型）★ 2026-10-22 入库
2001  Fedkiw 等《Visual Simulation of Smoke》：+ 涡量约束（找回被耗散的小涡）
2003  Stam《Real-Time Fluid Dynamics for Games》（GDC 版）
2005±  FLIP 引入图形学（Zhu & Bridson 等）→ 影视管线主流
2008  Maya Fluid Effects 获奥斯卡技术成就奖（Stam 为获奖人之一；Stam 1999 的产业化身）
2010s  Houdini FLIP 成为影视/游戏过场标准；EmberGen 等实时烟雾工具兴起
2020s+ 神经流体（学习式仿真/神经涡量；与 [[Neural Physics Simulation]] 合流）
2026  [[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]]：
      水的"感知路线"——连模拟都不做（光流注入 + 无缝循环）★ 2026-10-13 入库
```

## Important Papers

- [[Stam — Stable Fluids (1999)]] —— **CG 流体域枢纽**（无条件稳定：拉格朗日对流 + 隐式扩散 + 投影；"感知优先"哲学出处；Maya Fluid Effects 的理论来源）★ 2026-10-13 入库
- [[Tessendorf 2001 — Simulating Ocean Water]] —— **水面谱合成源头**（"不解方程、采统计"：Phillips 谱 + 一次 IFFT + 色散只管相位；尖锐位移与 Jacobian 泡沫判据；**固定成本、不可剔除**的成本模型；游戏/影视水面二十年事实模板）★ 2026-10-22 入库
- 记名（未单独入库）：Foster-Metaxas 1996/1997（完整 NS 进 CG）；Fedkiw 等 2001（Vorticity Confinement）；Zhu & Bridson 2005（FLIP 进图形学）；Stam 2003 GDC（游戏向实时版）
- 对照样本：[[2026-10-09-Make It Flow — Animating Water in 3D Gaussian Splatting Scenes|Make It Flow]]（2026，感知路线——"不模拟的水"）

## Related Concepts

- [[Particle Systems]]——离散派的祖先（同族成本法则：成本 ∝ 元素数；流体 = 成本 ∝ 单元数 × 步数）
- [[Neural Physics Simulation]]——神经侧：用学习替代求解器（"镜像 I/O"家族）
- [[Participating Media]]——渲染侧接口：流体模拟产出密度场 → 体渲染/自阴影（Stam 1999 的演示即体渲染）
- [[Physics-based Character Animation]]——同为"物理求解进 CG"的支线（布料/肌肉/角色）

## Technologies

- **Houdini FLIP**（影视/过场管线主流；粒子-网格混合）
- **Maya Fluid Effects / Bifröst**（Stam 1999 一系在 DCC 的落地形态）
- **Blender（Mantaflow）**、**EmberGen**（实时烟雾/火焰，烘焙为翻页图）
- **UE Niagara Fluids**（实验插件；网格类 2D/3D Gas/Liquid 模板）——离引擎侧最近的形态
- 游戏内常见"替代路线"：翻页纹理/flipbook、Niagara 粒子烟（离线烘焙演化，运行时播放）——**"实时演化 vs 离线烘焙"的预算分界**

## Game Applications

- **环境与战斗特效**：烟、火、雾、魔法流体、破坏烟尘（多为烘焙管线：Houdini→翻页图）
- **水体**：水面渲染（波面/FFT）≠ 流体求解——**水面是本域的"半独立分支"**（[[Tessendorf 2001 — Simulating Ocean Water|Tessendorf 2001]]：不做流体，只造"水面的样子"）；水花/飞溅走 FLIP 或粒子；氛围水（溪流/瀑布）2026 起出现"感知注入"路线（Make It Flow）
- **交互响应**：需要"跟着玩家/交互动"的流体 → 真实时求解（网格法/轻量 FLIP）或引擎内简化方案

## Personal Knowledge

- 新增概念（2026-10-13 建立）：`user_level: Normal`（**推断**：Houdini/FLIP 的工程使用侧有背景；"数值求解器理论层"按本篇结论层完成，推导层未验证）；
- 与用户工作的接口：**流体特效的预算轴 = 网格分辨率 × 步数**（网格线）或 **粒子数**（FLIP 线）——与五维预算、SABC 配比的"降档=换表示层级"原则一致。

## Learning Gap

- 数值推导层（投影/泊松求解/稳定性分析）暂列按需；
- FLIP 的"粒子-网格"耦合细节（待 FLIP 原始文献入库）；
- 涡量约束/湍流级联（Fedkiw 2001 一线）。

## Next Step

1. （可选，≈25 分钟）读 [[Stam — Stable Fluids (1999)]] 结论层 + 今日图解——"四步分解"即全部要点；
2. （可选，≈25 分钟）读 [[Tessendorf 2001 — Simulating Ocean Water]] 结论层——**只取三件**：谱造初值 / 相位演化 / Jacobian→泡沫；这是"不求解"那条路的全部要点（水花/水面技能可直接搬）；
2. （可选对照）看一遍手中 Houdini FLIP 工程（若在项目中）：**"粒子携带的是什么量、网格解的是什么"**——对照混合法定义；
3. （观察项）EmberGen/引擎内外"实时演化 vs 烘焙"的分界是否随时间推移向实时侧移动。
