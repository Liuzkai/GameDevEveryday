---
title: Things 主题预览检查
---

# H1 标题 — Things 主题视觉检查

这是一段正文，用于检查 **加粗**、*斜体*、~~删除线~~、`行内代码`、[链接](https://github.com/colineckert/obsidian-things)
以及 <mark>高亮</mark> 在预览里的效果。正文行高 1.5，字体跟随系统字体栈。

## H2 标题（带下划线）

Things 的 H2 有一条 2px 下划线，颜色取自主题的 `--background-modifier-border`。

### H3 标题（蓝色）

### 检查清单

- [x] 已完成任务
- [ ] 未完成任务
- 普通列表项
  - 二级列表项

### H4（黄色）

#### 四级标题在 Things 里是黄色、字重 500。

### H5（红色）

##### 五级标题是红色。

### H6（弱化）

###### 六级标题用 text-muted 颜色。

> 引用块：Things 里是斜体 + 青绿色（#3eb4bf），左侧 2px 竖线。

---

### 表格

| 元素 | Things 取值 | 说明 |
| --- | --- | --- |
| 正文色 | `#222222` | `--color-base-100` |
| H3 | `#2e80f2` | `--blue` |
| 引用 | `#3eb4bf` | `--quote-color` |
| 加粗 | `#ff82b2` | `--strong-color` |

### 代码块

```js
// 语法配色使用 Things 内置的 One Dark 色板
function things(name) {
  const colors = ["#e06c75", "#98c379", "#61afef"];
  return `${name}: ${colors.length}`;
}
```

```css
/* 行内代码与代码块背景在 Things 里是两种颜色 */
code { background-color: #eceef1; }
pre  { background-color: #f0f2f5; }
```

### 脚注与折叠

带脚注的句子[^1]。

<details>
<summary>折叠块（details / summary）</summary>

折叠内容也用主题的边框色和圆角。

</details>

[^1]: 脚注区域使用 text-muted 颜色，并有一条顶部分隔线。
