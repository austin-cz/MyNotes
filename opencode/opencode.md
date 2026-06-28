---
title: OpenCode 使用手册
tags:
  - opencode
  - tool
---

# OpenCode 使用手册

## 一、快速入门

### 安装与启动

```bash
# 启动 TUI 交互界面
opencode

# 非交互模式执行任务（适合脚本和 CI/CD）
opencode run "修复 src/main.ts 中的类型错误"

# 启动无头服务器（只有 API）
opencode serve

# 启动 Web 界面服务器
opencode web
```

### 核心概念速览

- **Agent**：可配置的 AI 人格，决定 AI 的身份、能力和行为
- **Skill**：按需加载的专业知识包，通过 SKILL.md 定义
- **MCP**：Model Context Protocol，让 AI 连接外部服务的标准协议
- **Permission**：权限系统，控制 AI 能做什么、不能做什么

## 二、配置系统

### 配置文件位置与优先级

配置按以下顺序加载，后加载的覆盖先加载的冲突键（非冲突设置都保留，是**合并**不是替换）：

| 优先级   | 位置                                 | 说明                     |
| ------- | ---------------------------------- | ---------------------- |
| 1（最低） | 远程 `.well-known/opencode`          | 远程组织默认配置（通过 Auth 机制获取） |
| 2     | `~/.config/opencode/opencode.json` | 全局用户配置                 |
| 3     | `OPENCODE_CONFIG` 环境变量             | 自定义配置文件路径              |
| 4     | `./opencode.json`                  | 项目根目录配置                |
| 5     | `./.opencode/opencode.json`        | 项目 .opencode 目录配置（推荐）  |
| 6     | `OPENCODE_CONFIG_CONTENT` 环境变量     | 内联配置内容（JSON 字符串）       |
| 7（最高） | 受管配置目录                             | 企业部署，管理员控制             |

> [!info] 企业受管配置
> 管理员可在系统级目录放置配置，优先级最高：macOS `/Library/Application Support/opencode`、Windows `%ProgramData%\opencode`、Linux `/etc/opencode`。

### 配置目录结构

```
~/.config/opencode/
├── opencode.json       # 全局配置
├── AGENTS.md           # 全局规则
├── agent/              # 全局 Agent
├── command/            # 全局命令
├── plugin/             # 全局插件
├── skill/              # 全局 Skill
└── themes/             # 全局主题

项目目录/
├── opencode.json       # 项目配置（优先级 4）
├── AGENTS.md           # 项目规则
└── .opencode/
    ├── opencode.json   # 项目配置（优先级 5，推荐）
    ├── agent/          # 项目 Agent
    ├── command/        # 项目命令
    ├── plugin/         # 项目插件
    ├── skill/          # 项目 Skill
    ├── tool/           # 自定义工具
    └── themes/         # 项目主题
```

配置文件名可以是 `opencode.json` 或 `opencode.jsonc`（支持注释的 JSON）。

### 配置字段速查

| 字段 | 类型 | 说明 |
|------|------|------|
| `model` | string | 主模型，格式 `provider/model-id` |
| `small_model` | string | 小模型，用于生成标题等轻量任务 |
| `default_agent` | string | 默认 primary agent（`"build"` / `"plan"` / 自定义名称） |
| `provider` | object | Provider 配置（==注意是单数==，不是 providers） |
| `disabled_providers` | string[] | 禁用的 Provider 列表 |
| `enabled_providers` | string[] | 只启用这些 Provider |
| `username` | string | 对话中显示的用户名 |
| `theme` | string | TUI 主题（==注意是 `theme`，不是 `tui.theme`==） |
| `autoupdate` | boolean / `"notify"` | `true` 自动更新 / `false` 禁用 / `"notify"` 只通知 |
| `instructions` | string[] | 额外规则文件引用列表 |
| `compaction` | object | 上下文压缩配置 |
| `share` | enum | `"manual"` / `"auto"` / `"disabled"` |
| `logLevel` | enum | `"DEBUG"` / `"INFO"` / `"WARN"` / `"ERROR"` |

> [!warning] 常见踩坑键名
> | 配置项 | 正确键名 | 常见错误 |
> |--------|----------|----------|
> | Provider | `provider` | ~~providers~~ |
> | Permission | `permission` | ~~permissions~~ |
> | Agent | `agent` | ~~agents~~ |
> | Command | `command` | ~~commands~~ |
> | Formatter | `formatter` | ~~formatters~~ |
> | Keybinds | `keybinds` | ~~keybind~~ |
> | Theme | `theme` | ~~tui.theme~~ |

