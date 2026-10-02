---
type: paper
title: "Lens Flare Removal and Reconstruction"
authors: [Tarun Yenamandra, Jonathon Luiten, Daniel Cremers, Nathan Matsuda]
year: 2026
published: "2026-09-30 (arXiv v1, 2609.39527)"
venue: "arXiv Preprint（Meta Reality Labs Research × TU Munich；cs.CV / cs.GR）"
url: "https://arxiv.org/abs/2609.39527"
code: ""
project_page: "（论文标注有项目页；链接未随 arXiv 摘要页放出）"
category: [rendering, computational-photography, gaussian-splatting, vfx, optical-effects]
importance: B+
historical_importance: 0
game_relevance: 4
production_readiness: "Prototype（实时实测 92.7 FPS @ 单光源场景，RTX 4090；对 3DGS 光栅器零改动）"
user_level: Normal
status: unread
tags: [lens-flare, camera-effects, gaussian-splatting, capture, vfx, meta]
---

# Lens Flare Removal and Reconstruction（Meta Reality Labs, 2026）

## TL;DR

**镜头光晕的两个账本，用同一套表示结清：对"清理户"（3D 重建管线）它是污染——用扩散模型去掉；对"艺术家户"它是资产——把它重建为"**相机锚定的、沿'光源—主点连线'排布的 1D 高斯**"，可编辑、可迁移、可与场景一起实时渲染（RTX 4090 实测 **92.7 FPS**，对光栅化成本只 +1.8%）。**

**核心机制**：光晕不是 3D 一致的（它是**镜头系统的产物**，随相机移动）——所以表示也要"不 3D 一致"：高斯**锚定在相机近平面平面上**，每个光源一组**一维规范高斯**（标量径向偏移 μ，沿"主点—光源投影"连线排布，Koreban & Schechner 2009 的几何性质），由**以相机/光源位置为条件的形变 MLP** 调制，渲染时展开为标准 3DGS 图元、与场景**同一趟光栅化**。与 scene 3DGS **联合优化**、scene 分支由去光晕模型的输出监督 → 自动分解为"场景 + 光晕"。

> **一句话定位**：库内**"光学效应 / 相机效应"题域的第一条**——且是"**表示要匹配现象性质**"母题的一个极端样本：3D 一致的用 3D 表示（3DGS），**不 3D 一致的效应用相机空间表示**，两者拼进同一个光栅器。

## Problem

- 光晕 = 相机成像系统属性（散射：灰尘/镜面缺陷/划痕 → 眩光、微光、条纹；反射：镜组互反射 → ghost 光斑），**不是场景内容**。它显著伤害下游 3D 重建质量；
- 既有去光晕方法只处理"围绕光源的小光晕"；**大光晕（覆盖整幅画面）**没有数据集、也做不好；
- 反方向：光晕又是影视/游戏/虚拍的**艺术工具**——但"多视图一致的光晕表示"此前**无人探索**（2D 模拟有，跨视图一致表示没有）。

## Core Idea（四件套）

1. **数据集**：真实大光晕（公开视频素材，如 ActionVFX 光学素材库）+ **程序化生成管线**（随机圆心/半径/颜色的"环状光晕"+ 径向不透明度渐变——专门补数据集缺口）；新 **VFX benchmark**（大反射光晕）+ 沿用 **Flare7K++** benchmark；
2. **去光晕模型**：微调 **Difix3D+** 的单步扩散模型（LoRA；连 VAE 编码器一起微调——因为带光晕图对 VAE 属 OOD），**单卡约 20 小时微调**（基线方法 >4 天）；两个版本：Ours（连光源眩光一起去）/ Ours-wGlare（保留眩光，主观更讨喜但不如前者真实）；
3. **光晕表示（本文最核心）**：
   - 光晕高斯**锚定在相机近平面后的一个平面上**（不是世界空间自由几何）——"光晕随相机走"这一物理事实直接写进表示；
   - 每个光源一组**1D 规范高斯**：标量均值 μ = 沿"主点 p —（修正后）光源投影 l+Δl"连线的径向偏移；**所有高斯被约束在这条 2D 连线上**（几何先验**进结构**）；
   - 小 MLP 以相机/光源位置为条件做形变（另有 δf"更细形变"补偿不对称性）；μ → 2D 点 → 反投影到近平面深度 → **标准 3DGS 图元** → 与场景高斯**一起光栅化（单趟）**；
