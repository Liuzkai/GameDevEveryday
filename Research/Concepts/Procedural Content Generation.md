---
type: concept
user_level: Normal
aliases: [PCG, Procedural Modeling, 程序化生成, 程序化建模]
prerequisites: []
first_introduced: "1970s（分形/噪声线）/ 1975（形状文法线）"
---

# Procedural Content Generation

## Definition

**用算法、规则或过程生成内容，而不是逐件手工制作。** 内容可以是几何（地形、植被、建筑、城市）、纹理、关卡、任务、叙事等一切游戏资产维度。

核心形式化：**把"内容"分解为"生成过程 + 参数/种子"**。

## Core Principle

**为什么游戏开发需要它**（2006 年论文原文给出的理由在今天依然成立）：

1. **规模**：手工程度做不完——[[Müller — Procedural Modeling of Buildings (2006)]] 原文例证：《超人归来》城市模型 = 15 人年；而规则化后**十亿多边形量级的城市可在一天内生成**
2. **多样性**：同一套规则 + 不同随机种子/参数 → 大量变体（城市填充、资产变体）
3. **可复用与可编辑**：生成的是**设计意图**（规则）而非**几何**——这使"设计资产"可以像代码一样复用、审查、迭代

**代价/边界**（同样重要）：

- **规则编写本身是专业技能**（2006 原文自述 "rule authoring is laborious"；2026 年 [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]] 仍在解决这个问题）
- **可能生成不合理的配置**（尤以从任意输入数据出发时为甚）
- **生成的量与运行时的代价是两本账**——离线生成十亿多边形 ≠ 实时能画（[[Müller — Procedural Modeling of Buildings (2006)]] 明确："实时化需要新的 LOD 技术"）

## Prerequisites

- 基础几何与建模概念（体块 / 面 / 网格）
- （文法线）产生式系统 / 形式文法的基本直觉
- （噪声线）基础随机数 / 信号概念

## Historical Evolution

```text
1970s–80s  分形与噪声线：Mandelbrot 分形地形 / Perlin 噪声（1985）
           → 地形、纹理、云的生成基础
1975       形状文法线：Stiny《Shape Grammars》（建筑学，人工推导）
1980s–90s  生长模拟线：L-system 植物（Prusinkiewicz & Lindenmayer 1991）
           → "规则生成自然形态"的范式确立
2001       城市：Parish & Müller《Procedural Modeling of Cities》
           （L-system + 简单体块 + shader 细节；CityEngine 前身）
2003       立面：Wonka《Instant Architecture》（split 规则）
2006       ★ CGA shape：[[Müller — Procedural Modeling of Buildings (2006)]]
           （复杂体块 + 一致细节；两阶段推导；CityEngine 的理论基础）
2007–2011  生产化：Procedural Inc. → CityEngine 商用 → Esri 收购
2010s      （关卡线）WFC / 约束求解进入独立游戏生态；
           Houdini 成为影视/游戏程序化管线主力；
           UE PCG Framework（5.2+）把 PCG 做成引擎内一等公民
2020s+     机器学习线：神经生成（GAN/扩散/NeRF/3DGS）进入资产与场景生成
2026       混合线：[[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]]
           （LLM 解析属性 + 网络推断结构角色 + 检索复用资产）
           —— "不写规则，推断规则的作用对象"
2026-09    混合线（续）：[[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction]]
           （单图 → 可编辑场景：几何/布局双流共生成；布局用"稠密有界对应"替代"稀疏无界位姿"）
           —— "不猜位姿，恢复对应、再交给几何对齐"（同一母题的第三天第三例）
```

## Important Papers

- [[Müller — Procedural Modeling of Buildings (2006)]] —— **本文库的程序化建筑源头**（CGA shape；CityEngine 理论基础）
- [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]] —— **2026 前沿样本**（推理 + 检索的混合范式；"结构锚定拓扑"原则）
- [[2026-09-20-Mira-Scene — Pixel-Aligned Layouts for Generative 3D Scene Reconstruction]] —— **2026 前沿样本（续）**（单图→场景；"稠密有界对应"替代"稀疏无界位姿"）
- 记名（未单独入库）：Stiny 1975《Shape Grammars》；Prusinkiewicz & Lindenmayer 1991（L-system 植物）；Parish & Müller 2001《Procedural Modeling of Cities》；Wonka et al. 2003《Instant Architecture》；Perlin 1985《An Image Synthesizer》

