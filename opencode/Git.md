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

### 仓库初始化

**git init**

作用：把当前文件夹变成 Git 本地仓库

场景：本地新建项目，没有远程仓库时用

**git clone 远程地址**

作用：完整下载远程仓库所有代码、分支、提交记录，自动绑定远程 origin

场景：已有线上项目，第一次拉代码到本地

### 文件暂存 & 提交

**git add .**

作用：把当前目录所有新增、修改文件放入暂存区（打包待提交）

场景：代码改完，准备提交前执行

**git commit -m "提交备注"**

作用：把暂存区文件生成一个永久版本快照

场景：功能写完、修复 bug 后保存版本

### 远程仓库同步

**git push**

作用：把当前分支的本地提交上传到远程仓库对应分支
Git 会检查：**远程分支最新提交节点，是否在你本地分支的提交历史链路里（快进）**。
1. 如果是快进（本地分支只是在远程后面新增了提交）→ 允许普通 `git push`；
2. 如果不是快进（本地和远程分叉、历史链路不一样）→ 拒绝普通 push，防止误删远端提交。
场景：写完代码同步给团队其他人

新分支首次推送：

```bash
# `-u` = `--set-upstream`，执行后本地分支绑定远程同名分支，以后直接 git push 即可
git push -u origin 本地分支名:远程分支名      # 默认本地、远程分支名相同，简写如下
git push -u origin 本地分支名
```

之后直接：`git push`

**git fetch origin**

作用：只拉取远程所有分支最新提交记录，**不改动本地文件**，安全预览别人的更新

场景：想先看看同事改了啥，避免直接合并冲突翻车

```bash
git fetch origin                # 拉取所有分支最新提交信息
git fetch origin main           # 只拉取 main 分支
git fetch origin feature        # 只拉取 feature 分支
```

**git pull**

等价：fetch + merge，拉取远程代码并自动合并到当前分支

场景：每天上班第一件事，快速同步线上最新代码

> [!warning] 注意
> `git pull` 只作用于**你当前所在的这一个分支**。
> `git pull origin feature` 会把远程 `origin/feature` 合并到**你当前所在的分支**，容易造成冲突。


`git pull --ff-only`

作用：**只允许快进拉取，本地不能有新提交，一旦本地和远端提交分叉**，直接拉取失败，不会自动生成合并提交。

适用场景：主干历史要求线性提交。
举例：
1. 小明、小红两个人同时拉取了 `main` 最新代码（当前提交节点：A）
2. 小明本地写完功能，提交了提交 B，直接 push 到远端 main
3. 小红在自己本地 A 节点的基础上，写完代码提交了 C，准备 push
4. 此时小红本地代码落后远端，不能直接推送，执行普通 `git pull`
    Git 会自动把远端 B、本地 C 做合并，生成一个全新的**合并提交 D**
    最终 `main` 提交历史变成：
    A → B
    ↓ ↘
    C → D
    **主干出现了分叉，再汇合，提交线不再是一条直线。**

适用场景：
1. 个人日常在公共主干分支（`main`/`master`）更新代码，保证分支提交线干净，杜绝意外产生杂乱的合并节点；
2. 团队强制要求主干分支必须线性提交，防止多人误操作拉出分叉；
3. 不确定本地是否改动，想安全拉远端最新代码，避免自动合并带来冲突隐患。
不适合：本地已经提交了未推送的代码。


`git pull --rebase`
作用：拉取远端代码，把你本地的提交 “挪到” 远端最新提交后面，**保持提交线性，不产生合并节点**。

举例：同样起点 A：
1. 小明提交 B 推送到远端 main
2. 小红执行 `git pull --rebase`
    Git 会先把小红本地提交 C 暂时拿下来，先把远端 B 拉到本地，再把 C 贴在 B 的后面
    最终提交线：`A → B → C`，从头到尾一条直线，没有任何分叉、没有多余合并节点。

适用场景：
1. 在个人功能分支开发，频繁拉主干最新代码，避免后续合并出现大量杂乱合并记录；
2. 提交代码前整理提交历史，保证推送到远端的分支是一条干净直线；
3. 开源协作、代码评审场景，团队要求提交历史整洁可追溯。
小提醒：不要在多人共用的公共分支 rebase。


