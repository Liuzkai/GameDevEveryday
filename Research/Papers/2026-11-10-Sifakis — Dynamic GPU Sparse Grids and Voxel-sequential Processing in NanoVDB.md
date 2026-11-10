---
type: paper
title: "Dynamic GPU Sparse Grids and Voxel-sequential Processing in NanoVDB"
authors: [Eftychios Sifakis, Francis Williams, Christopher Horvath, Rohan Sawhney, Mark Harris, Jonathan Swartz, Ken Museth]
year: 2026
published: "2026-11-05（arXiv v1）；ACM TOG 45(6) Article 225, December 2026"
venue: "ACM Transactions on Graphics 45(6), Article 225（DOI 10.1145/3842564）"
url: "https://arxiv.org/abs/2611.08665"
code: "https://github.com/AcademySoftwareFoundation/openvdb"
project_page: "https://github.com/openvdb/fvdb-core"
category: [volumetric, gpu-data-structure, simulation, level-set, voxelization, rendering, engine-runtime]
importance: A-
historical_importance: 3
game_relevance: 4
production_readiness: Early Production
user_level: Normal
status: unread
aliases: [Dynamic GPU Voxel Grids, NanoVDB Dynamic Topology, VoxelBlockManager, VBM, voxel-sequential access, 动态稀疏网格]
tags: [volumetric, vdb, gpu, data-structure, simulation, level-set, voxelization]
---

# Dynamic GPU Sparse Grids（Sifakis et al. 2026）

## TL;DR

**"只读"不是 NanoVDB 这个结构的天性，是维护它的算法的限制。**

原话更狠：`we demonstrate that static topology is not an inherent trait of the NanoVDB data structure, but rather a limitation of the algorithms previously used to maintain it.`

这篇补上两件东西，方向**正交**：

1. **拓扑能改了**：五个批量同步（bulk-synchronous）的 GPU 拓扑算子——morphological dilation / pruning / merging / coarsening / refinement。做法不是"逐体素插入删除"，而是**先集体决定目标拓扑，再一次性重建整棵树**，用位并行算法把上亿体素的树重建压到毫秒。
2. **顺序访问能跑了**：新结构 **VoxelBlockManager（VBM）**，把活跃体素流成一条**平铺的线性序列**，把树遍历从内层循环里删掉。开销 **≤1.5 bit/活跃体素**。

效果：567 M 活跃体素的 dilate **2.0 ms**；2.8 M 三角形的网格体素化成 **9.88 B 活跃体素用 3.24 s**；KITTI 上 **1.67 G** 个 ESDF 体素塞进 38.6 GB（nvblox 直接 OOM）。

## Problem

VDB 生态此前是一个**三选二**的局面：

| | 稀疏 | 随机访问快 | 拓扑能改 |
|---|---|---|---|
| 稠密网格 | ❌ | ✅ | ✅（但没意义） |
| 八叉树 / brick | ✅ | 🟡（深度大、指针追逐） | ✅ |
| DT-Grid / H-RLE | ✅✅ | 🟡（必须按序访问） | 🟡（重构代价大） |
| **OpenVDB 2013** | ✅ | ✅（accessor） | ✅（体素粒度） |
| **NanoVDB 2021** | ✅ | ✅✅（线性布局） | ❌ **只读** |
| **Index-Grid 2024** | ✅✅（<2 bit/voxel） | ✅✅ | ❌ **只读** |

NanoVDB 的做法是**用能力换速度**：放弃堆分配与指针，改成线性化布局 + 相对寻址，于是对 SIMT 极其友好——**代价是拓扑冻结，创建后不能再改**。结果是开发者在两条路里二选一：

> **CPU 上的动态拓扑（OpenVDB），还是 GPU 上的只读速度（NanoVDB）。**

而**模拟**恰好同时需要三样：窄带要随界面移动（拓扑变）、每步要走 stencil（快）、每步还要随机查邻近（随机访问）。

## Historical Context

