---
title: Git 基础
tags:
  - git
  - tool
---

# Git 基础

Git 做三件事：**记录快照**（commit）、**对比变化**（diff）、**同步协作**（push/pull）。

核心心智模型：

```text
工作区 (Working Tree)
  |  git add
  v
暂存区 (Index)
  |  git commit
  v
历史记录 (Commits)
```

- **工作区**：你正在编辑的文件夹
- **暂存区**：你"选中准备提交"的改动集合
- **提交（commit）**：一次可回到的快照

> [!note] Git vs GitHub
> **Git** 是版本管理工具，在本地运行；**GitHub** 是托管 Git 仓库的网站，用于远端备份和协作。
## 命令相关

### 一、仓库初始化（项目第一次用）

1. **git init**
    
    作用：把当前文件夹变成 Git 本地仓库
    
    场景：本地新建项目，没有远程仓库时用
    
2. **git clone 远程地址**
    
    作用：完整下载远程仓库所有代码、分支、提交记录，自动绑定远程 origin
    
    场景：已有线上项目，第一次拉代码到本地
    

### 二、文件暂存 & 提交（本地保存版本）

3. **git add .**
    
    作用：把当前目录所有新增、修改文件放入暂存区（打包待提交）
    
    场景：代码改完，准备提交前执行
    
4. **git commit -m "提交备注"**
    
    作用：把暂存区文件生成一个永久版本快照
    
    场景：功能写完、修复 bug 后保存版本
    

### 三、远程仓库同步（本地↔云端）

5. **git push**

- 新分支首次推送：`git push -u origin 分支名`（绑定上下游）
- 之后直接：`git push`
    
    作用：把本地提交上传到远程仓库
    
    场景：写完代码同步给团队其他人

6. **git fetch origin**
    
    作用：只拉取远程最新提交记录，**不改动本地文件**，安全预览别人的更新
    
    场景：想先看看同事改了啥，避免直接合并冲突翻车
    
7. **git pull**
    
    等价：fetch + merge，拉取远程代码并自动合并到当前分支
    
    场景：每天上班第一件事，快速同步线上最新代码
    `git pull` 只作用于**你当前所在的这一个分支**。
    

### 四、分支操作（多线并行开发）

8. **git branch**

- `git branch`：查看所有本地分支，* 代表当前所在分支
- `git branch dev`：新建分支但不切换
- `git branch -d dev`：删除已合并的本地分支
    
    场景：查看、新建、清理废弃分支

9. **git switch 分支名**

- `git switch main`：切换到主分支
- `git switch -c dev`：创建并直接切换到新分支
    
    作用：切换分支，本地文件自动切换成对应版本
    
    场景：主线修 bug、切回功能分支开发

### 五、代码合并

10. **git merge 分支名**
    
    作用：把目标分支代码合并进当前分支，保留分叉提交历史
    
    场景：公共分支（main、线上分支）合并代码，多人协作最安全
    
11. **git rebase 分支名**
    
    作用：重排提交记录，把自己所有提交接到主线最新节点后，提交历史一条直线
    
    场景：个人开发分支同步主线、整理压缩多次提交
    
    ⚠️ 禁忌：已经推送到远程的公共分支禁止 rebase
    

附属命令：

- `git rebase --continue`：冲突修复后继续变基
- `git rebase --abort`：放弃本次变基，回退到操作前

### 六、查看类命令（排查、查看状态日志）

12. **git status**
    
    作用：查看文件当前状态（未跟踪、已修改、已暂存）
    
    场景：不确定代码有没有 add、哪些文件被修改
    
13. **git log --oneline**
    
    作用：简洁查看所有提交历史、提交 ID
    
    场景：需要回滚版本、排查代码修改记录
    
14. **git diff**
    
    作用：查看未暂存的代码具体改动内容
    
    场景：忘记自己改了哪些内容，核对代码修改
    

### 七、代码临时储藏（没写完要切分支）

15. **git stash**
    
    作用：把未提交的代码临时存起来，工作区恢复干净
    
    场景：功能写到一半，需要切分支紧急改 bug，暂时不想提交半成品
    
16. **git stash pop**
    
    作用：把储藏的代码恢复到当前分支，同时清空储藏记录
    

### 八、撤销回滚救命命令

17. **git reset HEAD .**
    
    作用：把已经`git add`的文件从暂存区撤回
    
    场景：误把不需要的文件提交到暂存区
    
18. **git checkout -- 文件名**
    
    作用：放弃单个文件所有未暂存修改，恢复到上次提交版本
    
    场景：代码写崩，想要丢弃本地修改
    
