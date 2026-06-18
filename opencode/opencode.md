---
title: OpenCode 使用手册
tags:
  - opencode
  - tool
---

# OpenCode 使用手册

## 配置

### 文件位置

| 功能     | 配置文件位置                              |
| ------ | ----------------------------------- |
| 可执行文件 | `~/.opencode/bin/opencode`、`/opt/homebrew/bin/opencode` 或 `/usr/local/bin/opencode` |
| 全局配置 | `~/.config/opencode/opencode.json`  |
| 认证信息 | `~/.local/share/opencode/auth.json` |

### opencode.jsonc 详解

```jsonc
{
  "username": "Allen",
  "$schema": "https://opencode.ai/config.json",
  "shell": "powershell",
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"],
  // `true`：自动下载更新（默认）,`false`：不检查更新,`"notify"`：通知有新版本，但不自动下载
  "autoupdate": "notify",
  // 只启用你的内部 provider
  "enabled_providers": ["open_provider"],
  "model": "open_provider/glm-5.1",
  "provider": {
    "open_provider": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "open_provider",
      "options": {
        "baseURL": "http://192.168.1.1:2222/aigateway"
      },
      "models": {
        "open_provider/glm-5.1": { "name": "open_provider/glm-5.1" },
        "open_provider/minimax-m3": {
          "name": "open_provider/minimax-m3",
          "modalities": {
            "input": ["text", "image"],
            "output": ["text"]
          }
        },
        "open_provider/qwen3.6-plus": {
          "name": "open_provider/qwen3.6-plus",
          "modalities": {
            "input": ["text", "image"],
            "output": ["text"]
          }
        },
        "open_provider/qwen3.7-max": { "name": "open_provider/qwen3.7-max" }
      }
    }
  },
  //"permission": {
  //  "edit": "ask",
  //},
  // 指定额外的指令文件（与 AGENTS.md 合并）
  "instructions": [
    "workflow.md",
    "code.md"
  ],
  // 控制上下文压缩行为,自动压缩
  "compaction": {
    "auto": true
  }
}
```

### 认证

OpenCode 按以下顺序查找认证：**环境变量**（如 `ANTHROPIC_API_KEY`、`OPENAI_API_KEY`）→ **auth.json**（`~/.local/share/opencode/auth.json`）→ **配置文件**中的认证信息。

OpenCode 启动时会自动扫描系统环境变量，发现可用的 API Key 后自动启用对应的模型提供商。只要设了环境变量，不用在配置文件里写任何东西。

`~/.local/share/opencode/auth.json` 格式：

```json
{
  "zhanlu": {
    "type": "api",
    "key": "sdlfja-fj978sdfnlsoug-woue-22123"
  }
}
```

凭证管理：

```bash
opencode auth list     # 查看已存储的凭证
opencode auth logout   # 删除凭证
```

输出示例：

```
Credentials ~/.local/share/opencode/auth.json
┌
●  Zhipu AI Coding Plan  api
│
●  Google  oauth
│
●  OpenAI  oauth
│
└  3 credentials
```

- **`api`**：通过输入 `sk-...` 密钥添加
- **`oauth`**：通过网页跳转登录授权（如 Google, OpenAI）

> [!warning] 安全提示
> auth.json 以**明文 JSON 格式**存储 Key，请务必保护好你的电脑和该文件，不要将其上传到公开的代码仓库。

## Agent

### 内置 Agent

| Agent   | 类型       | 擅长                  | 默认权限                                      |
| ------- | -------- | ------------------- | ----------------------------------------- |
| Build   | Primary  | 全能开发（默认主 Agent）     | 全能（可读写文件、执行命令）                            |
| Plan    | Primary  | 分析代码、规划方案、审查建议      | 受限（禁止编辑源代码，仅 `.opencode/plans/*.md` 允许写入） |
| Explore | Subagent | 快速找到文件、搜索代码、回答代码库问题 | 只读                                        |
| General | Subagent | 复杂研究、多步骤任务          | 多任务执行（可用 Todo 工具）                         |

> [!note] 内部 Agent
> OpenCode 还有 3 个内部 Agent，自动在后台工作，无需手动调用：
> - **compaction**：上下文压缩（对话接近限制时自动触发）
> - **title**：会话标题生成（创建新会话后自动执行）
> - **summary**：会话摘要生成（压缩会话时生成摘要）