### 分支操作

**git branch**

- `git branch`：查看所有本地分支，* 代表当前所在分支
- `git branch dev`：新建分支但不切换
- `git branch -d dev`：删除已合并的本地分支

场景：查看、新建、清理废弃分支

**git switch 分支名**

- `git switch main`：切换到主分支
- `git switch -c dev`：创建并直接切换到新分支

场景：主线修 bug、切回功能分支开发

### 代码合并

**git merge 分支名**

作用：把目标分支代码合并进当前分支，保留分叉提交历史
场景：公共分支（main、线上分支）合并代码，多人协作最安全


**`git merge --ff-only origin/main`**
作用：保护主干分支（main/master），防止提交历史变乱、出现分叉。不会偷偷生成一个合并提交把两路分叉强行揉在一起，从根源保护主干历史不乱。

举例：公司主项目的`main`分支就是**官方唯一主生产线**，所有人只能依次往线上合代码。
1. 你拉了`main`开了`feature`分支开发需求，当时主线最新节点是【版本 A】
2. 你在 feature 写完代码，此时线上`main`还没有任何人提交新代码，依旧停在【版本 A】
3. 你切回`main`执行 `git merge --ff-only feature`
    Git 直接把 main 指针向前滑动，快进到 feature 最新提交，历史依旧是一条直线，没有新增任何合并记录，干净整洁。
结果：
✅ 可以快进（主线没别人提交）：合并成功，主线保持线性
❌ 不能快进（主线已经被其他人提交了新代码，出现分叉）：**直接拒绝合并，报错终止**


`git merge --no-ff feature`（禁止快进，强制生成合并提交记录）
作用：禁止快进，**强制生成一个新的合并提交节点**，完整保留功能分支从开发到合并的整条历史轨迹。

举例：每次需求上线，都要在公司上线台账单独记一笔：XX 需求、什么时间合并上线，方便以后查账、回退版本。
1. 同样起点：`main`当前节点【版本 A】，你拉取`feature`分支开发需求
2. 开发结束后，线上`main`还没更新，理论上可以直接快进合并
3. 执行 `git merge --no-ff feature`
    **哪怕满足快进条件，Git 也不会只移动指针，而是强制新建一个唯一的合并提交节点，专门用来标记：「本次把 XX 功能分支合入主干」**。


**`git rebase` 分支名**

#### 作用

重排提交记录，把自己所有提交接到主线最新节点后，提交历史一条直线。（`rebase` 本身会自动 commit）

#### 比喻

- 把 `main` 主干想象成**高速主干道**，所有人代码必须有序排队、一条直线往前走；
- 你拉了 `feature` 个人分支，相当于开到路边临时服务区写代码；
- `git rebase` = 等主干道所有车全部通行完毕，把你在服务区做的所有修改，拆下来、重新粘贴到主干道最新末尾排队，绝不插队、不出现分叉路口，保证主干道永远一条直线。

#### 两大核心使用场景

##### 场景一：个人功能分支，同步主干最新代码（最常用）

你在服务区（feature）写了 3 次代码，这期间主干道（main）同事又新增了好几轮提交，你想把最新代码拿过来，**同时不想产生杂乱的合并分叉记录**。

```bash
# 切到自己的功能分支
git switch feature

# 拉取远程最新分支
git fetch origin

# 把当前分支基于最新主干进行变基
git rebase origin/main

# 若出现冲突手动解决，然后 add / rebase(rebase本身会自动commit)，不想处理就直接 git rebase --abort
git add .
git rebase --continue

# 变基完成，再 push
git push
```

##### 场景二：整理个人提交历史，美化提交记录（合并多次细碎提交）

你在本地写需求频繁提交了 5 次小记录（修复拼写、临时调试等），想合并成 1 条干净的正式提交再提 MR。

