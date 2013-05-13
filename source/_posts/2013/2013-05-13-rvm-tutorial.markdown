---
layout: post
title: "RVM Tutorial"
date: 2013-05-13 11:30
comments: true
categories:
---

RVM是Ruby Version Manager的缩写，是管理Ruby版本的工具，方便在开发、部署的时候于多个Ruby环境中切换与使用

关于RVM的信息，请参考[官方网站](https://rvm.io)的最新资料为准

<!-- more -->

## 安装rvm

以bash为例

```
curl -kL https://get.rvm.io | bash -s stable
```

**注意：rvm安装后会自动在~/.bash_profile里添加如下的代码，目的在于让非login shell也能加载rvm的环境以使用，根据不同的Linux发行版和shell请自行调整**

```
[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm" # Load RVM into a shell session *as a function
```

## 升级rvm

升级到最新的stable版本

```
rvm get stable
```

升级到最新版本

```
rvm get latest
rvm get head
rvm get master
```

## 帮助

除了文档外，还可以使用以下命令获取帮助信息

```
rvm help [command]
```

# Ruby

使用rvm管理ruby的版本，可以在多个ruby的实现、版本间切换，易于开发和部署

**关于Ruby的实现，Ruby本身只是一份编程语言的规范；我们常说的Ruby其实指的是MRI（Matz's Ruby Implementation），即Matz亲自实现的Ruby，也称作CRuby（因为是C实现的）；它也是对最新Ruby Spec实现功能最全的一种版本**

Ruby和Python一样，可以用不同的平台实现，例如

* jruby （JVM实现）
* ree （Ruby Enterprice Edition，带上一些性能、稳定性和patch的MRI）
* ironruby （.NET实现）
* goruby (Go实现)

**rvm里安装ruby后，会把rubygems和bundler一起安装**

## 列出ruby版本

列出已经安装的ruby版本

```
rvm list
```

列出已知的ruby版本

```
rvm list known
```

## 安装ruby

**如果没有特殊版本需求，推荐使用最新稳定的ruby版本以获取最新的功能支持**

首先查看安装依赖，rvm会根据不同的系统环境给出依赖的提示，依次安装它们

```
rvm requirements
```


安装ruby

```
rvm install 2.0.0
rvm install 1.9.3
rvm install 1.9.3-p327
rvm install 1.8.7
rvm install ruby-head

rvm install jruby
rvm install goruby
rvm install ree
rvm install ironruby

```

## 使用rvm中的Ruby

默认RVM使用的是系统自带的ruby（如果没有则没有...）

**关于gemset：系统默认的gem都是统一的目录，在rvm中有一个global的gemset和各个不同名字的gemset（默认的叫default），用于隔离不同gem环境，详细请参考后面gemset小节**

在当前shell中使用rvm中特定版本的ruby

```
rvm use 2.0.0
rvm use 1.9.3

rvm use jruby

rvm use system # 切换到系统默认ruby
```

使用不同的ruby与gemset （关于创建gemset请参考后面小节）

```
rvm use 2.0.0@default
rvm use 1.9.3@foo
rvm use jruby@jfoo
```


# gemset

RVM默认采用gemset隔离gem环境，类似于Python的virtualenv。

除了可以使用github的bootstrap脚本（内部采用bundler的实现）来隔离项目的gemset外，也可以使用rvm自带的gemset功能

**存在特殊的gemset，把一切全局使用的gem安装在其中，例如rake、bundler等**

创建gemset

```
rvm gemset create foo
```

列出gemset

```
rvm gemset list # 注意这个是列出当前ruby版本的gemset 不是全部
rvm gemset list_all # 列出全部gemset
```

使用特定gemset

```
rvm use 2.0.0@foo
rvm use 2.0.0@global
```


# ruby版本控制文件

在特定目录可以使用配置文件控制使用rvm中ruby的版本

**rvm原先的.rvmrc已经在1.19版本开始deprecated，推荐用.ruby-version和.ruby-gemset以和其他Ruby版本管理工具保持兼容性**

## .ruby-version

控制ruby的版本，在目录中写入文件.ruby-version，内容如

```
ruby-1.9.3-p392
```

## .ruby-gemset

控制gemset，在目录中写入文件.ruby-gemset，内容如

```
foo
```

\_\_END\_\_
