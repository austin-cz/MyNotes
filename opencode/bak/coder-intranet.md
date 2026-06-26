---
title: B6 内网/离线部署
subtitle: 让 OpenCode 在无外网环境正常运行
course: OpenCode 中文实战课
stage: 第四阶段
lesson: "4.B6"
duration: 25 分钟
practice: 15 分钟
level: 进阶
description: 配置 OpenCode 在企业内网或离线环境运行，禁用所有外网请求，使用本地模型列表和内部 AI 网关。
tags:
  - 内网
  - 离线
  - 企业
  - 部署
prerequisite:
  - 1.4 连接模型
  - 5.1 配置全解
---

# 内网/离线部署

> 💡 **一句话总结**：用 7 个开关把外网请求关掉，让 OpenCode 在内网环境能跑起来。

## 📝 课程笔记

本课核心知识点整理：

<img src="/images/4-scenarios/coder-intranet-notes.mini.jpeg"
     alt="B6 内网/离线部署学霸笔记"
     data-zoom-src="/images/4-scenarios/coder-intranet-notes.jpeg" />

---

## 学完你能做什么

- 在完全无外网的环境运行 OpenCode
- 使用本地缓存的模型列表，不依赖 models.dev
- 连接公司内部 AI 网关，不触发任何外网请求
- 排查"启动卡住"、"网络超时"等内网常见问题
- 解决内网最常见的“依赖安装卡住”问题（`@opencode-ai/plugin`）

---

## 你现在的困境

- 公司内网不能访问外网，OpenCode 启动就卡住
- 设置了 `OPENCODE_DISABLE_MODELS_FETCH=true`，但还是卡
- 不知道 OpenCode 到底会请求哪些外网地址
- 想用公司内部的 AI 网关，但不知道怎么配置
- **核心问题**：启动后没有任何错误提示，但就是不动，日志中也没有"loading plugin"相关输出（这是 `@opencode-ai/plugin` SDK 安装卡住的典型现象）

---

## 什么时候用这一招

- 企业内网环境，机器不能访问外网
- 安全合规要求，不允许任何数据出内网
- 离线开发环境（飞机上、无网络的机房）
- CI/CD 环境，想加速启动、避免网络抖动

---

## 🎒 开始前的准备

- [ ] 有一个可用的内部 AI 网关（或本地 Ollama）
- [ ] 能从外网机器下载 `https://models.dev/api.json`
- [ ] 了解 [1.4 连接模型](../1-start/04-connect) 的基本配置方式
- [ ] **（可选）验证系统是否已安装 ripgrep**：`rg --version`

::: tip 💡 检查 ripgrep
在内网环境中，推荐提前检查 `rg` 是否已安装，避免使用 AI 的 grep 功能时失败。
:::

---

## 核心思路

OpenCode 启动时会尝试以下外网请求：

| 请求                    | 用途            | 禁用方式                                                                                                                                                                     |
| --------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `models.dev/api.json` | 获取模型列表        | 方案 A（完全离线）：`OPENCODE_DISABLE_MODELS_FETCH=true` + `OPENCODE_MODELS_PATH=...`；方案 B（内网镜像）：只设置 `OPENCODE_MODELS_URL=https://...`（不要设置 `OPENCODE_DISABLE_MODELS_FETCH=true`） |
| npm registry          | 安装内置插件        | `OPENCODE_DISABLE_DEFAULT_PLUGINS=true`                                                                                                                                  |
| GitHub releases       | 检查更新          | `OPENCODE_DISABLE_AUTOUPDATE=true` 或 `autoupdate: false`                                                                                                                 |
| LSP 服务器下载             | 语言服务器         | `OPENCODE_DISABLE_LSP_DOWNLOAD=true`                                                                                                                                     |
| 远程 skill 加载           | 加载外部 skill    | `OPENCODE_DISABLE_EXTERNAL_SKILLS=true`                                                                                                                                  |
| 项目配置加载                | 扫描 .opencode/ | `OPENCODE_DISABLE_PROJECT_CONFIG=true`                                                                                                                                   |
| 共享功能                  | 分享会话          | `OPENCODE_DISABLE_SHARE=true`                                                                                                                                            |

::: info 📖 两种模型列表方案
- 完全离线：下载 `models.json`，设置 `OPENCODE_MODELS_PATH`，并打开 `OPENCODE_DISABLE_MODELS_FETCH=true`
- 内网镜像：公司内网提供一个 `https://<host>/api.json`，设置 `OPENCODE_MODELS_URL=https://<host>`，不要设置 `OPENCODE_DISABLE_MODELS_FETCH=true`

