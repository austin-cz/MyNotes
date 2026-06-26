---
title: OpenCode 使用手册
tags:
  - opencode
  - tool
---

# OpenCode 使用手册

## 配置

### 文件位置与优先级

配置按以下顺序加载，后加载的覆盖先加载的冲突键（非冲突设置都保留，是**合并**不是替换）：

| 优先级   | 位置                                 | 说明                     |
| ----- | ---------------------------------- | ---------------------- |
| 1（最低） | 远程 `.well-known/opencode`          | 远程组织默认配置（通过 Auth 机制获取） |
| 2     | `~/.config/opencode/opencode.json` | 全局用户配置                 |
| 3     | `OPENCODE_CONFIG` 环境变量             | 自定义配置文件路径              |
| 4     | `./opencode.json`                  | 项目根目录配置                |
| 5     | `./.opencode/opencode.json`        | 项目 .opencode 目录配置（推荐）  |
| 6     | `OPENCODE_CONFIG_CONTENT` 环境变量     | 内联配置内容（JSON 字符串）       |
| 7（最高） | 受管配置目录                             | 企业部署，管理员控制             |

> [!info] 企业受管配置
> 管理员可在系统级目录放置配置，优先级最高：macOS `/Library/Application Support/opencode`、Windows `%ProgramData%\opencode`、Linux `/etc/opencode`。普通用户了解即可。

### 配置目录结构

```
~/.config/opencode/
├── opencode.json       # 全局配置
├── AGENTS.md           # 全局规则
├── agent/              # 全局 Agent
├── command/            # 全局命令
└── plugin/             # 全局插件

项目目录/
├── opencode.json       # 项目配置（优先级 4）
├── AGENTS.md           # 项目规则
└── .opencode/
    ├── opencode.json   # 项目配置（优先级 5，推荐）
    ├── agent/          # 项目 Agent
    ├── command/        # 项目命令
    └── plugin/         # 项目插件
```

