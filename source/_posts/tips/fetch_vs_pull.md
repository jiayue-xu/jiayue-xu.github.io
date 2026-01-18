---
title: git fetch vs git pull
date: 2024-01-24 18:29:43
---

## 创建一个仓库

1. 在github上创建一个远程仓库

2. 在本地创建一个本地仓库，创建README.md，`push -u origin main`到远程仓库

<!--more-->

3. 远程仓库和本地仓库状态如下，都只有一个README.md
![远程仓库初始状态](push_remote.jpg)
![本地仓库初始状态](push_local.jpg)

## 执行fetch命令

4. 在远程仓库添加一个文件`fetch.txt`

5. 此时在本地仓库执行`fetch`命令，可以看到在工作区并没有出现一个`fetch.txt`文件
![本地仓库fetch](fetch-local.jpg)

6. 而且在本地执行`fetch`命令后，既没有出现新的`commit`记录，也没有出现新的`branch`。实际上，在本地执行`fetch`之后，会创建一个隐藏的引用`FETCH_HEAD`，指向了本地仓库中的一个临时副本，可以使用`git diff FETCH_HEAD`或`git show FETCH_HEAD`命令查看从远程仓库fetch的内容与本地当前工作区的区别。
![本地仓库fetch状态](fetch-status-local.jpg)

7. 查看了`fetch`的新内容之后，可以使用`merge`命令进行合并，此时工作区中就出现了`fetch.txt`文件
![本地仓库fetch后merge](fetch-merge-local.jpg)

## 执行pull命令

8. 在远程仓库再创建一个新文件`pull.txt`
![远程仓库创建pull.txt文件](pull-remote.jpg)

9. 在本地仓库执行`pull`命令，相当于先执行`fetch`命令将远程仓库新内容拉取到本地仓库（通过FETCH_HEAD可以查看），接着执行merge命令合并到当前分支。
![本地仓库pull](pull-local.jpg)

## 总结

git pull相当于是git fetch加上git merge命令，git pull会将远程仓库新内容拉取到当前工作区中，而git fetch只是将远程仓库新内容存本地仓库的一个地方。