```text
2003  窄带 level set 的八叉树 / 单层 block（Losasso / Bridson / Lefohn）
  ↓
2006  DT-Grid（CRS 的 3D 类比）、H-RLE（RLE 逐轴推广）—— 极省，但按序访问
  ↓
2013 ★ OpenVDB / VDB：浅而宽的 B+ 树 + 每节点位掩码
        同时拿到 稀疏 + 随机访问 + 动态拓扑；代价是指针与堆 ⇒ CPU only
  ↓
2014  HDDA（Museth）：把树变成"光线可以直接跳过的加速结构"
  ↓
2021  NanoVDB：树线性化 + 相对寻址 ⇒ GPU；但拓扑冻结
  ↓
2024  Index-Grid / fVDB（Williams et al.）：拓扑与数据分离，sidecar 存值
        < 2 bit/体素，稀疏度拉满 —— 仍然只读
  ↓
2026 ★ 本篇：位并行拓扑算子 + VBM 顺序访问
        把"能改"补回来，且不放弃线性布局
```

**这条线的每一代都在做同一件事：把"树"这个抽象，换成"更少的一次间接寻址"。** 2013 是把树做浅做宽（少跳几次）；2021 是把树拍平（跳变成算）；2024 是把"拓扑"和"值"拆开（同一个跳服务多份数据）；2026 是把"改树"变成"重建一个平的表"（跳彻底消失）。

## Core Idea

### 想法一：不改树，重建树

OpenVDB 的拓扑更新是 **laissez-faire**：任何线程随时可以插入/删除一个体素。这套模型在 CPU 上可行（有锁、有堆），在 GPU 上是灾难——非线程安全，且"分配"本身是串行动作。

本篇的替代是 **epoch 式批量同步**：

```text
(a) 集体决定：这一轮"哪里要变、按什么规则变"（一次全局判定）
(b) 批量实现：一次性生成一个新的、合法的拓扑与数据状态
(c) 恢复计算：在这个新的 epoch 上继续
```

**关键洞察：拓扑变更在语义上从来不是"逐体素"的。** dilate 是"每个活跃体素向 26 邻域扩一格"，prune 是"满足谓词的体素全删"，merge 是两个集合求并——**它们全都是集合到集合的函数**。既然是函数，就可以"先算出结果集合，再一次性建出来"，而不必"边算边改"。

代价写在明处：**操作期间必须同时持有"改前"和"改后"两套拓扑**。因为拓扑/数据已分离（Index-Grid），这份额外开销很小。

### 想法二：不要按叶节点派线程，要按活跃体素派线程

GPU 上的传统做法是"一个 thread block 处理一个 8³ 叶节点"——那么 512 个线程里有多少落在**非活跃**体素上，就有多少 SIMT 吞吐被浪费。稀疏场景下这个比例可以到 95%。

理想做法是"按活跃体素的线性顺序派线程"，可这需要一个**反向映射**：

- Index-Grid 给的是 **coord → index**（正向）：给定坐标，算出它是第几个活跃体素。
- 流式处理要的是 **index → coord**（反向）：给定"第 137 个活跃体素"，它在哪个叶节点、叶内偏移多少。

**VDB 此前从未提供这个反向映射。** VBM 就是它：每 $W$ 个活跃体素一个块，每块只存两样东西——`firstLeafID`（第一个体素所在叶节点，1 个 uint32）+ `jumpMap`（每活跃体素 1 bit，标记"这里换叶节点了"）。

**合计 ≤1.5 bit/活跃体素，与网格大小、payload 类型、叶节点占用率都无关。**

### 为什么这两件事必须一起做

只做拓扑算子，模拟仍然卡在 stencil 的访存上（accessor 的 warp divergence）；只做 VBM，网格没法在每步更新窄带（拓扑冻结）。

**而它们共享同一个前提：Index-Grid 的"拓扑 / 数据分离"。** 拓扑重建只动拓扑（几百 MB 的位图），值留在 sidecar 里，用 `inject` 按坐标把旧值搬过去。**没有这个分离，"重建树"会变成"重建树 + 搬几十 GB 的场"。**

## Technical Approach

### 拓扑重建的七步骨架

