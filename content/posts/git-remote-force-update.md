---
title: "Git Remote Force Update"
date: 2019-03-09T05:41:02+08:00
tags: ["git", "tools"]
draft: false
summary: "git 覆盖已提交的分支"
---

## 场景

本地 commit 的时候忘记了设置用户名和邮箱，直接 push 了。在 github 上的 contributer 变成了我本地计算机上的用户名。

使用 `git config` 配置成 github 用户名和邮箱之后，使用 `git commit --amend --reset-author` 强行覆盖了本地分支。

但是想要 push 的时候被拒绝了：

```
→ git push origin master
To github.com:starliiit/starliiit.github.io.git
 ! [rejected]        master -> master (non-fast-forward)
error: failed to push some refs to 'git@github.com:starliiit/starliiit.github.io.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Integrate the remote changes (e.g.
hint: 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

google 之，解决办法是 force update。首先在 `git log` 中得到刚才覆盖后的 commit-hash，然后

```
git push --force origin <commit-hash>:<local-branch-name>
```

我的执行结果：

```
→ git push --force origin 16648de6c9cda050cb4e84001da9ca071f41f309:master
Counting objects: 10, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (9/9), done.
Writing objects: 100% (10/10), 5.55 KiB | 2.77 MiB/s, done.
Total 10 (delta 7), reused 0 (delta 0)
remote: Resolving deltas: 100% (7/7), completed with 5 local objects.
To github.com:starliiit/starliiit.github.io.git
 + 88de0b3...16648de 16648de6c9cda050cb4e84001da9ca071f41f309 -> master (forced update)
```

Reference: https://stackoverflow.com/questions/5816688/resetting-remote-to-a-certain-commit
