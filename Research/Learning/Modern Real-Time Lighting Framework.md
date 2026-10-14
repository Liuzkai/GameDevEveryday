---
type: explainer
title: "现代实时光照的共同设计框架：MegaLights、Lumen、ReSTIR 与 RTXDI"
parent: "[[Real-Time Global Illumination]]"
created: 2026-10-13
verified: 2026-10-13
tags: [rendering, lighting, sampling, temporal-reuse, unreal-engine]
---

# 现代实时光照的共同设计框架

> **先分配计算预算，再复用有效信息，最后重建图像。场景变化时，明确各层数据的更新条件和弃用条件。** 本文用这套框架比较技术，不表示 MegaLights、Lumen 和 ReSTIR 使用同一种算法。

本文面向技术美术。从一帧画面的光照问题出发，依次解释采样概率、历史可信度、缓存索引和公开代码。阅读后，应能分析噪声、拖影和离屏物体投影等问题，并找到相关数据与成本。

建议先读第 1–4 节，了解任务、数据和索引空间；再读第 5–8 节，比较具体实现；最后用第 9–12 节排查问题和练习。第 13 节列出版本、固定代码提交和核验限制。

本文使用以下标记：

- `[论文]`：研究方法。
- `[产品]`：指定版本的官方文档或作者讲稿。
- `[代码]`：原核验记录中实际阅读的公开实现。
- `[教学]`：为解释原理而抽象的流程、公式或排查建议。
- `[未确认]`：证据不足。

没有单独标记的过渡说明沿用所在段落的标记，不能据此认定引擎源码采用了该实现。

## 1. 区分各项技术的任务

以雨夜街道为例：路灯照亮墙面，属于**直接光照（direct lighting）**；墙面把光反射到屋檐下，属于**间接光照／全局光照（indirect lighting / GI）**；要在水面或光滑车漆中显示街道，还需要计算反射方向上的光。这些现象都属于光传输（light transport）。实时引擎根据计算预算、材质和稳定性要求，将它们拆成不同任务。

| 名称 | 主要任务 | 比较重点 |
|---|---|---|
| **MegaLights** | UE 的随机直接光照路径。选择少量光样本，测试遮挡并着色，然后重建图像 | 直接光预算、选灯、可见性和去噪。MegaLights 不是 GI 系统的别名 |
| **Lumen** | UE 的动态 GI 与反射系统。组合用于追踪的场景表示、Surface Cache、光照收集（gather）和多层复用 | 间接光的来源、命中点着色和各层更新速度 |
| **ReSTIR** | 从直接光发展而来的储备池重采样（reservoir resampling）方法族 | 复用的样本、样本转移到当前接收点的方法，以及转移后的权重是否有效 |
| **RTXDI** | NVIDIA 公开可读的 SDK 与示例，用于将重采样方法接入渲染器 | 具体工程实现。本文固定的 3.1.0 包含 ReSTIR DI、ReGIR、GI、PT |

依据：[MegaLights 5.6 文档][M56]、[Lumen 作者博客][LB]、[ReSTIR 原始项目页][R20]、[RTXDI Integration][CI]。算法、引擎系统和 SDK 属于不同层级，比较时需要先明确任务与范围。

### 共同框架：采样 → 时空复用 → 重建／去噪 → 更新／失效

`[教学]` 分析一项方案时，依次回答五个问题：

1. **计算什么？** 直接光、间接入射光、反射路径，还是体积散射？
2. **在哪里计算？** 屏幕像素、屏幕探针（screen probe）、世界空间探针、表面 Card，还是视锥体素（froxel）？
3. **保存什么？** 几何、材质、辐射亮度、候选样本及统计量、选灯信息，还是图像历史？
4. **何时可以复用？** 如何分别检查表面、方向、灯 ID、遮挡和材质是否满足复用条件？
5. **失效后如何处理？** 重新计算、回退到新样本、缩短历史、降低置信度，还是按预算分帧更新？

这四个阶段表示逻辑关系，不规定 GPU 各处理阶段（pass）的执行顺序。上一帧的信息可以在本帧采样前指导采样分布。每次读取历史都需要检查有效性，缓存维护也可能独立调度。因此，更新与失效检查可能发生在多个阶段，而非仅在帧末执行一次。

| 技术 | 采样 | 时空复用 | 重建／去噪 | 更新／失效 |
|---|---|---|---|---|
| **MegaLights** | 直接光样本 | 用历史可见灯等信息引导选灯；不能据此认定使用了储备池 | 从随机直接光估计重建图像 | 通过探索发现新可见灯；分别维护历史与场景表示 |
| **Lumen** | 间接光／反射方向与命中查询 | 复用表面照明、屏幕探针和世界空间探针等多层数据 | 光照收集中的重建，以及反射过滤等 | 覆盖范围、照明更新预算和屏幕历史各有约束 |
| **ReSTIR 方法族** | 候选光样本／路径样本 | 复用储备池统计量并重采样样本 | 输出仍可能有噪声，可接入重建器 | 重新评价目标，处理样本来源域和历史有效性 |
| **RTXDI 3.1.0** | 应用连接场景接口与采样模块 | 提供 DI / GI / PT 函数和可选的 ReGIR | 示例展示重建输入与置信度的集成 | 函数提供匹配、索引、历史和偏差校正策略 |

上表概括后文的比较内容。不同技术即使都使用“复用”，复用的数据也可能不同。

## 2. 不透明表面的直接光照流程

![不透明表面直接光的逻辑数据流；蓝线是当前帧，虚线是跨帧反馈](../Files/Pictures/ModernLighting-OpaqueFlow.svg)

**读图**：实线表示本帧的数据依赖，虚线表示经过有效性检查的跨帧反馈。选灯引导决定采样对象，去噪历史用于重建图像，两者不能互换。③ 中的储备池复用属于另一种采样分支，需要读取自己的样本历史；图中未展开全部输入。本图只表示共同的逻辑关系，不规定任何 UE 版本的调度数量、资源布局或执行顺序。

### 2.1 从 G-buffer 获取接收表面的信息

`[教学]` 光栅化确定当前相机可见的不透明表面，G-buffer 保存相关表面属性。深度与相机矩阵用于重建位置；法线、粗糙度（roughness）和材质参数用于计算表面反光。运动向量（motion vectors）用于建立跨帧对应关系，本身不保存照明结果。

不同引擎会压缩、拆分或重建这些属性。理解逻辑上需要哪些数据，不能直接确定数据位于哪张纹理的哪个通道。

普通不透明 G-buffer 通常只描述每个像素最前面的不透明表面，不会自动记录后方的每层透明表面。因此，透明物体需要其他光照路径。