配置文件名可以是 `opencode.json` 或 `opencode.jsonc`（支持注释的 JSON）：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  // 这是注释，JSONC 格式支持
  "model": "anthropic/claude-opus-4-5-thinking"
}
```

### 配置详解

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

#### 配置字段速查

| 字段 | 说明 |
|------|------|
| `model` | 主模型，格式 `provider/model-id` |
| `small_model` | 小模型，用于生成标题等轻量任务（不设置则回退主模型） |
| `default_agent` | 默认 primary agent（`"build"` / `"plan"` / 自定义名称） |
| `provider` | Provider 配置（注意是**单数**，不是 providers） |
| `disabled_providers` | 禁用的 Provider 列表（优先级高于 enabled_providers） |
| `enabled_providers` | 只启用这些 Provider |
| `username` | 对话中显示的用户名 |
| `theme` | TUI 主题（注意是 `theme`，不是 `tui.theme`） |
| `autoupdate` | `true` 自动更新（默认）/ `false` 禁用 / `"notify"` 只通知 |
| `instructions` | 额外规则文件引用列表 |
| `compaction` | 上下文压缩配置，`{"auto": true}` 自动压缩 |

Provider `options` 字段：

| 字段 | 说明 |
|------|------|
| `apiKey` | API 密钥（推荐用变量替换） |
| `baseURL` | 自定义 API 地址（代理场景） |
| `timeout` | 请求超时毫秒数，默认 300000，设 `false` 禁用 |
| `setCacheKey` | 启用提示缓存键（默认 false） |

> [!info] Amazon Bedrock
> Bedrock 额外支持 `region`（AWS 区域）、`profile`（AWS 配置文件）、`endpoint`（自定义端点 URL）。
>
> ```jsonc
> {
>   "provider": {
>     "amazon-bedrock": {
>       "options": {
>         "region": "us-east-1",       // 默认从 AWS_REGION 环境变量或 us-east-1
>         "profile": "my-aws-profile",     // 来自 ~/.aws/credentials
>         "endpoint": "https://bedrock-runtime.us-east-1.vpce-xxxxx.amazonaws.com"
>       }
>     }
>   }
> }
> ```

#### 变量替换

配置中可用变量动态获取值：

- `{env:变量名}` — 引用环境变量（不存在则替换为空字符串）
- `{file:路径}` — 引用文件内容（支持相对路径、绝对路径 `/`、home 目录 `~`）

```jsonc
{
  "model": "{env:OPENCODE_MODEL}",
  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "{env:ANTHROPIC_API_KEY}"  // 从环境变量读取
      }
    },
    "openai": {
      "options": {
        "apiKey": "{file:~/.secrets/openai-key}"  // 从文件读取
      }
    }
  }
}
```

> [!warning] 常见踩坑
> - 配置不生效 → 检查优先级，项目级覆盖全局
> - 变量替换失败 → 确认环境变量已设置
> - JSON 解析错误 → 用 JSONC 格式或检查语法
> - Provider 不加载 → 检查 `disabled_providers` 列表
> - 键名写错 → 是 `provider`（单数）不是 `providers`，是 `theme` 不是 `tui.theme`

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

| Agent   | 读写权限  | 命令权限     | 上下文    | 核心定位                   |
| ------- | ----- | -------- | ------ | ---------------------- |
| Explore | 只读    | 仅安全查询    | 轻量临时   | 查代码、摸架构、风险调研           |
| Plan    | 无读写   | 不能执行命令   | 临时     | 方案设计、步骤拆解、风险评估         |
| General | 读写全权限 | 任意 Shell | 独立子会话  | 批量改造、一次性复杂任务、文档生成      |
| Build   | 读写全权限 | 任意 Shell | 主线持续保留 | 日常迭代、连续开发、BUG 修复、小范围微调 |

- **Primary Agent**（Build / Plan）：你直接对话的主助手，用 <kbd>Tab</kbd> 切换
- **Subagent**（Explore / General）：由 Primary Agent 自动调用或你手动用 `@agent名` 调用

Plan 模式下生成的计划文件保存位置：

| 级别  | 路径                                              | 条件      |
| --- | ----------------------------------------------- | ------- |
| 项目级 | `.opencode/plans/<时间戳>-<slug>.md`               | 项目有 Git |
| 全局级 | `~/.local/share/opencode/plans/<时间戳>-<slug>.md` | 项目无版本控制 |

> [!note] 内部 Agent
> OpenCode 还有 3 个内部 Agent，自动在后台工作，无需手动调用：
> - **compaction**：上下文压缩（对话接近限制时自动触发）
> - **title**：会话标题生成（创建新会话后自动执行）
> - **summary**：会话摘要生成（压缩会话时生成摘要）

> [!warning] 实验性功能
> `plan_enter` / `plan_exit` 工具目前是实验性功能，需设置 `OPENCODE_EXPERIMENTAL=true` 或 `OPENCODE_EXPERIMENTAL_PLAN_MODE=true` 才能使用。启用后 AI 可主动调用工具在 Plan/Build 模式间切换（会弹出确认框）。

#### @Explore 代码勘探（只读安全）

- 接手陌生项目，梳理目录结构、模块架构、调用链路
- 全局检索接口、函数、关键字、硬编码配置
- 统计定时任务、数据表、接口清单
- 溯源 BUG：追踪参数流转、定位报错堆栈
- 代码审计、影响范围评估

```text
@Explore 梳理当前 EIS 项目分层架构，说明每个目录作用
@Explore 全局搜索所有数据库硬编码 IP，列出文件路径和行号
@Explore 追踪登录接口从 Controller 到 DAO 的完整调用链
```

#### @Plan 方案规划师（只思考不操作）

- 复杂需求方案评审：模块重构、架构改造、性能优化
- 拆解上线步骤、回滚方案、风险点、前置检查项
- 梳理技术债务、漏洞整改清单

> 不会碰你项目任何文件，只输出结构化方案。

```text
@Plan 设计 MySQL 慢查询优化整体方案，包含前置检查、参数调整、上线步骤、回滚策略
@Plan 梳理当前日志体系重构的实施方案、依赖影响、改造先后顺序
```

#### @General 批量任务执行者（隔离子会话）

- 多步骤批量任务：调研→整理→批量修改→生成文档
- 一次性独立任务：批量替换配置、抽取公共工具类、生成文档
- 同时并行多个互不依赖的工作

> 独立子会话运行，不会把大量检索和临时命令堆在主线导致上下文溢出。

```text
@General 根据前面调研的硬编码 IP，全部替换为环境变量配置，输出修改清单
@General 扫描项目所有定时任务，整理成运维巡检文档保存到 docs 目录
```

#### @Build 主线开发（默认 Agent，持续上下文）

- 日常迭代开发：写业务代码、修复 BUG、新增接口、单元测试
- 连续微调：改完一段代码，接着优化逻辑、调整参数、修复报错
- 本地调试：根据报错日志不断修正代码

> 上下文全程保留。直接提问不加 `@Agent` 就是默认调用 Build。

```text
优化这个 ssh 脚本，增加 IP 合法性校验，异常输出日志
修复当前定时任务重复执行的 BUG
```


### 标准工作流

**Explore → Plan → General → Build**

1. `@Explore` 摸底调研 → 确认代码范围、依赖、影响面
2. `@Plan` 输出实施方案 → 敲定改造步骤、风险、回滚方案
3. `@General` 批量落地改造 → 大范围统一修改、文档生成
4. 主线 Build 精细化调试 → 修正细节报错、迭代优化、本地验证

#### 实战示例：硬编码 IP 改环境变量

```text
# 第一步：安全调研
@Explore 全局检索项目下所有硬编码 MySQL 地址 192.168.1.100，
输出每个文件路径、行号、上下文，统计总引用数