### Provider 配置

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "{env:ANTHROPIC_API_KEY}",
        "baseURL": "https://api.anthropic.com",
        "timeout": 600000,
        "setCacheKey": true
      }
    }
  }
}
```

#### Provider options 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `apiKey` | string | API 密钥 |
| `baseURL` | string | 自定义 API 地址（代理场景常用） |
| `timeout` | number / false | 请求超时（毫秒），默认 300000，`false` 禁用 |
| `setCacheKey` | boolean | 启用提示缓存键（默认 false） |
| `enterpriseUrl` | string | GitHub Enterprise URL（仅 Copilot） |

#### Provider 级字段

| 字段 | 说明 |
|------|------|
| `name` | Provider 显示名称 |
| `env` | 环境变量名列表（用于自动检测 API Key） |
| `whitelist` | 只允许这些模型 |
| `blacklist` | 禁用这些模型 |

> `whitelist` 和 `blacklist` 互斥，同时存在时 `whitelist` 优先。

#### Amazon Bedrock 特殊配置

```jsonc
{
  "provider": {
    "amazon-bedrock": {
      "options": {
        "region": "us-east-1",
        "profile": "my-aws-profile",
        "endpoint": "https://bedrock-runtime.us-east-1.vpce-xxxxx.amazonaws.com"
      }
    }
  }
}
```

#### 模型级别自定义 API URL（v1.1.60+）

```jsonc
{
  "provider": {
    "openai": {
      "models": {
        "gpt-4o": {
          "provider": {
            "api": "https://api.custom-openai.com/v1"
          }
        }
      }
    }
  }
}
```

#### 黑白名单

```json
{
  "disabled_providers": ["openai", "gemini"],
  "enabled_providers": ["anthropic"]
}
```

> `disabled_providers` 优先级高于 `enabled_providers`。

#### 我的配置示例

```jsonc
{
  "username": "Allen",
  "$schema": "https://opencode.ai/config.json",
  "shell": "powershell",
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"],
  "autoupdate": "notify",
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
          "modalities": { "input": ["text", "image"], "output": ["text"] }
        },
        "open_provider/qwen3.6-plus": {
          "name": "open_provider/qwen3.6-plus",
          "modalities": { "input": ["text", "image"], "output": ["text"] }
        },
        "open_provider/qwen3.7-max": { "name": "open_provider/qwen3.7-max" }
      }
    }
  },
  "instructions": ["workflow.md", "code.md"],
  "compaction": { "auto": true }
}
```

### 变量替换

配置中可用变量动态获取值：

- `{env:变量名}` — 引用环境变量（不存在则替换为空字符串）
- `{file:路径}` — 引用文件内容（支持相对路径、绝对路径 `/`、home 目录 `~`）

```jsonc
{
  "model": "{env:OPENCODE_MODEL}",
  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "{env:ANTHROPIC_API_KEY}"
      }
    },
    "openai": {
      "options": {
        "apiKey": "{file:~/.secrets/openai-key}"
      }
    }
  }
}
```

### 界面配置

#### TUI 配置

```jsonc
{
  "tui": {
    "scroll_speed": 3,
    "scroll_acceleration": { "enabled": true },
    "diff_style": "auto"
  }
}
```

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `scroll_speed` | 滚动速度倍数（最小 0.001） | 3 |
| `scroll_acceleration.enabled` | 启用 macOS 风格加速滚动 | false |
| `diff_style` | `"auto"` 自适应 / `"stacked"` 始终单列 | `"auto"` |

> `scroll_acceleration.enabled` 优先于 `scroll_speed`。

#### Server 配置

```jsonc
{
  "server": {
    "port": 4096,
    "hostname": "0.0.0.0",
    "mdns": true,
    "mdnsDomain": "opencode.local",
    "cors": ["http://localhost:5173"]
  }
}
```

### 行为配置

#### Share 配置

| 值 | 说明 |
|-----|------|
| `"manual"` | 手动分享（默认），使用 `/share` 命令 |
| `"auto"` | 自动分享新会话 |
| `"disabled"` | 禁用分享功能 |

#### Compaction 配置

```jsonc
{
  "compaction": {
    "auto": true,
    "prune": true,
    "reserved": 10000
  }
}
```

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `auto` | 上下文满时自动压缩 | true |
| `prune` | 删除旧工具输出节省 token | true |
| `reserved` | 压缩时的 Token 缓冲区 | - |

#### Watcher 配置

```jsonc
{
  "watcher": {
    "ignore": ["node_modules/**", "dist/**", ".git/**", "*.log"]
  }
}
```

### 功能配置

#### Tools 配置

```jsonc
{
  "tools": {
    "write": false,
    "bash": false,
    "webfetch": true
  }
}
```

> [!warning] tools vs permission
> `tools` 是遗留配置，会自动转换为 `permission`。推荐使用 `permission` 配置提供更细粒度控制。

#### Permission 配置

```jsonc
{
  "permission": {
    "edit": "ask",
    "bash": {
      "*": "ask",
      "git *": "allow",
      "npm *": "allow",
      "rm *": "deny"
    }
  }
}
```

> 配置键是 `permission`（==单数==），不是 `permissions`。

#### Agent 配置

```jsonc
{
  "agent": {
    "code-reviewer": {
      "description": "代码审查专家",
      "mode": "subagent",
      "model": "anthropic/claude-opus-4-5-thinking",
      "temperature": 0.3,
      "steps": 50,
      "permission": { "edit": "deny" }
    }
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `description` | string | Agent 简介，影响主 Agent 的自动选择决策 |
| `mode` | enum | `subagent` / `primary` / `all`，默认 `all` |
| `model` | string | 格式 `provider/model`，不填则继承主 Agent 当前模型 |
| `temperature` | number | 0-1，控制回答的随机性 |
| `top_p` | number | 0-1，核采样参数 |
| `steps` | number | 最大迭代步数，防止死循环 |
| `color` | string | 十六进制颜色 `#RRGGBB` 或主题色名 |
| `hidden` | boolean | 从 @ 菜单隐藏（仅 subagent 生效） |
| `disable` | boolean | 是否禁用此 Agent |
| `prompt` | string | 系统提示词（JSON 配置专用，Markdown 用正文） |
| `variant` | string | 默认模型变体 |
| `options` | object | 透传参数容器 |

> `maxSteps` 已废弃，请使用 `steps`。

#### Command 配置

```jsonc
{
  "command": {
    "test": {
      "template": "运行测试并显示失败结果",
      "description": "运行测试",
      "agent": "build",
      "model": "anthropic/claude-opus-4-5-thinking"
    }
  }
}
```

#### Formatter 配置

```jsonc
{
  "formatter": {
    "prettier": { "disabled": true },
    "custom-prettier": {
      "command": ["npx", "prettier", "--write", "$FILE"],
      "extensions": [".js", ".ts", ".jsx", ".tsx"]
    }
  }
}
```

设为 `false` 完全禁用格式化。

#### MCP 配置

```jsonc
{
  "mcp": {
    "context7": {
      "type": "local",
      "command": ["npx", "-y", "@upstash/context7-mcp"]
    },
    "sentry": {
      "type": "remote",
      "url": "https://mcp.sentry.dev/mcp",
      "headers": { "Authorization": "Bearer your-token" },
      "oauth": { "clientId": "xxx", "clientSecret": "xxx", "scope": "read write" }
    }
  }
}
```

#### LSP 配置

```jsonc
{
  "lsp": {
    "typescript": { "disabled": true },
    "custom-lsp": {
      "command": ["my-lsp-server", "--stdio"],
      "extensions": [".custom", ".myext"],
      "env": { "DEBUG": "true" },
      "initialization": { "settings": {} }
    }
  }
}
```

设为 `false` 禁用所有 LSP。自定义 LSP 必须提供 `extensions` 字段。

#### Plugin 配置

```jsonc
{
  "plugin": ["opencode-helicone-session", "@my-org/custom-plugin"]
}
```

也可在 `.opencode/plugin/` 目录放置本地插件（`.ts` 或 `.js` 文件）。

### 实验性功能

```jsonc
{
  "experimental": {
    "batch_tool": true,
    "openTelemetry": true,
    "continue_loop_on_deny": false
  }
}
```

| 字段 | 说明 |
|------|------|
| `batch_tool` | 启用批量工具 |
| `openTelemetry` | 启用 OpenTelemetry 追踪 |
| `disable_paste_summary` | 禁用粘贴大段文本时的自动摘要 |
| `primary_tools` | 仅限 Primary Agent 使用的工具列表 |
| `continue_loop_on_deny` | 工具被拒绝时继续循环 |
| `mcp_timeout` | MCP 请求的全局超时时间（毫秒） |

> [!warning] 实验性功能
> 实验性功能可能随时变更或移除。

### 认证

OpenCode 按以下顺序查找认证：**环境变量** → **auth.json** → **配置文件**。

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

> [!warning] 安全提示
> auth.json 以明文 JSON 格式存储 Key，请保护好该文件，不要上传到公开仓库。

## 三、Agent、Skill 与权限系统

### Agent 系统

#### 内置 Agent 概览

| Agent | 读写权限 | 命令权限 | 上下文 | 核心定位 |
|-------|---------|---------|-------|---------|
|Explore|只读|仅安全查询|轻量临时|查代码、摸架构、风险调研|
|Plan|无读写|不能执行命令|临时|方案设计、步骤拆解、风险评估|
|General|读写全权限|任意 Shell|独立子会话|批量改造、一次性复杂任务|
|Build|读写全权限|任意 Shell|主线持续保留|日常迭代、连续开发、BUG 修复|

- **Primary Agent**（Build / Plan）：你直接对话的主助手，用 `Tab` 切换
- **Subagent**（Explore / General）：由 Primary Agent 自动调用或手动用 `@agent名` 调用

> [!note] 内部 Agent
> - **compaction**：上下文压缩（对话接近限制时自动触发）
> - **title**：会话标题生成（创建新会话后自动执行）
> - **summary**：会话摘要生成（压缩会话时生成摘要）

#### Agent 类型

| 类型 | 说明 | 调用方式 |
|------|------|----------|
| **Primary** | 主 Agent，你直接交互的对象 | Tab 键切换 |
| **Subagent** | 子 Agent，被主 Agent 调用执行专项任务 | `@agent名` 或自动调用 |
| **All** | 混合模式，既可作主 Agent，也可被调用 | Tab 切换或 @ 调用 |

子代理运行在**独立的 Session** 中，看不到主 Agent 的对话历史。调用时必须提供完整上下文。

#### 标准工作流

**Explore → Plan → General → Build**

1. `@Explore` 摸底调研 → 确认代码范围、依赖、影响面
2. `@Plan` 输出实施方案 → 敲定改造步骤、风险、回滚方案
3. `@General` 批量落地改造 → 大范围统一修改、文档生成
4. 主线 Build 精细化调试 → 修正细节报错、迭代优化

> [!tip] 记忆方法
> - **Explore** = 装修前上门量房，只勘察不施工
> - **Plan** = 设计师出施工图和预算，只出方案不动手
> - **General** = 施工队统一全屋水电改造，批量一次性施工
> - **Build** = 你自己收尾：缝隙打胶、开关微调、验收

**极简取舍规则**：
- 本地小需求、单文件修改、调试 BUG → 直接 Build
- 线上巡检、大范围代码检索 → 必须先 `@Explore`
- 线上配置变更、架构改造、高危操作 → 严格走标准流程

#### Agent 配置

```jsonc
{
  "agent": {
    "build": {
      "mode": "primary",
      "model": "anthropic/claude-opus-4-5-thinking",
      "temperature": 0.3,
      "permission": { "edit": "allow", "bash": "allow" }
    },
    "plan": {
      "mode": "primary",
      "model": "anthropic/claude-opus-4-5-thinking",
      "temperature": 0.1,
      "permission": {
        "edit": { "*": "deny", ".opencode/plans/*.md": "allow" },
        "bash": "allow"
      },
      "steps": 5
    }
  }
}
```

#### 系统提示词组装机制

Agent prompt 与 Provider 默认提示词是**二选一**关系（非叠加），环境信息和指令文件**始终追加**。

```text
┌────────────────────────────────────────────┐
│ 1. Agent prompt 或 Provider 默认提示词     │
│    ├─ 有 prompt → 使用你定义的              │
│    └─ 无 prompt → 使用模型默认提示词         │
├────────────────────────────────────────────┤
│ 2. 环境信息 + 指令文件（始终添加）           │
│    工作目录、git 状态、平台、日期、AGENTS.md  │
└────────────────────────────────────────────┘
```

Provider 默认提示词映射表：

| 模型 | 提示词文件 | 风格特点 |
|------|-----------|---------|
| Claude | `anthropic.txt` | 强调 TodoWrite、使用专用工具、简洁输出 |
| GPT-5/O1/O3 | `beast.txt`/`codex_header.txt` | 强调持续迭代、联网研究、自主解决 |
| Gemini | `gemini.txt` | 适配 Gemini 特性 |
| 其他（Qwen 等）| `qwen.txt` | 类似 Anthropic 但不含 TodoWrite |

#### 各模型默认 Temperature 值

| 模型 | 默认 Temperature | 说明 |
|------|-----------------|------|
| Claude | undefined | 使用 API 默认 |
| GPT | undefined | 使用 API 默认 |
| Gemini | 1.0 | 固定值 |
| Qwen | 0.55 | 固定值 |
| GLM-4.6/4.7 | 1.0 | 固定值 |
| MiniMax-M2 | 1.0 | 固定值 |
| Kimi-K2 | 0.6 或 1.0 | thinking/k2 版本为 1.0 |

优先级：`agent.temperature（用户设置）> 模型默认值 > undefined`

Plan 模式下生成的计划文件保存位置：

| 级别 | 路径 | 条件 |
|------|------|------|
| 项目级 | `.opencode/plans/<时间戳>-<slug>.md` | 项目有 Git |
| 全局级 | `~/.local/share/opencode/plans/<时间戳>-<slug>.md` | 项目无版本控制 |

#### Agent 权限与安全

权限系统有三种动作：`allow`（允许）、`ask`（询问）、`deny`（禁止）。

**规则优先级**：==最后匹配的规则生效==。

```jsonc
{
  "permission": {
    "bash": {
      "*": "ask",
      "git *": "allow",
      "git push*": "deny"
    }
  }
}
```

执行 `git push origin main` → 匹配 `*`(ask) → 匹配 `git *`(allow) → 匹配 `git push*`(deny) → **最终结果：deny**

#### Agent 设计模式

#### 五种 Workflow 模式

| 模式 | 原理 | 适用场景 |
|------|------|---------|
| **Prompt Chaining** | 顺序执行，每步输出作为下步输入 | 任务可分解为固定子任务 |
| **Routing** | 根据输入特征导向不同处理流程 | 不同类别需不同处理 |
| **Parallelization** | 多任务同时执行，汇总结果 | 子任务独立、需多视角 |
| **Orchestrator-Workers** | 中央 LLM 动态分析并分配给 Worker | 任务复杂度不确定 |
| **Evaluator-Optimizer** | 生成→评估→循环直到满足标准 | 有明确评估标准 |

#### ACI 设计原则

> "像设计人机界面（HCI）一样投入精力设计 Agent-计算机界面（ACI）。" —— Anthropic

1. **给模型思考空间**：避免需要精确计数或复杂转义的格式
2. **格式接近自然语言**：Markdown 代码块比 JSON 转义容易
3. **描述像优秀的 docstring**：包含功能说明、使用示例、边界情况

#### 透传参数

Agent 配置中未知字段会**透传给 provider**：

```jsonc
{
  "agent": {
    "deep-thinker": {
      "model": "openai/o1",
      "reasoningEffort": "high",
      "textVerbosity": "low"
    }
  }
}
```

| 参数 | Provider | 说明 |
|------|----------|------|
| `reasoningEffort` | OpenAI o 系列 | `high` / `medium` / `low` |
| `textVerbosity` | OpenAI | 输出详细程度 |
| `top_k` | Anthropic | 采样参数 |

#### 自定义 Agent 创建

#### Markdown 方式（推荐）

创建 `.opencode/agent/docs-writer.md`：

```markdown
---
description: 技术文档写作专家
mode: subagent
temperature: 0.3
permission:
  edit: deny
---

# 角色
你是技术文档专家。

# 工作流程
1. 理解代码
2. 写快速开始
3. 写 API 文档
```

#### JSON 方式

```jsonc
{
  "agent": {
    "code-reviewer": {
      "description": "代码审查专家",
      "mode": "subagent",
      "prompt": "{file:./prompts/code-reviewer.txt}",
      "temperature": 0.2
    }
  }
}
```

#### 嵌套子目录 Agent

```
.opencode/agent/
├── review/
│   ├── security.md       → @review/security
│   └── performance.md    → @review/performance
└── docs/
    └── api.md            → @docs/api
```

#### Agent 设计模式详解

#### 三种核心原则详细展开

1. **保持简单**（3 条实践）：
   - 能用单个 Agent 解决的，不要用多个
   - 能用固定流程的，不要用动态决策
   - 能用 prompt 解决的，不要加工具

2. **透明度优先**（3 条实践）：
   - Agent 的思考过程应该可见
   - 每个步骤都有明确的输入输出
   - 用户能理解 Agent 在做什么

3. **精心设计工具接口（ACI）**（3 条实践）：
   - 工具描述要像给初级开发者写的优秀 docstring
   - 包含使用示例和边界情况
   - 避免需要精确计数或复杂转义的格式

#### Workflow vs Agent 选择决策流程

```text
任务来了
    ↓
步骤是否固定？
    ├─ 是 → 用 Workflow（Skill/Command）
    └─ 否 → 需要多少自主性？
              ├─ 低 → 受限 Agent（steps + 权限控制）
              └─ 高 → 完全自主 Agent
```

| 类型 | 特点 | 适用场景 | OpenCode 实现 |
|------|------|---------|--------------|
| **Workflow** | 预定义代码路径，步骤固定 | 任务可预测、结构清晰 | Skill、Command |
| **Agent** | LLM 动态决策，自主探索 | 开放性问题、无法预测步骤 | Agent + Task tool |

#### 设计检查清单（7 项）

1. **简单性**：能用更简单的方案吗？
2. **模式选择**：选对 Workflow 模式了吗？
3. **工具描述**：描述清晰、易于理解吗？
4. **评估标准**：如何判断 Agent 完成了任务？
5. **失败处理**：出错时如何恢复？
6. **权限控制**：是否限制了不必要的权限？
7. **资源限制**：设置了 steps 限制吗？

#### 常见设计陷阱（5 项）

| 陷阱 | 表现 | 解决 |
|------|------|------|
| **过度设计** | 用多 Agent 解决简单问题 | 先用单 Agent 尝试 |
| **模糊描述** | description 太泛导致错误调用 | 具体说明适用场景 |
| **无限循环** | Agent 互相调用不停止 | 设置 steps 限制 |
| **权限过松** | subagent 可以做任何事 | 明确 task/edit/bash 权限 |
| **缺乏透明** | 不知道 Agent 在做什么 | 要求输出中间步骤 |

### Skill 系统

#### 核心理念与三层结构

Skill 是**按需加载**的专业知识包，通过 SKILL.md 定义。

**渐进式披露**：
- **第一层**：name + description（~100 词），始终可见，用于判断是否需要加载
- **第二层**：SKILL.md 正文，任务匹配时加载
- **第三层**：references/ 目录中的详细文档，仅在需要时加载

#### SKILL.md 格式

```yaml
---
name: sql-analysis
description: |
  用于分析业务数据：收入、ARR、客户分群。
  提供：表结构、指标定义、标准过滤器、查询模板。
  适用：写 SQL 分析业务数据、理解公司指标。
  不适用：数据库管理、DDL 操作、性能调优。
---
```

| 字段 | 必需 | 说明 |
|------|------|------|
| `name` | 是 | 技能标识符 |
| `description` | 是 | ==触发条件描述（最重要！）== |

#### 搜索位置

| 位置 | 作用范围 |
|------|---------|
| `.opencode/skill/<name>/SKILL.md` | 当前项目 |
| `~/.config/opencode/skill/<name>/SKILL.md` | 全局 |
| `.claude/skills/<name>/SKILL.md` | Claude 兼容格式 |

#### 创建流程与测试

1. **明确需求**：解决什么问题？何时触发？
2. **写 name**：小写字母、数字、连字符
3. **写 description**：具体能力 + 触发场景 + 边界限制
4. **写主要指令**：结构化、可扫描、可操作
5. **测试验证**：正向触发、负向测试、输出格式

#### 可执行脚本

```
skill/pdf-skill/
├── SKILL.md
├── references/
└── scripts/
    ├── extract_form_fields.py
    └── merge_pdfs.py
```

#### Skill 与 MCP 协作

- **MCP 提供**：连接服务、实时数据、工具调用（= 厨房）
- **Skill 提供**：使用最佳实践、工作流编排、领域知识（= 菜谱）

#### 5 种工作流模式

| 模式 | 适用场景 |
|------|---------|
| 顺序编排 | 固定步骤，依赖传递 |
| 多 MCP 协调 | 跨服务编排 |
| 迭代优化 | 质量循环 |
| 上下文选择 | 决策树选工具 |
| 领域智能 | 合规优先 |

#### 分发与共享

| 方式 | 适用场景 |
|------|---------|
| 本地目录 | 个人使用 |
| `skills.paths` 额外路径 | 团队共享（NAS） |
| `skills.urls` 远程 URL | 企业/社区 |
| Git 仓库 | 开源/团队 |

### 快捷命令

#### 命令文件与配置

| 位置 | 作用范围 |
|------|---------|
| `.opencode/command/**/*.md` | 当前项目 |
| `~/.config/opencode/command/**/*.md` | 全局 |

嵌套目录：`git/commit.md` → `/git/commit`

#### 两种配置方式

**Markdown 方式**（推荐）：

```markdown
---
description: 运行测试
agent: build
---

运行完整测试套件，关注失败用例。
```

**JSON 方式**：

```jsonc
{
  "command": {
    "test": {
      "template": "运行测试套件",
      "description": "运行测试",
      "agent": "build"
    }
  }
}
```

#### 模板语法

| 语法 | 说明 |
|------|------|
| `$ARGUMENTS` | 全部参数 |
| `$1`, `$2` | 位置参数 |
| `` !`command` `` | Shell 命令输出 |
| `@file` | 文件引用 |

#### 覆盖内置命令

创建 `.opencode/command/` 文件即可覆盖内置命令（如 `/help`、`/compact`）。

#### 命令配置完整选项

| 选项 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `template` | string | 是 | 发送给 LLM 的提示词 |
| `description` | string | 可选 | TUI 中显示的命令描述 |
| `agent` | string | 可选 | 执行命令的 Agent |
| `model` | string | 可选 | 覆盖默认模型 |
| `subtask` | boolean | 可选 | 强制作为子任务执行 |

> **subtask 行为**：如果 agent 是 subagent 类型，默认自动触发子任务，设置 `subtask: false` 可禁用。如果 agent mode 是 primary，设置 `subtask: true` 可强制以子任务方式执行，不污染主上下文。

### 权限系统

#### 权限模式

| 模式 | 说明 |
|------|------|
| `allow` | 允许执行，不询问 |
| `ask` | 每次询问用户确认 |
| `deny` | 禁止执行 |

#### 权限类型详解

**支持细粒度规则的权限**：

| 权限 | 匹配对象 | 说明 |
|------|---------|------|
| `read` | 文件路径 | 读取文件 |
| `edit` | 文件路径 | 所有文件修改 |
| `glob` | glob 模式 | 文件搜索 |
| `grep` | 正则 | 内容搜索 |
| `list` | 目录路径 | 列出目录 |
| `bash` | 命令字符串 | 执行 shell |
| `task` | subagent 名称 | 调用子 Agent |
| `skill` | skill 名称 | 加载技能 |
| `external_directory` | 目录路径 | 访问项目外路径 |

**仅支持简单语法的权限**：`todoread`、`todowrite`、`webfetch`、`websearch`、`codesearch`、`doom_loop`

#### Bash 权限详解

Bash 权限匹配的是**解析后的命令字符串**（按 arity 切分）：

```jsonc
{
  "permission": {
    "bash": {
      "*": "ask",
      "git status": "allow",
      "git log*": "allow",
      "npm run*": "allow",
      "rm -rf*": "deny",
      "sudo*": "deny"
    }
  }
}
```

| 输入命令 | 解析后匹配 |
|---------|-----------|
| `git checkout main` | `git checkout` |
| `npm install` | `npm install` |
| `npm run dev` | `npm run dev` |

#### Edit 权限详解

```jsonc
{
  "permission": {
    "edit": {
      "*": "allow",
      "*.env": "deny",
      "package-lock.json": "deny",
      "node_modules/*": "deny"
    }
  }
}
```

#### Task 权限

控制 Agent 可以调用哪些 subagent。设为 `deny` 时，该 subagent 从工具描述中移除。

#### 内置安全规则

- **.env 文件保护**：默认 `ask`
- **doom_loop 检测**：同一工具连续调用 3 次相同输入时触发
- **external_directory**：访问项目外路径默认 `ask`

#### 安全最佳实践

1. **最小权限原则**：只授予必要权限
2. **显式允许**：`*` deny，然后逐条 allow
3. **敏感操作 ask**：git push、npm publish 等

#### 子代理硬编码限制

| 限制 | 原因 |
|------|------|
| `todowrite: deny` | 防止子代理干扰主 Agent 任务 |
| `todoread: deny` | 同上 |
| `task: deny` | 防止无限递归（子代理不能再调子代理）|

#### 权限确认快捷键

| 按键 | 功能 |
|------|------|
| `y` | 单次允许 |
| `a` | 始终允许（本会话内） |
| `n` | 拒绝 |
| `Ctrl+C` | 中断 AI 执行 |

## 四、会话管理与上下文压缩

### 会话管理

#### 数据存储与生命周期

```
~/.local/share/opencode/storage/
├── session/<project-id>/<session-id>.json
├── message/<session-id>/<message-id>.json
└── part/<message-id>/<part-id>.json
```

生命周期：创建 → 活跃对话 → 压缩（上下文太长时自动触发）→ 归档/删除

#### 会话命令

| 命令 | 作用 |
|------|------|
| `/new` | 新建会话 |
| `/sessions` | 查看并切换会话 |
| `/undo` | 撤销上一步操作（需要 Git 仓库） |
| `/redo` | 重做被撤销的操作 |
| `/compact` | 压缩上下文 |
| `/export` | 导出对话（Markdown 格式） |
| `/share` | 分享会话（生成链接） |

#### 导入导出与 Fork

```bash
opencode export                   # 交互式选择会话导出
opencode export session_abc > backup.json
opencode import backup.json       # 从文件导入
```

Fork 会复制当前会话的所有历史消息，创建新会话。配置快捷键：

```jsonc
{ "keybinds": { "session_fork": "<leader>f" } }
```

#### 上下文压缩

**Context 百分比计算**：

```
Context % = (input + output + reasoning + cache.read + cache.write) / model.limit.context * 100
```

**自动压缩触发条件**：`token_count >= (input_limit - reserved)`，其中 reserved = min(20000, 模型最大输出)

**压缩两步操作**：
1. **Prune**：清除旧工具输出（保留最近 40K tokens）
2. **Summarize**：用 compaction agent 生成详细摘要

> 压缩**不删除数据库**中的历史消息，用户翻页仍可看到完整历史，但发送给模型时只发送压缩点之后的上下文。

| 模型 | Context | 触发阈值 |
|------|---------|---------|
| Claude Sonnet 4.5 | 200K | ~168K |
| Gemini 3 Flash | ~1M | ~1,016K |
| GPT 5.2 | 400K | ~368K |

#### 查看当前会话 Session ID

| 方式 | 操作 |
|------|------|
| **CLI** | `opencode session list` |
| **TUI** | 按 `<leader>l` 打开会话列表，当前会话有高亮标记 |

#### /sessions 的删除功能

`/sessions` 打开会话列表后，除了查看和切换，还可以直接删除不需要的会话。

## 五、MCP、IDE 与远程连接

### MCP 扩展

#### 本地 MCP

```jsonc
{
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
      "enabled": true,
      "environment": { "MY_ENV_VAR": "value" }
    }
  }
}
```

| 选项 | 类型 | 说明 |
|------|------|------|
| `type` | string | 必须是 `"local"` |
| `command` | array | 命令数组 |
| `environment` | object | 环境变量 |
| `enabled` | boolean | 默认 `true` |
| `timeout` | number | 连接超时（毫秒），默认 30000 |

#### 远程 MCP

```jsonc
{
  "mcp": {
    "sentry": {
      "type": "remote",
      "url": "https://mcp.sentry.dev/mcp",
      "headers": { "Authorization": "Bearer {env:MY_API_KEY}" },
      "oauth": { "clientId": "xxx", "scope": "read write" }
    }
  }
}
```

交互式添加：`opencode mcp add`

#### OAuth 认证

OpenCode 自动处理 OAuth 流程，Token 安全存储在 `~/.local/share/opencode/mcp-auth.json`。

```bash
opencode mcp auth my-server     # 手动触发认证
opencode mcp auth list          # 查看所有认证状态
opencode mcp debug my-server    # 调试连接
```

禁用 OAuth（使用 API Key）：

```jsonc
{ "oauth": false, "headers": { "Authorization": "Bearer {env:MY_API_KEY}" } }
```

#### 权限管理

MCP 工具命名格式：`{服务器名}_{工具名}`。使用通配符批量控制：

```jsonc
{
  "permission": {
    "my-mcp_*": "deny"
  }
}
```

#### Chrome DevTools MCP

让 AI 直接连接你的 Chrome 浏览器，调试网页、分析性能：

```jsonc
{
  "mcp": {
    "chrome-devtools": {
      "type": "local",
      "command": ["npx", "-y", "chrome-devtools-mcp@latest", "--autoConnect", "--channel=stable"]
    }
  }
}
```

前置条件：Chrome 146+，在 `chrome://inspect/#remote-debugging` 启用远程调试。

#### 常用 MCP 推荐

| MCP | 类型 | 用途 |
|-----|------|------|
| Sentry | remote | 错误监控 |
| Context7 | remote | 文档搜索 |
| Grep by Vercel | remote | GitHub 代码搜索 |
| Filesystem | local | 文件系统操作 |
| Postgres | local | PostgreSQL 查询 |
| Puppeteer | local | 浏览器自动化 |
| Memory | local | 持久化键值存储 |

#### Chrome DevTools MCP 工具完整列表（28 个）

| 工具 | 功能描述 |
|------|----------|
| `list_pages` | 列出所有打开的标签页 |
| `select_page` | 选择一个标签页 |
| `navigate_page` | 导航到 URL / 前进 / 后退 / 刷新 |
| `take_snapshot` | 获取页面快照（可访问性树）|
| `take_screenshot` | 截取页面或元素截图 |
| `click` | 点击元素 |
| `fill` | 填写表单 |
| `type_text` | 输入文本 |
| `press_key` | 按键操作 |
| `hover` | 悬停在元素上 |
| `evaluate_script` | 执行 JavaScript |
| `list_network_requests` | 列出网络请求 |
| `get_network_request` | 获取请求详情 |
| `list_console_messages` | 列出控制台消息 |
| `lighthouse_audit` | 运行 Lighthouse 审计 |
| `performance_start_trace` | 开始性能追踪 |
| `performance_stop_trace` | 停止性能追踪 |
| `new_page` | 新开标签页 |
| `close_page` | 关闭标签页 |
| `wait_for` | 等待页面加载/元素出现 |
| `drag` | 拖拽元素 |
| `fill_form` | 批量填写多项表单 |
| `handle_dialog` | 处理 alert/confirm/prompt 弹窗 |
| `upload_file` | 上传文件（图片、PDF、附件）|
| `emulate` | 模拟移动端/弱网/地理位置/明暗模式 |
| `get_console_message` | 获取某条控制台消息完整内容 |
| `performance_analyze_insight` | 分析性能追踪的某个性能洞察 |
| `take_memory_snapshot` | 内存堆快照（排查内存泄漏）|

### IDE 集成

#### VS Code/Cursor 集成

安装方式：在 VS Code 集成终端运行 `opencode`，扩展自动安装。

| 功能 | macOS | Windows/Linux |
|------|-------|---------------|
| 打开面板 | `Cmd+Esc` | `Ctrl+Esc` |
| 新建会话 | `Cmd+Shift+Esc` | `Ctrl+Shift+Esc` |
| 插入文件引用 | `Cmd+Option+K` | `Alt+Ctrl+K` |

选中行自动感应，生成 `@file.ts#L10-20` 格式的引用。

#### ACP 协议

支持 Zed、JetBrains、Neovim 等编辑器：

```bash
opencode acp
```

**Zed 配置**（`~/.config/zed/settings.json`）：

```json
{
  "agent_servers": {
    "OpenCode": { "command": "opencode", "args": ["acp"] }
  }
}
```

**JetBrains**：需要 opencode 的**绝对路径**。

#### ACP 详细配置示例

#### JetBrains acp.json 配置

```json
{
  "agent_servers": {
    "OpenCode": {
      "command": "/absolute/path/bin/opencode",
      "args": ["acp"]
    }
  }
}
```

> **注意**：JetBrains 必须使用 opencode 的绝对路径。

#### Neovim Avante.nvim 配置

```lua
{
  acp_providers = {
    ["opencode"] = {
      command = "opencode",
      args = { "acp" }
    }
  }
}
```

#### Neovim CodeCompanion.nvim 配置

```lua
require("codecompanion").setup({
  strategies = {
    chat = {
      adapter = {
        name = "opencode",
        model = "claude-sonnet-4",
      },
    },
  },
})
```

#### Zed keymap 配置

```json
[
  {
    "bindings": {
      "cmd-alt-o": [
        "agent::NewExternalAgentThread",
        {
          "agent": {
            "custom": {
              "name": "OpenCode",
              "command": {
                "command": "opencode",
                "args": ["acp"]
              }
            }
          }
        }
      ]
    }
  }
]
```

#### ACP 命令参数

| 参数 | 说明 |
|------|------|
| `--cwd <dir>` | 工作目录 |
| `--port <port>` | 监听端口 |
| `--hostname <addr>` | 监听主机名 |

#### ACP 不支持的功能

ACP 模式下不可用的 TUI 命令：`/undo`、`/redo`。

### 远程模式与 API

#### 服务器启动与连接

| 命令 | 用途 |
|------|------|
| `opencode` | 日常使用（自动启动本地服务器） |
| `opencode serve` | 无头服务（纯 API） |
| `opencode web` | Web 界面 |
| `opencode attach <url>` | 连接远程服务器 |

```bash
opencode serve --port 4096 --hostname 0.0.0.0
opencode attach http://192.168.1.100:4096
```

> [!warning] 安全
> 默认无认证保护。开放外网访问时必须设置 `OPENCODE_SERVER_PASSWORD` 环境变量。

#### HTTP API 参考

| API | 路径 | 说明 |
|-----|------|------|
| 健康检查 | `GET /global/health` | `{ healthy: true, version: string }` |
| 会话列表 | `GET /session` | 返回 Session[] |
| 创建会话 | `POST /session` | body: `{ title }` |
| 发送消息 | `POST /session/:id/message` | body: `{ parts }` |
| 事件流 | `GET /event` | SSE 事件流 |

API 文档：`http://<hostname>:<port>/doc`

#### SDK 开发

```bash
npm install @opencode-ai/sdk
```

```typescript
import { createOpencode } from "@opencode-ai/sdk"

const { client, server } = await createOpencode()
const sessions = await client.session.list()
server.close()
```

#### SDK 三种使用方式完整参数

**`createOpencode()`（服务器+客户端）**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `hostname` | string | `127.0.0.1` | 服务器主机名 |
| `port` | number | `4096` | 服务器端口 |
| `signal` | AbortSignal | - | 中止信号 |
| `timeout` | number | `2000+` | 启动超时（毫秒）|
| `config` | Config | `{}` | 配置对象 |

**`createOpencodeClient()`（仅客户端）**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `baseUrl` | string | 服务器 URL（默认 `http://localhost:4096`）|
| `fetch` | function | 自定义 fetch 实现 |
| `parseAs` | string | 响应解析：`auto`/`json`/`text`/`blob`/`stream` 等 |
| `responseStyle` | `"data" \| "fields"` | `data` 仅返回数据 |
| `throwOnError` | boolean | 出错时抛出异常（默认 false）|
| `directory` | string | 项目目录（X-Opencode-Directory header）|

**`createOpencodeTui()`（启动 TUI）**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `project` | string | 项目目录路径 |
| `model` | string | 使用的模型 |
| `session` | string | 恢复指定会话 ID |
| `agent` | string | 使用的 Agent |
| `signal` | AbortSignal | 中止信号 |

#### SDK 实时事件监听

```typescript
const events = await client.event.subscribe()

for await (const event of events.stream) {
  switch (event.type) {
    case "message.updated":
      console.log("消息更新")
      break
    case "session.idle":
      console.log("会话空闲")
      break
    case "permission.updated":
      // 可自动响应权限
      break
  }
}
```

常用事件：`message.updated`、`session.idle`、`permission.updated`、`file.edited`、`todo.updated`

#### SDK API 模块列表（21 个）

`global`、`project`、`session`、`file`、`find`、`config`、`app`、`tui`、`event`、`auth`、`provider`、`mcp`、`lsp`、`formatter`、`command`、`path`、`vcs`、`pty`、`tool`、`instance`

> API 文档地址：`http://<hostname>:<port>/doc`

#### 网络代理配置

```bash
# HTTPS 代理（推荐）
export HTTPS_PROXY=https://proxy.example.com:8080

# HTTP 代理
export HTTP_PROXY=http://proxy.example.com:8080

# 绕过代理的地址（必须包含本地地址）
export NO_PROXY=localhost,127.0.0.1
```

> 代理认证：`export HTTPS_PROXY=http://username:password@proxy.example.com:8080`

> **重要**：TUI 与本地服务器通信时必须绕过代理（`NO_PROXY=localhost,127.0.0.1`），否则连接失败。

#### 自定义证书

```bash
export NODE_EXTRA_CA_CERTS=/path/to/ca-cert.pem
```

适用于企业自签名 CA 证书环境，同时作用于代理连接和直接 API 访问。

#### mDNS 服务发现

- mDNS 仅在 hostname 非回环地址时生效
- 实际发布的 mDNS 名称格式为 `opencode-{port}`（如 `opencode-4096.local`）

#### attach 命令选项

| 参数 | 说明 |
|------|------|
| `--dir` | 启动 TUI 的工作目录 |
| `--session` / `-s` | 要继续的会话 ID |

#### HTTP API 完整分类（19 个）

| 分类 | 路径前缀 | 说明 |
|------|----------|------|
| `/global` | 全局 | 健康检查、全局事件流 |
| `/project` | 项目 | 项目列表、当前项目 |
| `/path`、`/vcs` | 路径信息 | 路径信息、VCS 信息 |
| `/instance` | 实例 | 实例管理（dispose）|
| `/config` | 配置 | 配置读写、providers |
| `/provider` | 提供商 | 提供商管理、OAuth |
| `/session` | 会话 | 会话 CRUD、fork、share、abort 等（18 个端点）|
| `/session/:id/message` | 消息 | 消息列表、发送、异步发送 |
| `/command` | 命令 | 命令列表 |
| `/find` | 搜索 | 文件内容/名称/符号搜索 |
| `/file` | 文件 | 文件列表、内容、状态 |
| `/experimental/tool` | 实验性 | 工具 ID 和 JSON Schema |
| `/lsp` | LSP | LSP 服务器状态 |
| `/formatter` | 格式化器 | 格式化器状态 |
| `/mcp` | MCP | MCP 管理、动态添加、OAuth |
| `/agent` | Agent | Agent 列表 |
| `/log` | 日志 | 写入日志 |
| `/tui` | TUI | 远程控制（12 个方法）|
| `/auth` | 认证 | 认证凭据设置 |

#### TUI 控制 API（12 个方法）

| 方法 | 路径 | 描述 |
|------|------|------|
| `POST` | `/tui/append-prompt` | 追加文本到提示框 |
| `POST` | `/tui/open-help` | 打开帮助对话框 |
| `POST` | `/tui/open-sessions` | 打开会话选择器 |
| `POST` | `/tui/open-themes` | 打开主题选择器 |
| `POST` | `/tui/open-models` | 打开模型选择器 |
| `POST` | `/tui/submit-prompt` | 提交当前提示 |
| `POST` | `/tui/clear-prompt` | 清空提示框 |
| `POST` | `/tui/execute-command` | 执行命令 `body: { command }` |
| `POST` | `/tui/show-toast` | 显示通知 |
| `GET` | `/tui/control/next` | 等待下一个控制请求 |
| `POST` | `/tui/control/response` | 响应控制请求 |

## 六、命令行与界面

### CLI 命令参考

#### 模型管理

```bash
opencode models                   # 列出所有可用模型
opencode models anthropic         # 只看 Anthropic 的模型
opencode models --verbose         # 包括价格等元数据
opencode models --refresh         # 强制刷新缓存
```

#### 费用统计

```bash
opencode stats                    # 总账单
opencode stats --project ""       # 当前项目费用
opencode stats --days 7           # 最近 7 天
opencode stats --models 5         # Top 5 模型消耗
```

OpenCode 依靠 Git 仓库的**第一条提交记录**识别项目。非 Git 目录的消耗归入 Global。

#### 所有 CLI 命令速查

| 命令 | 功能 |
|------|------|
| `opencode` | 启动 TUI |
| `opencode run` | 非交互执行 |
| `opencode serve` | 无头服务器 |
| `opencode web` | Web 界面 |
| `opencode attach` | 连接远程 |
| `opencode auth` | 认证管理 |
| `opencode models` | 模型列表 |
| `opencode agent` | Agent 管理 |
| `opencode mcp` | MCP 管理 |
| `opencode session` | 会话管理 |
| `opencode stats` | 使用统计 |
| `opencode export/import` | 会话导出/导入 |
| `opencode github install` | GitHub Actions 安装 |
| `opencode pr <number>` | 拉取并处理 PR |
| `opencode acp` | ACP 服务器 |
| `opencode upgrade` | 升级版本 |
| `opencode debug` | 调试诊断 |

#### 斜杠命令速查

| 命令 | 功能 |
|------|------|
| `/new` | 新建会话 |
| `/sessions` | 会话列表 |
| `/models` | 模型切换 |
| `/connect` | 添加提供商 |
| `/undo` / `/redo` | 撤销/重做 |
| `/compact` | 压缩上下文 |
| `/share` / `/unshare` | 分享/取消分享 |
| `/theme` | 主题切换 |
| `/init` | 生成项目规则 |
| `/editor` | 外部编辑器 |
| `/export` | 导出会话 |
| `/help` | 帮助 |

#### CLI 自动化

```bash
# 非交互模式
opencode run "列出所有 TypeScript 文件"
opencode run -m anthropic/claude-opus-4-5 "Review this code"
opencode run -f src/main.ts -f package.json "Analyze this"
opencode run --format json "List all TODO" > todos.json

# 从 stdin 读取
cat error.log | opencode run "分析错误日志"

# CI/CD 中连接已运行的服务器（避免 MCP 冷启动）
opencode run --attach http://localhost:4096 "Explain async/await"
```

#### CLI 子命令完整参考

#### opencode 命令完整参数

| 选项 | 短选项 | 说明 |
|------|--------|------|
| `--continue` | `-c` | 继续上次会话 |
| `--session` | `-s` | 指定会话 ID |
| `--prompt` | | 初始提示语 |
| `--model` | `-m` | 指定模型（provider/model）|
| `--agent` | | 指定 Agent |
| `--port` | | 服务器端口 |
| `--hostname` | | 服务器地址 |

#### opencode run 完整参数

| 选项 | 短选项 | 说明 |
|------|--------|------|
| `--command` | | 执行斜杠命令（message 作参数）|
| `--continue` | `-c` | 继续上次会话 |
| `--session` | `-s` | 指定会话 ID |
| `--share` | | 分享会话 |
| `--title` | | 会话标题 |
| `--variant` | | 推理力度：`high`/`max`/`minimal` |

#### opencode auth 子命令

| 子命令 | 说明 |
|--------|------|
| `login` | 交互式登录 |
| `list` / `ls` | 列出已认证提供商 |
| `logout` | 登出 |

#### opencode agent 子命令

| 子命令 | 说明 |
|--------|------|
| `list` | 列出所有 Agent |
| `create` | 创建新 Agent（交互式）|

#### opencode mcp 子命令

| 子命令 | 说明 |
|--------|------|
| `list` / `ls` | 列出 MCP 服务器状态 |
| `add` | 添加 MCP 服务器 |
| `auth [name]` | OAuth 认证 |
| `auth list` | 列出 OAuth 状态 |
| `logout [name]` | 移除 OAuth 凭证 |
| `debug <name>` | 调试连接问题 |

#### opencode uninstall 命令

| 选项 | 说明 |
|------|------|
| `--keep-config` | 保留配置文件 |
| `--keep-data` | 保留会话数据和快照 |
| `--dry-run` | 预览删除内容 |
| `--force` | 跳过确认 |

### 快捷键

#### Leader 键机制

默认 Leader 键：<kbd>Ctrl</kbd>+<kbd>X</kbd>

**操作方式**：先按 Leader 键，==松开==，再按字母键。

#### 完整快捷键列表

**应用控制**：

| 键名 | 默认值 | 说明 |
|------|--------|------|
| `leader` | `ctrl+x` | Leader 键 |
| `app_exit` | `ctrl+c,ctrl+d,<leader>q` | 退出 |
| `terminal_suspend` | `ctrl+z` | 挂起 |

**会话管理**：

| 键名 | 默认值 | 说明 |
|------|--------|------|
| `session_new` | `<leader>n` | 新建会话 |
| `session_list` | `<leader>l` | 会话列表 |
| `session_compact` | `<leader>c` | 压缩 |
| `session_fork` | `none` | 分叉 |
| `session_interrupt` | `escape` | 中断 |

**消息操作**：

| 键名 | 默认值 | 说明 |
|------|--------|------|
| `messages_copy` | `<leader>y` | 复制消息 |
| `messages_undo` | `<leader>u` | 撤销 |
| `messages_redo` | `<leader>r` | 重做 |
| `messages_page_up` | `pageup` | 上翻页 |
| `messages_page_down` | `pagedown` | 下翻页 |

**模型与 Agent**：

| 键名 | 默认值 | 说明 |
|------|--------|------|
| `model_list` | `<leader>m` | 模型列表 |
| `model_cycle_recent` | `f2` | 切换最近模型 |
| `variant_cycle` | `ctrl+t` | 切换模型变体（思考深度） |
| `agent_list` | `<leader>a` | Agent 列表 |
| `agent_cycle` | `tab` | 切换 Agent |
| `command_list` | `ctrl+p` | 命令面板 |

**界面控制**：

| 键名 | 默认值 | 说明 |
|------|--------|------|
| `sidebar_toggle` | `<leader>b` | 侧边栏 |
| `theme_list` | `<leader>t` | 主题列表 |
| `editor_open` | `<leader>e` | 外部编辑器 |
| `status_view` | `<leader>s` | 状态视图 |

**会话导航**：

| 键名 | 默认值 | 说明 |
|------|--------|------|
| `session_child_cycle` | `<leader>right` | 进入子会话 |
| `session_child_cycle_reverse` | `<leader>left` | 返回 |
| `session_parent` | `<leader>up` | 返回父会话 |

**配置快捷键**：

```jsonc
{
  "keybinds": {
    "leader": "ctrl+x",
    "session_new": "<leader>n",
    "app_exit": "ctrl+c,ctrl+d,<leader>q",
    "session_compact": "none"
  }
}
```

> [!warning] keybinds 是唯一使用复数的配置键！

#### 终端兼容性

Shift+Enter 问题：部分终端需要在 Windows Terminal 的 `settings.json` 中添加自定义 keybinding 发送 `\u001b[13;2u`。

#### 输入区快捷键

**基础操作**：

| 键名 | 默认值 | 说明 |
|------|--------|------|
| `input_submit` | `return` | 发送消息 |
| `input_newline` | `shift+return,ctrl+return` | 换行 |
| `input_clear` | `ctrl+c` | 清空输入 |
| `input_paste` | `ctrl+v` | 粘贴 |
| `input_undo` | `ctrl+-,cmd+z` | 撤销输入 |
| `input_redo` | `ctrl+.,cmd+shift+z` | 重做输入 |

**光标移动**：

| 键名 | 默认值 | 说明 |
|------|--------|------|
| `input_move_left` | `left,ctrl+b` | 左移 |
| `input_move_right` | `right,ctrl+f` | 右移 |
| `input_line_home` | `ctrl+a` | 行首 |
| `input_line_end` | `ctrl+e` | 行尾 |
| `input_word_forward` | `alt+f,alt+right` | 前进一单词 |
| `input_word_backward` | `alt+b,alt+left` | 后退一单词 |
| `input_buffer_home` | `home` | 缓冲区开头 |
| `input_buffer_end` | `end` | 缓冲区结尾 |

**选择**：

| 键名 | 默认值 | 说明 |
|------|--------|------|
| `input_select_left` | `shift+left` | 向左选择 |
| `input_select_right` | `shift+right` | 向右选择 |
| `input_select_word_forward` | `alt+shift+f` | 选择下一单词 |
| `input_select_word_backward` | `alt+shift+b` | 选择上一单词 |
| `input_select_line_home` | `ctrl+shift+a` | 选择到行首 |
| `input_select_line_end` | `ctrl+shift+e` | 选择到行尾 |
| `input_select_buffer_home` | `shift+home` | 选择到开头 |
| `input_select_buffer_end` | `shift+end` | 选择到结尾 |

**删除**：

| 键名 | 默认值 | 说明 |
|------|--------|------|
| `input_delete_line` | `ctrl+shift+d` | 删除整行 |
| `input_delete_to_line_end` | `ctrl+k` | 删除到行尾 |
| `input_delete_to_line_start` | `ctrl+u` | 删除到行首 |
| `input_delete_word_forward` | `alt+d,ctrl+delete` | 删除下一单词 |
| `input_delete_word_backward` | `ctrl+w,ctrl+backspace` | 删除上一单词 |

**其他**：

| 键名 | 默认值 | 说明 |
|------|--------|------|
| `session_rename` | `ctrl+r` | 重命名会话 |
| `session_delete` | `ctrl+d` | 删除会话 |
| `tips_toggle` | `<leader>h` | 切换提示 |

### 主题与外观

#### 内置主题

32 个内置主题：`opencode`、`system`、`tokyonight`、`catppuccin`、`catppuccin-macchiato`、`catppuccin-frappe`、`gruvbox`、`nord`、`everforest`、`ayu`、`kanagawa`、`one-dark`、`dracula`、`matrix`、`monokai`、`material`、`solarized`、`palenight`、`nightowl`、`rosepine`、`synthwave84`、`cobalt2`、`github`、`vercel`、`cursor`、`vesper`、`aura`、`flexoki`、`zenburn`、`mercury`、`orng`、`lucent-orng`、`osaka-jade`

切换：`/theme` 或 <kbd>Ctrl+X</kbd> → <kbd>T</kbd>

`system` 主题自动适配终端配色。

#### 自定义主题

主题加载优先级：内置 → 用户配置目录 → 项目根目录 → 当前工作目录。

创建 `~/.config/opencode/themes/my-theme.json`：

```jsonc
{
  "$schema": "https://opencode.ai/theme.json",
  "defs": { "bg": "#1a1b26", "fg": "#c0caf5" },
  "theme": {
    "primary": { "dark": "#7aa2f7", "light": "#3b7dd8" },
    "text": { "dark": "fg", "light": "#1a1a1a" },
    "background": { "dark": "bg", "light": "#ffffff" }
  }
}
```

颜色格式：Hex (`"#ffffff"`)、ANSI (`3`)、引用 (`"primary"`)、明暗变体 (`{"dark": "#000", "light": "#fff"}`)。

## 七、插件与工具系统

### 插件系统

#### 插件加载与创建

加载方式：
- **本地目录**：`.opencode/plugin/` 或 `~/.config/opencode/plugin/`
- **npm 包**：在 `opencode.json` 的 `plugin` 数组中指定
- **绝对路径**：开发模式，指向编译后的 `.js` 文件

```typescript
import type { Plugin } from "@opencode-ai/plugin"

export const MyPlugin: Plugin = async ({ project, client, $ }) => {
  return {
    event: async ({ event }) => {
      if (event.type === "session.idle") {
        await $`osascript -e 'display notification "完成"'`
      }
    }
  }
}
```

> [!warning] 导出规范
> 插件**只能导出 default 函数**，其他导出会导致加载失败。

#### 事件钩子

使用 `event` 统一订阅所有事件：`session.idle`、`session.created`、`file.edited`、`message.updated`、`permission.replied` 等 35+ 种事件。

#### 功能钩子

| 钩子 | 触发时机 | 可修改数据 |
|------|---------|-----------|
| `config` | 配置加载后 | ✅ |
| `chat.message` | 新消息接收 | ✅ |
| `chat.params` | LLM 调用前 | ✅ 温度/top_p |
| `permission.ask` | 权限请求 | ✅ |
| `tool.execute.before` | 工具执行前 | ✅ 可拦截 |
| `tool.execute.after` | 工具执行后 | ✅ |
| `tool.definition` | 工具注册时（v1.1.65+） | ✅ |
| `command.execute.before` | 命令执行前（v1.1.65+） | ✅ |
| `shell.env` | Shell 执行前（v1.1.65+） | ✅ 注入环境变量 |
| `auth` | 认证流程 | - |

实验性钩子：`experimental.session.compacting`、`experimental.chat.messages.transform`、`experimental.chat.system.transform`

#### 自定义工具

工具放在 `.opencode/tool/` 或 `~/.config/opencode/tool/`：

```typescript
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "查询数据库",
  args: { query: tool.schema.string().describe("SQL 查询语句") },
  async execute(args, context) {
    return `查询结果: ${args.query}`
  }
})
```

支持单文件多工具（`math_add`、`math_multiply`）、与内置工具同名时自定义优先。输出限制：最大 2000 行或 50KB。

#### Python 编写工具示例

Python 脚本：
```python
# .opencode/tool/add.py
import sys
a = int(sys.argv[1])
b = int(sys.argv[2])
print(a + b)
```

TypeScript 调用 Python：
```typescript
// .opencode/tool/python-add.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Add two numbers using Python",
  args: {
    a: tool.schema.number().describe("First number"),
    b: tool.schema.number().describe("Second number"),
  },
  async execute(args) {
    const result = await Bun.$`python3 .opencode/tool/add.py ${args.a} ${args.b}`.text()
    return result.trim()
  },
})
```

#### HTTP API 调用工具示例

```typescript
// .opencode/tool/jira.ts
import { tool } from "@opencode-ai/plugin"

export const getIssue = tool({
  description: "Get JIRA issue details by key",
  args: {
    key: tool.schema.string().describe("Issue key, e.g. PROJ-123"),
  },
  async execute(args, context) {
    const response = await fetch(
      `https://your-company.atlassian.net/rest/api/3/issue/${args.key}`,
      {
        headers: {
          Authorization: `Basic ${btoa("email:API_TOKEN")}`,
          Accept: "application/json",
        },
        signal: context.abort,
      }
    )
    if (!response.ok) {
      throw new Error(`Failed: ${response.status}`)
    }
    const issue = await response.json()
    return JSON.stringify(issue, null, 2)
  },
})
```

#### Hook 配置

快速参考：

```typescript
// 敏感文件拦截
"tool.execute.before": async (input, output) => {
  if (input.tool === "read" && String(output.args.filePath).includes(".env")) {
    throw new Error("禁止读取 .env 文件")
  }
}

