---
type: paper
title: "Streamlining IBL Workflows with Computer Vision and USD（DNEG iblManager）"
authors: ["Stefano Cieri", "Alexander Schwank"]
year: 2019
published: 2019
venue: "ACM SIGGRAPH 2019 Talks (SIGGRAPH '19), 2 pages"
url: "https://www.dneg.com/wp-content/uploads/2022/11/2019-2.pdf"
code: "闭源（DNEG 内部 Nuke Gizmo + USD LightCache）"
category: ["light-probes", "light-extraction", "image-based-lighting", "usd", "vfx-pipeline", "production"]
importance: B+
historical_importance: 3
game_relevance: 3
production_readiness: "Production（DNEG 实际管线部件，iblManager 与 LightCache 均已上线使用；《Venom》(2018) 制作驱动开发）"
user_level: Easy
status: unread
---

# Streamlining IBL Workflows with Computer Vision and USD（DNEG iblManager）

## TL;DR

**影视工业界"HDRI → 可编辑灯阵"的产品化样本（DNEG，2019）。** 两个部件：

1. **iblManager**：Nuke 内的三个 Gizmo（`iblLightFind` / `iblLightExtract` / `iblRigPublish`）——阈值分割出发光区域、OpenCV 找轮廓与最小外接矩形（顺带得到灯的 roll 角）、逐灯微调后发布；
2. **LightCache**：**基于 USD 的跨 DCC 灯光描述**——在 Maya / Clarisse / Houdini / Nuke 之间传递同一份灯阵，扩展 USD 原生 light schema 补齐缺失属性，DCC 间灯光类型不完全兼容处仍需手工微调。

对「灯光迁移」专题：落 **② 参数化（面光：位置/尺寸/旋转/纹理）与 ⑥ 落地约束（美术可控的半自动、管线集成、跨 DCC 交换格式）**。它回答了一个学术论文不写的问题：**"拟合/抽取出来的灯"该用什么数据结构在工具链里流动**——DNEG 的答案是 USD。

## Problem

DNEG 旧流程：美术在 Nuke 里手工从 HDRI 抠灯 → 迭代慢、易出错；且灯阵与 HDRI 之间**没有依赖追踪**（HDRI 更新了，灯阵不会联动），流程没进管线。目标：让"从 HDRI 出灯阵"半自动化、可追踪、跨 DCC 可用。

## Historical Context

影视侧"HDRI 抽灯"的谱系：Digital Domain 的 Light Kit（Mihashi et al. 2009，《本杰明·巴顿奇事》）→ ILM（Snow 2010）→ ILM LightCraft（Suter et al. 2016，双高度 HDRI 对 + Mars 摄影测量解灯位）。DNEG 认为 LightCraft 对**多灯场景扩展性差**、且要把现场勘察时间翻倍（两次拍摄），因此另走单张 HDRI + 计算机视觉路线。2019 年 USD 正处于成为影视交换格式事实标准的进程中，DNEG 选择不造私有框架而是扩展 USD light schema。

## Previous Work

- Debevec 2006 median-cut（[[Debevec 2006 — A Median Cut Algorithm for Light Probe Sampling]]）——论文明言："its programmatic approach to HDRI analysis is an effective strategy worth recovering"（其程序化 HDRI 分析策略值得恢复）；
- Mihashi et al. 2009（DD Light Kit）、Snow 2010（ILM）——带纹理面光的工业起点；
- Suter et al. 2016（LightCraft）——最近的前作，因扩展性与现场成本被弃用。

## Core Idea

**检测交给 CV，决策留给美术。** 算法只负责"找出候选发光区域"（亮度阈值 + 最小/最大面积 + dilate 因子聚拢相邻小光源），美术通过高层参数实时预览并修正算法的选择；确认后才逐灯生成 `iblLightExtract` 节点微调。——"全自动"不是目标，**缩短迭代周期**才是。

## Technical Approach

- **预处理**：HDR → 只保留能量阈值以上的 LDR 图（发光区与非发光区天然好分）；
- **检测**：OpenCV 轮廓提取 + 最小外接矩形（给出灯的位置、尺寸、roll 角）；dilate 因子把靠近的小发光体聚成一盏面光；
- **微调**：每盏灯一个 `iblLightExtract` 节点，可改位置/尺寸/旋转；
- **发布**：`iblRigPublish` 导出纹理 + 生成 USD LightCache，可在 Clarisse 预览或入库；
- **LightCache schema**：USD 原生 light schema 覆盖不了的属性做扩展；DCC 特定的导入/导出转换器保证灯阵可移植（但论文承认：DCC 间灯型不完全兼容时，**close visual matching 仍需手工微调**）。