| 步 | 做什么 | 是否算子相关 |
|---|---|---|
| 1 | **候选 root tile 集**：宁可超配（多余的后面会被剪掉） | 相关 |
| 2 | **分配稠密位图 scratchpad**：每个候选 tile 一张 `Mask<5>`（4 KB）+ 32³ 张 `Mask<4>`（16 MB） | 通用 |
| 3 | **并行标记**目标 lower / leaf 节点（`atomicOr`） | 相关 |
| 4 | **前缀和 + 单块分配**：`upperCounts` / `lowerCounts`（popcnt）/ `leafCounts` → CUB 前缀和 → 一次性分配整个 Index-Grid | 通用 |
| 5 | **并行重建连接**：从位图直接写 childMask / 偏移 | 通用 |
| 6 | **填 active flag**（此时树已存在，但还没有体素是活跃的） | 相关 |
| 7 | 自底向上刷 metadata（偏移、包围盒、checksum） | 通用 |

**"稠密 scratchpad + 位并行 + 前缀和"这套组合是全文的工程核心。** 它把"稀疏结构的动态构建"翻译成了"稠密位图上的布尔运算 + 一次扫描"，而这两件事 GPU 都极擅长。

### dilate 的三个非显然技巧

1. **leaf-major → tap-major 转置。** 原始状态是"每个叶节点一张 27-tap 位图"（我该让哪些邻居变活跃）；直接写会产生大量不规则 atomic 竞争。改成"每个 tap 一张 32-叶位图"（这个偏移上，32 个叶节点里哪些要扩），就能用 `CUB::WarpReduce` 把 32 个叶的投票压成一次位并行写。
2. **`MaskShift<di,dj,dk>`**：编译期常量偏移的 3D 位位移，用 warp shuffle/rotate 实现。于是 27 个偏移的卷积变成 27 次"移位 + OR"。
3. **越界的位不丢**：移出本 lower node 的位，被累加进对应邻居的 mask 里——所以一个 kernel 输出 27 张结果位图（自己 1 张 + 26 个邻居）。

### VBM 的两个不变量（必须显式声明）

NanoVDB 的核心结构**并不保证**这两条，但所有现有 Index-Grid 生成工具事实上都满足：

1. **没有零活跃体素的叶节点**（这种叶对坐标→索引映射毫无意义）；
2. **活跃体素索引跨叶连续**：叶 $L$ 的最后一个活跃体素索引是 $K$，则叶 $L+1$ 的第一个是 $K+1$。

**因为 VBM 依赖它们，本篇把它们升级为"用 VBM 的应用必须维持的不变式"。**

解码：块内 `jumpMap` 的 popcount + 1 = 本块跨了几个叶；然后对每个叶并行访问它的 512 个体素，用 popcnt 算线性索引，把 `leafIndex[W]` / `voxelOffset[W]` 写进 shared memory。**没有全局访存，不需要精细同步。**

### stencil tap 的摊薄

WENO5 有 **19 个 tap**，但沿三个轴各 ±3——**最多只碰 4 个叶节点**。所以先把 4 个叶指针取进寄存器，19 个 tap 就编译成对它们直接寻址的数组读，**没有一次 per-tap 的树遍历**。

同类的：3×3×3 卷积（ML 里常见）27 个 tap 最多碰 7 个叶。

## Key Contribution

1. **Bulk-Synchronous Topological Evolution**：把"动态拓扑"重新定义成"批量同步的 epoch 重写"，并给出位并行实现。
2. **Injection / Sidecar 算子**：拓扑与 payload 分开演化，`inject` 按坐标交集搬数据。
3. **Voxel-Sequential Access + VBM**：用 ≤1.5 bit/体素换来"按活跃体素派线程"，消除 warp divergence 与负载不均。
4. **验证**：在重建 / 流体 / 体素化三类真实负载上，与 nvblox / Open3D / VDBFusion / OpenVDB 正面比。

## Why It Works

### dilate 的那 2.0 ms 不是"优化出来的"，是"复杂度不同"

这是全文最漂亮的一段论证。567 M 活跃体素 dilate 用 2.0 ms。有人会说"那是位运算快"。作者直接把它变成**下界证明**：

