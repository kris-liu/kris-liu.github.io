---
title: Git常用命令
date: 2015-09-01 00:00:00
categories: Git
tags: Git
---

```
git init 通过git init命令创建一个Git可以管理的仓库
	liuxin@liuxin-ProBook:~/Work/test$ git init gittest
	初始化空的 Git 版本库于 /home/liuxin/Work`/test/gittest/.git/
	liuxin@liuxin-ProBook:~/Work/test$ cd gittest/
	liuxin@liuxin-ProBook:~/Work/test/gittest$ ll
	总用量 12
	drwxrwxr-x 3 liuxin liuxin 4096  5月 13 10:42 ./
	drwxrwxr-x 3 liuxin liuxin 4096  5月 13 10:42 ../
	drwxrwxr-x 7 liuxin liuxin 4096  5月 13 10:42 .git/

git clone git@server-name:path/repo-name.git 克隆远程仓库到本地
git remote add origin git@server-name:path/repo-name.git 将本地仓库关联一个远程仓库

git add readme.txt把文件修改添加到暂存区
git add .把所有文件修改添加到暂存区
git add -A把所有文件修改添加到暂存区

git commit -m 'message' 把暂存区的所有内容提交到当前分支，一定要带提交日志

```

<!--more-->

```
git branch 显示分支 *开头的是当前分支
	liuxin@liuxin-ProBook:~/Work/Projects/training$ git branch
	* liuxin
	  master
	  zhao

git baranch -avv 显示所有分支的详细信息
	liuxin@liuxin-ProBook:~/Work/Projects/training$ git branch -avv
	* liuxin                        2a8804e [origin/liuxin] update pom
	  master                           924d69f [origin/master] xxx standard initialization
	  zhao                     ab33686 [origin/zhao] change project structure
	  remotes/origin/HEAD              -> origin/master
	  remotes/origin/cheng     35234e2 modify question
	  remotes/origin/yang        fae3081 update resources

git branch dev origin/master 根据远程master分支创建分支
git branch -d dev 删除分支，-D强行删除，即使该分支未被合并
git checkout branch 切换分支
git checkout -b branch origin/master 根据远程master分支创建并切换分支

git checkout -- readme.txt 撤回工作区文件的修改。
git checkout . 撤回工作区的全部修改

git status 可以让我们时刻掌握仓库当前的状态
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git status
	位于分支 master
	要提交的变更：
	  （使用 "git reset HEAD <文件>..." 撤出暂存区）

		修改：     readme.txt

	尚未暂存以备提交的变更：
	  （使用 "git add <文件>..." 更新要提交的内容）
	  （使用 "git checkout -- <文件>..." 丢弃工作区的改动）

		修改：     readme.txt

git status -s
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git status -s
	MM readme.txt
	第一列的M代表的是暂存区和版本库有差异
	第二列的M代表的是工作区和版本库有差异

git diff 就是查看difference，查看工作区和HEAD的差异
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git status
	位于分支 master
	尚未暂存以备提交的变更：
	  （使用 "git add <文件>..." 更新要提交的内容）
	  （使用 "git checkout -- <文件>..." 丢弃工作区的改动）

		修改：     readme.txt

	修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git diff
	diff --git a/readme.txt b/readme.txt
	index 1f236c3..247024d 100644
	--- a/readme.txt
	+++ b/readme.txt
	@@ -1,2 +1,3 @@
	 git is free
	 git is good
	+git is nice

git diff --cached 查看暂存区和HEAD的差异
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git status
	位于分支 master
	要提交的变更：
	  （使用 "git reset HEAD <文件>..." 撤出暂存区）

		修改：     readme.txt

	liuxin@liuxin-ProBook:~/Work/test/gittest$ git diff --cached
	diff --git a/readme.txt b/readme.txt
	index 1f236c3..247024d 100644
	--- a/readme.txt
	+++ b/readme.txt
	@@ -1,2 +1,3 @@
	 git is free
	 git is good
	+git is nice

git push -u origin liuxin（HEAD） 第一次推送master分支的所有内容，第二次不用加-u
	liuxin@liuxin-ProBook:~/work/test/demos$ git push -u origin liuxin
	Total 0 (delta 0), reused 0 (delta 0)
	To git@server-name:path/repo-name.git
	 * [new branch]      liuxin -> liuxin
	分支 liuxin 设置为跟踪来自 origin 的远程分支 liuxin。