// 按 Agent 调参数
"chat.params": async (input, output) => {
  if (input.agent === "code") output.temperature = 0.2
}
```

#### 插件上下文参数（6 个）

| 参数 | 类型 | 说明 |
|------|------|------|
| `project` | `Project` | 当前项目信息 |
| `directory` | `string` | 当前工作目录 |
| `worktree` | `string` | Git 工作树路径 |
| `client` | `OpencodeClient` | OpenCode SDK 客户端 |
| `$` | `BunShell` | Bun 的 shell API，用于执行命令 |
| `serverUrl` | `URL` | OpenCode 服务器 URL |

#### 内置插件（2 个）

| 插件 | 功能 |
|------|------|
| `opencode-copilot-auth` | GitHub Copilot 认证 |
| `opencode-anthropic-auth` | Anthropic 认证 |

可通过 `OPENCODE_DISABLE_DEFAULT_PLUGINS=1` 禁用。

#### 插件加载顺序（4 级）

1. 全局配置（`~/.config/opencode/opencode.json`）
2. 项目配置（`opencode.json`）
3. 全局插件目录（`~/.config/opencode/plugin/`）
4. 项目插件目录（`.opencode/plugin/`）

#### 实验性钩子 experimental.text.complete

```typescript
{
  "experimental.text.complete": async (input, output) => {
    // input: { sessionID, messageID, partID }
    // output: { text }
    output.text = output.text.replace(/\bAI\b/g, "Assistant")
  }
}
```

### 内置工具

#### 17 个内置工具

| 工具 | 功能 | 条件 |
|------|------|------|
| `bash` | 执行 shell 命令 | 默认启用 |
| `edit` | 精确字符串替换 | 默认启用，需先 read |
| `write` | 创建/覆盖文件 | 默认启用，已存在需先 read |
| `read` | 读取文件或目录 | 默认启用，2000行/50KB限制 |
| `grep` | 正则搜索内容 | 默认启用，100条结果限制 |
| `glob` | 模式匹配文件 | 默认启用 |
| `list` | 列出目录内容 | 默认启用 |
| `patch` | 应用补丁 | 默认启用 |
| `multiedit` | 批量修改文件 | 默认启用 |
| `skill` | 加载技能 | 默认启用 |
| `todowrite` | 管理待办 | 默认启用（子代理禁用） |
| `todoread` | 读取待办 | 默认启用（子代理禁用） |
| `question` | 向用户提问 | 默认启用 |
| `webfetch` | 获取网页内容 | 默认启用，5MB限制 |
| `websearch` | 搜索互联网 | 需 Zen 或 `OPENCODE_ENABLE_EXA=true` |
| `codesearch` | 代码搜索 | 需 Zen 或 `OPENCODE_ENABLE_EXA=true` |
| `batch` | 并行执行多工具（实验性） | 需 `experimental.batch_tool: true` |
| `lsp` | LSP 代码智能（实验性） | 需 `OPENCODE_EXPERIMENTAL_LSP_TOOL` |

#### edit 工具匹配策略

edit 有 9 层匹配策略，从严格到宽松：SimpleReplacer → LineTrimmedReplacer → BlockAnchorReplacer → WhitespaceNormalizedReplacer → IndentationFlexibleReplacer → EscapeNormalizedReplacer → TrimmedBoundaryReplacer → ContextAwareReplacer → MultiOccurrenceReplacer。

#### 搜索工具组合使用

1. **先 glob 找文件，再 grep 搜内容**
2. **codesearch 找示例，grep 找本地实现**

> grep 和 glob 有 100 条结果限制，codesearch 有 30 秒超时。

#### 工具 AI 调用参数表

#### bash 工具参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `command` | string | 是 | 要执行的命令 |
| `description` | string | 是 | 命令描述（5-10 词）|
| `workdir` | string | 否 | 工作目录 |
| `timeout` | number | 否 | 超时（毫秒），默认 120000 |

#### edit 工具参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `filePath` | string | 是 | 文件绝对路径 |
| `oldString` | string | 是 | 要替换的文本 |
| `newString` | string | 是 | 替换后的文本 |
| `replaceAll` | boolean | 否 | 替换所有匹配（默认 false）|

前置条件：必须先用 `read` 读取文件。

#### write 工具参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `filePath` | string | 是 | 文件绝对路径 |
| `content` | string | 是 | 文件内容 |

#### read 工具参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `filePath` | string | 是 | 文件或目录绝对路径 |
| `offset` | number | 否 | 起始行号（1-indexed）|
| `limit` | number | 否 | 最多读取行数（默认 2000）|

#### grep 工具参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `pattern` | string | 是 | 正则表达式 |
| `path` | string | 否 | 搜索路径 |
| `include` | string | 否 | 文件过滤，如 `"*.ts"` |

结果限制：最多 100 个匹配。

#### webfetch 工具参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `url` | string | 是 | 完整 URL（http/https）|
| `format` | string | 否 | `markdown`/`text`/`html`，默认 markdown |
| `timeout` | number | 否 | 超时秒数（最大 120）|

#### codesearch 工具参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `query` | string | 是 | 搜索查询 |
| `tokensNum` | number | 否 | 返回 token 数量（默认 2000+）|

#### batch 工具（实验性）

```typescript
{
  tool_calls: [
    { tool: "工具名", parameters: { /* 参数 */ } },
    // 最多 25 个并行调用
  ]
}
```

限制：不支持 MCP 工具，禁止嵌套 batch。

### 高级功能

#### 格式化器

OpenCode 在文件写入后自动格式化。内置支持：gofmt、prettier、biome、zig、clang-format、ktlint、ruff、rustfmt、rubocop、standardrb 等 20+ 种。

```jsonc
{
  "formatter": {
    "prettier": {
      "command": ["npx", "prettier", "--write", "$FILE"],
      "extensions": [".js", ".ts"]
    }
  }
}
```

#### 格式化器完整列表（21 个）

| 格式化器 | 扩展名 | 要求 |
|----------|--------|------|
| gofmt | `.go` | `gofmt` 命令可用 |
| mix | `.ex`, `.exs` | `mix` 命令可用 |
| prettier | `.js`, `.tsx`, `.ts`, `.css`, `.md`, `.json` 等 | `package.json` 有 `prettier` 依赖 |
| biome | 同 prettier | 存在 `biome.json` 配置文件 |
| zig | `.zig`, `.zon` | `zig` 命令可用 |
| clang-format | `.c`, `.cpp`, `.h`, `.hpp` 等 | 存在 `.clang-format` 配置 |
| ktlint | `.kt`, `.kts` | `ktlint` 命令可用 |
| ruff | `.py`, `.pyi` | `ruff` 命令可用 |
| rustfmt | `.rs` | `rustfmt` 命令可用 |
| uv | `.py`, `.pyi` | `uv` 命令可用 |
| rubocop | `.rb`, `.rake`, `.gemspec` 等 | `rubocop` 命令可用 |
| standardrb | `.rb`, `.rake`, `.gemspec` 等 | `standardrb` 命令可用 |
| htmlbeautifier | `.erb`, `.html.erb` | `htmlbeautifier` 命令可用 |
| air | `.R` | `air` 命令可用 |
| dart | `.dart` | `dart` 命令可用 |
| ocamlformat | `.ml`, `.mli` | `ocamlformat` 命令可用 |
| terraform | `.tf`, `.tfvars` | `terraform` 命令可用 |
| gleam | `.gleam` | `gleam` 命令可用 |
| nixfmt | `.nix` | `nixfmt` 命令可用 |
| shfmt | `.sh`, `.bash` | `shfmt` 命令可用 |
| oxfmt（实验性）| `.js`, `.jsx`, `.ts`, `.tsx` | `package.json` 有 `oxfmt` 依赖 |

#### LSP 代码智能

30+ 内置语言服务器（TypeScript、Python pyright、Go gopls、Rust rust-analyzer 等），9 种操作：
- goToDefinition / findReferences / hover
- documentSymbol / workspaceSymbol
- goToImplementation
- prepareCallHierarchy / incomingCalls / outgoingCalls

大部分开箱即用，无需配置。

#### 内置语言服务器完整列表（35+）

**主流语言**：

| LSP 服务器 | 扩展名 | 要求 |
|-----------|--------|------|
| typescript | `.ts`, `.tsx`, `.js`, `.jsx` | 项目有 `typescript` 依赖 |
| pyright | `.py`, `.pyi` | 自动安装 |
| gopls | `.go` | `go` 命令可用 |
| rust-analyzer | `.rs` | `rust-analyzer` 命令可用 |
| jdtls | `.java` | 已安装 Java SDK（21+）|
| kotlin-ls | `.kt`, `.kts` | 自动下载安装 |
| clangd | `.c`, `.cpp`, `.cc`, `.h`, `.hpp` 等 | 自动下载安装 |
| csharp-ls | `.cs` | 已安装 .NET SDK |
| fsautocomplete | `.fs`, `.fsi`, `.fsx` | 已安装 .NET SDK |
| sourcekit-lsp | `.swift`, `.objc` | macOS 上为 Xcode |
| dart | `.dart` | `dart` 命令可用 |

**其他语言**：

| LSP 服务器 | 扩展名 |
|-----------|--------|
| ruby-lsp | `.rb` |
| elixir-ls | `.ex`, `.exs` |
| zls | `.zig` |
| lua-ls | `.lua` |
| intelephense | `.php` |
| ocaml-lsp | `.ml` |
| gleam-lsp | `.gleam` |
| clojure-lsp | `.clj`, `.cljs` |
| nixd | `.nix` |
| haskell-language-server | `.hs`, `.lhs` |
| deno | `.ts`, `.js`（需 `deno.json`）|

**前端框架**：

| LSP 服务器 | 扩展名 |
|-----------|--------|
| vue | `.vue` |
| svelte | `.svelte` |
| astro | `.astro` |

**工具和配置**：

| LSP 服务器 | 扩展名 | 用途 |
|-----------|--------|------|
| eslint | `.ts`, `.tsx`, `.js`, `.jsx` 等 | 代码规范检查 |
| oxlint | `.ts`, `.tsx`, `.js`, `.jsx` 等 | 快速 linter |
| biome | `.ts`, `.tsx`, `.json`, `.css` 等 | 格式化 + linter |
| yaml-ls | `.yaml`, `.yml` | YAML 支持 |
| bash | `.sh`, `.bash` | Shell 脚本 |
| terraform | `.tf` | IaC 配置 |
| prisma | `.prisma` | 数据库 schema |
| texlab | `.tex`, `.bib` | LaTeX |
| tinymist | `.typ` | Typst 排版 |
| dockerfile | `Dockerfile` | Docker |

> 设置 `OPENCODE_EXPERIMENTAL_LSP_TY=1` 启用实验性 ty Python 服务器（替代 pyright）。

#### 思考深度配置

通过模型变体控制思考预算，用 <kbd>Ctrl</kbd>+<kbd>T</kbd> 切换：

```jsonc
{
  "provider": {
    "anthropic": {
      "models": {
        "claude-sonnet-4-5": {
          "variants": {
            "high": { "thinking": { "type": "enabled", "budgetTokens": 20000 } },
            "max": { "thinking": { "type": "enabled", "budgetTokens": 32000 } }
          }
        }
      }
    }
  }
}
```

#### 调试与诊断

```bash
opencode debug config           # 查看最终配置（含插件注入）
opencode debug skill            # 列出实际加载的 Skill
opencode debug agent <name>     # Agent 详情
opencode debug paths            # 关键路径
opencode debug lsp diagnostics <file>  # LSP 诊断
opencode debug rg search "关键词"       # 搜索调试
```

#### 网络搜索与获取

| 工具 | 功能 | 启用条件 |
|------|------|---------|
| `webfetch` | 获取网页内容 | 默认可用 |
| `websearch` | 搜索互联网 | Zen 或 `OPENCODE_ENABLE_EXA=true` |

webfetch 参数：`format`（markdown/text/html）、`timeout`（30-120秒）、`url`（必须 http/https）。

#### Git Worktree

`git worktree` 让你在同一个仓库中同时检出多个分支：

```bash
git worktree add ../my-project-hotfix hotfix/login
git worktree list
git worktree remove ../my-project-hotfix
```

OpenCode 支持 `/workspaces` 快速切换（需 `OPENCODE_EXPERIMENTAL_WORKSPACES=1`）。

#### 规则与指令

| 作用域 | 位置 | 说明 |
|--------|------|------|
| 全局规则 | `~/.config/opencode/AGENTS.md` | 所有项目 |
| 项目规则 | 项目根目录 `AGENTS.md` | 项目特定 |
| 配置文件 | `opencode.json` 的 `instructions` | 引用多文件 |

`/init` 让 AI 分析项目自动生成 `AGENTS.md`。

`AGENTS.md` 中用 `@docs/xxx.md` 语法引用外部文件（按需加载）。

> [!important] instructions 路径列表在会话启动时加载，修改后需新会话生效。文件内容本身是热加载的。

## 八、实战与企业

### CI/CD 集成

#### GitHub 集成

```bash
opencode github install    # 交互式安装
```

在 `.github/workflows/opencode.yml`：

```yaml
name: opencode
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  opencode:
    if: |
      contains(github.event.comment.body, ' /oc') ||
      startsWith(github.event.comment.body, '/oc')
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: write
      pull-requests: write
      issues: read
    steps:
      - uses: actions/checkout@v6
      - uses: anomalyco/opencode/github@latest
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          model: anthropic/claude-sonnet-4-20250514
```

触发词：`/oc` 或 `/opencode`（行首或前面有空格）。

| 配置选项 | 说明 |
|---------|------|
| `model` | 必填，使用的模型 |
| `agent` | 使用的 Agent |
| `share` | 是否分享会话 |
| `prompt` | 自定义提示（schedule/dispatch 必填） |
| `use_github_token` | 使用 GITHUB_TOKEN 替代 OIDC |
| `mentions` | 自定义触发短语 |

分支命名：`opencode/issue{ID}-{timestamp}`、`opencode/pr{ID}-{timestamp}`。

#### GitHub 集成完整事件列表（6 种）

| 事件类型 | 触发方式 | 说明 |
|----------|----------|------|
| `issue_comment` | Issue 或 PR 上的评论 | 在评论中提及 `/opencode` 或 `/oc` |
| `pull_request_review_comment` | PR 中特定代码行的评论 | 代码审查时提及触发短语 |
| `issues` | Issue 创建或编辑 | 需要 `prompt` 输入 |
| `pull_request` | PR 创建或更新 | 用于自动审查 |
| `schedule` | 基于 cron 的定时任务 | 需要 `prompt` 输入 |
| `workflow_dispatch` | 从 GitHub UI 手动触发 | 需要 `prompt` 输入 |

#### 处理流程（7 步）

1. 事件触发
2. 检查触发短语（/opencode 或 /oc）
3. 获取访问令牌（OIDC 交换或 GITHUB_TOKEN）
4. 权限验证（仅用户事件，需要 admin 或 write 权限）
5. 添加 👀 reaction（仅用户事件，表示正在处理）
6. 根据事件类型处理：
   - Issue：创建新分支 → 执行任务 → 提交 → 创建 PR
   - 本地 PR：检出分支 → 执行任务 → 提交到同一 PR
   - Fork PR：添加 fork remote → 执行任务 → 推送到 fork
   - 仓库事件：创建新分支 → 执行任务 → 创建 PR
7. 创建评论并移除 reaction

#### Fork PR 处理方式

| 对比项 | 本地 PR | Fork PR |
|--------|---------|---------|
| 分支来源 | 同一仓库 | Fork 仓库 |
| 检出方式 | `git fetch origin && git checkout` | `git remote add fork && git fetch fork` |
| 推送目标 | 原分支 | Fork 仓库的分支 |

#### Co-author 归属

提交自动添加 Co-author 信息：
```
Co-authored-by: username <username@users.noreply.github.com>
```

> `schedule` 事件没有触发者，不添加 Co-author 信息。

#### GitHub 其他配置选项

| 选项 | 必填 | 说明 |
|------|------|------|
| `oidc_base_url` | 否 | 自定义 OIDC 令牌交换 API 地址，仅运行私有 GitHub App 时需要 |

#### GitLab 集成

使用社区 CI/CD 组件 `nagyv/gitlab-opencode@2`：

```yaml
include:
  - component: $CI_SERVER_FQDN/nagyv/gitlab-opencode/opencode@2
    inputs:
      config_dir: ${CI_PROJECT_DIR}/opencode-config
      auth_json: $OPENCODE_AUTH_JSON
      message: "Your prompt here"
