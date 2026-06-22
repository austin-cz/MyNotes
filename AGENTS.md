## 仓库性质

这是一个 **Obsidian 笔记库**，不是软件项目。没有构建、测试、lint 命令。

## 目录结构

- `opencode/` — OpenCode 使用笔记
- `.obsidian/` — Obsidian 配置（不要手动修改）
- `.opencode/` — OpenCode 插件和 skills（不要手动修改）
- `tmp_files/` — 临时文件（已 gitignore）

## Obsidian 功能

- 启用了 Canvas、Bases、Daily Notes、Templates、Backlink、Properties
- 安装了社区插件：Terminal (v3.26.0)
- 文件格式：`.md`（Obsidian Markdown）、`.canvas`（JSON Canvas）、`.base`（Bases）
- 创建/编辑笔记时使用对应 skill：
  - `.md` → obsidian-markdown skill
  - `.canvas` → json-canvas skill
  - `.base` → obsidian-bases skill
  - 与 Obsidian 交互 → obsidian-cli skill

## 注意事项

- 涉及 `~/.config` 和 `~/.local` 目录的文件操作需用户同意