如果同时设置 `OPENCODE_MODELS_PATH` 和 `OPENCODE_MODELS_URL`，会优先读取 `PATH` 指向的本地文件。
:::

**只要把这 7 类请求全部禁用，OpenCode 就能在纯内网环境运行。**

::: info 📋 配置文件也要检查
除了设置环境变量，还要确保 `opencode.json` 中**没有**以下配置（否则仍会触发外网请求）：

```jsonc
{
  "skills": {
    "urls": ["https://..."]  // 删除远程 skill URL
  },
  "instructions": [
    "https://..."  // 删除远程 instruction
  ],
  "auth": {
    "example.com": {
      "type": "wellknown"  // 删除 wellknown 认证
    }
  }
}
```

**`.well-known/opencode`** 是远程配置加载机制，即使设置了环境变量，如果配置中使用了 `wellknown` 认证，仍会发起外网请求。
:::

---

## 跟我做

### 第 1 步：下载模型列表文件

**为什么**  
OpenCode 需要知道有哪些模型可用。在外网环境先下载这个文件，然后拷贝到内网机器。

在**能访问外网的机器**上执行：

```bash
curl -o models.json https://models.dev/api.json
```

**你应该看到**：当前目录生成 `models.json` 文件（约 500KB）。

### 第 2 步：把模型列表放到内网机器

**为什么**  
内网机器需要这个文件来知道模型的能力（context 限制、是否支持 tool_call 等）。

把 `models.json` 拷贝到内网机器的固定位置：

```bash
# 推荐放到 ~/.cache/opencode/ 目录
mkdir -p ~/.cache/opencode
cp models.json ~/.cache/opencode/models.json
```

### 第 3 步：配置环境变量

**为什么**  
这是核心步骤。设置这些环境变量后，OpenCode 不会尝试任何外网请求。

::: code-group
```bash [macOS/Linux - 临时生效]
export OPENCODE_DISABLE_MODELS_FETCH=true
export OPENCODE_MODELS_PATH=~/.cache/opencode/models.json
export OPENCODE_DISABLE_DEFAULT_PLUGINS=true
export OPENCODE_DISABLE_AUTOUPDATE=true
export OPENCODE_DISABLE_LSP_DOWNLOAD=true
export OPENCODE_DISABLE_EXTERNAL_SKILLS=true
export OPENCODE_DISABLE_PROJECT_CONFIG=true
export OPENCODE_DISABLE_SHARE=true
```

```bash [macOS/Linux - 永久生效]
# 添加到 ~/.bashrc 或 ~/.zshrc
cat >> ~/.zshrc << 'EOF'
# OpenCode 内网配置
export OPENCODE_DISABLE_MODELS_FETCH=true
export OPENCODE_MODELS_PATH=~/.cache/opencode/models.json
export OPENCODE_DISABLE_DEFAULT_PLUGINS=true
export OPENCODE_DISABLE_AUTOUPDATE=true
export OPENCODE_DISABLE_LSP_DOWNLOAD=true
export OPENCODE_DISABLE_EXTERNAL_SKILLS=true
export OPENCODE_DISABLE_PROJECT_CONFIG=true
export OPENCODE_DISABLE_SHARE=true
EOF

source ~/.zshrc
```

```powershell [Windows PowerShell - 临时生效]
$env:OPENCODE_DISABLE_MODELS_FETCH = "true"
$env:OPENCODE_MODELS_PATH = "$env:USERPROFILE\.cache\opencode\models.json"
$env:OPENCODE_DISABLE_DEFAULT_PLUGINS = "true"
$env:OPENCODE_DISABLE_AUTOUPDATE = "true"
$env:OPENCODE_DISABLE_LSP_DOWNLOAD = "true"
$env:OPENCODE_DISABLE_EXTERNAL_SKILLS = "true"
$env:OPENCODE_DISABLE_PROJECT_CONFIG = "true"
$env:OPENCODE_DISABLE_SHARE = "true"
```
:::

### 第 4 步：解决依赖安装卡住问题（内网关键⚠️）

::: warning ⚠️ 重要性
这是**最常见**的内网启动卡住问题！即使设置了 `OPENCODE_DISABLE_DEFAULT_PLUGINS=true`，OpenCode 仍会尝试安装 `@opencode-ai/plugin` SDK，这个安装**不受任何环境变量控制**。