# 第二步：出方案
@Plan 设计把所有硬编码 MySQL 地址改成环境变量 MYSQL_HOST 的完整改造方案，
包含改造步骤、兼容兜底策略、上线前置检查、风险点、回滚方案

# 第三步：批量落地
@General 按方案批量替换所有硬编码为 ${MYSQL_HOST}，增加默认兜底值，
在 docs 目录生成《MySQL 环境变量改造说明文档.md》

# 第四步：精细化调试（默认 Build，不加 @）
启动项目检查配置解析是否报错；
没有配置 MYSQL_HOST 时校验是否兜底为旧 IP；
优化 shell 脚本增加变量合法性校验
```

> [!tip] 记忆方法
> - **Explore** = 装修前上门量房，只勘察不施工
> - **Plan** = 设计师出施工图和预算，只出方案不动手
> - **General** = 施工队统一全屋水电改造，批量一次性施工
> - **Build** = 你自己收尾：缝隙打胶、开关微调、验收

#### 什么时候必须用标准工作流

全程只用默认 Build 也能完成工作，但缺少**权限隔离 + 会话隔离 + 定位约束**，存在三个隐患：

| 隐患      | Build 直接做                              | 标准工作流                         |
| ------- | -------------------------------------- | ----------------------------- |
| 调研阶段误操作 | 权限全开，可能顺手改配置                           | `@Explore` 强制只读，绝不可能误变更       |
| 上下文溢出   | 调研→方案→批量改→调试全部堆在主线，token 消耗快，AI 容易遗忘约束 | `@General` 在独立子会话，批量操作日志不污染主线 |
| 跳过方案评审  | 可能直接上手写代码执行脚本，漏掉备份和回滚                  | `@Plan` 强制先敲定步骤和回滚策略          |

**极简取舍规则**：

- 本地小需求、单文件修改、调试 BUG → 直接 Build
- 线上巡检、大范围代码检索 → 必须先 `@Explore`
- 线上配置变更、架构改造、高危操作 → 严格走标准流程


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
| **项目规则** | 项目根目录 `AGENTS.md`（从当前目录向上遍历到工作目录根）  | 项目特定的规范   |
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



## 场景应用

### 开发工作流

```
理解代码(Explore) → 制定方案(Plan) → 实现功能(Build) → 验证测试(Build)
```

#### 第 1 步：快速理解代码

```text
@explore 帮我梳理这个项目的整体结构，包括：
1. 主要目录和功能模块
2. 入口文件和核心流程
3. 使用的技术栈和框架
```

深入某个文件：

```text
@src/services/auth.ts 这个认证模块的逻辑是什么？列出所有导出函数及其作用
```

#### 第 2 步：制定功能方案

```text
我要给这个项目添加一个「邮件通知」功能，当用户注册成功后发送欢迎邮件。
请帮我分析：
1. 需要修改哪些文件
2. 推荐的实现方案（2-3 种）
3. 每种方案的优缺点
4. 推荐哪种方案，为什么
```

#### 第 3 步：分步实现功能

```text
按照方案一实现邮件通知功能：

