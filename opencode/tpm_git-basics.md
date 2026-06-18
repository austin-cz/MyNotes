
Git 其实就做三件事：

1. **记录快照**：把“现在的代码状态”存一份，起个名字（commit）
2. **对比变化**：你能随时看出“改了什么”
3. **同步协作**：把你的 commit 推到远端（GitHub），或者拉下来别人的 commit

最关键的心智模型是这张图：

```text
工作区 (Working Tree)
  |  git add
  v
暂存区 (Index)
  |  git commit
  v
历史记录 (Commits)
```

::: info 这三个词到底是什么意思？

- **工作区**：你正在编辑的文件夹（你肉眼看到的文件）
- **暂存区**：你“选中准备提交”的改动集合（还没进入历史）
- **提交（commit）**：一次“可回到的快照”，每次提交都应该能解释“我做了什么/为什么做”

:::

::: details Git 和 GitHub 有啥区别？

- **Git**：版本管理工具，在你电脑上运行。
- **GitHub**：托管 Git 仓库的网站。你可以把本地仓库推上去，和别人协作。

:::

接下来你会跑一遍“最小闭环”：初始化仓库 → 第一次提交 → 推到 GitHub。

然后你会学两条路线：

| 你现在的情况                  | 推荐做法                          | 你要记住的关键词                        |
| ----------------------- | ----------------------------- | ------------------------------- |
| 你一个人写（个人项目 / 小工具）       | 直接在 `main` 上提交并推送（不必分支、不必 PR） | `status → diff → commit → push` |
| 你要协作（多人 / 有 CI / 有代码审查） | 分支 + PR                       | `branch → PR → merge → pull`    |

::: tip OpenCode 的定位
你不需要背一堆 Git 命令。更实用的方式是：

1) 让 OpenCode 先给你“看改动”的摘要（基于 `git status` / `git diff`）
2) 再让它生成 commit message，并执行 `add/commit/push`
:::

---


::: tip 你可以在 OpenCode 里跑这些命令
在 OpenCode 的 TUI 输入框里，命令前加 `!` 就能执行（例如 `!git status`）。如果你更习惯普通终端，也完全没问题。
:::

### 第 1 步：让 Git 认识你（只做一次）

**为什么**
提交记录会写入作者信息；否则你第一次 commit 很容易报错。

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"

# 检查一下
git config --global --get user.name
git config --global --get user.email

# 查看配置
git config --global --list
```

### 第 2 步：创建仓库（init 或 clone 二选一）

**为什么**
没有仓库（`.git` 目录），Git 就没有“历史”可记录。

```bash [我有一个本地文件夹（推荐新手）]
# 进入你的项目目录
cd /path/to/your/project

# 初始化仓库，执行完后，本地会生成一个.git的隐藏目录
git init

# 或者从远程clone一个已有仓库
git clone https://github.com/<owner>/<repo>.git
cd <repo>
```


**你应该看到**：`git init` 会提示已初始化仓库；`git clone` 会把代码下载到本地。


#### .gitignore的作用：
`.gitignore` 是 Git 的**忽略规则配置文件**，作用：**告诉 Git 哪些文件 / 文件夹不要纳入版本控制、不要提交到 GitHub 远程仓库**。

四大核心作用
1. 过滤无用文件，保持代码仓库干净

自动忽略系统、编辑器、编译产生的垃圾文件：

- Windows 的 `Thumbs.db`、Mac 的 `.DS_Store`
- VS Code、IDEA 等 IDE 的本地配置文件夹 `.vscode`、`.idea`
- 编译打包产物 `dist`、`build`、日志文件 `*.log`、依赖文件夹 `node_modules`

不用每次提交手动筛选文件，执行 `git add .` 会自动跳过这些文件，仓库体积更小、代码整洁。

2. 防止敏感隐私文件泄露（最重要）

避免把数据库账号、密码、密钥、AK/SK、环境配置文件（`.env`、`.pem`、密码配置文件）误提交到公开的 GitHub，防止机密泄露。

3. 避免团队代码冲突

每个人本地的配置、缓存、临时文件不一样，如果全部提交会频繁出现代码冲突。

统一的 `.gitignore` 提交到远程，所有拉取项目的成员共用一套忽略规则，大幅减少冲突。

4. 只管理源码，不管理运行时文件

Git 只用来保存**手写的源代码**，自动生成、下载来的文件交给本地环境管理，不用在版本库来回同步。

三、关键生效规则（一定要记住）

1. **只对从未被 Git add/commit 过的「未跟踪文件」生效**
    
    文件一旦被提交过，Git 已经开始跟踪它，此时再写到 `.gitignore` 无效。
    
    需要先执行 `git rm --cached 文件名` 取消跟踪，本地文件保留，规则才会生效Git。
    
2. `.gitignore` 本身需要提交到远程仓库
    
    团队所有人拉取代码后，会共用同一套忽略规则。
    
3. 可以放在项目任意目录：
    

- 根目录：整个项目全局生效；
- 子目录：仅当前文件夹及其下级目录生效，就近规则优先级更高。

四、三种忽略文件简单区分

1. **项目共享：.gitignore**
    
    提交到远程，所有团队成员生效，用来统一项目通用忽略规则。
2. **当前仓库私有：.git/info/exclude**
    
    只在你本机当前项目生效，不会被提交，适合忽略个人临时测试文件。
3. **本机全局忽略：全局 .gitignore_global**
    
    电脑上所有 Git 仓库统一生效，一般用来忽略系统、IDE 垃圾文件。



#### 添加.gitignore
顺手加一个 `.gitignore`（避免把生成物、缓存、密钥提交进去）：

```bash
# 依赖与构建产物
node_modules/
dist/
build/