19. **git reset --hard 提交 ID**
    
    作用：强制回退到指定历史版本，清空所有本地修改
    
    ⚠️ 高危操作，谨慎使用





## 核心概念

### 指针

Git 里的「指针」本质上就是一个指向某个 commit 的文件名。

**三种指针**：

| 指针         | 含义                     | 示例                        |
| ---------- | ---------------------- | ------------------------- |
| 分支（branch） | 指向 commit 的可移动指针       | `main` 存着最新 commit 的 hash |
| HEAD       | 指向当前分支的指针（间接指向 commit） | `HEAD → main → a1b2c3d`   |
| tag（标签）    | 永远不移动的指针，指向某个固定 commit | `v1.0 → a1b2c3d`          |

实际查看指针：

```bash
# 看 HEAD 当前指向哪里
cat .git/HEAD
# 输出：ref: refs/heads/main

# 看 main 分支指向哪个 commit
cat .git/refs/heads/main
# 输出：a1b2c3d4e5f6...
```

### Fast-forward 机制与 merge

以写小说为例，你和同事一起写同一本小说，主分支 `main` = 出版社定稿原稿。
Fast-forward（快进合并）：**不需要创建新 commit，直接把指针往前移。**
- 最开始只有一份原稿 `main`，内容：第 1~5 章
- 你拉了个新分支 `xiaoming`，准备写第 6、7 章
- **在你写稿这段时间里，没人去动主分支 main**，出版社那边没改任何内容
- 你在自己分支写完第 6、7 章，提交保存
合并过程：主分支根本不需要重新整合两份文件，只是把 `main` 的指针**直接往前挪一步**，挪到你写完的最新稿子位置。
就相当于：出版社直接把你写好的第 6、7 章，接在原有 5 章后面，稿子顺序连贯，没有出现两个版本，**不会生成新的合并记录**。

普通merge三方合并：**（出现分叉，必须新建合并记录）**
1. 初始 `main`：1~5 章
2. 你切分支 `xiaoming`，去写 6、7 章
3. **就在你写稿的时候，同事直接在主分支 main 改了内容**：他在 main 里新增了第 6 章番外

现在出现两条完全不一样的稿件路线：
- 你的分支：1-5 章 + 你写的第 6、7 章
- 主分支：1-5 章 + 同事写的第 6 章番外
合并过程：系统要拿「最初共同的原稿（1-5 章）」当基准，对比你和同事两边的修改，整合出一份全新终稿，**自动生成一条全新的「合并提交记录」**，用来标记这次两边分叉内容整合的操作。
如果你们俩都改了第 6 章，就会出现冲突，需要手动选保留谁的文字。

**为什么推荐保持 fast-forward**：

1. 历史是直线，`git log` 干净
2. 没有 merge commit，commit 数量少
3. 方便回滚，`git reset --hard` 就能直接退回

团队项目里保持线性历史的方法：

- **Squash and merge**：把 feature 多个 commit 合成一个，再 fast-forward
- **Rebase and merge**：把 feature rebase 到 main 最新提交之后，再 fast-forward

### 切换分支规则

核心规则：**凡是会"修改某个分支指向"或"把改动合入某个分支"的操作，都必须在那个分支上执行。**

| 操作 | 口诀 | 必须在哪个分支上执行 | 示例 |
|------|------|-------------------|------|
| merge | "合到哪，切到哪" | 目标分支（合入方） | `git switch main` → `git merge feat/login` |
| rebase | "谁要被挪，谁就是当前分支" | 被变基的分支 | `git switch feat/login` → `git rebase origin/main` |
| pull | 切到要更新的分支 | 目标分支 | `git switch main` → `git pull origin main` |
| reset | 切到要重置的分支 | 被重置的分支 | `git switch main` → `git reset --hard HEAD~1` |
| 删除分支 | 不能在当前分支上删自己 | 离开目标分支 | `git switch dev` → `git branch -d main` |
| push | 多数情况可不切 | 任意分支 | `git push origin feat/login` |

> [!warning] 切换分支前
> 未提交的修改一定要先 **commit** 或 **stash**，否则修改会被带到目标分支，严重时会导致代码覆盖或丢失。

常见错误示例：

```bash
# ❌ 在 feature 上想把 feature 合到 main——反了！
git switch feat/login
git merge main            # 这会把 main 合到 feat/login

# ✅ 正确：切到 main 再合并
git switch main
git merge feat/login
```

