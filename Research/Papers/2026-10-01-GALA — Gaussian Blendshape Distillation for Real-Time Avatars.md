---
type: paper
title: "One Basis to Animate Them All: Gaussian Blendshape Distillation for Real-Time Avatars"
authors: [Ramazan Fazylov, Stamatis Lefkimmiatis, Ivan Laptev]
year: 2026
published: "2026-10-01（arXiv v1）"
venue: "arXiv Preprint（cs.CV；MBZUAI × MWS AI）"
url: "https://arxiv.org/abs/2610.02207"
code: "（项目页标注有 Code；见 project_page）"
project_page: "https://ramazan793.github.io/gala/"
category: [gaussian-splatting, avatars, animation, distillation, real-time, mobile]
importance: B+
historical_importance: 0
game_relevance: 4
production_readiness: "Prototype（三宿主模型实测：CPU 动画最高 ×2,659 加速；手机浏览器 60 fps 演示；宿主零重训、工程接缝干净——但依赖具体宿主模型）"
user_level: "Normal"
status: unread
tags: [gaussian-avatars, distillation, blendshape, real-time, mobile, facial-animation, linear-basis]
---

# GALA — One Basis to Animate Them All（Fazylov, Lefkimmiatis & Laptev 2026）

## TL;DR

**3D 高斯化身"渲染便宜、动画昂贵"的不对称被一条线性结构抹平了：预训练化身模型的动画输出 ≈ 一套"身份无关的共享 blendshape 基"的线性组合。** GALA 把宿主模型（host）蒸馏成 **一个共享基 + 一个浅层 MLP 系数预测器**——每帧只跑小网络 + 线性混合，**宿主零重训**。三个异构宿主实测：Face 3D GAN **154 → 5.5 ms（×28）**、3D 头部 transformer **234 → 4.4 ms（×53）**、穿衣全身 transformer **42.9 s → 16.1 ms（×2,659，含蒙皮）**；**手机浏览器 60 fps** 跑通全部三个宿主。

> **一句话定位**：库内"**神经网络 → 传统资产形态**"的实时化样本——把"每帧重跑大网络"换成"**基 + 系数**"（blendshape 是动画行业最老的资产形态之一）。与毛发"**一条原型路径代表全部路径**"（Zinke 2008）、Instant Radiosity"**一组点光源代表整个光场**"（1997）跨域同构：**"选对表示，让贵的东西不升复杂度"家族的新成员。**

## Problem（不对称）

- **高斯化身**：渲染（光栅化）视角切换极快——新视角 **622 fps**；但**动画**要为每个新姿势重跑大网络——**1.8 s，即便在桌面 GPU 上**（DynaAvatar on RTX 6000 Ada）；
- 所以部署瓶颈不在"画"而在"动"：实时应用（视频会议、XR、游戏）需要每帧毫秒级动画；
- 既有蒸馏路线要么重训宿主、要么换架构——本文问的是：**能不能不碰宿主、且把推理换成更朴素的形式？**

## Core Idea（三步蒸馏，宿主零重训）

```text
① 基构造（Basis construction）
   在大量身份 × 姿势上跑宿主 → 收集"动画态 − 中性态"的残差
   所有残差拼成一个矩阵 → 块局部 PCA（block-local PCA）
   度量：rendering-aware metric（按渲染影响加权）；约束：memory budget
   → 一套基 U（同一模型的所有身份共享；对未见身份同样成立）
② 系数蒸馏（Coefficient distillation）
   浅层 MLP：从（身份 + 驱动信号）预测系数
③ 推理（Inference）
   中性态化身：每个主体算一次
   每帧：小网络预测系数 → 线性混合（+蒙皮）→ 得到动画态
```

**关键实证发现（本文最值钱的一条）**：*"Animation is a linear combination of basis vectors"*——动画残差可被**身份无关**的共享基逼近；**权重取决于身份与姿势，基不取决于身份**（"It even holds for identities never seen before"）。同一套基里，"闭眼"的基向量在身份 1/2/3 上都是闭眼，"抬夹克"的基向量跨身份成立（项目页交互演示）。