### 2.2 剔除无贡献的灯，再按重要度采样

`[教学]` 先根据几何范围，剔除不可能产生贡献的灯。再用重要性采样（importance sampling），优先抽取可能贡献较大的候选。材质、方向、距离、灯的形状和亮度，以及历史可见性，都可能影响重要度估计。不同方法使用的估计依据不同。

贡献可能很小，不等于贡献必定为零。粗略的选灯引导不应永久排除可能重新可见的光源。MegaLights 官方文档说明，系统需要周期性探索历史上被遮挡的灯。[M56]

### 2.3 测试接收点与光样本之间的可见性

可见性测试（visibility test）检查接收点与光样本之间是否存在几何遮挡。选中一盏灯后，仍需要判断这次连接是否被墙等物体挡住。对面光源（area light），还需要选择发光面上的位置。同一盏面光源上不同位置的样本，可能得到不同的软阴影结果。

`[产品：5.6]` MegaLights 先用短距离屏幕空间追踪（screen-space trace）补充细节。当屏幕数据无法继续提供可靠判断时，转到光线追踪场景（Ray Tracing Scene）。选择虚拟阴影贴图（VSM）阴影方式的灯使用另一种可见性来源。[M56]

屏幕深度只描述当前视图。包围体层次结构（BVH）可以包含离屏物体，以及被前景遮住的物体。因此，遮挡者即使没有出现在 G-buffer 中，也可能参与遮挡测试。但 BVH 仍受范围、剔除和几何简化限制，不能假定它包含全部精确几何。[M56]

### 2.4 用概率权重补偿抽样

`[教学：仅限离散点灯例子]` 固定一个接收表面。设第 $i$ 盏灯经过材质和当前遮挡计算后的真实贡献为 $c_i$，则全部直接光为：

$$L=\sum_{i=1}^{K} c_i.$$

按离散概率 $p_i$ 独立、有放回地抽取 $N$ 次。若所有非零贡献对应的灯都满足 $p_i>0$，普通蒙特卡洛（Monte Carlo）估计为：

$$\widehat L=\frac{1}{N}\sum_{j=1}^{N}\frac{c_{I_j}}{p_{I_j}},\qquad I_j\sim p.$$

其中，$K$ 是候选灯数，$N$ 是采样次数，$p_i$ 是归一化的离散概率。例：两盏点灯的贡献分别为 9 和 1，采样概率分别为 0.9 和 0.1。在这个理想化例子中，任一次采样的加权贡献都等于 10。因此，优先选择亮灯，同时仍可估计全部灯的贡献。

本节仅用上式解释离散点灯采样。它不是 MegaLights 的概率密度函数（PDF）公式，也不能直接用于 ReSTIR 的重采样输出。面光源涉及位置或方向的连续概率密度，以及测度变换。ReSTIR 还需要处理储备池的统计权重和样本来源域。

采样分布、权重、可见性和目标积分必须匹配。只累加抽中样本的原始贡献，不能补偿抽样；给遗漏的灯任意添加环境光，也不能替代概率补偿。任意历史权重不一定等于精确的 $1/p$。

### 2.5 从稀疏观测重建图像

`[教学]` 采样产生带噪声的估计。重建可以利用相邻像素和历史信息，但需要避免混入其他表面的亮度。常见方法包括：分开处理漫反射／镜面反射（diffuse / specular）信号，减少材质颜色对滤波的干扰，用深度和法线等信息阻止跨边界混合，并根据变化速度调整历史信息的使用。

上述解释对应 [SVGF 研究][SVGF] 的时空重建思想，以及 [RTXDI 示例的 ShadingHelpers / ShadeSamples][CS] 和 [Confidence 文档][CC] 中的信号准备与历史反馈。这些来源不证明 UE 使用了 NVIDIA 去噪器，也不证明 GI、透明和体积光照可以共用同一套过滤器。

## 3. 区分缓存的数据和有效条件

分析持久数据时，先明确保存内容、索引方式、使用者和有效条件。下表按数据含义分类；同一系统可能使用相关资源保存多类信息。

| 类别 | 保存内容 | 索引方式 | 更新或弃用的条件 | 需要区分的数据 |
|---|---|---|---|---|
| **几何表示**：BVH、Mesh / Global Distance Field | 用于求交或距离查询的场景表示。BVH 是加速结构，不保存光照结果 | 世界空间射线、实例／几何层级，或距离场中的空间位置 | 变换、形变、细节层级（LOD）、对象增删、剔除或流式加载发生变化时维护 | 灯光缓存、屏幕颜色 |
| **材质表示**：G-buffer／Surface Cache 的材质部分 | 当前可见表面属性，或 Card 捕获的表面属性 | G-buffer 按屏幕像素索引；Surface Cache 将命中表面映射到 Card／纹理图集（atlas） | 表面或捕获变化、分辨率不足或覆盖不足时，重建或重新捕获 | 已积分的 GI |
| **光照缓存**：Surface Cache 的照明、Radiance Cache | 已计算的表面光照，或带方向的入射辐射亮度（radiance）等结果 | 表面参数化，或探针的位置与方向 | 光源、遮挡、材质或覆盖范围变化时更新；更新受预算约束 | 仅保存灯 ID 的列表 |
| **样本储备池**：ReSTIR DI | 保留的光样本和重采样所需统计量；实现中还可保存用于复用的可见性信息 | 2020 DI 和本文 RTXDI 路径主要按屏幕像素索引，并跨帧重投影 | 表面不匹配、灯 ID 消失或历史过长时处理；复用时重新评价目标权重 | 最终像素的平均颜色、完整的灯光概率表 |
| **选灯引导**：MegaLights 的屏幕分块引导（screen tile guiding） | 历史可见灯等信息，用于分配采样预算 | 2025 讲稿中的屏幕分块和重投影位置 | 根据新帧可见性反馈、探索结果或对应关系失效更新；精确规则取决于实现 | ReSTIR 的逐像素储备池、最终可见性判定 |
| **去噪历史** | 过去的光照估计，以及方差、历史长度、置信度等辅助量；字段取决于算法 | 按二维像素或重建域索引，通过运动向量重投影 | 新显露区域（disocclusion）、相机切换、表面或光照变化时，拒绝历史、限制数值范围或降低权重 | 可重新着色的光样本 |

产品部分依据 [Lumen Technical Details][LT] 和 [Lumen 作者博客][LB]。储备池与历史维护的代码证据见第 7 节。BVH、Surface Cache、储备池和去噪历史分别维护，各自的有效期可能不同。

### 关灯后，各层数据分别响应