# 日志
*.log

# 本地环境变量（不要提交到仓库）
.env
.env.local

# 系统/编辑器杂项
.DS_Store
.idea/
```

::: tip 不想自己写？让 OpenCode 帮你挑“该忽略的文件”

你可以只说一句话，让 OpenCode 根据当前项目内容和 `git status` 来生成/更新 `.gitignore`：

```text
帮我检查哪些文件不必要提交，添加到 .gitignore。

要求：
1) 先运行并参考 git status / git diff（只看，不要直接提交/推送）
2) 只修改/创建 .gitignore，改完后把 .gitignore 的内容贴出来并解释每一条为什么要忽略
3) 如果发现 .env、密钥、token 之类的敏感文件，先停下提醒我
```

:::

::: warning `.gitignore` 有个“新手陷阱”
`.gitignore` 只对“还没被 Git 跟踪的文件”生效。

如果你已经把某个文件提交过，再把它写进 `.gitignore` 也不会消失。正确做法是先把它从仓库跟踪里移除（但保留本地文件），然后再提交：

```bash
git rm --cached <file>
git commit -m "chore: stop tracking <file>"
```

如果那个文件是密钥/口令，别只做 `rm --cached`，还需要去对应平台把密钥立刻作废并重新生成。
:::

::: warning 如果你已经有项目专用的 .gitignore
保留你已有的内容即可，不要强行改成这一份。
:::

### 第 3 步：做第一次提交（first commit）

**为什么**
第一次提交相当于“存档点 0”。从这一刻开始，你随时能回到这个状态。

```bash
# 看看当前状态
git status

# 把要提交的改动加入暂存区（第一次一般直接全加）
git add .

# 提交
git commit -m "init"
```

**你应该看到**：`git status` 从一堆红色“未跟踪/已修改”，变成“working tree clean”（干净）。

再看一下提交历史：

```bash
git log --oneline -5
```

**你应该看到**：有一行 `init`（前面是一个短哈希）。

### 第 4 步：把本地仓库推到 GitHub（第一次同步）

**为什么**
Git 的历史在你本机；推到 GitHub 后，你才有“远端备份 + 协作入口”。

1) 在 GitHub 新建一个空仓库（建议不要勾选 README / .gitignore，保持空仓库）。

2) 确认你当前在哪个分支：

```bash
git branch --show-current
```

如果你看到的是 `master`，建议把它改成 `main`（在当前分支执行就行）：

```bash
git branch -M main
```

3) 绑定远端并推送 `main`：

```bash
git remote -v

# 如果你是 git init 初始化的仓库：添加 origin
# 如果你是 git clone 下来的仓库：origin 通常已经存在，跳过这一行
git remote add origin https://github.com/<owner>/<repo>.git

# 确保在 main 上推送
git checkout main
git push -u origin main
```

**你应该看到**：GitHub 仓库页面出现 `main` 分支。

::: warning 如果 push 让你输入密码
GitHub 现在不支持用账号密码推送。你需要：

- 用 HTTPS：在提示凭证时使用 **Personal Access Token**（PAT）
- 或者用 SSH：把 SSH key 加到 GitHub

这一步属于 GitHub 账号配置，和 OpenCode 无关；卡住就先按 GitHub 的提示走。
:::

### 第 5 步：单人开发的日常节奏（不分支、不 PR）

**为什么**
你一个人做项目时，“分支 + PR”是可选项。最稳的节奏是：每做完一个小功能，就存一个可回到的存档点（commit），然后推送到 GitHub。

你只需要学会给 OpenCode 下指令（一句话版）：

```text
总结下这次修改，提交并推送。
```

::: details 想更稳妥（推荐）

如果你不想“盲推”，用这句（也不长，但更安全）：

```text
先检查 git status + git diff，总结改动并给出 commit message；
我确认后再执行提交和推送。