```bash
# 执行交互式变基，整理最近 5 次提交
git rebase -i HEAD~5

# 把后面提交的 `pick` 改成 `squash`，多条提交压缩合并为一条
# 重新填写规范的提交备注，完成历史整理

# 个人分支首次推送直接 `git push`；如果之前推送过，用安全强推：`git push --force-with-lease`
```

> [!danger] 禁忌
> - **未推送到远程**：只要你的提交还只存在于本地、**从来没有 push 推送到远程仓库**，随便 `rebase`，完全安全，没有任何团队灾难风险。
> - **已推送到远程**：已经推送到远程的公共分支**禁止 rebase**！

#### 举例说明

**一份共享文档（公共 main 分支），初始版本：A → B → C（文档写到第 3 版）**

**正常安全场景**（规范操作，无 `rebase`、不强推）：

1. 你拉取 C，新增一段内容 E，提交推送到共享文档：现在文档 `A-B-C-E`
2. 同事小甲拉取这份最新文档（带着 E 内容），复制一份自己的草稿（feature 分支），在 E 后面写了业务功能 F，一直存在自己电脑没合并。
3. 另一个同事在共享文档继续更新，在 E 后面写了 D，推送线上：共享文档 `A-B-C-E-D`

现在小甲要把自己写的 F 合并到线上文档：

- 公共主线：`A → B → C → E → D`
- 小甲分支：`A → B → C → E → F`

两条分支**从 E 之后才分开**，分叉距离很短。Git 只会对比：D 和 F 这两段新增内容。哪怕有冲突，也就少量几处，改一改就能合并。

最终文档：`A-B-C-E-D-F`，逻辑顺畅，没有重复内容。

**事故场景**：你在公共 main 做了 `rebase` + 强推

前面步骤一样：

1. 共享文档：`A-B-C-E`，小甲基于 E 写了本地 F 未合并。
2. 同事更新线上文档到：`A-B-C-E-D`。
3. 你本地没拉最新，在旧文档 `A-B-C-E` 执行变基，把 E 的内容挪到 D 后面生成新文档版本 E'，然后强推覆盖线上。

强推后线上公共文档变成：`A-B-C-E-D-E'`。E' 里面的文字内容，和最早的 E **一模一样**。

现在小甲来合并自己的分支（A-B-C-E-F），两条分支分叉节点是 **E**，从 E 开始两条路彻底走岔：

1. 公共文档路线：E 后面写了 D，又重复粘贴了一遍 E 的内容（E'）
2. 小甲的私人草稿路线：E 后面直接写了 F

真实灾难性影响：

- **大范围冲突** — Git 需要从 E 这个节点开始，对比往后所有修改：主线改了 D、又重复加了一遍 E；小甲改了 F。不再是几行冲突，而是大量文件、大量代码需要人工核对，一不小心就改错。
- **出现大量重复业务代码** — E' 和最早的 E 内容完全一样，合并后你的功能代码会在项目里出现两遍。比如你新增的用户登录逻辑，代码重复两份，极容易出现：接口重复注册、变量重复定义、循环重复执行，线上直接报错崩溃。
- **逻辑混乱，bug 很难排查** — 原本只需要一次登录检验，现在代码里两处一模一样的检验，谁都不知道该删哪一段，删错就会功能失效。

更极端：你交互式 `rebase` 直接删掉了 E 再强推。

- 线上文档变成：`A-B-C-D`，彻底找不到最早的 E 版本。
- 小甲的草稿是基于 E 写的 F，合并的时候 Git 会认为：E 里所有你新增的业务代码都是"多余内容"，会直接删掉这部分代码。
- 合并上线后，你做的登录功能直接消失，线上业务瘫痪。

在小甲打开合并冲突文件那一刻，他看到的现象是：同一个文件里，出现**两段完全一样的登录检验代码**。

他不知道背后是你把旧 E 挪位置生成了 E'，他只会疑惑三件事：

1. 为什么同样的登录逻辑，主线现在写了两遍？
2. 第一份 E 里的登录代码是老基础代码，第二份 E' 是后来谁重复写的？
3. 我自己的权限 F 是依赖第一份登录代码写的，如果删掉第一份，我的 F 会不会直接报错？如果删掉第二份，会不会误删别人刚提交的需求？

