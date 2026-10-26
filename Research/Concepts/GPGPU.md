---
type: concept
user_level: Normal
aliases: [GPGPU, General-Purpose Computing on Graphics Processing Units, 通用 GPU 计算, GPU 通用计算, GPU Compute]
prerequisites: [GPU Architecture, 纹理与光栅化的基本语义, 数据并行（SIMD）直觉]
first_introduced: "2002 前后（GPGPU 社区成形）；2004（[[Harris — Fast Fluid Dynamics Simulation on the GPU (2004)|Harris 2004]] 给出完整翻译手册）；2009（compute shader 使其正规化）"
---

# GPGPU

## Definition

**用图形处理器做非图形计算**——但本页关心的不是"GPU 能算通用问题"这个结论，而是**它的编程模型长什么样、代价长什么样**。

分水岭在 2009 年：在此之前，通用计算必须**伪装成绘制**（借顶点/片元管线）；在此之后，compute shader 给了它**自己的执行环境**。两代的**成本形状没有变**——都是"**一遍 pass 一遍全屏**"——变的是**能力**（能不能任意写、能不能组内通信）。

> 一句话：**GPGPU 的历史，是"把 GPU 缺的三样能力一样样补上"的历史**；补完之后，剩下的约束只有带宽和占用率。

## Core Principle

### 三条锁（2004 年的借法，也是理解今天的钥匙）

| # | 锁 | 具体表现 | 后果 |
|---|---|---|---|
| ① | **写位置由光栅化钉死** | 片元程序只能写"自己被光栅化到的那个像素" | **只能 gather，不能 scatter**——算法必须改写成"我该读谁" |
| ② | **一张图不能同时读和写** | 输出只能进 framebuffer，再经 RTT/CTT 变回纹理 | **必须双缓冲（ping-pong）**：显存 ×2，交换近乎免费 |
| ③ | **寄存器与精度受限** | 片元程序寄存器少；fp16 渲染目标普及而 fp32 又慢又常不支持 | **必须打包**：4 个标量塞进 RGBA；长迭代累积误差 |

**第 ① 条是最深的一条**：它不是性能问题，是**表达能力**问题。凡是"一个元素要写到任意另一个位置"的算法——粒子散射、直方图、排序、图遍历——在 2004 年**根本无法表达**，只能换成排序网络、点精灵、多 pass 归约之类的绕法。这就是为什么**粒子长期留在 CPU**，而**场（网格）类模拟先上 GPU**：场的通信只发生在邻居，天然是 gather。

### 一个成本公式（本页的实用部分）

$$\text{成本} \;\approx\; \underbrace{W\times H}_{\text{一遍全屏}} \times \underbrace{N_{\text{pass}}}_{\text{每帧 pass 数}} \times \underbrace{B_{\text{读写}}}_{\text{每像素字节}}$$

三条推论，每条都能直接用在评审上：

1. **pass 数是第一乘数，且通常被忽略**。一个流体步 = 5 + 40~80 次 Jacobi ≈ **65 个 pass**；说"我们跑 256² 的模拟"等于没说，**要说"256² × 65 pass"**；
2. **它是带宽账，不是算力账**。fp16 RGBA 每像素读 5 邻居写 1 = 48 B；$256^2\times65\times48\text{ B}\approx204$ MB/帧，60 fps 即 **12 GB/s**——先撞带宽墙（详见 [[Harris — Fast Fluid Dynamics Simulation on the GPU (2004)|Harris 2004]] 的估算）；
3. **降档的三个旋钮，省的钱与丢的画质不成比例**：**降迭代**省得是**线性**的（60→30 省一半），画质损失是**渐进的**（不到阈值看不出来）；**降分辨率**省得是**平方**的（256→128 省 3/4），但**结构直接变了**（细节消失，不能靠后处理补回）；**降精度**省一半带宽，却可能把长迭代推向不稳定。**所以顺序是：先降迭代 → 再降分辨率 → 精度最后动**——省钱最多的不是最该先动的。

### 两代写法对照

| 需求 | 2004 的借法（片元管线） | 2009 之后（compute） |
|---|---|---|
| 读邻居 | `tex2D` 采样（自带双线性） | `groupshared` / 纹理读取 |
| 写任意位置 | **做不到**（只能写自己） | `RWTexture` 任意写 / 原子操作 |
| 迭代反馈 | ping-pong 两张纹理 | 双缓冲 + 屏障（同一件事） |
| 边界 | 内部 quad + 边界线段，两个片元程序 | `if` 分支 |
| 组内共享 | **不存在**（片元间不能通信） | `groupshared` + `GroupMemoryBarrier` |
| 一次 dispatch 的粒度 | 一次 draw call（一个全屏 quad） | 一个线程组网格 |

**关键差别只有两条**：**能不能 scatter**、**能不能组内通信**。其余全部继承——包括"一遍 pass 一遍全屏"这个成本形状。

## Prerequisites

- [[GPU Architecture]] —— 三类资源（算力/带宽/延迟隐藏）与存储层级：本页的每条成本都落在它上面；
- **光栅化语义**：为什么"像素位置"是硬件给的，不是程序给的（这一条理解不了，第 ① 条锁就理解不了）；
- **数据并行直觉**：同一段代码作用在大量独立元素上——SIMD/SIMT 的差别在此不关键。

## Historical Evolution