**工程细节**：块局部 PCA 的"块"= 局部面部区域 / 服装部件——使每个基向量**局部可解释**（对应某个部位的运动）；rendering-aware 度量在同等内存预算下把"对宿主的 PSNR"从简单 PCA 的 **32.8 dB 提到 42.6 dB**（AGORA 宿主）。

## Key Data（项目页 + 摘要数字）

| 宿主（异构三类） | 宿主动画耗时 | GALA | 加速比 |
|---|---|---|---|
| AGORA（Face 3D GAN / CNN） | 154 ms | **5.5 ms** | ×28 |
| FlexAvatar（one-shot 3D 头部 transformer） | 234 ms | **4.4 ms** | ×53 |
| DynaAvatar（穿衣全身 transformer） | 42.9 s | **16.1 ms**（含蒙皮） | **×2,659** |

- **×2,659 = 目前"CPU 动画"最大加速记录**（项目页口径）；全部宿主**零重训**；
- 质量：多数指标超 SOTA 蒸馏网络——FID **3.47 @ 5.5 ms** vs 最佳蒸馏网络 **3.60 @ 124 ms**（**快 22× 且更准**）；LPIPS 维度仅有一个蒸馏网络更准，但**慢 40×**（0.100 @ 175 ms vs 0.109 @ 4.4 ms）；
- 端侧：**手机浏览器 60 fps**（全部三个宿主）；基的规模 = "1 basis per model, shared by all identities"。

## Limitations

- **一模型一基**：每个宿主一套基（不是跨模型万能基）；"One Basis to Animate Them All"指"一个模型的**所有身份**共用一套基"；
- 线性近似的上限 = 宿主表示自身的线性结构强度（本文证明了三类架构都有，但**这是经验发现，不是定理**）；
- 基构造需要"跑宿主 × 大量身份 × 姿势"的采样成本（蒸馏前置数据）；rendering-aware 度量与内存预算的具体折衷未在摘要层展开；
- 无引擎集成、无生产管线验证（学术原型 + Web 演示）。

## Why It Works（本库读法）

1. **"表示选择"的又一次胜利**：不改进网络、不改进训练——**换一个能整除成本的表示**（基 + 系数），让"每帧重算"变成"每帧查表"；与 Zinke-Yuksel"方差可加"同构：**选对表示，多次/多帧不升复杂度**；
2. **"局部 + 共享"分离了两个变量**：身份进系数（权重）、部位进基（向量）——**把耦合的两个维度拆到两套参数里**（与 Wonka"规则只管结构、参数外部决定"同族：**拆出去、给独立表示**）；
3. **"蒸馏到传统资产形态"**：blendshape 是动画工业最古老可维护的资产之一——**把神经资产翻译回传统资产**，显著降低运行时的"神经层依赖"（与 E-Day 主动放弃 Ray Reconstruction 的"神经层边界"思考互补：**该换线性就换线性**）。

## Game Development Relevance

- **实时化身 / 数字人的移动端路线**：60 fps 手机浏览器是货真价实的移动档样本——与库内移动端线索（[[Real-Time Global Illumination]] 的《Neural Dawn》、Gaussian Stippling 的 NPU 重建）构成"移动端神经资产"三连；
- **"蒸馏配方"可迁移**：任何"每帧重跑网络"的资产（化身、布料、面部）都可先问：**它的输出空间是否线性可逼近？** 若是 → 基 + 浅层预测器是最便宜的实时化形态；
- **对 VFX / 角色管线**：面部动画的 blendshape 语言（Morph Target）是引擎原生概念——**这条路线与引擎资产的接缝天然存在**（不像纯神经资产需要"黑盒运行时"）。

## Unreal Engine Relevance

- 形态映射：**共享基 ≈ UE 的 Morph Target / Pose Asset；浅层 MLP ≈ 一个极轻的运行时求值器**——理论上可把参数烘焙为 Curve 驱动（把"神经"彻底消掉）；
- 对照库内 [[Neural Animation]]（Hard）与 [[Real-Time Generative Motion]]（Technology）：GALA 提供的是"**神经动画实时化**"的一条具体梯子（蒸馏 → 线性）；若做面部/全身动画工具，这是"神经 + 传统"混合架构的参考实现。

