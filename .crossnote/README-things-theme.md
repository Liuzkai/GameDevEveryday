# Obsidian "Things" 主题 → VS Code Markdown 预览

把 `E:\BaiduSyncdisk\ObsidianNotes\刘中凯笔记\.obsidian\themes\Things`（Things v2.2.4，作者 @colineckert）
的样式移植到本工作区的两种 Markdown 预览里。

## 文件

| 文件 | 作用 |
| --- | --- |
| `.crossnote/things-markdown.css` | **主题本体**，唯一的样式来源 |
| `.crossnote/style.less` | Markdown Preview Enhanced 的入口，`@import (inline)` 主题本体 |
| `.vscode/settings.json` | 内置预览的 `markdown.styles` 指向主题本体；并把 MPE 的基础主题设成 `none.css` |
| `.crossnote/things-theme-check.md` | 视觉检查页：标题 / 列表 / 引用 / 表格 / 代码 / 脚注 |

## 生效方式

1. **VS Code 内置预览**（Ctrl+Shift+V、Ctrl+K V，或你把 `*.md` 关联到
   `vscode.markdown.preview.editor` 之后打开的编辑器）：靠工作区设置
   `"markdown.styles": [".crossnote/things-markdown.css"]` 加载。
2. **Markdown Preview Enhanced**（Ctrl+Shift+V 由该扩展接管时）：
   自动读取工作区 `.crossnote/style.less`，其中内联导入同一份主题。
   同时把 `previewTheme` 设为 `none.css`，避免 `github-light` 的
   标题/引用/代码块配色压在 Things 之上。

改完 CSS 后：
- 内置预览：命令面板 → `Markdown: Refresh Preview`（或重新打开预览）。
- MPE：保存一次 Markdown 文件即会重新渲染。

## 移植内容对照

| Things（Obsidian） | 本移植 |
| --- | --- |
| `--color-base-00` / `--color-base-100` 明暗底色 | `--things-bg` / `--things-text-normal`，通过 `body.vscode-dark`、`[data-vscode-theme-kind="vscode-dark"]`、`body.theme-dark` 切深色 |
| 标题 1.7/1.5/1.2/1.1/1/0.9，H2 下划线，H3 蓝、H4 黄、H5 红 | `h1`–`h6` 一一对应 |
| `--strong-color` / `--em-color` 粉色、`--quote-color` 青绿 | `strong` / `em` / `blockquote` |
| 行内代码 `#eceef1` 与代码块 `#f0f2f5`（深色 `#303540` / `#282c34`） | `code:not(pre code)` 与 `pre` |
| One Dark 语法色（`.token.*`，Prism） | 同时覆盖 `.token.*` 与 `.hljs-*`（内置预览用 highlight.js） |
| 图片圆角 + 双层阴影 | `img` |
| Style Settings 里的可选开关（fancy-code、fancy-highlight、h2-no-underline…） | 文件末尾第 11 节，注释状态，按需取消注释 |

## 注意

- `markdown.styles` 属于 VS Code 的「受限配置」：工作区处于**受限模式**（Restricted Mode）时，
  工作区 `.vscode/settings.json` 里的这一项会被忽略，内置预览不会套用主题。
  信任该工作区，或把同一行加进用户设置即可。
- 主题用 `html:root body` 前缀抬高选择器优先级，因此不需要到处写 `!important`
  （只有 `pre` 的背景色是例外：部分自带 prism 主题本身就写了 `!important`）。
- 想恢复原样：删掉 `.crossnote/things-markdown.css`、`.crossnote/style.less`，
  以及 `.vscode/settings.json` 里的 `markdown.styles` 等四项。