> **"结构锚定"母题三例**（9-21 ~ 9-23 连续入库）：Müller 2006 锚在"二维 scope" → ProxyBuild 锚在"面-边拓扑" → Mira-Scene 锚在"像素 ↔ 规范坐标"。**锚什么可以变，但"生成必须作用在可对齐、可查询、可修正的中间表示上"不变。** Mira-Scene 还给出本母题目前最干净的量化判据：**稠密化不够，"有界"才是关键**（场景空间稠密预测 3D-IoU 0.379 vs 有界规范空间 0.727）。

## Related Concepts

- [[Particle Systems]] —— **程序化生成的同族**：Reeves 1983 的"粒子系统树 + 随机过程"是**动态现象**的程序化生成（火焰/云/爆炸）；建筑/地形是**静态资产**的程序化生成。**共享同一组成本法则**（屏占比 × 密度、层级数量）
- [[Procedural Materials]] / [[Texture Synthesis]]（未入库，记名）—— 材质与纹理维度的程序化生成
- [[World Models for Games]] —— "生成"的另一种口径：世界模型生成的是**体验/状态**，PCG 生成的是**内容**

## Technologies

- **Houdini**（影视/游戏程序化管线主力；SOP + 属性传播）
- **CityEngine / ArcGIS CityEngine**（CGA shape 的商业化身，规则语言即 CGA）
- **UE PCG Framework**（5.2+；图执行框架，可与 Niagara / 地形 / 水体系统交互——**离用户最近的落地形态**）
- **Wave Function Collapse**（约束求解式 PCG 的代表，独立游戏生态常用）
- Blender Geometry Nodes（[[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]] 的资产库即用它生成多风格组件）

## Game Applications

- **开放世界**：地形、植被、道路、聚落（NGR 一类项目的基建）
- **城市/建筑**：CityEngine 一线（本文案例：庞贝重建 / Beverly Hills）
- **资产变体**：同一资产的程序化变体（降本）；ProxyBuild 展示的"同壳不同文本"即风格变体生成
- **关卡/地牢**：WFC 一线（以撒的结合、Bad North 等）
- **动态现象的预生成**：粒子系统的"生成过程"本身（Reeves 1983）

## Personal Knowledge

- `user_level: Normal`（推断：游戏开发背景对"程序化生成"概念不陌生；**形状文法形式化细节与图神经网络不做要求**）
- **库内重要连接**：用户工作（NGR / 开放世界）中 **PCG 生成的场景 = VFX 的承载容器**——PCG 产出多少几何/多少半透明植被，会直接影响运行时特效预算的"底噪"。**这是 PCG 域与你的预算工作的真实交叉点。**
- 另：用户在整理游戏开发知识库时已涉及 Houdini 方向 → 本概念的"技术"清单可直接对接

## Learning Gap

- 无硬性缺口。若要深入：
  - **形式化层**（形状文法的推导/优先级/`r` 相对值机制）→ [[Müller — Procedural Modeling of Buildings (2006)]]
  - **2026 前沿层**（拓扑角色推断、检索组装、NCR/ACC 指标）→ [[2026-09-20-ProxyBuild — Text-Guided Structured 3D Building Generation with Mesh-Anchored Procedural Proxies]]

## Next Step

1. （可选，30 分钟）如果在项目中用过 UE PCG：对照 Müller 2006 的"两阶段"（体块 → 表面 → 细节），检查自己图里"结构决策"与"细节填充"是否分离——**分离则改档/换风格便宜，耦合则贵**。
2. （观察项）ProxyBuild 式的"推断 + 检索"是否渗入引擎工具链（UE PCG 的自动规则生成一类）。