**原因**：安装逻辑在 `src/config/config.ts:237-257`，会执行 `bun add` 和 `bun install`，内网环境会卡住。

**解决方案（推荐方案 1）**：
```bash
# 创建空的 node_modules 目录，跳过安装检查
mkdir -p ~/.config/opencode/node_modules

# 验证目录已创建
ls -la ~/.config/opencode/
# 你应该看到 node_modules 目录
```

::: tip 💡 其他解决方案
如果方案 1 不适用，可以选择：
- **方案 2**：配置内网 npm 镜像（`~/.bunfig.toml` 或 `.npmrc`）
- **方案 3**：在有网络的机器预装依赖，然后把 `~/.config/opencode/`（以及项目的 `.opencode/`）拷贝到内网机器
- **方案 4**：临时禁用项目配置扫描：`OPENCODE_DISABLE_PROJECT_CONFIG=true`

详见下方"踩坑提醒 → 依赖安装卡住（@opencode-ai/plugin）"。
:::

### 第 5 步：配置内部 AI 网关

**为什么**  
禁用外网后，你需要告诉 OpenCode 用哪个内部模型。

创建或编辑 `~/.config/opencode/opencode.json`：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  
  // 禁用自动更新（双保险，环境变量已设置）
  "autoupdate": false,
  
  // 只启用你的内部 provider
  "enabled_providers": ["corp-gateway"],
  
  // 配置内部 AI 网关
  "provider": {
    "corp-gateway": {
      "name": "公司 AI 网关",
      "env": ["CORP_AI_TOKEN"],
      "api": "https://ai-gateway.company.internal/v1",
      "npm": "@ai-sdk/openai-compatible",
      "models": {
        "qwen2.5-72b": {
          "name": "Qwen 2.5 72B",
          "tool_call": true,
          "reasoning": true,
          "temperature": true,
          "limit": { "context": 128000, "output": 8192 }
        },
        "glm-4": {
          "name": "GLM-4",
          "tool_call": true,
          "temperature": true,
          "limit": { "context": 128000, "output": 4096 }
        }
      }
    }
  },
  
  // 默认使用内部模型
  "model": "corp-gateway/qwen2.5-72b"
}
```

::: tip 💡 如果用本地 Ollama
把 `api` 改成 `http://localhost:11434/v1`，参考 [1.4g Ollama 配置](../1-start/04g-ollama)。
:::

### 第 6 步：设置 API Token

**为什么**  
内部网关通常需要认证。

```bash
export CORP_AI_TOKEN="your-internal-token"
```

### 第 7 步：验证配置

**为什么**  
确保所有配置生效，OpenCode 能正常启动。

```bash
opencode run -m corp-gateway/qwen2.5-72b "1+1=?" --print-logs
```

**你应该看到**：
- 没有任何网络超时错误
- 模型正常返回结果（比如 `2`）

如果还是卡住，加 `--log-level DEBUG` 看详细日志：

```bash
opencode run -m corp-gateway/qwen2.5-72b "1+1=?" --print-logs --log-level DEBUG
```

---

## 检查点 ✅

> 全部通过才能继续；任一项失败，回到对应步骤重来

- [ ] 有一个可用的内部 AI 网关（或本地 Ollama）
- [ ] 能从外网机器下载 `https://models.dev/api.json`
- [ ] 了解 [1.4 连接模型](../1-start/04-connect) 的基本配置方式
- [ ] 模型列表已就绪：`~/.cache/opencode/models.json` 存在，且 `OPENCODE_MODELS_PATH` 指向它（或你使用内网镜像 `OPENCODE_MODELS_URL`）
- [ ] 环境变量已设置（用 `env | grep OPENCODE` 检查）
- [ ] 已创建 `~/.config/opencode/node_modules`（如果当前项目有 `.opencode/`，也创建 `./.opencode/node_modules`）
- [ ] `opencode.json` 配置了内部 provider
- [ ] `opencode run` 能正常返回结果，没有网络错误
- [ ] （可选）已安装 ripgrep（`rg --version` 能输出版本号）

---

## 踩坑提醒

