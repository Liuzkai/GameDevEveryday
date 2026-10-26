---
type: concept
title: "Overdraw"
user_level: Easy
tags: [rendering, performance, vfx, mobile]
---

# Overdraw

> **这是你的五维实测指标之一，但库里一直没有它自己的页。** 本页不是"什么是 overdraw"的科普，而是**成本记账口径**：它有多少种算法、每种算法下单价多少、以及为什么你那张"同屏粒子数"的预算表缺一个共变量。

## Definition

同一帧内，**同一个屏幕像素被着色 / 混合的次数**。

工程上有两种口径，**数值可以差一倍以上**，混用就会得出互相矛盾的结论：

| 口径 | 定义 | 用来回答 |
|---|---|---|
| **Coverage overdraw**（覆盖过绘制） | 落在该像素上的**片元总数** ÷ 像素数 | "画了多少东西"——美术/内容侧的密度 |
| **Shaded overdraw**（着色过绘制） | 该像素上**真正跑完片元着色器**的片元数 ÷ 像素数 | "算了多少东西"——性能侧的真实成本 |
| **Blended overdraw**（混合过绘制） | 该像素上**参与 ROP 混合**的片元数 ÷ 像素数 | "读写多少带宽"——带宽侧的账单 |

三者关系：**Coverage ≥ Shaded ≥ Blended** 视管线而定。不透明物体被 early-Z / HSR 剔掉后，**Coverage 很高但 Shaded ≈ 1**；半透明物体三者基本相等。

> **一句话**：overdraw 不是"画了几层"，是"**哪一层被收钱**"。先问口径，再谈数字。

## Core Principle

$$
\text{填充成本} \;=\; \underbrace{\sum_i A_i}_{\text{屏上总片元数}} \;\times\; \underbrace{c_{\text{shader}}}_{\text{每片元着色}} \;+\; \underbrace{\sum_i A_i \cdot b}_{\text{混合带宽}}
$$

其中 $A_i$ 是第 $i$ 个图元在屏幕上的**像素面积**，$b$ 是每片元的字节数（RGBA8 的读+写 ≈ 8 B）。

把它改写成三个可独立调节的乘数——**这才是预算表该有的形状**：

$$
\text{Overdraw 成本} \;=\; \underbrace{\text{层数}}_{\text{叠了几层}} \;\times\; \underbrace{\text{每层屏占比}}_{\text{每层多大}} \;\times\; \underbrace{\text{单价}}_{\text{每像素多贵}}
$$

- **层数**：同屏存活的粒子数 × 相互重叠程度；半透明粒子**没有遮挡剔除**，层数就是全量；
- **每层屏占比**：sprite 的屏幕面积，**与尺寸平方成正比**（尺寸 ×2 ⇒ 面积 ×4）；与相机距离平方成反比；
- **单价**：shader 复杂度 + 混合模式 + 是否需要额外屏幕采样（distortion / refraction / soft particles）。

## 三个乘数各自的"物理定律"

### ① 面积是平方律

把粒子屏幕尺寸缩到 80%，填充降 **36%**；缩到 70%，降 **51%**。**这是所有降级手段里性价比最高的一条**——它不动数量、不动观感结构，只动"每个粒子看起来多大"。

### ② 屏占比换算表（1080p = 2,073,600 px）

| Sprite 屏幕尺寸 | 单粒子面积 | 占全屏 | 铺满 1× 全屏需要 |
|---|---|---|---|
| 64 × 64 | 4,096 px | 0.20% | 506 个 |
| 128 × 128 | 16,384 px | 0.79% | 127 个 |
| 256 × 256 | 65,536 px | 3.16% | 32 个 |
| 512 × 512 | 262,144 px | 12.6% | 8 个 |

**读法**：**100 个 256² 的 sprite 平铺 ≈ 3.2 倍全屏 overdraw**；同样 100 个 64² 的只有 0.2 倍。**"同屏 100 个粒子"这句话，在两种尺寸下相差 16 倍填充。**

### ③ 带宽换算（PC / IMR 口径）