`[教学]` 关闭一盏灯，并保持几何不变，各层可能出现不同的响应：

- BVH 未必需要重建。
- 灯数据和选灯引导需要响应灯的变化。
- 储备池若引用了该灯，需要重新评价贡献，或处理灯被删除的情况。
- Surface Cache 中的照明可能仍在等待更新。
- 去噪历史可能仍保留旧亮斑。

画面逐渐变暗，可能由光照缓存更新延迟、去噪历史滞后，或两者叠加造成。不能统一归因于时间抗锯齿（TAA）。

## 4. 区分数据的索引空间

![四种索引域对照：像素历史、屏幕 tile、视锥 froxel、表面 Card atlas](../Files/Pictures/ModernLighting-IndexSpaces.svg)

**读图**：$u,v$ 表示屏幕方向，$z$ 表示视空间深度。本图不规定引擎的轴朝向、正负号或深度切片公式。Card 的 $s,t$ 表示表面捕获参数，不能当作屏幕 UV。图中网格数量只用于示意，不代表实际分辨率。

| 表示 | 用途 | 适用边界 |
|---|---|---|
| **二维历史（2D history）** | 用运动向量寻找上一帧与当前像素对应的位置，再检查历史是否有效 | 相同屏幕坐标不一定对应同一物体。历史纹理为二维，不代表光线只能在屏幕空间传播 |
| **屏幕分块引导（screen tile guiding）** | 在一小块屏幕内共享选灯信息，降低维护和查询成本 | 分块跨越前后景时，只能提供粗略引导。它不能替代当前样本的遮挡测试，也不同于移动 GPU 的分块存储（tile memory） |
| **视锥体素（3D froxel）** | 按屏幕方向和深度切分视锥，在空间位置上计算或共享光照与阴影，供雾或透明路径使用 | froxel 即 frustum voxel，通常随视图组织。它不是自动覆盖世界的均匀体素，也不是精确的透明表面列表 |
| **Surface Cache** | 降低射线命中表面后查询材质和缓存光照的成本 | 二维纹理图集索引的是场景表面，不是上一帧屏幕截图，也不是均匀的三维辐射亮度体积 |

还需要区分两种世界空间结构：Lumen 的 Radiance Cache 探针保存光照；RTXDI 的 ReGIR 网格为某个位置附近的灯提供采样候选。二者都按三维空间组织，但保存内容和后续操作不同。[LB][CI]

## 5. MegaLights：按样本分配直接光照预算

### 5.1 同时限制着色与阴影计算的采样成本

`[产品]` 传统逐灯评估中，重叠灯越多，光照计算与阴影成本通常越高。MegaLights 通过随机采样限制昂贵评估的数量，再结合选灯引导和图像重建生成画面。它同时改变着色与阴影计算，不能仅理解为给延迟光照（Deferred Lighting）更换阴影贴图。[M56]

`[产品：SIGGRAPH 2025]` 作者讲稿描述了上一帧中每个 8×8 屏幕分块的可见灯列表（visible-light list）。系统通过重投影和随机双线性查询（stochastic bilinear lookup），读取这些信息来指导采样。列表记录值得再次测试的灯，不保存照明颜色，也不是本文 RTXDI 实现中的储备池。

原核验记录通过在线索引确认了讲稿中的分块与深度不连续问题。重投影和随机查询等细节还结合了当次独立只读研究的补充记录；完整访问范围见第 13 节。[M25]

`[教学]` 建议为被遮挡的灯保留探索机会。例如，门打开后，系统必须能纠正原先的不可见判断。探索让缺少证据或曾被遮挡的候选仍有机会接受测试，并不要求忽略重要度、平均分配全部预算。

当前重要度与历史引导如何混合，取决于具体算法。本文没有确认 UE 的精确混合公式、阈值或缓冲区打包方式。

### 5.2 区分样本预算、灯数和整帧成本

`[教学]` 一帧中的 $N$ 次采样可能多次选中同一盏面光源，也可能选中不同的灯。场景灯数可以远大于 $N$。限制每像素的样本数，不等于永久限制该像素只能接收 $N$ 盏灯的贡献。

分析预算时，可以按以下项目拆分工作量：

```text
光照工作量 ≈ 场景表示维护 + 候选整理与重要度估计
           + 样本数 × 每条追踪／着色的代价
           + 时空复用与缓存读写 + 重建／去噪
           + 所选额外路径（例如 VSM、透明 front layer）
```

上式只拆分成本，不用于直接相加 GPU 性能分析器中可能重叠的异步处理阶段。分辨率、BVH 复杂度、动态几何、带宽和额外路径都会影响最终帧时间。2026-10-13 读取时标为 5.8 的官方文档，还明确列出了重叠灯的重要度估计成本。[M58]

`[教学]` 技术美术可以据此关注两点：

- **合理设置光源影响范围（bounds），可降低成本并改善质量。** 无必要的大范围灯，会让更多位置把采样预算用于最终没有贡献的候选。
- **结合光源分布和场景细节判断采样难度。** 灯数相同时，强小光源密集重叠、细碎遮挡和高频高光，可能比空间分散的大灯更难采样。增加样本可以减小方差，但不能补回缺失的遮挡几何。

### 5.3 在项目中确认实际配置

5.6 文档在提高质量的说明中使用 `r.MegaLights.NumSamplesPerPixel 16`；5.7 也将它列为质量扩展示例。这些示例不能证明默认样本数为 16。实际采样分辨率、质量档位（quality tier）和控制台变量（cvars），应以项目分支的配置及运行时查询为准。原核验未读取 Epic 授权 UE 源码，因此本文未确认跨版本的默认每像素光线数。[M56][M57]

## 6. Lumen：用多种表示处理间接光查询

![Lumen 的表示、命中照明、gather 与反射关系图](../Files/Pictures/ModernLighting-LumenFlow.svg)

**读图**：左侧确定射线命中位置，中间查询命中处照明，右侧处理主视图的光照接收与重建。Surface Cache 与 BVH 分工不同，世界空间 Radiance Cache 与 Surface Cache 也保存不同的数据。图中可选的 Hit Lighting 仅用于反射分支，不能据此认定 Screen Probe Gather 也启用了该路径。图中只显示主要逻辑依赖，省略具体追踪后端和质量分支。

### 6.1 分别检查 Surface Cache 的覆盖、材质和照明

`[产品]` Lumen 为网格建立 Cards，并将多个视角捕获的表面数据存入 Surface Cache。射线命中表面后，可以查询缓存照明，减少重复执行复杂材质和光照计算的成本。Card 的组织与覆盖、材质捕获、缓存照明更新，需要分别检查。其中一项有效，不能保证其余项也有效。[LT]