4. **联合优化 / 分解**：scene 3DGS 用"去光晕模型输出"监督（scene-only 渲染路径），光晕高斯用原始输入监督（合成路径）——一次训练同时得到干净场景与显式光晕。

## Key Data（原文数字）

**去光晕（Table 2, PSNR↑ / LPIPS↓）**：

| 基准 | 最佳基线 | Ours-wGlare | Ours |
|---|---|---|---|
| Flare7K++（散射） | Flare7K++ 26.42 / FlareX 25.26 | **26.68 / 0.0777**（最佳） | 25.76 |
| VFX（大反射，本文新） | FlareX 24.17 / Flare7K++ 21.37 | 25.93 | **27.06 / 0.0659**（最佳） |

（VFX 基准上基线普遍崩——LightsOut 13.04、ACL-FR 20.20——"大光晕把天空细节盖住后，去光晕=重生成天空"。）

**实时渲染（RTX 4090，单光源场景 "hat"，162,283 场景高斯 + 光晕高斯共 173,248）**：
- 形变 pass **3.92 ms** + 联合光栅化 **6.86 ms** = 帧时间 **10.79 ms（92.7 FPS）**；对照 vanilla 3DGS 同场景 6.74 ms（148 FPS）→ **光晕合批只 +1.8% 光栅化成本**；
- 形变 pass 是主开销、**与输出分辨率无关**（按高斯而非按像素）、随光源数增长：**1/2/3 光源 = 3.9 / 6.8 / 11.1 ms**（**每光源一套形变网络——"按光源计费"**）；
- 分解消融：去掉 1D 约束（自由 2D）→ 光晕 PSNR **33.27 → 27.77 dB**（**±5.5 dB 的"结构先验税"**）；对光源检测误差鲁棒（用带噪质心替代检测也只差 ~1.1 dB）。

**数据规模**：去光晕采集 5 个室内外场景（50–110 图/场景，10% 测试）；重建采集覆盖多镜头 + 5 类真实光源（头灯/车间灯/手机闪光灯/路灯/装饰灯），每段 **300–400 训练帧 / 40 测试帧**。

## Limitations（原文自述）

- **环状光晕做不好**：高斯天然"中心比边缘不透明"，重建出的环会**中心发黑**（"表示自身的固有偏好"）；
- 测试视角是**内插**（拍摄受"光晕可见时长"限制，无法远处外推）；分解评估用的是"模型导出参照"（真实 per-pixel 光晕层不可得——物理上纠缠于传感器）。

## Game Development Relevance

- **光学效应首次成为"可重建资产"**：光晕从"后期 2D 贴片/屏幕空间装饰"升级为**显式、可编辑、可迁移到新场景的对象**——对游戏 VFX / 影像级演出（cinematic）是一个新工具形态；原文明确点到 *"games and VR/AR, where view-consistent camera effects are needed"*；
- **对采集管线**：光晕是大场景扫描 / 数字孪生的常见污染源 → "去光晕 = 重建前的清账步骤"；
- **对库**：**"相机空间表示"新题域**（此前的 GS 全部在世界空间）；与 [[Gaussian Splatting]] 排序线不冲突（**复用同一光栅器**，单趟合批）。

## Unreal Engine Relevance

- 无直接 UE 实现；形态提示：**"相机锚定的近平面高斯 + 单趟合批"**对应引擎里"相机附着特效（Camera-attached FX / 后处理几何）"的组织方式——若未来引擎要"物理化镜头特效"，这是一个可参考的表示方案。

## Technology Evolution

