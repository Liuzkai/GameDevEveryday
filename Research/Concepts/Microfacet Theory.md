---
type: concept
user_level: Normal
aliases: [Microfacet Models, 微面理论, Microsurface Theory, Cook-Torrance Model]
prerequisites: [Rendering Equation, BRDF]
first_introduced: "Torrance-Sparrow 1967（物理光学）；Blinn 1977 引入 CG"
---

# Microfacet Theory

> 你正在研读的 D · G · F，就是这一套东西。这是你当前学习线的**目标节点**。

## Definition

把粗糙表面建模为**大量完美镜面的微小面片（microfacet）的集合**，用这些微面法线的统计分布 $D(m)$ 来描述粗糙度，从而把"表面看起来什么样"化归为"微面朝向的统计 + 几何遮挡的统计 + Fresnel"。

它不是一个具体的 BRDF，而是**一族** BRDF 的共同骨架。[[BRDF]] 是问题，微面理论是最成功的那个答案。

## Core Principle

微面 BRDF 的标准形式（反射）：

$$f_r(l,v) = \frac{D(h)\,G(l,v,h)\,F(v,h)}{4\,(n\cdot l)\,(n\cdot v)}$$

三个因子各管一件事，缺一不可：

| 因子 | 名字 | 物理含义 | 不管它会怎样 |
|---|---|---|---|
| **D** | Normal Distribution Function 法线分布 | 有多少微面的法线恰好指向半程向量 $h$（即"能把你看到的方向和光的方向连起来"的微面有多少） | 高光位置/形状全错 |
| **G** | Geometry / Masking-Shadowing 几何遮蔽 | 这些微面里有多少既**没被别的微面挡住**（masking，对观察者）又**没被别的微面遮住光**（shadowing，对光源） | 掠射角整体偏亮，能量不守恒 |
| **F** | Fresnel 菲涅尔 | 光在这个入射角下有多少被反射而非折射进去 | 掠射角反射强度不对（水在远处像镜子） |

分母 $4(n\cdot l)(n\cdot v)$ 是**雅可比**——把"微面法线空间的密度"换算到"方向空间"的修正项，不是凑出来的常数。

### 三个常被忽略的点

1. **D 的归一化是对投影面积做的**：$\int D(m)\,(n\cdot m)\,d\omega_m = 1$，不是 $\int D(m)\,d\omega_m = 1$。差这个 $(n\cdot m)$，很多推导会对不上。
2. **"可见法线分布" $D_\omega(m)$ 比 $D(m)$ 更本质**：真正参与着色的是"从当前方向**看得见**的微面法线分布"，$D_\omega(m) = D(m)\,G_1(v,m)\,(v\cdot m)/(v\cdot n)$。现代重要性采样都是采 $D_\omega$ 而不是 $D$——这解释了为什么"采样可见法线"比"采样法线再判可见"高效得多。
3. **经典形式只算单次散射**：光在微面之间弹第二次就被丢掉了，所以粗糙度越高能量损失越明显（表现为"高粗糙度变暗"）。多次散射修正是后来补的。

## Prerequisites

- [[Rendering Equation]] — $f_r$ 就是这里的核
- [[BRDF]] — 微面模型是 BRDF 的一族具体实现
- 半程向量 $h = \mathrm{normalize}(l+v)$ 的几何直觉
- Fresnel 效应（掠射角反射增强）

## Historical Evolution

```text
Torrance-Sparrow 1967 —— 物理光学，微面理论原型（1D 高斯遮蔽模型）
        ↓
Blinn 1977 —— 引入计算机图形学
        ↓
★ Cook-Torrance 1981 —— D·G·F 三因子齐备 + 能量守恒，完整物理框架
        ↓
★ Walter et al. 2007 —— 引入 GGX（长尾，更贴实测）+ 系统讨论 Smith height-correlated G
        ↓
Disney Principled BRDF 2012 —— 生产参数化（"原则"优先于"物理")
        ↓
Karis 2013（UE4 Real Shading）—— GGX + Smith + split-sum，进引擎，成为工业默认
        ↓
Heitz 2014 —— 把 masking-shadowing 讲透（可见法线分布 D_ω 的形式化）
        ↓
多次散射修正（2016 前后）—— 补回高粗糙度丢失的能量
        ↓
Substrate（UE 5.2+）—— 多 lobe 可组合化，微面仍是底层积木
        ↓
★ 2026 —— 从随机几何（GPIS）把 Beckmann / GGX / Smith 重新推导为特例
         （[[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]]）
```

这条链上有一个很好看的对称：**1981 年 Torrance 提出框架，2007 年 Torrance 本人署名把它推广到折射（GGX 出自那里），2026 年它被从更一般的随机几何里重新推导出来。**

## Important Papers

| 论文 | 角色 |
|---|---|
| [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] | 框架来源：D·G·F 与能量守恒 |
| [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] | **GGX 与 Smith height-correlated 的来源**——你实际在用的那个 D 和 G |
| Heitz 2014《Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs》 | 把 G 与可见法线分布讲透（推荐作为 G 项的深入读物） |
| [[2026-09-16-Gaussian Process Implicit Surfaces as Participating Media]] | 2026：D 与 G 被从随机几何反推，Smith 成为 height-field 极限 |

## Related Concepts

- [[BRDF]]（上位概念）
- [[Physically Based Rendering]]（工程化后的形态）
- [[Participating Media]]（对偶：微面是"随机表面"，介质是"随机体积"；2026 的工作把两者打通）
- [[Inverse Rendering]]（微面参数是逆渲染要反解的目标）

## Technologies

- UE 默认 Lit Shading Model（D_GGX × Vis_SmithJoint × F_Schlick）
- Unity / Filament / glTF / three.js —— 全部收敛到 GGX + 金属度工作流
- Substrate（UE 5.2+）—— 多 lobe 组合，底层仍是微面

## Game Applications

- **材质着色**：Roughness / Metallic 滑杆最终都汇入 D·G·F 求值；
- **性能预算**：D·G·F 是**每像素固定开销**；多一个 lobe（Clear Coat / Cloth / Hair / Subsurface）= 乘一份开销。所以 **Shading Model 的种类数是 DrawCall 之外的隐藏预算维度**；
- **调参手感**：GGX 的长尾让 Roughness 滑杆更线性可预测——这是它被选中的**工程理由**，不只是物理理由。

## Personal Knowledge

**Normal，正在研读中**（2026-09-14 起的 Cook-Torrance D/G/F 学习线，今日 2026-09-16 补上 Walter 2007 锚点）。

已具备：[[Rendering Equation]]（容器）、[[BRDF]]（数学形式）、[[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]]（框架）。

## Learning Gap

缺的是**"我实际在用的是哪一个 D、哪一个 G"**——即 Walter 2007 之后的实时化路径（Karis 2013 的 split-sum 与各种近似）。补上这一环，[[Physically Based Rendering]] 才能收口为 Easy。

## Next Step

1. 读 [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]]，**只看分布与 shadowing-masking 两节**，跳过折射（BTDF）部分；
2. 过一遍它下面的 **5 条 Mastery 自测**；
3. 通过后可把本笔记与 [[Physically Based Rendering]] 一并标 Easy，届时我会停止 PBR 基础推送，转向"PBR 之上的新研究"（如 Substrate 多 lobe 的性能风险、神经材质压缩）。

## Mastery Criteria

见 [[Walter — Microfacet Models for Refraction through Rough Surfaces (2007)]] 中的 5 条自测表（与 [[Cook-Torrance — A Reflectance Model for Computer Graphics (1981)]] 的 5 条合并即为 PBR 收口清单）。
