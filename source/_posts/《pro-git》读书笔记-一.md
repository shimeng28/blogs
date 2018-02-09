---
title: 《pro git》读书笔记(一)
date: 2017-12-08 13:45:49
tags: git
---
<h3>第一章入门：</h3>
<p>VCS（版本控制系统）都分为本地版本控制系统，集中式版本控制系统（svn)， 分布式版本控制系统（git），<br />git是在Linux和bigkeeper的相爱相杀中产生的，git的保存的是快照，git的操作都是在本地进行，所以速度很快。<br />
在git中，文件有三种状态： 已修改（modified），已暂存（staged），已提交（committed）<br />
所以文件有四个阶段：未跟踪，工作目录，暂存区，git仓库。
 </p>
 
**git配置使用git config**
	例如
	
	git config user.name 'shimeng'  --global  //配置用户名
    git config user.email 'shimeng@gmail.com' --global 配置邮件
	   
配置变量保存在三个地方<br />

	etc/gitconfig文件
	～/.gitconfig
	当前的git仓库
	其中依次从下往上覆盖。
<h3>第二章Git基础</h3>
获取git仓库有两种方式：<br />
1. 通过git init 初始化一个仓库<br />
2. 通过git clone 【url】获取远程仓库<br />
<p>当有一个git仓库后，当修改或添加一个文件后，此时文件处于已修改或者未跟踪状态，<br />

通过git add 【filename】，可以使文件处于暂存区，通过git commit ，文件将处于已提交状态，这时它在git 本地仓库中。</p>

若是我们想直接将修改的文件提交本地，可以通过git commit -a -m ，这条命令会跳过暂存区(也就是git add命令)，直接将修改提交。

以上是一个文件的基本流程，我们可以通过git status获取到当前的状态。

那么如果我们想更详细的知道，我们做了哪些修改，我们可以通过git diff 获取到未添加到暂存区的更改，也就是工作目录的修改，这条命令会将当前工作目录的内容和暂存区的内容对比，显示还有哪些没有暂存的新更改。

通过git diff --staged 会返回暂存区的修改，这条命令会将暂存区的内容和上次提交内容对比，显示暂存区中没有提交的修改。

git diff --cached 和git diff --staged 的作用一样。

知道了以上这些，当我们发现我们想撤销一些操作的时候，我们应该用哪些命令？

如果我们修改了一个文件的时候，这时我们可以使用 git checkout – <filename> 撤销文件的更改

如果我们git add 了一个文件，这时我们可以使用 git reset HEAD <file>  将撤销暂存区的修改。

 如果我们git commit 了一次提交，之后发现我们少提交了一些文件，这时可以使用 git add 先将要提交的文件置于暂存区，之后使用git commit --amend   提交文件，此时只会产生一次提交。
 
git rm <filename>  会将文件删除，工作目录和暂存区都将不存在，<br />
若想让文件在工作目录，但是从暂存区移除，这时我们可以使用

	git rm --cached <filename>
	git mv <filename> <newFilename>会移动文件到新文件(其中一个功能就是重命名)
	相当于 mv <filename> <newFilename>
	          git rm <filename>
	         git add <newFilename>
	         
若我们想要查看项目的提交历史，可以通过git log，这条命令会按照时间的顺序列出所有的提交，最新的提交在前面。<br />
其中git log -p会显示每次提交的差异，git log -<n> 会显示最近的n次提交  git log  --pretty=oneline 会在每一行显示一次提交。

当我们想讲文件与远程仓库关联时可以使用<br />
`git remote add <remote-name>  <url>`<br />
通过git remote 显示与当前分支关联的远程仓库。<br />
`git remote -v 显示远程仓库的URL
``git remote show origin 显示远程仓库的详细信息。`<br />

	通过git fetch <remote-name> 会拉去远程仓库的修改，但不会和本地分支合并。
	git pull <remote-name>会拉去远程的修改，并且会和本地分支合并。
	git push <remote-name> <branch-name> 会提交修改到远程仓库的分支。
	git remote rm <remote-name>删除远程仓库名
	git remote rename <remote-name> <new-remote-name>修改远程仓库名

git tag会列举所有的tag，<br />
git中标签分为两种：注释标签和轻量标签 注释标签包含完整的信息<br />
创建注释标签可以使用git tag a 《tag-name》<br />
轻量标签：git tag 《tag-name》<br />
git show <tagName> 查看详细信息<br />