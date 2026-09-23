---
type: concept
user_level: Normal
aliases: [Linear Transport Theory, Transport Theory, 线性输运理论, 输运理论, 线性玻尔兹曼方程]
prerequisites: ["[[Rendering Equation]]", "[[Participating Media]]"]
first_introduced: "1940s–1960s（中子输运与天体物理辐射传输；图形学侧 1986 起借用）"
---

# Linear Transport Theory

## Definition

**描述"中性粒子或能量在介质中如何输运"的数学理论**——它的中心方程是**线性 Boltzmann（输运）方程**：粒子与背景介质相互作用、但**粒子之间不相互作用**（线性），相互作用只有"吸收 / 散射 / 逃逸"三类随机事件。

一句话原文（本书引自 Pomraning 2002）：**"这个方程代数上复杂，但物理内容极简单——它只是相空间中粒子守恒的数学表述。"**

在图形学语境下，它是**一整套问题的公共母语**：

```text
表面上的微面多次散射  ┐
体积里的多次散射      ├─ 都是同一个方程在不同边界/几何/核下的"模型问题"
次表面散射 / 毛发     │
云 / 烟 / 雾          ┘
```

## Core Principle

**为什么一个方程能同时管这么多东西**：因为建模的对象不是"光"，而是"**随机游走**"。

- **基本量**：截面（吸收 / 散射）、自由程分布、相位函数（散射核）、源项、边界条件；
- **"模型问题"（model problems）**：简单到值得解、又足够代表一整族的场景——**点源 Green 函数、半空间 / 平板（slab）反照率问题（albedo problem）、searchlight、球几何**；
- **两类解法传统**：
  - **确定性**：Fredholm / Wiener-Hopf 积分方程、Chandrasekhar 伪问题与 H 函数、Case 奇异本征函数、各类扩散近似；
  - **统计（Monte Carlo）**：直接模拟随机游走——**"确定性解与 MC 互相验证"是这套理论最可靠的自我审查方式**；
- **一个反复出现的手法**：**把一个表面问题重写成体积问题**——粗糙表面的多次散射 = 半空间里的随机游走（带非标准截面）；毛发 / 纤维 = 细长散射体；这说明**"表面"与"体积"只是同一方程的两种边界**。

## Prerequisites

- [[Rendering Equation]]（**表面形式的同族方程**；体积形式即线性 Boltzmann 方程）
- [[Participating Media]]（本理论在图形学里最直接的落地）
- 概率密度 / 随机过程的直觉（自由程分布、随机游走）

## Historical Evolution

```text
1940s-50s  中子输运（反应堆）+ 天体物理辐射传输 两条线独立成熟
              ↓
1960       Chandrasekhar《Radiative Transfer》：H 函数 / 半空间反照率问题
1967       Case & Zweifel：奇异本征函数方法（"Caseology"）
              ↓
1960s-80s  基准解汇编传统（Ganapol 等）；同一批问题三个学科各解一遍、记号互不通
              ↓
1986       ★ 图形学借入：Kajiya 渲染方程（表面形式的"守恒式"）
1990s      Hanrahan-Krueger（体积光照）；Jensen（次表面散射）
              ↓
2010       Jakob et al. microflake 理论（微片介质的严格输运框架）
              ↓
2016       ★ Heitz et al.：Smith 随机输运——表面微面散射严格化为体积问题
2016-22    ★★ d'Eon《A Hitchhiker's Guide to Multiple Scattering》
              —— 把三个学科的"已解问题"收进一套记号 + MC 交叉验证 + 基准值
              ↓
2026       库内线（Kulla-Conty / Fdez-Agüera / Dupuy / GPIS）
              = 本书所收模型问题的现代工程化与再推导
```

## Important Papers

| 节点 | 角色 | 状态 |
|---|---|---|
| Chandrasekhar 1960《Radiative Transfer》 | H 函数 / 半空间反照率问题的经典源头 | 记名（书内引用） |
| [[Kajiya — The Rendering Equation (1986)]] | 图形学侧"守恒式"的起点（表面形式） | ✅ 入库 |
| [[d'Eon — A Hitchhiker's Guide to Multiple Scattering (2022)]] | **本理论的"地图册"**：问题普查 + 基准值 + 2000+ 条文献导向 | ✅ **2026-09-23 入库** |
| [[Heitz — Multiple-Scattering Microfacet BSDFs with the Smith Model (2016)]] | 表面微面散射严格化的里程碑（σ(u) 随机游走） | ✅ 入库 |
| [[2026-09-18-An Elementary Expression for Multiple Scattering in Homogeneous Microflake Media\|Dupuy 2026]] | 半无限微片介质随机游走逃逸分布的精确闭式 | ✅ 入库 |
| [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media\|GPIS 2026]] | 从随机几何重推 GGX / Smith（"假设降级为极限"） | ✅ 入库 |

