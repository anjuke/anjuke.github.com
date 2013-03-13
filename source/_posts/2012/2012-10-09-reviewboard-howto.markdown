---
layout: post
title: "reviewboard 简要使用介绍"
date: 2012-10-09 09:06
comments: true
categories:
---

{%img right http://0.gravatar.com/avatar/ce1e13bbf946c92e2abf740f8909bafa %}

## Hold a sec! What is reviewboard ?

[reviewboard](http://www.reviewboard.org/)是一款基于web的强大的code
review工具，他提供了便捷、强大的功能来帮助开发者更顺利地进行code review这一核心工作。支持众多版本控制系统。

<!-- more -->

## reviewboard in Anjuke Inc.

在安居客，我们使用reviewboard来进行code review，他的地址是

[http://git.corp.anjuke.com/reviews](http://git.corp.anjuke.com/reviews)

使用域帐号登录

{% img center /images/2012/10/reviewboard001.jpg %}

登录后，找到右上角的`my
account`，进入可以设置自己所在的组，这里需要设置到您所在的组，这样发送到这个组的review请求才会发送通知邮件

{% img center /images/2012/10/reviewboard002.jpg %}

## How could I get access ?

### 安装环境与配置

**以下这些操作都是针对unix-like的操作系统，如果您使用的是windows，可以在本地生成diff文件后上传**

RBTools包含了一些reviewboard的自动脚本，由[Python](http://python.org)写成，可以通过Python的包管理器`setuptools`或者`pip`安装

安装RBTools可以利用以下命令 (必要的时候需要sudo)

```
easy_install RBTools
```

或者

```
pip install RBTools
```

如果没有安装easy_install或者pip，以ubuntu为例，可以使用以下命令安装

```
aptitude install python-setuptools
aptitude install python-pip
```

关于Python环境，推荐使用[pythonbrew](https://github.com/utahta/pythonbrew)，以和系统自带的Python环境隔离

安装完成后，会发现有多出一个`post-review`的命令，它就是我们用来和reviewboard发送review请求的工具


## Looks great, let's get this thing done


### 设置代码仓库

如果您的仓库还未在reviewboard里可用，那么需要先设置一下

如果您不知道怎么做，可以联系管理员(例如尔宁、Sysdev)

使用admin帐号登录，并选择到后台

{%img center /images/2012/10/reviewboard003.jpg %}

{%img center /images/2012/10/reviewboard004.jpg %}

{%img center /images/2012/10/reviewboard005.jpg %}

新建仓库，设置参考下图，这里的名字将会在后面`.reviewboardrc`里使用

{%img center /images/2012/10/reviewboard006.jpg %}

### 配置.reviewboardrc

`post-review`命令会检查当前目录和用户的`$HOME`下是否有这个配置，里面可以写一些有用的信息来帮助`post-review`认识仓库和请求的一些信息

_**推荐: 在每个代码版本仓库下都保留一个通用的`.reviewboardrc`配置**_

一个典型的`.reviewboardrc`配置内容如下

```
REVIEWBOARD_URL="http://git.corp.anjuke.com/reviews"
REPOSITORY='image-service'
```

`REVIEWBOARD_URL`设置了reviewboard的地址，`REPOSITORY`则是之前设置好的仓库名字

### 发起review请求

对于发送review请求，主要可以有2种形式:

1. 先发送请求，再手动上传diff文件
2. 发送请求的同时上传diff文件


#### 创建请求并手工上传diff

简单的在有.reviewboardrc配置的目录下执行命令，或者在界面上新建，效果是一样的:

```
post-review
```

_**注意: 第一次执行会要求输入用户名和密码；当然也可以在`.reviewboardrc`里设置**_

![](/medias/20121009/rb.howto.001.png)

如上图会得到一个review请求的地址，访问它，再手工上传diff文件

{%img center /images/2012/10/reviewboard007.jpg %}

此外，还需要填入一些必要的信息

* summary
* branch
* 选择group或者people
* group的话，就和第一步准备里加入的组有关系了，组里人的才能收到邮件

在这个阶段和发布后，自己以及别人就可以通过这个URL来看到您的review请求，点右上角的`view diff`可以看到具体的代码diff情况，并作出评价等

#### 生成diff的方法

参考如下

```
git diff $sha1sum_1 $sha1sum_2 > foobar.diff # 注意是从1到2的变化，所以别搞反了
```

完成后点击publish即可发布这个review请求。

### 发起请求的同时上传diff文件

在git仓库里执行`post-review`时，可以通过参数直接生成diff文件并上传，命令实例:

```
post-review --revision-range=970521dfb4c42f89b1a4749cec77cd919c184632:401097033760d6b188039b11aab0c853b68541b4
```

_**注意: 这里第一个版本必须在远程的代码仓库里存在，否则将无法发布review请求**_

到生成的URL里发现已经有diff文件上传好了，只需要填入必要的字段就可以发布了。

### 便捷的方法

这里给出一个较为便捷的发起请求的方法，参数会告诉post-review来根据git的提交信息自动获取描述信息

```
post-review \
--guess-summary \ # 自动根据git commit comment提取
--guess-description \ # 自动根据git commit comment提取
--revision-range=":" \ # 这里填上需要的版本号
--branch="master" # 替换成需要的分支
```

## Oops

如果出现错误，这种时候请检查一下`.reviewboardrc`配置是否有问题；还可以通过`--debug`参数查看详细信息。

如果还是搞不定的话，果断求助吧!

## More?

抱歉，目前我也只能写这么多，更多关于`reviewboard`和`RBTools`的应用请翻阅其他资料网站或者其他手册。

* * *

Last updated 2012/11/27 By Lei Chen [chenlei@anjuke.com](mailto:chenlei@anjuke.com) from
sysdev team