**类型说明**：
- **Primary Agent**：你直接对话的主助手，用 <kbd>Tab</kbd> 切换
- **Subagent**：由 Primary Agent 自动调用或你手动用 `@agent名` 调用的专家助手

### Plan 与 Build

Plan Agent 使用权限隔离保护代码：

| 权限 | Plan Agent | Build Agent |
|------|-----------|-------------|
| `edit`（写/改文件） | deny（仅允许计划文件） | allow |
| `bash`（执行命令） | allow | allow |
| `read`、`grep`、`glob` 等 | allow | allow |

模式选择：**不确定 → 先用 Plan；确定了 → 直接 Build**

| 你的需求 | 推荐模式 |
|---------|---------|
| 写新功能 / 修简单 Bug / 快速原型 | Build |
| 学习新代码库 / 代码审查 | Plan |
| 重构核心模块 / 团队协作任务 | 先 Plan 后 Build |
| 不确定改动影响 | Plan |

Plan 模式下生成的计划文件保存位置：

| 级别 | 路径 | 条件 |
|------|------|------|
| 项目级 | `.opencode/plans/<时间戳>-<slug>.md` | 项目有 Git |
| 全局级 | `~/.local/share/opencode/plans/<时间戳>-<slug>.md` | 项目无版本控制 |

> [!warning] 实验性功能
> `plan_enter` / `plan_exit` 工具目前是实验性功能，需设置 `OPENCODE_EXPERIMENTAL=true` 或 `OPENCODE_EXPERIMENTAL_PLAN_MODE=true` 才能使用。启用后 AI 可主动调用工具在 Plan/Build 模式间切换（会弹出确认框）。

### 调用与导航

- <kbd>Tab</kbd> 切换 Primary Agent（Build ↔ Plan），<kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>A</kbd> 查看所有 Agent
- `@explore 任务描述` 调用 Explore Agent（支持 quick / medium / very thorough 三种深度，用"彻底""非常全面"等描述自动采用深度模式）
- `@general 任务描述` 调用 General Agent
- 主 Agent 会根据任务描述自动判断是否需要调用子 Agent

调用子代理后会创建子会话，用以下快捷键导航：

| 快捷键 | 功能 |
|--------|------|
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>→</kbd> | 进入子会话 |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>←</kbd> | 返回父会话 |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>↑</kbd> | 跳转至最顶层父会话 |

> [!tip] 任务进度跟踪
> 复杂任务时，告诉 AI "用 TODO 跟踪进度"，它会自动创建任务列表并逐步更新状态。适合 3 步以上的任务或中途离开后需要了解进度的场景。

### Agent 配置

在 `opencode.jsonc` 中可自定义 Agent 行为：

```jsonc
{
  "agent": {
    "build": {
      "mode": "primary",
      "model": "anthropic/claude-opus-4-5-thinking",
      "temperature": 0.3,
      "permission": {
        "edit": "allow",
        "bash": "allow"
      }
    },
    "plan": {
      "mode": "primary",
      "model": "anthropic/claude-opus-4-5-thinking",
      "temperature": 0.1,
      "permission": {
        "edit": {
          "*": "deny",
          ".opencode/plans/*.md": "allow"
        },
        "bash": "allow"
      },
      "steps": 5
    }
  }
}
```

| 配置项 | 说明 |
|--------|------|
| `model` | 使用的模型，格式 `provider/model-id` |
| `temperature` | 控制随机性（0-1），值越低越专注。Plan 常用 0.1，Build 常用 0.3 |
| `permission.edit` | `"allow"` / `"deny"` / 路径规则如 `{ "*": "deny", ".opencode/plans/*.md": "allow" }` |
| `permission.bash` | `"allow"` / `"deny"` |
| `steps` | 最多工具调用次数，避免过度操作 |

自定义 Agent 切换快捷键：

```json
{
  "keybinds": {
    "agent_cycle": "tab",
    "agent_cycle_reverse": "shift+tab"
  }
}
```

## 会话

### 数据存储与生命周期