> 一个 32-bit 带场（field-carrying）的等价实现，必须读写 $4(N_{in}+N_{out})\approx4.7$ GB 的场数据。在 1792 GB/s 的显存峰值下，**光这笔流量就要 2.63 ms**——已经比纯拓扑的 2.0 ms 慢。

**所以不是"我们的实现更好"，是"带场的实现无论多高效都追不上"。** 一旦你把"改拓扑"和"改值"解耦，后者就从必经路径上消失了。

> **可迁移判据：当你发现某个操作"不可能更快"时，先问它需不需要碰那些字节。不需要碰的字节，就不该出现在带宽账上。**

实测对照：vs 最好的旧路线（用 `voxelsToGrid` 从活跃坐标重建，且做了 tap 省略）**183× 快、139× 省显存**（356 MiB vs 48.2 GiB）；vs OpenVDB 的 TBB 主机实现 **76×**。

### 顺序访问为什么在稀疏时反而更值钱

accessor 的吞吐随叶节点占用率下降而下降（14.0 → 9.8 Gvoxel-updates/s），因为"一个 warp 32 个线程映射到 32 个体素槽"这个模式在稀疏时必然有线程空转。VBM 的吞吐**几乎是常数（~20）**，因为它的 launch 尺寸只由**活跃集大小**决定，与稀疏度无关。

于是**加速比随稀疏度上升**：1.43× → 1.98×（占用率 90% → 30%）。

> **判据：一个优化如果"在最坏情况下收益最大"，那它就是在修一个结构性问题，而不是在调参。**

### 为什么"重建"能赢"增量更新"

直觉上，改 1 个体素应该比重建整棵树便宜。但当稀疏结构的规模到了 $10^8$–$10^9$：

- 增量更新要维护"哪块内存归谁"的状态（分配器 / 锁 / 空闲表），这些状态本身就是随机访存；
- 重建是**一遍顺序读 + 一遍顺序写 + 一次前缀和**，全是带宽友好的规则操作。

**当"维护状态的代价"超过"重算一遍的代价"时，无状态重建就赢了。** 这是本篇与 11-08 那条"预算是初始化不是运行期"、以及本篇 Layer-2 式"无状态"的同一条思路——**在 GPU 上，"没有历史"常常是一种性能优势。**

## Performance（全部为论文实测）

| 项目 | 数字 | 对照 |
|---|---|---|
| dilate 567 M 活跃体素 | **2.0 ms** / 356 MiB | 重建路线 183× 慢、48.2 GiB（139×） |
| prune 同规模 | **8.4 ms** / 2.49 GiB | 12.1× 慢 / 16.9 GiB（7×） |
| WENO5 reinit 吞吐 | **~20 Gvoxel-updates/s（恒定）** | accessor 14.0 → 9.8（随占用率掉） |
| 1.67 G ESDF 体素 | **38.6 GB**（KITTI seq00，2 cm） | nvblox 直接 OOM |
| 同显存下可达体素尺寸 | **细 1.5–5×** | 活跃体素数多两个数量级 |
| occupancy 集成 | 比 nvblox **3.5–11.5×** 快 | — |
| LiDAR TSDF | **3.6–5.7×**（Mai City）/ **4.8–5.3×**（KITTI, 40 cm） | 20 cm 时收窄到 2.6–3.5× |
| 网格体素化 2.8 M 三角形 | 1.026 B 体素 / **2.30 s** | OpenVDB 1.025 B / 23.87 s（≈10×） |
| 同上，0.001 体素 | **9.88 B 体素 / 3.24 s** | OpenVDB 内存耗尽，未完成 |
| 动态场景衰减 | 持久体素 **−78%**（500 帧 KITTI seq00） | 两个算子，零积分器代码 |

## Limitations

### 论文自陈的五条（都很实在）

