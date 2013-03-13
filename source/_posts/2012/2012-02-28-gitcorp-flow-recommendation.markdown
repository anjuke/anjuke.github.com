---
layout: post
title: "GitCorp Flow - 安居客Git开发流程建议"
author: erning
date: 2012-02-28 09:13
comments: true
categories: [guide, git]
---

这里我们说一下使用gitcorp来进行源代码管理的开发流程建议。在gitcorp里，每个工程师都可以创建自己的个人目录。除了个人目录外，有这么几个"官方"原始代码仓库

 * [anjuke](http://git.corp.anjuke.com/cgit/anjuke)
 * [aifang](http://git.corp.anjuke.com/cgit/aifang)
 * [haozu](http://git.corp.anjuke.com/cgit/haozu)
 * [jinpu](http://git.corp.anjuke.com/cgit/jinpu)
 * [mobile](http://git.corp.anjuke.com/cgit/mobile)
 * [corp](http://git.corp.anjuke.com/cgit/corp)

## 代码仓库 - Git Repositories

### 原始代码仓库

在这篇文章的例子中，我们以`git@git.corp.anjuke.com:corp/flow-demo.git`为原始代码仓库。文章后面提到的origin都是指原始代码仓库。

origin上只包含有正式的分支，例如: **master**，**beta**和**ga**。

* **master**
  主要的开发分支，待发布的新功能都在这个分支上。通常对这个分支进行每日构建、集成测试等；新的功能开发也都在这个分支的基础上进行。

* **beta**
  这个分支将发布到预发布环境，通常其HEAD指向当前的预发布版本。在确定需要发布代码到预发布环境时，将代码从master分支合并到beta分支。

* **ga**
  这个分支将发布到对外公开的环境，通常其HEAD指向当前的对外公开版本。在确定预发布版本可以对外公开时，将代码从beta分支合并到ga分支。

<!-- more -->

### 本地开发仓库

当我们进行项目开发的时候，需要将origin克隆到本地，这时我们称呼本地的git仓库为本地开发仓库。下面这个命令将原始的`flow-demo.git`克隆到本地。

    $ git clone git@git.corp.anjuke.com:corp/flow-demo.git

在本地仓库查看所有分支

    $ git branch -a

将显示

    * master
      remotes/origin/beta
      remotes/origin/ga
      remotes/origin/master

### 共享开发仓库

当我们有项目需要多人共同开发时，又希望其他合作者可以直接push代码，可以设置共享仓库。

#### 创建共享仓库

例如想要将的本地仓库推到服务器上，

    $ git remote add enzhang git@git.corp.anjuke.com:enzhang/flow-demo.git
    $ git push enzhang master:master

者两个命令的格式是，

* `git remote add (共享仓库的名字) (共享仓库的地址)`
* `git push (共享仓库的名字) (要推送的本地的分支):(推送到远程仓库上的分支)`

其中**共享仓库名**在GitCorp环境下建议使用开发者的用户名，或者开发者的用户名为前缀


缺省情况下，这个共享仓库还只有创建者一个人有写权限，希望其他同事可以直接推送代码，还需要设置权限。可以先查看现有的用户权限

    $ ssh git@git.corp.anjuke.com getperms enzhang/flow-demo.git

输出:

    READERS @all

上面的`getperms`命令看到所有人有读权限。下面我们在原有权限的基础上给所有人增加写权限，

    $ (ssh git@git.corp.anjuke.com getperms enzhang/flow-demo.git \
        && echo "WRITERS @all") | \
        ssh git@git.corp.anjuke.com setperms enzhang/flow-demo.git

输出:

    New perms are:
    READERS @all
    WRITERS @all

现在所有人对这个共享的代码仓库都拥有读写权限了。

#### 使用共享仓库

当你需要加入别人创建的共享代码仓库时，只需要将其共享仓库加入git的remote里。

    $ git remote add enzhang git@git.corp.anjuke.com:enzhang/flow-demo.git

建议使用对方的开发者目录名作为git的remote名称，这样同时加入多个开发者的共享仓库也不容易混淆。

遵守我们之前的约定，**origin都应该指向原始代码仓库**。

## 项目开发 - Feature Development

已经准备好了本地代码仓库，现在看看项目开发的具体流程。

### 开发者信息

对于公司的项目，建议采用真实姓名和公司的邮箱地址作为git用户的信息。

    $ git config user.name "张尔宁"
    $ git config user.email "enzhang@anjuke.com"

### 项目开发分支

通常对代码的改进都应该在分支上进行，确认完成后再合并回去。特别是项目的开发，由于改动一般比较多，应该新建立一个对应该项目的分支。我们约定这个分支以PMT的编号开始，例如`pmt1001`。

正常的项目分支从`master`分支拉出，(本地的`master`分支，跟踪`origin/master`的)

    $ git checkout -b pmt1001 master

写代码，提交

    $ echo "这是PMT1001的文件" > pmt1001.txt
    $ git add pmt1001.txt
    $ git commit -m "增加某功能"

再写代码，提交

    $ echo "增加了一项功能" >> pmt1001.txt
    $ git commit -a -m "又增加了一项功能"

项目完成，合并回master，注意`--no-ff`参数。

    $ git checkout master
    $ git merge --no-ff pmt1001 -m '项目pmt1001完成，合并回master'

此时`pmt1001`这个分支已经不用，可以删除了。

    $ git branch -d pmt1001

git的日志大致如下

    $ git log --graph --abbrev-commit --pretty=format:'%h -%d %s'

    *   43fe646 - (HEAD, master) 项目pmt1001完成，合并回master
    |\
    | * 43a7407 - 增加了一项功能
    | * 757c89e - 增加某功能
    |/
    * 0774825 - (origin/master) GitCorp Flow Demo Initial Version

将改动push回原始仓库

    $ git push origin master:master

### 并行的项目

同一时候会有很多个项目在并行开发，假设现在有两个并行的项目`pmt1002`和`pmt1003`。我们先从`master`创建两个项目分支

    $ git branch pmt1002 master
    $ git branch pmt1003 master

然后在`pmt1002`分支里进行开发

    $ git checkout pmt1002
    $ echo "这是PMT1002的文件" > pmt1002.txt
    $ git add pmt1002.txt
    $ git commit -m '增加pmt1002文件'
    $ echo "修改PMT1002的文件" >> pmt1002.txt
    $ git commit -a -m '修改pmt1002文件'

同时，在`pmt1003`分支里进行开发

    $ git checkout pmt1003
    $ echo "这是PMT1003的文件" > pmt1003.txt
    $ git add pmt1003.txt
    $ git commit -m '增加pmt1003文件'
    $ echo "修改PMT1003的文件" >> pmt1003.txt
    $ git commit -a -m '修改pmt1003文件'

看有下两个分支的情况，`pmt1003`

    $ git log pmt1003 --graph --abbrev-commit --pretty=format:'%h -%d %s'
    * dad3119 - (HEAD, pmt1003) 修改pmt1003文件
    * c69f390 - 增加pmt1003文件
    *   43fe646 - (origin/master, master) 项目pmt1001完成，合并回master
    |\
    | * 43a7407 - 又增加了一项功能
    | * 757c89e - 增加某功能
    |/
    * 0774825 - GitCorp Flow Demo Initial Version

`pmt1002`

    $ git log pmt1002 --graph --abbrev-commit --pretty=format:'%h -%d %s'
    * e174508 - (pmt1002) 修改pmt1002文件
    * b2698ed - 增加pmt1002文件
    *   43fe646 - (origin/master, master) 项目pmt1001完成，合并回master
    |\
    | * 43a7407 - 又增加了一项功能
    | * 757c89e - 增加某功能
    |/
    * 0774825 -  GitCorp Flow Demo Initial Version

这时`pmt1003`开发完毕，我们按照之前的流程，将`pmt1003`分支合并回`master`。

    $ git checkout master
    $ git merge --no-ff pmt1003 -m '项目pmt1003完成，合并回master'

将得到这样的`master`分支

    $ git log --graph --abbrev-commit --pretty=format:'%h -%d %s'
    *   2110b32 - (HEAD, master) 项目pmt1003完成，合并回master
    |\
    | * dad3119 - (pmt1003) 修改pmt1003文件
    | * c69f390 - 增加pmt1003文件
    |/
    *   43fe646 - (origin/master) 项目pmt1001完成，合并回master
    |\
    | * 43a7407 - 又增加了一项功能
    | * 757c89e - 增加某功能
    |/
    * 0774825 - GitCorp Flow Demo Initial Version

然后`pmt1002`继续开发

    $ git checkout pmt1002
    $ echo "这是PMT1002的文件" > pmt1003.txt
    $ git add pmt1003.txt
    $ git commit -m '创建了一个和pmt1003冲突的文件'

同样，`pmt1002`开发完毕，将`pmt1002`分支合并回`master`

    $ git checkout master
    $ git merge --no-ff pmt1002 -m '项目pmt1002完成，合并回master'

这时合并遇到冲突了。解决冲突后再提交。

    $ echo "这是PMT1003，PMT1002的共同文件" > pmt1003.txt
    $ git add pmt1003.txt
    $ git commit -m '项目pmt1002完成，合并回master'

现在`master`分支的历史记录为

    $ git log --graph --abbrev-commit --pretty=format:'%h -%d %s'
    *   f7c4c36 - (HEAD, master) 项目pmt1002完成，合并回master
    |\
    | * 6ec4909 - (pmt1002) 创建了一个和pmt1003冲突的文件
    | * e174508 - 修改pmt1002文件
    | * b2698ed - 增加pmt1002文件
    * |   2110b32 - 项目pmt1003完成，合并回master
    |\ \
    | |/
    |/|
    | * dad3119 - (pmt1003) 修改pmt1003文件
    | * c69f390 - 增加pmt1003文件
    |/
    *   43fe646 - (origin/master) 项目pmt1001完成，合并回master
    |\
    | * 43a7407 - 又增加了一项功能
    | * 757c89e - 增加某功能
    |/
    * 0774825 - GitCorp Flow Demo Initial Version

分支合并的线路图显示，`pmt1003`分支从`43fe646`开始，合并到`2110b32`；`pmt1002`分支也从`43fe646`开始，合并到`f7c4c36`

### rebase

除了直接合并回`master`，还可以选择在项目分支内先进行**rebase**再合并回去的方法。我们先将`master`分支回退到`pmt1002`合并前的状态

    $ git reset --hard 2110b32

然后采用在分支`pmt1001`下执行rebase

    $ git checkout pmt1002
    $ git rebase master

一样会遇到合并冲突，着手解决后继续

    $ echo "这是PMT1003，PMT1002的共同文件" > pmt1003.txt
    $ git add pmt1003.txt
    $ git rebase --continue

完成后`pmt1002`的分支成为

    $ git log --graph --abbrev-commit --pretty=format:'%h -%d %s'
    * b0fe0ac - (HEAD, pmt1002) 创建了一个和pmt1003冲突的文件
    * e402ad1 - 修改pmt1002文件
    * 48faf83 - 增加pmt1002文件
    *   2110b32 - (master) 项目pmt1003完成，合并回master
    |\
    | * dad3119 - (pmt1003) 修改pmt1003文件
    | * c69f390 - 增加pmt1003文件
    |/
    *   43fe646 - (origin/master) 项目pmt1001完成，合并回master
    |\
    | * 43a7407 - 又增加了一项功能
    | * 757c89e - 增加某功能
    |/
    * 0774825 - GitCorp Flow Demo Initial Version

这时再将`pmt1002`分支合并回`master`

    $ git checkout master
    $ git merge --no-ff pmt1002 -m '项目pmt1002完成，合并回master'

此时，`master`分支为

    $ git log --graph --abbrev-commit --pretty=format:'%h -%d %s'
    *   b42d61d - (HEAD, master) 项目pmt1002完成，合并回master
    |\
    | * b0fe0ac - (pmt1002) 创建了一个和pmt1003冲突的文件
    | * e402ad1 - 修改pmt1002文件
    | * 48faf83 - 增加pmt1002文件
    |/
    *   2110b32 - 项目pmt1003完成，合并回master
    |\
    | * dad3119 - (pmt1003) 修改pmt1003文件
    | * c69f390 - 增加pmt1003文件
    |/
    *   43fe646 - (origin/master) 项目pmt1001完成，合并回master
    |\
    | * 43a7407 - 又增加了一项功能
    | * 757c89e - 增加某功能
    |/
    * 0774825 - GitCorp Flow Demo Initial Version

这里rebase的优点是，冲突在项目的分支里解决，并且`master`上图看起来比较清晰。

不再使用的分支应该删掉

    $ git branch -d pmt1002 pmt1003

**注意**: 已经push到服务器上的commits不应该再改动，否则再改动之前已经fetch这些commits的同事可能遇到麻烦。


### 多人共同开发一个项目

现在来说一下，多个人共同开发一个项目分支的情况。这在我们平时的项目开发中是最常见的一种情况。

            /------------\    +------------+      +------------+ +------------+
            | origin     |    | enzhang    |      | bob        | | alice      |
            |  * master  |    |  * master  |  ... |  * master  | |  * master  |
            |  * beta    |    |  - pmt1004 |      |  - pmt1005 | |  - pmt1006 |
            |  * ga      |    |            |      |            | |            |
            \------------/    +------------+      +------------+ +------------+
             origin repo \    / shared repo         shared repo    shared repo
                  |       \  /       |
                  |        \/        |
                  |        /\        |
                  |       /  \       |
     local repo   |      /    \      |    local repo
    +--------------------+    +---------------------+
    | enzhang            |    | liangshan           |
    |  * master          |    |   * master          |
    |  - pmt1004         |    |   - pmt1004         |
    |  * origin/master   |    |   * origin/master   |
    |  * enzhang/master  |    |   * enzhang/master  |
    |  * enzhang/pmt1004 |    |   * enzhang/pmt1004 |
    +--------------------+    +---------------------+

举例开发项目`pmt1004`。

enzhang首先开始。如果之前的例子，首先在本地仓库建立`pmt1004`分支

    $ git checkout -b pmt1004 master

将需要共同开发的项目分支推送到共享代码仓库中 (假设共享的代码仓库之前已经设置好)

    $ git push enzhang pmt1004:pmt1004

并通知合作伙伴liangshan，在共享仓库的`pmt1004`分支下一起开发。同时自己还在继续进行开发

    $ echo "PMT1004" > pmt1004.txt
    $ git add pmt1004.txt
    $ git commit -m '完成pmt1004第一个功能点'
    $ echo "PMT1004.balabala" >> pmt1004.txt
    $ git commit -a -m '完成pmt1004第二个功能点'

假设liangshan已经从原始仓库clone了一份代码，如果还没有需要执行

    $ git clone git@git.corp.anjuke.com:corp/flow-demo.git

liangshan在接到共享的开发分支位置后，开始工作

    $ git remote add enzhang git@git.corp.anjuke.com:enzhang/flow-demo.git
    $ git fetch enzhang
    $ git checkout -b pmt1004 enzhang/pmt1004
    $ echo "PMT1004" > pmt1004.ls.txt
    $ git add pmt1004.ls.txt
    $ git commit -m '完成pmt1004第三个功能点'
    $ echo "PMT1004.balabala" >> pmt1004.ls.txt
    $ git commit -a -m '完成pmt1004第四个功能点'

这时，enzhang将本地的最新修改push到共享仓库

    $ git push enzhang pmt1004

之后，liangshan也将最新的修改push到共享仓库

    $ git push enzhang pmt1004

这时可以看到一窜错误提示，无法fast-forward合并

    To git@git.corp.anjuke.com:enzhang/flow-demo.git
     ! [rejected]        pmt1004 -> pmt1004 (non-fast-forward)
    error: failed to push some refs to 'git@git.corp.anjuke.com:enzhang/flow-demo.git'
    To prevent you from losing history, non-fast-forward updates were rejected
    Merge the remote changes (e.g. 'git pull') before pushing again.  See the
    'Note about fast-forwards' section of 'git push --help' for details.

应该在本地先合并好再push到远程的共享仓库，并且建议使用rebase参数

    $ git pull --rebase enzhang
    $ git push enzhang pmt1004

enzhang还继续在本来进行新的开发

    $ echo "PMT1004.foobar" >> pmt1004.txt
    $ git commit -a -m '完成pmt1004第五个功能点'

当enzhang需要push代码到共享仓库的时候，同样需要pull再push

    $ git pull --rebase enzhang
    $ git push enzhang

现在项目开发完成，由liangshan来负责合并代码到`master`分支

    $ git fetch --all
    $ git checkout pmt1004
    $ git rebase enzhang/pmt1004
    $ git checkout master
    $ git rebase origin/master
    $ git merge pmt1004 --no-ff -m '项目pmt1004完成'
    $ git push origin

由于项目`pmt1004`已经开发完毕，而且已经合并入`master`分支，可以删除这个分支了

    $ git branch -d pmt1004

enzhang的共享代码仓库里的pmt1004分支也可以删除

    $ git push enzhang :pmt1004

看看此时`master`分支的情况

    $ git log --graph --abbrev-commit --pretty=format:'%h -%d %s <%an>'
    *   6f6cb8f - (HEAD, origin/master, master) 项目pmt1004完成 <梁山>
    |\
    | * fe42061 - 完成pmt1004第五个功能点 <张尔宁>
    | * a581641 - 完成pmt1004第四个功能点 <梁山>
    | * 35a2cc4 - 完成pmt1004第三个功能点 <梁山>
    | * 3b92be2 - 完成pmt1004第二个功能点 <张尔宁>
    | * 74746a5 - 完成pmt1004第一个功能点 <张尔宁>
    |/
    *   b42d61d - 项目pmt1002完成，合并回master <张尔宁>
    |\
    | * b0fe0ac - 创建了一个和pmt1003冲突的文件 <张尔宁>
    | * e402ad1 - 修改pmt1002文件 <张尔宁>
    | * 48faf83 - 增加pmt1002文件 <张尔宁>
    |/
    *   2110b32 - 项目pmt1003完成，合并回master <张尔宁>
    |\
    | * dad3119 - 修改pmt1003文件 <张尔宁>
    | * c69f390 - 增加pmt1003文件 <张尔宁>
    |/
    *   43fe646 - 项目pmt1001完成，合并回master <张尔宁>
    |\
    | * 43a7407 - 又增加了一项功能 <张尔宁>
    | * 757c89e - 增加某功能 <张尔宁>
    |/
    * 0774825 - GitCorp Flow Demo Initial Version

也可以仅查看在`master`分支上的提交

    $ git log --graph --abbrev-commit --pretty=format:'%h -%d %s <%an>' --first-parent
    * 6f6cb8f - (HEAD, origin/master, master) 项目pmt1004完成 <梁山>
    * b42d61d - 项目pmt1002完成，合并回master <张尔宁>
    * 2110b32 - 项目pmt1003完成，合并回master <张尔宁>
    * 43fe646 - 项目pmt1001完成，合并回master <张尔宁>
    * 0774825 - GitCorp Flow Demo Initial Version
    
## 代码发布

各事业部具体代码发布的策略不尽相同。这里暂不描述发布的流程。

### BETA预发布

BETA预发布我们定义为部署在正式的生成环境，由小部分可选择的真实用户访问的版本。

### GA正式发布

GA版本我们定义为部署在正式的生成环境，对真实用户开发的版本。

### 自定义版本的发布

发布到正式环境以外的其他地方。一般为测试环境，例如功能测试环境等。

### hotfix

hotfix一般为production bug，必须立即在非master分支(如beta/ga)上直接修改的。

hotfix的修改还需要合并回master分支。ga版本的hotfix修改可能还需要合并回beta。