```

认证信息存储为 **File 类型** 的 CI/CD 环境变量。

#### 会话分享

```jsonc
{ "share": "manual" }    // 手动（默认）
{ "share": "auto" }      // 自动分享
{ "share": "disabled" }  // 禁用
```

`/share` 生成链接，`/unshare` 取消。分享链接格式：`opncd.ai/s/<share-id>`

### 企业功能

#### 企业版特性

| 功能 | 开源版 | 企业版 |
|------|--------|--------|
| 核心功能 | ✅ | ✅ |
| 本地处理 | ✅ | ✅ |
| 中央配置管理 | ❌ | ✅ |
| SSO 集成 | ❌ | ✅ |
| 内部 AI 网关 | ❌ | ✅ |
| 优先支持 | ❌ | ✅ |

OpenCode ==不存储你的代码或上下文数据==，所有处理都在本地进行。企业试用建议禁用分享功能。

#### 企业认证集成（三种方式）

| 方式 | 适用场景 | 复杂度 |
|------|---------|--------|
| **环境变量** | 固定 Key / K8s/CI 注入 | 最低 |
| **well-known** | 统一登录 + 下发默认配置 | 中 |
| **认证插件** | OAuth/刷新/签名/改 headers | 最高 |

#### 方式 A：环境变量

```jsonc
{
  "provider": {
    "corp-gateway": {
      "name": "Company AI Gateway",
      "env": ["COMPANY_AI_TOKEN"],
      "api": "https://ai-gateway.company.internal/v1",
      "npm": "@ai-sdk/openai-compatible",
      "models": { "qwen2.5-72b": { "name": "Qwen 2.5 72B" } }
    }
  },
  "model": "corp-gateway/qwen2.5-72b"
}
```

#### 方式 B：well-known

```bash
opencode auth login https://ai.company.internal
```

服务端返回 `/.well-known/opencode`：

```jsonc
{
  "auth": {
    "command": ["corpctl", "ai", "token", "--format=plain"],
    "env": "COMPANY_AI_TOKEN"
  },
  "config": {
    "provider": { "corp-gateway": { "...": "..." } },
    "model": "corp-gateway/qwen2.5-72b"
  }
}
```

> [!warning] Token 续期
> well-known 的 `auth.command` 只在 `opencode auth login` 时运行一次。短期 Token 需要走插件方式。

#### 方式 C：认证插件

```typescript
const plugin: Plugin = async () => ({
  auth: {
    provider: "corp-gateway",
    methods: [{
      type: "api",
      label: "Login with SSO",
      async authorize() {
        const token = "<replace-me>"
        return { type: "success", key: token }
      }
    }]
  }
})
```

#### enterprise.url 配置

```jsonc
{
  "enterprise": {
    "url": "https://your-company.opencode.internal"
  }
}
```

#### 私有 NPM 注册表配置

登录方式：
```bash
npm login --registry=https://your-company.jfrog.io/api/npm/npm-virtual/
```

手动配置 `~/.npmrc`：
```bash
registry=https://your-company.jfrog.io/api/npm/npm-virtual/
//your-company.jfrog.io/api/npm/npm-virtual/:_authToken=${NPM_AUTH_TOKEN}
```

> OpenCode 通过 Bun 原生支持 `.npmrc` 文件。开发者必须先登录私有注册表，再运行 OpenCode。

#### well-known 完整 JSON 结构详解

```jsonc
{
  "auth": {
    "command": ["corpctl", "ai", "token", "--format=plain"],
    "env": "COMPANY_AI_TOKEN"
  },
  "config": {
    "disabled_providers": ["openai", "anthropic", "openrouter"],
    "provider": {
      "corp-gateway": {
        "name": "Company AI Gateway",
        "env": ["COMPANY_AI_TOKEN"],
        "api": "https://ai-gateway.company.internal/v1",
        "npm": "@ai-sdk/openai-compatible",
        "models": {
          "qwen2.5-72b": {
            "name": "Qwen 2.5 72B",
            "limit": { "context": 128000, "output": 8192 }
          }
        }
      }
    },
    "model": "corp-gateway/qwen2.5-72b"
  }
}
```

> **重要**：`auth.command` 只在你执行 `opencode auth login` 时运行一次，并把 token 保存到本机。后续启动只会注入已保存的 token，不会自动刷新/续期。短期 Token 需要走插件方式。

#### token 注入机制

- `auth.env` 是环境变量名（如 `COMPANY_AI_TOKEN`）
- OpenCode 启动时会在**当前进程**里设置 `process.env[auth.env]=token`
- **不会**写入系统的 `.zshrc/.bashrc`，所以另一个终端 `echo $COMPANY_AI_TOKEN` 看不到值

#### 配置合并优先级（详细版）

1. 远程 `/.well-known/opencode`（组织默认）
2. 全局配置（`~/.config/opencode/`）
3. 自定义配置路径（`OPENCODE_CONFIG`）
4. 项目配置（`opencode.json`）
5. `.opencode/` 目录配置
6. 内联配置（`OPENCODE_CONFIG_CONTENT`）
7. （企业版）managed config 目录覆盖所有

> `plugin`/`instructions` 等数组字段是**合并并去重**（不是覆盖），插件还会额外按插件名去重。

### 场景应用

#### 开发工作流

```text
@explore 帮我梳理项目整体结构
@plan 设计邮件通知功能方案
按照方案一实现邮件通知功能
@src/hooks/useAuth.ts 修复登录后 loading 问题
```

#### 重构安全三步法

1. 先识别代码坏味道
2. 先生成测试（作为安全网）
3. 小步重构 → 测试通过

```text
!npm test src/utils/data.test.ts
```

> 以 `!` 开头的消息会执行 shell 命令并把输出带进对话。

#### 文档自动化

| 文档类型 | AI 辅助方式 |
|---------|------------|
| README | 分析项目结构生成 |
| API 文档 | 从代码提取生成 |
| Commit 消息 | 分析 `!git diff` 生成 |
| PR 描述 | 汇总 `!git log` 生成 |

#### 专属开发 Agent

```markdown
---
description: 严格的代码审查专家
mode: subagent
model: anthropic/claude-opus-4-5-thinking
temperature: 0.3
permission:
  edit: deny
  bash: deny