| 现象 | 可能原因 | 解决 |
|---|---|---|
| 启动卡住，没有更多日志 | 依赖安装在内网环境挂起（`@opencode-ai/plugin`） | 先看“依赖安装卡住（@opencode-ai/plugin）” |
| 提示 "model not found" | `models.json` 路径或内容不对 | 检查 `OPENCODE_MODELS_PATH` 是否指向正确文件 |
| 提示 "provider not found" | `enabled_providers` 和 provider id 不一致 | 检查 `enabled_providers`、`provider` 的 key、以及 `model` 前缀 |
| 调用模型返回 401/403 | Token 未注入或过期 | 检查 `CORP_AI_TOKEN` 是否正确、是否已导出 |
| grep 工具失败 | ripgrep 未安装且无法下载 | 手动安装 `rg`（见下方“grep 工具特殊情况”） |

### 依赖安装卡住（@opencode-ai/plugin）

**为什么会卡**

OpenCode 扫描配置目录时，会触发依赖安装（源码：`src/config/config.ts:157-159` 和 `src/config/config.ts:237-257`）。它会在目录里执行：

```bash
bun add @opencode-ai/plugin@<version> --exact
bun install
```

如果某个目录里 **`node_modules/` 不存在**，OpenCode 会等待这次安装完成再继续；内网无法访问 npm registry 时，就会表现为“卡住”。

**先定位是不是卡在这里**

用 DEBUG 日志跑一次：

```bash
opencode run "test" --print-logs --log-level DEBUG
```

如果你看到类似 `service=bun` 的日志里出现 `cmd=[..., "add", "@opencode-ai/plugin@..."]`，基本可以确定是依赖安装挂起。

**快速止血（不再等待安装）**

把 OpenCode 可能扫描到的目录都补上 `node_modules/`：

```bash
# 全局配置目录
mkdir -p ~/.config/opencode/node_modules

# 如果你在某个项目目录里运行，并且项目有 .opencode/，也补一个
mkdir -p ./.opencode/node_modules
```

这招的作用是：避免 OpenCode 在启动时 `await installDependencies(...)`。

::: warning ⚠️ 但这不是“彻底解决”
`installDependencies(dir)` 依然会被触发（只是不会在启动时等待它完成）。如果你希望它能真正完成安装，还是需要内网 npm 镜像或预装依赖。
:::

**长期方案（推荐）**

1) 配置内网 npm 镜像（Bun 会使用你的配置）

`~/.bunfig.toml`：

```toml
[install]
registry = "http://your-internal-npm-registry/"
```

2) 预装依赖并拷贝

在有网络的机器上，让 `~/.config/opencode/` 和项目 `.opencode/` 目录完成一次依赖安装，然后把这两个目录拷贝到内网机器。

3) 临时禁用项目配置扫描

如果你不需要项目级 `.opencode/`（只用全局配置），可以：

```bash
export OPENCODE_DISABLE_PROJECT_CONFIG=true
```

### grep 工具特殊情况

**现象**：AI 使用 grep 工具时失败，提示找不到 `rg` 二进制文件。

**原因**：grep 工具依赖 ripgrep（`rg`）。如果系统中没有 `rg`，OpenCode 会尝试从 GitHub 下载（内网通常会失败）。

**解决方案**：

```bash
# macOS
brew install ripgrep

# Linux
apt install ripgrep  # Ubuntu/Debian
yum install ripgrep  # CentOS/RHEL

# Windows
scoop install ripgrep
# 或
choco install ripgrep

rg --version
```

### 如何确认环境变量生效？

```bash
env | grep OPENCODE
```

### 如何定位还在请求外网？

```bash
opencode run "test" --print-logs --log-level DEBUG
```

常见关键字：
- `service=models.dev`：在拉模型列表
- `service=bun`：在执行 bun add/install（安装依赖或插件）

---

## 完整配置速查

### 环境变量清单

| 环境变量 | 作用 | 值 |
|---------|------|-----|
| `OPENCODE_DISABLE_MODELS_FETCH` | （完全离线模式）禁止拉取模型列表 | `true` |
| `OPENCODE_MODELS_PATH` | （完全离线模式）本地模型列表文件路径 | 文件绝对路径 |
| `OPENCODE_MODELS_URL` | （内网镜像模式）把 models.dev 指到内网地址（需要提供 `/api.json`） | `https://models-mirror.company.internal` |
| `OPENCODE_DISABLE_DEFAULT_PLUGINS` | 禁止安装内置插件 | `true` |
| `OPENCODE_DISABLE_AUTOUPDATE` | 禁止自动更新检查 | `true` |
| `OPENCODE_DISABLE_LSP_DOWNLOAD` | 禁止下载 LSP 服务器 | `true` |
| `OPENCODE_DISABLE_EXTERNAL_SKILLS` | 禁止加载远程 skill | `true` |
| `OPENCODE_DISABLE_PROJECT_CONFIG` | 禁止扫描项目级 `.opencode/` 配置 | `true` |
| `OPENCODE_DISABLE_SHARE` | 禁止共享功能 | `true` |
| `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS` | 禁止 Claude Code 兼容技能 | `true` |