他没有上帝视角，不知道 `E` 和 `E'` 是同一次改动的两次快照，只能小心翼翼逐行对比所有文件：

- 不敢随便删任意一段登录代码；
- 只能一段段看上下文、查提交记录、找当事人确认；
- 项目文件一多，核对工作量直接翻几十倍。

#### 附属命令

- `git rebase --continue`：冲突修复后继续变基
- `git rebase --abort`：放弃本次变基，回退到操作前


### 查看类命令

**git status**

作用：查看文件当前状态（未跟踪、已修改、已暂存）

场景：不确定代码有没有 add、哪些文件被修改

**git log --oneline**

作用：简洁查看所有提交历史、提交 ID

场景：需要回滚版本、排查代码修改记录

**git diff**

作用：查看未暂存的代码具体改动内容

场景：忘记自己改了哪些内容，核对代码修改

### 代码临时储藏

**git stash**

作用：把未提交的代码临时存起来，工作区恢复干净

场景：功能写到一半，需要切分支紧急改 bug，暂时不想提交半成品

**git stash pop**

作用：把储藏的代码恢复到当前分支，同时清空储藏记录

### 撤销回滚

**git reset HEAD .**

作用：把已经 `git add` 的文件从暂存区撤回

场景：误把不需要的文件提交到暂存区

**git checkout -- 文件名**

作用：放弃单个文件所有未暂存修改，恢复到上次提交版本

场景：代码写崩，想要丢弃本地修改

**git reset --hard 提交 ID**

作用：强制回退到指定历史版本，清空所有本地修改

> [!danger] 高危操作，谨慎使用

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

**Fast-forward（快进合并）**：不需要创建新 commit，直接把指针往前移。

- 最开始只有一份原稿 `main`，内容：第 1~5 章
- 你拉了个新分支 `xiaoming`，准备写第 6、7 章
- **在你写稿这段时间里，没人去动主分支 main**，出版社那边没改任何内容
- 你在自己分支写完第 6、7 章，提交保存

合并过程：主分支根本不需要重新整合两份文件，只是把 `main` 的指针**直接往前挪一步**，挪到你写完的最新稿子位置。出版社直接把你写好的第 6、7 章，接在原有 5 章后面，稿子顺序连贯，没有出现两个版本，**不会生成新的合并记录**。

**普通 merge 三方合并**（出现分叉，必须新建合并记录）：

1. 初始 `main`：1~5 章
2. 你切分支 `xiaoming`，去写 6、7 章
3. **就在你写稿的时候，同事直接在主分支 main 改了内容**：他在 main 里新增了第 6 章番外

现在出现两条完全不一样的稿件路线：

- 你的分支：1-5 章 + 你写的第 6、7 章
- 主分支：1-5 章 + 同事写的第 6 章番外

合并过程：系统要拿「最初共同的原稿（1-5 章）」当基准，对比你和同事两边的修改，整合出一份全新终稿，**自动生成一条全新的「合并提交记录」**。如果你们俩都改了第 6 章，就会出现冲突，需要手动选保留谁的文字。

**为什么推荐保持 fast-forward**：

1. 历史是直线，`git log` 干净
2. 没有 merge commit，commit 数量少
3. 方便回滚，`git reset --hard` 就能直接退回

团队项目里保持线性历史的方法：

- **Squash and merge**：把 feature 多个 commit 合成一个，再 fast-forward
- **Rebase and merge**：把 feature rebase 到 main 最新提交之后，再 fast-forward

### 切换分支规则

核心规则：**凡是会"修改某个分支指向"或"把改动合入某个分支"的操作，都必须在那个分支上执行。**

