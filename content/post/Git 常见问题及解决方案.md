---
title: Git 常见问题及解决方案
date: 2022-10-15
tags:
    - Git
---

# 本地领先远程一个 commit 的同时落后一个 commit

一般发生于本地写完准备 push 时发现有人已经 push 了一个 commit，这时直接 pull 会报错。

标准做法：假设当前分支为 master，先执行 git fetch 同步本地的 origin/master 分支，之后执行 git rebase origin/master 将本地 master 上领先的 commit 添加到远程分支（origin/master）的末尾，之后就可以正常 push。

切忌merge分支，否则会出现名为“Merge branch 'master' of xxx”的提交，非常令人困惑。

# 本地已经修改了一部分文件导致无法 pull

一般发生于本地正在改代码时，想要同步远程仓库。

标准做法：使用 git stash 暂存更改，然后正常 pull，之后使用 git stash pop 应用之前的更改。

遇到任何“本地有修改无法执行操作”时都可以使用本方法。

# push 到远程的 commit 存在问题

标准做法：git commit --amend && git push --force-with-lease，也可以 git revert && git push —force-with-lease 或者 git reset && git push --force-with-lease

当然更推荐在 push 前仔细检查下自己的 commit 有没有错误。

# lock 文件冲突

一般发生于 package-lock.json 或 pnpm-lock.yaml 文件冲突时。

标准做法：npm 官方文档提供了一种[标准做法](https://docs.npmjs.com/cli/v6/configuring-npm/package-locks#resolving-lockfile-conflicts "标准做法")，当 lock 文件冲突时，可以直接重新 install（前提是 package.json 文件正常），也可以使用 npm install —package-lock-only 选项，这样 npm install 就不会真的去修改 node\_modules 内的文件，而是只修改 lock 文件。pnpm 下可以使用 pnpm install —lockfile-only 只修改 pnpm-lock.yaml 文件。

文末推荐一个 git playground [https://learngitbranching.js.org/?locale=zh\_CN](https://learngitbranching.js.org/?locale=zh_CN "https://learngitbranching.js.org/?locale=zh_CN")。