第一步：创建邮件服务模块 src/services/email.ts
- 使用 nodemailer
- 支持 SMTP 配置
- 导出 sendEmail 函数
```

#### 第 4 步：定位 Bug

```text
用户反馈「登录后页面一直 loading」，请帮我分析：
1. 可能的原因有哪些
2. 如何排查（给出具体步骤）
3. 最可能的问题在哪个文件
```

#### 第 5 步：修复 Bug

```text
@src/hooks/useAuth.ts 问题定位到这里：
- 登录成功后 isLoading 没有重置为 false
- 请修复这个问题
```

### 重构安全三步法

```
1. 先有测试 → 2. 小步重构 → 3. 测试通过
```

#### 第 1 步：识别代码坏味道

```text
@src/utils/data.ts 请分析这个文件的代码质量：
1. 列出发现的"坏味道"
2. 每个问题的严重程度（高/中/低）
3. 推荐的重构方式
4. 重构的优先级建议
```

#### 第 2 步：先生成测试

```text
@src/utils/data.ts 为这个文件生成单元测试：
1. 使用 Vitest 框架
2. 覆盖所有导出函数
3. 包含正常情况和边界情况
4. 保存为 src/utils/data.test.ts
```

#### 第 3 步：安全重构

```text
@src/utils/data.ts 请重构 parseUserData 函数：
- 问题：函数过长（50 行），职责不单一
- 要求：拆分成 3 个小函数
- 保持对外接口不变
- 重构后运行测试确认
```

#### 第 4 步：补充边界测试

```text
@src/utils/data.ts @src/utils/data.test.ts

分析代码的边界条件，补充以下测试：
1. 空输入（null、undefined、空数组）
2. 极值（超大数字、超长字符串）
3. 类型错误（传入错误类型）
4. 并发情况（如果有异步操作）
```

#### 第 5 步：运行测试验证

以 `!` 开头的消息会执行 shell 命令并把输出带进对话：

```text
!npm test src/utils/data.test.ts
```


### 文档自动化

| 文档类型 | 生成时机 | AI 辅助方式 |
|---------|---------|-----------|
| README | 项目初期/重大更新 | 分析项目结构生成 |
| API 文档 | 接口开发完成 | 从代码提取生成 |
| Commit 消息 | 每次提交 | 分析变更生成 |
| PR 描述 | 提交 PR | 汇总 commit 生成 |

#### 生成 README

```text
@explore 分析项目结构，生成一个专业的 README.md：
1. 项目名称和简介
2. 功能特性
3. 快速开始（安装和运行）
4. 使用示例
5. 配置说明
6. 贡献指南
7. 许可证
保存为 README.md
```

#### 生成 commit 消息

以 `!` 开头执行 shell 命令，先把变更带入对话：

```text
!git diff
```

然后让 AI 生成：

```text
根据以上变更，生成符合 Conventional Commits 规范的 commit 消息：
- 格式：type(scope): description
- type：feat/fix/docs/style/refactor/test/chore
- 简洁明了，不超过 50 字符
```

#### 生成 PR 描述

```text
!git log --oneline -10

根据以上 commit 历史，生成 PR 描述：
## 变更概述 / 变更详情 / 测试情况 / 相关 Issue
```

#### 撤销 AI 修改

```text
/undo
```

> 撤销文件操作需要项目是 Git 仓库。更多细节见 https://opencode.ai/docs/tui#undo

#### 补充代码注释

```text
@src/services/payment.ts 为这个文件添加 JSDoc 注释：
- 每个导出函数都加
- 包含参数说明和返回值
- 包含使用示例
```

### CI/CD 集成

- **优先用 GitHub Agent**：不自己在 workflow 里手写胶水代码
- **触发方式**：在评论里写 `/oc` 或 `/opencode`
- 官方说明：https://opencode.ai/docs/github

```bash
# 运行安装向导
opencode github install