git push origin :liuxin  删除远程liuxin分支
	liuxin@liuxin-ProBook:~/work/test/demos$ git push origin :liuxin	liuxin@liuxin-ProBook:~/work/test/demos$ git push origin :liuxin
	To git@server-name:path/repo-name.git
	 - [deleted]         liuxin

	To git@server-name:path/repo-name.git
	
git merge 合并指定分支到当前分支
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git merge liuxin
	更新 e8c3536..b1aaac9
	Fast-forward
	 readme.txt | 1 +
	 1 file changed, 1 insertion(+)

git rm txt 在工作区删除已有文件到暂存区，删除后需要commit才真的删除
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git rm txt
	rm 'txt'

git rm -f abc 强制删除一个添加到暂存区的新文件
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git rm -f abc
	rm 'abc'

git stash 可以把当前工作现场“储藏”起来，等以后恢复现场后继续工作
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git stash
	Saved working directory and index state WIP on zzz: b50026a merge bbvvv
	HEAD 现在位于 b50026a merge bbvvv

git stash list 查看之前的工作现场
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git stash list
	stash@{0}: WIP on zzz: b50026a merge bbvvv

git stash pop 恢复工作现场的同时把stash内容也删了
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git stash pop
	位于分支 zzz
	尚未暂存以备提交的变更：
	  （使用 "git add <文件>..." 更新要提交的内容）
	  （使用 "git checkout -- <文件>..." 丢弃工作区的改动）

		修改：     readme.txt

	修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
	丢弃了 refs/stash@{0} (5fc274d7fe1e2027e702c8c3af29f29c6853dfb2)

git log 显示从最近到最远的提交日志
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git log
	commit 892db85b816e3f7ca7adadef37bda0c9add7ad08
	Author: liuxin <895195774@qq.com>
	Date:   Wed May 13 11:43:16 2015 +0800

	    add reset

	commit cf7a405ba00c92af371104ebee39bf1fe8b2b25f
	Author: liuxin <895195774@qq.com>
	Date:   Wed May 13 11:40:48 2015 +0800

	    add nice

git log --pretty=oneline 单行显示提交日志
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git log --pretty=oneline
	892db85b816e3f7ca7adadef37bda0c9add7ad08 add reset
	cf7a405ba00c92af371104ebee39bf1fe8b2b25f add nice
	731e31b625ed1e95df267daec37cdd39ccb426b7 add good
	eaad49cd3e88129e00dd65d7419e9f3052d74197 add git free

git log --graph --pretty=oneline --abbrev-commit 可以看到分支合并图
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git log --graph --pretty=oneline --abbrev-commit
	*   5569274 merge bc
	|\
	| * 8213220 insert b
	* | 4e04be8 insert c
	|/
	* b1aaac9 add aaa

git reflog 用来记录你的每一次命令、
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git reflog
	cf7a405 HEAD@{0}: reset: moving to HEAD~1
	892db85 HEAD@{1}: reset: moving to 892db85b
	cf7a405 HEAD@{2}: reset: moving to HEAD~1
	892db85 HEAD@{3}: commit: add reset
	cf7a405 HEAD@{4}: commit: add nice
	731e31b HEAD@{5}: commit: add good
	eaad49c HEAD@{6}: commit (initial): add git free

git reset HEAD readme.txt 撤出暂存区的修改到工作区
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git reset HEAD readme.txt
	重置后撤出暂存区的变更：
	M	readme.txt

git reset --hard HEAD~1(某一版本，可以是commit的版本号前几位数字) 回退到某一个版本
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git reset --hard HEAD~1
	HEAD 现在位于 cf7a405 add nice
	或
	liuxin@liuxin-ProBook:~/Work/test/gittest$ git reset --hard 892db85b
	HEAD 现在位于 892db85 add reset


git reset --hard de1ada01a0 撤销一次错误的merge
	~/work/projects/test$ git reset --hard de1ada01a0
	HEAD 现在位于 de1ada0 fix

git remote -v 查看远程库的详细信息
	liuxin@liuxin-ProBook:~/Work/Projects/training$ git remote -v
	origin	git@server-name:path/repo-name.git (fetch)
	origin	git@server-name:path/repo-name.git (push)
	liuxin@liuxin-ProBook:~/Work/Projects/training$
```