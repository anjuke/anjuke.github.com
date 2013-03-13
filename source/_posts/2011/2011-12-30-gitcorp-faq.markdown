---
layout: post
title: "GitCorp FAQ"
date: 2011-12-30 14:12
author: erning
comments: true
categories: [guide, git]
---

[FAQ更新，请访问新地址](/blog/2012/04/24/gitcorp-faq/)

<!-- more -->

--------------------

### 怎样获得自己的代码仓库目录?

首先以公司的域用户登录[GitCorp][1]站点，找到[管理SSH公钥][2]的页面。然后根据提示增加你的SSH公钥。提交之后，系统将创建和域用户名相同名称的目录，作为你的代码仓库的根目录。

GitCorp允许一个帐号拥有多个公钥，这样你可以在不同的机器上访问你的代码仓库而不需要共享私钥。

### 怎样新建用户代码仓库?

直接从用户目录下clone的新的代码仓库即可。如果GitCorp的目录下还没有相应代码仓库的系统将自动创建。

    git clone git@git.corp.anjuke.com:($USER)/($REPO)

或者在本地已经由代码仓库了，也可以直接push到GitCorp的用户目录下。

    git remote add origin git@git.corp.anjuke.com:($USER)/($REPO)
    git push origin master


### 怎样删除用户代码仓库?

用以下命令将代码仓库移到回收站。

    ssh git@git.corp.anjuke.com trash ($USER)/($REPO)

如果永久的删除代码仓库可采用`unlock`和`rm`两个命令。为了减小误删除，所有需要先unlock。
  
    ssh git@git.corp.anjuke.com unlock ($USER)/($REPO)
    ssh git@git.corp.anjuke.com rm ($USER)/($REPO)

只能删除自己目录下的代码仓库。

### 如何修改代码仓库在cgit/gitweb上的描述?

新建的代码仓库在[cgit][3]/[gitweb][4]上描述显示为“Unnamed repository; edit this file 'description' to name the repository.”。我们希望显示正确和美观的描述，可以使用这个命令

    echo "代码仓库的描述" | ssh git@git.corp.anjuke.com setdesc ($USER)/($REPO)

### 如何fork其他代码仓库?

首先从其他代码仓库(例如github的)clone到本地，然后push到自己的代码仓库。

    git clone --mirror ($GIT_REPO_URL) ($LOCAL_REPO)
    cd ($LOCAL_REPO)
    git remote add gitcorp git@git.corp.anjuke.com:($USER)/($REPO)
    git push --mirror gitcorp

如果需要fork的代码仓库也在GitCorp上，可以使用fork命令。这个操作类似github的fork。

    ssh git@git.corp.anjuke.com fork ($SOURCE_REPO) ($USER)/($REPO)

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

改变仓库的权限可以采用这样操作，首先编辑文件，设文件名为perms.txt，其内容如下，

    R    @all
    RW    ($USER1) ($USER2)
    RW    ($USER3)

这表示该代码仓库对USER1, USER2, USER3有读写的权限，其他用户有读权限。然后执行命令

    cat perms.txt | ssh git@git.corp.anjuke.com setperms ($USER)/($REPO)


  [1]: http://git.corp.anjuke.com
  [2]: http://git.corp.anjuke.com/account/ssh
  [3]: http://git.corp.anjuke.com/cgit
  [4]: http://git.corp.anjuke.com/gitweb
