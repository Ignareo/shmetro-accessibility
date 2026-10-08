# Fork 维护指南：私有改动与上游 PR 并存

本仓库是 `Hecate2/shmetro-accessibility` 的 fork。一部分改动（新城市变体、通用功能增强、
bug 修复）要回馈上游，另一部分（数据格式变更、本地数据文件）永远留在自己的 fork。
本文档讲解这套工作流的 git 机制与日常操作。

## 1. 仓库拓扑：三角结构

```
        Hecate2/shmetro-accessibility  (upstream, 只读)
              ^                  |
   fetch 同步 |                  | Pull Request
              |                  v
        Ignareo/shmetro-accessibility  (origin, 自己的 fork)
              ^                  |
        push  |                  | fetch/pull
              |                  v
              本地工作区 (working tree + .git)
```

```bash
git remote -v
# origin    git@github.com:Ignareo/shmetro-accessibility.git   ← 推这里
# upstream  git@github.com:Hecate2/shmetro-accessibility.git   ← 只 fetch
```

**机制要点**：remote 只是 `.git/config` 里的一条 URL 记录。`git fetch upstream`
之后，上游的分支以**只读引用** `refs/remotes/upstream/main` 存进本地对象库——
它和你自己的 `main` 是两条独立的指针，fetch 永远不会改动你的工作区或本地分支。

## 2. 分支布局

| 分支 | 基点 | 内容 | 去向 |
|---|---|---|---|
| `main` | upstream/main + 合并 + 私有提交 | 全部内容（生产用） | origin/main |
| `feat/crawl-progress-line-scope` | upstream/main | 线路过滤、进度状态行、frontend JSON | PR 到上游 |
| `feat/xiamen-variant` | upstream/main | 厦门变体（含漳州角美处理） | PR 到上游 |
| `chore/gitignore-venv` | upstream/main | .gitignore 加 .venv | PR 到上游 |
| `wip` | 任意 | 拆分前的完整快照 | 保险带，不删 |

**机制要点**：分支只是一个 41 字节文件（`.git/refs/heads/<name>`），内容是一个
commit hash。创建分支零成本；"分支包含什么"完全由它的 tip commit 沿父链回溯决定。

## 3. 心智模型：commit DAG

- commit 是**不可变**的对象：{tree（目录快照）, parents, message, author}。
  "修改历史"实际是生成新 commit、把分支指针挪过去，旧 commit 仍在对象库里。
- `merge`：新建一个有两个 parent 的 commit。保留两条历史的完整上下文。
- `rebase`：把一串 commit 的 diff 逐个重放到新基点上，生成**新 hash** 的提交。
  等价于"剪掉一段树枝嫁接到别处"。已推送的分支 rebase 后必须 force push。
- `cherry-pick`：单个 commit 的重放。同一个改动在不同分支上是不同 hash 的
  commit——git 靠 **patch-id**（diff 内容的指纹）识别"等价提交"，`git cherry`
  和 rebase 的跳过逻辑都靠它。

由此得到本工作流最重要的推论：

> **PR 分支必须基于 `upstream/main`，不能基于你的 `main`。** 否则 PR 会把
> 私有提交一起带进 diff。私有提交只许出现在"合并回 main"这一个方向上。

## 4. 日常操作

### 4.1 新改动先分类

写代码前先问一句"上游会要么"：

- 会要 → `git checkout -b feat/xxx upstream/main`，在干净基线上写
- 不会要（数据格式、本地约定、私货）→ 直接在 `main` 上写
- 不确定 → 在 main 上写，之后用 §5 的工具拆出去

### 4.2 同步上游（PR 被合并后尤其要做）

```bash
git fetch upstream
git checkout main
git merge upstream/main
```

PR 合并后，上游历史里出现了与你 feat 分支**内容相同但 hash 不同**的提交
（尤其维护者用 squash merge 时）。merge 时 git 做三方合并，发现两边改动
一致，自动消解为空，不会重复应用——这是"内容级"而非"提交级"的去重。
万一冲突，以 `upstream/main` 的版本为准（`git checkout --theirs <file>`），
因为你本地的那份已经通过 PR 进入了上游。

### 4.3 私有改动的两种维护流派

- **merge 派（本仓库采用）**：私有提交留在 main 上，定期 `merge upstream/main`。
  历史有分叉但从不改写，已推送也安全。
- **rebase 派**：`git rebase upstream/main` 让私有提交永远"浮在"上游最新提交
  之上，历史线性漂亮，但每次 hash 全变，需要 `push --force-with-lease`，
  且冲突要逐 commit 解决。只适合没有协作者的个人 fork。

### 4.4 反复出现的冲突：rerere

如果同一个冲突在每次同步时都出现（典型的：上游改了你私有改过的段落）：

```bash
git config rerere.enabled true
```

git 会把每次冲突的手工解法记在 `.git/rr-cache/`，下次遇到相同冲突自动应用。

## 5. 拆分工具箱（把混在一起的改动分开）

| 场景 | 命令 |
|---|---|
| 交互式按 hunk 暂存 | `git add -p` |
| 从别的分支拿**整个文件** | `git checkout wip -- path/to/file` |
| 交互式从别的分支挑 hunk | `git checkout -p wip -- path/to/file` |
| 只要暂存区、不动工作区 | `git restore --source=wip --staged <file>` |
| 提交前自检拆对了没 | `git diff --cached upstream/main` |

**本次实操的教训**：`git checkout` 切分支时，凡是"在源分支被跟踪、在目标分支
不存在"的文件会被**从磁盘删除**。这就是切换分支时 5 个 `.txt` 和厦门 HTML
一度消失的原因——它们已提交进 wip，对象库里有完整快照，`git checkout wip --
*.txt` 一秒恢复。规则：**大改之前先做一个 wip 快照提交**，git 里提交过的东西
几乎丢不掉。

## 6. 事故恢复

```bash
git reflog            # 本地所有分支指针的移动历史，误删分支/误 reset 都能找回
git checkout -b rescue <hash>   # 从 reflog 里的任意 hash 拉一条救命分支
```

对象库里的 commit 默认 90 天才会被 gc 回收，所以"先提交再说"永远是最安全的策略。

## 7. 本仓库速查

```bash
# 同步上游到 fork main
git fetch upstream && git checkout main && git merge upstream/main && git push

# 开一个新的上游 PR
git checkout -b feat/yyy upstream/main
# ...写代码...
git push -u origin feat/yyy
gh pr create --repo Hecate2/shmetro-accessibility --base main --head Ignareo:feat/yyy

# PR 合并后把 feat 分支合回 main（或等 §4.2 的同步自然回流）
git checkout main && git merge upstream/main && git push
git branch -d feat/yyy   # 已合并，安全删除
```
