---
title: "整理常用Git命令行"
catalog: true
date: 2020-07-7 21:51:24
subtitle: ""
header-img: "/img/default.png"
tags:
- Git
catagories:
---

咳咳，这个东东早就想要整理一下啦，以前用惯了Git Extensions和Sourcetree，很少用得上命令行。现在的工作环境由于很多安全因素的限制，迫不得已只能用命令行，好多次都是临时抱佛脚现去百度，好在慢慢适应一段时间也就习惯了，今天就整理一些常用的命令。

### 新建代码库
在当前本地目录新建Git代码库
$ git init
从远端下载一个项目和它的代码历史 
$ git clone [url]

### 分支管理
新建分支
$ git branch [branch_name]
查看本地分支
$ git branch
查看本地和远端所有分支
$ git branch -a 
删除分支
$ git branch -d [branch_name]
切换分支
$ git checkout [branch_name]
创建并切换指定分支
$ git checkout -b [branch_name]
合并指定分支
$ git merge [branch_name]

### 提交代码
提交变更的文件
$ git add [file]
提交所有变更的文件
$ git add .
查看变更
$ git status
提交变更到仓库区
$ git commit -m "message"
撤销上一次commit并保留代码
$ git reset --soft HEAD^
推送代码到远端
$ git push 
拉取代码到本地
$ git pull

### 修改配置
显示所有远程地址
$ git remote -v
修改远端地址
$ git remote set-url origin [url]
显示当前git配置
$ git config --list
修改配置信息
$ git config -e [--global]

### 解决冲突
一般解决代码冲突会推荐在本地解决，将两个冲突的分支都拉到本地，然后`run git merge`命令。之后会提示有哪些文件需要解决冲突，解决完成后将这些文件提交，然后`git commit -m "fix conflicts"`将冲突解决的结果同步到远程分支上去.








