---
title: Git rebase 实战：避免合代码时的"merge 地狱"
description: 用 rebase 保持线性历史，并理解它什么时候不该用
date: 2026-08-13
slug: git-rebase-in-practice
categories:
    - 工具链
tags:
    - Git
toc: true
draft: false
---

`git pull` 默认会创建 merge commit，时间一长 `main` 上就长成毛线球。`rebase` 能让本地未推送的提交"重演"到最新分支顶端，产生线性历史。但**只对未推送的本地提交 rebase**——一旦提交被推到了共享分支，rebase 改写历史会让队友的本地仓库脱节。

## 日常流程

```sh
# 切到 feature 分支
git checkout feature/login

# 拉取 main 上的最新提交，重演到当前分支顶端
git fetch origin
git rebase origin/main

# 解决冲突后继续
git rebase --continue
# 或放弃
git rebase --abort
```

常见工作流：`feature/...` 分支在合到 `main` 之前先 rebase 一次，`main` 历史会是干净的直线。

## 交互式 rebase 整理提交

推送前常用 `git rebase -i HEAD~n` 把本地散落的 commit 合并、拆分、改写：

```text
pick a1b2c3 修复登录页样式
squash d4e5f6 同上的小幅调整
reword f7g8h9 WIP: 加 token 检查
pick i0j1k2 完成 token 校验
```

`pick` 保留，`squash` 合并到上一个，`reword` 改写 commit message，`drop` 丢弃。

## 千万别 rebase 公开提交

```sh
# 反例：team 里三个人都在 dev 分支上推过
git checkout dev
git rebase main   # 你重写了 dev 历史，其他人 pull 后会大量冲突
```

替代方案：把要清理的提交挪到一个临时分支，由原作者 rebase 后用 `git push --force-with-lease` 推一次，**其他人 rebase 一次再说**——但这只在团队约定一致的情况下可行。

## 几个救命命令

- `git rebase --abort`：放弃当前 rebase，回到 rebase 之前
- `git rebase --continue`：解决完冲突后继续
- `git reflog`：rebase 搞砸了可以在 reflog 里找到旧 commit hash，`git reset --hard <hash>` 救回来
- `git push --force-with-lease`：比 `--force` 安全，远程有你不知道的提交时会拒绝覆盖

## 总结

- **本地提交随便 rebase**，历史干净
- **推送过的提交不要 rebase**，除非团队约定且用 `--force-with-lease`
- 出问题先 `reflog`，几乎都能找回
