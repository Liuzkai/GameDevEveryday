---
type: paper
title: "Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields"
authors: [Zander Majercik, Jean-Philippe Guertin, Derek Nowrouzezahrai, Morgan McGuire]
year: 2019
published: "2019-06-05（JCGT 8(2), pp. 1–30；2018-12-22 投稿 / 2019-03-11 接收；I3D 2019 演讲；GTC'19 / GDC'19 首次公开）"
venue: "JCGT — Journal of Computer Graphics Techniques, Vol. 8, No. 2, pp. 1–30, 2019（JCGT 无 DOI；ISSN 2331-7418）"
url: "https://jcgt.org/published/0008/02/01/"
code: "G3D Innovation Engine（casual-effects.com/g3d）内置；官方 supplement 含参考实现与 GLSL 代码"
project_page: "https://jcgt.org/published/0008/02/01/"
category: [global-illumination, irradiance-probes, ray-tracing, real-time-rendering, caching, classical]
importance: A（经典）
historical_importance: 5
game_relevance: 5
production_readiness: "Industry Adopted（NVIDIA RTXGI SDK 产品化：2020-03 v1.0 → 首个游戏《逆水寒》2020-11 → ICARUS / THE FINALS / ARC Raiders 等；UE4 插件 + Unity + 多家自研引擎；2024 起 SDK v2.0 引入 NRC / SHaRC）"
user_level: "Normal（结论层）"
status: unread
aliases: [DDGI, Dynamic Diffuse GI, 动态漫反射全局光照, 辐照度场, irradiance field, ray-traced irradiance fields, Majercik 2019, RTXGI 论文]
tags: [classic, gi, probes, irradiance-field, ray-tracing, real-time, jcgt]
---

# Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields（Majercik et al. 2019）

## TL;DR

**把"预烘焙的静态光照探针"改造成"每帧用硬件光追增量更新的辐照度场"——第一个真正可出货的动态漫反射 GI 探针方案。** 三件套：① **紧凑编码**（每个探针 = 8×8 八面体辐照度 + 16×16 距离/距离平方，打包进图集）；② **摊销更新**（每帧只发 m 探针 × n 条射线，着色后按滞后系数 α∈[0.85, 0.98] 混入探针，多弹射跨帧累积）；③ **可见性加权的查询插值**（8 探针 cage + 背面软剔除 + 感知抑制 + **Chebyshev 矩可见性** + 法线偏置——五级权重专治探针法的老毛病"漏光"）。1080p 全 GI **6 ms/frame**（2080 Ti 首发实测），对照离线路径追踪 1 min/frame。

> **一句话定位**：库内 [[Global Illumination]] 谱系表**第 6 行（缓存族）与第 7 行（光追 GI）的合流点**——[[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)]] 的"缓存答案"、[[Keller — Instant Radiosity (1997)]] / [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]] 的"光侧记账"，在硬件光追时代被重新合成成"场"。**它也是 [[Neural Global Illumination]] 桥的"现代侧"锚点**：NRC（2021）要替换的"缓存介质"，在产品线上就是从 DDGI 的插值公式开始的（RTXGI SDK v2.0 直接并入了 NRC）。

## Problem（2019 年的处境）

- **探针法三个老毛病**：手动放置 + 代理几何（反射错位）、**漏光/暗漏**（light/dark leaking）、**静态**（预计算，只在初始化时烘一次）；IBL 探针体系在游戏里无处不在，但"免手动、动态、无泄漏"三者从未同时成立；
- **屏幕空间方案（SSRT / SSGI）**：屏幕外几何一律丢失，且视角相关 → 时间不稳定；
- **体素锥追踪（VXGI 类）**：几何表示（八叉树）与参数化耦合 → 泄漏难治；厘米级几何与米级体素是不可兼得的成本；
- **离线路径追踪**：质量参照系，但离实时有 3-4 个数量级；
- 目标：**动态物体 + 动态光照**下的实时漫反射 GI，**无需手动放置**，且对几何与辐射复杂度都鲁棒。

## 前史与继承（Historical Context）

```text
探针传统（greeting-card irr. volumes → IBL 预计算）
  ↓ SH 表示（9 阶，[[Ramamoorthi-Hanrahan — An Efficient Representation for Irradiance Environment Maps (2001)]]，2001）
  ↓ CloudLight（2015，把"摊销"引入实时间接光：跨帧累积）
  ↓ 光场探针（McGuire et al. 2017，I3D）：探针 + 径向距离/距离平方编码 → 可见性加权查询，
    但只支持【静态】几何与光照
  ↓ ★ 本文（2019）：把上面全部动态化 —— 运行时增量更新 + 滞后混合 + 矩可见性
```