```text
2009  Koreban & Schechner：孔径 ghost 光晕的几何性质（位于"光源投影—光心"连线上）
2011+ 物理光晕模拟线：按镜组 prescription 建模（逼真但只对已知镜组、衍射不可算）
2018+ 去光晕学习线：Flare7K（合成）→ Flare7K++（+真实散射）→ BracketFlare（小反射）→ DiffFlare / ACL-FR / FlareX / LightsOut
2023  3DGS 出现；"场景重建里的光晕"成为公害（要么烘焙进场景、要么被丢弃）
2026 ★ 本文：把两条线接上——去光晕（清理）↔ 表示重建（资产）用同一套 1D 相机锚定高斯结清
```

## Relationships

### Based On

- **Koreban & Schechner 2009**（记名）：ghost 光晕位于"光源投影—主点"连线上的几何性质——**本文把它从"观测"变成"参数化约束"**；
- **Deformable 3DGS**（Luiten et al., 记名）：形变高斯框架——本文把"以时间为条件"换成"**以相机+光源位置为条件**"、把 3D 规范高斯换成 **1D**；
- **Difix3D+**（Wu et al. 2025，记名）：单步扩散去光晕骨干（LoRA 微调）。

### Related

- [[Gaussian Splatting]]（库内概念）：本文是其"表示变体"家族的新成员（相机空间 / 1D / 条件形变）；
- **"先验进结构，不进损失"家族**（库内跨篇母题）：[[2026-09-29-Texture Space Material Diffusion|Texture Space Material Diffusion]]（已知投影当归纳偏置）⟷ 本文（**几何性质当参数化约束**——1D 约束消融 ±5.5 dB 给出该家族目前**最直白的量化证据**）；
- **"表示匹配现象性质"**（[[Hair Rendering]] 发片/发丝、[[2026-09-01-ToCo-Mesh — Topology-Consistent Dynamic Mesh Reconstruction via Adaptive Tessellation and Surface-Aligned 2DGS|ToCo-Mesh]] 拓扑一致性）：**不 3D 一致的效应 → 不 3D 一致的表示**（相机空间 + 每帧随相机重排）；
- **"按光源计费"**（形变 pass 1/2/3 光源 = 3.9/6.8/11.1 ms）——与库内"固定项 vs 随场景增长项"成本账本同构（[[2026-09-22-Stochastic GS Denoising — Ultra-fast Neural Inference for Stochastic Gaussian Splatting Denoising|Stochastic GS Denoising]] 的"每像素固定项"对偶）。

## Personal Knowledge State

- `user_level: Normal（接口层）`——读法：**"两本账 + 一个约束"**（清理户/艺术家户；1D 连线约束）即可拿走，形变 MLP 细节不要求；
- 与用户工作交叉点：**VFX 域的新工具形态**（镜头特效的"资产化"）+ 采集清理。

## Learning Value

- **三条可迁移抽象**：
  1. **"同一物理现象的两个账本，可以用同一套表示同时结清"**——去光晕模型反哺表示训练（监督信号），表示反过来给出"干净场景"（分解）——**解法共用**是这篇最值得记的架构级判断；
  2. **"物理先验进结构"的极限样本**：把观测规律（连线性质）直接写成**参数化自由度**（1D μ）——消融 ±5.5 dB 证明这不是装饰；
  3. **"表示跟现象走"**：3D 一致的放世界空间，不 3D 一致的放相机空间——**先问现象的不变群是什么，再选表示**。

## Visualization

（暂无独立图解；机制图见原文 Fig. 6/7。）

## Notes

- 来源核对：arXiv v1 全文（HTML 版逐节核对，含 Table 2 / Fig.10 数字；正文 math 标签数值单独提取核对）。
- "窗口先见"状态：9-30 提交，10-1 公告组放出；Run #26（10-2）入库。
- Meta Reality Labs Research（工作含实习期成果）+ TUM（Cremers 组）。项目页链接未随摘要页放出（待补）。