::: tip 💡 优先级说明
- 如果同时设置 `OPENCODE_MODELS_PATH` 和 `OPENCODE_MODELS_URL`，会优先读取 `OPENCODE_MODELS_PATH` 指向的本地文件
- 如果你走“内网镜像模式”，不要设置 `OPENCODE_DISABLE_MODELS_FETCH=true`，否则不会发起拉取
:::

### 一键配置脚本

**基础脚本（完全离线：手动下载 models.json）**：
```bash
#!/bin/bash
# save as: setup-intranet.sh

# 创建目录
mkdir -p ~/.cache/opencode

# 检查模型列表文件
if [ ! -f ~/.cache/opencode/models.json ]; then
    echo "❌ 请先下载 models.json 到 ~/.cache/opencode/models.json"
    exit 1
fi

# 解决 SDK 安装卡住问题（关键步骤！）
mkdir -p ~/.config/opencode/node_modules
echo "✅ 已创建 ~/.config/opencode/node_modules 目录"

# 如果当前目录有 .opencode/，也补一个（避免项目级等待安装）
if [ -d ./.opencode ]; then
  mkdir -p ./.opencode/node_modules
  echo "✅ 已创建 ./.opencode/node_modules 目录"
fi

# 添加环境变量到 shell 配置
SHELL_RC="$HOME/.$(basename $SHELL)rc"
cat >> "$SHELL_RC" << 'EOF'

# OpenCode 内网配置
export OPENCODE_DISABLE_MODELS_FETCH=true
export OPENCODE_MODELS_PATH=~/.cache/opencode/models.json
export OPENCODE_DISABLE_DEFAULT_PLUGINS=true
export OPENCODE_DISABLE_AUTOUPDATE=true
export OPENCODE_DISABLE_LSP_DOWNLOAD=true
export OPENCODE_DISABLE_EXTERNAL_SKILLS=true
export OPENCODE_DISABLE_PROJECT_CONFIG=true
export OPENCODE_DISABLE_SHARE=true
EOF

echo "✅ 环境变量已添加到 $SHELL_RC"
echo "请运行: source $SHELL_RC"
```

**高级脚本（内网镜像：自动拉取模型列表）**：
```bash
#!/bin/bash
# save as: setup-intranet-advanced.sh

# 公司内部 models.dev 服务器
MODELS_MIRROR="https://models-mirror.company.internal"

# 添加环境变量到 shell 配置
SHELL_RC="$HOME/.$(basename $SHELL)rc"
cat >> "$SHELL_RC" << EOF

# OpenCode 内网配置（使用内部镜像）
unset OPENCODE_DISABLE_MODELS_FETCH
export OPENCODE_MODELS_URL="$MODELS_MIRROR"
export OPENCODE_DISABLE_DEFAULT_PLUGINS=true
export OPENCODE_DISABLE_AUTOUPDATE=true
export OPENCODE_DISABLE_LSP_DOWNLOAD=true
export OPENCODE_DISABLE_EXTERNAL_SKILLS=true
export OPENCODE_DISABLE_PROJECT_CONFIG=true
export OPENCODE_DISABLE_SHARE=true
EOF

# 解决 SDK 安装卡住问题
mkdir -p ~/.config/opencode/node_modules
echo "✅ 已创建 ~/.config/opencode/node_modules 目录"

echo "✅ 环境变量已添加到 $SHELL_RC"
echo "已配置内部 models.dev 镜像: $MODELS_MIRROR"
echo "请运行: source $SHELL_RC"
```

::: tip 💡 两种模型列表方案对比
| 方案 | 优点 | 缺点 |
|-----|------|------|
| `OPENCODE_MODELS_PATH` | 完全离线、控制版本 | 需要手动更新 `models.json` |
| `OPENCODE_MODELS_URL` | 自动更新、多机同步 | 依赖内部镜像服务器 |
:::

---