注意：如果发现 .env、密钥、token 之类的文件，先停下提醒我。
```

::: 

如果你更想“看得见摸得着”，也可以用 `!` 把关键命令跑一遍：

```bash
git status
git diff
```

**你应该看到**：

- OpenCode 先总结改动，再给出一条 commit message
- 推送成功后，GitHub 仓库能看到最新提交

::: warning 一句话就让它 push，有没有风险？
有，所以这里的套路是：先 `status/diff`，再 commit/push。你永远保留最后一次人工确认的机会。
:::

::: details 协作路线（可选）：分支 + PR

你需要下面这条路线的常见场景：

- 你的仓库开了 `main` 分支保护（不允许直接 push）
- 你希望每次合并前都跑 CI / 代码审查

```bash
# 创建分支并提交
git checkout -b feature/first-change
git add -A
git commit -m "feat: first change"
git push -u origin feature/first-change
```

然后到 GitHub 上创建 PR：base 选 `main`，compare 选 `feature/first-change`。

合并后本地同步：

```bash
git checkout main
git pull
```

:::

### 第 6 步：保持本地与 GitHub 同步

**为什么**
只要你的代码在 GitHub 上发生过变化（你在另一台电脑 push 了、你合并了 PR、或者别人推送了），本地就需要拉取更新。

推荐习惯：

- 开工先 `pull`
- 收工 `commit + push`

```bash
git checkout main
git pull
```

（可选）删除本地分支：

```bash
git branch -d feature/first-change
```

**你应该看到**：`git log --oneline -5` 里包含你刚才在 PR 里合并的提交。

::: info `/undo` 能帮你什么（别误会）

- `/undo` 用来撤销“最近一次消息导致的文件改动”。
- 它不是 `git reset`，不会替你撤销已经提交（commit）或已经推送（push）的历史。

如果你已经 `git commit` 了，那就用新的 commit 来修复；如果你已经 `git push` 了，先别乱改历史。

:::

---

## 检查点 ✅

- [ ] `git status` 显示 `working tree clean`
- [ ] `git log --oneline -5` 至少能看到 2 条提交
- [ ] `git remote -v` 能看到 `origin` 指向你的 GitHub 仓库
- [ ] GitHub 上能看到最新提交（如果你走 PR 路线，也能看到 PR）
- [ ] 本地执行 `git pull` 不报错，并提示 up to date 或拉到新提交

---

## 踩坑提醒

| 现象 | 原因 | 解决 |
|-----|-----|-----|
| `git commit` 报“Please tell me who you are” | 没设置作者信息 | 按第 1 步配置 `user.name` / `user.email` |
| `git status` 看到一堆文件，不敢提交 | 不知道哪些该进仓库 | 先写 `.gitignore`，再用 `git status` 确认 |
| `git push` 要你输密码，最后失败 | GitHub 禁用账号密码推送 | 改用 PAT（HTTPS）或 SSH key |
| 远端默认分支不对（main/master 混乱） | 分支命名不一致 | 先 `git branch --show-current` 确认你在 `master` 上，再执行 `git branch -M main`，然后推送 `main` |
| 不小心把 `.env` 提交上去了 | `.gitignore` 没写，或写了但已经提交过 | 立刻删除并重新生成密钥；再处理仓库历史（别拖） |
| `git remote add origin` 提示已存在 | 你是从 GitHub clone 下来的 | 跳过添加；需要改地址用 `git remote set-url origin <url>` |
| 你在 GitHub 勾了 README，push 时提示冲突 | 远端不是空仓库 | 最省事：重新建一个空仓库；或按提示先 `git pull --rebase` 再 push |
| 你想直接 push main，但被拒绝 | 仓库开了分支保护 | 走“分支 + PR”路线，或调整仓库规则 |

---

## 本课小结

你学会了：

1. Git 的三段式流程：工作区 → 暂存区 → commit
2. 用分支 + PR 做一次“可审查的改动”
3. 把本地仓库推到 GitHub，并在合并后保持同步

---

## 下一课预告

> 下一课建议你接着看 **[B3 文档与 Git](coder-docs-git.md)**。
>
> 你会把今天的 Git 基础，接到 OpenCode 的日常用法里：
> - 让 OpenCode 自动生成 commit 消息
> - 自动写 PR 描述
> - 用 `/undo` / `/redo` 回滚不满意的改动

---

## 附录：源码参考

<details>
<summary><strong>点击展开查看源码位置</strong></summary>

> 更新时间：2026-02-06

| 功能 | 文件路径 | 行号 |
|-----|---------|------|
| 在 TUI 中用 `!` 前缀执行 shell 命令（提示文案） | [`src/cli/cmd/tui/component/tips.tsx`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/cmd/tui/component/tips.tsx#L51-L56) | 51-56 |
| 项目识别：从 `.git` 向上查找并用 root commit 生成项目 ID | [`src/project/project.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/project/project.ts#L53-L110) | 53-110 |
| `/undo` 的核心：对消息产生的文件补丁做 revert（调用 Snapshot） | [`src/session/revert.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/session/revert.ts#L13-L76) | 13-76 |
| Snapshot 仅在 Git 项目启用，并用内部 git repo 记录/恢复快照 | [`src/snapshot/index.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/snapshot/index.ts#L50-L76) | 50-76 |
| Snapshot 对文件执行 `git checkout <hash> -- <file>` 做回滚 | [`src/snapshot/index.ts`](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/snapshot/index.ts#L130-L160) | 130-160 |

</details>