OpenCode 把会话数据存储在本地文件系统中，按 JSON 文件组织：

```
~/.local/share/opencode/storage/
├── session/           # 会话信息
│   └── <project-id>/
│       └── <session-id>.json
├── message/           # 消息记录
│   └── <session-id>/
│       └── <message-id>.json
└── part/              # 消息片段（文本、工具调用等）
    └── <message-id>/
        └── <part-id>.json
```

`~/.local/share/opencode/` 是 XDG 标准的数据目录。macOS 和 Linux 都遵循这个规范。会话按项目隔离，不同项目的会话互不干扰。

一个会话从创建到结束经历：**创建 → 活跃对话 → 压缩（上下文太长时自动触发）→ 归档/删除**。大多数时候不需要关心，OpenCode 会自动处理。

### 会话命令

| 命令                | 作用                                                                                                             |
| ----------------- | -------------------------------------------------------------------------------------------------------------- |
| `/new`            | 新建会话                                                                                                           |
| `/sessions`       | 查看并切换会话                                                                                                        |
| `/undo`           | 撤销上一步操作。撤销**文件操作**需要项目是 **Git 仓库**。Git 项目文件会被恢复，非 Git 项目文件不会变化。如果当前目录不是 Git 仓库，先执行 `git init` |
| `/redo`           | 重做被撤销的操作                                                                                                       |
| `/compact`        | 压缩上下文                                                                                                          |
| `/export`         | 导出对话记录                                                                                                         |
| `/share`          | 分享会话（生成链接）                                                                                                     |
| `opencode import` | 从文件或 URL 导入会话（CLI 命令）                                                                                          |
| `opencode export` | 导出会话为 JSON（CLI 命令）                                                                                             |

> [!tip] 导出格式区别
> TUI 里的 `/export` 导出的是 Markdown 格式，适合阅读。但如果你想备份完整数据、或者在另一台机器上恢复会话，需要用 CLI 命令导出 JSON 格式。

### 导入导出与 Fork

**导出**：

```bash
# 交互式选择会话导出
opencode export

# 指定会话 ID 导出，重定向到文件
opencode export session_abc123 > backup.json
```

不指定会话 ID 时，会弹出列表让你选：

```
Export session
◇ Select session to export
│   Fix authentication bug • 2026-02-14 10:30 • a1b2c3d4
│   Add new API endpoint • 2026-02-13 15:20 • e5f6g7h8
└   Update documentation • 2026-02-12 09:00 • i9j0k1l2
```

**导入**：

```bash
# 从本地文件导入
opencode import backup.json
```

导入后，用 `/sessions` 就能看到恢复的会话。

**Fork**：有时候你想从对话的某个节点"分叉"出去，尝试不同的方向，但又不想丢掉原来的对话。Fork 会复制当前会话的所有历史消息，创建一个新会话。新会话标题加上 `(fork #1)` 后缀，再次 Fork 递增为 `(fork #2)`、`(fork #3)`……

Fork 没有默认快捷键，可以在 `opencode.json` 中绑定：

```json
{
  "keybinds": {
    "session_fork": "<leader>f"
  }
}
```

配置后，按 <kbd>Ctrl</kbd>+<kbd>X</kbd> <kbd>f</kbd> 就能 Fork 当前会话。

## 快捷键

### Leader 键机制

OpenCode 使用 **Leader 键**（默认 <kbd>Ctrl</kbd>+<kbd>X</kbd>）作为快捷键前缀，避免与终端快捷键冲突。

**操作方式（三步法）**：

```
┌─────────────────────────────────────────────────────────────┐
│  第 1 步          第 2 步          第 3 步                   │
│  按下 Ctrl+X  →   松开所有键   →   按下字母键（如 N）        │
│                                                             │
│  ⚠️ 关键：第 2 步必须松开！不是同时按住三个键！              │
└─────────────────────────────────────────────────────────────┘
```

| 你想做什么  | 完整操作                | 错误操作                 |
| ------ | ------------------- | -------------------- |
| 新建会话   | 按 Ctrl+X → 松开 → 按 N | ❌ 同时按 Ctrl+X+N       |
| 打开会话列表 | 按 Ctrl+X → 松开 → 按 L | ❌ 按住 Ctrl 不放按 X 再按 L |
| 切换模型   | 按 Ctrl+X → 松开 → 按 M | ❌ 按太快没松开             |