作者在 *Journey to Lumen* 中回顾了 Cards 的用途变化：早期探索曾用 Cards 进行几何追踪，后来改为缓存表面计算。因此，早期实验中的高度场／锥体追踪（heightfield / cone tracing）管线，不能代表当前 Lumen 的完整机制。[LJ]

### 6.2 区分屏幕探针收集与世界空间光照复用

`[产品／作者解释]` 对漫反射 GI，屏幕探针收集（Screen Probe Gather）在屏幕上的稀疏位置收集带方向的入射光，再结合全分辨率材质重建。这样，多个像素可以共享昂贵的追踪结果。历史入射光还可指导本帧的方向选择。[LB]

较远的光照通过世界空间 Radiance Cache 复用，有助于处理小窗照亮室内等情况。粗糙表面的反射可以利用相关收集信息；更光滑的反射通常需要额外追踪。[LB]

`[教学]` 屏幕采样位置只确定射线从主视图的何处出发，后续射线仍可转到场景空间求交。世界空间探针复用可以减少对当前屏幕的依赖，但仍受覆盖范围、更新预算和方向分辨率限制。

### 6.3 单独评估反射路径的精度与成本

`[产品：5.6]` Lumen 可以从 Surface Cache 查询反射命中处的照明，也提供成本更高的 **Hit Lighting for Reflections**。镜面方向对位置、法线和遮挡很敏感。因此，用于粗糙表面 GI 的近似，不一定适用于镜面反射；不能仅靠模糊漫反射 GI 得到所需反射。[LT]

`[教学]` MegaLights 与 Lumen 可以共存，并共享 Ray Tracing Scene 的部分开销。但共用 BVH，不能证明它们共用光照缓存，也不能证明一个系统的开关控制了另一个系统的全部直接光评估。具体版本的集成状态未核验前，无法确认 Lumen 每次命中着色都会调用完整的 MegaLights 主视图路径。

### 6.4 分开排查缓存更新延迟与去噪历史滞后

`[产品：5.6]` Lumen 性能文档分别说明了 Surface Cache 的直接光照（Direct Lighting）与辐射度（Radiosity）更新比例，以及 Radiance Cache 探针的更新预算。更新工作可以分摊到多帧。相机快速移动或光照突然变化时，缓存更新不及时就可能造成响应延迟。[LP]

`[教学]` 排查时先检查缓存照明是否已经更新，再检查主视图历史是否仍保留旧光。增加去噪强度可能掩盖噪声，但也可能加重主视图历史的响应滞后。

## 7. ReSTIR 与 RTXDI：复用样本及其统计量

### 7.1 通过储备池表示候选样本集合

`[论文]` ReSTIR 2020 对候选光样本进行重采样，并复用时间和空间邻域的样本。储备池用少量保留样本和统计量，表示已处理的候选集合，而非保存旧像素颜色的平均值。原研究讨论了无偏（unbiased）和有偏（biased）估计。不同配置有不同取舍，不能认定所有 ReSTIR 配置都无偏。[R20]

`[论文补充材料]` 邻域储备池的样本分布可能不同。将样本移到当前像素时，需要处理目标分布和归一化。补充材料 §1 与 Algorithm 1/2 讨论了多重重要性采样（MIS）：估计 MIS 权重时使用了什么分布近似，以及最终估计量是否无偏，需要分别论证。两个量即使都称为 PDF，也不能据此直接互换。[R20S]

### 7.2 按固定代码提交检查各阶段

2026-10-13 核验记录中实际检出的版本为：

- **RTXDI 3.1.0**：`a6efab966b7c3b272da0461578eb56ac61c7cbff`，提交日期 2026-09-03。
- 对应的 **RTXDI-Library 子模块**：`f12037fa8e97ebc08e9e3edfd2de528ed1772a4b`，提交日期 2026-09-03。

下表提供代码阅读入口和核验内容。表中解释不转载 NVIDIA 实现，也不将这些字段映射到 UE。

| 阅读入口 | 可核验的行为 | 对共同框架的意义 |
|---|---|---|
| [`GenerateInitialSamples.hlsl` 的 `main` / `RayGen`][CInit] | 读取 `RAB_GetGBufferSurface`，调用 `RTXDI_SampleLightsForSurface`，写入储备池 | 将 G-buffer 与初始采样连接起来 |
| [`Reservoir.hlsli` 的 `RTXDI_DIReservoir`、`RTXDI_StreamSample`][CR] | 保存样本引用、目标值和统计量；流式处理候选，按权重选择保留样本 | 储备池保存样本及统计量，不保存已平均的图像 |
| [`TemporalResampling.hlsli` 的 `RTXDI_DITemporalResampling`][CT] | 通过运动向量回投；检查表面有效性、深度和法线；限制历史 `M`；映射灯索引；在当前表面重算目标值；按选项校正偏差 | 每次复用都需要处理有效性和权重 |
| [`SpatialResampling.hlsli` 的 `RTXDI_DISpatialResampling`][CSpatial] | 访问相容邻域并合并候选；归一化与偏差校正有不同路径 | 空间复用需要样本兼容与权重处理，不能直接复制邻域颜色 |
| [`ShadingHelpers.hlsli` 的 `ShadeSurfaceWithLightSample`][CS] | 按设置尝试复用可见性，否则执行追踪；使用储备池权重与光样本密度信息着色 | 最终能量与选样统计必须匹配 |
| [`Reservoir.hlsli` 的 `RTXDI_GetDIReservoirVisibility`][CR] | 检查可见性记录的存续时间（age）和空间距离（spatial distance） | 可见性缓存有独立的有效条件 |
| [`Confidence.md` 及关联着色器][CC] | 比较同一表面／光样本跨帧的照明变化，形成去噪置信度 | 样本复用与图像历史可以相互提供信息，但仍需分层处理 |

灯索引映射失败时，时间重采样代码清除旧样本的有效引用和权重，并通过归一化逻辑处理统计量。它没有简单地将整个结构的所有字段归零。因此，不能将该行为概括为“失效就 memset reservoir”。[CT]

`weightSum` 的含义随阶段变化：流式阶段表示权重和，最终化（finalize）后用于储备池贡献权重。函数名或变量名中出现 PDF，不能证明该量就是精确的最终抽样密度。[CR]

### 7.3 分别判断重要度与历史可信度

`[教学]` 重要度用于判断当前优先测试哪些候选；历史可信度用于判断旧记录现在是否仍可靠。例如，一盏灯的当前贡献可能很重要，但移动遮挡使其旧可见性不可靠。另一块墙的历史可能很稳定，却只接收很弱的光。因此，不能仅凭重要度高就提高历史权重。

