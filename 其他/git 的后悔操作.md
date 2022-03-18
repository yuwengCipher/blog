---
title: git 的后悔操作
date: 2021-08-22
categories:
 - Git
tags:
 - Git
---

## 前言

人非圣贤，孰能无过。在我们使用 git 的过程中，不可避免的会有各种误提交，例如 `commit message` 存在错别字，或者提交了不该提交的文件等。遇到这些情况，先不要慌，因为 `git` 拥有很强的能力，我们只需要好好利用就可以补救了。

补救的操作分为两大类，分别是撤销和回滚。对本地的操作称为撤销，而对远程仓库的操作称为回滚。

需要注意的是：如果是团队项目，回滚操作风险很大，因为更改之后会给同事本地状态造成影响，所以必须先备份并告知同事。如果是个人项目，那就随便怎么整 -。-

## 撤销

### 撤销工作区修改

文件被修改了，但还没有 `add`，此时只是工作区发生了改变，就可以使用 `checkout` 命令

撤销单个文件

```bash
git checkout a.txt
```

撤销所有修改

```bash
git checkout .
```

### 撤销索引区修改

文件被修改了，并且执行了 `add`，那么需要使用 `reset`

撤销单个或多个文件

```bash
git reset HEAD a.txt b.txt
```

撤销所有文件

```bash
git reset HEAD
```

### 撤销本地仓库修改

撤销最近一次提交并直接修改

```bash
git commit --amend，此时会进入 vi 模式，重新编辑好 commit message 之后保存退出
git push -f 强制提交
```

越过多条去撤销指定一次 `commit`，有三种方式

一种是使用 `reset`，它会撤销 `commit`

```bash
git reset --hard commit-hash(前一次的 commit)
# 或者
git reset --soft commit-hash(前一次的 commit)
```

使用 `--soft` 会保留文件的修改，只是撤销了这次的 `commit`，而 `--hard` 则不仅撤销 `commit`，还会清除暂存区的修改。另外 这里的 `commit-hash` 是需要你撤销的 `commit` 的前一次 `commit` 的 `hash` 值。因为 `reset` 实际上是移动 `HEAD`，这里就是将 `HEAD` 指向前一次，那么在它后面的 `commit` 都会被丢失。

使用 `reset` 的一个缺点就是指定 `commit` 后的 `commit` 都会被撤销，如果只是想撤销某一次，就需要使用 `revert`。

```bash
git revert HEAD
# 或者
git revert commit-hash
```

输完命令后，就可以进入 `vi` 模式，编辑此次逆向操作的 `commit message`，保存退出查看 `commit` 记录，你会发现你想撤销的那次 `commit` 还在，但是修改的记录被撤销了。

使用 `rebase`

```bash
git rebase -i HEAD~3 # 选择最近三条记录进行修改，其中包括需要更改的，进入 vi 模式
```

![rebase](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/rebase.png)

每条记录前面都有一个处置的 `command`，你可以设置成相应的，此处我们可以使用 `edit`，`reword`，`drop`。

- `edit` 允许你修改 `commit message` 和内容
- `reword` 只允许修改 `commit message`
- `drop` 丢弃 `commit message` 和 修改

1. 对于 `drop` 的 `commit`，会直接执行成功，丢弃 `commit`
2. 对于使用 `reword` 和 `edit` 的 `commit`，在此次保存退出后，会分别有一个针对它们的 `vi` 操作界面。我们要做的就是编辑 `message` 保存退出。此时会有两种结果，如果是 `reword`，直接就结束修改；如果是 `edit`，会出现如下提示：

![rebase_step_1](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/rebase_step_1.png)

意思就是你还需要进行两次操作，使用 `git commit --amend` 修改，使用 `git rebase continue` 继续 `rebase`。

- 执行 `git commit --amend`，修改之后如下，可以看到 `rebase` 的第二步完成了
  
![rebase_step_2](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/rebase_step_2.png)

- 执行 `git rebase continue`，继续最后一步操作，这样就完成了一次 `rebase`。

![rebase_step_3](https://coding-pages-bucket-3560923-8733773-16868-593524-1259394930.cos-website.ap-hongkong.myqcloud.com/blogImgs/rebase_step_3.png)

## 回滚

使用 `git commit --amend`

1. `git commit --amend`，此时会进入 `vi` 模式，重新编辑好 `commit message` 之后保存退出
2. `git push -f` 强制提交

结束后就可以看到原有的 `message` 被替换了。

使用 `git reset`

```bash
git reset --hard commit-hash(前一次的 commit)
# 或者
git reset --hard HEAD^

git push -f
```

使用 `revert`

```bash
git revert commit-hash
# 或者
git revert HEAD

git push
```

## vi 模式基本操作

在 `bash` 环境下，使用如下命令可以打开文件进入 `vi` 模式

> vi filename

按 `i` 或这 `s` 键进入编辑模式，其中 `s` 会同时删除一个字符。

而在这里，我们使用 `rebase` 也是直接进入 `vi` 模式。

修改完成后，按 `ESC` 退出编辑模式，输入 `:wq` 可以退出 `vi` 模式。

## 最后

- 修改操作涉及工作区、暂存区、本地仓库、远程仓库
- 后悔操作尽量在本地，对远程的修改要慎重