### 操作速查表

| 快捷键 | 功能 | 说明 |
| --- | --- | --- |
| <kbd>Enter</kbd> | 发送消息 | 回车发送 |
| <kbd>Shift</kbd>+<kbd>Enter</kbd> | 换行（不发送） | 写多行提示词时用 |
| <kbd>Ctrl</kbd>+<kbd>C</kbd> | 清空输入 / 关闭对话框 / 退出 |  |
| <kbd>Escape</kbd> | 中断 AI 响应 | 按两次可强制中断 |
| <kbd>↑</kbd> / <kbd>↓</kbd> | 翻阅历史输入 | 输入框为空时生效 |
| <kbd>Tab</kbd> | 切换 Agent | 在 Plan/Build 间切换 |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>N</kbd> | 新建会话 | **N**ew |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>L</kbd> | 会话列表 | **L**ist |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>M</kbd> | 模型列表 | **M**odel |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>A</kbd> | Agent 列表 | **A**gent |
| <kbd>F2</kbd> | 快速切换最近模型 | IDE 通用 |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>U</kbd> | 撤销消息 | **U**ndo |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>R</kbd> | 重做消息 | **R**edo |
| <kbd>Ctrl</kbd>+<kbd>P</kbd> | 命令面板 | 同 VS Code |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>Y</kbd> | 复制消息 | 复制 AI 回复 |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>C</kbd> | 压缩上下文 | 对话太长时 |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>B</kbd> | 切换侧边栏 | 看会话树 |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>→</kbd> | 进入子会话 | Agent 导航 |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>←</kbd> | 返回父会话 | Agent 导航 |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>↑</kbd> | 跳转至最顶层父会话 | Agent 导航 |
| <kbd>PageUp</kbd> / <kbd>PageDown</kbd> | 翻页浏览 | 长对话翻阅 |
| <kbd>Ctrl</kbd>+<kbd>X</kbd> → <kbd>T</kbd> | 主题列表 | 换个心情 |
| <kbd>Ctrl</kbd>+<kbd>A</kbd> | 跳到行首 | Readline 风格 |
| <kbd>Ctrl</kbd>+<kbd>E</kbd> | 跳到行尾 | Readline 风格 |
| <kbd>Ctrl</kbd>+<kbd>K</kbd> | 删除光标到行尾 | Readline 风格 |
| <kbd>Ctrl</kbd>+<kbd>U</kbd> | 删除光标到行首 | Readline 风格 |
| <kbd>Ctrl</kbd>+<kbd>W</kbd> | 删除上一个单词 | Readline 风格 |
| <kbd>Alt</kbd>+<kbd>B</kbd> | 后退一个单词 | Readline 风格 |
| <kbd>Alt</kbd>+<kbd>F</kbd> | 前进一个单词 | Readline 风格 |

## 规则与指令

### 作用域与加载

OpenCode 支持三种作用域的规则：

| 作用域      | 位置                                  | 适用场景      |
| -------- | ----------------------------------- | --------- |
| **全局规则** | `~/.config/opencode/AGENTS.md`      | 所有项目通用的偏好 |
| **项目规则** | 项目根目录 `AGENTS.md`（从当前目录向上遍历到工作目录根） | 项目特定的规范   |
| **配置文件** | `opencode.json` 的 `instructions` 字段 | 引用多个规则文件  |

> [!note] OpenCode 同时支持 `AGENTS.md` 和 `CLAUDE.md`（兼容 Claude Code）。推荐用 `AGENTS.md`，这是 OpenCode 的标准名称。如设置了 `OPENCODE_CONFIG_DIR` 环境变量，也会加载其中的 `AGENTS.md`。

规则按以下顺序加载，后加载的会**补充**（不是覆盖）前面的：

```
1. 全局 ~/.config/opencode/AGENTS.md
2. 全局 ~/.claude/CLAUDE.md（兼容模式）
3. 项目目录从当前向上遍历查找 AGENTS.md / CLAUDE.md
4. OPENCODE_CONFIG_DIR 环境变量指定目录中的 AGENTS.md
5. 配置文件 instructions 指定的文件
```