`[代码]` 在本文核验的时间重采样路径中，旧光样本根据当前表面重新计算目标值，历史候选数也受到限制。可见性复用则另行检查存续时间与距离。更强的偏差校正可能增加射线。代码注释还指出：需要上一帧可见性，却只能使用当前 BVH 时，偏差会增大。[CT][CR]

`[教学]` 初始候选评估、复用校正和最终着色是否执行追踪，取决于策略。仅观察最后一次阴影射线（shadow ray），不足以推断整个 ReSTIR 处理阶段的射线预算。同样，不能认定 ReSTIR 的每个候选都必须追踪一条阴影射线。[CN]

### 7.4 GI 与 PT 扩展了复用对象

| 研究／实现 | 复用对象与实现范围 | 适用边界 |
|---|---|---|
| **ReSTIR DI，2020** | 对当前接收面有贡献的光样本 | 不能仅更改名称就用于多跳 GI |
| **ReSTIR GI，2021** | 时空复用间接路径产生的样本，并处理不同接收点之间的路径连接关系 | 不能简化为缓存 GI 颜色后取平均 |
| **GRIS / ReSTIR PT，2022** | 从理论上处理相关样本、不同域和转移映射（shift mappings），并展示多跳漫反射／镜面反射路径复用 | 不能保证直接复制任意路径的储备池就得到正确结果 |
| **RTXDI 3.1.0** | 提供 DI / ReGIR / GI / PT 工程模块；应用负责场景、材质和可见性接口 | 集成需要修改渲染器，不能作为无需适配的自动光照插件使用 |

来源：[ReSTIR GI 作者项目页][RGI]、[GRIS / ReSTIR PT 作者项目页][RPT]、[RTXDI 集成说明][CI]。GI / PT 正文 PDF 的访问限制见第 13 节；本文没有根据摘要推导完整数学证明。

`[代码／产品文档]` RTXDI 3.1.0 的 GI 集成文档说明，其基本的次级表面辐射亮度（secondary-surface radiance）表示不含方向性，对低粗糙度表面和尖锐焦散有局限。文档还讨论了最终着色中的 MIS 组合。因此，SDK 提供 ReSTIR GI，不能保证所有镜面、折射和焦散都获得同等质量。[CGI]

## 8. 按版本和路径分析透明与体积光照

### 8.1 确认文档对应的版本

| 资料范围 | 原核验记录中的说明 | 结论范围 |
|---|---|---|
| **MegaLights UE 5.6 文档** | 不支持透明光照（translucency）；支持体积雾（Volumetric Fog） | 不能将后续透明功能写成 5.6 已有的功能 |
| **MegaLights UE 5.7 文档** | 受光透明物体与雾使用统一的视锥体素体积；前向着色的镜面反射通过球谐函数（SH）提取近似光表示 | 这是空间光照近似，不代表所有透明层都逐像素完整计算光照 |
| **MegaLights 无版本入口，读取时标题为 UE 5.8** | 最前方的前向着色透明层使用专用 G-buffer 和光照处理阶段；后层使用体积表示 | 最前层可获得更细的表面光照，后层仍有不同的质量限制 |
| **SIGGRAPH 2025 讲稿** | 描述作者当时的设计 | 不能据此确定每个发行版、每个质量档位的默认行为 |

来源分别为 [5.6][M56]、[5.7][M57]、[无版本入口][M58]、[2025 讲稿][M25]。本表记录的是 **2026-10-13 的读取结果**，不能用于确定功能的首发版本。版本化网页仍可能被维护更新，无版本入口的内容尤其可能变化。

### 8.2 区分最前透明层与后层的光照近似

`[产品：读取时为 5.8 的页面]` 最前透明层（front layer）路径额外记录该表面的属性，再执行采样、追踪和去噪。后方透明层从 Translucency Lighting Volume 获取漫反射光照，并用单个主方向近似镜面反射。最前透明层的 G-buffer 可以与 Lumen Front Layer Reflections 共享部分开销，但后续直接光照仍需要单独计算。[M58]

`[教学]` 玻璃上的窄高光依赖精确的表面法线、视角和光源方向，粗粒度体积表示很难保留这些细节。雾需要在空间中连续积分散射，适合先在体积格点计算，再重建结果。两条路径即使都使用视锥体素或体积表示，误差表现也可能不同。

透明物体受光、向其他物体投下透射阴影，以及折射后正确显示多层照明，是三个不同问题。支持受光，不能证明后两项也受支持。不透明表面的运动向量和深度历史，也不能未经处理就用于所有透明层。

前向着色透明路径（Forward-shaded translucency）与项目级前向渲染器（Forward Renderer）由不同配置控制。原核验读取的 MegaLights 文档仍列明其与 Forward Renderer 不兼容。透明材质具有前向着色路径，不能证明项目级前向渲染器受支持。[M57][M58]

## 9. 按数据层排查噪声、拖影、漏光和响应延迟

以下内容标记为 `[教学：基于来源的排查推导]`，需要结合具体引擎判断，不能作为通用自动诊断。排查时每次只改变一个变量，同时观察原始照明、重建输出和场景表示。

| 现象 | 优先检查的可能原因 | 区分原因的实验 | 不能直接得出的结论 |
|---|---|---|---|
| 静止画面有随机亮点或颗粒 | 候选分布不合适、强小光源难以抽中、样本不足，或间接路径难以采样 | 固定相机和曝光，比较原始照明与累积结果；收紧无必要的光源影响范围 | 去噪越强越好 |
| 大片噪斑缓慢变化 | 重采样相关性、缺少新信息，或去噪器误判信号已经稳定 | 观察原始采样序列，对比复用策略和历史长度 | 储备池的 M 大，就代表独立的新样本多 |
| 物体移动后留下亮边或暗尾 | 运动向量错误、新显露区域未充分拒绝历史，或复用了过期可见性 | 对同一动画比较原始照明与过滤结果，检查历史重置和置信度 | 原因一定是 TAA |
| 墙后持续漏光，长时间不收敛 | 光线追踪几何缺失或简化、剔除、符号距离场（SDF）的厚度或覆盖、最大追踪距离，或跨边界插值 | 对照主视图与 Ray Tracing Scene／距离场；分别观察屏幕空间追踪和场景追踪 | 增加样本就能修复 |
| 镜中出现黑块，或 GI 随视角变化突然出现 | Surface Cache 或追踪覆盖不足，屏幕路径掩盖了缺失 | 检查 Surface Cache 覆盖与反射视图，并与 Hit Lighting 做局部对照 | 原因全部是采样噪声 |
| 开灯或关灯后，画面缓慢变化 | 光照缓存未更新、去噪历史滞后，或两者叠加 | 先检查缓存是否及时响应，再检查屏幕过滤后的响应 | 只需调整时间混合参数 |
| 后方透明层的阴影或高光模糊 | 体积近似、空间分辨率不足，或该版本没有最前透明层路径 | 在同一视角分别检查最前层与后层，确认版本和材质路径 | 可以直接套用不透明路径的参数 |