| 操作     | 口诀             | 必须在哪个分支上执行 | 示例                                                 |
| ------ | -------------- | ---------- | -------------------------------------------------- |
| merge  | "合到哪，切到哪"      | 目标分支（合入方）  | `git switch main` → `git merge feat/login`         |
| rebase | "谁要被挪，谁就是当前分支" | 被变基的分支     | `git switch feat/login` → `git rebase origin/main` |
| pull   | 切到要更新的分支       | 目标分支       | `git switch main` → `git pull origin main`         |
| reset  | 切到要重置的分支       | 被重置的分支     | `git switch main` → `git reset --hard HEAD~1`      |
| 删除分支   | 不能在当前分支上删自己    | 离开目标分支     | `git switch dev` → `git branch -d main`            |
| push   | 多数情况可不切        | 任意分支       | `git push origin feat/login`                       |

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

**何时需要切到 main？**

- 日常在自己功能分支同步代码：不需要切 main，直接 `git pull --rebase` 即可
- 准备上线合并前：必须切 main 拉最新，防止本地 main 太旧导致合并后推远程报错

```bash
git switch main
git pull
git merge feature
git push
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

**pull 的两种安全模式**：

|                   | `--ff-only`                 | `--rebase`                     |
| ----------------- | --------------------------- | ------------------------------ |
| 等价于               | `fetch` + `merge --ff-only` | `fetch` + `rebase origin/main` |
| 能 fast-forward 时 | 合并成功                        | 合并成功                           |
| 不能 fast-forward 时 | **直接报错，什么都不做**              | 改写本地未推送的提交历史（产生新 hash）         |
| 改写提交 hash         | 否                           | 是（X → X'）                      |
| 适用场景              | 本地没新提交                      | 本地有新提交想同步远端                    |

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

# 把本地 main 分支推送到远程 main 分支
git push -u origin main
```

> [!note] 为什么要 `git branch -M main`
> GitHub 2020 年起默认分支改为 `main`。本地分支叫 `master` 时直接 push 会报错，改为 `main` 可一次绑定成功。

> [!note] 为什么要 `git push -u origin main`
> `origin` 是默认的远程仓库别名。执行后本地 `main` ↔ 远程 `origin/main` 绑定成功，只需第一次推送时用一次，之后直接 `git push` / `git pull` 即可。
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

首次直接 clone 即可：

```bash
git clone xxxx
```

##### 个人功能分支

分支这条路只有你一个人走，直接 rebase 变基：

```bash
# 非首次
git fetch origin             # 拉取所有分支最新信息
git fetch origin main        # 只拉取主 main 最新
```

查看远程主线 main 和本地主线 main 是否同步：

```bash
git log HEAD..origin/main --oneline
```

有输出说明不同步：

```bash
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

##### 公共主线 main 分支

主线多人共用，**不能用 rebase**，而 merge 不会重写提交历史，所有修改都会留下合并记录，方便追溯、团队不会出现代码错乱。

```bash
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

```bash
git status

# 有未提交代码就先：
git add .
git commit -m "功能开发完成，准备合并上线"
```

再执行一遍拉取远程代码步骤，保持本地 main 和远程 main 到最新状态。

```bash
git push        # push 的是本地 feature 分支到远程仓库 feature 分支
```

切换到主线 main 分支，合并上线：

```bash
# 1. 切到主线
git switch main
# 2. 拉取远程主线最新代码（多人公共分支必须先更新本地 main）
git pull
# 3. 合并功能分支，推荐 --no-ff 强制保留合并记录
# 此时满足 fast-forward 条件，默认不会生成合并记录，--no-ff 强制生成一条合并提交
# 能清晰看到哪次提交、哪个功能分支合入上线，线上回滚、排查问题有据可查，企业团队强制规范
git merge --no-ff feature -m "上线：xxx功能"

# 推送到远程主线，正式上线
git push origin main         # 仅首次推送需要，因为要绑定远程仓库分支
git push                     # 非首次，之前已经 push 过了
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

## 参考

1. vscode搭配git工作流基础使用流程 — [VS Code使用Git可视化管理源代码详细教程 - 追逐时光者 - 博客园](https://www.cnblogs.com/Can-daydayup/p/14413914.html)
2. sourcetree：[【最全面】SourceTree使用教程详解（连接远程仓库，克隆，拉取，提交，推送，新建/切换/合并分支，冲突解决，提交PR） - 追逐时光者 - 博客园](https://www.cnblogs.com/Can-daydayup/p/13128633.html)