- 与 **[[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)|RSM]]** 的同构：都是"把间接光信息存进一个小本子"；RSM 存"光源视野像素"，DDGI 存"空间网格上的方向分布"——**DDGI 是 RSM 的探针版重写**；
- 与 **[[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)|辐照度缓存]]** 的同构：缓存对象都是"辐照度"、都要处理"哪里该更新"；Ward 用八叉树 + 误差容限，DDGI 用规则网格 + 时间滞后；**更新对象的介质从"插值"一路演化到 NRC 的"神经网络"**；
- 原文自述的定位：**"把光追当作光栅化的补充，而不是替代"**——只在非相干可见性查询（世界空间射线）上动用 RT 核，其余（直接光、GBuffer、合成）保持光栅化管线。这句话是本文的方法论核心，也是本库"分工判据"的又一源头。

## Core Idea（机制）

### ① 编码：一个探针 = 两张八面体小图

| 数据 | 格式 | 分辨率 | 用途 |
|---|---|---|---|
| 球面辐照度 | `GL_R11G11B10F` | **8 × 8** 八面体 | 漫反射间接光查询 |
| 平均距离 + 距离平方 | `GL_RG16F` | **16 × 16** 八面体 | 可见性（Chebyshev 矩）|

- 全部探针打包进**一张 2D 图集**（`gutter` 边 + 4×4 写边界对齐——保证双线性插值不出缝、GPU 写入对齐）；
- 深度值经 **cosine-power lobe 加权后锐化（depth sharpening）**；低于阈值（0.001）的 texel 不更新（省成本）；
- 相比前作（128×128×6 高精度 cube map 存深度），因为靠加权方案兜底数值鲁棒性，**降到 16×16 也不出问题**；
- 格式结论（原文 Figure 12）：**R11G11B10F 在"16 位浮点保真度"与"省 45% 存储"之间取得平衡**；8 位整型会出伪影、RGB10A2 是"质量-尺寸"备选。

### ② 更新：每帧一轮"探针-射线批"，滞后混入

```text
每帧循环（参考实现：所有探针都更新——保守上界）：
  1) 从 m 个探针各发 n 条射线（随机旋转 Fibonacci 螺旋方向，线程一致地批处理）
     → 命中点存 surfel（G-buffer 样式：位置 + 法线）
  2) 用【与最终画面同一套 shading 例程】给 surfel 着色
     （直接光 + 上一帧探针数据算出的间接光）
  3) 按 α（滞后参数）把新结果混入探针 texel：
     newIrr = lerp(oldIrr, Σ(cosine 加权射线辐射度), hysteresis)，α ∈ [0.85, 0.98]
  多弹射：本帧的"间接"= 上一弹射的探针结果 → 跨帧累积收敛（摊销）
```

- **为什么需要滞后**：单帧射线数远不足以收敛（噪声），滞后 = 时间滤波 = 在"响应速度"与"噪声"之间调档；原文承认副作用——间接光会在剧烈可见性变化处"流动"进出（延迟），但动态场景下不可察觉，且被先前工作证明为可接受的感知折中；
- **为什么能省**：间接光是**低频、视图无关**的量，变化远慢于直接光——**"贵的东西用便宜的时间常数摊销"**。

### ③ 查询：8 探针 cage + 五级权重（治漏光的核心）

对每个着色点，取包围它的 8 个探针（cage），按顺序加权：

1. **软背面剔除**——法线与"指向探针的方向"点积接近 0 时平滑淡出（截断下方探针）；
2. **感知抑制**——对"极暗区里的极低辐照度"（< 5% 强度范围）按单调衰减曲线压低贡献（人对暗区漏光最敏感——原文专门做了 HVS 依据的权重）；
3. **Chebyshev 矩可见性**——用平均距离与距离平方算**Chebyshev 上界**（方差阴影贴图同源的矩不等式），把"探针看不到的着色点"的权重压下去；
4. **法线偏置**——按法线与探针方向做偏置偏移，避开"阴影-非阴影"边界处的权重失真；
5. **三线性插值**——最后按着色点与探针中心距离做标准三线性混合。

- **成立前提（原文明确写出）**：*"着色点处的入射光 ≈ 包围它的探针处的入射光——当且仅当二者互相可见"*。探针越稀，这个假设的误差越大（Figure 9 的漏光来源）；
- 消融链（Figure 7）：传统探针 → +背面剔除 → +可见性 → +法线偏置 → 逼近路径追踪参照——**每级权重对应消除一类伪影**。

## Key Numbers（原文核对）