---

# Code Reviewer Agent

## 审查清单
- [ ] 函数职责单一
- [ ] 命名清晰准确
- [ ] 边界条件处理
- [ ] 错误处理完整
```

组合命令 `.opencode/command/全面审查.md`：

```markdown
---
description: 综合代码审查
---

请依次执行：
1. @code-reviewer 审查代码质量
2. @security-auditor 检查安全隐患
3. @test-writer 分析测试覆盖率

目标文件：$ARGUMENTS
```

#### Prompt 工程规范

好的 Agent prompt 结构：

```markdown
# 角色定义    ← 你是谁、擅长什么
# 工作流程    ← 步骤 1/2/3
# 约束条件    ← 不要做什么
# 输出格式    ← 如何组织输出
# 自我检查    ← 完成前确认（可选）
```

#### Prompt 工程 7 条核心原则

1. **结构化优先（Structure First）**：每个模板必须包含明确的结构层次：`角色定义 → 任务说明 → 输入规范 → 输出规范 → 示例 → 约束条件`
2. **Few-shot 必备（Example-Driven）**：复杂任务必须提供 1-2 个具体示例，让模型理解预期输入/输出格式
3. **显式思维链（Chain of Thought）**：对于推理类任务，明确引导"先分析，再输出"的步骤式思考
4. **量化评估标准（Quantified Criteria）**：任何评分/评估必须有明确的评分维度和每个分数段的定义
5. **错误处理机制（Graceful Degradation）**：模板应包含：输入不完整时如何处理、遇到歧义时如何澄清、超出能力范围时如何反馈
6. **自检机制（Self-Verification）**：复杂任务输出前进行自我检查，确保符合要求
7. **占位符一致性（Consistent Placeholders）**：全文档使用统一的占位符语法，清晰区分必填和选填

#### 占位符规范语法（5 种类型）

| 类型 | 语法 | 示例 |
|------|------|------|
| **必填变量** | `【变量名】` | `【编程语言】` |
| **选填变量** | `【变量名?】` | `【框架?】` |
| **带默认值** | `【变量名=默认值】` | `【风格=轻松】` |
| **多选一** | `【选项1/选项2/选项3】` | `【玄幻/都市/言情】` |
| **文本块** | `[粘贴区域描述]` | `[粘贴代码]` |

#### Agent/Skill description 规范模板

```yaml
description: |
  [一句话说明核心能力]
  提供：[该 Skill 包含的资源]
  适用：[触发场景1]、[触发场景2]
  不适用：[边界场景1]、[边界场景2]