## Related Concepts

- [[Rendering Equation]] —— **表面形式 ↔ 体积形式**：同一守恒式的两种边界表达；Kajiya 1986 是图形学第一次把它写下来（该文参考文献 [22] 等亦指向输运传统）；
- [[Participating Media]] —— 本理论在图形学里的**第一落地**（云 / 烟 / 雾 / 水体）；
- [[Multiple Scattering and Energy Compensation]] —— **本理论在"表面"上的特化**：被遮挡 ≠ 被吸收；
- [[Microfacet Theory]] —— 提供表面几何统计（NDF / Λ 函数），是"表面→体积"翻译的输入；
- [[Hair Rendering]] —— **"非经典"介质的候选**：细长散射体、强前向散射——与书内第 XV 部分（非指数自由程）同方向。

## Technologies

- **Monte Carlo 路径追踪**（输运方程最通用的求解器；图形学的"主力微分方程求解机"）
- **体积渲染**（participating media 的实时/离线实现）
- **多次散射补偿项**（引擎里的 energy compensation 开关 = 本章理论的一个 4KB 近似）
- **diffusion / H 函数类解析近似**（医学成像、大气辐射长期在用）

## Game Applications

- **VFX 与环艺**：云 / 烟 / 雾 / 尘 / 水的观感本质上都是输运问题——**"谁被谁遮住、遮住之后去哪"**；
- **材质**：能量补偿（本库已闭环的线）就是"表面版"的输运账；
- **判断"一个近似够不够用"**：本理论的"模型问题 + 基准值"传统，是**评估任何近似廉价方案**的最强工具（先看它偏离基准解多远，再谈够不够）；
- **与预算工作的关系**：间接但真实——**输运问题是"为什么某些效果天然贵"的第一原理来源**（每一次散射事件 = 一次额外的积分/采样；这不随硬件消失）。

## Personal Knowledge

- `user_level: Normal`（**结论层 / 检索层**）——推导层（Fredholm / Wiener-Hopf / H 函数）标 **Hard，不必现在动**；
- **本概念的建立理由**：库内已有 7+ 篇笔记各自处理"输运的某个特例"，但**没有共同坐标原点**；本书入库的同时建立本概念，使"我的问题在输运理论里叫什么、解过没有"成为可检索的问题；
- **一句话检验**：能说出"**微面多次散射和体积多次散射是同一个方程的两个特例**"即到位。

## Learning Gap

- **不需要**：Fredholm / Wiener-Hopf 方程的推导、H 函数构造（Hard 层，挂着即可）；
- **需要**（Normal，30 分钟级）：熟悉"模型问题"的命名法（几何 × 散射类型 × 源）——**会查，比会推重要**；
- **可选**（Easy 层）：把"随机游走"直觉与 [[Particle Systems]] 的粒子循环对照——**粒子系统是"可视化了的随机游走"**。

## Next Step

1. **做 furnace test 时：先在 [[d'Eon — A Hitchhiker's Guide to Multiple Scattering (2022)]] 的 13.7.1 / 13.8.1 查到对应粗糙度与 IOR 的球面 albedo 值**，再与引擎实测对照（这是本概念带来的第一个可执行动作）；
2. 把本概念加入 [[Multiple Scattering and Energy Compensation]]、[[Participating Media]] 的 Related 列表（建立双向链接）；
3. （观察项）第 XV 部分"非经典输运"何时进图形学工程实践——**毛发的相关介质建模**是最近的候选观察点。

## Notes

- **为什么 2026-09-23 才建**：在此之前，库内材料都是"从一个特例走向另一个特例"，缺一本把它们收进同一坐标的原著；本书入库使本概念第一次有了可引用的依托。**本概念是"先有实例、后有母题"的——按需建立，不提前造概念。**