RGBA8 混合一次 = 读 4 B + 写 4 B = **8 B/片元**。

| 片元数 | 每帧带宽 | @60 fps |
|---|---|---|
| 2.07 M（1× 全屏） | 16.6 MB | 1.0 GB/s |
| 12.3 M（≈ 6× 全屏） | 98 MB | 5.9 GB/s |
| 24.6 M（≈ 12× 全屏） | 197 MB | 11.8 GB/s |

> **TBDR 例外**：移动端 GPU 的混合发生在**片上 tile 内存**，这 8 B 不逐片元打到 DRAM——**所以移动端的 overdraw 账单里"带宽"项被改写成了"tile 内存容量 + 溢出 flush 次数"**。见 [[Tile-Based Rendering]]。

## 单价取决于管线阶段（最容易搞混的一格）

| 图元类型 | 能否省掉着色 | 主要成本 | 备注 |
|---|---|---|---|
| **不透明几何** | ✅ early-Z / HSR | 深度带宽 ≈ 每像素 1 次 | Coverage 再高也基本不烧 ALU |
| **Alpha test / masked** | ⚠️ 部分 | 着色 + 带宽 | 要先跑 shader 才知道丢不丢 ⇒ 早深度失效 |
| **半透明 alpha blend** | ❌ 不能 | 着色 + ROP 读写 + **排序** | 三层全价 |
| **加法混合** | ❌ 不能 | 着色 + ROP 读写，**但免排序** | 交换律买来的便宜（[[Reeves — Particle Systems (1983)]] 法则 ③） |
| **半透明 + 屏幕采样**（distortion / refraction / soft particle） | ❌ 不能 | 以上 **+ 每片元一次纹理/深度采样** | 单价可能翻倍，且带宽不可预测 |

**这张表的用法**：拿到一个特效，先判断它落在哪一行——**同样 10× overdraw，第一行可能免费，第四行可能吃掉半帧**。

## 测量：四种口径，四种用途

| 手段 | 给的是什么 | 适用 |
|---|---|---|
| **解析估算** $\sum A_i / A_{\text{screen}}$ | 期望 overdraw（不需要跑起来） | **最该进预算表的一种**：从粒子数据直接算，不依赖机型 |
| **Niagara 视口的 Overdraw 视图** | 每像素被覆盖次数的伪彩图 | 调特效时看"哪里叠爆了"（以 5.x 编辑器实际视图列表为准） |
| **UE Optimization Viewmodes**（Shader Complexity / Quad Overdraw） | 着色复杂度 / 2×2 quad 利用率 | 定位"贵在哪个材质" |
| **GPU 侧工具**（Arm Performance Studio / Snapdragon Profiler / Xcode Metal / Nsight） | 逐 draw / 逐 tile 的片元与带宽计数 | 出问题后的归因 |

**口径陷阱三条**：

1. **工具报的数一般是 Coverage，不是 Shaded**——不透明部分会被高估；
2. **alpha test / 半透明裁剪**会让 Coverage 与 Shaded 分离，两者都不等于带宽；
3. **不同机型的 early-Z 策略不同**，同一个特效在 PC 与 Android 上的 overdraw 曲线形状可能完全不同——**跨平台对比必须先统一口径**。

## 对你那张 SABC 表的一条修订建议

现行表里有一行是「**同屏粒子数**：S 3000 / A 1200 / B 400 / C 100」。

**这一行单独存在时没有性能含义**——按上面的换算：

| 假设平均 sprite 尺寸 | S 档 3000 粒子的屏占比 |
|---|---|
| 64² | **5.9× 全屏** |
| 128² | **23.7× 全屏** |
| 256² | **94.8× 全屏** |

同一个"3000"，可以是 6 倍屏，也可以是 95 倍屏。**Reeves 1983 的法则 ①（数量随屏占比）说的就是这件事**，而预算表把它丢掉了。

**建议的替代指标**（不改分级、只改度量）：

$$
\text{像素预算} \;=\; \sum_i A_i \;\le\; k \times A_{\text{screen}}, \quad k = \text{档位系数}
$$