规则是热加载的，AI 帮你创建文件后，下一条消息就会遵守新规则。不需要新建会话。

### 规则文件引用

**`/init` 自动生成规则**：用 `/init` 让 AI 分析项目自动生成 `AGENTS.md`，也可以带参数：`/init 特别关注 TypeScript 类型安全和错误处理`。

`/init` 会扫描项目结构、检测构建/测试/lint 命令、分析代码风格，自动整合 Cursor 规则（`.cursor/rules/`、`.cursorrules`）和 Copilot 规则（`.github/copilot-instructions.md`）。如果 `AGENTS.md` 已存在会改进而非覆盖。

**`instructions` 配置引用**：规则分散在多个文件时，可在 `opencode.json` 中统一引用：

```jsonc
// opencode.json
{
  "instructions": [
    "CONTRIBUTING.md",
    "docs/coding-standards.md",
    ".cursor/rules/*.md",
    "~/my-rules/common.md"
  ]
}
```

支持的格式：相对路径（从项目目录向工作目录根查找，支持 glob）、绝对路径（`~/my-rules/common.md`）、Glob 模式（`packages/*/AGENTS.md`）、URL（支持 `https://`，5 秒超时）。

**`AGENTS.md` 中引用外部文件**：可以在 `AGENTS.md` 中用 `@docs/xxx.md` 语法指示 AI 按需加载外部文件：

```markdown
## 技术规范

TypeScript 风格：@docs/typescript-guidelines.md
React 组件模式：@docs/react-patterns.md
```

AI 遇到这类引用时会用 Read 工具按需加载，不会预先加载所有文件。

> [!important] 刷新规则
> 通过 `instructions` 引用规则文件的**路径列表**是在会话启动时加载的。新增或修改 `instructions` 条目需要**开启新会话**才能生效。
> 但已加载路径的**文件内容**是热加载的——修改规则文件本身，下一条消息就会生效。

## CLI 命令

### 模型管理

查看可用模型：

```bash
opencode models
```

输出示例：

```
opencode/glm-4.7-free
anthropic/claude-3-5-sonnet-20241022
google/gemini-2.0-flash
ollama/deepseek-r1:7b
zhipuai-coding-plan/glm-4.7
...
```

每一行都是一个**模型 ID**（格式为 `提供商/模型名`），可以直接复制在启动时指定：

```bash
# 比如直接用智谱 GLM-5 启动
opencode --model zhipuai-coding-plan/glm-5
```

列表太长时，可以指定厂商名字过滤：

```bash
# 只看 Anthropic 的模型
opencode models anthropic

# 只看 DeepSeek 的模型（前提是你已配置）
opencode models deepseek
```

查看某个模型的详细定价，加上 `--verbose` 参数：

```bash
opencode models --verbose
```

输出会包含详细的元数据，包括 `inputCost`（输入价格）和 `outputCost`（输出价格）：

```
zhipuai-coding-plan/glm-4.7
{
  "id": "zhipuai-coding-plan/glm-4.7",
  "name": "GLM 4.7",
  "provider": "zhipuai-coding-plan",
  "inputCost": 0,    // 0元！
  "outputCost": 0    // 0元！
}
```

OpenCode 会缓存模型列表。如果你刚在 `opencode.json` 里加了新配置，或者刚申请了新模型的权限，列表可能没及时更新。需要**强制刷新**：

```bash
opencode models --refresh
```

### 费用统计

默认情况下，`opencode stats` 显示的是你所有项目的**总账单**。

如果你在一个 **Git 项目**里，想查看**仅属于该项目**的费用，加上 `--project ""` 参数：

```bash
opencode stats --project ""
```

OpenCode 依靠 Git 仓库的**第一条提交记录**来识别项目。

- **Project（独立账单）**：拥有至少一个 Commit 的 Git 仓库
- **Global（全局账单）**：除此之外的所有情况（普通文件夹、空 Git 仓库）

> [!warning] 注意
> 如果你在普通文件夹下运行 `opencode stats --project ""`，显示的金额是**本机所有 Global 操作的总和**（比如你在 Downloads、Desktop 等所有非 Git 目录的消耗都会算在一起），而不仅仅是当前文件夹的消耗。