```

良好的 Agent prompt 结构：

```markdown
## 角色
你是[专业身份]，擅长[核心能力]

## 任务
[清晰描述要完成的任务]

## 输入信息
### 必填项
- 变量名：【变量说明】

## 输出规范
按以下结构输出：1. [输出项1] 2. [输出项2] ...

## 约束条件
- ✅ 应该：[正面行为]
- ❌ 避免：[负面行为]

## 自检清单
## 错误处理
```

#### Prompt 模板库（13 个）

#### 写作类（7 个）

| 模板 | 用途 |
|------|------|
| 小说大纲生成 | 根据类型/篇幅/风格生成完整小说大纲 |
| 角色卡生成 | 根据姓名/身份生成完整角色卡 |
| 对话润色 | 润色对话文本，增加潜台词、情感层次 |
| 黄金三章生成 | 按"黄金三章"法则生成网文开头 |
| AIDA 营销文案 | 用 AIDA 模型写高转化率营销文案 |
| 专业翻译 | 保持原意和专业术语一致性的高质量翻译 |
| 降重改写 | 三个改写程度（轻度30%/中度50%/重度70%）|

#### 开发类（6 个）

| 模板 | 用途 |
|------|------|
| 代码解释 | 逐块解释代码功能+原理+关键概念 |
| 功能实现 | 将需求描述转化为完整可运行代码+注释+示例+单元测试 |
| Bug 定位 | 按可能性排序的根因分析+验证方法+三层修复方案 |
| 代码审查 | 四维度审查（代码质量/潜在问题/安全隐患/最佳实践）+ A-D评级 |
| 测试用例生成 | AAA模式覆盖正常流程+边界条件+异常+空值处理 |
| README 生成 | 生成结构化项目文档 |
| Commit 消息 | 根据 git diff 生成符合 Conventional Commits 规范的提交消息 |

#### 通用类（4 个）

| 模板 | 用途 |
|------|------|
| 内容总结 | 提炼关键信息：一句话总结+核心要点+关键数据+行动项 |
| 会议纪要 | 结构化会议内容：基本信息+议题结论表+待办事项表 |
| 邮件撰写 | 根据类型/背景/目的/关系撰写商务邮件 |
| 学习笔记 | 整理知识：概念定义+核心原理+关键点+常见误区 |

## 九、参考资料与附录

### 参考资料

#### 模型提供商列表

OpenCode 基于 AI SDK 和 Models.dev 支持 75+ 模型提供商。添加方式：`/connect` 命令。

**OpenCode Zen**（推荐新手）：官方精选模型服务，开箱即用。访问 [opencode.ai/auth](https://opencode.ai/auth) 获取 API Key。

#### 故障排除

**日志位置**：`~/.local/share/opencode/log/`

```bash
opencode --log-level DEBUG --print-logs
```

**常见错误**：
- API 401/403 → 检查 API Key 是否有效
- Context overflow → 使用 `/compact` 压缩上下文
- Permission denied → 检查 `permission` 配置
- MCP 连接失败 → `opencode mcp debug <name>`
- LSP 不工作 → `opencode debug lsp diagnostics <file>`

#### 错误代码速查表

**HTTP 错误**：

| 错误码 | 含义 | 自动重试 |
|--------|------|---------|
| 400 | 请求格式错误 | ❌ |
| 401 | 认证失败（Key 无效/过期）| ❌ |
| 403 | 权限不足 | ❌ |
| 404 | 模型不存在 | ❌ |
| 413 | 请求体过大（上下文溢出）| ❌ |
| 429 | 请求过多（速率限制）| ✅ |
| 503 | 服务不可用（过载/维护）| ✅ |

**OpenCode 内部错误**：

| 错误类型 | 含义 | 解决方案 |
|---------|------|---------|
| `ContextOverflowError` | 上下文溢出 | `/compact` 压缩 |
| `ProviderInitError` | 提供商初始化失败 | 检查配置和认证 |
| `ProviderModelNotFoundError` | 模型不存在 | 检查格式 `provider/model` |
| `RejectedError` | 用户拒绝权限 | 权限提示时按 `y` |
| `DeniedError` | 配置规则拒绝操作 | 检查 `permission` 配置 |

#### API 自动重试机制

重试策略：初始 2 秒延迟，每次重试延迟翻倍（2x 退避），最大 30 秒（无 `Retry-After` 头时）。如果提供商返回 `Retry-After` 头，按指定时间等待，不受 30 秒上限限制。

#### 输出截断规则

| 限制 | 值 |
|------|-----|
| 最大行数 | 2,000 行 |
| 最大字节数 | 50 KB |
| 完整文件保留时间 | 7 天 |

截断后处理方式：
1. 用 Grep 工具搜索截断输出中的特定内容
2. 用 Read 工具的 `offset`/`limit` 分段查看
3. 委托给子 Agent（Task 工具）处理

#### 上下文溢出错误信息（按提供商）

| 提供商 | 错误信息特征 |
|--------|-----------|
| Anthropic | `prompt is too long` |
| OpenAI | `exceeds the context window` |
| Google Gemini | `input token number exceeds the maximum` |
| DeepSeek / OpenRouter | `maximum context length is X tokens` |
| Groq | `reduce the length of the messages` |
| Amazon Bedrock | `input is too long for requested model` |
| xAI | `maximum prompt length is X` |
| llama.cpp | `exceeds the available context size` |

#### 快速诊断三连命令

```bash
opencode --version
opencode auth list
opencode --log-level DEBUG --print-logs
```

#### FAQ

#### 提供商完整分类列表

#### 国产模型（4 个）

| 提供商 | 代表模型 | 获取 API Key |
|--------|---------|-------------|
| DeepSeek（深度求索） | `deepseek-chat`、`deepseek-reasoner` | platform.deepseek.com |
| Moonshot AI（月之暗面） | `kimi-k2` | platform.moonshot.ai |
| MiniMax | `M2.7` | platform.minimax.io |
| Z.AI（智谱） | `GLM-5` | z.ai |

#### 国际模型（8 个）

| 提供商 | 代表模型 |
|--------|---------|
| Anthropic Claude | `claude-sonnet-4`、`claude-opus-4`、`claude-3-5-haiku` |
| OpenAI | `gpt-4o`、`gpt-4o-mini`、`o1`、`o3-mini` |
| Google Gemini | `gemini-2.0-flash`、`gemini-2.0-pro` |
| xAI Grok | `grok-2`、`grok-2-mini` |
| Mistral | `mistral-large`、`mistral-small`、`codestral` |
| Cohere | 企业级 NLP（Rerank/Embed）|
| Perplexity | 集成搜索能力 |
| GitHub Copilot | 使用 Copilot 订阅 |

#### 云平台（5 个）

| 平台 | 认证方式 |
|------|---------|
| Amazon Bedrock | AWS_PROFILE 或 opencode.json 配置 |
| Azure OpenAI | `/connect` 配置 |
| Azure Cognitive Services | `/connect` + AZURE_COGNITIVE_SERVICES_RESOURCE_NAME |
| Cloudflare Workers AI | `/connect` + API Token |
| GitLab Duo | `/connect` + GitLab Access Token |

#### 聚合平台（19 个）

OpenRouter、Groq、Cerebras、Fireworks AI、Deep Infra、Together AI、Hugging Face、Baseten、Cortecs、Nebius、IO.NET、Venice AI、OVHcloud、SAP AI Core、Cloudflare AI Gateway、Vercel AI Gateway、Helicone、ZenMux、Ollama Cloud

#### 本地模型（3 个）

| 运行时 | 默认 baseURL |
|--------|-------------|
| Ollama | `http://localhost:11434/v1` |
| LM Studio | `http://127.0.0.1:1234/v1` |
| llama.cpp | `http://127.0.0.1:8080/v1` |