## Key Contribution

1. 把"HDRI 抽灯"从手工活变成**参数化的 CV 检测 + 美术监督**的半自动流程；
2. 给出"灯阵作为管线资产"的实现：**USD LightCache**——灯阵有 schema、可版本化、可跨 DCC、可被依赖追踪；
3. 诚实的工程边界记录：格式交换不能消除 DCC 灯型差异，末端仍需人工对齐。

## Why It Works

发光区域在 HDR 域与背景的数值差距巨大，阈值分割"trivial, fast and reliable"（原文）；复杂度全部转移到"参数给美术调"的交互设计上。USD 提供现成的 light schema 与扩展机制，避免造私有格式的轮子。

## Limitations

- 灯距/深度**不在本文范围**——HDRI 无深度，灯只能放在球面上（后来的 Sony Spherelight 用单目深度估计补这一步，见收集页清单 Shlyaev 2025）；
- DCC 间灯型不兼容处仍需手工微调（原文明确承认）；
- 检测对"什么算一盏灯"的语义判断仍依赖美术参数（dilate、面积上下限）；
- 2 页 Talk，无定量评估（无误差数字、无时间对比数字——"快很多"是定性表述）。

## Game Development Relevance

- 迁移工具的**交互形态**参考：检测自动化 + 美术高层参数 + 逐灯微调，比"一键出结果"更符合美术工作流（呼应收集页"工具形态：一键 vs 半自动"的第 3 个决定）；
- **灯阵交换格式**问题在游戏管线同样存在（编辑器 ↔ DCC ↔ 引擎），USD LightCache 是现成的参考实现思路；
- 阈值+轮廓+dilate 的检测三件套可直接用于"从旧 HDRI 探针恢复灯"的第一阶段。

## Unreal Engine Relevance

UE 有 USD Stage 支持，USD light 属性可映射到 UE 灯组件；若迁移管线经 USD 中转，本文的 schema 扩展经验（原生 schema 不够就扩、DCC 差异列清单）直接适用。UE 内对应的消费端是 Rect Light（对应面光）。

## Technology Evolution

上游：DD Light Kit 2009 → ILM LightCraft 2016（半自动、要双高度拍摄）→ **本文 2019（单张 HDRI + CV + USD）** → Sony Spherelight 2025（再加单目深度估计定灯距、Kabsch-Umeyama 对齐 Lidar，"几小时→约一分钟"）。下游另有独立工具 HDRI Light Export for Nuke（Daniele Tosti，免费发布，EV/lux/3D 位置元数据 + Depth Anything V2 可选深度），思路与 iblManager 同族且**个人可直接获取**（见收集页清单）。

## Relationships

- [[Debevec 2006 — A Median Cut Algorithm for Light Probe Sampling]]：明言继承其程序化分析策略；
- [[Sloan 2008 — Stupid Spherical Harmonics Tricks]]：同族不同源（SH 路线）；
- 收集页清单中 LightCraft 2016、Shlyaev 2025（Sony Spherelight）、HDRI Light Export：同一"HDRI→灯阵"谱系的兄弟节点；
- [[Lipp 2024 — View-Independent Adjoint Light Tracing for Lighting Design Optimization]]：学术连续优化路线 vs 本文工业检测路线，目标数据不同（lightmap vs HDRI）。

## Personal Knowledge State

正文为 2 页 Talk；**官方 PDF 直连被拦（403 Forbidden）**，内容经两处独立检索快照交叉核验（摘要、§1 Rethinking Light Extraction、§1.1 Related Work、§1.2 Employing Computer Vision、§2 Implementation、§2.1 UI、§2.2 Pipeline、§3 Conclusion、致谢与参考文献均已亲见摘录）；DOI: 10.1145/3306307.3328170。图 1–3 未获取原图（PDF 不可达）。

## Learning Value

学术文献几乎不回答"灯阵怎么在管线里流动、怎么进美术手里"——这篇（连同 LightCraft、Shlyaev）构成 ⑥ 槽的工业证据链：**检测半自动化 + USD 资产化 + 保留人工末端**是过去十年影视侧收敛出的形态。

## Visualization

论文原图未获取（PDF 被 403 拦截）；内容以流程描述为主，MD 表格即可表达，无需 HTML 图示。

## Notes

- 引用格式（原文）：Cieri and Schwank. 2019. In *Proceedings of SIGGRAPH '19 Talks*. ACM, 2 pages. DOI: 10.1145/3306307.3328170。
- ACM 页面显示有补充材料（18MB ZIP + 143MB MP4 演示视频），未获取。
- DNEG 官网 PDF 链接当前对本机直连返回 403；如需原图可日后换网络环境重试。
