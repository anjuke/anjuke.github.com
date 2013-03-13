---
layout: post
title: "GitCorp FAQ"
date: 2012-04-24 14:12
author: erning
comments: true
categories: [guide, git]
---

### 怎样获得自己的代码仓库目录?

首先以公司的域用户登录[GitCorp][1]站点，找到[管理SSH公钥][2]的页面。然后根据提示增加你的SSH公钥。提交之后，系统将创建和域用户名相同名称的目录，作为你的代码仓库的根目录。

GitCorp允许一个帐号拥有多个公钥，这样你可以在不同的机器上访问你的代码仓库而不需要共享私钥。

### 怎样新建用户代码仓库?

登录[GitCorp][1]站点，选择"Repo"菜单，然后点击"New"按钮创建新的代码仓库。创建代码仓库名要求输入仓库名称($REPO)和描述，新的仓库将建立这你的个人目录($USER)下。

之后可以通过以下命令，将代码仓库克隆到本地。

    git clone git@git.corp.anjuke.com:($USER)/($REPO)

或者在本地已经有代码仓库了，就可以push到GitCorp的用户目录下。

    git remote add origin git@git.corp.anjuke.com:($USER)/($REPO)
    git push origin master

<!-- more -->

### 怎样删除用户代码仓库?

登录[GitCorp][1]站点，选择"Repo"菜单，然后进入要删除的仓库。选择"Admin"，可以看到"Delete"选项。

只能删除自己目录下的代码仓库。

### 如何修改代码仓库在cgit/gitweb上的描述?

登录[GitCorp][1]站点，选择"Repo"菜单，然后进入要删除的仓库。选择"Admin"，可以修改仓库的描述。

### 如何fork其他代码仓库?

这[GitCorp][1]站点提供Fork功能之前，必须首先将其他代码仓库clone到本地，然后push到自己的代码仓库。

    git clone --mirror ($GIT_REPO_URL) ($LOCAL_REPO)
    cd ($LOCAL_REPO)
    git remote add gitcorp git@git.corp.anjuke.com:($USER)/($REPO)
    git push --mirror gitcorp

注意：在push之前需要通过Web界面先创建一个空的仓库。

### 如何merge其他工程师的改进到自己的代码仓库?

假设自己的代码仓库在`git@git.corp.anjuke.com:my/repo.git`，另一工程师的代码仓库在`git@git.corp.anjuke.com:peer/repo.git`。

首先将自己的代码仓库clone到本地

    git clone git@git.corp.anjuke.com:my/repo.git
    cd repo.git

增加对方的代码仓库，并获取更新

    git remote add peer git@git.corp.anjuke.com:peer/repo.git
    git fetch peer

然后就可以merge对方的代码，例如

    git checkout master
    git merge --no-commit peer/master

最后将合并好的代码push到自己的代码仓库

    git push origin master

### 如何直接将改动push到其他人的代码仓库?

这需要对方用户代码仓库的写权限，因此要对方操作，开放该代码仓库的写权限。

改变仓库的权限可以通过Web界面进行。登录[GitCorp][1]站点，选择"Repo"菜单，然后进入要修改的仓库。选择"Admin"，可以给其他用户赋予写权限。


  [1]: http://git.corp.anjuke.com
  [2]: http://git.corp.anjuke.com/account/public_keys
  [3]: http://git.corp.anjuke.com/cgit
  [4]: http://git.corp.anjuke.com/gitweb