| 项 | 数字 | 备注 |
|---|---|---|
| 总帧时间 | **6 ms/frame**（1080p，RTX 2080 Ti）| 对照同场景离线路径追踪 1 min/frame（Figure 1）|
| 间接光组件合计 | **2.6 ms**（32×8×32 探针、64 rays/probe）| 分解：ray cast 0.8 + ray shade 0.4 + 探针更新 0.7 + 查询 0.5 + ray gen 0.1 + 延迟直接光 0.1（Table 2）|
| 光追吞吐 | **> 1.5 GRays/s** 起（多配置）| 希腊神庙场景 876,127 primitives |
| 探针密度建议 | 人类尺度场景 **1–2 m 间距** | 网格 2 的幂；每房间至少一个完整 cage |
| 质量结论 | **"探针数量比探针分辨率更重要"** | 低密度 → 漏光；低分辨率 → 基本无感（Figure 9/10/11）|
| 格式结论 | R11G11B10F ≈ 16-bit 保真度、存储 -45% | 深度 16×16 替代前作 128×128×6（Table 2 / Fig 12）|
| 网格鲁棒性 | 对探针网格的旋转/平移鲁棒 | 失效例：着色区域**完全落在**探针覆盖之外（Figure 14）|

## Why It Works（为什么成立——四个前提）

1. **低频 + 视图无关**：间接光的空间/方向变化远比直接光平滑 → 探针之间可插值、可时间滤波；
2. **互可见假设**：把"可见性"用矩（均值/平方距离）便宜地估计出来——Chebyshev 上界是"宁多勿漏"的保守估计，恰好匹配漏光的感知方向；
3. **摊销**：把"收敛"从空间维度（多发射线）挪到时间维度（多帧累积）——**帧预算有限时，时间是第二根预算轴**；
4. **RT 只做它擅长的事**：光栅化做相干部分，光追只补非相干的世界空间可见性——**混合渲染的分工范式**。

## Limitations（原文承认 + 后续补齐）

- **保守更新**：参考实现每帧更新所有探针（大场景大量浪费）——自适应选择/探针流式/多级网格归入未来工作（**后续在 "Scaling for Production" 2021 中补齐**：probe state machine、级联卷、self-shadow bias）；
- **时滞伪影**：间接光在剧烈变化处"流动"（静态场景静止视角下最显眼——非主要用例）；
- **假设的边界**：探针密度↓ → 误差↑；着色区域出网格 → 失效；
- **内存随体积增长**：大世界需要流式/级联（2021 论文的 Infinite Scrolling Volumes 即此问题的产品答案）。

## Game Development Relevance

- **产品线（RTXGI）**：2020-03 SDK v1.0 → **2020-11《逆水寒》成为首个实装游戏**（来源：百度百科/NVIDIA 宣传，二手）→ 2021-12 **ICARUS 首个开放世界"无限滚动体积"** → 2023 **THE FINALS** → 2026 报道 **ARC Raiders / 《侏罗纪世界：进化 3》** 采用（ARC Raiders 为"以 RTXGI 替代 Lumen"样本，二手待核）；RTXGI SDK **v2.0（2024）并入 NRC 与 SHaRC**——**同一条产品线里，"经典探针"与"神经缓存"并置**；
- **对分档工作（你的五维之外的第六维候选）**：DDGI 的档位旋钮是**三个正交量**——探针数（网格密度）× 探针分辨率 × **每帧更新射线数/更新率**；且"数量 > 分辨率"给出了**调参次序**的一条实证；
- **"预算语言"新节点**：每帧 m 探针 × n 射线 = **"固定样本预算"家族的 2019 形态**（1978 每灯 +1× → 1997 每 VPL 一遍 → 2005 每像素 ~400 → **2019 每帧 m×n 射线** → 2026 MegaLights 每像素采样预算）——且它多出一根轴：**跨帧摊销**（把"收敛时间"本身当成可调预算）；
- **与 Lumen/MegaLights 对照**：同代两条 GI 路线——探针场（DDGI）vs 表面缓存 + 软件/硬件光追（Lumen）；DDGI 的存活证明了**"探针族"在 2026 仍是活跃选项**（ARC Raiders 样本）。

## Unreal Engine Relevance

- **UE4**：NVIDIA NvRTX 分支提供 RTXGI 插件（小团队接入案例：Escape from Naraka——"放一个 DDGI 体积就能立刻看到差别"）；
- **UE5**：Lumen 是引擎默认的另一路线（表面缓存/SDF/光追混合）；DDGI 不进入 UE5 主路径，但**"探针 + 摊销更新 + 可见性加权"的思想**仍值得作为 Lumen 的对照物理解；
- 对 NGR 的直接意义：**UE5 项目不必实装 DDGI**；本篇的价值在于 ① 补齐 GI 谱系（[[Neural Global Illumination]] 桥的现代侧）；② 提供"档位化 GI"的第二种语言（更新率/探针预算）；③ "互补而非替代"的混合渲染判据实例。

## Technology Evolution（谱系位置）