## Technology Evolution

```text
2023 3DGS → 3D 高斯化身（渲染快、动画慢的不对称出生）
2024-25  化身动画线：逐帧神经解码 / 高斯绑定骨架 / 变形网络（实时性靠重型工程）
2026 ★ 本文：蒸馏成"共享线性基 + 浅层 MLP"——把每帧成本压到毫秒级（手机 60 fps）
         ⟂ 并行线索：库内"神经 → 传统资产"翻译的另一样本
           （[[2026-09-17-HairCS — Reconstructing Strand-Based Hair from Hair Cards|HairCS]] 发片→发丝是"传统→神经"，
            GALA 是"神经→传统"——两个方向都有了）
```

## Relationships

### Related

- [[Neural Animation]]（Hard）——本笔记是它"实时化缺口"的桥材料（"蒸馏成线性"是 Hard → Normal 的可行梯子）；
- [[Gaussian Splatting]]（Easy）——宿主模型的表示基础（高斯化身是 GS 的资产化分支）；
- **"一个表示代表全部"家族**（跨域）：[[Keller — Instant Radiosity (1997)]]（**一组点光源代表整个光场**）· [[Zinke-Yuksel — Dual Scattering Approximation for Fast Multiple Scattering in Hair (2008)|Zinke-Yuksel 2008]]（一条原型路径代表全部路径）· **GALA**（**一套基代表全部身份**）——**三次出现，同一句式："把无限/多样的量，折叠进一个有限、可复用的表示"**；
- **"蒸馏/降级为可维护形态"**：库内 [[2026-09-25-ReFM — Semantic-Aware Refinement Flow Model for Motion Retargeting|ReFM]]（HumanIK 降级为初始化）反向样本——GALA 是"神经降级为线性"，ReFM 是"传统降级为初始化"：**降级的对象不同，判据相同：看它的错误是否与正确信息缠在一起**。

## Personal Knowledge State

- `user_level: Normal（接口层）`——读法：**"共享基 + 系数预测 + 每帧线性混合"三件事**，块局部 PCA / 度量细节不要求；
- 与你的工作交叉点：移动端性能预算（60 fps 手机样本）+ 面部/角色动画工具的架构参考。

## Learning Value

**三条可迁移抽象**：

1. **"线性可逼近性"是该问的第一个问题**——面对"每帧重跑网络"的资产：*它的输出残差是否近似线性结构（身份×姿势可分离）？* 是 → 基 + 浅层预测器；
2. **"局部可解释的基"优于整体基**：块局部 PCA 让每个基向量对应一个可命名部位——**可解释性在这里不是为了理解，是为了"局部纠错/局部降级"的工程空间**（一个部位的基坏掉不毁全脸）；
3. **"度量要按下游影响加权"**（rendering-aware metric）：基的构造不优化回归误差、优化**渲染误差**——**"优化到眼睛"家族新样本**（[[2026-09-25-ControlGS — Conditioning Neural Gaussians for Downstream-Processing-Aware XR Rendering|ControlGS]] 同族）。

## Visualization

（暂无独立图解；机制演示见项目页交互 Demo。）

## Notes

- **来源核对**：arXiv 摘要页 + 项目页（ramazan793.github.io/gala/）逐节核对；数字均取自摘要与项目页明示值（×2,659 / 60 fps / 154→5.5 ms / 234→4.4 ms / 42.9 s→16.1 ms / PSNR 42.6 vs 32.8 dB / FID 3.47@5.5ms vs 3.60@124ms）。
- 作者归属：MBZUAI（Fazylov / Laptev）× MWS AI（Lefkimmiatis）；宿主：AGORA（Fazylov et al.）、FlexAvatar（Kirschstein et al., CVPR 2026）、DynaAvatar（Kwon et al., CVPR 2026）。
- 窗口状态：10-1 提交、Fri 10-2 公告组放出——本运行（10-3）入库。