# 提交并推送 workflow 文件（通常是 .github/workflows/opencode.yml）
git add . && git push
```

在 Issue 或 PR 里评论即可触发：

```text
/oc summarize
```

## 专属开发 Agent

Agent 配置文件放置位置：
- 全局：`~/.config/opencode/agent/`
- 项目级：`.opencode/agent/`
- 调用名默认来自文件名：`code-reviewer.md` → `@code-reviewer`

### 第 1 步：创建 Code Reviewer Agent

```text
帮我创建一个 Code Reviewer Agent，保存到 ~/.config/opencode/agent/code-reviewer.md：

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

你是一位经验丰富的高级工程师，专门负责代码审查。

## 审查清单

### 代码质量
- [ ] 函数职责单一
- [ ] 命名清晰准确
- [ ] 无重复代码
- [ ] 适当的注释

### 潜在问题
- [ ] 边界条件处理
- [ ] 错误处理完整
- [ ] 无内存泄漏风险
- [ ] 无竞态条件

### 可维护性
- [ ] 代码易于理解
- [ ] 可测试性好
- [ ] 符合项目规范

## 输出格式

对于每个问题，按以下格式输出：
- **位置**：文件名:行号
- **问题**：问题描述
- **严重程度**：高 / 中 / 低
- **建议**：修复建议
```

### 第 2 步：创建 Security Auditor Agent

```text
帮我创建一个 Security Auditor Agent，保存到 ~/.config/opencode/agent/security-auditor.md：

---
description: 安全漏洞猎人
mode: subagent
model: anthropic/claude-opus-4-5-thinking
temperature: 0.2
permission:
  edit: deny
  bash: deny
---

# Security Auditor Agent

你是一位安全专家，专门发现代码中的安全隐患。

## 检查项目

### 输入验证
- SQL 注入
- XSS 攻击
- 命令注入
- 路径遍历

### 认证授权
- 身份验证绕过
- 权限提升
- 会话管理

### 敏感数据
- 硬编码密钥
- 敏感信息泄露
- 不安全的存储

### 依赖安全
- 已知漏洞依赖
- 过时的包版本

## 输出格式

对于每个安全问题：
- **漏洞类型**：OWASP 分类
- **位置**：文件名:行号
- **风险等级**：Critical/High/Medium/Low
- **描述**：漏洞描述
- **修复建议**：如何修复
- **参考**：相关 CWE/CVE
```

### 第 3 步：创建 Test Writer Agent

```text
帮我创建一个 Test Writer Agent，保存到 ~/.config/opencode/agent/test-writer.md：

---
description: 测试用例专家
mode: subagent
model: anthropic/claude-opus-4-5-thinking
temperature: 0.4
permission:
  edit: deny
  bash: deny
---

# Test Writer Agent

你是一位测试专家，擅长设计和编写测试用例。

## 测试策略

1. **单元测试**：隔离测试每个函数
2. **集成测试**：测试模块间交互
3. **边界测试**：测试边界条件
4. **异常测试**：测试错误处理

## 测试覆盖

每个函数必须覆盖：
- 正常输入
- 边界值（最大、最小、临界）
- 非法输入（null、undefined、错误类型）
- 异常情况（网络错误、超时）

## 输出格式

使用项目的测试框架，生成可直接运行的测试代码。
```

### 第 4 步：使用专属 Agent

```text
@code-reviewer @src/services/auth.ts 请审查这个认证模块
@security-auditor @src/controllers/ 对这个目录进行安全审计
@test-writer @src/utils/validate.ts 为这个文件生成完整的测试用例
```

### 第 5 步：组合工作流：command

创建一个综合审查命令 `.opencode/command/全面审查.md`：

```text
---
description: 综合代码审查
---

请依次执行：
1. @code-reviewer 审查代码质量
2. @security-auditor 检查安全隐患
3. @test-writer 分析测试覆盖率

目标文件：$ARGUMENTS

最后汇总所有问题，按优先级排序。
```

使用：

```text
/全面审查 src/services/payment.ts
```

