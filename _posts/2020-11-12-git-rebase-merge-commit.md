---
title:  "rebase 合并多个commit"
author: adream307
date:   2020-11-12 14:54:00 +0800
categories: [linux,git]
tags: [git]
---

设置编辑器为 `vim`
```bash
git config --global core.editor "vim"
```

当前 `commit` 日志信息如下:
- `master` 目前包含两条 `commit` : `a79fcc1` 和 `a79fcc1`
- 在 `master` 分支上切出一个分支 `tb`
- `tb` 包含 `3` 个 `commit` : `124b6c7`，`dd81b51`，`b44f3c1`

```log
* 124b6c7 (HEAD -> tb) 5
* dd81b51 4
* b44f3c1 3
* a79fcc1 (master) 2
* 9d38256 1
```

现在需要把 `tb` 的 `3` 个 `commmit` 合并成一个 `commit`

确保当前在 `tb` 分支
```bash
$ git branch
  master
* tb
```

`git rebase`
```bash
git rebase -i master
```

`git rebase` 命令之后，会出现一个 `vim` 的编辑页面
```log
pick b44f3c1 3
pick dd81b51 4
pick 124b6c7 5

# Rebase a79fcc1..124b6c7 onto a79fcc1 (3 commands)
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```

将其内容给成如下形式，第一个 `pick` 改成 `r`，其它 `pick` 均改成 `f`
```log
r b44f3c1 3
f dd81b51 4
f 124b6c7 5

# Rebase a79fcc1..124b6c7 onto a79fcc1 (3 commands)
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```

保存并退出会进入如下的 `vim` 编辑页面
```log
3

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
# Date:      Thu Nov 12 14:58:53 2020 +0800
#
# interactive rebase in progress; onto a79fcc1
# Last command done (1 command done):
#    reword b44f3c1 3
# Next commands to do (2 remaining commands):
#    fixup dd81b51 4
#    fixup 124b6c7 5
# You are currently editing a commit while rebasing branch 'tb' on 'a79fcc1'.
#
# Changes to be committed:
#       modified:   t.txt
#
```

编辑合并后的 `commit` 日志，保存并退出
```log
345

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
#
# Date:      Thu Nov 12 14:58:53 2020 +0800
#
# interactive rebase in progress; onto a79fcc1
# Last command done (1 command done):
#    reword b44f3c1 3
# Next commands to do (2 remaining commands):
#    fixup dd81b51 4
#    fixup 124b6c7 5
# You are currently editing a commit while rebasing branch 'tb' on 'a79fcc1'.
#
# Changes to be committed:
#       modified:   t.txt
#
```

合并后的 `commit` 日志信息
```log
* 0f4e0d8 (HEAD -> tb) 345
* a79fcc1 (master) 2
* 9d38256 1
```

