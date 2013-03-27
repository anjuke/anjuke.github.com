---
layout: post
title: "Aifang GIT流程"
date: 2013-03-20 12:00
author: wbsong
comments: true
categories:
---

## GtiCorp - Aifang代码开发流程

### 远程代码仓库


####生产环境 `git@git.corp.anjuke.com:aifang/aifang-site`
生产环境的仓库只有**master**可用，**master**上面的代码都是经过测试并且可以直接上线的代码，只有项目需要上线的时候，才可以合并到**master**，否则是不允许合并到**master**，因为有可能有些上线流程没有做好，配置等没有完成，如果直接上线可能会出现一些问题。
**权限方面**生产环境只有部分人员有全部的权限，安全方面有一定保证。

####开发环境 `git@git.corp.anjuke.com:aifang/aifang-branch`
**aifang-branch**分支是开发人员用户最多的仓库，多人协作在这个仓库里面开发测试项目，这个仓库的**权限**对所有人员开放
### git全局参数
为了更好的使用git，需要设置一些全局化的参数
	
	git config --global branch.autosetuprebase always //从远程仓库创建分支的时候，自动加上rebase选项
	git config --global branch.master.rebase true //设置master分支自动rebase
	
	git config --global user.name xxx //为了方便沟通，设置自己的中文名称
	git config --global user.email xxx@anjuke.com //设置自己的anjuke邮箱
	
 
### 本地仓库
本地开发的时候，需要将**生产环境仓库**&**开发环境仓库**克隆到本地，使用如下的命令
	
	$ git clone git@git.corp.anjuke.com:aifang/aifang-site
	$ git remote add aifang-branch git@git.corp.anjuke.com:aifang/aifang-branch
	$ git pull aifang-branch
	
### 如何参加一个正在开发的分支例如`pmt7993-loupan_comm`已经在开发过程中
	$ git pull aifang-branch
	$ git branch -r | grep pmt7993-loupan_comm //确保这个分支存在
	$ git checkout pmt7993-loupan_comm //创建本地的pmt7993-loupan_comm分支，并且对应远程的pmt7993-loupan_comm分支
	// 经过一段时间的开发……
	// 需要提交代码
	$ git pull
	$ git add .
	$ git commit -am "xxxxx"
	// 可能要解决一些冲突
	$ git push aifang-branch pmt7993-loupan_comm //将本地的代码提交到远程仓库
	// 项目开发过程中，我们发现我们需要的一个功能，master上面已经存在了，这个时候，我们需要把master中的更新rebase到我们自己的分支
	$ git checkout master
	$ git pull
	// 通知所有开发此分支的同事提交代码
	$ git checkout pmt7993-loupan_commt 
	$ git pull //我们需要下载所有的更新
	$ git rebase master //将master中的代码更新过来
	//可能过程中有一些冲突
	$ git push -f aifang-branch pmt7993-loupan_commt //注意这个时候只能强制提交代码
	 
### 自己如何从头开发一个新的分支，例如开发`pmt10000-loupan`项目
	$ git pull origin //确保从最新的生产环境仓库创建分支
	$ git checkout master //切换到master分支
	$ git checkout -b pmt10000-loupan //从最新的master上面创建自己的分支
	$ git push aifang-branch pmt10000-loupan //将本地的pmt10000-loupan分支推送到远程仓库中，名称保持不变
	$ git branch --set-upstream-to=aifang-branch/pmt10000-loupan pmt10000-loupan //设置本地分支和远程分支之间的对应关系 
	$ ./build/set-remote.sh //上面2个命令可以使用仓库中的set-remote.sh来解决
	
### 如何在测试过程中fix bug	
项目开发完成后，提交测试，例如我们刚完成了`pmt10000-loupan`项目的开发，紧接着我们又开发了一个`pmt10001-crm`的项目，这个时候发现之前的项目`pmt10000-loupan`发现了一个bug
	
	//提交我们正在做的工作	
	$ git add .
	$ git commit -am "xxxx" //提交现有开发的代码
	$ git checkout pmt10000-loupan //切换到之前的分支
	$ git pull //下载最新的代码，这一点很重要
	// 完成bug的修改……
	$ git push aifang-branch pmt10000-loupan //提交我们的bug修改
	$ git checkout pmt10001-crm //切换回我们之前开发的分支
    //继续之前的工作
    
### 测试完成&合并上线，例如`pmt10000-loupan`项目要合并上线
	$ git checkout master
	$ git pull
	$ git checkout pmt10000-loupan
	$ git rebase master
	$ git checkout master
	$ git merge --no-ff pmt10000-loupan //将分支合并到master，采用no-ff的方式   
	$ git push origin master