#### 模型选择推荐

| 需求 | 推荐 |
|------|------|
| 国内使用最简单 | DeepSeek |
| 最强推理能力 | Claude Opus 4 |
| 代码能力最强 | Claude Sonnet 4 |
| 长文档处理 | Gemini 1.5 Pro |
| 完全离线 | Ollama + Llama3.1 |
| 多模型切换 | OpenRouter |

#### OpenCode Zen 模型列表（24 个）

| 模型 | 模型 ID | 端点 |
|------|---------|------|
| GPT 5.2 | `gpt-5.2` | `/zen/v1/responses` |
| GPT 5.1 | `gpt-5.1` | `/zen/v1/responses` |
| GPT 5.1 Codex | `gpt-5.1-codex` | `/zen/v1/responses` |
| GPT 5.1 Codex Max | `gpt-5.1-codex-max` | `/zen/v1/responses` |
| GPT 5.1 Codex Mini | `gpt-5.1-codex-mini` | `/zen/v1/responses` |
| GPT 5 | `gpt-5` | `/zen/v1/responses` |
| GPT 5 Codex | `gpt-5-codex` | `/zen/v1/responses` |
| GPT 5 Nano | `gpt-5-nano` | `/zen/v1/responses` |
| Claude Sonnet 4.5 | `claude-sonnet-4-5` | `/zen/v1/messages` |
| Claude Sonnet 4 | `claude-sonnet-4` | `/zen/v1/messages` |
| Claude Haiku 4.5 | `claude-haiku-4-5` | `/zen/v1/messages` |
| Claude Haiku 3.5 | `claude-3-5-haiku` | `/zen/v1/messages` |
| Claude Opus 4.5 | `claude-opus-4-5` | `/zen/v1/messages` |
| Claude Opus 4.1 | `claude-opus-4-1` | `/zen/v1/messages` |
| MiniMax M2.7 | `minimax-m2.7-free` | `/zen/v1/messages` |
| Gemini 3 Pro | `gemini-3-pro` | `/zen/v1/models/gemini-3-pro` |
| Gemini 3 Flash | `gemini-3-flash` | `/zen/v1/models/gemini-3-flash` |
| GLM 4.6 | `glm-4.6` | `/zen/v1/chat/completions` |
| GLM 4.7 | `glm-4.7-free` | `/zen/v1/chat/completions` |
| Kimi K2 | `kimi-k2` | `/zen/v1/chat/completions` |
| Kimi K2 Thinking | `kimi-k2-thinking` | `/zen/v1/chat/completions` |
| Qwen3 Coder 480B | `qwen3-coder` | `/zen/v1/chat/completions` |
| Grok Code Fast 1 | `grok-code` | `/zen/v1/chat/completions` |
| Big Pickle | `big-pickle` | `/zen/v1/chat/completions` |

