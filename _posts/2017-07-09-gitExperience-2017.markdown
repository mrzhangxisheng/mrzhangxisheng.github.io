---
layout:     post
title:      "git心得"
subtitle:   " \"学无止境\""
date:       2017-07-09 14:27:00
author:     "Zhangxs"
header-img: "img/post-gitbg-2017.jpg"
catalog: true
tags:
    - 技术
---



## 概述

最近再写博客，代码的上传都有用到git，踩了很多坑，不说废话直接上代码....

<p id = "build"></p>
---

## 博文
git的下载安装链接远程就不多说了，网上一搜一大堆。
安装好后一般都是，本机设置ssh key与远程用户github对应起来。例：
```
git remote add origin git@github.com:****/hello.git
```
---
不知道大家是否碰到git Bash命令窗口不能复制代码(其实是可以的,只要选中代码按住不松再按右键就ok了)。但在这里给大家介绍一个非常好用的工具[Conemu](https://www.fosshub.com/ConEmu.html)点击即可下载。
安装完之后打开界面,点击下拉找到 {Bash}-->{Git bash}
![Alt text](/img/2017-gitpost-1.jpg)
现在打开一个会报错。
```
error launching git
```
别急这只是git bash要在ConEmu中的路径没有配好
点击右上角设置-->Settings-->tasks--->{Bash::Git bash}把下面地址改成你git安装目录下bin\sh.exe就ok了
![Alt text](/img/2017-gitpost-2.jpg)
改好之后在按照第一步打开界面就是这样的了。
![Alt text](/img/2017-gitpost-3.jpg)
现在你就可以尽情复制粘贴了。这个工具远不止这一点好处，其他的就等你去发现了。
---
git的基础命令可以参考[GIT入门](http://www.liaoxuefeng.com)这里面的教程都是入门级。

我列举一些常用的。

相当于svn对项目的更新同步
```
git pull  
```
添加文件，add只是从工作目录添加到git的缓存区，并没有真正提交
```
git add fileName 

```
可根据命令git status 查看项目空间状态
```
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Untracked files:
  (use "git add <file>..." to include in what will be committed)

        img/2017-gitpost-1.jpg
        img/2017-gitpost-2.jpg
        img/2017-gitpost-3.jpg
        img/post-gitbg-2017.jpg

nothing added to commit but untracked files present (use "git add" to track)
```
意思是这四张图片既没有在暂存区也没添加到git里面，只是在工作目录下。那么我们现在来添加一张img/post-gitbg-2017.jpg.
```
$ git add img/post-gitbg-2017.jpg
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        new file:   img/post-gitbg-2017.jpg

Untracked files:
  (use "git add <file>..." to include in what will be committed)

        img/2017-gitpost-1.jpg
        img/2017-gitpost-2.jpg
        img/2017-gitpost-3.jpg
```
明显看到img/post-gitbg-2017.jpg已经被add了到了git的暂存区等待提交。那么我们现在来提交一张img/post-gitbg-2017.jpg再添加一张img/2017-gitpost-1.jpg
```
$ git commit  -m"img/post-gitbg-2017.jpg背景图片的添加"
[master 0742f3b] img/post-gitbg-2017.jpg背景图片的添加
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 img/post-gitbg-2017.jpg
$ git add img/2017-gitpost-1.jpg
$ git status
On branch master
Your branch is ahead of 'origin/master' by 1 commit.
  (use "git push" to publish your local commits)
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        new file:   img/2017-gitpost-1.jpg

Untracked files:
  (use "git add <file>..." to include in what will be committed)

        img/2017-gitpost-2.jpg
        img/2017-gitpost-3.jpg
```
观察仔细的是否发现by 1 commit。已经有了一个提交，img/2017-gitpost-1.jpg也到暂存区了。想查看提交的仔细内容可以用git log

---
好了，commit之后也只是在你本地的git仓库了，并没有提交到远程仓库中，所以这里需要push上去。
```
$ git push origin master
Counting objects: 10, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (10/10), done.
Writing objects: 100% (10/10), 212.87 KiB | 0 bytes/s, done.
Total 10 (delta 4), reused 0 (delta 0)
remote: Resolving deltas: 100% (4/4), completed with 2 local objects.
To https://github.com/mrzhangxisheng/mrzhangxisheng.github.io.git
   f53672d..d6fa76c  master -> master
```
-  origin：是你定义远程仓库的名称。也就是本博客第一条命令的添加的name
-  master 所提交的分支
-  如果你的用户名密码设置好了的话是可以直接提交的，不然会提示你输入用户名密码。
push成功就可以去看你远程github的代码是否有变化。

---
还有什么基础不解的可以看看[GIT入门](http://www.liaoxuefeng.com)

---


## 后记

学无止境。Tomorrow Will Be Better

—— Zhangxs 后记于 2017.0709