**想开启独立记账？** 只需三步（**不需要关联 GitHub，本地仓库即可**）：

1. `git init`
2. `git add .`
3. `git commit -m "init"` 👈 **这一步至关重要！**

> [!tip] 自动过户
> 你可能会担心："我在这个文件夹里已经写了好几天代码，花了不少 Token，但一直没用 Git。现在才初始化仓库，之前的账单是不是就丢了？"
> OpenCode 比你想的更智能：**它会自动过户！** 当你在一个文件夹里成功创建 Git 项目（并提交）后，OpenCode 会自动扫描"全局账本"，把所有**属于这个文件夹路径**的历史会话，统统"过户"到这个新项目名下。

按模型统计：

```bash
opencode stats --models 5   # 显示消耗最高的 5 个模型
opencode stats --models     # 显示所有模型的详细列表
```

按时间筛选（`--days` 参数）：

```bash
opencode stats --days 1             # 只看过去 24 小时
opencode stats --days 7             # 查看最近 7 天
opencode stats --days 1 --models    # 列出过去24小时的模型消耗
```



# 场景应用

## coder
### 开发工作流

```
理解代码(Plan) → 制定方案(Plan) → 实现功能(Build) → 验证测试(Build)
```


### 第 1 步：快速理解代码


**为什么**
接手项目第一步是理解代码结构。

切换到 Plan Agent：

```
@explore 帮我梳理这个项目的整体结构，包括：
1. 主要目录和功能模块
2. 入口文件和核心流程
3. 使用的技术栈和框架
```

深入某个文件：

```
@src/services/auth.ts 这个认证模块的逻辑是什么？列出所有导出函数及其作用
```

### 第 2 步：制定功能方案

<AdInArticle />

**为什么**
写代码前先想清楚方案。

继续在 Plan Agent：

```
我要给这个项目添加一个「邮件通知」功能，当用户注册成功后发送欢迎邮件。
请帮我分析：
1. 需要修改哪些文件
2. 推荐的实现方案（2-3 种）
3. 每种方案的优缺点
4. 推荐哪种方案，为什么
```

### 第 3 步：分步实现功能

**为什么**
复杂功能拆成小步，降低出错风险。

切换到 Build Agent：

```
按照方案一实现邮件通知功能：

第一步：创建邮件服务模块 src/services/email.ts
- 使用 nodemailer
- 支持 SMTP 配置
- 导出 sendEmail 函数
```

确认第一步完成后：

```
第二步：在用户注册成功后调用邮件服务
@src/controllers/auth.ts 在 register 函数成功后添加发送欢迎邮件的逻辑
```

### 第 4 步：定位 Bug

**为什么**
修 Bug 前先理解问题。

切换到 Plan Agent：

```
用户反馈「登录后页面一直 loading」，请帮我分析：
1. 可能的原因有哪些
2. 如何排查（给出具体步骤）
3. 最可能的问题在哪个文件
```

### 第 5 步：修复 Bug

**为什么**
定位清楚后再动手修复。

切换到 Build Agent：

```
@src/hooks/useAuth.ts 问题定位到这里：
- 登录成功后 isLoading 没有重置为 false
- 请修复这个问题
```


### 重构安全三步法

```
1. 先有测试 → 2. 小步重构 → 3. 测试通过
```

### 第 1 步：识别代码坏味道

**为什么**
先诊断，再治疗。

切换到 Plan Agent：

```
@src/utils/data.ts 请分析这个文件的代码质量：
1. 列出发现的"坏味道"
2. 每个问题的严重程度（高/中/低）
3. 推荐的重构方式
4. 重构的优先级建议
```

### 第 2 步：先生成测试

**为什么**
有测试才能安全重构。

切换到 Build Agent：

```
@src/utils/data.ts 为这个文件生成单元测试：
1. 使用 Vitest 框架
2. 覆盖所有导出函数
3. 包含正常情况和边界情况
4. 保存为 src/utils/data.test.ts
```

### 第 3 步：安全重构

**为什么**  
小步重构，每步验证。

```
@src/utils/data.ts 请重构 parseUserData 函数：
- 问题：函数过长（50 行），职责不单一
- 要求：拆分成 3 个小函数
- 保持对外接口不变
- 重构后运行测试确认
```