> 配置中模型 ID 格式：`opencode/<model-id>`。获取 API Key：[opencode.ai/auth](https://opencode.ai/auth)

- 安装报错 → 检查 PATH 环境变量，Windows 用 Scoop
- 配置不生效 → 检查优先级和键名拼写
- 模型切换 → `/models` 或 `F2` 快捷键
- 撤销文件操作 → 需要项目是 Git 仓库

#### 生态系统

**社区插件**：opencode-helicone-session（请求分组）、opencode-type-inject（类型注入）、opencode-wakatime（时间追踪）等。

**社区资源**：[awesome-opencode](https://github.com/awesome-opencode/awesome-opencode)

#### 版本迁移

从 0.x 升级到 1.0：

```bash
opencode upgrade 1.0.0
# 或
opencode upgrade 0.15.31   # 降级回 0.x
```

1.0 是 TUI 完全重写版本（Zig + SolidJS），连接相同的 OpenCode 服务器。

#### 0.x → 1.0 迁移详情

#### 键绑定重命名（5 项）

| 旧名称 | 新名称 |
|--------|--------|
| `messages_revert` | `messages_undo` |
| `switch_agent` | `agent_cycle` |
| `switch_agent_reverse` | `agent_cycle_reverse` |
| `switch_mode` | `agent_cycle` |
| `switch_mode_reverse` | `agent_cycle_reverse` |

#### 已移除键绑定（11 项，需从配置中删除）

`messages_layout_toggle`、`messages_next`、`messages_previous`、`file_diff_toggle`、`file_search`、`file_close`、`file_list`、`app_help`、`project_init`、`tool_details`、`thinking_blocks`

#### 1.0 版本 UX 变更

- 新增命令栏（`Ctrl+P`）
- 新增会话侧边栏
- 会话历史更紧凑（只显示 edit 和 bash 工具详情）
- TUI 从 Go + Bubbletea 重写为 OpenTUI（Zig + SolidJS）

### 附录：速查表

#### 环境变量速查

| 变量 | 说明 |
|------|------|
| `ANTHROPIC_API_KEY` | Anthropic API Key |
| `OPENAI_API_KEY` | OpenAI API Key |
| `OPENCODE_ENABLE_EXA` | 启用 websearch/codesearch |
| `OPENCODE_EXPERIMENTAL` | 启用所有实验性功能 |
| `OPENCODE_SERVER_PASSWORD` | 服务器密码 |
| `OPENCODE_DISABLE_AUTOCOMPACT` | 禁用自动压缩 |
| `OPENCODE_DISABLE_LSP_DOWNLOAD` | 禁用 LSP 自动下载 |
| `OPENCODE_EXPERIMENTAL_WORKSPACES` | 启用工作区 |
| `OPENCODE_EXPERIMENTAL_LSP_TOOL` | 启用 LSP 工具 |
| `EDITOR` | 外部编辑器 |

#### 配置完整示例

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  
  // === 模型 ===
  "model": "anthropic/claude-opus-4-5-thinking",
  "small_model": "anthropic/claude-haiku-4-5",
  "default_agent": "build",
  
  // === Provider ===
  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "{env:ANTHROPIC_API_KEY}",
        "timeout": 600000,
        "setCacheKey": true
      }
    }
  },
  "disabled_providers": ["gemini"],
  
  // === 用户 ===
  "username": "开发者",
  "theme": "catppuccin",
  "autoupdate": true,
  
  // === 界面 ===
  "tui": { "scroll_speed": 3, "diff_style": "auto" },
  "keybinds": { "leader": "ctrl+x", "session_new": "<leader>n" },
  "server": { "port": 4096, "hostname": "localhost" },
  
  // === 行为 ===
  "share": "manual",
  "compaction": { "auto": true, "prune": true },
  "watcher": { "ignore": ["node_modules/**", "dist/**"] },
  "instructions": ["CONTRIBUTING.md"],
  
  // === 权限 ===
  "permission": {
    "edit": "ask",
    "bash": { "*": "ask", "git *": "allow" }
  },
  
  // === Agent ===
  "agent": {
    "code-reviewer": {
      "description": "代码审查专家",
      "mode": "subagent",
      "temperature": 0.2,
      "permission": { "edit": "deny" }
    }
  },
  
  // === 命令 ===
  "command": { "test": { "template": "运行测试", "description": "运行测试套件" } },
  
  // === 格式化器 ===
  "formatter": { "prettier": { "disabled": false } },
  
  // === MCP ===
  "mcp": {
    "context7": { "type": "remote", "url": "https://mcp.context7.com/mcp" }
  }
}
```

#### 实验性功能完整列表

**环境变量**：

| 环境变量 | 说明 |
|----------|------|
| `OPENCODE_EXPERIMENTAL=true` | 一键启用所有实验性功能 |
| `OPENCODE_ENABLE_EXA=true` | 启用 websearch/codesearch |
| `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` | 启用 LSP 工具 |
| `OPENCODE_EXPERIMENTAL_LSP_TY=true` | 启用 ty Python 服务器（替代 pyright）|
| `OPENCODE_EXPERIMENTAL_PLAN_MODE=true` | 启用计划模式（plan_enter/plan_exit 工具）|
| `OPENCODE_EXPERIMENTAL_WORKSPACES=1` | 启用 TUI 工作区（`/workspaces`）|
| `OPENCODE_EXPERIMENTAL_OXFMT=true` | 启用高性能 oxfmt 格式化器 |
| `OPENCODE_EXPERIMENTAL_FILEWATCHER=true` | 为整个目录启用文件监视器 |
| `OPENCODE_EXPERIMENTAL_DISABLE_FILEWATCHER=true` | 禁用文件监视器 |
| `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS=<n>` | Bash 命令默认超时（毫秒）|
| `OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX=<n>` | LLM 响应最大输出 token 数 |

**opencode.json experimental 配置**：

| 字段 | 说明 |
|------|------|
| `experimental.batch_tool` | 批量工具，一次执行多个操作 |
| `experimental.openTelemetry` | OpenTelemetry 链路追踪 |
| `experimental.disable_paste_summary` | 粘贴大段文本时不自动摘要 |
| `experimental.continue_loop_on_deny` | 权限被拒时 Agent 继续执行 |
| `experimental.primary_tools` | 仅限 Primary Agent 使用的工具列表 |
| `experimental.mcp_timeout` | MCP 请求全局超时（毫秒）|

#### 配置补充字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `snapshot` | boolean | 是否启用 Git 快照备份，设为 `false` 禁用（默认启用）|
| `skills.paths` | string[] | 额外 Skill 文件夹路径（支持 `~` 和相对路径）|
| `skills.urls` | string[] | 远程 Skill URL（需要 `index.json` 描述）|
| `enterprise.url` | string | 企业 GitHub Enterprise URL，格式 `"enterprise": { "url": "https://..." }` |

#### /export 导出格式说明

TUI 中的 `/export` 命令默认导出为 **Markdown 格式**（便于阅读）。CLI 命令 `opencode export [sessionID]` 导出为 **JSON 格式**（完整数据，便于备份和导入）。

#### /sessions 的删除功能

`/sessions` 打开会话列表后，除了查看和切换，还可以直接删除不需要的会话。

#### 限制 AI 只读操作的方法

- **Tab 切换到 Plan 模式**：Plan 模式禁止编辑普通文件
- **启动时指定 Agent**：`opencode -a plan`

#### 社区插件完整列表（23 个）

| 插件 | 说明 |
|------|------|
| opencode-helicone-session | 自动注入 Helicone 会话头，用于请求分组 |
| opencode-type-inject | 自动注入 TypeScript/Svelte 类型到文件读取 |
| opencode-openai-codex-auth | 使用 ChatGPT Plus/Pro 订阅替代 API 额度 |
| opencode-gemini-auth | 使用现有 Gemini 计划替代 API 计费 |
| opencode-antigravity-auth | 使用 Antigravity 免费模型 |
| opencode-devcontainers | 多分支 devcontainer 隔离，浅克隆和自动端口分配 |
| opencode-google-antigravity-auth | Google Antigravity OAuth 插件 |
| opencode-dynamic-context-pruning | 通过裁剪过时工具输出来优化 Token 使用 |
| opencode-websearch-cited | 为支持的提供商添加原生网页搜索 |
| opencode-pty | 允许 AI 代理在 PTY 中运行后台进程并发送交互输入 |
| opencode-shell-strategy | 非交互式 shell 命令指令，防止 TTY 操作挂起 |
| opencode-wakatime | 使用 WakaTime 追踪 OpenCode 使用时间 |
| opencode-md-table-formatter | 清理 LLM 生成的 Markdown 表格 |
| opencode-morph-fast-apply | 使用 Morph Fast Apply API 实现 10 倍速代码编辑 |
| oh-my-opencode | 后台代理、预构建 LSP/AST/MCP 工具、精选代理 |
| opencode-notificator | OpenCode 会话的桌面通知和声音提醒 |
| opencode-notifier | 权限、完成和错误事件的桌面通知 |
| opencode-zellij-namer | 基于 OpenCode 上下文的 AI 自动 Zellij 会话命名 |
| opencode-skillful | 允许 OpenCode 代理按需延迟加载提示词 |
| opencode-supermemory | 使用 Supermemory 实现跨会话持久记忆 |
| @plannotator/opencode | 交互式计划审查，支持可视化注释 |
| @openspoon/subtask2 | 将 /commands 扩展为强大的编排系统 |
| opencode-scheduler | 使用 launchd/systemd 调度定时任务 |

#### 社区项目（8 个）

| 项目 | 说明 |
|------|------|
| kimaki | 控制 OpenCode 会话的 Discord Bot，基于 SDK 构建 |
| opencode.nvim (NickvanDyke) | Neovim 插件，支持编辑器感知提示，基于 API |
| portal | 移动优先的 Web UI，支持 Tailscale/VPN 访问 |
| opencode plugin template | OpenCode 插件开发模板 |
| opencode.nvim (sudo-tee) | 另一个 Neovim 前端 |
| ai-sdk-provider-opencode-sdk | Vercel AI SDK 提供商，通过 @opencode-ai/sdk 使用 |
| OpenChamber | Web/桌面应用和 VS Code 扩展 |
| OpenCode-Obsidian | 将 OpenCode 嵌入 Obsidian UI 的插件 |

#### Agent 集合（2 个）

| 项目 | 说明 |
|------|------|
| Agentic | 模块化 AI 代理和命令，用于结构化开发 |
| opencode-agents | 增强工作流的配置、提示词、代理和插件集合 |

#### 社区资源入口

- [awesome-opencode](https://github.com/awesome-opencode/awesome-opencode) — 社区精选资源列表
- [opencode.cafe](https://opencode.cafe) — 社区聚合网站