```text
2001  GeForce 3 / DX8：第一代可编程顶点+片元着色器（只能画图）
2002  GPGPU 社区成形（课程、邮件列表、gpgpu.org）——"用显卡算东西"成为一门手艺
2003  Bolz 等：稀疏矩阵求解器（CG/多重网格）上 GPU；Krüger & Westermann：线性代数算子
  ↓
2004  ★ [[Harris — Fast Fluid Dynamics Simulation on the GPU (2004)|Harris, GPU Gems 38]]：
      完整时变求解器上 GPU——数组=纹理 / 循环=片元程序 / 反馈=双缓冲 / 边界=图元
2006  DX10：geometry shader + stream out（第一次能"写出去到别处"）；CUDA 公布
2007  CUDA 发布：正规编程模型（scatter / 共享内存 / 原子 / 线程组）
2009  DX11 Compute Shader、OpenCL 1.0（2008 末发布）：GPGPU 从"借"变成"标准件"
2014+ 异步计算队列（主机/计算/拷贝并行）——模拟与渲染开始抢同一块 GPU 的时间片
2018+ 网格/任务着色器、GPU 驱动管线：**调度本身**也变成 GPU 上的数据（[[GPU-Driven Rendering]]）
2020s  神经替代：不求解，学求解（[[Neural Physics Simulation]]）——但推理本身仍是 pass 序列
```

**每一代补的能力**：scatter（2006/2009）→ 组内通信（2009）→ 并发队列（2014）→ 调度数据化（2018）→ "要不要算"（2020s）。**唯一从未被补上的是带宽**。

## Important Papers

- [[Harris — Fast Fluid Dynamics Simulation on the GPU (2004)]] —— **本页的原型文本**：三条锁、pass 成本结构、双缓冲与边界的工程解法全在里面；
- 记名（未单独入库）：Bolz et al. 2003《Sparse Matrix Solvers on the GPU》（CG + 多重网格；证明 GPU 能解 PDE）；Krüger & Westermann 2003《Linear Algebra Operators for GPU Implementation of Numerical Algorithms》；Goodnight et al. 2003（GPU 多重网格）。
- 库内下游样本：[[Niagara]]（GPU 模拟的引擎化形态）、[[Bitterli — Spatiotemporal Reservoir Resampling for Real-Time Ray Tracing with Dynamic Direct Lighting (2020)|ReSTIR]]（把"复用"做成 pass 结构）、[[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]（用 binning 把 scatter 改写成 gather 的现代案例）。

## Related Concepts

- [[GPU Architecture]] —— **硬件侧**：本页讲"程序模型"，它讲"资源账"；两页合起来才解释得清"为什么 pass 贵"；
- [[Overdraw]] —— **同一个乘数的另一面**：每个 pass 都是一次全屏写入，所以 GPU 模拟的 overdraw 是**结构性**的（by construction），不像美术叠层那样可谈；
- [[Particle Systems]] —— 对照侧：粒子是 scatter 型负载，**晚十年才上 GPU**；理解这条时间差就理解了第 ① 条锁；
- [[GPU-Driven Rendering]] —— 后代：把"剔除/调度"也变成 GPU 上的数据流（indirect draw）；
- [[Niagara]] —— 引擎里的落点：Simulation Stage = pass，Grid2D/3D = 纹理场；
- [[Tile-Based Rendering]] —— 移动端分支：TBDR 把"一遍全屏"的成本挪到 tile 内存与 flush，**同一个 pass 在两种架构上是两种价钱**。

## Technologies

- **Compute Shader（DX11 / Vulkan / Metal）**：现代默认执行环境；
- **CUDA / OpenCL / HIP**：厂商或跨平台模型；
- **Niagara GPU 模拟 / VFX Graph**：引擎化封装（Simulation Stage 迭代次数 = pass 数）；
- **Transform Feedback / Stream Out**：2006 年的"半个 scatter"，粒子线的过渡形态；
- **Async Compute**：把模拟塞进图形队列的空隙——**预算从"占多少时间"变成"能不能塞进去"**。

## Game Applications

- **GPU 粒子/流体/布料**：pass 数 × 分辨率的账（见上公式）；
- **GPU 剔除与 LOD 选择**：把"每个物体的判断"变成一遍全屏/一遍间接参数的写；
- **后处理链**：本质上就是 GPGPU 的最简形态——**每个 pass 一遍全屏**，所以"后处理很多"和"分辨率很高"是同一笔账；
- **判断一项技术该不该做**：问三句——① 元素之间要不要任意写？② 要不要组内通信？③ 每帧几个 pass？**三问全过才是"天然适合 GPU"。**

## Personal Knowledge

Current Level: **Normal**

**你已经在用的部分**：Niagara GPU 粒子、模拟 stage、带宽直觉——这套语言是日常工具。

**这一页要补的**：把"**pass 数**"变成你评审时的口头指标。你知道同屏粒子数，但**说不出一个 GPU 模拟跑几个 pass**——这就是预算表里缺的那一栏。

## Learning Gap

- **组内通信的实战**：`groupshared` 的 bank conflict、屏障语义——按需再补，不必现在；
- **异步计算的排布**：模拟与渲染抢同一块 GPU 时怎么切时间片（这是主机侧经验，非论文能教）；
- **移动端 TBDR 下的 GPGPU**：tile 内存让"多 pass"更贵，"一遍多渲染目标（MRT）"更划算——与 PC 的直觉相反。

## Next Step

1. （5 分钟，零成本）打开任一条你手上的 Niagara GPU 模拟，**数一遍它的 Simulation Stage 迭代次数与 Grid 分辨率**，代进公式算一遍 `W×H×pass×48 B`；
2. （20 分钟）读 [[Harris — Fast Fluid Dynamics Simulation on the GPU (2004)]] 的 38.3.1 与 38.3.3——四张类比表 + 八行 advect 代码；
3. （观察项）下次有人提"我们上 GPU 模拟"，先问 **pass 数**，再问分辨率——**顺序反了就会得到错误的结论**。