- **S / A / B / C 各给一个 $k$**（例如 8× / 3× / 1× / 0.3×），比"粒子数"稳定得多；
- 数量上限仍然保留，但降级为**辅助约束**（它管的是 CPU 侧 spawn/更新与 draw call，不是填充）；
- 好处：**美术可以自己换算**——"我要把粒子从 128² 做到 256²，就得把数量砍到 1/4"，这是一条能当场执行的规则。

## Historical Evolution

```text
1990s   fill rate 是显卡规格表的第一行（"多少 Mpx/s"）
          ↓ 可编程着色后，overdraw 从"像素吞吐"变成"每片元的算法复杂度"
2000s中后期  deferred shading 普及：着色改成每像素一次（G-buffer）
          ↓ 但 G-buffer 的写入本身仍是 overdraw，只是变得"便宜且可预测"
2007+   移动 TBDR 成熟：HSR 让不透明 overdraw 近乎免费
          ↓ 代价转移到 tile 内存容量与 flush
2010s   visibility buffer / GPU-driven：先定可见性，再决定给谁着色
          ↓ 图元本身变成"半透明层"
2023+   3D Gaussian Splatting：每像素几十~几百层 ⇒ overdraw 回到舞台中心
```

**这条线不是"overdraw 越来越贵"，而是"overdraw 的计价单位一直在换"**：像素吞吐 → 每片元算法 → 写入次数 → tile 容量 → **层数**。

## Important Papers

- [[Everitt — Interactive Order-Independent Transparency (2001)]] —— **"层数即成本"的最早清晰表述**：depth peeling 的成本 = `层数 × 一遍场景`；也是"正确顺序必须为最坏层数付费"的证明；
- [[Reeves — Particle Systems (1983)]] —— 法则 ①（数量随屏占比）与法则 ③（加法混合免排序）的源头，本页两条主线的共同祖先；
- [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]] —— 页内分桶：把 overdraw 从"全屏随机"变成"tile 内有序"。

## Related Concepts

- [[GPU Architecture]] —— 单价从哪来（ROP / 带宽 / 早深度）；
- [[Tile-Based Rendering]] —— 移动端 overdraw 的计价方式完全不同；
- [[Particle Systems]] / [[Niagara]] —— 层数的主要来源；
- [[Gaussian Splatting]] —— 每像素层数的极端案例（几十~几百层）；
- [[Scalability and Quality Tiers]] —— 降档时"先动哪个乘数"；
- [[Real-Time VFX Performance Budgeting]] —— 本页服务的预算体系；
- [[Shadow Mapping]] —— 阴影贴图写入也是一种 overdraw（但口径不同：光源视角）；
- [[GPGPU]] —— **结构性 overdraw**：GPU 模拟每个 pass 都是一次全屏写入（`分辨率 × pass 数`），**没有一层是美术叠出来的，全部由算法产生**。这是"层数"的另一类来源，也无法靠美术缩尺寸解决。

## Technologies

- Niagara 的 `Overdraw` 视图与粒子计数（内容侧自查）
- UE Optimization Viewmodes（Shader Complexity / Quad Overdraw）
- 移动端 GPU profiler 的 tile / bandwidth 计数

## Game Applications

- 技能特效的峰值帧降级（先缩尺寸，再减数量，最后简化 shader）
- 移动端三档画质：Android 上的第一约束常是 tile 内存而非 ALU
- 半透明 UI / 光晕 / 屏幕后处理叠加（同样按面积记账）

## Personal Knowledge

Current Level: **Easy** —— 你在日常调优里天天用它。本页补的是**口径与换算**，不是概念本身。

## Next Step

**一次 5 分钟的换算，不需要打开引擎**：拿你手上任意一个 S 档技能，问美术要三个数——**峰值粒子数、平均 sprite 屏幕尺寸、混合模式**，然后算出 $\sum A_i / A_{\text{screen}}$。这个数就是你那条"同屏粒子数"预算的真实含义。**如果它 >10×，说明这条预算目前是"数量合规、填充超支"的状态。**