```bash
# ❌ 在 main 上 rebase——主次颠倒！
git switch main
git rebase feat/login     # 会把 main 接到 feat/login 之后

# ✅ 正确：在 feature 上 rebase
git switch feat/login
git rebase origin/main    # 把 feat/login 重新接到 main 之后
```

### fetch 与 pull

`git pull` = `git fetch` + `git merge`（配置了 `pull.rebase` 则是 rebase）

| 命令 | 做了什么 | 修改工作区？ | 风险 |
|------|---------|-----------|------|
| `git fetch origin` | 下载远程最新信息到 `origin/xxx` 引用 | 否 | 无 |
| `git pull` | fetch 后自动合并到当前分支 | 是 | 可能冲突 |

使用建议：

- **想安全查看远程变化**：先 `git fetch`，再用 `git log main..origin/main` 查看差异，确认后再合并
- **确定要同步**：直接 `git pull`
- **单人开发**：`git pull` 即可，冲突概率低

### git pull --ff-only vs --rebase

|                   | `--ff-only`                 | `--rebase`                     |
| ----------------- | --------------------------- | ------------------------------ |
| 等价于               | `fetch` + `merge --ff-only` | `fetch` + `rebase origin/main` |
| 能否 fast-forward 时 | 合并成功                        | 合并成功                           |
| 不能 fast-forward 时 | **直接报错，什么都不做**              | 改写本地未推送的提交历史（产生新 hash）         |
| 改写提交 hash         | 否                           | 是（X → X'）                      |
| 适用场景              | 本地没新提交                      | 本地有新提交想同步远端                    |


### switch main

***例子 1：日常在自己功能分支同步代码（不用切 main）***

```
# 当前在 feature 分支
git pull --rebase
```

只更新当前 feature 分支。

***例子 2：准备上线合并前，必须切 main 拉最新***

```
git switch main
git pull
git merge feature
git push
```

防止你的本地 main 太旧，合并后推远程报错。

## 快速上手

### 配置身份（只做一次）

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
git config --global init.defaultBranch main
git config --global pull.rebase true
git config --global core.editor vim
git config --global alias.lg "log --oneline --graph --decorate --all"

# 验证
git config --global --list
```

### 创建仓库

```bash
# 本地初始化
cd /path/to/project
git init

# 或克隆远程仓库
git clone https://github.com/<owner>/<repo>.git
```

### .gitignore

告诉 Git 哪些文件不要纳入版本控制。模板：

```bash
node_modules/
dist/
build/
*.log
.env
.env.local
.DS_Store
.idea/
```

关键规则：

- **只对未跟踪文件生效**。已提交过的文件写进 `.gitignore` 不会自动消失，需先 `git rm --cached <file>` 取消跟踪
- `.gitignore` 本身要提交到远程，团队共用同一套规则

三种忽略文件的区别：

| 文件 | 作用范围 | 提交到远程 |
|------|---------|----------|
| `.gitignore` | 项目内所有人 | 是 |
| `.git/info/exclude` | 仅本机当前项目 | 否 |
| 全局 `.gitignore_global` | 本机所有项目 | 否 |

> [!warning] 密钥已提交怎么办？
> 如果 `.env`、密钥等已被提交，不仅要做 `git rm --cached`，还必须去对应平台**立刻作废并重新生成密钥**。

### 首次提交与推送

```bash
git status            # 查看当前状态
git add .             # 全部加入暂存区
git commit -m "init"  # 提交

# 确认分支名为 main（GitHub 默认分支）
git branch -M main

# 绑定远端并推送（clone 的仓库跳过 remote add）
git remote add origin https://github.com/<owner>/<repo>.git

# 把本地main分支推送到远程main分支
git push -u origin main
```

> [!note] 为什么要 `git branch -M main`
> GitHub 2020 年起默认分支改为 `main`。本地分支叫 `master` 时直接 push 会报错，改为 `main` 可一次绑定成功。

> [!note] 为什么要 `git push -u origin main`
> `origin` 是默认的远程仓库别名（默认远程仓库名字就叫 origin）。执行后本地 `main` ↔ 远程 `origin/main` 绑定成功，只需第一次推送时用一次，之后直接 `git push` / `git pull` 即可。
> 如果你在 dev 分支不小心敲成 `git push -u origin main`，才会把 dev 的代码直接推送到远程 main，造成覆盖风险。正常写法一定是本地什么分支就推远程同名分支；只有执行 merge 才会把 dev 合并进 main，否则两条分支永远独立。

### SSH 免密推送

```bash
# 1. 生成密钥（连续3次回车，不设密码）
ssh-keygen -t ed25519 -C "你的GitHub邮箱"