## 本课小结

你学会了：

1. **7 个关键开关**：完全离线（`OPENCODE_DISABLE_MODELS_FETCH` + `OPENCODE_MODELS_PATH`）或内网镜像（`OPENCODE_MODELS_URL`），再加上禁用内置插件/自动更新/LSP 下载/远程 skill/项目配置/共享功能
2. **本地模型列表**：从外网下载 `models.json`，放到内网机器（或用 `OPENCODE_MODELS_URL` 指向内部 mirrors）
3. **内部 Provider 配置**：用 `enabled_providers` 只启用内部网关
4. **核心 SDK 安装问题**：通过创建空 `node_modules` 目录或其他方案解决
5. **配置文件检查**：确保 `opencode.json` 中没有 `skills.urls`、`instructions` 中的 URL、以及 `auth` 中的 `wellknown` 认证
6. **排查技巧**：用 `--print-logs --log-level DEBUG` 定位卡住位置

::: tip 💡 额外提示
如果你的公司有内部 mirrors.dev 服务器，可以用 `OPENCODE_MODELS_URL` 替代 `OPENCODE_MODELS_PATH`：
```bash
export OPENCODE_MODELS_URL=https://models-mirror.company.internal
```
这样 OpenCode 会从你的内部服务器获取模型列表，无需手动下载 `models.json`。
:::

---

## 下一课预告

> 如果你想更进一步，把认证也集中管理（让团队不用每人配置 Token），可以学习 **[5.11a 企业认证集成](11a-enterprise-auth.md)**。
>
> 那里介绍了 `/.well-known/opencode` 机制，可以实现"一条命令登录 + 自动下发组织配置"。

---

## 附录：源码参考

<details>
<summary><strong>点击展开查看源码位置</strong></summary>

> 更新时间：2026-02-05

| 功能 | 文件路径 | 行号 |
|-----|---------|------|
| 环境变量定义（包含所有 `OPENCODE_*` 标志） | [`src/flag/flag.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/flag/flag.ts#L12-L50) | 12-50 |
| 模型列表加载逻辑（`OPENCODE_DISABLE_MODELS_FETCH`、`OPENCODE_MODELS_PATH`、`OPENCODE_MODELS_URL`） | [`src/provider/models.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/provider/models.ts#L83-L99) | 83-99 |
| 配置目录扫描与依赖安装等待（`node_modules` 检查） | [`src/config/config.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/config/config.ts#L157-L160) | 157-160 |
| 依赖安装命令（`bun add` + `bun install`） | [`src/config/config.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/config/config.ts#L236-L257) | 236-257 |
| 插件加载（内置插件列表、`OPENCODE_DISABLE_DEFAULT_PLUGINS` 检查） | [`src/plugin/index.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/plugin/index.ts#L18-L49) | 18-49 |
| Bun 包安装逻辑（可能导致启动卡住） | [`src/bun/index.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/bun/index.ts#L64-L133) | 64-133 |
| ripgrep 下载逻辑（内网常失败） | [`src/file/ripgrep.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/file/ripgrep.ts#L125-L199) | 125-199 |
| 自动更新检查（`OPENCODE_DISABLE_AUTOUPDATE`） | [`src/cli/upgrade.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/upgrade.ts#L6-L25) | 6-25 |
| LSP 服务器下载检查（`OPENCODE_DISABLE_LSP_DOWNLOAD`） | [`src/lsp/server.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/lsp/server.ts) | 135, 177, 372... |

**关键常量**：
- `BUILTIN = ["opencode-anthropic-auth@0.0.13", "@gitlab/opencode-gitlab-auth@1.3.2"]`：内置插件列表
- `Flag.OPENCODE_DISABLE_MODELS_FETCH`：禁用模型列表拉取
- `Flag.OPENCODE_MODELS_PATH`：本地模型列表文件路径
- `Flag.OPENCODE_MODELS_URL`：内部 models.dev 服务器 URL
- `Flag.OPENCODE_DISABLE_DEFAULT_PLUGINS`：禁用内置插件安装

**网络请求优先级**：
1. `OPENCODE_MODELS_PATH`：如果设置，优先级高于 `OPENCODE_MODELS_URL`
2. `OPENCODE_MODELS_URL`：指向内部 models.dev 服务器
3. `https://models.dev`：默认地址（当上述两者都未设置且 `OPENCODE_DISABLE_MODELS_FETCH=false` 时）

</details>