依据：[MegaLights 文档][M56]、[Lumen 排错说明][LT]、[Lumen 更新预算][LP]、[RTXDI Noise and Bias][CN]、[Confidence][CC]。噪声表现为有限观测下的波动；有偏估计即使取平均，也可能偏离正确结果。图像更平滑，不代表缺失几何导致的漏光、跨阴影边界的错误复用或历史滞后已经修复。

## 10. 确认平台、渲染器和引擎版本的适用范围

| 场景 | 判断依据与适用范围 |
|---|---|
| **PC／本世代主机的 UE5 项目** | 分别预算 MegaLights 直接光照和 Lumen GI／反射。检查渲染器、光线追踪能力、版本和可伸缩性设置（scalability），再在目标机上测量。示例帧率不能保证项目帧率 |
| **移动端 MegaLights** | 原核验读取的 Epic 文档仍列明不支持移动端。Arm 于 2026-06-10 公开了基于 UE 5.7 修改的 Lumilings 实验，并明确其不属于 UE 官方移动端支持。该实验不能证明方案普遍具备产品发布条件 |
| **移动端 Lumen** | 5.6 移动端专页给出的实验条件为高端 Android、Vulkan SM5 和桌面渲染器；可以使用或不使用硬件光线追踪（HWRT）。这不代表默认移动端前向／延迟渲染器均可使用。该页还列明不支持 iOS/tvOS/iPadOS |
| **UE4 学习或维护项目** | 可以学习 G-buffer、BRDF、灯光剔除、阴影贴图／BVH、概率采样、运动向量和去噪。MegaLights／Lumen 的 UE5 产品流程和控制台变量不能直接用于 UE4 |
| **在独立渲染器中学习 RTXDI** | 公开着色器和 SDK 接口可用于验证方法。完整集成仍需要应用提供 G-buffer、材质、灯光表示和可见性等接口，不能仅靠替换材质节点完成 |

来源：[Epic 平台范围][M58]、[Arm 移动端实验][ARM]、[Lumen on Mobile 5.6][LM]、[RTXDI Integration][CI]。Lumen 总体技术页与移动端专页对渲染器的表述不完全一致；本文采用移动端专页中更具体的条件。

`[教学]` 评估移动端时，除射线数外，还需要考虑渲染目标与历史数据的带宽、缓存读写、功耗和热稳定性。Arm 博客中的实验结果只证明方案在其测试条件下可运行。博客面向入门读者的 ReSTIR 描述，不能替代原论文与 RTXDI 代码，也不能证明 ReSTIR 对每个候选都执行昂贵的可见性测试。

## 11. 区分理论前置、研究扩展和产品实现

| 时间 | 相关变化 | 与其他技术的关系 |
|---|---|---|
| 2005：Talbot 等，RIS | 先从易采样的分布产生候选，再向目标分布重采样 | ReSTIR 的理论前置；原核验通过论文补充材料与 SDK 文档核对这一关系 |
| 2017：SVGF | 用时空信息和方差指导，从低样本照明重建图像 | 属于重建分支，与 ReSTIR 分工不同 |
| 2020：ReSTIR DI | 将储备池采样与时空样本复用用于复杂直接光照 | 属于采样复用分支 |
| 2021：ReSTIR GI；2022：GRIS / ReSTIR PT | 扩展到间接路径、相关样本和跨域映射 | 属于研究扩展，不能仅按样本数比较 |
| 2022：Lumen 作者分享 | 用多种场景表示与光照缓存组织动态 GI／反射 | 属于产品系统，可以与重采样研究比较，但两者不等同 |
| 2025：MegaLights 作者分享 | 用有限采样、历史引导和重建处理大量动态直接光 | 属于产品路径，不能认定它是 ReSTIR 更名后接入 UE 的结果 |
| 2026：本文核验的 RTXDI 3.1.0 | 同时提供 DI / ReGIR / GI / PT 模块和重建接口 | 是 SDK 在固定版本中的具体工程实现 |

### 光照方案比较模板

`[教学]` 评估新的光照方案时，先填写以下信息，再结合演示画面判断：

```text
目标积分：______  接收／求解域：______  新采样来源：______
复用的数据：______  对应关系／索引：______  权重校正：______
几何与可见性来源：______  重建方式：______
更新预算：______  失效与回退：______  平台／版本：______
证据：论文______ / 产品文档______ / 固定代码______ / 实测______
```

## 12. 技术美术的实践步骤

以下实验属于 `[教学建议]`。原研究及本次改写均未运行 UE 基准测试，也未改变知识库中的已读／掌握状态。

1. **两盏灯的采样实验。** 固定接收面与遮挡，计算全部灯的参考结果，再按已知概率采样。检查长期平均是否与参考一致。随后去掉概率补偿，观察错误。使用第 2.4 节的离散条件即可，无需先实现 ReSTIR。
2. **门开关的历史实验。** 固定相机，将强光源放在门后。记录原始照明、选灯引导或储备池数据（若工具提供），以及过滤输出。分别确定探索发现灯、旧可见性失效和去噪历史清理发生在哪一层。
3. **离屏遮挡实验。** 将遮挡物移出画面，但保持其位于接收点与灯的连线上。比较仅使用屏幕数据的追踪与包含该物体的场景追踪，并检查剔除。不要用删除物体代替将物体移出画面。
4. **Lumen 缓存实验。** 将相机快速转向此前未关注的区域，再开关一盏强光源。对照 Lumen Scene／Surface Cache 与最终 GI，区分覆盖范围、更新预算和屏幕历史的影响。
5. **透明分层实验。** 确认 UE 版本后，放置前后两层透明表面，以及产生窄高光的光源。比较最前层与后层的结果，并结合具体路径判断，不能只检查 MegaLights 是否开启。

实验记录至少包含以下信息：

- 引擎精确版本、目标 GPU、渲染器和质量档位。
- 内部分辨率、输出分辨率和测试路径。
- 静态与动态场景条件。
- GPU 耗时，以及原始信号和重建信号的截图。

实验用于解释误差来源。性能数字需要与测试条件一同记录。

### 相关知识库条目

