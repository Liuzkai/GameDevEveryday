---
type: explainer
title: "GS 图解 2 — 排序瓶颈、管线冲突与密度控制"
parent: "[[Gaussian Splatting]]"
created: 2026-09-11
tags: [gaussian-splatting, rasterization, deferred-rendering, tbdr]
---

# GS 图解 2 — 排序瓶颈、管线冲突与密度控制

> [[Learning Path — Gaussian Splatting]] 三个 Gap 的闭环笔记。出发点只有一个：**你懂半透明粒子排序，就懂 3DGS 一半的工程难点**。

## Gap 1 · 排序瓶颈（机制）：blending 不满足交换律

![[GS_半透明顺序依赖.html]]

- 先红后蓝 ≠ 先蓝后红 —— alpha blending **天生依赖绘制顺序**
- 不透明物体有 z-buffer「只留最近」，任何顺序画都对；半透明没有这种捷径，**必须先排好再画**

## Gap 1 · 排序瓶颈（规模）：三个乘数

![[GS_排序成本三乘数_TileGS.html]]

| 乘数 | 内容 | 预算语言对应 |
|---|---|---|
| 数量 ×300 | S 级爆发技 ~3,000 粒子 → 一个 GS 场景 1,000,000+ 高斯 | Particles |
| 每像素 ×几十层 | 椭圆 footprint 大，一个像素被几十个 2D 椭圆轮流盖住 | OverDraw |
| 顺序锁死 | 不能乱序画，early-z 全部失效；每帧百万级全局排序 | CPUTime / GPUTime |

**TileGS 的拆法**：全局大排序 → 切成 16×16 的小 tile，各自分桶、各自排序——百万级一锅端变成几千个小任务并行，正是 TBDR 擅长的形状。你的 Android 三档全是 TBDR，"排序/混合"类开销在 TBDR 上比 PC 更贵 → [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]]

## Gap 2 · 为什么进不了 deferred

![[GS_与deferred管线冲突.html]]

- deferred 的全部效率来自一个假设：**每像素只有 1 个表面**（G-buffer 存 depth / normal / albedo 各一份）
- GS 每个像素是**几十层雾的叠色** —— 假设直接被违反，G-buffer 无处存放
- 关键认知：半透明**从来**进不了 deferred（项目里透明物件也是最后单独走 forward pass）。GS 不是"独特地坏"，它是**全场 100% 半透明**

## Gap 3 · 自适应密度控制 = 训练时的发射器管理

| GS 操作 | 触发条件 | VFX 对应 |
|---|---|---|
| 分裂 split | 椭球太大、盖不住细节，拆成两个小的 | 大发射器拆小 |
| 克隆 clone | 区域欠重建、密度不够，复制一个 | 补 emitter 加密 |
| 剪枝 prune | α 低到几乎看不见，删掉 | kill 无贡献粒子 |
| 定期重置 α | 防雾团抱团挡视线，强制重新竞争 | （训练独有，无对应） |

## 闭环自测（学习路径 Practical Exercise）

1. **粒子换成高斯椭球做体积感，排序成本怎么变？**
   数量 ×300、每个 footprint 覆盖几十像素、顺序锁死 early-z 失效 → 每像素几十层有序 blend，Overdraw 爆炸。
2. **这个特效为什么进不了 deferred？GS 版本同理吗？**
   同一个理由：G-buffer 每像素只存 1 个表面，半透明叠色依赖顺序——该特效和全部 GS 都只能走末尾的 forward 半透明 pass。
3. **训练时的密度控制对应工作里的什么操作？**
   发射器粒子数的运行时调整：split = 拆小、clone = 加密、prune = kill。

---

相关：[[Gaussian Splatting]] · [[Learning Path — Gaussian Splatting]] · [[TileGS — Tile-Local Depth Binning for Gaussian Splatting Rasterization]] · [[Tile-Based Rendering]] · [[GS 图解 1 — 协方差与椭球：高斯的形状说明书]]