### 第 4 步：补充边界测试

**为什么**  
边界条件往往是 Bug 高发区。

```
@src/utils/data.ts @src/utils/data.test.ts

分析代码的边界条件，补充以下测试：
1. 空输入（null、undefined、空数组）
2. 极值（超大数字、超长字符串）
3. 类型错误（传入错误类型）
4. 并发情况（如果有异步操作）
```

### 第 5 步：运行测试验证

**为什么**  
确认重构没有破坏功能。

> 在 OpenCode TUI 里，以 `!` 开头的消息会执行 shell 命令，并把输出带进对话：https://opencode.ai/docs/tui

```
!npm test src/utils/data.test.ts
```

**你应该看到**：所有测试通过


### 文档自动化

| 文档类型 | 生成时机 | AI 辅助方式 |
|---------|---------|-----------|
| README | 项目初期/重大更新 | 分析项目结构生成 |
| API 文档 | 接口开发完成 | 从代码提取生成 |
| Commit 消息 | 每次提交 | 分析变更生成 |
| PR 描述 | 提交 PR | 汇总 commit 生成 |

### 第 1 步：生成 README

**为什么**
好的 README 是项目的门面。

切换到 Build Agent：

```
@explore 分析项目结构，生成一个专业的 README.md：

包含以下部分：
1. 项目名称和简介
2. 功能特性（Feature list）
3. 快速开始（安装和运行）
4. 使用示例
5. 配置说明
6. 贡献指南
7. 许可证

保存为 README.md
```

### 第 2 步：生成 commit 消息

**为什么**  
规范的 commit 消息便于追溯。

先查看变更：

> 在 OpenCode TUI 里，以 `!` 开头的消息会执行 shell 命令，并把输出带进对话：https://opencode.ai/docs/tui

```
!git diff
```

然后让 AI 生成：

```
根据以上变更，生成符合 Conventional Commits 规范的 commit 消息：
- 格式：type(scope): description
- type：feat/fix/docs/style/refactor/test/chore
- 简洁明了，不超过 50 字符
```

执行提交：

```
!git add . && git commit -m "feat(auth): add email verification on registration"
```

### 第 3 步：生成 PR 描述

<AdInArticle />

**为什么**  
清晰的 PR 描述帮助 reviewer 理解。

> 获取最近提交列表（示例）：

```
!git log --oneline -10
```

```
根据以上 commit 历史，生成 PR 描述：

## 变更概述
（一句话总结）

## 变更详情
（分点列出主要修改）

## 测试情况
（测试了什么）

## 相关 Issue
（关联的 issue 编号）
```

### 第 4 步：使用 /undo 撤销

**为什么**  
AI 修改错了可以撤销。

如果 AI 的修改不满意：

```
/undo
```

**你应该看到**：最近一次对话变更被撤销，相关文件改动也会被回滚（需要 Git 仓库）。更多细节见 https://opencode.ai/docs/tui#undo 或 [附录/命令速查](commands.md)。


### 第 5 步：补充代码注释

**为什么**  
好的注释是活文档。

```
@src/services/payment.ts 为这个文件添加 JSDoc 注释：
- 每个导出函数都加
- 包含参数说明和返回值
- 包含使用示例
```


### CI/CD集成
<AdInArticle />

- **优先用 GitHub Agent**：不自己在 workflow 里手写“安装 CLI + 跑脚本 + 发评论”的胶水代码。
- **触发方式统一**：在评论里写 `/oc` 或 `/opencode`，让 Actions runner 执行。

官方说明：https://opencode.ai/docs/github

### 第 1 步：运行安装向导

在你的 GitHub 仓库根目录执行：

```bash
opencode github install
```

安装向导会引导你完成：安装 GitHub App、生成 workflow 文件、提示需要配置哪些 secrets。

### 第 2 步：提交并推送 workflow

把生成的 workflow 文件提交并推送到仓库（通常是 `.github/workflows/opencode.yml`）。

### 第 3 步：用评论触发

在 Issue 或 PR 里评论（示例）：

```text
/oc summarize
```

或者：

```text
/opencode summarize
```