- 前置知识：[[Rendering Equation]] 说明需要估计的量；[[BRDF]] 说明表面响应；[[Shadow Mapping]] 介绍可见性的一种表示。GitHub 入口：[Rendering Equation](../Concepts/Rendering%20Equation.md) · [BRDF](../Concepts/BRDF.md) · [Shadow Mapping](../Concepts/Shadow%20Mapping.md)。
- 上位主题：[[Real-Time Global Illumination]]。本文补充该主题的机制框架，未引用其中未经原研究独立核验的产业消息或性能数字。[技术主题页](../Technologies/Real-Time%20Global%20Illumination.md)。
- 工程联系：[[Temporal Stability and Artistic Intent]]、[[Real-Time VFX Performance Budgeting]]、[[Participating Media]]。对应条目为 [时间稳定性](../Concepts/Temporal%20Stability%20and%20Artistic%20Intent.md)、[预算](../Concepts/Real-Time%20VFX%20Performance%20Budgeting.md)、[参与介质](../Concepts/Participating%20Media.md)。

## 13. 关键结论的证据与核验范围

**原技术核验日期：2026-10-13。** 本次 STE 改写仅调整表达，未重新执行技术核验。下表中的“已读”“已核验”均指原核验记录。

“已读代码”仅表示检查了指定文件或函数，不表示已编译或运行 SDK。原核验未访问、搜索或引用 NGR 私有源码，也未读取 Epic 授权 UE 源码。本文 SVG 均为自行绘制的逻辑示意，不是引擎截图或原讲稿图片。

| 结论 | 一手依据与定位 | 日期／版本 | 原核验记录与限制 |
|---|---|---|---|
| 四阶段可用于比较技术，但需要区分复用对象 | 本文对下列来源的综合 | 本文原核验记录，2026-10-13 | 属于教学综合，不是某篇论文提出的统一架构 |
| MegaLights 使用固定采样预算，屏幕空间追踪可转到场景追踪 | [Technique Overview / Performance][M56] | UE 5.6 页面 | 已读官方正文；固定样本预算不代表整帧成本恒定 |
| 2025 讲稿的屏幕分块引导不能称为 ReSTIR 储备池 | [MegaLights 2025 slides][M25]，检索 `8x8`、`guiding` | SIGGRAPH 2025 | 在线索引已确认分块与深度不连续问题；重投影和随机查询细节来自当次只读研究补充。完整 PDF 和准确页码未核实 |
| 透明路径随版本变化 | [Limitations 5.6][M56]、[Lit Translucency / Forward Shading 5.7][M57]、[Forward Shaded Translucency 无版本入口][M58] | 5.6 / 5.7 / 标为 5.8 | 已逐版读取；未考证功能首发版本 |
| Surface Cache、屏幕光照收集、世界空间辐射亮度缓存各有用途 | [作者博客的 Surface caching / Final gather and denoising][LB] | 2022-05-27 | 已读作者正文；未据此推断当前全部 GPU 处理阶段 |
| Cards 的早期追踪实验不能代表当前实现 | [Journey to Lumen，Shipping First Demo / Epilogue][LJ] | 2022-08-18 | 已读作者复盘，区分了已废弃的探索与后来的用途 |
| Lumen 2022 技术分享的资料入口 | [SIGGRAPH 2022 course][L22]，`Lumen: Real-time Global Illumination in Unreal Engine 5` | 讲稿更新 2022-09-19 | 已核验作者、课程和资料链接。完整 PDF 受大小限制；详细机制以已读官方博客和文档交叉支撑 |
| Surface Cache 与探针各有更新预算 | [Surface Cache / Radiance Cache 等性能说明][LP] | UE 5.6 | 已读官方正文；不能将全部延迟归因于去噪 |
| ReSTIR 2020 重采样样本，不同估计量有不同取舍 | [原始作者项目页][R20]、[原论文 PDF][R20PDF]、[supplement §1、Algorithm 1/2][R20S] | TOG / SIGGRAPH，2020-07 | 已读项目页和两页原始补充正文。主文 PDF 过大，未完整通读证明 |
| GI / PT 不能机械套用直接光储备池 | [ReSTIR GI][RGI]、[GRIS / ReSTIR PT][RPT] | 2021-06-24 / 2022-07-24 | 已读作者摘要和元数据；完整 PDF 未载入，证明细节未核验 |
| 储备池、灯消失、历史长度、可见性存续时间与距离的处理行为 | [Reservoir][CR]、[TemporalResampling][CT]、[SpatialResampling][CSpatial] | Library `f12037f…`，2026-09-03 | 已检出代码并阅读指定函数；仅证明该提交的行为 |
| 初始采样、最终着色与重建信号准备属于不同阶段 | [GenerateInitialSamples][CInit]、[ShadingHelpers][CS]、[ShadeSamples][CShade] | RTXDI `a6efab9…`，3.1.0 | 已读代码；未转载缓冲区二进制布局 |
| 重要度、偏差与去噪可信度需要分别判断 | [NoiseAndBias][CN]、[Confidence][CC] | 同一 RTXDI 提交 | 已读 SDK 文档，并与实际函数交叉核验 |
| SVGF 研究图像重建，不负责选灯 | [SVGF 作者研究页][SVGF] | HPG 2017，2017-07-28 | 已读研究摘要；原始大 PDF 未载入，完整过滤器实现细节未核验 |
| Lumen 移动端路径有具体实验条件 | [Lumen on Mobile][LM] | UE 5.6；页面说明自 5.4 起实验支持 | 已读移动端专页，采用其条件限定平台范围 |
| MegaLights 移动端实验不代表 Epic 标准支持 | [Arm 的 Lumilings 文章，How to enable Megalights for mobile][ARM] 与 [Epic Platform Support][M58] | Arm 2026-06-10，基于 5.7 | 已读两个一手来源；未将 Arm 简化的算法比较作为 ReSTIR 定义 |

以下内容仍未确认：

- UE 内部缓冲区布局。
- 各版本、各质量档位的默认样本数和分块配置。
- 精确的采样引导 PDF，以及完整失效阈值。
- MegaLights 与 Lumen 所有分支的集成状态。
- 本项目目标机的性能。

原核验记录显示，大型原始 PDF 超出网页工具的文件大小限制，执行环境直连下载返回 403。因此，本文没有提供未经核实的讲稿页码、完整论文阅读记录或 UE 源码验证结论。上表保留原始入口，便于后续补充核验。

### 来源链接

产品与作者材料：