# 2. 复制公钥
cat ~/.ssh/id_ed25519.pub

# 3. GitHub → Settings → SSH and GPG keys → New SSH key → 粘贴公钥

# 4. 测试连通
ssh -T git@github.com

# 5. 改远程地址为 SSH
git remote set-url origin git@github.com:用户名/仓库名.git
```

之后 `git push / git pull` 不再需要输入密码。

## 日常流程

### 开发流程
#### 1. 拉取远程代码
首次直接clone即可
```bash
# 首次直接clone即可
git clone xxxx
```
##### 个人功能分支
分支这条路只有你一个人走，直接 rebase变基
```
# 非首次
git fetch origin <branch>
git fetch origin main        # 只拉取主main最新
git fetch origin             # 拉取所有分支最新信息，团队多分支场景
```

查看远程主线main和本地主线main是否同步，本地主线是否落后了
```
git log HEAD..origin/main --oneline
```

有输出说明不同步：
```
git switch feature/xxx
git rebase origin/main      # 根据远程仓库最新的 main 快照做变基

# 遇到冲突，手动修改冲突文件
git add .
git rebase --continue      # 继续完成变基
git rebase --abort         # 不想保留本次变基，放弃回滚

# 变基完成后，首次推送绑定上游
git push -u origin feature
git push                   # 后续直接
```

##### 公共主线main分支

主线多人共用，**不能用 rebase**，而 merge 不会重写提交历史，所有修改都会留下合并记录，方便追溯、团队不会出现代码错乱。
```
#################### 方式一，大厂规范
# 1. 切换到主线
git switch main

# 2. 拉取远程最新代码
git fetch origin

# 3. 合并远程主线最新代码 
git merge origin/main

# 冲突处理，改完冲突代码后
git add . 
git commit -m "解决合并冲突，同步远程代码"

################## 方式二 git pull = git fetch + git merge
# 1. 切到主线 
git switch main 
# 2. 同步远程最新代码（fetch+merge） 
git pull
# pull 冲突时，解决冲突文件后
git add .
git commit -m "解决合并冲突，同步远程代码"
```

#### 2. 本地修改代码
#### 3. 提交代码
```
git status 

# 有未提交代码就先： 
git add . 
git commit -m "功能开发完成，准备合并上线"
```

再执行一边拉取远程代码步骤，保持本地main和远程main到最新状态。
```
git push        # push的是本地feature分支到远程仓库feature分支
```

切换到主线main分支，合并上线
```
# 1. 切到主线 
git switch main 
# 2. 拉取远程主线最新代码（多人公共分支必须先更新本地main） 
git pull 
# 3. 合并功能分支，推荐 --no-ff 强制保留合并记录，方便以后追溯上线版本 
# 因此此时满足fast-forward条件，默认就不会生成合并记录，--no-ff强制生成一条合并提交，能清晰看到**哪次提交、哪个功能分支合入上线**，线上回滚、排查问题有据可查，企业团队强制规范。
git merge --no-ff feature -m "上线：xxx功能"

# 推送到远程主线，正式上线
git push origin main         # 仅首次推送需要，因为要绑定远程仓库分支
git push                     # 非首次，之前已经push过了
# 清理本地废弃功能分支
git branch -d feature/xxx
```

习惯：**开工先 pull，收工 commit + push**。

> [!danger] 两个致命禁忌
> 1. 禁止在 main 分支直接开发——多人协作极易冲突和误提交
> 2. 禁止 `git push -f origin main`——会覆盖远程别人的提交，代码丢失无法恢复

## 常见问题

| 现象 | 原因 | 解决 |
|-----|------|------|
| `commit` 报 "Please tell me who you are" | 没设作者信息 | `git config --global user.name/email` |
| `status` 一堆文件不敢提交 | 没有 .gitignore | 先写 `.gitignore`，再 `git status` 确认 |
| `push` 要密码且失败 | GitHub 禁用密码推送 | 配置 SSH key 或 PAT |
| 分支 main/master 混乱 | 命名不一致 | `git branch -M main` |
| `.env` 已提交 | .gitignore 未覆盖或已跟踪 | 立刻作废密钥 + `git rm --cached` |
| `remote add origin` 提示已存在 | clone 的仓库已有 origin | 跳过；改地址用 `git remote set-url` |
| push 被拒绝 | 分支保护 | 走分支 + PR 路线 |
| GitHub 勾了 README 导致冲突 | 远端非空 | 重建空仓库，或 `git pull --rebase` 再 push |