```text
1988  Ward：接收侧缓存（答案视图无关 + 误差容限 a）      ★ 已入库
1997  Keller：Instant Radiosity（VPL = 光路顶点变灯）    ★ 已入库
2005  Dachsbacher：RSM（每像素固定样本 gather）           ★ 已入库
2001  Ramamoorthi：SH 环境光（低频解析表示）              ★ 已入库
2015  CloudLight：摊销间接光（跨帧累积的先行者）
2017  McGuire：光场探针（探针 + 距离编码，静态）
  ↓
★ 2019  DDGI（本篇）：动态化三件套 = 增量更新 + 滞后 + 矩可见性
  ↓
2020  RTXGI SDK v1.0 → 《逆水寒》首个实装
2021  Scaling for Production（probe state machine / 级联卷 / 无限滚动）
2021  DDGI Resampling（与 ReSTIR 结合，Majercik/Müller/Keller 等）
2021  NRC（把同一"辐照度场"的介质从插值换成神经网络 → [[Neural Global Illumination]]）
2024  RTXGI SDK v2.0：NRC + SHaRC 并入产品 —— 经典与神经在同一 SDK 会合
2026  ARC Raiders / JWE3 报道采用（探针族存活样本，二手）
```

## Relationships

### Based On
- 光场探针（McGuire et al. 2017）+ 辐照度探针传统（IBL / greeting-card）——本篇是它们在动态方向的彻底外推

### Extends
- [[Ward — A Ray Tracing Solution for Diffuse Interreflection (1988)]]（缓存对象 = 辐照度；更新策略从误差容限扩展为时间滞后 + 规则网格）
- [[Dachsbacher-Stamminger — Reflective Shadow Maps (2005)]]（"固定样本预算"的探针版：从"每像素 N 样本"变为"每帧 m×n 射线"）

### Related
- [[Keller — Instant Radiosity (1997)]]——同为"光侧"记账的可对照方案（VPL 求和不变量不同）
- [[Kajiya — The Rendering Equation (1986)]]——GI 问题的容器：本篇用探针场近似它的漫反射分量
- [[Shadow Mapping]]——Chebyshev 矩可见性与 VSM 同源（原文直引 Donnelly & Lauritzen）

### Followed By
- RTXGI SDK（产品化）· Scaling for Production（JCGT 10(2), 2021）· DDGI Resampling（CGF 2021）· **[[Neural Global Illumination]]**（NRC 2021：同一目标的神经网络介质）

## Personal Knowledge State

- `user_level: Normal（结论层）`——你的 GI 线已读 [[Kajiya — The Rendering Equation (1986)]] 与 [[2026-09-14-Gaussian Light Transport]]；**缓存族（Ward/IR/RSM）与本篇均"材料已入库、阅读动作待执行"**（与 2026-10-05 校正后的 PKM 一致）；
- **30 分钟读法**：Figure 1（效果/成本对比）→ Figure 3（编码与图集）→ Figure 6/7（查询权重与消融链）→ Table 2（成本分解）——机制层即可拿下；
- **本笔记的"跨域借条"**：① "预算加在数量上还是分辨率上"（≈ 你的"档位调参次序"）；② "把时间当第二根预算轴"（摊销）；③ "互补而非替代"。

## Learning Value

- **对 [[Neural Global Illumination]] 桥**：经典侧最后一块"现代形态"就位——桥材料自此为 **Ward 1988 + IR 1997 + RSM 2005 + DDGI 2019 + 双图解 + 本篇图解**；
- **对分档/预算工作**：DDGI 是"档位 = 多个正交子系统的组合"（9-21 判断）在 GI 域的又一实证——探针数 × 分辨率 × 更新率；
- **判据入库**：**"收敛可以从空间维度挪到时间维度"**（预算的第二根轴）。

## Visualization

![[探针辐照度场_DDGI 2019 图解.html]]

## Notes

- **来源与核对**：JCGT 官方低分辨率版 PDF（1.2 MB / 30 页）**逐页核对**；官方引用页确认 vol. 8, no. 2, **pp. 1–30**（并核对了 2018-12-22 投稿 / 2019-03-11 接收 / 2019-06-05 发布）；I3D 2019 演讲幻灯片与 supplement（589 MB）在官方页存目。**注意：不是 pp. 50–68**（此前多处二手来源记错，以官方引用页为准）；
- 产品线时间线中：逆水寒实装、RTXGI v2.x 版本号、ARC Raiders/JWE3 采用——均为**二手来源**（百度百科/媒体/NVIDIA 博客），已在正文标注；Scaling 2021 的作者与摘要经 NVIDIA Research 官方页核对；
- 与 [[2026-09-14-Gaussian Light Transport]] 的对照：同为"把 GI 解存成场的参数化"，一条是显式高斯基函数 + 残差优化（离线/毫秒级），一条是探针网格 + 光追更新（实时/毫秒级，含动态）——**"场"的两种存法与两种时间尺度**。