1. **批量同步 ⇒ 必须同时持两套拓扑。** 拓扑本身小，所以现实里不疼；但这是模型的固有成本。
2. **不是"逐体素"动态。** 刻意放弃 OpenVDB 的 laissez-faire 模型（它非线程安全、不可确定性并行）。**这是一次明确的能力取舍，不是遗漏。**
3. **scratchpad 每个候选 root tile 要 ~16 MB。** 4096³ 的 tile 下无所谓（payload 远大于此）；但**极小域**上，这套记账可能比"从零重建"还贵。
4. **dilation 半径 ≤ 叶节点边长（8 体素）**；非矩形 / 非对称 stencil 的位并行机会显著变少；**erosion 还没实现**（计划用 prune 表达）。
5. **coarsen / refine 没有端到端跑过**；跨分辨率的 `inject`（prolongation / restriction）留作未来工作。

### VBM 的适用边界（作者写得很清楚，比数字有用）

- 叶节点**密集**时收益消失——那时传统（叶级稠密）NanoVDB 网格更合适。
- **compute-bound 的 stencil**（WENO5、稀疏卷积 / GEMM）收益最大；**memory-bound 的小 stencil**（梯度、散度）看不出来。
- **空间相干但非流式**的访问（光线追踪、碰撞查询）仍应走 accessor。

> **这是"什么情况下不该用我的东西"的一节，含金量不低于方法本身。**

### 我自己的两处保留

- **一切性能都在 RTX 6000 Ada（48 GB）+ EPYC 上测**。消费级 / 主机 GPU 上，"位并行 + 前缀和"这套对 `popcnt` / warp shuffle / 共享内存带宽的依赖形态可能不同。论文没给跨架构数据。
- **"帧"这个词在全文没出现**。这是仿真 / 重建的批处理口径（每帧一次集成），不是实时渲染的帧预算口径。**把它换算成游戏帧预算时，要自己加回"上传/回读"与"同步"的成本。**

## Game Development Relevance

三个直接影响面：

1. **Houdini 侧是标准，且现在是 GPU 原生的。** Pyro / Vellum / MPM 的缓存格式就是 VDB；NanoVDB 已被 Cycles、Arnold、V-Ray、RenderMan 采用。**"每帧重建窄带"从"不可能"变成 2 ms 之后，实时烟火的可行性被重新打开。**
2. **SDF 碰撞与体素化。** 9.88 B 体素 / 3.24 s 这个量级意味着：**一个 280 万面的资产可以在 3 秒内变成任意精度的窄带 UDF**。对破坏预处理（预计算每个碎片的内部距离场）、贴花投影、体积 GI 的探针初始化，这是可用的时间尺度。
3. **数据结构层面的迁移价值最大。** 游戏里大多数"体积"系统是自建的（体素地形、体积雾、SDF 场）。这篇给的是**三件可搬的模式**：
   - 拓扑变更做成 epoch（先算后建），不做事后修补；
   - 拓扑与 payload 分离（换一套数据不重建结构）；
   - 顺序访问层（把"按结构单元派线程"换成"按有效元素派线程"）。

**不该用的地方**：Minecraft 式方块地形（那是 dense chunk + 面剔除，稠密才是对的）；任何"叶节点基本全满"的数据（VBM 收益归零）；单帧只有一个物体的小规模 SDF（不值得引入树）。

## Unreal Engine Relevance

**UE 没有内置 VDB，所以这不是"接入"，是"借模式"。**

- 三条可直接借的：
  1. **Niagara 的粒子/网格数据布局**：粒子常常是"按发射器分块"处理，块内空槽多——这正是访问器模式的 GPU 版本。**按活跃粒子线性化 + 一个位图记录"换发射器了"** 是 VBM 的直接类比。
  2. **自建体素/体积系统**：如果已有 SDF 或体素 GI，**"拓扑/值分离"是最便宜的一次重构**——把"哪些格子存在"做成位图，值放独立 buffer，改分辨率只换一份。
  3. **破坏预处理**：碎片距离场是离线资产，**体素化速度从 24 s 降到 2.3 s 之后，"按需生成"才进入工具链的可行区间**。
- **不该硬套的**：UE 的 `Texture3D` / Volume Texture 是稠密资源，与稀疏体表示不是同一层抽象，**不要试图用 UTexture3D 模拟 VDB**。

## Technology Evolution