- [M56：Epic MegaLights，UE 5.6][M56]；[M57：UE 5.7][M57]；[M58：无版本入口，原核验时标题为 UE 5.8][M58]。
- [M25：Krzysztof Narkowicz、Tiago Costa — MegaLights: Stochastic Direct Lighting in Unreal Engine 5，SIGGRAPH 2025][M25]。
- [LB：Daniel Wright、Krzysztof Narkowicz — Unreal Engine 5 goes all-in on dynamic global illumination with Lumen，2022-05-27][LB]。
- [L22：Daniel Wright、Krzysztof Narkowicz、Patrick Kelly — Lumen: Real-time Global Illumination in Unreal Engine 5，SIGGRAPH 2022][L22]；[原讲稿 PDF][L22PDF]。
- [LJ：Krzysztof Narkowicz — Journey to Lumen，2022-08-18][LJ]。
- [LT：Epic Lumen Technical Details，5.6][LT]；[LP：Lumen Performance Guide，5.6][LP]；[LM：Lumen on Mobile，5.6][LM]。
- [ARM：Arm — Lighting at scale: Bringing hundreds of dynamic lights to mobile with Unreal MegaLights，2026-06-10][ARM]。

论文与可读代码：

- [R20：Bitterli 等 — Spatiotemporal reservoir resampling for real-time ray tracing with dynamic direct lighting，2020][R20]；[主文][R20PDF]；[原始补充材料][R20S]。
- [RGI：Ouyang 等 — ReSTIR GI: Path Resampling for Real-Time Path Tracing，2021][RGI]。
- [RPT：Lin 等 — Generalized Resampled Importance Sampling: Foundations of ReSTIR，2022][RPT]。
- [SVGF：Schied 等 — Spatiotemporal Variance-Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination，2017][SVGF]。
- [RTXDI 固定提交][RTXDI] 与 [RTXDI-Library 固定提交][LIB]。代码可公开读取，文件带有 NVIDIA 许可标识。公开可读不能证明全部代码采用宽松开源许可。本文提供原创解释和定位链接，不转载实现。

[M56]: https://dev.epicgames.com/documentation/en-us/unreal-engine/megalights-in-unreal-engine?application_version=5.6
[M57]: https://dev.epicgames.com/documentation/en-us/unreal-engine/megalights-in-unreal-engine?application_version=5.7
[M58]: https://dev.epicgames.com/documentation/en-us/unreal-engine/megalights-in-unreal-engine
[M25]: https://advances.realtimerendering.com/s2025/content/MegaLights_Stochastic_Direct_Lighting_2025.pdf
[LB]: https://www.unrealengine.com/tech-blog/unreal-engine-5-goes-all-in-on-dynamic-global-illumination-with-lumen?lang=en-US
[L22]: https://advances.realtimerendering.com/s2022/index.html#Lumen
[L22PDF]: https://advances.realtimerendering.com/s2022/SIGGRAPH2022-Advances-Lumen-Wright%20et%20al.pdf
[LJ]: https://knarkowicz.wordpress.com/2022/08/18/journey-to-lumen/
[LT]: https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-technical-details-in-unreal-engine?application_version=5.6
[LP]: https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-performance-guide-for-unreal-engine?application_version=5.6
[LM]: https://dev.epicgames.com/documentation/en-us/unreal-engine/using-lumen-global-illumination-on-mobile-in-unreal-engine?application_version=5.6
[ARM]: https://developer.arm.com/community/arm-community-blogs/b/mobile-graphics-and-gaming-blog/posts/lighting-at-scale-bringing-hundreds-of-dynamic-lights-to-mobile-with-unreal-megalights
[R20]: https://research.nvidia.com/labs/rtr/publication/bitterli2020spatiotemporal/
[R20PDF]: https://research.nvidia.com/sites/default/files/pubs/2020-07_Spatiotemporal-reservoir-resampling/ReSTIR.pdf
[R20S]: https://cs.dartmouth.edu/~wjarosz/publications/bitterli20spatiotemporal-supplemental.pdf
[RGI]: https://research.nvidia.com/publication/2021-06_restir-gi-path-resampling-real-time-path-tracing
[RPT]: https://research.nvidia.com/publication/2022-07_generalized-resampled-importance-sampling-foundations-restir
[SVGF]: https://research.nvidia.com/publication/2017-07_spatiotemporal-variance-guided-filtering-real-time-reconstruction-path-traced
[RTXDI]: https://github.com/NVIDIA-RTX/RTXDI/tree/a6efab966b7c3b272da0461578eb56ac61c7cbff
[LIB]: https://github.com/NVIDIA-RTX/RTXDI-Library/tree/f12037fa8e97ebc08e9e3edfd2de528ed1772a4b
[CI]: https://github.com/NVIDIA-RTX/RTXDI/blob/a6efab966b7c3b272da0461578eb56ac61c7cbff/Doc/Integration.md
[CN]: https://github.com/NVIDIA-RTX/RTXDI/blob/a6efab966b7c3b272da0461578eb56ac61c7cbff/Doc/NoiseAndBias.md
[CC]: https://github.com/NVIDIA-RTX/RTXDI/blob/a6efab966b7c3b272da0461578eb56ac61c7cbff/Doc/Confidence.md
[CGI]: https://github.com/NVIDIA-RTX/RTXDI/blob/a6efab966b7c3b272da0461578eb56ac61c7cbff/Doc/RestirGI.md
[CInit]: https://github.com/NVIDIA-RTX/RTXDI/blob/a6efab966b7c3b272da0461578eb56ac61c7cbff/Samples/FullSample/Shaders/LightingPasses/DI/GenerateInitialSamples.hlsl
[CR]: https://github.com/NVIDIA-RTX/RTXDI-Library/blob/f12037fa8e97ebc08e9e3edfd2de528ed1772a4b/Include/Rtxdi/DI/Reservoir.hlsli
[CT]: https://github.com/NVIDIA-RTX/RTXDI-Library/blob/f12037fa8e97ebc08e9e3edfd2de528ed1772a4b/Include/Rtxdi/DI/TemporalResampling.hlsli
[CSpatial]: https://github.com/NVIDIA-RTX/RTXDI-Library/blob/f12037fa8e97ebc08e9e3edfd2de528ed1772a4b/Include/Rtxdi/DI/SpatialResampling.hlsli
[CS]: https://github.com/NVIDIA-RTX/RTXDI/blob/a6efab966b7c3b272da0461578eb56ac61c7cbff/Samples/FullSample/Shaders/LightingPasses/ShadingHelpers.hlsli
[CShade]: https://github.com/NVIDIA-RTX/RTXDI/blob/a6efab966b7c3b272da0461578eb56ac61c7cbff/Samples/FullSample/Shaders/LightingPasses/DI/ShadeSamples.hlsl