```text
2013  VDB：浅而宽的 B+ 树 + 位掩码 → 稀疏 / 随机访问 / 动态拓扑三者兼得（CPU）
  ↓
2014  HDDA：树本身成为光线加速结构
  ↓
2021  NanoVDB：线性化 + 相对寻址 → GPU；用"只读"换速度
  ↓
2024  Index-Grid (fVDB)：拓扑 / 数据分离，<2 bit/体素 → 稀疏度拉满，仍只读
  ↓
2026 ★ 本篇：位并行批量拓扑算子 + VBM 顺序访问
        "静态不是结构的天性，是算法的限制"
  ↓
（待补）erosion、跨分辨率 inject、稀疏金字塔多重网格
```

## Relationships

### Based On

- **[[Museth 2013 — VDB High-Resolution Sparse Volumes with Dynamic Topology|Museth 2013（VDB）]]**：整个数据结构的源头。本篇的贡献可以精确表述为"**把 2013 年那个'动态拓扑'的能力，在保持 GPU 线性布局的前提下找回来**"。
- **NanoVDB 2021（Museth）/ Index-Grid 2024（Williams et al.）**：直接前身。本篇明确说"我们与 Index-Grid 协同"，因为它提供了拓扑/数据分离——没有这一层，"重建树"会变成"搬几十 GB 的场"。

### Extends

- **OpenVDB 的拓扑形态学**：dilate / erode / CSG 在 OpenVDB 里都有，但都是 CPU、指针、逐体素。本篇把其中四个搬成 GPU 批量算子，并新增 merge / coarsen / refine。

### Related

- **[[Fluid Simulation]]**：窄带 level set 是它的主战场（见今日对该页的回填：dilate → advect → reinit → prune 四步）。
- **[[Participating Media]]**：体积渲染读的就是这类网格；HDDA 的"直接遍历稀疏树"是它的加速结构。
- **[[GPU Architecture]]**：全文的立论基础是"显存带宽是硬下界"（2.63 ms 那段），以及"warp 的空转率决定稀疏算法的好坏"。
- **[[Streaming]]**：VBM 是"流式"在数据结构层的实现——**把随机访问模式改造成顺序访问模式**，与资源流式的动机同构（都在躲随机访存）。
- **[[Budget Units]]**：本篇给了一个极干净的"换单位"样本——**从"叶节点数"换成"活跃体素数"**，加速比 1.43→1.98× 全靠这一步。

### Contrasts

- **与 block-sparse hash grid（KinectFusion / nvblox / Voxblox / Open3D）**：那一路是"哈希表 + 每块堆分配 dense payload"，稀疏度高但**只支持"长出来"，不支持"重写"**（dilate / union / prune / refine）。判据：**只需要"新观测来了就加块" ⇒ 哈希够用；需要"整体重塑" ⇒ 必须批量拓扑。**
- **与 SPGrid（Setaluri 2014）/ MSBG（Braun 2025）**：同属"用规则内存布局避开树遍历"，但它们是**多层均匀网格金字塔**，不是"稀疏集合的拓扑重建"。本篇的 coarsen/refine 在概念上正对着它们（作者自己点了这个方向）。

### Followed By

- 待观察：erosion 的 prune 实现；跨分辨率 `inject`；以及 **NKSR / XCube 这类"靠 fVDB 拿拓扑"的神经方法会不会直接用上 refine 算子**（论文自己点名了这两个用例）。

## Personal Knowledge State

Current Level: **Normal**（推断）

- **Easy**：为什么稀疏结构要"树 + 位掩码"（你在 Houdini 里天天见 VDB，位掩码的直觉你有）；
- **Normal**：三选二的困局、bulk-synchronous 为什么适合 GPU、VBM 为什么需要"反向映射"、顺序访问在稀疏时反而更值钱、带宽下界那段论证；
- **Hard**：前缀和 + 单块分配的具体实现、`MaskShift` 的 warp 级位位移、dilate 的 27 邻域转置——**不自己写 CUDA 就不必碰**。

## Learning Value

三条可搬走的判断，按可迁移性排序：

1. **"静态不是结构的天性，是维护它的算法的限制。"** 这句话可以套在任何一个"大家都说它只能这样"的数据结构上。**先问：限制来自结构本身，还是来自我们维护它的方式？**
2. **"不需要碰的字节，不该出现在带宽账上。"** 2.0 ms vs 2.63 ms 的下界论证，是把"优化"升级成"复杂度不同"的标准动作。
3. **"当一个优化在最坏情况下收益最大，它修的是结构问题。"** VBM 的 1.43× → 1.98× 是判据，不是卖点。

## Visualization

**图 4｜拓扑算子的七步管线（第 7 步未画）。** 绿框 = 算子相关，灰框 = 五个算子通用。这张图真正想说的是：**"重建"这个动作里，只有两步需要知道你在做什么算子，其余五步是同一套机械。**

![[Research/Files/Pictures/Sifakis 2026 — 图4 拓扑算子的七步管线.png]]

**读图**：① 候选 root tile（4096³）先超配；② 每个 tile 一张稠密位图 scratchpad（上 128³ / 下 8³ 两级）；③ 并行标记要存在的节点；④ 前缀和 + 单块分配；⑤ 并行重建连接；⑥ 填 active flag。**注意 ④ 之后树就已经"存在"了，只是没有体素是活跃的**——这个中间态是"先算后建"的关键。出处：Sifakis et al. 2026, Fig. 4。

**图 7｜VBM 用 1.5 bit/体素编码了"反向映射"。** 32 个连续活跃体素跨了 4 个叶节点（橙），VBM 只存 `firstLeafID = 100` 和一个 32 位 `jumpMap`——后者的第 10、19、28 位被置起，标记"这里换叶了"。

![[Research/Files/Pictures/Sifakis 2026 — 图7 VBM 的块元数据.png]]

**读图**：左边彩色数字是"活跃体素的线性序号"，格子是叶内的 8³ 体素。**四个叶节点、一个块、两个元数据项**——这就是"用 1.5 bit 换来"的全部。出处：Sifakis et al. 2026, Fig. 7。

**图 9｜为什么"按活跃体素派线程"在稀疏时更值钱。** 横轴是叶节点平均占用率（从 90% 降到 30%），纵轴是 WENO5 重初始化的吞吐。蓝线（VBM）基本平；橙线（accessor）随稀疏度下降。**加速比从 1.43× 涨到 1.98×。**

![[Research/Files/Pictures/Sifakis 2026 — 图9 WENO 吞吐随叶占用率变化.png]]

**读图**：注意横轴是**递减**的（从 90 到 30）。**越稀疏，差距越大**——因为 accessor 的空转线程越来越多。标记颜色是 dilate 宽度，不是第三个变量。出处：Sifakis et al. 2026, Fig. 9。

**图 11｜四个面板：dilate / prune 的时间与显存（双对数）。** 绿方块（本篇原语）在三组曲线里都贴着底边；蓝圆（重建路线）在最大输入处高出两个数量级；灰三角（OpenVDB TBB）在中段就掉队。

![[Research/Files/Pictures/Sifakis 2026 — 图11 dilate 与 prune 的规模曲线.png]]

**读图**：左两图是**每次调用的中位耗时**，右两图是**峰值显存**；横轴是输入活跃体素数（0.98 M → 567 M）。**对数轴，实际差距比看上去更宽。** 出处：Sifakis et al. 2026, Fig. 11。

## Notes

- **标题有两版**：arXiv 列表页写 `Dynamic GPU Voxel Grids...`，PDF / ACM 正式版写 `Dynamic GPU Sparse Grids...`。**以 ACM 版为准**，arXiv 版标题作为别名保留。
- 一作 Sifakis（NVIDIA + UW-Madison），末位 Museth（NanoVDB 作者、OpenVDB 作者）——**"原作者 + 新团队"的组合**，这也是为什么这篇能把 2013 与 2021 两次取舍讲得那么清楚（Limitations 里"我们还没有实现 erosion"这种话，一般是原作者才敢写）。
- OpenVDB 仓库已合并，fVDB 侧示例在 `openvdb/fvdb-core`。**代码可用性是 ASF 级别的，不是"即将开源"。**
- 一句话记住这篇的位置：**它是 OpenVDB 2013 那篇标题里"with Dynamic Topology"在 GPU 上的补完。**
